---
name: erp-print-qtrpt-parity
description: "Print designer QtRPT parity (2026-10-13) — field frames, line styles, charts, rich text, highlights, report pages, watermark etc.; where each lives, the engine traps, and what was deliberately not ported"
metadata:
  type: project
---

Built 2026-10-13 on the user's request "compare with qtrpt designer and add all
that's missing" (started as "fields need borders like QtRPT"). Uncommitted in
both repos at the end of that session.

**QtRPT reference:** only a binary install exists, at
`/home/vk/VKPOS/VKSoftwares/Tools/QtRpt/bin/QtRptDesigner`. `strings` on it gives
the full property model (borderTop/…/borderStyle, highlighting, rotate,
autoHeight, arrowStart, watermark, startNewNumeration, chart props) and every
toolbar/dialog label — that is how the gap list was made.

**Schema additions** (server `template-definition.schema.ts`, mirrored in client
`types/template-definition.ts`, all optional/defaulted so old bodies parse):
element `border{top,right,bottom,left,widthPt,color,style}`, `stretchWithBand`,
`group`; text `rotation` 0/90/180/270, `markup`, `highlights[]`, font
`strikeout`, align `justify`; LINE `arrowStart/arrowEnd/arrowSizeMm`; RECT
`shape` ELLIPSE/TRIANGLE/RHOMBUS; `style.lineStyle`; new kind CHART
(BAR/STACKED_BAR/LINE/PIE+hole, CHART_BANDS = CROSSTAB_BANDS); crosstab
`totalsFill`, `showSubtotals`, `subtotalLabel`; band `reportPage`,
`startNewPage`, `startNewNumeration`; root `watermark`; pageBorder `style`;
8 more barcode symbologies (incl. 2D pdf417/datamatrix/aztec).

**Engine traps fixed on the way:**
- Last-page header/footer suppression used to count ELEMENTS; frames, fills and
  arrowheads add primitives, so it now records the primitives each page's
  header/footer actually drew (`headerPrimitiveCounts`).
- Page-number text is re-laid-out in pass 2 by a stored `rebuild` closure
  (rotation/justify/markup/highlights all shape it), not by re-wrapping text.
- Text colour expressions were evaluated against the ROOT context (row-based
  colours never worked); `textLook` now uses the band context.
- Text `fill` and `style.padding` were canvas-only; both now print.
- New primitives: `path` (mm SVG, absolute M/L/C/Z only, renderer scales the
  numbers), `dash` on line/rect, `rotation`/`opacity`/`justify`/`rich` on text.
- Report pages = separate LayoutRun per `reportPage`, pages concatenated, each
  numbered on its own; singletons are per report page.

**Markup** is parsed from the TEMPLATE before evaluation (data never becomes
tags). Server `engine/layout/markup.ts` and client `lib/markup.ts` are twin
parsers — parity cases are in both `qtrpt-parity.spec.ts` and `markup.test.ts`;
change both together.

**Key-by-key rebuild traps (again):** client `canvasBridge.normaliseBand` must
name `reportPage`/`startNewPage`/`startNewNumeration`, and both
`definition-from-version.ts` and `toTemplateDefinition` must name `watermark` —
same trap as [[erp-print-page-border]] and [[erp-print-flow-layout]].

**Designer:** FormatToolbar (QtRPT's format strip), Drawing select + Rich-text
button + W=/H= + front/back + group/ungroup + mm/inch in DesignerToolbar,
ReportPageTabs under the canvas (view.reportPage filters `selectOrderedBands`;
band indexes stay global), Border/Conditional-formatting panel sections.
Group selection keeps CLICK order (same width/height uses the first selected).

**Not ported, on purpose:** QtRPT's Script Editor (arbitrary script on the
server — the expression engine covers computed values); static chart data (use a
SQL dataset); settings/plugins/update-check/about.

Verified: server qtrpt-parity.spec + showcase PDF rasterised with pdftoppm;
designer driven over CDP with a faked API ([[erp-client-ui-browser-verification]]).
Related: [[erp-print-designer-ui]].
