# Testing the service and the database

The service suite and the database checks. The model and PDF suites, the file table and the run command
are in [testing.md](testing.md).

## The service

```typescript
// lib/invoicing/service.test.ts
import { describe, expect, it } from "vitest";
import { createMemoryInvoiceStore } from "./memory-store";
import { InvoiceError, type InvoiceErrorCode, type InvoiceLineInput } from "./model";
import {
  createInvoicing,
  type EInvoicePort,
  type InvoiceActor,
  type InvoiceAuditEvent,
  type InvoicingDeps,
  type SellerProfile,
} from "./service";
import { invoiceStatus } from "./store";

const actor: InvoiceActor = { tenantId: "t1", userId: "u1", canRead: true, canWrite: true };
const reader: InvoiceActor = { ...actor, canWrite: false };
const stranger: InvoiceActor = { ...actor, tenantId: "t2" };
const line: InvoiceLineInput = { name: "Service", qty: 1, unit: "szt.", unitPrice: 123, vatRate: "23" };
const buyer = { name: "Buyer sp. z o.o.", taxId: "525-224-84-81", address: "ul. Prosta 2" };

function setup(over: Partial<InvoicingDeps> = {}) {
  const seller: SellerProfile = {
    name: "Seller sp. z o.o.",
    taxId: "526-587-76-35",
    address: "ul. Kwiatowa 1",
    bankAccount: "PL61 1090 1014 0000 0712 1981 2874",
    paymentTermsDays: 14,
    vatExemptionBasis: null,
    currency: "PLN",
  };
  const events: InvoiceAuditEvent[] = [];
  const errors: string[] = [];
  const store = createMemoryInvoiceStore();
  const invoicing = createInvoicing({
    store,
    loadSeller: async () => seller,
    today: () => "2026-07-15",
    audit: async (event) => {
      events.push(event);
    },
    onError: (context) => {
      errors.push(context);
    },
    ...over,
  });
  return { invoicing, store, seller, events, errors };
}

async function codeOf(run: Promise<unknown>): Promise<InvoiceErrorCode | null> {
  try {
    await run;
    return null;
  } catch (error) {
    if (error instanceof InvoiceError) return error.code;
    throw error;
  }
}

describe("issue", () => {
  it("numbers consecutively per series and year, and takes the year from the issue date", async () => {
    const { invoicing } = setup();
    const a = await invoicing.issue(actor, { buyer, lines: [line] });
    const b = await invoicing.issue(actor, { buyer, lines: [line] });
    const c = await invoicing.issue(actor, { buyer, lines: [line], issueDate: "2027-01-02" });
    expect([a.number, b.number, c.number]).toEqual(["FV/2026/00001", "FV/2026/00002", "FV/2027/00001"]);
    expect(a).toMatchObject({ dueDate: "2026-07-29", totals: { net: 100, vat: 23, gross: 123 } });
  });

  it("stores the tax identifiers as bare digits and rejects a bad checksum", async () => {
    const { invoicing } = setup();
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    expect(issued.buyer.taxId).toBe("5252248481");
    expect(issued.seller.taxId).toBe("5265877635");
    expect(await codeOf(invoicing.issue(actor, { buyer: { ...buyer, taxId: "5252248480" }, lines: [line] }))).toBe("tax_id_invalid");
  });

  it("takes no number for a refused document", async () => {
    const { invoicing } = setup();
    expect(await codeOf(invoicing.issue(actor, { buyer: { name: " " }, lines: [line] }))).toBe("buyer_name_required");
    expect(await codeOf(invoicing.issue(actor, { buyer, lines: [] }))).toBe("no_lines");
    expect(await codeOf(invoicing.issue(actor, { buyer, lines: [{ ...line, qty: 0 }] }))).toBe("line_qty_invalid");
    expect(await codeOf(invoicing.issue(actor, { buyer, lines: [line], issueDate: "2026-02-30" }))).toBe("date_invalid");
    expect((await invoicing.issue(actor, { buyer, lines: [line] })).number).toBe("FV/2026/00001");
  });

  it("bills a source once, and the refusal leaves the series continuous", async () => {
    const { invoicing } = setup();
    const source = { kind: "sale", id: "s1" };
    await invoicing.issue(actor, { buyer, lines: [line], source });
    expect(await codeOf(invoicing.issue(actor, { buyer, lines: [line], source }))).toBe("source_already_invoiced");
    expect((await invoicing.issue(actor, { buyer, lines: [line] })).number).toBe("FV/2026/00002");
  });

  it("freezes the seller: a later profile change does not reach an issued document", async () => {
    const { invoicing, seller } = setup();
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    seller.name = "Renamed sp. z o.o.";
    seller.address = "ul. Nowa 9";
    const pdf = await invoicing.pdfData(actor, issued.id);
    expect(pdf?.seller).toMatchObject({ name: "Seller sp. z o.o.", address: "ul. Kwiatowa 1" });
  });

  it("requires the exemption basis for an exempt line and copies it onto the document", async () => {
    const { invoicing, seller } = setup();
    const exempt = [{ ...line, vatRate: "zw" as const }];
    expect(await codeOf(invoicing.issue(actor, { buyer, lines: exempt }))).toBe("exemption_basis_required");
    seller.vatExemptionBasis = "art. 43 ust. 1 pkt 19";
    const issued = await invoicing.issue(actor, { buyer, lines: exempt });
    seller.vatExemptionBasis = "changed later";
    expect((await invoicing.get(actor, issued.id))?.vatExemptionBasis).toBe("art. 43 ust. 1 pkt 19");
  });

  it("refuses a seller with no name", async () => {
    const { invoicing } = setup({ loadSeller: async () => null });
    expect(await codeOf(invoicing.issue(actor, { buyer, lines: [line] }))).toBe("seller_incomplete");
  });
});

describe("permissions and tenant scope", () => {
  it("needs write to mutate and read to see", async () => {
    const { invoicing } = setup();
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    expect(await codeOf(invoicing.issue(reader, { buyer, lines: [line] }))).toBe("forbidden");
    expect(await codeOf(invoicing.markPaid(reader, issued.id))).toBe("forbidden");
    expect(await codeOf(invoicing.list({ ...actor, canRead: false }))).toBe("forbidden");
    expect((await invoicing.list(reader)).rows).toHaveLength(1);
  });

  it("shows another tenant nothing, and lets it correct nothing", async () => {
    const { invoicing } = setup();
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    expect(await invoicing.get(stranger, issued.id)).toBeNull();
    expect(await invoicing.pdfData(stranger, issued.id)).toBeNull();
    expect((await invoicing.list(stranger)).rows).toHaveLength(0);
    expect(await codeOf(invoicing.correct(stranger, { originalId: issued.id, correctedLines: [], reason: "x" }))).toBe("not_found");
    expect((await invoicing.issue(stranger, { buyer, lines: [line] })).number).toBe("FV/2026/00001");
  });
});

describe("correct", () => {
  it("derives signed deltas, numbers in its own series and links the original", async () => {
    const { invoicing } = setup();
    const original = await invoicing.issue(actor, { buyer, lines: [line, { ...line, name: "Paint", unitPrice: 54, vatRate: "8" }] });
    const correction = await invoicing.correct(actor, {
      originalId: original.id,
      correctedLines: [{ ...line, unitPrice: 100 }, { ...line, name: "Paint", unitPrice: 54, vatRate: "8" }],
      reason: " Wrong price ",
    });
    expect(correction).toMatchObject({ number: "FK/2026/00001", kind: "correction", correctionReason: "Wrong price", dueDate: null });
    expect(correction.lines.map((l) => [l.qty, l.unitPrice])).toEqual([[-1, 123], [1, 100]]);
    expect(correction.totals.gross).toBe(-23);
    expect(correction.buyer).toEqual(original.buyer);
    expect((await invoicing.get(actor, original.id))?.correctedByInvoiceId).toBe(correction.id);
  });

  it("keeps a paid invoice paid when it is corrected", async () => {
    const { invoicing } = setup();
    const original = await invoicing.issue(actor, { buyer, lines: [line] });
    await invoicing.markPaid(actor, original.id);
    await invoicing.correct(actor, { originalId: original.id, correctedLines: [], reason: "Withdrawn" });
    const after = await invoicing.get(actor, original.id);
    expect(after?.paidAt).not.toBeNull();
    expect(after && invoiceStatus(after)).toBe("corrected");
  });

  it("allows one correction, of a base document, that changes something, with a reason", async () => {
    const { invoicing } = setup();
    const original = await invoicing.issue(actor, { buyer, lines: [line] });
    const input = { originalId: original.id, correctedLines: [{ ...line, unitPrice: 100 }], reason: "Wrong price" };
    expect(await codeOf(invoicing.correct(actor, { ...input, reason: " " }))).toBe("correction_reason_required");
    expect(await codeOf(invoicing.correct(actor, { ...input, correctedLines: [line] }))).toBe("correction_empty");
    const correction = await invoicing.correct(actor, input);
    expect(await codeOf(invoicing.correct(actor, input))).toBe("already_corrected");
    expect(await codeOf(invoicing.correct(actor, { ...input, originalId: correction.id }))).toBe("not_correctable");
    // The three refusals took no number from the correction series.
    const second = await invoicing.issue(actor, { buyer, lines: [line] });
    expect((await invoicing.correct(actor, { ...input, originalId: second.id })).number).toBe("FK/2026/00002");
  });

  it("refuses two concurrent corrections of one document in the store itself", async () => {
    const { invoicing } = setup();
    const original = await invoicing.issue(actor, { buyer, lines: [line] });
    const input = { originalId: original.id, correctedLines: [{ ...line, unitPrice: 100 }], reason: "Wrong price" };
    const results = await Promise.allSettled([invoicing.correct(actor, input), invoicing.correct(actor, input)]);
    expect(results.filter((r) => r.status === "fulfilled")).toHaveLength(1);
  });
});

describe("markPaid", () => {
  it("sets the payment once", async () => {
    const { invoicing } = setup({ now: () => "2026-07-16T09:00:00.000Z" });
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    await invoicing.markPaid(actor, issued.id);
    expect((await invoicing.get(actor, issued.id))?.paidAt).toBe("2026-07-16T09:00:00.000Z");
    expect(await codeOf(invoicing.markPaid(actor, issued.id))).toBe("already_paid");
    expect(await codeOf(invoicing.markPaid(actor, "missing"))).toBe("not_found");
  });
});

describe("list", () => {
  it("filters in the store and pages by cursor without losing or repeating a row", async () => {
    const { invoicing } = setup();
    for (let i = 0; i < 7; i++) {
      await invoicing.issue(actor, { buyer: { name: i % 2 ? "Odd Ltd" : "Even Ltd" }, lines: [line], issueDate: `2026-07-${String(10 + i)}` });
    }
    const seen: string[] = [];
    let cursor: string | null = null;
    do {
      const page = await invoicing.list(actor, {}, { limit: 3, cursor });
      seen.push(...page.rows.map((r) => r.number));
      cursor = page.nextCursor;
    } while (cursor);
    expect(seen).toHaveLength(7);
    expect(new Set(seen).size).toBe(7);
    expect(seen[0]).toBe("FV/2026/00007");

    expect((await invoicing.list(actor, { query: "odd" })).rows).toHaveLength(3);
    expect((await invoicing.list(actor, { from: "2026-07-14", to: "2026-07-15" })).rows).toHaveLength(2);
    expect((await invoicing.list(actor, { status: "paid" })).rows).toHaveLength(0);
  });
});

describe("audit and what happens after the document exists", () => {
  it("records every mutation and every download", async () => {
    const { invoicing, events } = setup();
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    await invoicing.markPaid(actor, issued.id);
    await invoicing.pdfData(actor, issued.id);
    await invoicing.correct(actor, { originalId: issued.id, correctedLines: [], reason: "Withdrawn" });
    expect(events.map((e) => e.action)).toEqual(["invoice.issue", "invoice.paid", "invoice.pdf", "invoice.correct"]);
    expect(events.every((e) => e.tenantId === "t1" && e.userId === "u1")).toBe(true);
  });

  it("returns the invoice when the audit fails after issue, and reports the failure", async () => {
    const { invoicing, errors } = setup({
      audit: async () => {
        throw new Error("audit down");
      },
    });
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    expect(issued.number).toBe("FV/2026/00001");
    expect(errors).toHaveLength(1);
  });

  it("stops a download that cannot be logged", async () => {
    let fail = false;
    const { invoicing } = setup({
      audit: async () => {
        if (fail) throw new Error("audit down");
      },
    });
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    fail = true;
    await expect(invoicing.pdfData(actor, issued.id)).rejects.toThrow("audit down");
  });

  it("marks a document not_required with no e-invoice port, and prints no code", async () => {
    const { invoicing } = setup();
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    expect(issued.eInvoice.status).toBe("not_required");
    expect((await invoicing.pdfData(actor, issued.id))?.verification).toBeNull();
  });

  it("keeps the invoice and records the error when the e-invoice hand-off fails", async () => {
    const port: EInvoicePort = {
      isEnabled: async () => true,
      preflight: () => {},
      onIssued: async () => {
        throw new Error("transport down");
      },
      verification: async () => null,
    };
    const { invoicing, errors } = setup({ eInvoice: port });
    const issued = await invoicing.issue(actor, { buyer, lines: [line] });
    const stored = await invoicing.get(actor, issued.id);
    expect(stored?.eInvoice).toMatchObject({ status: "pending", error: "transport down" });
    expect(errors).toHaveLength(1);
  });

  it("runs the e-invoice preflight before a number is taken", async () => {
    const port: EInvoicePort = {
      isEnabled: async () => true,
      preflight: () => {
        throw new InvoiceError("e_invoice_unsendable", { reason: "rate" });
      },
      onIssued: async () => {},
      verification: async () => null,
    };
    const blocked = setup({ eInvoice: port });
    expect(await codeOf(blocked.invoicing.issue(actor, { buyer, lines: [line] }))).toBe("e_invoice_unsendable");
    expect((await blocked.store.list("t1", {}, { limit: 10 })).rows).toHaveLength(0);
  });
});
```

## Database checks

The in-memory store proves the service. These statements prove the schema in
[postgres.md](postgres.md): they run inside one transaction that is rolled back, so they leave nothing
behind, and any failed assertion stops the script.

```sql
-- db/checks/invoicing-checks.sql
\set ON_ERROR_STOP on
begin;

create function pg_temp.draft(p_over jsonb default '{}') returns jsonb language sql as $$
  select jsonb_build_object(
    'tenantId', 't-check', 'kind', 'base', 'series', 'FV', 'year', 2031,
    'issueDate', '2031-03-01', 'saleDate', '2031-03-01', 'dueDate', '2031-03-15',
    'paymentMethod', 'transfer', 'currency', 'PLN',
    'seller', jsonb_build_object('name', 'Seller', 'taxId', '5265877635', 'address', 'Street 1', 'bankAccount', null),
    'buyer', jsonb_build_object('name', 'Buyer', 'taxId', null, 'address', null),
    'customerId', null, 'source', null,
    'totals', jsonb_build_object('net', 100, 'vat', 23, 'gross', 123),
    'vatExemptionBasis', null, 'correctsInvoiceId', null, 'correctionReason', null,
    'eInvoiceStatus', 'not_required', 'createdBy', 'u-check',
    'lines', jsonb_build_array(jsonb_build_object(
      'name', 'Service', 'qty', 1, 'unit', 'szt.', 'unitPrice', 123, 'vatRate', '23', 'net', 100, 'vat', 23, 'gross', 123))
  ) || p_over
$$;

do $$
declare
  a uuid; b uuid; c uuid; k uuid;
  n text;
  sold jsonb := pg_temp.draft('{"source": {"kind": "sale", "id": "s1"}}');
begin
  -- Consecutive numbers, and the lines arrive with the document.
  a := invoicing_create(pg_temp.draft());
  b := invoicing_create(pg_temp.draft());
  select number into n from invoices where id = b;
  assert n = 'FV/2031/00002', 'second number is ' || n;
  assert (select count(*) from invoice_lines where invoice_id = a) = 1, 'lines missing';

  -- A refused insert gives its number back.
  perform invoicing_create(sold);
  begin
    perform invoicing_create(sold);
    assert false, 'a source was invoiced twice';
  exception when others then
    assert sqlerrm like '%invoicing:source_already_invoiced%', sqlerrm;
  end;
  c := invoicing_create(pg_temp.draft());
  select number into n from invoices where id = c;
  assert n = 'FV/2031/00004', 'the refused insert consumed a number: ' || n;

  -- A sequence past 99999 keeps every digit.
  update invoice_number_counters set last_seq = 21921240 where tenant_id = 't-check' and series = 'FV' and year = 2031;
  c := invoicing_create(pg_temp.draft());
  select number into n from invoices where id = c;
  assert n = 'FV/2031/21921241', 'wide number is ' || n;

  -- An issued document does not change, and nothing is deleted.
  begin
    update invoices set total_gross = 1 where id = a;
    assert false, 'a total was edited';
  exception when others then assert sqlerrm like '%invoicing:immutable%', sqlerrm;
  end;
  begin
    update invoice_lines set name = 'x' where invoice_id = a;
    assert false, 'a line was edited';
  exception when others then assert sqlerrm like '%invoicing:immutable%', sqlerrm;
  end;
  begin
    delete from invoices where id = a;
    assert false, 'an invoice was deleted';
  exception when others then assert sqlerrm like '%invoicing:immutable%', sqlerrm;
  end;

  -- Payment is set once.
  update invoices set paid_at = now() where id = a;
  begin
    update invoices set paid_at = null where id = a;
    assert false, 'a payment was undone';
  exception when others then assert sqlerrm like '%invoicing:immutable%', sqlerrm;
  end;

  -- One correction, of a base document, in its own series, and the original stays paid.
  k := invoicing_create(pg_temp.draft(jsonb_build_object(
    'kind', 'correction', 'series', 'FK', 'correctsInvoiceId', a, 'correctionReason', 'wrong price')));
  assert (select number from invoices where id = k) = 'FK/2031/00001', 'correction series';
  assert (select corrected_by_invoice_id from invoices where id = a) = k, 'original not linked';
  assert (select paid_at from invoices where id = a) is not null, 'payment lost on correction';
  begin
    perform invoicing_create(pg_temp.draft(jsonb_build_object(
      'kind', 'correction', 'series', 'FK', 'correctsInvoiceId', a, 'correctionReason', 'again')));
    assert false, 'a second correction was accepted';
  exception when others then assert sqlerrm like '%invoicing:already_corrected%', sqlerrm;
  end;
  begin
    perform invoicing_create(pg_temp.draft(jsonb_build_object(
      'kind', 'correction', 'series', 'FK', 'correctsInvoiceId', k, 'correctionReason', 'nested')));
    assert false, 'a correction was corrected';
  exception when others then assert sqlerrm like '%invoicing:not_correctable%', sqlerrm;
  end;
  begin
    perform invoicing_create(pg_temp.draft(jsonb_build_object(
      'tenantId', 't-other', 'kind', 'correction', 'series', 'FK', 'correctsInvoiceId', b, 'correctionReason', 'x')));
    assert false, 'another tenant corrected this document';
  exception when others then assert sqlerrm like '%invoicing:not_found%', sqlerrm;
  end;

  -- Erasing a tenant is possible only with the switch, inside the transaction that sets it.
  perform set_config('invoicing.allow_delete', 'on', true);
  delete from invoices where tenant_id = 't-check';
  assert (select count(*) from invoice_lines where tenant_id = 't-check') = 0, 'lines survived erasure';
end;
$$;

rollback;
\echo invoicing checks passed
```

```bash
psql "$DATABASE_URL" -f db/checks/invoicing-checks.sql
```

Concurrency needs two sessions and so cannot live in that file. Issue one base document, then start two
corrections of it at once: the row lock in `invoicing_create` queues the second, which then finds the link
already set. Exactly one succeeds, and the correction series has no gap. The same holds for parallel
`issue` calls: twenty at once produce twenty consecutive numbers.
