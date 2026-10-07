# Provenance

The engineering ledger for whoever edits this skill. The templates were written by the engineer who has
shipped this module, and audited against the earlier implementation: an invoicing section inside a
multi-tenant Next.js business application on Postgres, with VAT invoices, corrections, a PDF and a KSeF
integration. Four lists follow and they are kept apart: what the audit changed, what was kept on purpose,
what is new in the skill and has never run outside its own tests, and the defects that agent evals found in
a released template.

## Changed by the audit

Sixteen entries. Each says what the earlier implementation did, what the templates do, and what verifies it.

### 1. The seller is a snapshot

The earlier implementation copied the buyer onto the invoice and read the seller from the tenant profile
every time a PDF or an e-invoice file was built. Editing the company name, address or bank account changed
every past document on its next render.
**Shipped:** `SellerSnapshot` on the invoice, written at issue ([data-model.md](data-model.md)).
`service.test.ts` changes the profile after issue and reads the document back.

### 2. A document and its lines are one write

The earlier implementation inserted the invoice, then its lines, then the e-invoice job, as three calls. A
failure after the first left a numbered document with no lines and no way to remove it.
**Shipped:** `invoicing_create` ([postgres.md](postgres.md)). The database checks count the lines and show
that a refused insert returns its number.

### 3. A correction is one write, and one per document

The earlier implementation inserted the correction, then its lines, then marked the original, with no
unique constraint on the link. Two operators correcting the same invoice at once could both succeed.
**Shipped:** a row lock on the original and a unique index on `corrects_invoice_id`. Four parallel
corrections of one document were run against the scratch database: one succeeded and the correction series
advanced by one.

### 4. Payment and correction are separate facts

The earlier implementation stored one status, `issued`, `paid` or `corrected`. Correcting a paid invoice
overwrote `paid`, and a corrected invoice could never be marked paid.
**Shipped:** `paidAt` and `correctedByInvoiceId`, with `invoiceStatus` derived. `service.test.ts` corrects
a paid invoice and finds it still paid.

### 5. Immutability is enforced by the database

The earlier implementation relied on the application never updating an invoice, while the row-level policy
allowed any billing role to update any column through the data API.
**Shipped:** guard triggers on both tables. The database checks try to edit a total, edit a line, delete a
document and undo a payment.

### 6. The register filters and pages on the server

The earlier implementation loaded the newest 200 invoices and filtered them in the browser, so a search
for an older document returned nothing.
**Shipped:** `InvoiceFilters` and a cursor in the store. `service.test.ts` pages through seven documents
three at a time and filters by text, date range and status.

### 7. Rounding is symmetric

The earlier implementation rounded with `Math.round`, which rounds a half toward positive infinity. With a
fractional quantity, a correction's negated line could differ from the original by one unit in the last
place. The measured rate on a sweep of fractional quantities and two-decimal prices was more than a quarter
of cases.
**Shipped:** `roundMoney` ([model.md](model.md)). `model.test.ts` sweeps five quantities across two
thousand prices.

### 8. Text wraps instead of being cut

The earlier implementation cut a line name at 52 characters with an ellipsis and drew party names,
addresses and the correction reason on one row each, so a long seller name ran into the buyer column.
**Shipped:** `wrapText` ([pdf-writer.md](pdf-writer.md)) and its use in the layout. `invoice-pdf.test.ts`
checks that every word of a long name is printed.

### 9. The verification code is never overprinted

The earlier implementation let table rows run to 130 points from the bottom of the first page while the
code's top edge was at 146, so the last rows of a long invoice were drawn across it.
**Shipped:** the `floor` function in the layout ([pdf.md](pdf.md)). The test renders five document lengths
and fails when the floor is removed.

### 10. No verification code without e-invoicing

The earlier implementation computed the e-invoice hash at issue for every tenant with a tax identifier,
whether or not it had credentials, and built the link with a default test environment. Its invoices also
stayed in the pending state for good.
**Shipped:** `isEnabled` decides `pending` or `not_required` at issue, and `verification` returns null
without a stored hash ([ksef-bridge.md](ksef-bridge.md)). `bridge.test.ts` covers the disabled tenant.

### 11. The hash is of the enqueued bytes, stored once

The earlier implementation built the XML twice, at issue for the hash and again at send time, and relied
on the two builds being identical. They read the live seller profile, so an edit in between produced a
link that did not verify.
**Shipped:** the bridge hashes the buffer it hands to `enqueue`, and the builder reads the snapshot.

### 12. The exemption basis is printed

The earlier implementation stored the legal basis for exempt lines and sent it in the e-invoice, and did
not print it on the PDF.
**Shipped:** the payment details block prints it. `invoice-pdf.test.ts` checks the line.

### 13. Input is validated before a number is reserved

The earlier implementation checked the VAT rate and that there was at least one line. A zero or negative
quantity, an unparseable price, a blank line name and a malformed date reached the database.
**Shipped:** `validateLines` and `isIsoDate`, called in the service. The form helper turns an unparseable
number into `NaN`, not `0`, so it is refused.

### 14. Nothing after the insert fails the request

The earlier implementation ran the e-invoice enqueue and the audit after the insert and let their errors
reach the form. The operator saw a failure, retried, and issued a second document.
**Shipped:** `afterCreate` in [service.md](service.md). Two tests in `service.test.ts` hold it.

### 15. Expected failures are return values with codes

The earlier implementation threw `Error` with a sentence in one language from its Server Actions.
**Shipped:** `InvoiceError` codes and `InvoiceActionResult`, following the Next.js error-handling guide,
which says to model expected errors as return values.

### 16. The picker asks about the ids on screen

The earlier implementation loaded every invoiced source id in one unbounded query to filter its picker,
through a data API that caps a response at a fixed number of rows.
**Shipped:** `invoicedSourceIds(tenantId, kind, ids)`.

## Kept deliberately

- **Gross prices, with net and VAT derived per line.** It is the right model where the price list is what
  the customer pays, and it makes every line add up. Net pricing is an extension, not a flag.
- **Amounts as two-decimal numbers, not integer minor units.** Every amount passes through `roundMoney`
  and is stored as `numeric(12,2)`. The tests hold the sums.
- **One correction per base document.** A chain needs the corrected state stored or replayed. The single
  link is simple, and the limit is stated.
- **Positional correction deltas.** Removing an early line shows the later ones as changed. The amounts
  are right, and the rule is small enough to reason about.
- **Payment is set once.** There is no "unpaid" action. It keeps the guard trigger simple; a payments
  table is the extension.
- **A chosen issue date, and the series year read from it.** Month-end work needs it. The consequence for
  chronology is in [operations.md](operations.md).
- **No stored PDF.** The document is rendered from immutable rows on every request.
- **No PDF library, no embedded font, no compression.** Small deterministic output, at the cost of scripts
  outside Latin-1 and Polish. Bold text is measured with regular metrics.
- **The verification code as vector rectangles**, and the `OFFLINE` label until a number exists.
- **`np` is refused for KSeF, and `0` is filed as domestic.** The schema splits both into cases that only a
  tax determination can tell apart.
- **`TypKorekty` is omitted** from a correction, for the same reason.
- **A net unit price at four decimals** in the e-invoice, so a price below one hundredth still multiplies
  back to the line net.
- **The download is audited before the bytes exist**, and an audit failure there stops the download.

## Added in the skill

Designed here. These have run in the skill's own tests and scratch database and nowhere else.

- The `InvoiceStore` interface, the in-memory store, and the Postgres store written against `Queryable`.
  The earlier implementation called one database client inline.
- `invoicing_create`, the guard triggers and the `invoicing.allow_delete` switch.
- The `source` pair in place of one nullable foreign key per origin.
- `EInvoicePort`, `preflight`, and the bridge's `onAccepted` and `onRejected`. The earlier implementation
  had its own queue and its own KSeF client; the join to the `ksef` skill's tables is a contract and a
  sketch, and has not been run against that skill's code.
- The `NrKSeFN` branch for a correction whose original has no KSeF number. Validated against the schema,
  never sent.
- The English label table, the `locale` and `currency` parameters, and the separate VAT column in the VAT
  table with the wider amount columns.
- The guard in `encodeText` for bytes 128 to 159, and the extra entries in `ASCII_FALLBACK`.
- The single-company wiring from environment variables.
- `InvoiceLinesEditor` in bare HTML elements.
- Everything under *Other data layers* in [postgres.md](postgres.md) and *Extensions* in
  [operations.md](operations.md): designs, not run.

## Found by the agent evals

Defects in a released template, found when an eval agent changed the template and a probe confirmed the
claim. Each was fixed in the template, and the suites gained an assertion that fails on the old code.

- **`roundMoney` lost some halves (0.1.2).** `Math.round((x + Number.EPSILON) * 100)` adds an epsilon too
  small to matter above 1: 10.075 is stored as 10.07499..., and it rounded to 10.07. A sweep of quantity
  and price products, checked against integer arithmetic, found thousands that rounded the wrong way. The
  scaled value is now cut to 15 significant digits before rounding, and the same sweep finds none.
  Symmetry (entry 7) was never affected.
- **The header and the parties did not break pages (0.1.2).** A correction reason or a buyer address has no
  length limit, and a few thousand characters ran the rows below the bottom margin and through the
  verification code (entry 9). Those rows now start a new page at the same floor as table rows.
- **An empty variable was not unset (0.1.2).** The wiring read the environment with `??`, so an example
  file copied with empty values set the time zone to "" and every issue threw, and set the payment terms
  to 0 days. It reads with `||`. The wiring file is outside the suites; the fix was checked by hand.

## How the templates were checked

Every TypeScript block was written to its named path and compiled with TypeScript 5.9 under `strict` and
`noUncheckedIndexedAccess`. The five suites, 72 tests, ran under vitest 4 and under bun. The service and
bridge suites also ran against the Postgres store on PostgreSQL 18. The migration and the SQL checks ran
on the same database, with twenty parallel issues and four parallel corrections. Five documents from the
FA(3) builder validated against the Ministry's `1-0E` schema with `xmllint`. Three rendered files were
opened and read. Nothing was sent to KSeF.

## Upgrading a module that looks like the earlier implementation

Most damaging first:

1. Make issue and correction single transactions, and add the unique index on the correction link (2, 3).
2. Snapshot the seller, and backfill existing rows from the current profile (1).
3. Add the guard triggers (5).
4. Stop printing a verification code for tenants with no e-invoicing, and store the hash of sent bytes
   (10, 11).
5. Move the register's filters to the server (6).
6. Split the status (4), fix the rounding (7), and stop throwing after the insert (14).
7. The layout entries (8, 9, 12).
