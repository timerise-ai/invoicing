# The PDF writer: pages, two built-in fonts, wrapped text

`lib/invoicing/pdf.ts` is a dependency-free PDF writer with exactly the surface an invoice needs. The layout
that uses it is in [pdf.md](pdf.md).

## Why there is no PDF library

A PDF needs a font to place text, and embedding one costs a parser, a subsetter and hundreds of kilobytes.
The writer uses the two Helvetica faces every viewer already has and reaches Polish diacritics through a
`/Differences` array that renames eighteen byte values on top of WinAnsi. The output is a few kilobytes, the
content streams are uncompressed, and the same calls always produce the same bytes.

The limits follow from that choice and are deliberate: text is Latin-1 plus the Polish letters, anything
else prints as `?`, and widths are measured with the regular face's metrics for both faces. A host that
needs Czech, Greek or Cyrillic names on documents needs an embedded font, which is a design in
[operations.md](operations.md).

```typescript
// lib/invoicing/pdf.ts

export const A4 = { width: 595.28, height: 841.89 } as const;

export type PdfFontName = "regular" | "bold";

/** A grey level (0 is black, 1 is white) or a `#rrggbb` colour. */
export type PdfColor = number | string;

export type PdfTextOptions = {
  font?: PdfFontName;
  size?: number;
  /** With `align: "right"`, `x` is the right edge of the rendered text. */
  align?: "left" | "right";
  gray?: number;
  /** Overrides `gray` when set. */
  color?: PdfColor;
};

/** Polish glyphs mapped onto bytes 128 to 145 through /Differences. */
const POLISH: Record<string, { byte: number; glyph: string; width: number }> = {
  ą: { byte: 128, glyph: "aogonek", width: 556 },
  ć: { byte: 129, glyph: "cacute", width: 500 },
  ę: { byte: 130, glyph: "eogonek", width: 556 },
  ł: { byte: 131, glyph: "lslash", width: 222 },
  ń: { byte: 132, glyph: "nacute", width: 556 },
  ó: { byte: 133, glyph: "oacute", width: 556 },
  ś: { byte: 134, glyph: "sacute", width: 500 },
  ź: { byte: 135, glyph: "zacute", width: 500 },
  ż: { byte: 136, glyph: "zdotaccent", width: 500 },
  Ą: { byte: 137, glyph: "Aogonek", width: 667 },
  Ć: { byte: 138, glyph: "Cacute", width: 722 },
  Ę: { byte: 139, glyph: "Eogonek", width: 667 },
  Ł: { byte: 140, glyph: "Lslash", width: 556 },
  Ń: { byte: 141, glyph: "Nacute", width: 722 },
  Ó: { byte: 142, glyph: "Oacute", width: 778 },
  Ś: { byte: 143, glyph: "Sacute", width: 667 },
  Ź: { byte: 144, glyph: "Zacute", width: 611 },
  Ż: { byte: 145, glyph: "Zdotaccent", width: 611 },
};

/** Typographic characters flattened to ASCII so no byte falls outside the map. */
const ASCII_FALLBACK: Record<string, string> = {
  "\u00a0": " ", // no-break space: the group separator of pl-PL numbers
  "\u202f": " ", // narrow no-break space: the group separator of fr-FR and others
  "\u2009": " ",
  "\u201e": '"',
  "\u201d": '"',
  "\u201c": '"',
  "\u2019": "'",
  "\u2013": "-",
  "\u2014": "-",
  "\u2026": "...",
  "\u20ac": "EUR",
};

/** Helvetica AFM widths (per 1000 units) for the ASCII range. */
const HELVETICA_WIDTHS: Record<string, number> = {
  " ": 278, "!": 278, '"': 355, "#": 556, $: 556, "%": 889, "&": 667, "'": 191,
  "(": 333, ")": 333, "*": 389, "+": 584, ",": 278, "-": 333, ".": 278, "/": 278,
  "0": 556, "1": 556, "2": 556, "3": 556, "4": 556, "5": 556, "6": 556, "7": 556,
  "8": 556, "9": 556, ":": 278, ";": 278, "<": 584, "=": 584, ">": 584, "?": 556,
  "@": 1015, A: 667, B: 667, C: 722, D: 722, E: 667, F: 611, G: 778, H: 722,
  I: 278, J: 500, K: 667, L: 556, M: 833, N: 722, O: 778, P: 667, Q: 778,
  R: 722, S: 667, T: 611, U: 722, V: 667, W: 944, X: 667, Y: 667, Z: 611,
  "[": 278, "\\": 278, "]": 278, "^": 469, _: 556, "`": 333,
  a: 556, b: 556, c: 500, d: 556, e: 556, f: 278, g: 556, h: 556, i: 222,
  j: 222, k: 500, l: 222, m: 833, n: 556, o: 556, p: 556, q: 556, r: 333,
  s: 500, t: 278, u: 556, v: 500, w: 722, x: 500, y: 500, z: 500,
  "{": 334, "|": 260, "}": 334, "~": 584,
};

/** Approximate rendered width in points. Regular metrics stand in for bold. */
export function textWidth(text: string, size: number): number {
  let units = 0;
  for (const raw of text) {
    const ch = ASCII_FALLBACK[raw] ?? raw;
    for (const c of ch) units += POLISH[c]?.width ?? HELVETICA_WIDTHS[c] ?? 556;
  }
  return (units / 1000) * size;
}

/**
 * Break text into rows no wider than `maxWidth`. Breaks at spaces; a single word wider than the column is
 * split by character, because a document that drops part of a name is worse than one that hyphenates
 * badly. Line feeds in the input start a new row. Always returns at least one row.
 */
export function wrapText(text: string, size: number, maxWidth: number): string[] {
  const rows: string[] = [];
  for (const paragraph of text.split(/\r?\n/)) {
    let row = "";
    for (const word of paragraph.split(/\s+/).filter(Boolean)) {
      const candidate = row ? `${row} ${word}` : word;
      if (textWidth(candidate, size) <= maxWidth) {
        row = candidate;
        continue;
      }
      if (row) rows.push(row);
      row = "";
      let piece = "";
      for (const ch of word) {
        if (piece && textWidth(piece + ch, size) > maxWidth) {
          rows.push(piece);
          piece = "";
        }
        piece += ch;
      }
      row = piece;
    }
    if (row) rows.push(row);
  }
  return rows.length > 0 ? rows : [""];
}

/** Encode a string into PDF literal-string bytes: delimiters escaped, Polish letters remapped. */
function encodeText(text: string): string {
  let out = "";
  for (const raw of text) {
    const ch = ASCII_FALLBACK[raw] ?? raw;
    for (const c of ch) {
      const pol = POLISH[c];
      const code = pol ? pol.byte : c.charCodeAt(0);
      if (code === 0x28 || code === 0x29 || code === 0x5c) out += `\\${c}`;
      else if (code >= 32 && code <= 126) out += String.fromCharCode(code);
      // Bytes 128 to 159 belong to the Polish map above, so only a mapped letter may produce one.
      else if (pol || (code >= 160 && code <= 255)) out += `\\${code.toString(8).padStart(3, "0")}`;
      else out += "?";
    }
  }
  return out;
}

function num(n: number): string {
  return Number.isInteger(n) ? String(n) : n.toFixed(2);
}

/**
 * Colour operator for a fill (`g`, `rg`) or a stroke (`G`, `RG`). An unparseable value falls back to black:
 * a malformed operator takes the whole page down, a black rectangle is only ugly.
 */
function colorOp(color: PdfColor, stroke: boolean): string {
  if (typeof color === "number") return `${num(color)} ${stroke ? "G" : "g"}`;
  const hex = color.replace("#", "");
  const full = hex.length === 3 ? [...hex].map((c) => c + c).join("") : hex;
  if (!/^[0-9a-fA-F]{6}$/.test(full)) return stroke ? "0 G" : "0 g";
  const channel = (i: number): string => num(parseInt(full.slice(i, i + 2), 16) / 255);
  return `${channel(0)} ${channel(2)} ${channel(4)} ${stroke ? "RG" : "rg"}`;
}

/** One A4 portrait page. The y axis runs upward from the bottom-left corner, as in PDF space. */
export class PdfPage {
  readonly ops: string[] = [];

  text(x: number, y: number, content: string, opts: PdfTextOptions = {}): void {
    const size = opts.size ?? 9;
    const font = opts.font === "bold" ? "/F2" : "/F1";
    const startX = opts.align === "right" ? x - textWidth(content, size) : x;
    const ink = colorOp(opts.color ?? opts.gray ?? 0, false);
    this.ops.push(
      `q ${ink} BT ${font} ${num(size)} Tf 1 0 0 1 ${num(startX)} ${num(y)} Tm (${encodeText(content)}) Tj ET Q`,
    );
  }

  line(x1: number, y1: number, x2: number, y2: number, width = 0.7, color: PdfColor = 0.75): void {
    this.ops.push(`q ${num(width)} w ${colorOp(color, true)} ${num(x1)} ${num(y1)} m ${num(x2)} ${num(y2)} l S Q`);
  }

  rect(x: number, y: number, w: number, h: number, color: PdfColor = 0.94): void {
    this.ops.push(`q ${colorOp(color, false)} ${num(x)} ${num(y)} ${num(w)} ${num(h)} re f Q`);
  }

  /**
   * Fill many rectangles with one graphics state and one paint. A QR symbol is hundreds of module runs,
   * and a `q ... Q` block per rectangle would grow the content stream by an order of magnitude.
   */
  rects(boxes: ReadonlyArray<{ x: number; y: number; w: number; h: number }>, color: PdfColor = 0): void {
    if (boxes.length === 0) return;
    const paths = boxes.map((b) => `${num(b.x)} ${num(b.y)} ${num(b.w)} ${num(b.h)} re`).join(" ");
    this.ops.push(`q ${colorOp(color, false)} ${paths} f Q`);
  }
}

/** Multi-page document builder. */
export class PdfDoc {
  private readonly pages: PdfPage[] = [];

  addPage(): PdfPage {
    const page = new PdfPage();
    this.pages.push(page);
    return page;
  }

  get pageCount(): number {
    return this.pages.length;
  }

  /** Assemble the file: catalog, page tree, fonts and encoding, xref, trailer. */
  build(): Uint8Array {
    if (this.pages.length === 0) this.addPage();

    const differences = `[128 ${Object.values(POLISH)
      .sort((a, b) => a.byte - b.byte)
      .map((p) => `/${p.glyph}`)
      .join(" ")}]`;

    // Fixed ids: 1 catalog, 2 pages, 3 F1, 4 F2, 5 encoding, then a (page, content stream) pair per page.
    const objects: string[] = [];
    const pageIds = this.pages.map((_, i) => 6 + i * 2);
    objects.push(`<< /Type /Catalog /Pages 2 0 R >>`);
    objects.push(`<< /Type /Pages /Kids [${pageIds.map((id) => `${id} 0 R`).join(" ")}] /Count ${this.pages.length} >>`);
    objects.push(`<< /Type /Font /Subtype /Type1 /BaseFont /Helvetica /Encoding 5 0 R >>`);
    objects.push(`<< /Type /Font /Subtype /Type1 /BaseFont /Helvetica-Bold /Encoding 5 0 R >>`);
    objects.push(`<< /Type /Encoding /BaseEncoding /WinAnsiEncoding /Differences ${differences} >>`);

    for (const page of this.pages) {
      const contentId = objects.length + 2; // the stream object follows the page object
      objects.push(
        `<< /Type /Page /Parent 2 0 R /MediaBox [0 0 ${A4.width} ${A4.height}] ` +
          `/Resources << /Font << /F1 3 0 R /F2 4 0 R >> >> /Contents ${contentId} 0 R >>`,
      );
      const stream = page.ops.join("\n");
      objects.push(`<< /Length ${stream.length} >>\nstream\n${stream}\nendstream`);
    }

    // The second line is four bytes above 127, which marks the file as binary for transfer tools.
    let body = "%PDF-1.4\n%âãÏÓ\n";
    const offsets: number[] = [];
    objects.forEach((obj, i) => {
      offsets.push(body.length);
      body += `${i + 1} 0 obj\n${obj}\nendobj\n`;
    });

    const xrefStart = body.length;
    body += `xref\n0 ${objects.length + 1}\n0000000000 65535 f \n`;
    for (const off of offsets) body += `${String(off).padStart(10, "0")} 00000 n \n`;
    body += `trailer\n<< /Size ${objects.length + 1} /Root 1 0 R >>\nstartxref\n${xrefStart}\n%%EOF\n`;

    // Every char in `body` is one byte by construction: encodeText emits ASCII only.
    const bytes = new Uint8Array(body.length);
    for (let i = 0; i < body.length; i++) bytes[i] = body.charCodeAt(i) & 0xff;
    return bytes;
  }
}
```

## What the writer guarantees

- **Deterministic output.** No timestamp, no random id and no compression: the same calls produce the same
  bytes, so a test can assert on them.
- **One byte per character.** `encodeText` writes every byte above 126 as an octal escape, which is why
  `stream.length` is the stream's byte length and every xref offset is right. A change that puts a raw
  non-ASCII character into an op breaks `/Length` and each offset after it.
- **Nothing is dropped silently.** `wrapText` keeps every character of its input, and a character the font
  cannot show prints as `?` in place, where a reader of the document can see it.

## What it does not do

Images, embedded fonts, compression, links, form fields, page sizes other than A4. Each is a reason to
reach for a library, and none is needed to print an invoice. The one that a host is most likely to need,
an embedded font for scripts outside Latin-1 and Polish, is a design in [operations.md](operations.md).
