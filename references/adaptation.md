# Adaptation: the seam contract with the host app

The module touches its host in the places listed here and nowhere else. Fill in the right-hand column
before writing a file: a seam that is not named gets hardcoded.

## The seam contract

| Seam | The skill ships | The host supplies |
|---|---|---|
| Domain entities | Canonical names and the rename table below | Its own vocabulary |
| Tenant scope | One `tenantId` on every row, read from the actor | Organisation, workspace, company, or a constant for a single-company app |
| Auth guard | `InvoiceActor` and `getInvoiceActor()` in [service.md](service.md) | Its session lookup and its two permissions, read and write |
| Seller profile | `SellerProfile` and `loadSeller(tenantId)` | The tenant's name, tax identifier, address, bank account, payment terms, exemption basis, currency |
| Data access | `InvoiceStore`, an in-memory store, a Postgres store and migration | Its database client, behind `Queryable` or its own implementation of the interface |
| Clock | `today()` in the business time zone | The time zone, per tenant if tenants differ |
| Audit | `InvoiceAuditEvent` and the `audit` dependency | Its audit log |
| UI primitives | Structure, states and one unstyled editor in [admin-ui.md](admin-ui.md) | Buttons, dialogs, tables, pills, toasts, confirm dialog |
| Styling | Nothing | Its design system |
| Strings | Error codes, status names, and the PDF label tables | Its i18n catalogue, in every locale it has |
| Validation | Checks inside the service | Optionally its schema library at the action boundary |
| Sources | `InvoiceSourceRef` and the two picker rules | The loader for what can be invoiced: sales, orders, bookings |
| Delivery | `pdfDataForDelivery` | Its mailer or outbox, see [operations.md](operations.md) |
| E-invoicing | `EInvoicePort`, and a KSeF implementation in [ksef-bridge.md](ksef-bridge.md) | Nothing, or the `ksef` skill |
| Tests | Three suites and SQL checks | Its runner |

## Probe the host first

Read these before generating anything, and write down what they say:

```bash
cat package.json                      # Next.js major, database client, UI kit, i18n, validation, test runner
cat CLAUDE.md AGENTS.md 2>/dev/null   # the house rules, stated
ls app src/app 2>/dev/null            # where routes live, and whether there is a locale segment
ls proxy.ts middleware.ts src/proxy.ts src/middleware.ts 2>/dev/null
```

Then read two existing feature slices end to end, the closest analogue first, and note: the exact guard
call at the top of a protected route, how a Server Action reports a failure to its form, how a list is
filtered and paged, which component confirms a destructive action, and where a feature's files live. The
module follows those conventions even where its templates do something else. A reviewer should not be able
to tell that it arrived from outside.

Add no dependency the host does not have. The core module needs none. `server-only` ships with Next.js
apps that already use it; the KSeF bridge needs `qrcode`, and the Postgres store needs whatever client the
host already has.

## The domain rename

Decide the vocabulary once, before the first file, and apply it everywhere at once: types, tables,
columns, routes, components, comments and strings. Confirm it with the person who owns the product; it is
the one column that cannot be inferred.

| Canonical | Typical host names | Notes |
|---|---|---|
| `Invoice`, `invoices` | Invoice, Document, Bill | |
| `InvoiceLine`, `invoice_lines` | Line, Item, Position | |
| `tenantId` | `organizationId`, `workspaceId`, `companyId` | Use the host's existing scope column type |
| `seller` | Company, Organisation, Issuer | The snapshot keeps the canonical field names |
| `buyer` | Customer, Client, Contractor | The snapshot on the document |
| `customerId` | `clientId`, `contactId` | The live record the buyer was filled from, if any |
| `source` | Sale, Order, Booking, Visit | `source.kind` holds the host's word |
| `INVOICE_SERIES` `FV`, `FK` | Any prefixes | The series is text; the two must differ |
| `eInvoice` | KSeF, e-invoice | |
| `invoicing` (the service) | Billing, Invoices | |

Do not rename what belongs to a standard: `VatRate` values, the FA(3) element names, `NIP`, `KOD I`.

## Where each seam joins

- **Auth and tenancy.** `getInvoiceActor()` is the only function that reads the session. It returns the
  tenant, the user and two booleans. Map the host's roles onto them there: typically an accountant role
  reads and a billing role writes. Nothing downstream looks at a role.
- **Routes.** The templates use `/invoices` and `app/invoices`. A host with a locale segment or an admin
  area moves them, for example to `app/[lang]/console/invoices`, and passes the same path to
  `revalidatePath`. The PDF route lives under the same guard as the register.
- **Seller profile.** In a multi-tenant app `loadSeller` reads the tenant row. Add the fields the host
  lacks to that table: bank account, payment terms in days, exemption basis, currency. A tenant without a
  name cannot issue, by design.
- **Database.** Run the migration in [postgres.md](postgres.md) through the host's migration tool,
  changing `tenant_id text` and `created_by text` to the host's key types. Keep the triggers and the
  function as raw SQL whatever the ORM.
- **Strings.** The service throws codes. Add one catalogue entry per `InvoiceErrorCode`, per status and
  per label, in every locale. The PDF takes an `InvoicePdfStrings` table: pass the one that matches the
  document's language, which for a tax document is usually the seller's, not the viewer's.
- **Audit.** Wire `audit` to the host's log. The download event is written before the bytes are produced,
  so an audit sink that is down blocks downloads; that is the intended trade.
- **Background work.** The core module has none. The KSeF bridge relies on the `ksef` skill's cron routes.

## Order of work

1. Types and the pure model: [model.md](model.md), [data-model.md](data-model.md). Run `model.test.ts`.
2. The writer and the layout: [pdf-writer.md](pdf-writer.md), [pdf.md](pdf.md). Run `invoice-pdf.test.ts`
   and open one rendered file.
3. The service on the in-memory store: [service.md](service.md). Run `service.test.ts`.
4. The migration and the Postgres store: [postgres.md](postgres.md). Run the database checks.
5. The wiring file with the host's guard, seller profile and clock, then the route and the actions.
6. The admin surface, on the host's components: [admin-ui.md](admin-ui.md).
7. Only then, if wanted, the KSeF bridge: [fa3-xml.md](fa3-xml.md), [ksef-bridge.md](ksef-bridge.md).

Type-check after each step. A rename caught at step 1 is one edit; at step 6 it is every file.

## The non-negotiables

These hold whatever the host. Each is a hard rule in `SKILL.md`.

1. The number is assigned inside the insert that creates the document.
2. A document and its lines are written in one transaction, and so are a correction and the link on its
   original.
3. An issued document never changes: both parties are snapshots and a mistake is fixed by a correction.
4. The tenant and the permissions come from the server-side actor on every call.
5. The server computes every amount and every correction delta from the stored lines.
6. Once the document exists, no later step may fail the request.

## Checklist

- [ ] The host probe is written down and the rename is confirmed
- [ ] `getInvoiceActor`, `loadSeller` and `today` are the host's; nothing else in `lib/invoicing` was edited
- [ ] No new dependency, UI kit or styling arrived with the module
- [ ] Every string is in the host's catalogue and language
- [ ] The host's type-check, lint, tests and build pass
