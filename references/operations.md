# Operations: running it, delivering documents, traps, extensions

## What an operator can see and do

| Question | Where the answer is |
|---|---|
| Was this invoice paid, and was it corrected? | Two separate facts on the row: `paidAt` and `correctedByInvoiceId` |
| Did the tax system accept it? | `eInvoice.status`, with `eInvoice.error` holding the transport's own words |
| Who issued it, who downloaded it, when? | The host's audit log, fed by `InvoiceAuditEvent` |
| Can I find an invoice from two years ago? | The register filters and pages on the server, with no cap on age |
| What does the buyer receive? | The PDF route renders exactly what a mailer attaches |
| Can I fix a mistake? | Issue a correction. Nothing else changes an issued document |
| Can I undo "paid"? | No. `paidAt` is set once; see *Extensions* |

## Environment

The wiring template in [service.md](service.md) reads these for a single-company app. A multi-tenant host
keeps the same fields on its tenant row instead and sets only the time zone, if at all.

| Variable | Meaning | Default |
|---|---|---|
| `INVOICE_SELLER_NAME` | Seller name. Unset means nobody can issue | none |
| `INVOICE_SELLER_TAX_ID` | Seller tax identifier | none |
| `INVOICE_SELLER_ADDRESS` | Seller address, one or more lines | none |
| `INVOICE_SELLER_BANK_ACCOUNT` | Printed in the payment details | none |
| `INVOICE_PAYMENT_TERMS_DAYS` | Days from issue to due date | `14` |
| `INVOICE_VAT_EXEMPTION_BASIS` | Legal basis for exempt lines | none |
| `INVOICE_CURRENCY` | ISO currency code | `PLN` |
| `INVOICE_TIME_ZONE` | Business time zone for "today" | `Europe/Warsaw` |

Read them at request time, as the template does, so the app builds without them.

## Delivering an invoice by e-mail

The module renders; the host sends. A sender needs the same bytes the download route produces, without an
actor, because it usually runs from a queue.

```ts
// in the host's mail job, with the tenant id and invoice id the job was queued with
const data = await invoicing.pdfDataForDelivery(job.tenantId, job.invoiceId);
if (!data) return; // the invoice is gone or the job is for another tenant: drop it
const attachment = {
  filename: `${data.number.replace(/[^A-Za-z0-9._-]+/g, "_")}.pdf`,
  contentType: "application/pdf",
  contentBase64: Buffer.from(renderInvoicePdf(data)).toString("base64"),
};
```

Rules for the host's side of it:

- **Queue, do not send inline.** A Server Action that waits for a mail provider turns a slow provider into
  a form that appears to hang, and a retry into a second e-mail.
- **Deduplicate on the invoice id** while a job is pending, so two clicks are one e-mail.
- **Record the send** in the host's audit log with the recipient, and show the last send in the register.
- **Render at send time.** The document cannot change, so there is nothing to gain from storing the file,
  and the verification code on a later render carries the assigned number in place of the pending label.

## Going live

- [ ] The seller profile is complete for every tenant that will issue: name, tax identifier, address
- [ ] The migration ran in production, triggers and function included, and the database checks passed on a
      copy
- [ ] The series start is agreed with the accountant. To continue an existing series, insert the counter
      row with the last used sequence before the first document:
      `insert into invoice_number_counters values ('<tenant>', 'FV', 2026, 311)`
- [ ] One base invoice, one correction and one two-page invoice were opened in a PDF viewer and printed
- [ ] The read-only role sees the register and no mutating control
- [ ] The audit log shows an issue, a download and a correction
- [ ] With the KSeF bridge: the checklist in [ksef-bridge.md](ksef-bridge.md) and the `ksef` skill's own
      go-live list

## Traps

- **"Today" in UTC is yesterday's date for part of every night.** A server in UTC issues an invoice at
  00:30 local time with the previous day's date, and on 1 January with the previous year's number series.
  `today()` formats the date in the business time zone for that reason.
- **The year in the number is the issue date's year.** An invoice issued on 2 January with an issue date
  of 31 December takes a number in the old year's series, after documents that were issued later. The
  service allows a chosen issue date because month-end work needs it; an accountant who wants strict
  chronology wants the form's date field removed, not a rule in the service.
- **A sequence is continuous only if nothing is deleted.** The schema forbids deletes. A host that erases a
  tenant with the `invoicing.allow_delete` switch removes that tenant's whole history, not single rows.
- **A buyer's tax identifier is optional, and a wrong one is worse than none.** The checksum catches typing
  errors. It does not prove the identifier belongs to the buyer.
- **A source picker that loads "every invoiced source" stops being complete.** Any unbounded list is cut at
  the API's row limit, and the picker then offers sources that are already invoiced. Ask about the ids on
  screen with `invoicedSourceIds`.
- **An exempt line needs its legal basis before it can be issued.** The service refuses it otherwise. The
  basis is seller configuration, so the error belongs to whoever administers the tenant, and the form's
  message should say where to set it.
- **This module is not tax advice.** It prints and sends what it is given. Which rate applies, whether an
  exemption holds and when a correction is the right document are questions for an accountant.

## Extensions

Designs. None of these is in the templates and none has run anywhere; each names what it would touch.

- **Net-priced documents.** A `priceMode` on the document: net lines compute `net = qty * unitPrice`,
  `vat = net * rate`, `gross = net + vat`, and the layout's price column changes its label. The FA(3)
  builder already emits a net unit price. The model, the layout label and the tests change.
- **A chain of corrections.** Replace the single `correctedByInvoiceId` link with a list, and compute each
  correction's deltas against the state after the previous one, which means storing the corrected state or
  replaying deltas. The unique index, the store checks and the editor's loader change.
- **Undoing a payment, or partial payments.** A `payments` table of dated amounts, with `paidAt` derived
  from it. The guard trigger's one-way rule for `paid_at` goes away with the column.
- **An embedded font.** For names outside Latin-1 and Polish. The writer gains a TrueType subsetter and a
  CID font object, or the host swaps the writer for a PDF library and keeps the layout's structure. The
  deterministic-output tests then need the font file fixed in the repository.
- **Bulk download.** A date range as one archive for the accountant: a route that streams a ZIP of rendered
  files, audited once with the count and the range.
- **A register export.** The same filters as the register, as CSV, with one row per rate per document.
- **Offline issue and the second verification code.** In the `ksef` skill's QR reference. The layout
  places one code; a second goes to its left with its own label.
- **Buyers abroad and other currencies.** Country and EU VAT number on the buyer snapshot, an exchange rate
  on the document, and the matching FA(3) elements listed at the end of [fa3-xml.md](fa3-xml.md).
