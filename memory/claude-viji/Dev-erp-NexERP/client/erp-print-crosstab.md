---
name: erp-print-crosstab
description: "CROSSTAB element (dynamic-column pivot) in the print engine + designer — where it expands, why it is banned from repeating bands, and the stale vocab generator it fixed"
metadata: 
  node_type: memory
  type: project
  originSessionId: 0911b52d-9351-4449-8688-dfc861e70b41
  modified: 2026-09-01T13:42:35.072Z
---

Built 2026-09-01, full stack, GRAPHIC/PDF only. A `CROSSTAB` element prints a
whole pivot table whose COLUMN COUNT COMES FROM THE DATA.

**It expands in the LAYOUT PASS, not in a renderer.** `engine/layout/crosstab.ts`
turns the model into ordinary text/line/rect primitives, so **no renderer was
touched** — pdfkit, ESC/POS and ESC/P print one without knowing crosstabs exist.
That is the payoff of `layout-tree.types.ts`'s "one layout pass, many renderers".
Three phases, deliberately split: `buildCrosstabModel` (aggregate, no paper
knowledge) → `planCrosstab` (fit columns into the `w` budget) → `sliceCrosstab` /
`emitCrosstab` (one page's rows at a time). The split is what makes pagination
possible: the engine asks "how many rows fit in the 40mm left" without drawing.

## Traps found the hard way

- **`needsPageBreak` is the WRONG routing test** for a crosstab. It deliberately
  answers *false* for a band taller than a whole page (breaking would loop), so
  routing on it meant a 611mm table drew entirely on page 1 and overflowed. Added
  `fitsInRemainingBody` — the plain geometric test — for this.
- **Only four bands may hold one** (`CROSSTAB_BANDS`: REPORT_HEADER, SUMMARY,
  REPORT_FOOTER, NO_DATA). DETAIL and the GROUP bands repeat and a crosstab has
  no group filter, so it would print the same complete table once per row/group.
  PAGE_HEADER/PAGE_FOOTER are worse: `visibleElementCount` clips a suppressed
  band by primitive COUNT, and a crosstab's count is variable. Group-scoped
  crosstabs are a real future feature, deliberately not this one.
- **`crosstabPlans` is cached by element id alone.** Sound *only* because those
  four bands are all in the schema's `singletonBands`. Lifting the band
  restriction means keying by group path too.
- `bandHeight` grows for a crosstab **whether or not `autoGrow` is set** —
  autoGrow is a promise about wrapped text a designer could eyeball; a row count
  the query decides is not something a declared height can be right about.
- Folds and row totals **re-aggregate the source rows**, they do not add up
  already-aggregated cells — otherwise folding three `avg` columns prints an
  average of averages. `Accumulator`/`accumulate`/`readAccumulator` are now
  exported from `aggregate.accumulator.ts` so both paths agree on what `count`
  and `avg` mean.

## Client side

`features/print-designer`: new kind threads through types, vocabulary, defaults,
validate, geometry (`isBoxLike`), ElementBox (a SKETCH — three sample columns,
never a preview), KindSection (three property sections), toolbar + tree icons,
and **`ElementPatch` in designerSlice** (the flattened intersection — a new kind
must be added there or every property edit fails to typecheck).
Client `validate.ts` mirrors every server refusal so the designer marks the field
instead of the save returning a JSON path.

**`scripts/gen-print-vocab.mjs` was pointing at `src/modules/reporting/...`,
which no longer exists** (the engine moved to `modules/settings/print-render`) —
so `npm run gen:print-vocab` had been dead, and the parity vitest was frozen
against a stale JSON. Paths fixed; regenerating showed the rest of the
vocabulary had NOT drifted.

Verified: 31 new server tests, a real 2-page A4 PDF rasterised and eyeballed
(repeated header, bold totals row, footer after the table), and the element
placed in a live headless designer per [[erp-client-ui-browser-verification]].

## The shipped crosstab design (2026-09-02)

`quotation-rate-matrix-a4.json` — an A4 quotation whose SUMMARY band cross-tabs
**HSN (rows) x GST rate slab (columns) -> sum(taxable_amt)**. It pivots the
quotation's EXISTING `items` dataset, so installing it is a `ptv_body` update
with no new `print_template_dataset` row. Byte-identical copies live in the
client `lib/__fixtures__/` and the server `__fixtures__/` (plus a
`.sample.json` only the server has, since a crosstab render test with no rows
asserts nothing). Wired into `shipped-templates.test.ts`,
`template-definition.schema.spec.ts` and `crosstab.spec.ts`.

Two facts it is built on, both verified against the engine:
- **`paper.orientation` is a LABEL** — nothing reads it. `pdfkit.renderer` takes
  `widthMm`/`heightMm` straight through, so landscape means writing 297x210
  yourself. Reachable in-app because `ptvWidthMm`/`ptvHeightMm` override the
  preset in both `paperFor` implementations.
- **The body only supplies BANDS.** Paper comes from the version row and
  datasets from the ptd rows ([[erp-printing-module]]), so a layout JSON alone
  cannot reproduce a report — the version's paper columns and dataset rows are
  part of the deliverable.

Accepted wart: the item-grid column headings live in PAGE_HEADER (the only band
that repeats), so on a page carrying only crosstab continuation they print with
no rows beneath. The existing QUOTATION_A4 design behaves the same way; there is
no expression that can know whether a page has detail rows.

**An unbound crosstab breaks the WHOLE template** (found in use 2026-09-02).
`dataset` is a required zod `identifier`, so a crosstab left at `""` makes the
entire definition fail to parse: preview returns three confusing
`bands.N.elements.M.dataset` messages and the template stops rendering
altogether — not just that element. `elementAdded` now binds a new crosstab to
the first `cardinality: "many"` dataset, so inserting one is always safe;
it stays `""` only when the template has nothing repeating, which the designer's
problem list reports as "has no dataset to pivot".
Related trap: on an EDITABLE revision, Preview posts the LIVE canvas body
(`previewsUnsaved: editable` in `LayoutCanvasScreen`), so an unsaved broken
element fails the render even though nothing was saved.

**There is no in-app Import.** `DesignerMenuBar`'s "Export JSON" calls
`/reports/templates/:id/export`, one of the dead `/reports` routes, and no
import counterpart exists. Loading a layout means writing `ptv_body` directly.

Related: [[erp-print-template-contract]], [[erp-print-designer-ui]], [[erp-print-flow-layout]],
[[erp-printing-module]].
