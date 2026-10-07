# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.2] - 2026-10-07

Fix release, from scoring the prompt-1 agent eval runs against 0.1.1.

### Fixed

- `roundMoney` in `references/model.md` rounded some half-cent amounts down: 10.075 gave 10.07. The scaled
  value is cut to 15 significant digits before rounding. Apps built from 0.1.0 or 0.1.1 copy the new
  function; documents already issued keep their stored amounts.
- The invoice layout in `references/pdf.md` now starts a new page when a correction reason or the parties
  would cross the floor, instead of printing below the margin and over the verification code. Apps built
  from earlier versions copy the `room` helper and its three calls.
- The wiring file in `references/service.md` reads the environment with `||`, so an empty variable means
  unset. An empty `INVOICE_TIME_ZONE` made every issue throw, and an empty `INVOICE_PAYMENT_TERMS_DAYS`
  gave a due date equal to the issue date. Apps built from earlier versions make the same change.

### Changed

- `SKILL.md` gains the bare-app clauses: with no sign-in, `getInvoiceActor` keeps returning null; with no
  database, the in-memory store stays; with no runner, vitest is installed and the suites run as written;
  only the eight `INVOICE_*` variables; and the three facts the handover states.
- `references/adaptation.md` and `references/testing.md` say the package registry is not an external
  service and that a host with no runner installs vitest.
- `references/testing.md`: the rounding and verification-code tests assert the two fixes. Still 72 tests.
- `references/provenance.md` gains *Found by the agent evals*.

## [0.1.1] - 2026-10-06

Documentation-only release: the templates, the references' rules and the non-negotiables are unchanged from
0.1.0. Wording brought in line with the skill standard.

### Changed

- `CLAUDE.md`, the README verification recipe, `references/provenance.md` and `references/adaptation.md`
  describe the audit against the earlier implementation in the standard's wording.
- `CLAUDE.md` gains the single-suite test run and the rule that every changed factual claim is sourced and
  every template change gets its `provenance.md` entry.

## [0.1.0] - 2026-10-03

First release. A VAT invoicing module for a Next.js App Router app: numbering, corrections, a register, a
PDF, and an optional bridge to the `ksef` skill.

### Added

- `SKILL.md` with the architecture, seven critical facts, six hard rules, the quick start and the reference
  directory.
- `references/model.md`: the pure model, with symmetric rounding, the number format, correction deltas,
  validation and the tax identifier.
- `references/data-model.md`: the entities, the `InvoiceStore` interface and an in-memory store.
- `references/postgres.md`: the migration with numbering stamped inside the insert, immutability triggers
  and `invoicing_create`, and a Postgres store written against a minimal `Queryable`.
- `references/service.md`: issue, correct, mark paid, list and render data, the wiring file and the Server
  Actions.
- `references/pdf-writer.md` and `references/pdf.md`: a dependency-free PDF writer with Polish diacritics
  and wrapped text, the invoice layout with Polish and English label tables, the download route.
- `references/admin-ui.md`: the register, the issue form and the correction editor as structure and
  behaviour, with one unstyled line editor.
- `references/fa3-xml.md` and `references/ksef-bridge.md`: an FA(3) builder from a stored document and the
  bridge that hands it to the `ksef` skill and draws the KOD I code.
- `references/adaptation.md`, `references/operations.md`, `references/testing.md`,
  `references/testing-service.md` and `references/provenance.md`.
- `evals/prompts.md` with three operator prompts, and `.github/workflows/agent-eval.yml`.
