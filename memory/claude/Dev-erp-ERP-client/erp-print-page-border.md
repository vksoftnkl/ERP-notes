---
name: erp-print-page-border
description: "Whole-page border for GRAPHIC print templates (built 2026-10-13) — `pageBorder` on the definition ROOT (not paper), the two key-by-key rebuild traps, and where the designer exposes it"
metadata:
  node_type: memory
  type: project
  originSessionId: 7102eab9-29b7-4d7a-be7d-a14231289db7
  modified: 2026-10-13T08:26:30.791Z
---

`pageBorder` = `{insetMm: number|null, widthPt, color, radiusMm, style}` on the
definition root. Null inset = frame on the margin line; a number = fixed
distance from the paper edge on all sides. Absent = no border.

- **Root key, not `paper`**: the paper is rebuilt from the version row on every
  render, so anything on it is thrown away. Two places rebuild the definition
  key by key and had to name it explicitly: server
  `definition-from-version.ts` (body → candidate) and client
  `canvasBridge.toTemplateDefinition`. Same trap as band `flowLayout` in
  `normaliseBand` ([[erp-print-flow-layout]]).
- Engine: `drawPageBorders()` runs LAST in `run()` and `unshift`s a rect onto
  each page — after `suppressLastPageBands` and deferred page-number text,
  because both address primitives by index. Skipped in GRID mode and on
  continuous paper (heightMm null); a zero/negative frame draws nothing.
- Client: `selectPageBorder`, `pageBorderSet(patch|null)` (add is its own undo
  step; edits coalesce under "pageBorder"; empty colour → default), the
  "Page border" section in `PaperSection`, `.pageBorderGuide` on the canvas
  sheet (sides only — the canvas is band strips, Preview shows the real page),
  and `validate.ts` problems (server-refused values = error, unprintable = warning).
- Verified 2026-10-13: server PDF preview on SaleInvoiceA5 rev 10 (margin and
  5mm-inset variants) + headless designer panel. Not yet saved on any revision.

Since 2026-10-13 it also takes a line `style` (QtRPT pen styles; DOUBLE draws two rects) and has a sibling root key `watermark` with the same rebuild traps — see [[erp-print-qtrpt-parity]].
