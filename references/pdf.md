# The PDF: a dependency-free writer, the invoice layout, the download route

The document is rendered on demand from the stored rows: no file is kept, so there is no stored copy to
drift from the data. Rendering is a pure, synchronous function, rows in and bytes out, which is what lets the
tests assert on the raw bytes.

| File | Role |
|---|---|
| `lib/invoicing/pdf.ts` | The writer, in [pdf-writer.md](pdf-writer.md): A4 pages, two built-in fonts, lines, fills, right-aligned and wrapped text |
| `lib/invoicing/invoice-pdf.ts` | The layout: header, parties, line table, VAT table, totals, payment details, the verification code |
| `lib/invoicing/pdf-response.ts` | The download response: `attachment`, `no-store`, a header-safe file name |
| `app/invoices/[id]/pdf/route.ts` | The route: guard, audit, render |

## The layout

Every label comes from a strings table, so the host can print the document in its own language without
touching the layout. Two tables ship: Polish, the default, and English. Long text wraps: a line name takes
as many rows as it needs, party names and addresses wrap inside their column, and the correction reason
wraps across the page. Nothing is truncated, because the name of the goods or service on an invoice is
content, not decoration.

The verification code, when the e-invoicing bridge supplies one, sits bottom-right on the first page. The
line table and the totals block keep clear of it: on that page they stop above the symbol and continue on
the next page.

```typescript
// lib/invoicing/invoice-pdf.ts
import { A4, PdfDoc, wrapText, type PdfPage } from "./pdf";
import type { InvoiceTotals, VatRate, VatSummaryRow } from "./model";

/** A run of horizontally adjacent dark modules, in module coordinates, row 0 at the top. */
export type QrRun = { row: number; col: number; length: number };
export type QrMatrix = { size: number; runs: QrRun[] };

export type InvoicePdfParty = { name: string; taxId: string | null; address: string | null };

export type InvoicePdfLine = {
  position: number;
  name: string;
  qty: number;
  unit: string;
  unitPrice: number;
  vatRate: VatRate;
  net: number;
  vat: number;
  gross: number;
};

export type InvoicePdfData = {
  number: string;
  isCorrection: boolean;
  correctsNumber?: string | null;
  correctionReason?: string | null;
  issueDate: string;
  saleDate: string;
  dueDate: string | null;
  paymentMethod: string | null;
  currency: string;
  seller: InvoicePdfParty & { bankAccount: string | null };
  buyer: InvoicePdfParty;
  lines: InvoicePdfLine[];
  totals: InvoiceTotals;
  summary: VatSummaryRow[];
  vatExemptionBasis?: string | null;
  /** The number the tax system assigned, printed in the payment details. */
  eInvoiceNumber?: string | null;
  /** The verification code and the text under it. Absent when no e-invoicing bridge is wired. */
  verification?: { matrix: QrMatrix; label: string } | null;
};

export type InvoicePdfStrings = {
  /** BCP 47 tag for number and date formatting. */
  locale: string;
  /** Printed after an amount, keyed by ISO currency code. A missing code prints the code itself. */
  currency: Record<string, string>;
  titleBase: string;
  titleCorrection: string;
  numberPrefix: string;
  issueDate: string;
  saleDate: string;
  dueDate: string;
  correctionOf: string;
  correctionReason: string;
  seller: string;
  buyer: string;
  taxId: string;
  colNo: string;
  colName: string;
  colQty: string;
  colUnit: string;
  colUnitPrice: string;
  colVat: string;
  colNet: string;
  colGross: string;
  vatRate: string;
  total: string;
  toPay: string;
  toRefund: string;
  paymentMethod: string;
  /** Labels for the host's payment method codes. A missing code prints the code itself. */
  paymentMethods: Record<string, string>;
  bankAccount: string;
  exemptionBasis: string;
  eInvoiceNumber: string;
  /** Labels for the non-numeric rates. A numeric rate prints as `23%`. */
  rateLabels: Partial<Record<VatRate, string>>;
  footer: string;
};

export const INVOICE_PDF_STRINGS_PL: InvoicePdfStrings = {
  locale: "pl-PL",
  currency: { PLN: "zł" },
  titleBase: "Faktura VAT",
  titleCorrection: "Faktura korygująca",
  numberPrefix: "nr",
  issueDate: "Data wystawienia",
  saleDate: "Data sprzedaży",
  dueDate: "Termin płatności",
  correctionOf: "do faktury nr",
  correctionReason: "Przyczyna korekty",
  seller: "Sprzedawca",
  buyer: "Nabywca",
  taxId: "NIP",
  colNo: "Lp",
  colName: "Nazwa towaru / usługi",
  colQty: "Ilość",
  colUnit: "J.m.",
  colUnitPrice: "Cena brutto",
  colVat: "VAT",
  colNet: "Netto",
  colGross: "Brutto",
  vatRate: "Stawka VAT",
  total: "Razem",
  toPay: "Do zapłaty",
  toRefund: "Do zwrotu",
  paymentMethod: "Forma płatności",
  paymentMethods: { transfer: "Przelew", cash: "Gotówka", card: "Karta", blik: "BLIK" },
  bankAccount: "Rachunek bankowy",
  exemptionBasis: "Podstawa zwolnienia z VAT",
  eInvoiceNumber: "KSeF",
  rateLabels: { zw: "zw.", np: "np." },
  footer: "Dokument wystawiony elektronicznie.",
};

export const INVOICE_PDF_STRINGS_EN: InvoicePdfStrings = {
  locale: "en-GB",
  currency: { PLN: "PLN" },
  titleBase: "VAT invoice",
  titleCorrection: "Correcting invoice",
  numberPrefix: "no.",
  issueDate: "Issue date",
  saleDate: "Sale date",
  dueDate: "Payment due",
  correctionOf: "to invoice no.",
  correctionReason: "Reason for correction",
  seller: "Seller",
  buyer: "Buyer",
  taxId: "Tax ID",
  colNo: "No.",
  colName: "Goods or service",
  colQty: "Qty",
  colUnit: "Unit",
  colUnitPrice: "Gross price",
  colVat: "VAT",
  colNet: "Net",
  colGross: "Gross",
  vatRate: "VAT rate",
  total: "Total",
  toPay: "To pay",
  toRefund: "To refund",
  paymentMethod: "Payment method",
  paymentMethods: { transfer: "Bank transfer", cash: "Cash", card: "Card", blik: "BLIK" },
  bankAccount: "Bank account",
  exemptionBasis: "Basis of VAT exemption",
  eInvoiceNumber: "E-invoice number",
  rateLabels: { zw: "exempt", np: "n/a" },
  footer: "Issued electronically.",
};

const MARGIN = 48;
const RIGHT = A4.width - MARGIN;
const BOTTOM = MARGIN + 6;

/** Column anchors: a left x for text columns, a right edge for numeric ones. */
const COL = {
  no: MARGIN,
  name: MARGIN + 24,
  qty: 300,
  unit: 306,
  unitPrice: 384,
  vat: 410,
  net: 478,
  gross: RIGHT,
} as const;

/** The VAT table under the lines: a rate label, then net, VAT and gross, each in its own column. */
const SUM = { rate: 262, net: 392, vat: 468, gross: RIGHT } as const;

/** The name column ends where the widest quantity begins. */
const NAME_WIDTH = COL.qty - 36 - COL.name;
const NAME_LEADING = 10;

/** Verification code footprint in points, and the y of its top edge. */
const QR_SIZE = 84;
const QR_BOTTOM = MARGIN + 14; // clears the footer text at MARGIN - 12
const QR_TOP = QR_BOTTOM + QR_SIZE;

function tableHeader(page: PdfPage, y: number, s: InvoicePdfStrings): number {
  page.rect(MARGIN - 4, y - 4, RIGHT - MARGIN + 8, 16);
  page.text(COL.no, y, s.colNo, { font: "bold", size: 8 });
  page.text(COL.name, y, s.colName, { font: "bold", size: 8 });
  page.text(COL.qty, y, s.colQty, { font: "bold", size: 8, align: "right" });
  page.text(COL.unit, y, s.colUnit, { font: "bold", size: 8 });
  page.text(COL.unitPrice, y, s.colUnitPrice, { font: "bold", size: 8, align: "right" });
  page.text(COL.vat, y, s.colVat, { font: "bold", size: 8, align: "right" });
  page.text(COL.net, y, s.colNet, { font: "bold", size: 8, align: "right" });
  page.text(COL.gross, y, s.colGross, { font: "bold", size: 8, align: "right" });
  return y - 18;
}

/**
 * Drawn as vector module runs, not an embedded image: the writer has no image support and needs none, and
 * vectors stay sharp at any zoom. The page is white, so the quiet zone is the empty corner around it.
 */
function drawVerification(page: PdfPage, matrix: QrMatrix, label: string): void {
  const cell = QR_SIZE / matrix.size;
  const originX = RIGHT - QR_SIZE;
  page.rects(
    matrix.runs.map((r) => ({
      x: originX + r.col * cell,
      // QR row 0 is the top row and PDF y grows upward, so the row is flipped.
      y: QR_BOTTOM + (matrix.size - 1 - r.row) * cell,
      w: r.length * cell,
      h: cell,
    })),
    0,
  );
  page.text(RIGHT, QR_BOTTOM - 9, label, { size: 6.5, align: "right", gray: 0.35 });
}

type PartyRow = { text: string; head: boolean };

function partyRows(party: InvoicePdfParty, s: InvoicePdfStrings, width: number): PartyRow[] {
  const rows: PartyRow[] = wrapText(party.name, 10, width).map((text) => ({ text, head: true }));
  if (party.address) for (const text of wrapText(party.address, 9, width)) rows.push({ text, head: false });
  if (party.taxId) rows.push({ text: `${s.taxId} ${party.taxId}`, head: false });
  return rows;
}

export function renderInvoicePdf(
  data: InvoicePdfData,
  strings: InvoicePdfStrings = INVOICE_PDF_STRINGS_PL,
): Uint8Array {
  const s = strings;
  const currency = s.currency[data.currency] ?? data.currency;
  const money = (n: number): string =>
    `${n.toLocaleString(s.locale, { minimumFractionDigits: 2, maximumFractionDigits: 2 })} ${currency}`;
  const qty = (n: number): string =>
    n.toLocaleString(s.locale, { minimumFractionDigits: 0, maximumFractionDigits: 2 });
  const date = (iso: string): string =>
    new Intl.DateTimeFormat(s.locale, { day: "2-digit", month: "2-digit", year: "numeric", timeZone: "UTC" }).format(
      new Date(`${iso.slice(0, 10)}T12:00:00Z`),
    );
  const rate = (r: VatRate): string => s.rateLabels[r] ?? `${r}%`;

  const doc = new PdfDoc();
  let page = doc.addPage();
  const firstPage = page;
  const qr = data.verification ?? null;
  /** The lowest y a table row may use: above the verification code on the page that carries it. */
  const floor = (p: PdfPage): number => (qr && p === firstPage ? QR_TOP + 14 : 130);
  let y = A4.height - MARGIN - 8;
  /** A reason or an address has no length limit: a row that would cross the floor starts a new page. */
  const room = (height: number): void => {
    if (y - height < floor(page)) {
      page = doc.addPage();
      y = A4.height - MARGIN;
    }
  };

  // Header
  page.text(MARGIN, y, data.isCorrection ? s.titleCorrection : s.titleBase, { font: "bold", size: 17 });
  page.text(RIGHT, y, `${s.issueDate}: ${date(data.issueDate)}`, { size: 9, align: "right" });
  y -= 16;
  page.text(MARGIN, y, `${s.numberPrefix} ${data.number}`, { font: "bold", size: 12 });
  page.text(RIGHT, y, `${s.saleDate}: ${date(data.saleDate)}`, { size: 9, align: "right" });
  y -= 13;
  if (data.dueDate) page.text(RIGHT, y, `${s.dueDate}: ${date(data.dueDate)}`, { size: 9, align: "right" });
  if (data.isCorrection && data.correctsNumber) {
    page.text(MARGIN, y, `${s.correctionOf} ${data.correctsNumber}`, { size: 10 });
  }
  y -= 14;
  if (data.isCorrection && data.correctionReason) {
    for (const row of wrapText(`${s.correctionReason}: ${data.correctionReason}`, 9, RIGHT - MARGIN)) {
      room(12);
      page.text(MARGIN, y, row, { size: 9, gray: 0.25 });
      y -= 12;
    }
    y -= 2;
  }
  y -= 8;

  // Parties, side by side, each wrapped inside its own column
  const colB = A4.width / 2 + 10;
  const partyWidth = RIGHT - colB;
  room(25);
  page.text(MARGIN, y, s.seller, { font: "bold", size: 8, gray: 0.4 });
  page.text(colB, y, s.buyer, { font: "bold", size: 8, gray: 0.4 });
  y -= 13;
  const sellerRows = partyRows(data.seller, s, partyWidth);
  const buyerRows = partyRows(data.buyer, s, partyWidth);
  for (let i = 0; i < Math.max(sellerRows.length, buyerRows.length); i++) {
    const left = sellerRows[i];
    const right = buyerRows[i];
    room(12);
    if (left) page.text(MARGIN, y, left.text, left.head ? { font: "bold", size: 10 } : { size: 9, gray: 0.2 });
    if (right) page.text(colB, y, right.text, right.head ? { font: "bold", size: 10 } : { size: 9, gray: 0.2 });
    y -= 12;
  }
  y -= 12;

  // Lines
  y = tableHeader(page, y, s);
  for (const l of data.lines) {
    const nameRows = wrapText(l.name, 8.5, NAME_WIDTH);
    const extra = (nameRows.length - 1) * NAME_LEADING;
    if (y - extra < floor(page)) {
      page = doc.addPage();
      y = tableHeader(page, A4.height - MARGIN, s);
    }
    page.text(COL.no, y, String(l.position), { size: 8.5, gray: 0.3 });
    nameRows.forEach((row, i) => page.text(COL.name, y - i * NAME_LEADING, row, { size: 8.5 }));
    page.text(COL.qty, y, qty(l.qty), { size: 8.5, align: "right" });
    page.text(COL.unit, y, l.unit, { size: 8.5 });
    page.text(COL.unitPrice, y, money(l.unitPrice), { size: 8.5, align: "right" });
    page.text(COL.vat, y, rate(l.vatRate), { size: 8.5, align: "right" });
    page.text(COL.net, y, money(l.net), { size: 8.5, align: "right" });
    page.text(COL.gross, y, money(l.gross), { size: 8.5, align: "right" });
    page.line(MARGIN - 4, y - extra - 5, RIGHT + 4, y - extra - 5, 0.5, 0.88);
    y -= extra + 15;
  }
  y -= 8;

  // Payment details are measured first, so the whole closing block moves to a new page together.
  const detailWidth = RIGHT - MARGIN - (qr ? QR_SIZE + 16 : 0);
  const details: string[] = [];
  if (data.paymentMethod) {
    details.push(`${s.paymentMethod}: ${s.paymentMethods[data.paymentMethod] ?? data.paymentMethod}`);
  }
  if (data.seller.bankAccount) details.push(`${s.bankAccount}: ${data.seller.bankAccount}`);
  if (data.vatExemptionBasis) {
    details.push(...wrapText(`${s.exemptionBasis}: ${data.vatExemptionBasis}`, 9, detailWidth));
  }
  if (data.eInvoiceNumber) details.push(`${s.eInvoiceNumber}: ${data.eInvoiceNumber}`);

  const summaryHeight = 13 + data.summary.length * 12 + 8;
  const blockHeight = summaryHeight + 44 + details.length * 12;
  if (y - summaryHeight < floor(page) || y - blockHeight < BOTTOM) {
    page = doc.addPage();
    y = A4.height - MARGIN;
  }

  // VAT table and totals
  page.text(SUM.rate, y, s.vatRate, { font: "bold", size: 8, gray: 0.4 });
  page.text(SUM.net, y, s.colNet, { font: "bold", size: 8, gray: 0.4, align: "right" });
  page.text(SUM.vat, y, s.colVat, { font: "bold", size: 8, gray: 0.4, align: "right" });
  page.text(SUM.gross, y, s.colGross, { font: "bold", size: 8, gray: 0.4, align: "right" });
  y -= 13;
  for (const row of data.summary) {
    page.text(SUM.rate, y, rate(row.vatRate), { size: 8.5 });
    page.text(SUM.net, y, money(row.net), { size: 8.5, align: "right" });
    page.text(SUM.vat, y, money(row.vat), { size: 8.5, align: "right" });
    page.text(SUM.gross, y, money(row.gross), { size: 8.5, align: "right" });
    y -= 12;
  }
  page.line(SUM.rate - 4, y + 4, RIGHT + 4, y + 4, 0.7, 0.6);
  y -= 8;
  page.text(SUM.rate, y, s.total, { font: "bold", size: 9 });
  page.text(SUM.net, y, money(data.totals.net), { font: "bold", size: 9, align: "right" });
  page.text(SUM.vat, y, money(data.totals.vat), { font: "bold", size: 9, align: "right" });
  page.text(SUM.gross, y, money(data.totals.gross), { font: "bold", size: 9, align: "right" });
  y -= 24;

  // A negative total is a refund: a correction that lowers the amount owes the buyer money.
  const closing = data.totals.gross < 0 ? s.toRefund : s.toPay;
  page.text(MARGIN, y, `${closing}: ${money(Math.abs(data.totals.gross))}`, { font: "bold", size: 13 });
  y -= 20;

  for (const row of details) {
    page.text(MARGIN, y, row, { size: 9, gray: 0.2 });
    y -= 12;
  }

  page.text(MARGIN, MARGIN - 12, s.footer, { size: 7.5, gray: 0.5 });

  if (qr) drawVerification(firstPage, qr.matrix, qr.label);

  return doc.build();
}
```

## The response and the route

```typescript
// lib/invoicing/pdf-response.ts

/**
 * A rendered PDF as a download. `no-store` because the document carries a buyer's details, and a cached
 * copy in a shared browser or an intermediary outlives the permission check that produced it.
 * `attachment` keeps the viewer from rendering it inline in a tab somebody leaves open.
 */
export function pdfResponse(bytes: Uint8Array, filename: string): Response {
  // An invoice number contains slashes and may contain anything a series prefix does. A header value
  // takes none of that, so everything outside a conservative set becomes an underscore.
  const safe = filename.replace(/[^A-Za-z0-9._-]+/g, "_");
  return new Response(Buffer.from(bytes), {
    headers: {
      "Content-Type": "application/pdf",
      "Content-Disposition": `attachment; filename="${safe}"`,
      "Cache-Control": "no-store",
    },
  });
}
```

The route resolves the actor, asks the service for the render-ready data, and renders. The service checks
the read permission and writes the audit event before it returns the data, so a download that cannot be
logged does not happen. The route path is the host's; `invoicing` and `getInvoiceActor` come from the
wiring file in [service.md](service.md).

```typescript
// app/invoices/[id]/pdf/route.ts
import { getInvoiceActor, invoicing } from "@/lib/invoicing";
import { renderInvoicePdf } from "@/lib/invoicing/invoice-pdf";
import { InvoiceError } from "@/lib/invoicing/model";
import { pdfResponse } from "@/lib/invoicing/pdf-response";

export async function GET(
  _request: Request,
  { params }: { params: Promise<{ id: string }> },
): Promise<Response> {
  const actor = await getInvoiceActor();
  if (!actor) return Response.json({ error: "unauthorized" }, { status: 401 });

  const { id } = await params;
  try {
    const data = await invoicing.pdfData(actor, id);
    // Out of the actor's tenant reads the same as missing: the route never confirms an id exists.
    if (!data) return Response.json({ error: "not_found" }, { status: 404 });
    return pdfResponse(renderInvoicePdf(data), `${data.number}.pdf`);
  } catch (error) {
    if (error instanceof InvoiceError && error.code === "forbidden") {
      return Response.json({ error: "forbidden" }, { status: 403 });
    }
    throw error;
  }
}
```

In Next.js 15 and 16 `params` is a promise, as written. In 13 and 14 it is a plain object; `await` on it
still works, and the type annotation is the only line to change.

## Traps

- **A number formatted for a locale contains no-break spaces.** `toLocaleString("pl-PL")` separates
  thousands with U+00A0 and other locales with U+202F. Both are in `ASCII_FALLBACK`; a new locale with a
  different separator needs an entry, or the separator prints as `?`.
- **Right-aligned bold text is measured with regular metrics.** Digits have the same width in both faces,
  so amounts align. Bold letters are slightly wider, so a bold right-aligned label overshoots its edge by
  under a point. It is not worth a second metrics table.
- **The verification code encodes a URL whose hash is of exact bytes.** The layout only draws the matrix it
  is given. Where the matrix comes from, and why it is never recomputed at render time, is in
  [ksef-bridge.md](ksef-bridge.md).
- **The amount columns are sized for six digits before the decimal point.** A line amount of a million or
  more, with a three-letter currency label, reaches into the column to its left. The VAT table under the
  lines has room for seven. A host that invoices larger line amounts narrows the name column or drops the
  currency label from the line table.
- **Formatting depends on the runtime's ICU data.** Node.js ships full ICU by default. A runtime built with
  small ICU prints every locale as `en-US`; the tests in [testing.md](testing.md) catch that.

## Checklist

- [ ] Labels come from an `InvoicePdfStrings` table, in the host's language
- [ ] No text is truncated: a long line name wraps and a long party name wraps
- [ ] The route returns 401 without an actor, 403 without the read permission, 404 outside the tenant
- [ ] The response is `attachment` and `no-store`
- [ ] The download is audited before the bytes leave
