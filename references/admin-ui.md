# The admin surface: register, issue form, correction editor

This file ships structure, state and behaviour. It ships no styling: buttons, dialogs, tables, pills,
toasts and every colour come from the host's own components, and the one component below is written in
bare HTML elements so that there is nothing to strip before the host's primitives go in.

| Surface | Purpose | Permission |
|---|---|---|
| Register | Every invoice, filtered on the server, with status, e-invoice state and the PDF | read |
| Issue form | A new base document, standalone or from something already sold | write |
| Correction editor | The original's lines edited into the corrected state, with a reason | write |

## The register

A Server Component reads `invoicing.list(actor, filters, page)` with the filters from the URL and renders
the rows. Filtering happens in the store. A register that loads one page and filters it in the browser
shows a search for an older invoice as "no results", which reads as "it does not exist".

| Column | Content | Notes |
|---|---|---|
| Number | `number`, and a marker when `kind` is `correction` | Links nowhere: the PDF is the detail view |
| Buyer | `buyer.name`, then `buyer.taxId` | |
| Dates | `issueDate`, then `dueDate` | |
| Net, VAT, Gross | `totals` | Right-aligned, tabular figures; net first, since it is the accountant's number |
| Status | `invoiceStatus(invoice)`, and a separate paid marker when a corrected invoice has `paidAt` | Derived, see [data-model.md](data-model.md) |
| E-invoice | `eInvoice.status`; `eInvoice.error` as the tooltip or an expandable row when present | Hidden entirely when no bridge is wired and every row is `not_required` |
| Actions | Download PDF for everyone with read; the rest behind write | |

Row actions and when each is offered:

| Action | Offered when | Calls |
|---|---|---|
| Download PDF | always | `GET /invoices/[id]/pdf`, opened in a new tab |
| Mark as paid | `paidAt` is null | `markInvoicePaidAction(id)`, behind the host's confirm dialog: it cannot be undone |
| Issue correction | `kind` is `base` and `correctedByInvoiceId` is null | opens the correction editor |
| Send by e-mail | the host has a mailer and the invoice has a recipient | the host's delivery action, see [operations.md](operations.md) |

Filters, all in the URL so that a reload and the back button return to the same view: free text (number,
buyer, tax identifier), status, e-invoice status, customer, and an issue-date range. Below the rows, a
"load more" control passes `nextCursor` back; it is absent when `nextCursor` is null.

States the register must have: empty with no filters (a call to action for a writer, a plain statement for
a read-only role, which cannot follow a call to action), empty with filters ("nothing matches"), and the
list.

## Form state

The line editor works on strings, because a field being typed into is not a number yet, and converts at
submit. A comma is accepted as the decimal separator, which is what a Polish keyboard produces.

```typescript
// lib/invoicing/draft.ts
import { VAT_RATES, isVatRate, type InvoiceErrorCode, type InvoiceLineInput, type VatRate } from "./model";

/** One editable line: raw form state. */
export type LineDraft = {
  name: string;
  qty: string;
  unit: string;
  unitPrice: string;
  vatRate: string;
};

export const EMPTY_LINE: LineDraft = { name: "", qty: "1", unit: "", unitPrice: "", vatRate: "23" };

function parseDecimal(value: string): number {
  const normalized = value.trim().replace(/\s/g, "").replace(",", ".");
  return normalized === "" ? Number.NaN : Number(normalized);
}

/**
 * A draft as a line. An unparseable number becomes NaN, not 0: the service then refuses the line by
 * position, instead of a typo quietly issuing a zero-priced line.
 */
export function draftToLine(draft: LineDraft): InvoiceLineInput {
  return {
    name: draft.name.trim(),
    qty: parseDecimal(draft.qty),
    unit: draft.unit.trim(),
    unitPrice: parseDecimal(draft.unitPrice),
    vatRate: (isVatRate(draft.vatRate) ? draft.vatRate : "") as VatRate,
  };
}

export function lineToDraft(line: InvoiceLineInput): LineDraft {
  return {
    name: line.name,
    qty: String(line.qty),
    unit: line.unit,
    unitPrice: String(line.unitPrice),
    vatRate: line.vatRate,
  };
}

/** Rows the user never touched are dropped at submit. A row with any content is kept and validated. */
export function isBlankDraft(draft: LineDraft): boolean {
  return draft.name.trim() === "" && draft.unitPrice.trim() === "";
}

export const VAT_RATE_VALUES: readonly VatRate[] = VAT_RATES;

/** The i18n key for an error code. The host's catalogue holds the sentences. */
export function invoiceErrorKey(code: InvoiceErrorCode): string {
  return `invoices.errors.${code}`;
}
```

## The line editor

Shared by the issue form and the correction editor. It owns no state: the parent holds the drafts, so the
parent can reset them when the dialog opens for a different document. The totals line under the grid uses
the same `computeLine` the server uses, so the preview and the issued document cannot disagree.

```tsx
// components/invoices/invoice-lines-editor.tsx
"use client";

import { EMPTY_LINE, VAT_RATE_VALUES, draftToLine, isBlankDraft, type LineDraft } from "@/lib/invoicing/draft";
import { computeLine, computeTotals, isVatRate, type InvoiceTotals } from "@/lib/invoicing/model";

export type LinesEditorLabels = {
  name: string;
  qty: string;
  unit: string;
  unitPrice: string;
  vatRate: string;
  addLine: string;
  removeLine: string;
  /** Formats the preview, for example "Net 100.00, VAT 23.00, Gross 123.00". */
  totals: (totals: InvoiceTotals) => string;
  /** The label for a rate, for example "23%" or the host's word for exempt. */
  rate: (rate: string) => string;
};

/** Totals of the lines that are complete enough to compute. Null while nothing is. */
export function previewTotals(lines: readonly LineDraft[]): InvoiceTotals | null {
  const ready = lines
    .filter((draft) => !isBlankDraft(draft))
    .map(draftToLine)
    .filter((l) => isVatRate(l.vatRate) && Number.isFinite(l.qty) && Number.isFinite(l.unitPrice));
  return ready.length > 0 ? computeTotals(ready.map(computeLine)) : null;
}

export function InvoiceLinesEditor({
  lines,
  onChange,
  labels,
  disabled = false,
}: {
  lines: LineDraft[];
  onChange: (next: LineDraft[]) => void;
  labels: LinesEditorLabels;
  disabled?: boolean;
}) {
  const patch = (index: number, part: Partial<LineDraft>): void =>
    onChange(lines.map((l, i) => (i === index ? { ...l, ...part } : l)));
  const totals = previewTotals(lines);

  return (
    <fieldset disabled={disabled}>
      <table>
        <thead>
          <tr>
            <th scope="col">{labels.name}</th>
            <th scope="col">{labels.qty}</th>
            <th scope="col">{labels.unit}</th>
            <th scope="col">{labels.unitPrice}</th>
            <th scope="col">{labels.vatRate}</th>
            <th />
          </tr>
        </thead>
        <tbody>
          {lines.map((l, i) => (
            <tr key={i}>
              <td>
                <input aria-label={labels.name} value={l.name} onChange={(e) => patch(i, { name: e.target.value })} />
              </td>
              <td>
                <input aria-label={labels.qty} inputMode="decimal" value={l.qty} onChange={(e) => patch(i, { qty: e.target.value })} />
              </td>
              <td>
                <input aria-label={labels.unit} value={l.unit} onChange={(e) => patch(i, { unit: e.target.value })} />
              </td>
              <td>
                <input aria-label={labels.unitPrice} inputMode="decimal" value={l.unitPrice} onChange={(e) => patch(i, { unitPrice: e.target.value })} />
              </td>
              <td>
                <select aria-label={labels.vatRate} value={l.vatRate} onChange={(e) => patch(i, { vatRate: e.target.value })}>
                  {VAT_RATE_VALUES.map((rate) => (
                    <option key={rate} value={rate}>
                      {labels.rate(rate)}
                    </option>
                  ))}
                </select>
              </td>
              <td>
                <button type="button" aria-label={labels.removeLine} onClick={() => onChange(lines.filter((_, x) => x !== i))}>
                  &times;
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
      <button type="button" onClick={() => onChange([...lines, { ...EMPTY_LINE }])}>
        {labels.addLine}
      </button>
      {totals ? <p aria-live="polite">{labels.totals(totals)}</p> : null}
    </fieldset>
  );
}
```

## The issue form

Fields, in order: the mode (standalone, or from a source), the source picker when the mode asks for one,
the customer (optional, fills the buyer), buyer name, buyer tax identifier (optional), buyer address
(optional), issue date (today in the business time zone), payment method, the line editor.

Behaviour:

- **Submitting calls `issueInvoiceAction`** with `lines.filter((d) => !isBlankDraft(d)).map(draftToLine)`.
  On `{ ok: false }` the message for `code` is shown inside the form and the form stays open with its
  contents. On success the dialog closes, the register refreshes, and a toast names the new number.
- **The submit control is disabled while the action runs**, and the dialog does not close on a backdrop
  click during that time. A second click must not reach the server: a standalone invoice has no source to
  make the second one a duplicate, so two clicks would be two documents.
- **Choosing a customer fills the buyer fields** from the customer's stored billing details and replaces
  what was typed. Choosing a source fills the lines and, where the source has a customer, the buyer, but
  only into fields that are still empty.
- **The form resets when it opens**, not when it closes, so a deep link that opens it with a preselected
  source behaves like any other open.

### Issuing from a source

A source is whatever the host already sold: a point-of-sale sale, an order, a completed booking. The
contract between the host and this module is one type and two rules.

```ts
// what the host's picker loader returns
type InvoiceableSource = {
  kind: string;          // the host's word: "sale", "order", "booking"
  id: string;
  occurredAt: string;    // becomes the sale date
  total: number;         // what was actually charged
  customerId: string | null;
  customerName: string | null;
  lines: InvoiceLineInput[];
};
```

1. **Offer only what is not invoiced.** Load the recent sources, pass their ids to
   `store.invoicedSourceIds(tenantId, kind, ids)`, and drop the ones it returns. The unique index is the
   backstop, not the filter: a picker that offers an invoiced source ends in an error the operator did
   nothing to deserve.
2. **Itemise only when the items reconcile.** If a source's item prices do not sum to what was charged
   (a discount applied to the total, a price corrected after the fact), invoicing the items would put a
   total on the document that contradicts the payment. Fall back to one line carrying the charged amount.

## The correction editor

It loads the original's lines (`invoicing.get`), shows them in the line editor, and takes a reason. The
operator edits the lines **into the state they should have had**; the server derives the signed deltas.
The editor never computes or sends a delta.

- The submit control stays disabled until the lines have loaded and the reason is not blank.
- Removing every line is allowed: it withdraws the document, and the correction is a full reversal.
- Deltas are positional, so removing the first line makes every later line show as changed. An editor
  that keeps a removed row in place, struck through, and sends the remaining rows in their original
  positions is not possible with the current delta rule; the amounts are right either way.
- Reset the editor when the target invoice changes, by comparing the target id during render rather than
  in an effect, and ignore a lines response that arrives after the target has changed.

## Strings

Every label, placeholder, status name and error sentence is a key in the host's i18n catalogue, in every
locale the host has. The module supplies the keys' meaning, not their text: `invoiceErrorKey(code)` for
the nineteen error codes, the three statuses, the four e-invoice states, and the labels of
`LinesEditorLabels`. A host with no i18n system keeps the sentences in one constants module next to the
components.

## Checklist

- [ ] The register filters and pages on the server; "load more" passes the cursor back
- [ ] A read-only role sees the register and the PDF and no mutating control
- [ ] Mark as paid goes through the host's confirm dialog
- [ ] Both forms show the error for the returned `code` and keep their contents
- [ ] The submit control cannot fire twice
- [ ] No class name, colour or spacing value from this file survives into the host
