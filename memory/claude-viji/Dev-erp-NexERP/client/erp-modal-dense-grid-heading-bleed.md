---
name: erp-modal-dense-grid-heading-bleed
description: Master-form inline section headings can visually bleed fields into the wrong subsection due to dense CSS grid packing
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 7dd09634-a73e-4ab6-b9e4-832f418187c8
---

DynamicModalForm defaults to `denseGrid={true}` → `grid-auto-flow: row dense`. When a
section ends on a partial row (e.g. a lone trailing checkbox leaving empty cells) and is
followed by a full-width inline `custom` section heading (`gridColumn:"1 / -1"`), dense
packing BACK-FILLS those empty cells with the fields that come AFTER the heading in the DOM
— so those fields render visually ABOVE their own subsection heading even though the DOM
order is correct.

**Symptom seen (item master, 2026-07-15):** Company/Branch (first "Reference Links" fields)
appeared on the "Is Active" row inside Core Details. Core Details had 10 visible fields
(9 configured + unmapped SKU) → Is Active alone in a 3-col row → grid pulled Company/Branch
into the 2-cell gap.

**Fix:** pass `modalFormDenseGrid={false}` on the page's `<CrudMasterPage>` (prop forwards
to DynamicModalForm `denseGrid`). Non-dense = `grid-auto-flow: row`, so headings break to a
fresh row and nothing back-fills across them. Safe here because every other wide item in the
item form is followed by full-width content, and the EAN/Rules&Status block uses explicit
`gridRowStart` (unaffected by dense).

**Why:** the widget-config reorder ([[erp-widget-masters-config]]) makes hardcoded
gridRowStart impractical for reorderable sections, so the layout must stay flow-based —
disabling dense is the clean lever, not per-field column pinning (which fails: constraining
only the first post-heading field still lets the second back-fill).

**How to apply:** for any master form with inline `custom` subsection headings, if fields
look like they're in the wrong subsection, check DOM/interleaved order FIRST (it's usually
correct) before touching field order — the bug is CSS packing, not ordering. Verify visually,
not by label-only DOM extraction (headings aren't `<label>`s, so a label-only list hides
where the heading falls). Browser-drive recipe: [[erp-client-ui-browser-verification]].
