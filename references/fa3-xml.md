# FA(3) XML: the structured invoice built from a stored document

Optional. Needed only when the app sends its invoices to KSeF through the bridge in
[ksef-bridge.md](ksef-bridge.md). The `ksef` skill transports an FA(3) file and does not build one; this
file is the builder, from the `InvoiceWithLines` this module stores.

| File | Role |
|---|---|
| `lib/invoicing/ksef/fa3-vat.ts` | The rate vocabulary mapped onto the schema's `P_12` codes and `P_13_*` / `P_14_*` buckets |
| `lib/invoicing/ksef/fa3-xml.ts` | The builder: header, parties, buckets, annotations, correction block, lines |

Three facts shape it:

1. **The output is deterministic.** The same document gives byte-identical XML. The verification code on
   the PDF hashes the exact bytes that are sent, and KSeF's duplicate detection lets a crashed send be
   retried, so nothing in the builder reads a clock: `DataWytworzeniaFa` is the document's own creation
   time.
2. **Element order is part of validity.** `Fa` is an `xsd:sequence`, so a correctly named element in the
   wrong position is invalid. The order below follows the published schema, version `1-0E`.
3. **A bucket is a tax declaration.** KSeF validates the shape of the file, not the intent. An amount in
   the wrong `P_13_*` element is accepted and declares the wrong thing. The map below was read from the
   schema's own documentation strings, and a rate with no mechanical mapping is refused.

## Rate buckets

```typescript
// lib/invoicing/ksef/fa3-vat.ts
import { InvoiceError, type VatRate } from "../model";

/** `TStawkaPodatku`: the closed `P_12` enum of the FA(3) schema. */
export type Fa3TaxRateCode =
  | "23" | "22" | "8" | "7" | "5" | "4" | "3"
  | "0 KR" | "0 WDT" | "0 EX" | "zw" | "oo" | "np I" | "np II";

export type Fa3VatBucket = {
  /** The `P_12` code on each `FaWiersz`. */
  readonly rateCode: Fa3TaxRateCode;
  /** The net-sum element in `Fa`. */
  readonly netField: string;
  /** The tax-sum element, or null where the bucket has no tax amount by construction. */
  readonly vatField: string | null;
};

/**
 * Only P_13_1 to P_13_5 have a paired P_14 element. The zero-rated and exempt buckets carry no tax field.
 * Bare "0" is not a member of the enum: a domestic 0% is "0 KR", which is what a seller with domestic
 * sales issues. Intra-community supply ("0 WDT") and export ("0 EX") are different buckets and need their
 * own rate in the vocabulary before they can be sent.
 */
export const FA3_VAT_BUCKETS: Partial<Record<VatRate, Fa3VatBucket>> = {
  "23": { rateCode: "23", netField: "P_13_1", vatField: "P_14_1" },
  "8": { rateCode: "8", netField: "P_13_2", vatField: "P_14_2" },
  "5": { rateCode: "5", netField: "P_13_3", vatField: "P_14_3" },
  "0": { rateCode: "0 KR", netField: "P_13_6_1", vatField: null },
  zw: { rateCode: "zw", netField: "P_13_7", vatField: null },
};

/**
 * `np` has no entry. The schema splits it into "np I" (supply outside the country) and "np II" (services
 * under art. 100 ust. 1 pkt 4), and choosing between them is a tax determination, not a transformation.
 */
export function isSendableRate(rate: VatRate): boolean {
  return FA3_VAT_BUCKETS[rate] !== undefined;
}

export function fa3Bucket(rate: VatRate): Fa3VatBucket {
  const bucket = FA3_VAT_BUCKETS[rate];
  if (!bucket) throw new InvoiceError("e_invoice_unsendable", { reason: "rate", rate });
  return bucket;
}
```

## The builder

```typescript
// lib/invoicing/ksef/fa3-xml.ts
import { InvoiceError, vatSummary, type VatRate, type VatSummaryRow } from "../model";
import type { InvoiceWithLines } from "../store";
import { fa3Bucket } from "./fa3-vat";

export const FA3_NAMESPACE = "http://crd.gov.pl/wzor/2025/06/25/13775/";
export const FA3_SYSTEM_CODE = "FA (3)";
export const FA3_SCHEMA_VERSION = "1-0E";

export type Fa3Line = {
  name: string;
  qty: number;
  unit: string;
  /** Net unit price. Signed on a correction delta. */
  unitNet: number;
  /** Net line total. Signed on a correction delta. */
  net: number;
  vatRate: VatRate;
};

export type Fa3Correction = {
  /** `DataWystFaKorygowanej`: the original's issue date. */
  originalIssueDate: string;
  /** `NrFaKorygowanej`: the original's own number. */
  originalNumber: string;
  /** The KSeF number of the original, or null when the original was issued outside KSeF. */
  originalKsefNumber: string | null;
  /** `PrzyczynaKorekty`. */
  reason: string;
};

export type Fa3Document = {
  invoiceNumber: string;
  /** `P_1`, YYYY-MM-DD. */
  issueDate: string;
  /** `P_6`, YYYY-MM-DD. */
  saleDate: string;
  currency: string;
  seller: { nip: string; name: string; address: string };
  /** A null `nip` is a buyer with no tax identifier, emitted as `BrakID`. */
  buyer: { nip: string | null; name: string; address: string | null };
  lines: Fa3Line[];
  vatSummary: VatSummaryRow[];
  /** `P_15`. */
  totalGross: number;
  /** Required when any line is `zw`. */
  vatExemptionBasis: string | null;
  correction: Fa3Correction | null;
  /** `Naglowek/DataWytworzeniaFa`. Passed in so the output stays deterministic. */
  createdAt: string;
  /** `Naglowek/SystemInfo`: the name of the software that produced the file. */
  systemInfo: string;
};

export function escapeXml(value: string): string {
  return value
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&apos;");
}

function el(name: string, value: string): string {
  return `<${name}>${value}</${name}>`;
}

/** TKwotowy: exactly two decimals, optional leading minus. */
function money(n: number): string {
  const magnitude = Math.round((Math.abs(n) + Number.EPSILON) * 100) / 100;
  return (n < 0 && magnitude !== 0 ? -magnitude : magnitude).toFixed(2);
}

/** TIlosci: up to six decimals, trailing zeros dropped. */
function quantity(n: number): string {
  return (Math.round(n * 1e6) / 1e6).toString();
}

/**
 * TKwotowy2: up to eight decimals. A unit price below one hundredth (a product sold by the millilitre)
 * does not survive two decimals: `P_9A * P_8B` would then disagree with the line net `P_11`.
 */
function unitPrice(n: number): string {
  return (Math.round(n * 1e8) / 1e8).toFixed(8).replace(/(\.\d*?)0+$/, "$1").replace(/\.$/, "");
}

/**
 * TNrNIP is ten bare digits. People type separators and forms prefill a formatted value, so the builder
 * strips them: it is the last gate before the bytes leave, and it covers every row already stored.
 */
function nipDigits(nip: string): string {
  return nip.replace(/[^\d]/g, "");
}

/** TAdres takes address lines, not structured parts. */
function buildAdres(address: string): string {
  const line = address.replace(/\s+/g, " ").trim().slice(0, 512);
  return `<Adres>${el("KodKraju", "PL")}${el("AdresL1", escapeXml(line))}</Adres>`;
}

function buildPodmiot1(seller: Fa3Document["seller"]): string {
  return (
    `<Podmiot1>` +
    `<DaneIdentyfikacyjne>${el("NIP", nipDigits(seller.nip))}${el("Nazwa", escapeXml(seller.name))}</DaneIdentyfikacyjne>` +
    buildAdres(seller.address) +
    `</Podmiot1>`
  );
}

function buildPodmiot2(buyer: Fa3Document["buyer"]): string {
  const ident = buyer.nip ? el("NIP", nipDigits(buyer.nip)) : el("BrakID", "1");
  return (
    `<Podmiot2>` +
    `<DaneIdentyfikacyjne>${ident}${el("Nazwa", escapeXml(buyer.name))}</DaneIdentyfikacyjne>` +
    (buyer.address ? buildAdres(buyer.address) : "") +
    // JST and GV are mandatory markers. "2" declares that the buyer is neither a subordinate unit of a
    // local government nor a member of a VAT group. A seller with such buyers needs both as inputs.
    el("JST", "2") +
    el("GV", "2") +
    `</Podmiot2>`
  );
}

/** Buckets are emitted in schema order, whatever order the rates appear on the document. */
const BUCKET_ORDER = [
  "P_13_1", "P_13_2", "P_13_3", "P_13_4", "P_13_5", "P_13_6_1", "P_13_6_2", "P_13_6_3",
  "P_13_7", "P_13_8", "P_13_9", "P_13_10", "P_13_11",
] as const;

function buildVatBuckets(summary: readonly VatSummaryRow[]): string {
  const nets = new Map<string, number>();
  const vats = new Map<string, { field: string; amount: number }>();
  for (const row of summary) {
    const bucket = fa3Bucket(row.vatRate);
    nets.set(bucket.netField, (nets.get(bucket.netField) ?? 0) + row.net);
    if (bucket.vatField) {
      vats.set(bucket.netField, {
        field: bucket.vatField,
        amount: (vats.get(bucket.netField)?.amount ?? 0) + row.vat,
      });
    }
  }
  let out = "";
  for (const field of BUCKET_ORDER) {
    const net = nets.get(field);
    if (net === undefined) continue;
    out += el(field, money(net));
    const vat = vats.get(field);
    if (vat) out += el(vat.field, money(vat.amount));
  }
  return out;
}

/**
 * `Adnotacje` is mandatory and so is every child of it: there is no leaving out what does not apply. Each
 * special regime is declared as "no", except that an exempt line forces a real legal basis.
 */
function buildAdnotacje(doc: Fa3Document): string {
  let zwolnienie = `<Zwolnienie>${el("P_19N", "1")}</Zwolnienie>`;
  if (doc.vatSummary.some((r) => r.vatRate === "zw")) {
    const basis = doc.vatExemptionBasis?.trim();
    if (!basis) throw new InvoiceError("exemption_basis_required");
    zwolnienie = `<Zwolnienie>${el("P_19", "1")}${el("P_19A", escapeXml(basis))}</Zwolnienie>`;
  }
  return (
    `<Adnotacje>` +
    el("P_16", "2") + // cash accounting: no
    el("P_17", "2") + // self-billing: no
    el("P_18", "2") + // reverse charge: no
    el("P_18A", "2") + // split payment: no
    zwolnienie +
    `<NoweSrodkiTransportu>${el("P_22N", "1")}</NoweSrodkiTransportu>` +
    el("P_23", "2") + // simplified triangular procedure: no
    `<PMarzy>${el("P_PMarzyN", "1")}</PMarzy>` +
    `</Adnotacje>`
  );
}

/**
 * Emitted immediately after `RodzajFaktury`. `TypKorekty` is optional in the schema and is a tax
 * determination this module holds no data for, so it is left out. The original is identified by its KSeF
 * number, or by `NrKSeFN` when it was issued outside KSeF and has none.
 */
function buildCorrectionBlock(c: Fa3Correction): string {
  const reference = c.originalKsefNumber
    ? el("NrKSeF", "1") + el("NrKSeFFaKorygowanej", escapeXml(c.originalKsefNumber))
    : el("NrKSeFN", "1");
  return (
    el("PrzyczynaKorekty", escapeXml(c.reason)) +
    `<DaneFaKorygowanej>` +
    el("DataWystFaKorygowanej", c.originalIssueDate) +
    el("NrFaKorygowanej", escapeXml(c.originalNumber)) +
    reference +
    `</DaneFaKorygowanej>`
  );
}

function buildLines(lines: readonly Fa3Line[]): string {
  return lines
    .map(
      (line, i) =>
        `<FaWiersz>` +
        el("NrWierszaFa", String(i + 1)) +
        el("P_7", escapeXml(line.name)) +
        el("P_8A", escapeXml(line.unit)) +
        el("P_8B", quantity(line.qty)) +
        el("P_9A", unitPrice(line.unitNet)) +
        el("P_11", money(line.net)) +
        el("P_12", fa3Bucket(line.vatRate).rateCode) +
        `</FaWiersz>`,
    )
    .join("");
}

/** UTF-8 with no byte order mark: KSeF rejects a BOM. */
export function buildFa3Xml(doc: Fa3Document): string {
  return (
    `<?xml version="1.0" encoding="UTF-8"?>` +
    `<Faktura xmlns="${FA3_NAMESPACE}">` +
    `<Naglowek>` +
    `<KodFormularza kodSystemowy="${FA3_SYSTEM_CODE}" wersjaSchemy="${FA3_SCHEMA_VERSION}">FA</KodFormularza>` +
    el("WariantFormularza", "3") +
    el("DataWytworzeniaFa", doc.createdAt) +
    el("SystemInfo", escapeXml(doc.systemInfo)) +
    `</Naglowek>` +
    buildPodmiot1(doc.seller) +
    buildPodmiot2(doc.buyer) +
    `<Fa>` +
    el("KodWaluty", doc.currency) +
    el("P_1", doc.issueDate) +
    el("P_2", escapeXml(doc.invoiceNumber)) +
    el("P_6", doc.saleDate) +
    buildVatBuckets(doc.vatSummary) +
    el("P_15", money(doc.totalGross)) +
    buildAdnotacje(doc) +
    el("RodzajFaktury", doc.correction ? "KOR" : "VAT") +
    (doc.correction ? buildCorrectionBlock(doc.correction) : "") +
    buildLines(doc.lines) +
    `</Fa>` +
    `</Faktura>`
  );
}

/** A stored document as an FA(3) document. Reads only the snapshot, never the live seller profile. */
export function toFa3Document(
  invoice: InvoiceWithLines,
  correction: Fa3Correction | null,
  systemInfo: string,
): Fa3Document {
  if (!invoice.seller.taxId) throw new InvoiceError("e_invoice_unsendable", { reason: "seller_tax_id" });
  if (!invoice.seller.address) throw new InvoiceError("e_invoice_unsendable", { reason: "seller_address" });
  return {
    invoiceNumber: invoice.number,
    issueDate: invoice.issueDate,
    saleDate: invoice.saleDate,
    currency: invoice.currency,
    seller: { nip: invoice.seller.taxId, name: invoice.seller.name, address: invoice.seller.address },
    buyer: { nip: invoice.buyer.taxId, name: invoice.buyer.name, address: invoice.buyer.address },
    lines: invoice.lines.map((l) => ({
      name: l.name,
      qty: l.qty,
      unit: l.unit,
      // Prices are stored gross and FA(3) wants a net unit price. Four decimals, so that a
      // per-millilitre price still multiplies back to the line net.
      unitNet: l.qty === 0 ? 0 : Math.round((l.net / l.qty) * 10000) / 10000,
      net: l.net,
      vatRate: l.vatRate,
    })),
    vatSummary: vatSummary(invoice.lines),
    totalGross: invoice.totals.gross,
    vatExemptionBasis: invoice.vatExemptionBasis,
    correction,
    // Milliseconds, UTC. The stored value may carry microseconds; one normalised form keeps the bytes
    // identical however the store formats a timestamp.
    createdAt: new Date(invoice.createdAt).toISOString(),
    systemInfo,
  };
}
```

## Tests

String assertions on the builder's output. They hold the order, the buckets, the correction reference and
the determinism; they do not prove schema validity, which the next section does.

```typescript
// lib/invoicing/ksef/fa3-xml.test.ts
import { describe, expect, it } from "vitest";
import { InvoiceError, computeLine, computeTotals, type InvoiceLineInput } from "../model";
import type { InvoiceWithLines } from "../store";
import { isSendableRate } from "./fa3-vat";
import { buildFa3Xml, toFa3Document } from "./fa3-xml";

function invoice(inputs: InvoiceLineInput[], over: Partial<InvoiceWithLines> = {}): InvoiceWithLines {
  const lines = inputs.map(computeLine);
  return {
    id: "inv-1",
    tenantId: "t1",
    kind: "base",
    series: "FV",
    year: 2026,
    seq: 12,
    number: "FV/2026/00012",
    issueDate: "2026-07-15",
    saleDate: "2026-07-15",
    dueDate: "2026-07-29",
    paymentMethod: "transfer",
    currency: "PLN",
    seller: { name: "Seller sp. z o.o.", taxId: "526-587-76-35", address: "ul. Prosta 1\n00-001 Warszawa", bankAccount: null },
    buyer: { name: "Buyer & Co", taxId: "5252248481", address: "ul. Nabywcy 2, 00-002 Warszawa" },
    customerId: null,
    source: null,
    totals: computeTotals(lines),
    vatExemptionBasis: null,
    correctsInvoiceId: null,
    correctionReason: null,
    correctedByInvoiceId: null,
    paidAt: null,
    eInvoice: { status: "pending", number: null, xmlSha256: null, error: null },
    createdBy: "u1",
    createdAt: "2026-07-15T10:30:00.123456Z",
    lines: lines.map((l, i) => ({ ...l, position: i + 1 })),
    ...over,
  };
}

const service: InvoiceLineInput = { name: "Service", qty: 1, unit: "szt.", unitPrice: 246, vatRate: "23" };
const xmlOf = (inv: InvoiceWithLines) => buildFa3Xml(toFa3Document(inv, null, "test-app"));

describe("buildFa3Xml", () => {
  it("is deterministic and reads no clock", () => {
    expect(xmlOf(invoice([service]))).toBe(xmlOf(invoice([service])));
    expect(xmlOf(invoice([service]))).toContain("<DataWytworzeniaFa>2026-07-15T10:30:00.123Z</DataWytworzeniaFa>");
  });

  it("strips separators from a tax identifier and escapes text", () => {
    const xml = xmlOf(invoice([service]));
    expect(xml).toContain("<NIP>5265877635</NIP>");
    expect(xml).toContain("<Nazwa>Buyer &amp; Co</Nazwa>");
    expect(xml).toContain("<AdresL1>ul. Prosta 1 00-001 Warszawa</AdresL1>");
  });

  it("emits a buyer with no tax identifier as BrakID", () => {
    const xml = xmlOf(invoice([service], { buyer: { name: "Walk-in", taxId: null, address: null } }));
    expect(xml).toContain("<BrakID>1</BrakID><Nazwa>Walk-in</Nazwa>");
  });

  it("files each rate in its own bucket, in schema order", () => {
    const xml = xmlOf(
      invoice(
        [
          { ...service, name: "Exempt", unitPrice: 100, vatRate: "zw" },
          { ...service, name: "Reduced", unitPrice: 108, vatRate: "8" },
          service,
        ],
        { vatExemptionBasis: "art. 43 ust. 1 pkt 19" },
      ),
    );
    expect(xml).toContain("<P_13_1>200.00</P_13_1><P_14_1>46.00</P_14_1><P_13_2>100.00</P_13_2><P_14_2>8.00</P_14_2><P_13_7>100.00</P_13_7>");
    expect(xml).toContain("<P_19>1</P_19><P_19A>art. 43 ust. 1 pkt 19</P_19A>");
    expect(xml).toContain("<P_12>zw</P_12>");
  });

  it("refuses an exempt line with no legal basis", () => {
    expect(() => xmlOf(invoice([{ ...service, vatRate: "zw" }]))).toThrow(InvoiceError);
  });

  it("refuses a rate with no bucket and a seller with no tax identifier", () => {
    expect(isSendableRate("np")).toBe(false);
    expect(() => xmlOf(invoice([{ ...service, vatRate: "np" }]))).toThrow(InvoiceError);
    const noTaxId = invoice([service], { seller: { name: "S", taxId: null, address: "A", bankAccount: null } });
    expect(() => xmlOf(noTaxId)).toThrow(InvoiceError);
  });

  it("keeps a unit price below one hundredth", () => {
    const xml = xmlOf(invoice([{ name: "Oil", qty: 30, unit: "ml", unitPrice: 0.06, vatRate: "23" }]));
    // 1.80 gross is 1.46 net; 1.46 / 30 is 0.0487, which two decimals would turn into 0.05.
    expect(xml).toContain("<P_8B>30</P_8B><P_9A>0.0487</P_9A><P_11>1.46</P_11>");
  });

  it("emits a whole unit price with no trailing zeros", () => {
    expect(xmlOf(invoice([service]))).toContain("<P_9A>200</P_9A>");
  });

  it("references the original by KSeF number, or declares that it has none", () => {
    const delta = invoice([{ ...service, qty: -1 }], { kind: "correction", number: "FK/2026/00001" });
    const base = { originalIssueDate: "2026-07-15", originalNumber: "FV/2026/00012", reason: "Wrong price" };
    const withNumber = buildFa3Xml(
      toFa3Document(delta, { ...base, originalKsefNumber: "5265877635-20260715-ABCDEF012345-51" }, "test-app"),
    );
    expect(withNumber).toContain("<RodzajFaktury>KOR</RodzajFaktury><PrzyczynaKorekty>Wrong price</PrzyczynaKorekty>");
    expect(withNumber).toContain("<NrKSeF>1</NrKSeF><NrKSeFFaKorygowanej>5265877635-20260715-ABCDEF012345-51</NrKSeFFaKorygowanej>");
    expect(withNumber).toContain("<P_13_1>-200.00</P_13_1><P_14_1>-46.00</P_14_1>");

    const without = buildFa3Xml(toFa3Document(delta, { ...base, originalKsefNumber: null }, "test-app"));
    expect(without).toContain("<NrFaKorygowanej>FV/2026/00012</NrFaKorygowanej><NrKSeFN>1</NrKSeFN>");
  });
});
```

## Validating against the schema

The tests above run anywhere. Schema validity is checked with `xmllint` against the Ministry's XSD, which
is published with the FA(3) documentation and imports a second schema by URL, so the check needs network
access the first time:

```bash
xmllint --noout --schema "schemat_FA(3)_v1-0E.xsd" invoice.xml
```

Run it on a base invoice, an invoice with an exempt line, a buyer with no tax identifier, and both forms of
correction before the first send to the TEST environment, and again whenever the builder changes. The
templates in this file were validated that way for those five documents. The TEST environment accepts some
files DEMO and production reject, so a clean TEST send is not a substitute.

## What the builder does not cover

Each of these is a field the schema has and this module holds no data for. They are gaps to fill from the
host's data, not defaults to guess:

- A buyer outside Poland (`KodKraju` is written as `PL`, and an EU buyer needs `KodUE` and `NrVatUE`).
- A currency other than PLN on a document that must also state the VAT in PLN (`KursWaluty`).
- Intra-community supply, export and the two `np` codes.
- Split payment, reverse charge, margin schemes and the other `Adnotacje` regimes, all declared as "no".
- `TypKorekty` and the seller's or buyer's earlier data on a correction (`Podmiot1K`, `Podmiot2K`).
- Payment terms, bank account and the other optional `Platnosc` content.
