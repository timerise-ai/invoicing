# The KSeF bridge: joining this module to the `ksef` skill

Optional. Without it the module issues, numbers, corrects and prints invoices, every document is
`not_required`, and the PDF carries no verification code. With it, each issued document is built as FA(3),
handed to the [ksef](https://github.com/timerise-ai/ksef) skill's send path, and mirrored back: the register
shows whether KSeF accepted it and the PDF carries the KOD I code.

The two skills split the work along one line. **This module owns the document; the `ksef` skill owns the
conversation with the Ministry.**

| Concern | Owner | Where |
|---|---|---|
| Invoice rows, numbering, corrections, PDF | this skill | the other references |
| FA(3) XML built from a stored document | this skill | [fa3-xml.md](fa3-xml.md) |
| Which documents go out, in what order, and what the register shows | this skill | `createKsefBridge` below |
| Credentials, token auth, encryption, sessions, sending, polling, UPO | `ksef` skill | its `lib/ksef/*` and cron routes |
| The seller NIP compared with the authenticating context | `ksef` skill | its pre-send validation |
| The outbound row a cron resumes from | `ksef` skill | its `ksef_invoices` table |

## The four joins

| When | Direction | Call |
|---|---|---|
| A document is issued | this module to `ksef` | `bridge.port.onIssued` builds the XML and calls `enqueue`, which writes the `ksef` skill's outbound row |
| KSeF accepts it | `ksef` to this module | the status cron calls `bridge.onAccepted(tenantId, invoiceId, ksefNumber)` |
| KSeF rejects it | `ksef` to this module | the status cron calls `bridge.onRejected(tenantId, invoiceId, description)` |
| A PDF is rendered | this module | `bridge.port.verification` builds the KOD I link from the stored hash |

## The bridge

```typescript
// lib/invoicing/ksef/bridge.ts
import { createHash } from "node:crypto";
import { InvoiceError } from "../model";
import type { EInvoicePort, Verification } from "../service";
import type { Invoice, InvoiceStore, InvoiceWithLines, NewInvoice } from "../store";
import { isSendableRate } from "./fa3-vat";
import { buildFa3Xml, toFa3Document, type Fa3Correction } from "./fa3-xml";
import { qrMatrix } from "./qr";

/** The exact bytes to send, and enough to file them. */
export type KsefOutboundDocument = {
  tenantId: string;
  invoiceId: string;
  invoiceNumber: string;
  sellerNip: string;
  xml: Buffer;
};

export type KsefBridgeDeps = {
  store: InvoiceStore;
  /** True when the tenant has KSeF credentials and sending is switched on. */
  isEnabled(tenantId: string): Promise<boolean>;
  /** The QR base URL of the environment the tenant sends to, such as https://qr-test.ksef.mf.gov.pl. */
  qrHost(tenantId: string): Promise<string>;
  /** `Naglowek/SystemInfo`: the name of the host application. */
  systemInfo: string;
  /** Write the outbound row the `ksef` skill's send cron picks up. Idempotent on `invoiceId`. */
  enqueue(doc: KsefOutboundDocument): Promise<void>;
};

/** Printed under the code until KSeF has assigned a number. The literal is the specification's. */
export const OFFLINE_LABEL = "OFFLINE";

export function createKsefBridge(deps: KsefBridgeDeps) {
  const { store } = deps;

  /**
   * Build and enqueue one document. Returns without sending when a correction's original is still on its
   * way: `onAccepted` for the original calls this again.
   */
  async function send(invoice: InvoiceWithLines): Promise<void> {
    let correction: Fa3Correction | null = null;
    if (invoice.kind === "correction") {
      const original = invoice.correctsInvoiceId ? await store.get(invoice.tenantId, invoice.correctsInvoiceId) : null;
      if (!original) throw new Error(`invoicing: original of ${invoice.number} not found`);
      // A correction names the original's KSeF number, which exists only once KSeF has accepted it.
      if (original.eInvoice.status === "pending") return;
      if (original.eInvoice.status === "rejected") {
        await store.setEInvoice(invoice.tenantId, invoice.id, { error: "original_rejected" });
        return;
      }
      correction = {
        originalIssueDate: original.issueDate,
        originalNumber: original.number,
        // Null for an original issued before KSeF was switched on: the builder then declares NrKSeFN.
        originalKsefNumber: original.eInvoice.number,
        reason: invoice.correctionReason ?? "",
      };
    }

    const document = toFa3Document(invoice, correction, deps.systemInfo);
    const xml = Buffer.from(buildFa3Xml(document), "utf8");
    await deps.enqueue({
      tenantId: invoice.tenantId,
      invoiceId: invoice.id,
      invoiceNumber: invoice.number,
      sellerNip: document.seller.nip,
      xml,
    });
    // The hash of the bytes that were enqueued, stored once. The PDF never rebuilds the XML to get it.
    await store.setEInvoice(invoice.tenantId, invoice.id, {
      xmlSha256: createHash("sha256").update(xml).digest("hex"),
      error: null,
    });
  }

  const port: EInvoicePort = {
    isEnabled: deps.isEnabled,

    preflight(draft: NewInvoice): void {
      if (!draft.seller.taxId) throw new InvoiceError("e_invoice_unsendable", { reason: "seller_tax_id" });
      if (!draft.seller.address) throw new InvoiceError("e_invoice_unsendable", { reason: "seller_address" });
      for (const line of draft.lines) {
        if (!isSendableRate(line.vatRate)) {
          throw new InvoiceError("e_invoice_unsendable", { reason: "rate", rate: line.vatRate });
        }
      }
    },

    onIssued: send,

    async verification(invoice: Invoice): Promise<Verification | null> {
      const { xmlSha256, number, status } = invoice.eInvoice;
      if (!xmlSha256 || !invoice.seller.taxId || status === "rejected") return null;
      const [year, month, day] = invoice.issueDate.split("-");
      const hash = Buffer.from(xmlSha256, "hex").toString("base64url");
      const nip = invoice.seller.taxId.replace(/[^\d]/g, "");
      // KOD I: seller NIP, the issue date as DD-MM-YYYY, the SHA-256 of the file as unpadded Base64URL.
      const url = `${await deps.qrHost(invoice.tenantId)}/invoice/${nip}/${day}-${month}-${year}/${hash}`;
      return { url, label: number ?? OFFLINE_LABEL, matrix: qrMatrix(url) };
    },
  };

  /** Called by the `ksef` skill's status cron when a document is accepted. Safe to call twice. */
  async function onAccepted(tenantId: string, invoiceId: string, ksefNumber: string): Promise<void> {
    await store.setEInvoice(tenantId, invoiceId, { status: "accepted", number: ksefNumber, error: null });
    const invoice = await store.get(tenantId, invoiceId);
    if (!invoice?.correctedByInvoiceId) return;
    // Release the correction that was waiting for this number.
    const correction = await store.get(tenantId, invoice.correctedByInvoiceId);
    if (correction && correction.eInvoice.status === "pending" && !correction.eInvoice.xmlSha256) {
      await send(correction);
    }
  }

  /** Called by the status cron on a rejection. `description` is KSeF's own text, stored as it came. */
  async function onRejected(tenantId: string, invoiceId: string, description: string): Promise<void> {
    await store.setEInvoice(tenantId, invoiceId, { status: "rejected", error: description });
  }

  return { port, onAccepted, onRejected };
}

export type KsefBridge = ReturnType<typeof createKsefBridge>;
```

## The verification code

The code is a QR symbol of the KOD I link. It is produced as a matrix of module runs, which the layout in
[pdf.md](pdf.md) draws as vector rectangles. This is the one file in the module with a dependency:
`qrcode`, whose `create` is synchronous, so rendering stays a pure function.

```typescript
// lib/invoicing/ksef/qr.ts
import QRCode from "qrcode";
import type { QrMatrix, QrRun } from "../invoice-pdf";

/**
 * Encode a URL as dark-module runs. Merging horizontal neighbours is what keeps the PDF small: a KOD I
 * link has several hundred dark modules, and one rectangle per module would be several times the size of
 * the rest of the document.
 */
export function qrMatrix(url: string): QrMatrix {
  const { modules } = QRCode.create(url, { errorCorrectionLevel: "M" });
  const { size, data } = modules;
  const runs: QrRun[] = [];
  for (let row = 0; row < size; row++) {
    let start = -1;
    // One column past the edge closes a run that reaches the end of the row.
    for (let col = 0; col <= size; col++) {
      const dark = col < size && data[row * size + col] === 1;
      if (dark && start < 0) start = col;
      if (!dark && start >= 0) {
        runs.push({ row, col: start, length: col - start });
        start = -1;
      }
    }
  }
  return { size, runs };
}
```

## Wiring

In `lib/invoicing/index.ts`, build the bridge and pass its port to the service. `enqueue` is the only code
that touches the `ksef` skill's tables; the sketch below writes the outbound row of that skill's state
schema and is a sketch because the table is the `ksef` skill's to define.

```ts
// lib/invoicing/index.ts (with the bridge)
import { createKsefBridge } from "./ksef/bridge";

export const ksefBridge = createKsefBridge({
  store,
  systemInfo: "your-app-name",
  isEnabled: async (tenantId) => (await loadKsefCredentials(tenantId)) !== null, // from the ksef skill
  qrHost: async () => process.env.KSEF_QR_HOST ?? "https://qr-test.ksef.mf.gov.pl",
  enqueue: async (doc) => {
    await db.query(
      `insert into ksef_invoices (id, tenant_id, direction, status, xml)
       values ($1, $2, 'outbound', 'created', $3)
       on conflict (id) do nothing`,
      [doc.invoiceId, doc.tenantId, doc.xml],
    );
  },
});

export const invoicing = createInvoicing({ store, loadSeller, today, eInvoice: ksefBridge.port });
```

In the `ksef` skill's status cron, after it stores the KSeF number or the rejection, add the two calls:

```ts
// app/api/cron/ksef-status/route.ts (inside the loop over polled invoices)
if (status.code === 200) await ksefBridge.onAccepted(row.tenant_id, row.id, status.ksefNumber);
else if (isFinalRejection(status)) await ksefBridge.onRejected(row.tenant_id, row.id, describeRejection(status));
```

`describeRejection` should carry `status.description` and `status.details`, as the `ksef` skill's hard rule
on rejections says: the code names a category and only the text names the fault.

## Rules

1. **Hash the bytes you enqueue, once.** The KOD I link carries the SHA-256 of the file KSeF receives. The
   bridge hashes the buffer it hands to `enqueue` and stores the digest; `verification` reads the digest.
   Rebuilding the XML at render time would need every input to be unchanged forever, and a link that does
   not verify looks exactly like one that does.
2. **Build from the snapshot.** `toFa3Document` reads the seller from the invoice, not from the tenant
   profile, so the file built for a correction next year still names the seller as the original did.
3. **A correction waits for its original.** It cannot be built until the original has a KSeF number, so
   `send` returns and `onAccepted` releases it. An original that was never sent to KSeF has no number, and
   the correction then declares `NrKSeFN`.
4. **The environment is in the link.** The QR host and the API host belong to the same environment. Derive
   both from one setting, per tenant if tenants can differ, or a production invoice will carry a test link
   that renders perfectly and verifies nothing.
5. **Refuse at the form what KSeF would refuse later.** `preflight` runs before a number is reserved: a
   missing seller NIP or address and the `np` rate are rejected while the operator can still fix them. A
   document that passes preflight and still fails is recorded on the invoice and shown in the register.
6. **A tenant with no credentials sends nothing and prints no code.** `isEnabled` decides at issue whether
   the document is `pending` or `not_required`. A document issued before KSeF was switched on stays
   `not_required`; it is not sent retroactively.

## Test

The flow against the in-memory store: enqueue on issue, the hash and the link, a correction held until its
original is accepted, and the preflight.

```typescript
// lib/invoicing/ksef/bridge.test.ts
import { describe, expect, it } from "vitest";
import { createMemoryInvoiceStore } from "../memory-store";
import { InvoiceError, type InvoiceLineInput } from "../model";
import { createInvoicing, type InvoiceActor, type SellerProfile } from "../service";
import { createKsefBridge, type KsefOutboundDocument } from "./bridge";

const actor: InvoiceActor = { tenantId: "t1", userId: "u1", canRead: true, canWrite: true };
const seller: SellerProfile = {
  name: "Seller sp. z o.o.",
  taxId: "5265877635",
  address: "ul. Prosta 1, 00-001 Warszawa",
  bankAccount: null,
  paymentTermsDays: 14,
  vatExemptionBasis: null,
  currency: "PLN",
};
const line: InvoiceLineInput = { name: "Service", qty: 1, unit: "szt.", unitPrice: 123, vatRate: "23" };

function setup(enabled = true) {
  const store = createMemoryInvoiceStore();
  const outbox: KsefOutboundDocument[] = [];
  const bridge = createKsefBridge({
    store,
    systemInfo: "test-app",
    isEnabled: async () => enabled,
    qrHost: async () => "https://qr-test.ksef.mf.gov.pl",
    enqueue: async (doc) => {
      if (!outbox.some((d) => d.invoiceId === doc.invoiceId)) outbox.push(doc);
    },
  });
  const invoicing = createInvoicing({
    store,
    loadSeller: async () => seller,
    today: () => "2026-07-15",
    eInvoice: bridge.port,
  });
  return { store, outbox, bridge, invoicing };
}

describe("KSeF bridge", () => {
  it("enqueues a base document and stores the hash of the enqueued bytes", async () => {
    const { store, outbox, invoicing } = setup();
    const issued = await invoicing.issue(actor, { buyer: { name: "Buyer" }, lines: [line] });
    expect(outbox).toHaveLength(1);
    const stored = await store.get("t1", issued.id);
    expect(stored?.eInvoice.status).toBe("pending");
    expect(stored?.eInvoice.xmlSha256).toMatch(/^[0-9a-f]{64}$/);
    expect(outbox[0]?.xml.toString("utf8")).toContain("<P_2>FV/2026/00001</P_2>");
  });

  it("builds the KOD I link from the stored hash and labels it OFFLINE until a number exists", async () => {
    const { bridge, invoicing } = setup();
    const issued = await invoicing.issue(actor, { buyer: { name: "Buyer" }, lines: [line] });
    const before = await invoicing.pdfData(actor, issued.id);
    expect(before?.verification?.label).toBe("OFFLINE");

    await bridge.onAccepted("t1", issued.id, "5265877635-20260715-ABCDEF012345-51");
    const after = await invoicing.get(actor, issued.id);
    const verification = after ? await bridge.port.verification(after) : null;
    expect(verification?.label).toBe("5265877635-20260715-ABCDEF012345-51");
    expect(verification?.url).toMatch(
      /^https:\/\/qr-test\.ksef\.mf\.gov\.pl\/invoice\/5265877635\/15-07-2026\/[A-Za-z0-9_-]{43}$/,
    );
    expect(verification?.matrix.runs.length).toBeGreaterThan(50);
  });

  it("holds a correction until its original is accepted, then sends it with the KSeF number", async () => {
    const { outbox, bridge, invoicing } = setup();
    const original = await invoicing.issue(actor, { buyer: { name: "Buyer" }, lines: [line] });
    const correction = await invoicing.correct(actor, {
      originalId: original.id,
      correctedLines: [{ ...line, unitPrice: 100 }],
      reason: "Wrong price",
    });
    expect(outbox.map((d) => d.invoiceId)).toEqual([original.id]);

    await bridge.onAccepted("t1", original.id, "5265877635-20260715-ABCDEF012345-51");
    expect(outbox.map((d) => d.invoiceId)).toEqual([original.id, correction.id]);
    expect(outbox[1]?.xml.toString("utf8")).toContain(
      "<NrKSeFFaKorygowanej>5265877635-20260715-ABCDEF012345-51</NrKSeFFaKorygowanej>",
    );

    // A second notification for the same acceptance sends nothing more.
    await bridge.onAccepted("t1", original.id, "5265877635-20260715-ABCDEF012345-51");
    expect(outbox).toHaveLength(2);
  });

  it("refuses the np rate before a number is reserved", async () => {
    const { invoicing } = setup();
    await expect(
      invoicing.issue(actor, { buyer: { name: "Buyer" }, lines: [{ ...line, vatRate: "np" }] }),
    ).rejects.toBeInstanceOf(InvoiceError);
    const next = await invoicing.issue(actor, { buyer: { name: "Buyer" }, lines: [line] });
    expect(next.number).toBe("FV/2026/00001");
  });

  it("sends nothing and prints no code for a tenant with KSeF switched off", async () => {
    const { outbox, invoicing } = setup(false);
    const issued = await invoicing.issue(actor, { buyer: { name: "Buyer" }, lines: [{ ...line, vatRate: "np" }] });
    expect(issued.eInvoice.status).toBe("not_required");
    expect(outbox).toHaveLength(0);
    expect((await invoicing.pdfData(actor, issued.id))?.verification).toBeNull();
  });

  it("records a rejection with KSeF's own description", async () => {
    const { store, bridge, invoicing } = setup();
    const issued = await invoicing.issue(actor, { buyer: { name: "Buyer" }, lines: [line] });
    await bridge.onRejected("t1", issued.id, "450: semantic validation failed: P_13_1");
    const stored = await store.get("t1", issued.id);
    expect(stored?.eInvoice).toMatchObject({ status: "rejected", error: "450: semantic validation failed: P_13_1" });
    expect((await invoicing.pdfData(actor, issued.id))?.verification).toBeNull();
  });
});
```

## What this bridge does not do

- **Offline modes and KOD II.** A document issued while KSeF is unreachable needs a second, signed code and
  an `Offline` certificate. Both are in the `ksef` skill's QR reference; the layout has room for one code.
- **Resending a rejected document.** A rejection is shown; fixing it is a correction or a resend through
  the `ksef` skill's technical-correction path, decided by a person.
- **Purchase invoices.** Receiving and syncing documents issued to the seller is the `ksef` skill's.
- **Tax decisions.** The bridge refuses what it cannot express. It does not choose a bucket, a regime or a
  correction type on the operator's behalf.

## Checklist

- [ ] `enqueue` is idempotent on the invoice id
- [ ] The status cron calls `onAccepted` and `onRejected` with the tenant id from its own row
- [ ] The QR host and the API base come from the same environment setting
- [ ] A correction issued before its original is accepted is sent after it, not before
- [ ] An accepted invoice's PDF shows the KSeF number under the code; a pending one shows `OFFLINE`
