# invoicing

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build a VAT invoicing module in a
**Next.js App Router** app: invoices with continuous numbers issued standalone or from a sale, correcting
invoices with signed deltas, a register that filters on the server, and an A4 PDF rendered by a
dependency-free writer with Polish diacritics and no font files. An optional bridge builds each document as
FA(3) and hands it to the [ksef](https://github.com/timerise-ai/ksef) skill, which makes the two together
a complete invoicing system for Poland: issue, number, print, send to KSeF, show the verification code.

The insight the whole design turns on: **an issued invoice is a fact, not a record.** It is numbered once,
by the database, inside the write that creates it; it carries its own copy of the seller and the buyer; and
from then on nothing edits it. A mistake is answered with another document.

This skill is written by the engineer who has shipped this module, an invoicing section in a multi-tenant
business application on Postgres. The templates hold these properties: a number is consumed only by a
document that exists, a document and its lines are one write, an issued document cannot be updated or
deleted, a correction's negated line cancels the original to the last digit, no text on the PDF is cut, and
the verification code is never overprinted. Each is verified by the suites in `references/testing.md` and
`references/testing-service.md` or by the database checks. The record of what the audit changed, what was
kept and what is new is in [references/provenance.md](references/provenance.md).

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/invoicing
```

Name the agents instead with `-a`, for example `npx skills add timerise-ai/invoicing -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/invoicing.git ~/.claude/skills/invoicing
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For
another agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull`
updates every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/invoicing ~/.agents/skills/invoicing
```

Update the skill with `git pull` in its directory. The current release is **0.1.2**. See
[CHANGELOG.md](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: issuing invoices, credit notes or
correcting invoices from an app; an invoice PDF; continuous invoice numbering under concurrent issuers; an
audit of an existing invoices module; sending invoices to KSeF with the code on the PDF; or the vocabulary
itself, such as faktura, faktura korygujaca, invoice number, VAT summary, NIP, FA(3) and KOD I. Invoke it
explicitly with `/invoicing` in Claude Code, `$invoicing` in Codex CLI, or from `/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture diagram, seven critical facts, six hard rules, the quick start, and the reference directory |
| `README.md` | This file: the human-facing front door |
| `CHANGELOG.md` | Every release, newest first |
| `CLAUDE.md` | What the repository is and its editing conventions, for an agent editing the skill itself |
| `references/adaptation.md` | The seam contract with the host app: actor, seller profile, store, clock, audit, strings, sources, the domain rename, the order of work |
| `references/model.md` | The pure model: rates, symmetric rounding, line and document totals, the number format, correction deltas, validation, the tax identifier |
| `references/data-model.md` | The entities, the `InvoiceStore` interface and the in-memory store |
| `references/postgres.md` | The migration with the numbering trigger, the immutability triggers and `invoicing_create`, the Postgres store, notes for other data layers |
| `references/service.md` | The service: issue, correct, mark paid, list, render data; the wiring file; the Server Actions |
| `references/pdf-writer.md` | The dependency-free PDF writer: pages, two built-in fonts, Polish diacritics, wrapped text |
| `references/pdf.md` | The invoice layout with its label tables, the download response and the route |
| `references/admin-ui.md` | The register, the issue form and the correction editor: structure, states and behaviour, with one unstyled line editor |
| `references/fa3-xml.md` | Optional: the FA(3) builder and its rate buckets, with its tests |
| `references/ksef-bridge.md` | Optional: the bridge to the `ksef` skill, the verification code, with its tests |
| `references/operations.md` | Environment, delivery by e-mail, the go-live list, traps, extensions |
| `references/testing.md` | The model and PDF suites and how to run them |
| `references/testing-service.md` | The service suite and the SQL checks for the schema |
| `references/provenance.md` | The engineering ledger: what the audit of the earlier implementation changed and how the templates verify it, what was kept on purpose, and what is new in the skill |
| `evals/` | The prompts an operator types after installing (`prompts.md`) and one file per agent eval: the skill installed into an empty Next.js app, one prompt, no help, then type-checked, built and tested |
| `.github/workflows/agent-eval.yml` | The caller of the index's eval workflow, the same in every skill |

The seam is the table at the top of `references/adaptation.md`. It bounds a module that owns its documents
and nothing else: the session, the tenant, the company profile, the clock's time zone, the audit log, every
component and every sentence are the host's. The data layer sits behind `InvoiceStore`, with an in-memory
implementation for tests and a first build and a Postgres implementation for production. E-invoicing sits
behind `EInvoicePort`, and the module is complete without it.

## The six non-negotiables

These travel with the module and are never optional. Each is stated as a hard rule in `SKILL.md`:

1. **Assign the number inside the insert.** Never compute it in application code or a separate round trip:
   the counter must roll back with the document it numbered. The database checks refuse an insert and find
   the next document holding the number it would have had; twenty parallel issues produce twenty
   consecutive numbers.
2. **Write a document and its lines in one transaction**, and a correction with the link on its original.
   A numbered document with no lines cannot be removed. `invoicing_create` is one call, and four parallel
   corrections of one document produce exactly one.
3. **Never update or delete an issued document.** Both parties are snapshots, a mistake is fixed by a
   correction, and database triggers enforce it against any client. The checks try to edit a total, edit a
   line, delete a document and undo a payment.
4. **Take the tenant and the permissions from the server-side actor on every call.** No route, action or
   store method accepts a tenant id from input. The service suite shows a second tenant nothing and lets it
   correct nothing.
5. **Compute every amount and every correction delta on the server**, from the stored lines. The client
   sends lines and a corrected state, never totals or deltas. The model suite holds the arithmetic,
   including a negated line that cancels its original for fractional quantities.
6. **Let nothing fail the request once the document exists.** A hand-off or audit failure is recorded and
   reported; an error would send the operator back to issue a second invoice. Two tests in the service
   suite fail the audit and the e-invoice hand-off and still receive the invoice.

Everything else is the host app's: auth, tenancy, the company profile, components, styling, language.

## Requirements

- A Next.js App Router app. The route template uses the promise form of `params` from Next.js 15 and 16.
- Postgres for production. The module runs on the in-memory store with no database, for tests and a first
  build only.
- No runtime dependency for the core module. The optional KSeF bridge needs `qrcode` and the
  [ksef](https://github.com/timerise-ai/ksef) skill.

## Verification

Every TypeScript block in `references/` names its destination file and is meant to be written to
that file and compiled. The blocks form one project: write each to its path in a scratch directory with
TypeScript, `next`, `react`, `vitest`, `qrcode` and their types installed, then

```bash
npx tsc --noEmit        # strict, noUncheckedIndexedAccess, skipLibCheck, jsx react-jsx, paths {"@/*": ["./*"]}
npx vitest run lib      # 72 tests in five files; bun test lib runs the same files
psql "$DATABASE_URL" -f db/migrations/0001_invoicing.sql -f db/checks/invoicing-checks.sql
```

Blocks fenced as `ts` are sketches of host code and are not part of that project. `references/provenance.md`
says what was run, on which versions, and what was not.

## Not this

| Not this | Use instead |
|---|---|
| Charging a card, a checkout, subscriptions, dunning | The payment provider's invoices, or [stripe-connect-subscriptions](https://github.com/timerise-ai/stripe-connect-subscriptions) |
| Talking to KSeF: auth, encryption, sessions, UPO, purchase invoices | [ksef](https://github.com/timerise-ai/ksef); this skill builds the file and joins to it |
| A general ledger, VAT returns, JPK files | An accounting system |
| Deciding a rate, an exemption or whether to correct | An accountant; this skill covers the module, not interpretations of the VAT Act |
| Fiscal receipts from a cash register | The fiscal device's integration |

## Contributing

Issues and pull requests are welcome here. This repository is markdown only: nothing in it executes, and
what is checked is that the code blocks in `references/` compile and their tests pass in a scratch project,
by the recipe under *Verification*. Claims in this skill are meant to be verifiable: if you change a factual
claim, say how you verified it: against the published FA(3) schema with `xmllint`, the PDF 1.4 reference
and the Helvetica metrics, PostgreSQL's documentation, or a reproduction, never from memory.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a documented reason, and `references/provenance.md` is the ledger that must stay
truthful: read it before simplifying anything, and add an entry for anything you change. Commits follow
Conventional Commits and releases follow
[STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the index; `CLAUDE.md` carries
the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
