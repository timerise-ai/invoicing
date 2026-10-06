# The pure model: VAT math, numbering format, correction deltas, validation

Everything in this file is a pure function: rows in, rows out, no clock, no database, no framework. It is the
part of the module the tests in [testing.md](testing.md) hold, and the part every other reference imports.

| Export | What it decides |
|---|---|
| `VAT_RATES`, `VatRate`, `vatPercent` | The closed rate vocabulary and its numeric percent |
| `roundMoney` | Two-decimal rounding, half away from zero, so a negated amount is the exact negative |
| `computeLine`, `computeTotals`, `vatSummary` | Net and VAT derived from a gross line, document totals, the per-rate table |
| `formatInvoiceNumber` | `<series>/<year>/<seq>`, padded to five digits and never truncated |
| `paymentDueDate` | Issue date plus payment terms, in calendar days |
| `buildCorrectionLines` | Signed deltas between the issued lines and the corrected state |
| `validateLines`, `isIsoDate` | The input rules the service applies before a number is reserved |
| `InvoiceError`, `InvoiceErrorCode` | One error type whose code is an i18n key |
| `normalizeTaxId`, `isValidNip`, `formatNip`, `sameNip` | The Polish tax identifier, the default validator |

## Prices are gross

**A line carries a gross unit price, and net and VAT are derived from the gross line total.** That is the
right model wherever the price list is what the customer pays: retail, services, a point of sale. The line
gross is `qty * unitPrice` rounded to two places, the net is `gross / (1 + rate)` rounded, and the VAT is the
difference, so the three always add up on every line. Document totals and the per-rate table are sums of line
amounts, so they also agree with each other. A net-priced document is a different model and is a design in
[operations.md](operations.md), not a flag here.

## The module

```typescript
// lib/invoicing/model.ts

/** The closed rate vocabulary. `zw` is exempt, `np` is outside the scope of VAT; both carry 0%. */
export const VAT_RATES = ["23", "8", "5", "0", "zw", "np"] as const;
export type VatRate = (typeof VAT_RATES)[number];

const VAT_PERCENT: Record<VatRate, number> = { "23": 23, "8": 8, "5": 5, "0": 0, zw: 0, np: 0 };

export function isVatRate(value: string): value is VatRate {
  return (VAT_RATES as readonly string[]).includes(value);
}

export function vatPercent(rate: VatRate): number {
  return VAT_PERCENT[rate];
}

/** Base documents and corrections number in separate series. The host may rename the prefixes. */
export const INVOICE_SERIES = { base: "FV", correction: "FK" } as const;
export type InvoiceKind = keyof typeof INVOICE_SERIES;

export type InvoiceErrorCode =
  | "forbidden"
  | "not_found"
  | "seller_incomplete"
  | "buyer_name_required"
  | "tax_id_invalid"
  | "date_invalid"
  | "no_lines"
  | "line_name_required"
  | "line_qty_invalid"
  | "line_price_invalid"
  | "vat_rate_invalid"
  | "exemption_basis_required"
  | "source_already_invoiced"
  | "not_correctable"
  | "already_corrected"
  | "correction_reason_required"
  | "correction_empty"
  | "already_paid"
  | "e_invoice_unsendable";

/** The code is an i18n key; `params` carries the values a message interpolates (a line number, a rate). */
export class InvoiceError extends Error {
  readonly code: InvoiceErrorCode;
  readonly params: Readonly<Record<string, string | number>>;

  constructor(code: InvoiceErrorCode, params: Record<string, string | number> = {}) {
    super(code);
    this.name = "InvoiceError";
    this.code = code;
    this.params = params;
  }
}

export type InvoiceLineInput = {
  name: string;
  /** Signed on a correction delta: a negation row carries a negative quantity. */
  qty: number;
  unit: string;
  /** Gross unit price. */
  unitPrice: number;
  vatRate: VatRate;
};

export type InvoiceLine = InvoiceLineInput & { net: number; vat: number; gross: number };
export type InvoiceTotals = { net: number; vat: number; gross: number };
export type VatSummaryRow = { vatRate: VatRate } & InvoiceTotals;

/**
 * Two decimals, half away from zero. `Math.round` alone rounds half toward positive infinity, so -0.125
 * would become -0.12 while 0.125 becomes 0.13, and a correction that negates a line would leave a
 * one-unit residue. Rounding the magnitude and restoring the sign makes roundMoney(-x) === -roundMoney(x).
 */
export function roundMoney(n: number): number {
  const magnitude = Math.round((Math.abs(n) + Number.EPSILON) * 100) / 100;
  if (magnitude === 0) return 0; // never -0, which fails strict equality checks against 0
  return n < 0 ? -magnitude : magnitude;
}

/** Net and VAT from the gross line total. Signed input gives signed output. */
export function computeLine(line: InvoiceLineInput): InvoiceLine {
  if (!isVatRate(line.vatRate)) throw new InvoiceError("vat_rate_invalid", { rate: String(line.vatRate) });
  const gross = roundMoney(line.qty * line.unitPrice);
  const net = roundMoney(gross / (1 + vatPercent(line.vatRate) / 100));
  const vat = roundMoney(gross - net);
  return { ...line, gross, net, vat };
}

export function computeTotals(lines: readonly InvoiceTotals[]): InvoiceTotals {
  return {
    net: roundMoney(lines.reduce((sum, l) => sum + l.net, 0)),
    vat: roundMoney(lines.reduce((sum, l) => sum + l.vat, 0)),
    gross: roundMoney(lines.reduce((sum, l) => sum + l.gross, 0)),
  };
}

/** Per-rate totals for the document's VAT table, in the order of VAT_RATES, not of appearance. */
export function vatSummary(lines: readonly InvoiceLine[]): VatSummaryRow[] {
  const byRate = new Map<VatRate, InvoiceTotals>();
  for (const l of lines) {
    const acc = byRate.get(l.vatRate) ?? { net: 0, vat: 0, gross: 0 };
    byRate.set(l.vatRate, {
      net: roundMoney(acc.net + l.net),
      vat: roundMoney(acc.vat + l.vat),
      gross: roundMoney(acc.gross + l.gross),
    });
  }
  const rows: VatSummaryRow[] = [];
  for (const rate of VAT_RATES) {
    const totals = byRate.get(rate);
    if (totals) rows.push({ vatRate: rate, ...totals });
  }
  return rows;
}

/**
 * `<series>/<year>/<seq>`, the sequence padded to five digits. `padStart` only pads, so a sequence past
 * 99999 keeps every digit. The database stamps the number with the same rule; see postgres.md.
 */
export function formatInvoiceNumber(series: string, year: number, seq: number): string {
  return `${series}/${year}/${String(Math.max(0, Math.trunc(seq))).padStart(5, "0")}`;
}

export function isIsoDate(value: string): boolean {
  if (!/^\d{4}-\d{2}-\d{2}$/.test(value)) return false;
  const d = new Date(`${value}T12:00:00Z`);
  return !Number.isNaN(d.getTime()) && d.toISOString().slice(0, 10) === value;
}

/** Issue date plus the seller's payment terms. Noon UTC keeps the arithmetic clear of DST edges. */
export function paymentDueDate(issueDate: string, termsDays: number): string {
  const d = new Date(`${issueDate.slice(0, 10)}T12:00:00Z`);
  d.setUTCDate(d.getUTCDate() + Math.max(0, Math.trunc(termsDays)));
  return d.toISOString().slice(0, 10);
}

function sameLine(a: InvoiceLineInput, b: InvoiceLineInput): boolean {
  return (
    a.name === b.name &&
    a.qty === b.qty &&
    a.unit === b.unit &&
    a.unitPrice === b.unitPrice &&
    a.vatRate === b.vatRate
  );
}

/**
 * Signed correction deltas, position by position: a changed line gives its negation plus the corrected
 * line, a removed line gives the negation only, an added line passes through, and an unchanged line gives
 * nothing. An identical document therefore corrects to an empty set, which the service rejects.
 */
export function buildCorrectionLines(
  original: readonly InvoiceLineInput[],
  corrected: readonly InvoiceLineInput[],
): InvoiceLineInput[] {
  const deltas: InvoiceLineInput[] = [];
  const max = Math.max(original.length, corrected.length);
  for (let i = 0; i < max; i++) {
    const before = original[i];
    const after = corrected[i];
    if (before && after && sameLine(before, after)) continue;
    if (before) deltas.push({ ...before, qty: -before.qty });
    if (after) deltas.push({ ...after });
  }
  return deltas;
}

/**
 * The line rules, applied before a number is reserved. A base document takes positive quantities only;
 * a delta set may carry negative ones. `line` in the params is the 1-based position for the message.
 */
export function validateLines(lines: readonly InvoiceLineInput[], kind: InvoiceKind): void {
  if (lines.length === 0) throw new InvoiceError(kind === "base" ? "no_lines" : "correction_empty");
  lines.forEach((l, i) => {
    const line = i + 1;
    if (!l.name.trim()) throw new InvoiceError("line_name_required", { line });
    if (!isVatRate(l.vatRate)) throw new InvoiceError("vat_rate_invalid", { line, rate: String(l.vatRate) });
    if (!Number.isFinite(l.qty) || l.qty === 0 || (kind === "base" && l.qty < 0)) {
      throw new InvoiceError("line_qty_invalid", { line });
    }
    if (!Number.isFinite(l.unitPrice) || l.unitPrice < 0) {
      throw new InvoiceError("line_price_invalid", { line });
    }
  });
}
```

## Tax identifier

The default validator is the Polish NIP: ten digits, the last a weighted checksum of the first nine. It runs
at the form and again in the service, because an identifier with a wrong checksum is otherwise reported by
the tax system long after the document is numbered and in the buyer's hands. A host in another jurisdiction
replaces `isValidNip` through the `validateTaxId` dependency in [service.md](service.md) and keeps
`normalizeTaxId`.

```typescript
// lib/invoicing/tax-id.ts

const NIP_WEIGHTS = [6, 5, 7, 2, 3, 4, 5, 6, 7] as const;

/** Strip the separators people type: "123-456-78-19", "123 456 78 19". */
export function normalizeTaxId(value: string): string {
  return value.replace(/[\s-]/g, "");
}

export function isValidNip(value: string): boolean {
  const nip = normalizeTaxId(value);
  if (!/^\d{10}$/.test(nip)) return false;
  // Ten identical digits pass the checksum by arithmetic accident and are never issued.
  if (/^(\d)\1{9}$/.test(nip)) return false;
  const sum = NIP_WEIGHTS.reduce((acc, weight, i) => acc + weight * Number(nip.charAt(i)), 0);
  const checksum = sum % 11;
  // A remainder of 10 cannot be written as one digit, so no valid NIP produces it.
  return checksum !== 10 && checksum === Number(nip.charAt(9));
}

/** Display form, 123-456-78-19. Returns the input unchanged when it is not ten digits. */
export function formatNip(value: string): string {
  const nip = normalizeTaxId(value);
  if (!/^\d{10}$/.test(nip)) return value;
  return `${nip.slice(0, 3)}-${nip.slice(3, 6)}-${nip.slice(6, 8)}-${nip.slice(8)}`;
}

/** Whether two identifiers name the same taxpayer however they were typed. Empty is never the same. */
export function sameNip(a: string | null | undefined, b: string | null | undefined): boolean {
  const left = normalizeTaxId(a ?? "");
  return left !== "" && left === normalizeTaxId(b ?? "");
}
```

## What the rules above settle

- **Rounding is symmetric.** A correction negates a line by flipping the sign of its quantity. With a
  fractional quantity the gross can land on a half unit, and only half-away-from-zero rounding makes the
  negation cancel the original to the last digit. `model.test.ts` walks a range of fractional quantities.
- **Summary order is fixed.** The per-rate table follows `VAT_RATES`, so two invoices with the same rates
  print the same table whatever order the lines were typed in.
- **Deltas are positional.** Removing the first of three lines shifts the rest, so every line shows as
  changed. The amounts still sum correctly. An editor that keeps removed rows in place, struck through,
  gives a shorter correction; see [admin-ui.md](admin-ui.md).
- **Validation throws codes, not sentences.** The host maps `InvoiceError.code` to its own strings, in its
  own language. Nothing in `lib/invoicing` contains a user-facing sentence except the PDF label tables.
