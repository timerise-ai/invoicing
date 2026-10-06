---
name: invoicing
description: >
  Build a VAT invoicing module in a Next.js App Router app: gapless numbered invoices issued standalone or
  from a sale, correcting invoices with signed deltas, a register, and an A4 PDF rendered by a
  dependency-free writer, with an optional bridge that sends each document to KSeF through the ksef skill.
  Use when: (1) an app must issue invoices, credit notes or correcting invoices and print or e-mail them as
  PDF, (2) invoice numbers must be continuous per series and year under concurrent issuers, (3) an existing
  invoices module needs auditing: gaps in numbering, documents that change after issue, a PDF that cuts
  long names, (4) invoices must also go to KSeF as FA(3) with the KOD I code on the PDF, (5) the user
  mentions: invoice, faktura, faktura VAT, faktura korygujaca, correction invoice, credit note, invoice
  PDF, invoice number, gapless numbering, numeracja faktur, VAT summary, net and gross, NIP, invoice
  register, sales register, issue invoice from order, FA(3), KSeF, KOD I, QR on invoice. Carries numbering
  stamped inside the insert, one-transaction issue and correction, seller and buyer snapshots, database
  immutability triggers, symmetric rounding, a PDF writer with Polish diacritics and no font files, and
  the suites that hold them. Next.js App Router; the store, the auth guard, the seller profile and the
  e-invoicing port are seams, with Postgres and in-memory stores shipped. Not a payment processor, not
  subscription billing, not accounting or tax advice, and not the KSeF transport itself.
---

# Invoicing: numbered VAT invoices, corrections and a PDF

An invoicing module issues a document, numbers it, lets it be corrected, and prints it. The idea the design
turns on: **an issued invoice is a fact, not a record.** It is numbered once, by the database, inside the
write that creates it; it carries its own copy of both parties; and from then on nothing edits it. A mistake
is answered with another document. Everything else in the module follows from holding that.

Written by the engineer who has shipped this module. The earlier implementation it was audited against was
an invoicing section in a multi-tenant business application on Postgres. The facts and rules below are what
the suites and the database checks verify; [provenance.md](references/provenance.md) has the record.

## When to use

A Next.js App Router app that sells something and must hand the buyer a VAT invoice: issued by staff in an
admin area, standalone or from a sale, order or booking the app already holds; corrected when wrong;
downloaded or e-mailed as PDF; and, in Poland, sent to KSeF.

## When NOT to use

| Instead of this | Use |
|---|---|
| Charging a card, a checkout, subscriptions, dunning | The payment provider's own invoices, or [stripe-connect-subscriptions](https://github.com/timerise-ai/stripe-connect-subscriptions) |
| Talking to KSeF: auth, encryption, sessions, UPO, purchase invoices | [ksef](https://github.com/timerise-ai/ksef); this skill only builds the file and joins to it |
| A general ledger, VAT returns, JPK files | An accounting system; this module is the sales document, not the books |
| Deciding a rate, an exemption or whether to correct | An accountant; the module prints and sends what it is given |
| Fiscal receipts from a cash register | The fiscal device's integration |

## Architecture

```
admin UI (host components)          Server Actions / route            lib/invoicing (copied as written)
  register, filters, cursor  ---->  getInvoiceActor()  ----------->   service: issue, correct, markPaid,
  issue form, line editor           InvoiceActionResult               list, get, pdfData
  correction editor                 GET /invoices/[id]/pdf               |          |            |
                                                                         v          v            v
                                                                   InvoiceStore   model      invoice-pdf
                                                                   memory | pg    (pure)     pdf writer
                                                                         |
                             Postgres: invoices, invoice_lines, counters, numbering trigger,
                             guard triggers, invoicing_create()

optional:  service --EInvoicePort--> ksef bridge --FA(3) XML--> ksef skill (send, poll) --onAccepted--> store
```

- **`lib/invoicing`** is pure TypeScript with no dependency. One file, `index.ts`, joins it to the host.
- **The store** is an interface. The in-memory one backs the tests; the Postgres one backs the app.
- **The seams** are in [adaptation.md](references/adaptation.md): actor, seller profile, clock, audit,
  store, strings, sources, delivery, e-invoicing.

## Critical facts

1. **Continuity of numbering can only be held by the database.** A "next number" query followed by an
   insert leaves a gap whenever the insert fails, and two issuers can read the same number.
2. **An invoice is a snapshot, not a view.** A document that joins to the live company profile changes when
   the profile does, and so does the hash an e-invoice link depends on.
3. **Prices are gross, and net and VAT are derived per line.** Rounding is half away from zero, so a
   correction's negated line cancels the original exactly, for fractional quantities too.
4. **Status is two facts.** Paid and corrected are independent; a single stored status loses one of them.
5. **A Server Action is a public endpoint.** Its arguments come from the network whatever its TypeScript
   signature says, so the service validates every field it stores and derives the tenant itself.
6. **The PDF is rendered, never stored, and the writer has two fonts.** Text is Latin-1 plus Polish
   letters; anything else prints as a question mark, in place, where it can be seen.
7. **E-invoicing is a port.** Without it every document is `not_required` and the PDF has no code.

## Hard rules

1. **Assign the number inside the insert.** Never compute it in application code or a separate round trip:
   the counter must roll back with the document it numbered.
2. **Write a document and its lines in one transaction**, and a correction with the link on its original.
   A numbered document with no lines cannot be removed.
3. **Never update or delete an issued document.** Both parties are snapshots, a mistake is fixed by a
   correction, and database triggers enforce it against any client.
4. **Take the tenant and the permissions from the server-side actor on every call.** No route, action or
   store method accepts a tenant id from input.
5. **Compute every amount and every correction delta on the server**, from the stored lines. The client
   sends lines and a corrected state, never totals or deltas.
6. **Let nothing fail the request once the document exists.** A hand-off or audit failure is recorded and
   reported; an error would send the operator back to issue a second invoice.

## Quick start

1. Probe the host, fill in the seam table and confirm the rename: [adaptation.md](references/adaptation.md).
2. Copy the pure model and the tax identifier: [model.md](references/model.md).
3. Copy the types, the store interface and the in-memory store: [data-model.md](references/data-model.md).
4. Copy the writer and the layout, and add the download route: [pdf-writer.md](references/pdf-writer.md),
   [pdf.md](references/pdf.md).
5. Copy the service, write the wiring file with the host's guard, seller and clock, add the actions:
   [service.md](references/service.md).
6. Run the migration and switch to the Postgres store: [postgres.md](references/postgres.md).
7. Build the register, the issue form and the correction editor on the host's components:
   [admin-ui.md](references/admin-ui.md).
8. Run the suites and the database checks: [testing.md](references/testing.md),
   [testing-service.md](references/testing-service.md).
9. Wire delivery, set the environment, walk the go-live list: [operations.md](references/operations.md).
10. Optional, for KSeF: [fa3-xml.md](references/fa3-xml.md), then
    [ksef-bridge.md](references/ksef-bridge.md), with the `ksef` skill installed.

With no database yet, steps 1 to 5 and 7 give a working module on the in-memory store.

## Reference directory

| Scenario | Trigger keywords | Reference |
|---|---|---|
| Fitting it into an existing app | adapt, seam, rename, tenant, auth guard, getInvoiceActor, seller profile, i18n, locale segment, order of work | [adaptation.md](references/adaptation.md) |
| VAT math, number format, deltas, validation | VAT_RATES, computeLine, roundMoney, vatSummary, formatInvoiceNumber, buildCorrectionLines, validateLines, InvoiceError, NIP, isValidNip | [model.md](references/model.md) |
| Entities and the store interface | Invoice, InvoiceStore, snapshot, invoiceStatus, NewInvoice, InvoiceFilters, cursor, createMemoryInvoiceStore | [data-model.md](references/data-model.md) |
| Schema, numbering, immutability | migration, invoice_number_counters, trigger, lpad, invoicing_create, immutable, row lock, Queryable, pg, Supabase, RLS, Drizzle, Prisma | [postgres.md](references/postgres.md) |
| Issue, correct, pay, list, wiring, actions | createInvoicing, issue, correct, markPaid, pdfData, InvoiceActor, EInvoicePort, audit, InvoiceActionResult, revalidatePath | [service.md](references/service.md) |
| The PDF writer | PdfDoc, PdfPage, textWidth, wrapText, Helvetica, /Differences, WinAnsi, diacritics, no font embedding | [pdf-writer.md](references/pdf-writer.md) |
| The invoice layout and download | renderInvoicePdf, InvoicePdfData, InvoicePdfStrings, QR position, pdfResponse, Content-Disposition, no-store, route | [pdf.md](references/pdf.md) |
| Register, issue form, correction editor | list, filters, load more, LineDraft, draftToLine, InvoiceLinesEditor, source picker, read-only role | [admin-ui.md](references/admin-ui.md) |
| FA(3) XML from a stored invoice | FA(3), buildFa3Xml, P_12, P_13, bucket, Adnotacje, KOR, NrKSeFN, xmllint, XSD | [fa3-xml.md](references/fa3-xml.md) |
| Joining the ksef skill | KSeF, createKsefBridge, enqueue, onAccepted, onRejected, KOD I, qrMatrix, OFFLINE, preflight | [ksef-bridge.md](references/ksef-bridge.md) |
| Running it | env vars, e-mail attachment, go-live, time zone, year rollover, series start, extensions, net prices, payments | [operations.md](references/operations.md) |
| Model and PDF tests | vitest, bun test, model.test.ts, invoice-pdf.test.ts | [testing.md](references/testing.md) |
| Service tests and database checks | service.test.ts, psql, invoicing-checks.sql, concurrency | [testing-service.md](references/testing-service.md) |
| The audit ledger | provenance, changed, kept deliberately, added, upgrading an existing module | [provenance.md](references/provenance.md) |

Part of the [Timerise Skills](https://github.com/timerise-ai/skills) index, which lists the sibling skills.
