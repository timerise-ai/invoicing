# Data model and the store seam

One document type, its lines, and a counter. The store interface below is the data-access seam: the service
in [service.md](service.md) talks only to `InvoiceStore`, the in-memory implementation in this file backs the
tests and a first run with no database, and [postgres.md](postgres.md) carries the production one.

| Entity | Holds | Mutable after issue |
|---|---|---|
| `Invoice` | Number, dates, both parties as snapshots, totals, correction links, payment and e-invoice state | Only `paidAt`, `correctedByInvoiceId` and the `eInvoice` block |
| `StoredInvoiceLine` | Position, name, quantity, unit, gross unit price, rate, and the derived net, VAT and gross | Never |
| Number counter | The last sequence per tenant, series and year | Only by the insert that takes a number |

## Types and the interface

```typescript
// lib/invoicing/store.ts
import type { InvoiceKind, InvoiceLine, InvoiceTotals } from "./model";

/** The seller as it was when the document was issued. Read from the tenant profile once, then frozen. */
export type SellerSnapshot = {
  name: string;
  taxId: string | null;
  address: string | null;
  bankAccount: string | null;
};

export type BuyerSnapshot = { name: string; taxId: string | null; address: string | null };

/** What the invoice bills: a sale, an order, a booking. `kind` is the host's own word for it. */
export type InvoiceSourceRef = { kind: string; id: string };

export type EInvoiceStatus = "not_required" | "pending" | "accepted" | "rejected";

export type EInvoiceState = {
  status: EInvoiceStatus;
  /** The number the tax system assigned, once it accepted the document. */
  number: string | null;
  /** SHA-256, hex, of the exact bytes handed to the e-invoicing transport. Drives the verification QR. */
  xmlSha256: string | null;
  /** The rejection or enqueue failure, in the transport's own words. */
  error: string | null;
};

export type Invoice = {
  id: string;
  tenantId: string;
  kind: InvoiceKind;
  series: string;
  year: number;
  seq: number;
  number: string;
  issueDate: string;
  saleDate: string;
  dueDate: string | null;
  paymentMethod: string | null;
  currency: string;
  seller: SellerSnapshot;
  buyer: BuyerSnapshot;
  customerId: string | null;
  source: InvoiceSourceRef | null;
  totals: InvoiceTotals;
  vatExemptionBasis: string | null;
  correctsInvoiceId: string | null;
  correctionReason: string | null;
  /** Set on a base document when a correction to it is issued, in the same transaction. */
  correctedByInvoiceId: string | null;
  paidAt: string | null;
  eInvoice: EInvoiceState;
  createdBy: string;
  createdAt: string;
};

export type StoredInvoiceLine = InvoiceLine & { position: number };
export type InvoiceWithLines = Invoice & { lines: StoredInvoiceLine[] };

export type InvoiceStatus = "issued" | "paid" | "corrected";

/**
 * Status is derived, never stored. Payment and correction are separate facts: a paid invoice that is later
 * corrected is still paid, and the register can show both.
 */
export function invoiceStatus(invoice: Pick<Invoice, "paidAt" | "correctedByInvoiceId">): InvoiceStatus {
  if (invoice.correctedByInvoiceId) return "corrected";
  return invoice.paidAt ? "paid" : "issued";
}

/** Everything the store needs to write one document. `seq` and `number` are assigned by the store. */
export type NewInvoice = {
  tenantId: string;
  kind: InvoiceKind;
  series: string;
  year: number;
  issueDate: string;
  saleDate: string;
  dueDate: string | null;
  paymentMethod: string | null;
  currency: string;
  seller: SellerSnapshot;
  buyer: BuyerSnapshot;
  customerId: string | null;
  source: InvoiceSourceRef | null;
  totals: InvoiceTotals;
  vatExemptionBasis: string | null;
  correctsInvoiceId: string | null;
  correctionReason: string | null;
  eInvoiceStatus: EInvoiceStatus;
  createdBy: string;
  lines: InvoiceLine[];
};

export type InvoiceFilters = {
  status?: InvoiceStatus;
  eInvoiceStatus?: EInvoiceStatus;
  /** ISO dates, inclusive, applied to `issueDate`. */
  from?: string;
  to?: string;
  customerId?: string;
  /** Matches the number, the buyer name and the buyer tax identifier. */
  query?: string;
};

export type InvoicePage = { rows: Invoice[]; nextCursor: string | null };

export interface InvoiceStore {
  /**
   * One transaction: take the next number for (tenant, series, year), insert the document and its lines,
   * and for a correction lock the original and set its `correctedByInvoiceId`. If any step fails nothing is
   * written and the number is not consumed. Throws InvoiceError: `source_already_invoiced` when a base
   * document already bills the same source, `not_found`, `not_correctable` or `already_corrected` for a
   * correction.
   */
  create(input: NewInvoice): Promise<InvoiceWithLines>;
  get(tenantId: string, id: string): Promise<InvoiceWithLines | null>;
  /** Filtered on the server, newest issue date first. `nextCursor` is null on the last page. */
  list(tenantId: string, filters: InvoiceFilters, page: { limit: number; cursor?: string | null }): Promise<InvoicePage>;
  /** False when the invoice does not exist or is already paid. */
  markPaid(tenantId: string, id: string, at: string): Promise<boolean>;
  /** Of the given source ids, the ones a base document already bills. Feeds the "issue from" picker. */
  invoicedSourceIds(tenantId: string, kind: string, ids: readonly string[]): Promise<string[]>;
  setEInvoice(tenantId: string, id: string, patch: Partial<EInvoiceState>): Promise<void>;
}
```

## The in-memory store

Synchronous under the hood, so every `create` is atomic by construction. It enforces the same rules the
Postgres schema does, which is what lets the service tests run with no database. It keeps data for the life
of the process only: use it for tests, a demo and a first build, never for issued documents.

```typescript
// lib/invoicing/memory-store.ts
import { InvoiceError, formatInvoiceNumber } from "./model";
import {
  invoiceStatus,
  type EInvoiceState,
  type Invoice,
  type InvoiceFilters,
  type InvoicePage,
  type InvoiceStore,
  type InvoiceWithLines,
  type NewInvoice,
} from "./store";

type Entry = { invoice: InvoiceWithLines; order: number };

export type MemoryStoreOptions = {
  now?: () => string;
  newId?: () => string;
};

export function createMemoryInvoiceStore(options: MemoryStoreOptions = {}): InvoiceStore {
  const now = options.now ?? (() => new Date().toISOString());
  const newId = options.newId ?? (() => crypto.randomUUID());
  const entries = new Map<string, Entry>();
  const counters = new Map<string, number>();
  let order = 0;

  const ofTenant = (tenantId: string, id: string): Entry | undefined => {
    const entry = entries.get(id);
    return entry && entry.invoice.tenantId === tenantId ? entry : undefined;
  };

  function matches(invoice: Invoice, filters: InvoiceFilters): boolean {
    if (filters.status && invoiceStatus(invoice) !== filters.status) return false;
    if (filters.eInvoiceStatus && invoice.eInvoice.status !== filters.eInvoiceStatus) return false;
    if (filters.customerId && invoice.customerId !== filters.customerId) return false;
    if (filters.from && invoice.issueDate < filters.from) return false;
    if (filters.to && invoice.issueDate > filters.to) return false;
    const q = filters.query?.trim().toLowerCase();
    if (!q) return true;
    return `${invoice.number} ${invoice.buyer.name} ${invoice.buyer.taxId ?? ""}`.toLowerCase().includes(q);
  }

  return {
    async create(input: NewInvoice): Promise<InvoiceWithLines> {
      // Every check runs before the counter moves, so a refused document takes no number.
      if (input.kind === "base" && input.source) {
        const source = input.source;
        for (const { invoice } of entries.values()) {
          if (
            invoice.tenantId === input.tenantId &&
            invoice.kind === "base" &&
            invoice.source?.kind === source.kind &&
            invoice.source.id === source.id
          ) {
            throw new InvoiceError("source_already_invoiced");
          }
        }
      }
      let original: Entry | undefined;
      if (input.kind === "correction") {
        original = input.correctsInvoiceId ? ofTenant(input.tenantId, input.correctsInvoiceId) : undefined;
        if (!original) throw new InvoiceError("not_found");
        if (original.invoice.kind !== "base") throw new InvoiceError("not_correctable");
        if (original.invoice.correctedByInvoiceId) throw new InvoiceError("already_corrected");
      }

      const counterKey = `${input.tenantId}|${input.series}|${input.year}`;
      const seq = (counters.get(counterKey) ?? 0) + 1;
      counters.set(counterKey, seq);

      const { lines, eInvoiceStatus, ...rest } = input;
      const invoice: InvoiceWithLines = {
        ...rest,
        id: newId(),
        seq,
        number: formatInvoiceNumber(input.series, input.year, seq),
        correctedByInvoiceId: null,
        paidAt: null,
        eInvoice: { status: eInvoiceStatus, number: null, xmlSha256: null, error: null },
        createdAt: now(),
        lines: lines.map((line, i) => ({ ...line, position: i + 1 })),
      };
      entries.set(invoice.id, { invoice: structuredClone(invoice), order: ++order });
      if (original) original.invoice.correctedByInvoiceId = invoice.id;
      return invoice;
    },

    async get(tenantId: string, id: string): Promise<InvoiceWithLines | null> {
      const entry = ofTenant(tenantId, id);
      // A copy, so a caller cannot edit an issued document through the reference it was handed.
      return entry ? structuredClone(entry.invoice) : null;
    },

    async list(tenantId, filters, page): Promise<InvoicePage> {
      const after = page.cursor ? Number(page.cursor) : 0;
      const sorted = [...entries.values()]
        .filter((e) => e.invoice.tenantId === tenantId && matches(e.invoice, filters))
        .sort((a, b) => b.invoice.issueDate.localeCompare(a.invoice.issueDate) || b.order - a.order);
      const slice = sorted.slice(after, after + page.limit);
      const rows = slice.map(({ invoice }) => {
        const { lines: _lines, ...header } = structuredClone(invoice);
        return header;
      });
      const next = after + slice.length;
      return { rows, nextCursor: next < sorted.length ? String(next) : null };
    },

    async markPaid(tenantId: string, id: string, at: string): Promise<boolean> {
      const entry = ofTenant(tenantId, id);
      if (!entry || entry.invoice.paidAt) return false;
      entry.invoice.paidAt = at;
      return true;
    },

    async invoicedSourceIds(tenantId: string, kind: string, ids: readonly string[]): Promise<string[]> {
      const wanted = new Set(ids);
      const taken: string[] = [];
      for (const { invoice } of entries.values()) {
        const source = invoice.source;
        if (invoice.tenantId !== tenantId || invoice.kind !== "base" || !source) continue;
        if (source.kind === kind && wanted.has(source.id)) taken.push(source.id);
      }
      return taken;
    },

    async setEInvoice(tenantId: string, id: string, patch: Partial<EInvoiceState>): Promise<void> {
      const entry = ofTenant(tenantId, id);
      if (entry) entry.invoice.eInvoice = { ...entry.invoice.eInvoice, ...patch };
    },
  };
}
```

## Why the shape is what it is

- **Both parties are snapshots.** The buyer is copied from the form and the seller from the tenant profile at
  issue. The PDF, the register and the e-invoice all read the snapshot, so editing the company address next
  year does not rewrite last year's documents, and the bytes hashed for a verification code stay the bytes
  that are sent.
- **Status is derived from two facts.** `paidAt` and `correctedByInvoiceId` are stored; `issued`, `paid` and
  `corrected` are computed by `invoiceStatus`. A single stored status cannot say "paid and corrected", and
  the one that is overwritten is lost.
- **One correction per base document.** `correctedByInvoiceId` is a single link and the schema makes it
  unique. A correction to a correction is a design in [operations.md](operations.md), not a shipped path.
- **The source is a pair, not a column per table.** `source.kind` and `source.id` let a base document bill
  a sale, an order or a booking without a foreign key per origin, and one unique index over the pair holds
  the rule that a source is invoiced once.
- **The cursor is opaque.** The in-memory store uses an offset and the Postgres store a keyset. Callers pass
  back what they were given and never build one.
- **Amounts are two-decimal numbers.** Every amount passes through `roundMoney` before it is stored, and the
  schema stores `numeric(12,2)`. The tests hold the sums; integer minor units would also work and are not
  what the templates use.

## Checklist

- [ ] Every query takes `tenantId` from the server-side actor, never from the request body
- [ ] `create` is one transaction: number, document, lines and the link on the original
- [ ] No code path updates a line, a total, a party or a number after `create`
- [ ] `list` filters in the store and returns a cursor; the UI never filters a truncated page
