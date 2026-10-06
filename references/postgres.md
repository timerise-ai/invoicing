# Postgres: schema, gapless numbering, immutability, the store

The production implementation of `InvoiceStore` from [data-model.md](data-model.md). Three properties live
in the database, not in application code, because only the database can hold them under concurrency and
against a client that bypasses the app:

| Property | Held by |
|---|---|
| A number is consumed only by a document that exists | The counter is bumped by a trigger inside the insert, so a failed insert rolls the bump back |
| A document and its lines appear together or not at all | `invoicing_create` is one function call, one transaction |
| An issued document does not change | Update and delete triggers on both tables |

## The migration

```sql
-- db/migrations/0001_invoicing.sql

create table invoices (
  id                      uuid primary key default gen_random_uuid(),
  tenant_id               text not null,
  kind                    text not null check (kind in ('base', 'correction')),
  series                  text not null,
  year                    int  not null,
  seq                     int  not null,
  number                  text not null,
  issue_date              date not null,
  sale_date               date not null,
  due_date                date,
  payment_method          text,
  currency                text not null default 'PLN',
  -- Both parties are snapshots taken at issue, never a live join.
  seller_name             text not null,
  seller_tax_id           text,
  seller_address          text,
  seller_bank_account     text,
  buyer_name              text not null,
  buyer_tax_id            text,
  buyer_address           text,
  customer_id             text,
  source_kind             text,
  source_id               text,
  total_net               numeric(12,2) not null,
  total_vat               numeric(12,2) not null,
  total_gross             numeric(12,2) not null,
  vat_exemption_basis     text,
  corrects_invoice_id     uuid references invoices(id),
  correction_reason       text,
  corrected_by_invoice_id uuid references invoices(id),
  paid_at                 timestamptz,
  e_invoice_status        text not null default 'not_required'
    check (e_invoice_status in ('not_required', 'pending', 'accepted', 'rejected')),
  e_invoice_number        text,
  e_invoice_xml_sha256    text,
  e_invoice_error         text,
  created_by              text not null,
  created_at              timestamptz not null default now(),
  unique (tenant_id, series, year, seq),
  unique (tenant_id, number),
  constraint invoices_correction_link check ((kind = 'correction') = (corrects_invoice_id is not null)),
  constraint invoices_correction_reason check (corrects_invoice_id is null or correction_reason is not null),
  constraint invoices_source_pair check ((source_kind is null) = (source_id is null))
);

-- One base document per source. Corrections point at the invoice, not at the source.
create unique index invoices_source_uniq
  on invoices (tenant_id, source_kind, source_id)
  where source_id is not null and kind = 'base';
-- One correction per base document.
create unique index invoices_corrects_uniq
  on invoices (corrects_invoice_id)
  where corrects_invoice_id is not null;
create index invoices_list_idx on invoices (tenant_id, issue_date desc, created_at desc, id desc);
create index invoices_customer_idx on invoices (tenant_id, customer_id) where customer_id is not null;

create table invoice_lines (
  invoice_id  uuid not null references invoices(id) on delete cascade,
  tenant_id   text not null,
  position    int  not null,
  name        text not null,
  -- Signed on a correction: a delta row may carry a negative quantity and negative amounts.
  qty         numeric(12,2) not null,
  unit        text not null,
  unit_price  numeric(12,2) not null,
  vat_rate    text not null check (vat_rate in ('23', '8', '5', '0', 'zw', 'np')),
  net         numeric(12,2) not null,
  vat         numeric(12,2) not null,
  gross       numeric(12,2) not null,
  primary key (invoice_id, position)
);

create table invoice_number_counters (
  tenant_id text not null,
  series    text not null,
  year      int  not null,
  last_seq  int  not null default 0,
  primary key (tenant_id, series, year)
);

-- The number is stamped inside the insert. The upsert takes a row lock on the counter, so concurrent
-- issuers queue and receive consecutive values; if the insert fails, the bump rolls back with it.
create function invoicing_assign_number() returns trigger
language plpgsql as $$
declare
  v_seq int;
begin
  insert into invoice_number_counters (tenant_id, series, year, last_seq)
  values (new.tenant_id, new.series, new.year, 1)
  on conflict (tenant_id, series, year)
  do update set last_seq = invoice_number_counters.last_seq + 1
  returning last_seq into v_seq;

  new.seq := v_seq;
  -- lpad truncates a longer string to the length given, so the width is never less than the number's own.
  new.number := new.series || '/' || new.year::text || '/'
    || lpad(v_seq::text, greatest(5, length(v_seq::text)), '0');
  return new;
end;
$$;

create trigger invoices_assign_number
  before insert on invoices
  for each row execute function invoicing_assign_number();

-- An issued document is fixed. Only payment, the correction link and the e-invoice mirror may change,
-- and the first two only once.
create function invoicing_guard_update() returns trigger
language plpgsql as $$
declare
  v_mutable constant text[] := array[
    'paid_at', 'corrected_by_invoice_id',
    'e_invoice_status', 'e_invoice_number', 'e_invoice_xml_sha256', 'e_invoice_error'
  ];
begin
  if (to_jsonb(new) - v_mutable) is distinct from (to_jsonb(old) - v_mutable) then
    raise exception 'invoicing:immutable';
  end if;
  if old.paid_at is not null and new.paid_at is distinct from old.paid_at then
    raise exception 'invoicing:immutable';
  end if;
  if old.corrected_by_invoice_id is not null
     and new.corrected_by_invoice_id is distinct from old.corrected_by_invoice_id then
    raise exception 'invoicing:immutable';
  end if;
  return new;
end;
$$;

create trigger invoices_guard_update
  before update on invoices
  for each row execute function invoicing_guard_update();

-- Lines never change, and nothing is deleted. A host that must erase a whole tenant sets
-- `set local invoicing.allow_delete = 'on'` inside that one transaction.
create function invoicing_forbid_change() returns trigger
language plpgsql as $$
begin
  if tg_op = 'DELETE' and current_setting('invoicing.allow_delete', true) = 'on' then
    return old;
  end if;
  raise exception 'invoicing:immutable';
end;
$$;

create trigger invoice_lines_forbid_change
  before update or delete on invoice_lines
  for each row execute function invoicing_forbid_change();

create trigger invoices_forbid_delete
  before delete on invoices
  for each row execute function invoicing_forbid_change();

-- One call, one transaction: the document, its lines, and for a correction the link on the original.
-- Errors are raised as 'invoicing:<code>' so the store can map them onto InvoiceError codes.
create function invoicing_create(p jsonb) returns uuid
language plpgsql as $$
declare
  v_id         uuid;
  v_original   invoices%rowtype;
  v_constraint text;
begin
  if p->>'kind' = 'correction' then
    -- The row lock is what serialises two corrections of the same document.
    select * into v_original
    from invoices
    where id = (p->>'correctsInvoiceId')::uuid and tenant_id = p->>'tenantId'
    for update;
    if not found then raise exception 'invoicing:not_found'; end if;
    if v_original.kind <> 'base' then raise exception 'invoicing:not_correctable'; end if;
    if v_original.corrected_by_invoice_id is not null then raise exception 'invoicing:already_corrected'; end if;
  end if;

  begin
    insert into invoices (
      tenant_id, kind, series, year, issue_date, sale_date, due_date, payment_method, currency,
      seller_name, seller_tax_id, seller_address, seller_bank_account,
      buyer_name, buyer_tax_id, buyer_address, customer_id, source_kind, source_id,
      total_net, total_vat, total_gross, vat_exemption_basis,
      corrects_invoice_id, correction_reason, e_invoice_status, created_by
    ) values (
      p->>'tenantId', p->>'kind', p->>'series', (p->>'year')::int,
      (p->>'issueDate')::date, (p->>'saleDate')::date, (p->>'dueDate')::date,
      p->>'paymentMethod', p->>'currency',
      p#>>'{seller,name}', p#>>'{seller,taxId}', p#>>'{seller,address}', p#>>'{seller,bankAccount}',
      p#>>'{buyer,name}', p#>>'{buyer,taxId}', p#>>'{buyer,address}',
      p->>'customerId', p#>>'{source,kind}', p#>>'{source,id}',
      (p#>>'{totals,net}')::numeric, (p#>>'{totals,vat}')::numeric, (p#>>'{totals,gross}')::numeric,
      p->>'vatExemptionBasis',
      (p->>'correctsInvoiceId')::uuid, p->>'correctionReason', p->>'eInvoiceStatus', p->>'createdBy'
    )
    returning id into v_id;
  exception when unique_violation then
    get stacked diagnostics v_constraint = constraint_name;
    if v_constraint = 'invoices_source_uniq' then raise exception 'invoicing:source_already_invoiced'; end if;
    if v_constraint = 'invoices_corrects_uniq' then raise exception 'invoicing:already_corrected'; end if;
    raise;
  end;

  insert into invoice_lines (invoice_id, tenant_id, position, name, qty, unit, unit_price, vat_rate, net, vat, gross)
  select v_id, p->>'tenantId', l.ordinality, l.value->>'name', (l.value->>'qty')::numeric, l.value->>'unit',
         (l.value->>'unitPrice')::numeric, l.value->>'vatRate',
         (l.value->>'net')::numeric, (l.value->>'vat')::numeric, (l.value->>'gross')::numeric
  from jsonb_array_elements(p->'lines') with ordinality as l(value, ordinality);

  if p->>'kind' = 'correction' then
    update invoices set corrected_by_invoice_id = v_id where id = v_original.id;
  end if;

  return v_id;
end;
$$;
```

## The store

Written against the smallest surface every Postgres client offers: a function that takes SQL text and
parameters and returns rows. `pg`'s `Pool` satisfies `Queryable` as it is. For postgres.js, Neon's serverless
driver or Drizzle's `db.execute`, wrap the call in three lines so it returns `{ rows }`.

```typescript
// lib/invoicing/postgres-store.ts
import { InvoiceError, type InvoiceErrorCode, type InvoiceKind, type VatRate } from "./model";
import type {
  EInvoiceState,
  EInvoiceStatus,
  Invoice,
  InvoiceFilters,
  InvoicePage,
  InvoiceStore,
  InvoiceWithLines,
  NewInvoice,
  StoredInvoiceLine,
} from "./store";

export type Queryable = {
  query(text: string, params?: unknown[]): Promise<{ rows: Record<string, unknown>[] }>;
};

// Dates and timestamps are selected as text. A driver that parses a `date` into a JS Date does it in the
// server's local time zone, and the document's issue date then moves by a day east of Greenwich.
const INVOICE_COLUMNS = `
  id::text, tenant_id, kind, series, year, seq, number,
  issue_date::text, sale_date::text, due_date::text, payment_method, currency,
  seller_name, seller_tax_id, seller_address, seller_bank_account,
  buyer_name, buyer_tax_id, buyer_address, customer_id, source_kind, source_id,
  total_net::float8, total_vat::float8, total_gross::float8, vat_exemption_basis,
  corrects_invoice_id::text, correction_reason, corrected_by_invoice_id::text,
  to_char(paid_at at time zone 'utc', 'YYYY-MM-DD"T"HH24:MI:SS.MS"Z"') as paid_at,
  e_invoice_status, e_invoice_number, e_invoice_xml_sha256, e_invoice_error, created_by,
  -- Microseconds, not milliseconds: created_at is part of the list cursor, and a truncated value would
  -- skip rows created inside the same millisecond.
  to_char(created_at at time zone 'utc', 'YYYY-MM-DD"T"HH24:MI:SS.US"Z"') as created_at`;

const text = (value: unknown): string => String(value);
const textOrNull = (value: unknown): string | null => (value === null || value === undefined ? null : String(value));

function toInvoice(row: Record<string, unknown>): Invoice {
  const sourceKind = textOrNull(row.source_kind);
  const sourceId = textOrNull(row.source_id);
  return {
    id: text(row.id),
    tenantId: text(row.tenant_id),
    kind: text(row.kind) as InvoiceKind,
    series: text(row.series),
    year: Number(row.year),
    seq: Number(row.seq),
    number: text(row.number),
    issueDate: text(row.issue_date),
    saleDate: text(row.sale_date),
    dueDate: textOrNull(row.due_date),
    paymentMethod: textOrNull(row.payment_method),
    currency: text(row.currency),
    seller: {
      name: text(row.seller_name),
      taxId: textOrNull(row.seller_tax_id),
      address: textOrNull(row.seller_address),
      bankAccount: textOrNull(row.seller_bank_account),
    },
    buyer: {
      name: text(row.buyer_name),
      taxId: textOrNull(row.buyer_tax_id),
      address: textOrNull(row.buyer_address),
    },
    customerId: textOrNull(row.customer_id),
    source: sourceKind && sourceId ? { kind: sourceKind, id: sourceId } : null,
    totals: { net: Number(row.total_net), vat: Number(row.total_vat), gross: Number(row.total_gross) },
    vatExemptionBasis: textOrNull(row.vat_exemption_basis),
    correctsInvoiceId: textOrNull(row.corrects_invoice_id),
    correctionReason: textOrNull(row.correction_reason),
    correctedByInvoiceId: textOrNull(row.corrected_by_invoice_id),
    paidAt: textOrNull(row.paid_at),
    eInvoice: {
      status: text(row.e_invoice_status) as EInvoiceStatus,
      number: textOrNull(row.e_invoice_number),
      xmlSha256: textOrNull(row.e_invoice_xml_sha256),
      error: textOrNull(row.e_invoice_error),
    },
    createdBy: text(row.created_by),
    createdAt: text(row.created_at),
  };
}

function toLine(row: Record<string, unknown>): StoredInvoiceLine {
  return {
    position: Number(row.position),
    name: text(row.name),
    qty: Number(row.qty),
    unit: text(row.unit),
    unitPrice: Number(row.unit_price),
    vatRate: text(row.vat_rate) as VatRate,
    net: Number(row.net),
    vat: Number(row.vat),
    gross: Number(row.gross),
  };
}

const RAISED = /invoicing:([a-z_]+)/;
const KNOWN: readonly InvoiceErrorCode[] = ["not_found", "not_correctable", "already_corrected", "source_already_invoiced"];

/** `raise exception 'invoicing:<code>'` in the migration becomes InvoiceError(code) here. */
function toInvoiceError(error: unknown): unknown {
  const code = (error instanceof Error ? RAISED.exec(error.message)?.[1] : undefined) as InvoiceErrorCode | undefined;
  return code && KNOWN.includes(code) ? new InvoiceError(code) : error;
}

const E_INVOICE_COLUMNS: Record<keyof EInvoiceState, string> = {
  status: "e_invoice_status",
  number: "e_invoice_number",
  xmlSha256: "e_invoice_xml_sha256",
  error: "e_invoice_error",
};

type Cursor = [issueDate: string, createdAt: string, id: string];

export function createPostgresInvoiceStore(db: Queryable): InvoiceStore {
  async function get(tenantId: string, id: string): Promise<InvoiceWithLines | null> {
    // A malformed id is "not found", not a cast error from the database.
    if (!/^[0-9a-f-]{36}$/i.test(id)) return null;
    const head = await db.query(`select ${INVOICE_COLUMNS} from invoices where tenant_id = $1 and id = $2::uuid`, [
      tenantId,
      id,
    ]);
    const row = head.rows[0];
    if (!row) return null;
    const lines = await db.query(
      `select position, name, qty::float8, unit, unit_price::float8, vat_rate, net::float8, vat::float8, gross::float8
       from invoice_lines where invoice_id = $1::uuid order by position`,
      [id],
    );
    return { ...toInvoice(row), lines: lines.rows.map(toLine) };
  }

  return {
    async create(input: NewInvoice): Promise<InvoiceWithLines> {
      let id: string;
      try {
        const result = await db.query(`select invoicing_create($1::jsonb)::text as id`, [JSON.stringify(input)]);
        id = text(result.rows[0]?.id);
      } catch (error) {
        throw toInvoiceError(error);
      }
      const created = await get(input.tenantId, id);
      if (!created) throw new Error(`invoicing: created invoice ${id} is not readable`);
      return created;
    },

    get,

    async list(tenantId: string, filters: InvoiceFilters, page): Promise<InvoicePage> {
      const where: string[] = ["tenant_id = $1"];
      const params: unknown[] = [tenantId];
      const bind = (value: unknown): string => `$${params.push(value)}`;

      if (filters.status === "corrected") where.push("corrected_by_invoice_id is not null");
      if (filters.status === "paid") where.push("corrected_by_invoice_id is null and paid_at is not null");
      if (filters.status === "issued") where.push("corrected_by_invoice_id is null and paid_at is null");
      if (filters.eInvoiceStatus) where.push(`e_invoice_status = ${bind(filters.eInvoiceStatus)}`);
      if (filters.customerId) where.push(`customer_id = ${bind(filters.customerId)}`);
      if (filters.from) where.push(`issue_date >= ${bind(filters.from)}::date`);
      if (filters.to) where.push(`issue_date <= ${bind(filters.to)}::date`);
      const q = filters.query?.trim();
      if (q) {
        // The user's text is a literal, so the LIKE wildcards in it are escaped.
        const like = bind(`%${q.replace(/[\\%_]/g, "\\$&")}%`);
        where.push(`(number ilike ${like} or buyer_name ilike ${like} or buyer_tax_id ilike ${like})`);
      }
      if (page.cursor) {
        const [issueDate, createdAt, id] = JSON.parse(Buffer.from(page.cursor, "base64url").toString("utf8")) as Cursor;
        where.push(
          `(issue_date, created_at, id) < (${bind(issueDate)}::date, ${bind(createdAt)}::timestamptz, ${bind(id)}::uuid)`,
        );
      }

      // One row past the limit says whether there is a next page without a second count query.
      const result = await db.query(
        `select ${INVOICE_COLUMNS} from invoices where ${where.join(" and ")}
         order by issue_date desc, created_at desc, id desc limit ${bind(page.limit + 1)}`,
        params,
      );
      const rows = result.rows.slice(0, page.limit).map(toInvoice);
      const last = rows[rows.length - 1];
      const hasMore = result.rows.length > page.limit;
      const cursor: Cursor | null = hasMore && last ? [last.issueDate, last.createdAt, last.id] : null;
      return { rows, nextCursor: cursor ? Buffer.from(JSON.stringify(cursor), "utf8").toString("base64url") : null };
    },

    async markPaid(tenantId: string, id: string, at: string): Promise<boolean> {
      const result = await db.query(
        `update invoices set paid_at = $3::timestamptz
         where tenant_id = $1 and id = $2::uuid and paid_at is null returning id`,
        [tenantId, id, at],
      );
      return result.rows.length > 0;
    },

    async invoicedSourceIds(tenantId: string, kind: string, ids: readonly string[]): Promise<string[]> {
      if (ids.length === 0) return [];
      const result = await db.query(
        `select source_id from invoices
         where tenant_id = $1 and kind = 'base' and source_kind = $2 and source_id = any($3::text[])`,
        [tenantId, kind, [...ids]],
      );
      return result.rows.map((r) => text(r.source_id));
    },

    async setEInvoice(tenantId: string, id: string, patch: Partial<EInvoiceState>): Promise<void> {
      const params: unknown[] = [tenantId, id];
      const sets: string[] = [];
      for (const key of Object.keys(E_INVOICE_COLUMNS) as (keyof EInvoiceState)[]) {
        if (!(key in patch)) continue;
        params.push(patch[key] ?? null);
        sets.push(`${E_INVOICE_COLUMNS[key]} = $${params.length}`);
      }
      if (sets.length === 0) return;
      await db.query(`update invoices set ${sets.join(", ")} where tenant_id = $1 and id = $2::uuid`, params);
    },
  };
}
```

Wiring it in `lib/invoicing/index.ts` replaces one line:

```ts
// lib/invoicing/index.ts (the store line, with `pg`)
import { Pool } from "pg";
import { createPostgresInvoiceStore } from "./postgres-store";

const store = createPostgresInvoiceStore(new Pool({ connectionString: process.env.DATABASE_URL }));
```

## Other data layers

- **Supabase.** Run the migration as it is. Call the function with
  `supabase.rpc("invoicing_create", { p: input })` and read with the query builder, or point `Queryable` at
  the database through the connection pooler and use the store unchanged. With PostgREST, tenant scope
  moves into row level security: a `select` policy on both tables, an `insert` policy only if the function
  runs as the caller, and **no `update` policy on `invoices` wider than the server role**. The triggers
  above stop an edit of an issued document even where a policy allows the update. PostgREST caps a
  response at its `max_rows` setting, so an unbounded `select` of every invoiced source stops being
  complete at that cap; `invoicedSourceIds` asks only about the ids on screen.
- **Drizzle or Prisma.** Keep the migration as raw SQL: the triggers and the function are the point, and
  neither ORM expresses them. Declare the three tables in the ORM's schema for reads, and call
  `invoicing_create` through `db.execute` or `$queryRaw`. Do not rebuild `create` as two ORM inserts in an
  application transaction with a separate "next number" query: that is the shape that leaves a gap when
  the second insert fails after the counter has committed.
- **A document store.** The numbering needs a transaction that reads and writes a counter and writes the
  document together. Firestore's `runTransaction` can hold that; a store with no multi-document transaction
  cannot, and is the wrong home for this module.

## What was checked

The migration and the store were run against PostgreSQL 18 in a scratch database. The statements in
[testing-service.md](testing-service.md) under *Database checks* walk the numbering, the rollback, the
immutability triggers and the correction rules, and the service suite ran against this store. The Supabase,
Drizzle, Prisma and Firestore notes above are designs and were not run.

## Checklist

- [ ] The migration ran whole, triggers and function included
- [ ] No application code computes "the next number"
- [ ] Dates are selected as text, amounts as `float8` and rounded by the model
- [ ] Row level security, where the host has it, grants no general `update` on `invoices`
- [ ] The database checks in [testing-service.md](testing-service.md) pass against the host's database
