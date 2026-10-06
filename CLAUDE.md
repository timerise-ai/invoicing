# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json` here and nothing
in this repository executes. It teaches an agent how to build a VAT invoicing module in a **Next.js App
Router** app: continuous numbering, corrections, a register, an A4 PDF from a dependency-free writer, and an
optional bridge to the `ksef` skill.

Keep the two straight: the commands and code in `references/` describe the app the agent will generate, not
this repository. The `psql` calls, the migration, the `vitest` and `bun test` invocations and the host probe
all run in that generated app. The one thing checked here is that the templates compile and their tests
pass, and that check runs in a scratch project; the recipe is under *Editing conventions* below.

The skill was written by the engineer who has shipped this module; the earlier implementation it was audited
against was an invoicing section in a multi-tenant business application on Postgres.
`references/provenance.md` is the ledger of that audit: sixteen entries on what changed and how the
templates verify it, what was kept deliberately, and what was designed here and has run only in the skill's
own tests. That file is the rationale layer: read it before "simplifying" anything.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. The frontmatter `description` is the trigger surface; the body carries the
  architecture diagram, seven **critical facts**, six **hard rules**, the quick-start order, the **reference
  directory table** mapping trigger keywords to files, and a closing line linking the skills index.
- `README.md`: the human-facing front door, in the section order of the skill standard: install, activation,
  the file table, the six non-negotiables, requirements, verification, the *Not this* table, contributing.
- `references/*.md`: one topic per file, loaded on demand. `adaptation.md` (the seam contract) is the design
  entry point. `model.md`, `data-model.md`, `service.md`, `pdf-writer.md` and `pdf.md` are the core module;
  `postgres.md` the production store; `admin-ui.md` the surface; `fa3-xml.md` and `ksef-bridge.md` the
  optional e-invoicing half; `operations.md` running it; `testing.md` and `testing-service.md` the suites;
  `provenance.md` the audit.
- `evals/prompts.md`: the prompts the agent evals run. Result files join it after each run.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval workflow, copied verbatim
  from the skill standard. Never edit it and never add a trigger.

## Editing conventions

- **Code blocks name their destination on the first line** as a comment, for example
  `// lib/invoicing/model.ts` or `-- db/migrations/0001_invoicing.sql`. A `typescript` block with no such
  line continues the file named by the block before it in the same reference. That line is what lets a
  block be written to its file, so keep it and keep imports complete.
- **The fence language says whether a block is compiled.** `typescript`, `tsx` and `sql` blocks are
  templates and are compiled. `ts` blocks are sketches of host code (the `pg` wiring line, the bridge
  wiring, the status-cron calls, the mail job, the source type) and are not.
- **The templates are compiled and run.** Write every `typescript`, `tsx` and `sql` block to its named
  path in a scratch directory, then

  ```bash
  npm i -D typescript next react @types/react @types/node vitest qrcode @types/qrcode server-only
  npx tsc --noEmit      # strict, noUncheckedIndexedAccess, skipLibCheck, jsx react-jsx, paths @/* to ./*
  npx vitest run lib    # 72 tests in five files: 20 model, 16 PDF, 21 service, 9 FA(3), 6 bridge
  psql "$DATABASE_URL" -f db/migrations/0001_invoicing.sql -f db/checks/invoicing-checks.sql
  ```

  `skipLibCheck` is not optional, or Next's own declarations fail the run. Do not set `baseUrl`. Re-run
  after editing any block. A change to `fa3-xml.md` also needs the `xmllint` run that file describes.
  One suite at a time: `npx vitest run lib/invoicing/model.test.ts` (or `bun test <file>`). The suites
  come from `testing.md` (`model.test.ts`, `invoice-pdf.test.ts`), `testing-service.md`
  (`service.test.ts`), `fa3-xml.md` and `ksef-bridge.md` (`ksef/fa3-xml.test.ts`, `ksef/bridge.test.ts`).
- **Identifiers are shared across files.** `VAT_RATES`, `VatRate`, `roundMoney`, `computeLine`,
  `computeTotals`, `vatSummary`, `formatInvoiceNumber`, `buildCorrectionLines`, `validateLines`,
  `InvoiceError`, `InvoiceErrorCode`, `INVOICE_SERIES`, `Invoice`, `InvoiceWithLines`, `NewInvoice`,
  `InvoiceStore`, `invoiceStatus`, `SellerSnapshot`, `EInvoiceState`, `createMemoryInvoiceStore`,
  `createPostgresInvoiceStore`, `Queryable`, `createInvoicing`, `InvoiceActor`, `SellerProfile`,
  `EInvoicePort`, `Verification`, `InvoiceActionResult`, `getInvoiceActor`, `invoicing`, `PdfDoc`,
  `wrapText`, `renderInvoicePdf`, `InvoicePdfData`, `InvoicePdfStrings`, `QrMatrix`, `pdfResponse`,
  `LineDraft`, `draftToLine`, `buildFa3Xml`, `toFa3Document`, `fa3Bucket`, `createKsefBridge`, `qrMatrix`,
  the SQL names `invoicing_create`, `invoice_number_counters` and the `invoicing:<code>` exception prefix,
  and the `INVOICE_*` environment variables appear in several references. Rename in all of them or none.
- **The error codes are a contract in three places:** `InvoiceErrorCode` in `model.md`, the
  `raise exception 'invoicing:<code>'` strings in `postgres.md` with the `KNOWN` list beside them, and the
  in-memory store in `data-model.md`. A new store-level refusal is added to all three.
- **The two stores enforce the same rules.** The service suite runs against the in-memory store; a rule
  added to the migration and not to `memory-store.ts` is a rule the tests no longer see.
- **Keep the three tables in sync** with `references/`: the reference directory in `SKILL.md`, the
  quick-start list in `SKILL.md`, and the file table in `README.md`. Links are relative:
  `[x.md](references/x.md)` from `SKILL.md`, `[x.md](x.md)` between references.
- **Do not remove the odd-looking parts.** The sign handling in `roundMoney`, the trigger instead of a
  "next number" query, `greatest(5, length(...))` around `lpad`, dates selected as text, microseconds in
  `created_at` and milliseconds in `paid_at`, the `floor` function in the layout, the octal escapes and the
  128 to 159 guard in `encodeText`, `NaN` instead of `0` in `draftToLine`, the `try` blocks in
  `afterCreate`, the audit before the bytes in `pdfData`, the hash stored at enqueue: each is a ledger
  entry or a documented judgement call. Check `provenance.md` before touching one.
- **The numbers that remain are load-bearing.** 72 tests and their split, sixteen ledger entries, the page
  geometry in the layout, the five-digit number padding, the 100-row list cap, the FA(3) schema version
  `1-0E`, the versions in *How the templates were checked*. Do not restate them loosely and do not add new
  ones. Figures describing the earlier implementation's deployment do not appear anywhere.
- **Non-ASCII characters appear only as data.** Polish label strings, the glyph map and test fixtures
  carry Polish letters. Prose, tables, headings and code comments are plain ASCII with plain punctuation.
- **Mark additions as additions.** Anything designed in the skill and never run in the earlier
  implementation belongs in *Added in the skill* in `provenance.md`, or in `operations.md` under
  *Extensions* as a design. The join to the `ksef` skill's tables is a contract and a sketch and is stated
  as such.
- **Evals are not skill content.** A new prompt or an eval result is committed as `chore(evals): ...`,
  never causes a version bump and never rides in a release commit. The frontmatter of a result file is what
  was measured and is not edited; a failing run stays committed, and the fix is the next release.
- **Never present the non-negotiables as optional.** The number inside the insert, the single transaction,
  immutability, the server-side actor, server-computed amounts and deltas, and nothing failing after the
  document exists are hard rules in `SKILL.md` and non-negotiables in `README.md` and `adaptation.md`; keep
  the three lists identical and in the same order.
- The route path `/invoices`, the series prefixes `FV` and `FK`, the table names, the default unit and the
  label tables are meant to be renamed by the host, and `adaptation.md` carries that procedure. The type
  and function names in `lib/invoicing`, the `invoicing:<code>` prefix and the FA(3) element names are the
  authoring contract and are not renamed.
- **Changes are sourced and recorded.** A changed factual claim is verified against the FA(3) schema with
  `xmllint`, the PDF 1.4 reference and Helvetica metrics, PostgreSQL's documentation or a reproduction,
  never from memory, and any change to a template gets its entry in `provenance.md`. Commits follow
  Conventional Commits; `CHANGELOG.md` follows Keep a Changelog and SemVer, and releases follow
  STANDARD.md in the skills index.
