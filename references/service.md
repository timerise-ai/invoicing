# The service: issue, correct, mark paid, list, render data

`createInvoicing(deps)` is the one place the rules run. It takes its collaborators as arguments, so the same
code is driven by the in-memory store in the tests and by Postgres in the app, and nothing in it reaches for
a session, a clock or a database on its own.

| Method | Permission | What it guarantees |
|---|---|---|
| `issue(actor, input)` | write | Input validated, seller snapshotted, document and lines written in one transaction, e-invoice hand-off attempted, audit written |
| `correct(actor, input)` | write | Deltas derived on the server from the stored lines, one correction per base document, the original linked in the same transaction |
| `markPaid(actor, id)` | write | Sets `paidAt` once; a second call is `already_paid` |
| `list(actor, filters, page)` | read | Filtered in the store, paged by cursor, limit clamped to 100 |
| `get(actor, id)` | read | The document with its lines, or null outside the tenant |
| `pdfData(actor, id)` | read | Render-ready data, after the download is audited |
| `pdfDataForDelivery(tenantId, id)` | none | The same data for a background mailer that has no actor; the caller audits the send |

## Dependencies

```typescript
// lib/invoicing/service.ts
import {
  INVOICE_SERIES,
  InvoiceError,
  buildCorrectionLines,
  computeLine,
  computeTotals,
  isIsoDate,
  paymentDueDate,
  validateLines,
  vatSummary,
  type InvoiceLine,
  type InvoiceLineInput,
} from "./model";
import type { InvoicePdfData, QrMatrix } from "./invoice-pdf";
import type {
  Invoice,
  InvoiceFilters,
  InvoicePage,
  InvoiceSourceRef,
  InvoiceStore,
  InvoiceWithLines,
  NewInvoice,
  SellerSnapshot,
} from "./store";
import { isValidNip, normalizeTaxId } from "./tax-id";

/** Who is acting. Built by the host from its own session, on the server, for every request. */
export type InvoiceActor = {
  tenantId: string;
  userId: string;
  canRead: boolean;
  canWrite: boolean;
};

/** The seller as the tenant profile holds it today. Copied onto each document at issue. */
export type SellerProfile = SellerSnapshot & {
  paymentTermsDays: number;
  /** The legal basis printed and sent when a line is exempt (`zw`). */
  vatExemptionBasis: string | null;
  currency: string;
};

export type InvoiceAuditEvent = {
  action: "invoice.issue" | "invoice.correct" | "invoice.paid" | "invoice.pdf";
  tenantId: string;
  userId: string;
  invoiceId: string;
  number: string;
  detail?: Record<string, string | number | null>;
};

export type Verification = { url: string; label: string; matrix: QrMatrix };

/**
 * The seam to an e-invoicing transport. Optional: without it every document is `not_required` and the PDF
 * carries no verification code. ksef-bridge.md implements it for KSeF.
 */
export interface EInvoicePort {
  isEnabled(tenantId: string): Promise<boolean>;
  /**
   * Runs before a number is reserved. Throws InvoiceError("e_invoice_unsendable") for a document the
   * transport could never accept, while the person who can fix it is still looking at the form.
   */
  preflight(draft: NewInvoice): void;
  /** Runs after the document exists. Idempotent on the invoice id. */
  onIssued(invoice: InvoiceWithLines): Promise<void>;
  /** The code for the PDF, or null when the document has none. */
  verification(invoice: Invoice): Promise<Verification | null>;
}

export type InvoicingDeps = {
  store: InvoiceStore;
  loadSeller(tenantId: string): Promise<SellerProfile | null>;
  /** Today in the business time zone, as YYYY-MM-DD. */
  today(): string;
  /** The current instant, ISO 8601. Defaults to the system clock. */
  now?: () => string;
  /** Defaults to the Polish NIP checksum. */
  validateTaxId?: (taxId: string) => boolean;
  /** Written for every mutation and every download. */
  audit?: (event: InvoiceAuditEvent) => Promise<void>;
  eInvoice?: EInvoicePort;
  /** The unit printed when a line leaves it blank. */
  defaultUnit?: string;
  /** Where a failure after the document exists is reported. Defaults to console.error. */
  onError?: (context: string, error: unknown) => void;
};

export type IssueInvoiceInput = {
  buyer: { name: string; taxId?: string | null; address?: string | null };
  lines: InvoiceLineInput[];
  issueDate?: string;
  saleDate?: string;
  paymentMethod?: string | null;
  customerId?: string | null;
  source?: InvoiceSourceRef | null;
};

export type CorrectionInput = {
  originalId: string;
  /** The full corrected state. The signed deltas are derived here, never sent by the client. */
  correctedLines: InvoiceLineInput[];
  reason: string;
};
```

## The service

```typescript
export function createInvoicing(deps: InvoicingDeps) {
  const { store } = deps;
  const validateTaxId = deps.validateTaxId ?? isValidNip;
  const defaultUnit = deps.defaultUnit ?? "szt.";
  const now = deps.now ?? (() => new Date().toISOString());
  const report = deps.onError ?? ((context: string, error: unknown) => console.error(`invoicing: ${context}`, error));

  function requireRead(actor: InvoiceActor): void {
    if (!actor.canRead) throw new InvoiceError("forbidden");
  }

  function requireWrite(actor: InvoiceActor): void {
    if (!actor.canWrite) throw new InvoiceError("forbidden");
  }

  async function requireSeller(tenantId: string): Promise<SellerProfile> {
    const seller = await deps.loadSeller(tenantId);
    if (!seller || !seller.name.trim()) throw new InvoiceError("seller_incomplete");
    return seller;
  }

  function normalizeLine(line: InvoiceLineInput): InvoiceLineInput {
    return { ...line, name: line.name.trim(), unit: line.unit.trim() || defaultUnit };
  }

  function snapshot(seller: SellerProfile): SellerSnapshot {
    return {
      name: seller.name.trim(),
      taxId: seller.taxId ? normalizeTaxId(seller.taxId) : null,
      address: seller.address,
      bankAccount: seller.bankAccount,
    };
  }

  /**
   * Everything that happens after the document exists. None of it may throw to the caller: the invoice is
   * numbered and stored, and an error here would send the operator back to a form that issues a second one.
   */
  async function afterCreate(
    actor: InvoiceActor,
    invoice: InvoiceWithLines,
    action: "invoice.issue" | "invoice.correct",
  ): Promise<void> {
    if (deps.eInvoice && invoice.eInvoice.status === "pending") {
      try {
        await deps.eInvoice.onIssued(invoice);
      } catch (error) {
        report(`e-invoice hand-off for ${invoice.number}`, error);
        const message = error instanceof Error ? error.message : String(error);
        await store.setEInvoice(invoice.tenantId, invoice.id, { error: message }).catch((e) => report("setEInvoice", e));
      }
    }
    try {
      await deps.audit?.({
        action,
        tenantId: actor.tenantId,
        userId: actor.userId,
        invoiceId: invoice.id,
        number: invoice.number,
        detail: { gross: invoice.totals.gross, corrects: invoice.correctsInvoiceId },
      });
    } catch (error) {
      report(`audit for ${invoice.number}`, error);
    }
  }

  async function eInvoiceStatusFor(tenantId: string): Promise<"pending" | "not_required"> {
    return deps.eInvoice && (await deps.eInvoice.isEnabled(tenantId)) ? "pending" : "not_required";
  }

  async function issue(actor: InvoiceActor, input: IssueInvoiceInput): Promise<InvoiceWithLines> {
    requireWrite(actor);

    const buyerName = input.buyer.name.trim();
    if (!buyerName) throw new InvoiceError("buyer_name_required");
    // Stored as bare digits whatever was typed, and checked here as well as at the form.
    const buyerTaxId = normalizeTaxId(input.buyer.taxId ?? "") || null;
    if (buyerTaxId && !validateTaxId(buyerTaxId)) throw new InvoiceError("tax_id_invalid");

    const inputLines = input.lines.map(normalizeLine);
    validateLines(inputLines, "base");

    const issueDate = input.issueDate ?? deps.today();
    const saleDate = input.saleDate ?? issueDate;
    if (!isIsoDate(issueDate) || !isIsoDate(saleDate)) throw new InvoiceError("date_invalid");

    const seller = await requireSeller(actor.tenantId);
    const lines = inputLines.map(computeLine);

    // An exempt line needs its legal basis on the document. The basis is copied from the seller profile
    // now, so a later settings change does not rewrite an issued invoice.
    const basis = seller.vatExemptionBasis?.trim() || null;
    const hasExempt = lines.some((l) => l.vatRate === "zw");
    if (hasExempt && !basis) throw new InvoiceError("exemption_basis_required");

    const draft: NewInvoice = {
      tenantId: actor.tenantId,
      kind: "base",
      series: INVOICE_SERIES.base,
      year: Number(issueDate.slice(0, 4)),
      issueDate,
      saleDate,
      dueDate: paymentDueDate(issueDate, seller.paymentTermsDays),
      paymentMethod: input.paymentMethod || null,
      currency: seller.currency,
      seller: snapshot(seller),
      buyer: { name: buyerName, taxId: buyerTaxId, address: input.buyer.address?.trim() || null },
      customerId: input.customerId ?? null,
      source: input.source ?? null,
      totals: computeTotals(lines),
      vatExemptionBasis: hasExempt ? basis : null,
      correctsInvoiceId: null,
      correctionReason: null,
      eInvoiceStatus: await eInvoiceStatusFor(actor.tenantId),
      createdBy: actor.userId,
      lines,
    };
    if (draft.eInvoiceStatus === "pending") deps.eInvoice?.preflight(draft);

    const invoice = await store.create(draft);
    await afterCreate(actor, invoice, "invoice.issue");
    return invoice;
  }

  async function correct(actor: InvoiceActor, input: CorrectionInput): Promise<InvoiceWithLines> {
    requireWrite(actor);
    const reason = input.reason.trim();
    if (!reason) throw new InvoiceError("correction_reason_required");

    const original = await store.get(actor.tenantId, input.originalId);
    if (!original) throw new InvoiceError("not_found");
    if (original.kind !== "base") throw new InvoiceError("not_correctable");
    if (original.correctedByInvoiceId) throw new InvoiceError("already_corrected");

    // The corrected state may be empty (everything is withdrawn); the lines it does have obey base rules.
    const corrected = input.correctedLines.map(normalizeLine);
    if (corrected.length > 0) validateLines(corrected, "base");

    const before: InvoiceLineInput[] = original.lines.map((l) => ({
      name: l.name,
      qty: l.qty,
      unit: l.unit,
      unitPrice: l.unitPrice,
      vatRate: l.vatRate,
    }));
    const deltas = buildCorrectionLines(before, corrected);
    validateLines(deltas, "correction"); // throws correction_empty when nothing changed

    const lines = deltas.map(computeLine);
    const totals = computeTotals(lines);
    const seller = await requireSeller(actor.tenantId);
    const issueDate = deps.today();

    const draft: NewInvoice = {
      tenantId: actor.tenantId,
      kind: "correction",
      series: INVOICE_SERIES.correction,
      year: Number(issueDate.slice(0, 4)),
      issueDate,
      saleDate: original.saleDate,
      // Only a correction that raises the amount has something to pay by a date.
      dueDate: totals.gross > 0 ? paymentDueDate(issueDate, seller.paymentTermsDays) : null,
      paymentMethod: original.paymentMethod,
      currency: original.currency,
      seller: snapshot(seller),
      buyer: original.buyer,
      customerId: original.customerId,
      source: null,
      totals,
      // A correction adjusts the same supply, so the same exemption basis applies to it.
      vatExemptionBasis: original.vatExemptionBasis ?? (seller.vatExemptionBasis?.trim() || null),
      correctsInvoiceId: original.id,
      correctionReason: reason,
      eInvoiceStatus: await eInvoiceStatusFor(actor.tenantId),
      createdBy: actor.userId,
      lines,
    };
    if (lines.some((l) => l.vatRate === "zw") && !draft.vatExemptionBasis) {
      throw new InvoiceError("exemption_basis_required");
    }
    if (draft.eInvoiceStatus === "pending") deps.eInvoice?.preflight(draft);

    // The store repeats the two correction checks under a row lock, so two concurrent corrections of the
    // same document cannot both succeed.
    const invoice = await store.create(draft);
    await afterCreate(actor, invoice, "invoice.correct");
    return invoice;
  }

  async function markPaid(actor: InvoiceActor, id: string): Promise<void> {
    requireWrite(actor);
    const invoice = await store.get(actor.tenantId, id);
    if (!invoice) throw new InvoiceError("not_found");
    if (!(await store.markPaid(actor.tenantId, id, now()))) throw new InvoiceError("already_paid");
    try {
      await deps.audit?.({
        action: "invoice.paid",
        tenantId: actor.tenantId,
        userId: actor.userId,
        invoiceId: id,
        number: invoice.number,
      });
    } catch (error) {
      report(`audit for ${invoice.number}`, error);
    }
  }

  async function list(
    actor: InvoiceActor,
    filters: InvoiceFilters = {},
    page: { limit?: number; cursor?: string | null } = {},
  ): Promise<InvoicePage> {
    requireRead(actor);
    const limit = Math.min(100, Math.max(1, Math.trunc(page.limit ?? 50)));
    return store.list(actor.tenantId, filters, { limit, cursor: page.cursor ?? null });
  }

  async function get(actor: InvoiceActor, id: string): Promise<InvoiceWithLines | null> {
    requireRead(actor);
    return store.get(actor.tenantId, id);
  }

  async function toPdfData(invoice: InvoiceWithLines): Promise<InvoicePdfData> {
    const original = invoice.correctsInvoiceId ? await store.get(invoice.tenantId, invoice.correctsInvoiceId) : null;
    const lines: InvoiceLine[] = invoice.lines;
    const verification = (await deps.eInvoice?.verification(invoice)) ?? null;
    return {
      number: invoice.number,
      isCorrection: invoice.kind === "correction",
      correctsNumber: original?.number ?? null,
      correctionReason: invoice.correctionReason,
      issueDate: invoice.issueDate,
      saleDate: invoice.saleDate,
      dueDate: invoice.dueDate,
      paymentMethod: invoice.paymentMethod,
      currency: invoice.currency,
      seller: invoice.seller,
      buyer: invoice.buyer,
      lines: invoice.lines,
      totals: invoice.totals,
      summary: vatSummary(lines),
      vatExemptionBasis: invoice.vatExemptionBasis,
      eInvoiceNumber: invoice.eInvoice.number,
      verification: verification ? { matrix: verification.matrix, label: verification.label } : null,
    };
  }

  /**
   * Render-ready data for a download. The audit event is written first and its failure propagates: a
   * document that leaves the server without a record of who took it is the case the log exists for.
   */
  async function pdfData(actor: InvoiceActor, id: string): Promise<InvoicePdfData | null> {
    requireRead(actor);
    const invoice = await store.get(actor.tenantId, id);
    if (!invoice) return null;
    await deps.audit?.({
      action: "invoice.pdf",
      tenantId: actor.tenantId,
      userId: actor.userId,
      invoiceId: id,
      number: invoice.number,
    });
    return toPdfData(invoice);
  }

  /** For a background sender. `tenantId` comes from the queued job, never from a request. */
  async function pdfDataForDelivery(tenantId: string, id: string): Promise<InvoicePdfData | null> {
    const invoice = await store.get(tenantId, id);
    return invoice ? toPdfData(invoice) : null;
  }

  return { issue, correct, markPaid, list, get, pdfData, pdfDataForDelivery };
}

export type Invoicing = ReturnType<typeof createInvoicing>;
```

## Wiring

One file joins the service to the host: the store, the seller profile, the clock and the actor. This is the
file the host edits; everything else in `lib/invoicing` is copied as written. The template below runs with
no database and no auth, which is enough for a first build and for nothing else: the store forgets on
restart and `getInvoiceActor` fails closed until the host's session lookup replaces its body.

```typescript
// lib/invoicing/index.ts
import "server-only";
import { createMemoryInvoiceStore } from "./memory-store";
import { createInvoicing, type InvoiceActor, type SellerProfile } from "./service";
import type { InvoiceStore } from "./store";

// One store per process. Without the global, a dev server's module reload would hand each request a
// fresh, empty store. Replace with createPostgresInvoiceStore(...) from postgres.md.
const globalForInvoicing = globalThis as typeof globalThis & { __invoiceStore?: InvoiceStore };
const store: InvoiceStore = (globalForInvoicing.__invoiceStore ??= createMemoryInvoiceStore());

/**
 * Single-company default: the seller comes from the environment. A multi-tenant host reads its tenant row.
 * `||`, not `??`: an example file copied with empty values sets each variable to "", which must mean unset.
 */
async function loadSeller(_tenantId: string): Promise<SellerProfile | null> {
  const name = process.env.INVOICE_SELLER_NAME;
  if (!name) return null;
  return {
    name,
    taxId: process.env.INVOICE_SELLER_TAX_ID || null,
    address: process.env.INVOICE_SELLER_ADDRESS || null,
    bankAccount: process.env.INVOICE_SELLER_BANK_ACCOUNT || null,
    paymentTermsDays: Number(process.env.INVOICE_PAYMENT_TERMS_DAYS || 14),
    vatExemptionBasis: process.env.INVOICE_VAT_EXEMPTION_BASIS || null,
    currency: process.env.INVOICE_CURRENCY || "PLN",
  };
}

/** Today in the business time zone. The server's own zone is UTC on most hosts, which is a day off at night. */
function today(): string {
  const timeZone = process.env.INVOICE_TIME_ZONE || "Europe/Warsaw";
  return new Intl.DateTimeFormat("en-CA", { timeZone }).format(new Date());
}

export const invoicing = createInvoicing({ store, loadSeller, today });

/**
 * The auth seam. Replace the body with the host's session lookup and return the tenant, the user and the
 * two permissions. Until then nobody is an actor, so every route answers 401.
 */
export async function getInvoiceActor(): Promise<InvoiceActor | null> {
  return null;
}
```

## Server Actions

Expected failures are return values, not thrown errors. The Next.js error-handling guide says to model
expected errors that way, and it matters here: the form needs the code to show the right message next to
the right field, and an action that throws gives it a rejected promise instead. `runInvoiceAction` turns an
`InvoiceError` into a result and lets everything else propagate to the error boundary.

```typescript
// app/invoices/actions.ts
"use server";

import { revalidatePath } from "next/cache";
import { getInvoiceActor, invoicing } from "@/lib/invoicing";
import { InvoiceError, type InvoiceErrorCode } from "@/lib/invoicing/model";
import type { CorrectionInput, InvoiceActor, IssueInvoiceInput } from "@/lib/invoicing/service";

export type InvoiceActionResult<T> =
  | { ok: true; value: T }
  | { ok: false; code: InvoiceErrorCode; params: Readonly<Record<string, string | number>> };

async function runInvoiceAction<T>(run: (actor: InvoiceActor) => Promise<T>): Promise<InvoiceActionResult<T>> {
  const actor = await getInvoiceActor();
  if (!actor) return { ok: false, code: "forbidden", params: {} };
  try {
    const value = await run(actor);
    revalidatePath("/invoices");
    return { ok: true, value };
  } catch (error) {
    if (error instanceof InvoiceError) return { ok: false, code: error.code, params: error.params };
    throw error;
  }
}

export async function issueInvoiceAction(
  input: IssueInvoiceInput,
): Promise<InvoiceActionResult<{ id: string; number: string }>> {
  return runInvoiceAction(async (actor) => {
    const invoice = await invoicing.issue(actor, input);
    return { id: invoice.id, number: invoice.number };
  });
}

export async function issueCorrectionAction(
  input: CorrectionInput,
): Promise<InvoiceActionResult<{ id: string; number: string }>> {
  return runInvoiceAction(async (actor) => {
    const invoice = await invoicing.correct(actor, input);
    return { id: invoice.id, number: invoice.number };
  });
}

export async function markInvoicePaidAction(id: string): Promise<InvoiceActionResult<null>> {
  return runInvoiceAction(async (actor) => {
    await invoicing.markPaid(actor, id);
    return null;
  });
}
```

A Server Action is a public endpoint: its arguments arrive from the network whatever the TypeScript
signature says. The service validates every field it stores, so the actions pass their input straight
through. A host that already validates with zod or valibot adds its schema at the top of each action and
keeps the service checks; they are not a substitute for each other.

## The rules the order encodes

1. **Validate, then load the seller, then reserve the number.** Every check that can fail runs before
   `store.create`. A number is taken only by a document that is going to exist.
2. **The e-invoice preflight runs before the number, the hand-off after it.** A rate the transport cannot
   express, or a seller with no tax identifier, is an error the person at the form can fix. A transport
   that is down is not, so it is recorded on the document and retried by the bridge.
3. **Nothing after `store.create` throws.** The document exists. A hand-off or audit failure is reported and
   stored, and the caller receives the invoice.
4. **The download audit is the exception.** It runs before any bytes are produced and its failure stops the
   download, because nothing has been created yet and a retry is harmless.
5. **The tenant is the actor's.** No method takes a tenant id from input. `pdfDataForDelivery` takes one
   from the queued job that the server itself wrote.

## Checklist

- [ ] `getInvoiceActor` reads the host session on the server and returns null when there is none
- [ ] `loadSeller` returns the tenant's own profile; a tenant with no name cannot issue
- [ ] `today` uses the business time zone, not the server's
- [ ] Actions return `InvoiceActionResult`; the form maps `code` to a message in the host's language
- [ ] `audit` is wired to the host's audit log, or its absence is a recorded decision
