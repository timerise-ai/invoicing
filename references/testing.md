# Testing

Three test files travel with the core module and two with the KSeF bridge. They run under vitest, and
under `bun test`, which rewrites the `vitest` import to its own runner. None needs a database, a network or
a browser: the service suite runs against the in-memory store, which enforces the same rules as the
schema. The database has its own checks, in [testing-service.md](testing-service.md).

| File | Tests | Holds |
|---|---|---|
| `lib/invoicing/model.test.ts` | 20 | VAT math, symmetric rounding, the number format, correction deltas, validation, the tax identifier |
| `lib/invoicing/invoice-pdf.test.ts` | 16 | The writer's file structure and encoding, wrapping, the layout, the verification code's position |
| `lib/invoicing/service.test.ts` | 21 | In [testing-service.md](testing-service.md): permissions, tenant scope, numbering, snapshots, corrections, payment, paging, audit, the e-invoice hand-off |
| `lib/invoicing/ksef/fa3-xml.test.ts` | 9 | In [fa3-xml.md](fa3-xml.md) |
| `lib/invoicing/ksef/bridge.test.ts` | 6 | In [ksef-bridge.md](ksef-bridge.md) |

```bash
npx vitest run lib/invoicing        # or: bun test lib/invoicing
```

Wire them to `npm test` in the host. The three core files import nothing outside `lib/invoicing`.

## The model

```typescript
// lib/invoicing/model.test.ts
import { describe, expect, it } from "vitest";
import {
  InvoiceError,
  buildCorrectionLines,
  computeLine,
  computeTotals,
  formatInvoiceNumber,
  isIsoDate,
  paymentDueDate,
  roundMoney,
  validateLines,
  vatSummary,
  type InvoiceErrorCode,
  type InvoiceLineInput,
  type VatRate,
} from "./model";
import { formatNip, isValidNip, normalizeTaxId, sameNip } from "./tax-id";

const line = (over: Partial<InvoiceLineInput> = {}): InvoiceLineInput => ({
  name: "Service",
  qty: 1,
  unit: "szt.",
  unitPrice: 123,
  vatRate: "23",
  ...over,
});

function codeOf(run: () => unknown): InvoiceErrorCode | null {
  try {
    run();
    return null;
  } catch (error) {
    if (error instanceof InvoiceError) return error.code;
    throw error;
  }
}

describe("formatInvoiceNumber", () => {
  it("pads the sequence to five digits", () => {
    expect(formatInvoiceNumber("FV", 2026, 12)).toBe("FV/2026/00012");
    expect(formatInvoiceNumber("FK", 2026, 1)).toBe("FK/2026/00001");
  });

  it("keeps every digit of a sequence past 99999", () => {
    expect(formatInvoiceNumber("FV", 2026, 21921241)).toBe("FV/2026/21921241");
    expect(formatInvoiceNumber("FV", 2026, 21921241)).not.toBe(formatInvoiceNumber("FV", 2026, 21921242));
  });
});

describe("computeLine", () => {
  it("derives net and VAT from the gross line total", () => {
    expect(computeLine(line())).toMatchObject({ gross: 123, net: 100, vat: 23 });
    expect(computeLine(line({ qty: 2, unitPrice: 54, vatRate: "8" }))).toMatchObject({ gross: 108, net: 100, vat: 8 });
  });

  it("carries no VAT on exempt and out-of-scope rates", () => {
    expect(computeLine(line({ vatRate: "zw", unitPrice: 200 }))).toMatchObject({ net: 200, vat: 0 });
    expect(computeLine(line({ vatRate: "np", unitPrice: 200 }))).toMatchObject({ net: 200, vat: 0 });
  });

  it("keeps the sign of a correction delta", () => {
    expect(computeLine(line({ qty: -1 }))).toMatchObject({ gross: -123, net: -100, vat: -23 });
  });

  it("rejects a rate outside the vocabulary", () => {
    expect(codeOf(() => computeLine(line({ vatRate: "19" as VatRate })))).toBe("vat_rate_invalid");
  });

  it("makes a negated line the exact negative, for fractional quantities too", () => {
    for (const qty of [0.25, 0.5, 1.5, 2.5, 30]) {
      for (let cents = 1; cents <= 2000; cents++) {
        const positive = computeLine(line({ qty, unitPrice: cents / 100 }));
        const negative = computeLine(line({ qty: -qty, unitPrice: cents / 100 }));
        expect(positive.gross + negative.gross).toBe(0);
        expect(positive.net + negative.net).toBe(0);
        expect(positive.vat + negative.vat).toBe(0);
      }
    }
  });
});

describe("roundMoney", () => {
  it("rounds half away from zero and never returns negative zero", () => {
    expect(roundMoney(0.125)).toBe(0.13);
    expect(roundMoney(-0.125)).toBe(-0.13);
    expect(Object.is(roundMoney(-0.001), 0)).toBe(true);
  });
});

describe("totals and the VAT table", () => {
  const lines = [
    computeLine(line({ name: "Paint", qty: 2, unitPrice: 54, vatRate: "8" })),
    computeLine(line()),
    computeLine(line({ name: "Exempt", unitPrice: 50, vatRate: "zw" })),
  ];

  it("sums the document", () => {
    expect(computeTotals(lines)).toEqual({ net: 250, vat: 31, gross: 281 });
  });

  it("groups by rate in the order of the vocabulary, not of the lines", () => {
    const rows = vatSummary(lines);
    expect(rows.map((r) => r.vatRate)).toEqual(["23", "8", "zw"]);
    expect(rows[0]).toMatchObject({ net: 100, vat: 23, gross: 123 });
  });
});

describe("paymentDueDate and isIsoDate", () => {
  it("adds calendar days across a month end", () => {
    expect(paymentDueDate("2026-07-25", 14)).toBe("2026-08-08");
    expect(paymentDueDate("2026-07-06", 0)).toBe("2026-07-06");
  });

  it("accepts a real date and nothing else", () => {
    expect(isIsoDate("2026-02-28")).toBe(true);
    expect(isIsoDate("2026-02-30")).toBe(false);
    expect(isIsoDate("2026-13-01")).toBe(false);
    expect(isIsoDate("26-02-28")).toBe(false);
  });
});

describe("buildCorrectionLines", () => {
  const original = [line(), line({ name: "Paint", qty: 2, unitPrice: 54, vatRate: "8" })];

  it("yields nothing for an unchanged document", () => {
    expect(buildCorrectionLines(original, original)).toEqual([]);
  });

  it("yields the negation and the corrected line for a changed one", () => {
    const deltas = buildCorrectionLines(original, [line({ unitPrice: 100 }), line({ name: "Paint", qty: 2, unitPrice: 54, vatRate: "8" })]);
    expect(deltas).toHaveLength(2);
    expect(deltas[0]).toMatchObject({ qty: -1, unitPrice: 123 });
    expect(deltas[1]).toMatchObject({ qty: 1, unitPrice: 100 });
  });

  it("yields the negation only for a removed line and passes an added one through", () => {
    expect(buildCorrectionLines(original, [line()])).toMatchObject([{ name: "Paint", qty: -2 }]);
    expect(buildCorrectionLines(original, [...original, line({ name: "Brush" })])).toMatchObject([{ name: "Brush", qty: 1 }]);
  });

  it("reverses the whole document when everything is withdrawn", () => {
    const before = computeTotals(original.map(computeLine));
    const after = computeTotals(buildCorrectionLines(original, []).map(computeLine));
    expect(after).toEqual({ net: -before.net, vat: -before.vat, gross: -before.gross });
  });
});

describe("validateLines", () => {
  it("names the rule and the line", () => {
    expect(codeOf(() => validateLines([], "base"))).toBe("no_lines");
    expect(codeOf(() => validateLines([], "correction"))).toBe("correction_empty");
    expect(codeOf(() => validateLines([line({ name: "  " })], "base"))).toBe("line_name_required");
    expect(codeOf(() => validateLines([line({ qty: 0 })], "base"))).toBe("line_qty_invalid");
    expect(codeOf(() => validateLines([line({ qty: Number.NaN })], "base"))).toBe("line_qty_invalid");
    expect(codeOf(() => validateLines([line({ unitPrice: -1 })], "base"))).toBe("line_price_invalid");
  });

  it("takes a negative quantity on a delta and refuses it on a base document", () => {
    expect(codeOf(() => validateLines([line({ qty: -1 })], "base"))).toBe("line_qty_invalid");
    expect(codeOf(() => validateLines([line({ qty: -1 })], "correction"))).toBeNull();
  });
});

describe("tax identifier", () => {
  it("validates the checksum whatever the separators", () => {
    expect(isValidNip("526-587-76-35")).toBe(true);
    expect(isValidNip("5265877636")).toBe(false);
    expect(isValidNip("0000000000")).toBe(false);
    expect(normalizeTaxId("526 587 76 35")).toBe("5265877635");
    expect(formatNip("5265877635")).toBe("526-587-76-35");
  });

  it("compares two spellings of one identifier and never matches empty", () => {
    expect(sameNip("526-587-76-35", "5265877635")).toBe(true);
    expect(sameNip("", "")).toBe(false);
    expect(sameNip(null, "5265877635")).toBe(false);
  });
});
```

## The PDF

Content streams are uncompressed, so the assertions read the raw bytes. The verification code is given as
a hand-made matrix, which keeps this file free of the QR dependency.

```typescript
// lib/invoicing/invoice-pdf.test.ts
import { describe, expect, it } from "vitest";
import { INVOICE_PDF_STRINGS_EN, renderInvoicePdf, type InvoicePdfData, type QrMatrix } from "./invoice-pdf";
import { computeLine, computeTotals, vatSummary, type InvoiceLineInput } from "./model";
import { PdfDoc, textWidth, wrapText } from "./pdf";

const latin1 = (bytes: Uint8Array): string => Buffer.from(bytes).toString("latin1");

function data(inputs: InvoiceLineInput[], over: Partial<InvoicePdfData> = {}): InvoicePdfData {
  const lines = inputs.map(computeLine);
  return {
    number: "FV/2026/00001",
    isCorrection: false,
    issueDate: "2026-07-06",
    saleDate: "2026-07-06",
    dueDate: "2026-07-20",
    paymentMethod: "transfer",
    currency: "PLN",
    seller: { name: "Seller sp. z o.o.", taxId: "5265877635", address: "ul. Kwiatowa 1, Warszawa", bankAccount: "PL61 1090 1014 0000 0712 1981 2874" },
    buyer: { name: "Buyer sp. z o.o.", taxId: "5252248481", address: "ul. Prosta 2, Warszawa" },
    lines: lines.map((l, i) => ({ ...l, position: i + 1 })),
    totals: computeTotals(lines),
    summary: vatSummary(lines),
    ...over,
  };
}

const one: InvoiceLineInput = { name: "Usługa doradcza", qty: 1, unit: "szt.", unitPrice: 123, vatRate: "23" };
const many = (n: number): InvoiceLineInput[] =>
  Array.from({ length: n }, (_, i) => ({ name: `Item ${i + 1}`, qty: 1, unit: "szt.", unitPrice: 10, vatRate: "23" as const }));

/** A 21-module symbol with one run per row: enough to exercise placement without a QR library. */
const matrix: QrMatrix = { size: 21, runs: Array.from({ length: 21 }, (_, row) => ({ row, col: row % 3, length: 7 })) };
const QR_FILL = /q 0 g ((?:[-\d.]+ [-\d.]+ [-\d.]+ [-\d.]+ re ?){2,})f Q/;

/** Start x and baseline y of every text op in the first page's content stream. */
function firstPageText(pdf: string): Array<{ x: number; y: number }> {
  const first = pdf.slice(pdf.indexOf("stream"), pdf.indexOf("endstream"));
  return [...first.matchAll(/1 0 0 1 ([-\d.]+) ([-\d.]+) Tm/g)].map((m) => ({ x: Number(m[1]), y: Number(m[2]) }));
}

describe("PdfDoc", () => {
  it("emits a well-formed file whose xref offset points at the xref table", () => {
    const doc = new PdfDoc();
    doc.addPage().text(50, 700, "Hello");
    const s = latin1(doc.build());
    expect(s.startsWith("%PDF-1.4\n")).toBe(true);
    expect(s).toContain("/Count 1");
    expect(s.trimEnd().endsWith("%%EOF")).toBe(true);
    const start = Number(/startxref\n(\d+)/.exec(s)?.[1]);
    expect(s.slice(start, start + 4)).toBe("xref");
  });

  it("maps Polish letters through /Differences and escapes string delimiters", () => {
    const doc = new PdfDoc();
    doc.addPage().text(50, 700, "Usługa (montaż) 50\\50");
    const s = latin1(doc.build());
    expect(s).toContain("/Differences [128 /aogonek");
    expect(s).toContain("Us\\203uga \\(monta\\210\\) 50\\\\50");
  });

  it("keeps every content byte in ASCII, printing an unsupported letter as a question mark", () => {
    const doc = new PdfDoc();
    doc.addPage().text(50, 700, "Škoda café 5 €");
    const s = latin1(doc.build());
    expect(s).toContain("(?koda caf\\351 5 EUR)");
    const stream = s.slice(s.indexOf("stream\n") + 7, s.indexOf("\nendstream"));
    expect(/^[\x20-\x7e\n]*$/.test(stream)).toBe(true);
  });

  it("measures with Helvetica metrics", () => {
    expect(textWidth("00", 10)).toBeCloseTo(11.12, 2);
    expect(textWidth("ął", 10)).toBeCloseTo(7.78, 2);
  });
});

describe("wrapText", () => {
  it("breaks at spaces and keeps every word", () => {
    const text = "Konsultacja techniczna z przegladem instalacji, pomiarami kontrolnymi i protokolem odbioru";
    const rows = wrapText(text, 8.5, 210);
    expect(rows.length).toBeGreaterThan(1);
    expect(rows.join(" ")).toBe(text);
    for (const row of rows) expect(textWidth(row, 8.5)).toBeLessThanOrEqual(210);
  });

  it("splits a single word wider than the column and returns one row for empty text", () => {
    const rows = wrapText("A".repeat(120), 8.5, 210);
    expect(rows.length).toBeGreaterThan(1);
    expect(rows.join("")).toBe("A".repeat(120));
    expect(wrapText("", 9, 100)).toEqual([""]);
  });
});

describe("renderInvoicePdf", () => {
  it("renders the base document with parties, dates and totals", () => {
    const s = latin1(renderInvoicePdf(data([one])));
    expect(s).toContain("Faktura VAT");
    expect(s).toContain("FV/2026/00001");
    expect(s).toContain("06.07.2026");
    expect(s).toContain("123,00 z\\203");
    expect(s).toContain("NIP 5265877635");
    expect(s).toContain("Forma p\\203atno\\206ci: Przelew");
  });

  it("renders a correction with the original's number, the reason and a refund", () => {
    const s = latin1(
      renderInvoicePdf(
        data([{ ...one, qty: -1 }], {
          number: "FK/2026/00001",
          isCorrection: true,
          correctsNumber: "FV/2026/00001",
          correctionReason: "Wrong price",
        }),
      ),
    );
    expect(s).toContain("do faktury nr FV/2026/00001");
    expect(s).toContain("Przyczyna korekty: Wrong price");
    expect(s).toContain("Do zwrotu: 123,00");
  });

  it("prints a long line name whole, on more than one row", () => {
    const name = "Konsultacja techniczna z przegladem instalacji, pomiarami kontrolnymi, regulacja i protokolem odbioru";
    const s = latin1(renderInvoicePdf(data([{ ...one, name }])));
    for (const word of name.split(" ")) expect(s).toContain(word);
    expect(s).not.toContain("...");
    // The line's amounts sit on the first row of the name, and the rule under the row moves down.
    expect((s.match(/\(Konsultacja[^)]*\) Tj/g) ?? []).length).toBe(1);
  });

  it("prints the exemption basis when the document has one", () => {
    const s = latin1(renderInvoicePdf(data([{ ...one, vatRate: "zw" }], { vatExemptionBasis: "art. 43 ust. 1 pkt 19" })));
    expect(s).toContain("Podstawa zwolnienia z VAT: art. 43 ust. 1 pkt 19");
    expect(s).toContain("(zw.)");
  });

  it("takes its labels and number format from the strings table", () => {
    const s = latin1(renderInvoicePdf(data([{ ...one, unitPrice: 1234.5 }]), INVOICE_PDF_STRINGS_EN));
    expect(s).toContain("VAT invoice");
    expect(s).toContain("Payment method: Bank transfer");
    expect(s).toContain("1,234.50 PLN");
    expect(s).not.toContain("Faktura");
  });

  it("breaks a long table across pages and keeps the last line", () => {
    const s = latin1(renderInvoicePdf(data(many(60))));
    expect((s.match(/\/Type \/Page /g) ?? []).length).toBeGreaterThanOrEqual(2);
    expect(s).toContain("(Item 60)");
  });
});

describe("the verification code", () => {
  it("is absent when no bridge supplies one", () => {
    expect(QR_FILL.test(latin1(renderInvoicePdf(data([one]))))).toBe(false);
  });

  it("is drawn in one fill with one rectangle per run, with the label under it", () => {
    const s = latin1(renderInvoicePdf(data([one], { verification: { matrix, label: "5265877635-20260706-ABCDEF012345-51" } })));
    const fill = QR_FILL.exec(s)?.[1] ?? "";
    expect((fill.match(/ re/g) ?? []).length).toBe(matrix.runs.length);
    expect(s).toContain("(5265877635-20260706-ABCDEF012345-51)");
  });

  it("stays on the first page and inside the page", () => {
    const s = latin1(renderInvoicePdf(data(many(60), { verification: { matrix, label: "OFFLINE" } })));
    const first = s.slice(s.indexOf("stream"), s.indexOf("endstream"));
    const fill = QR_FILL.exec(first)?.[1] ?? "";
    const nums = (fill.match(/[-\d.]+/g) ?? []).map(Number);
    expect(nums.length).toBe(matrix.runs.length * 4);
    for (let i = 0; i + 3 < nums.length; i += 4) {
      const [x = 0, y = 0, w = 0, h = 0] = nums.slice(i, i + 4);
      expect(x).toBeGreaterThanOrEqual(0);
      expect(x + w).toBeLessThanOrEqual(595.28);
      expect(y).toBeGreaterThanOrEqual(0);
      expect(y + h).toBeLessThanOrEqual(841.89);
    }
  });

  it("is never overprinted: nothing on the first page reaches into its band from the right", () => {
    // The symbol spans x 463 to 547 and y 62 to 146. Payment details may sit beside it, starting at the
    // left margin; a table row or a total, which ends at the right margin, may not.
    for (const count of [1, 28, 34, 40, 60]) {
      const s = latin1(renderInvoicePdf(data(many(count), { verification: { matrix, label: "OFFLINE" } })));
      const intruders = firstPageText(s).filter((t) => t.y > 60 && t.y < 150 && t.x > 60);
      expect(intruders, `${count} lines`).toEqual([]);
    }
  });
});
```

## The service and the database

`lib/invoicing/service.test.ts` and the SQL checks for the schema are in
[testing-service.md](testing-service.md).

## What is not tested here

- Rendering in a viewer. The suite proves structure and encoding; open one base invoice, one correction
  and one two-page document in a PDF viewer and on paper before the first customer does.
- The host's guard, its seller profile and its audit sink, which are the host's to test.
- FA(3) schema validity, which is an `xmllint` run described in [fa3-xml.md](fa3-xml.md).
