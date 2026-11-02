---
name: erp-ui-table-width-is-not-a-percent
description: fixed.ui_table_columns width is a percent-of-grid number that must be converted to px; using it as a CSS % on a table-layout:fixed table blows the table to 33 million pixels
metadata: 
  node_type: memory
  type: reference
  originSessionId: febff91c-b70f-4e0c-9cf0-588ab8852047
  modified: 2026-07-30T08:13:01.880Z
---

**Since 2026-09-18 that column is a FALLBACK only.** A grid sizes from
`ui_tbl_clm_px` ("212px", the exact width a drag left the column at) wherever it is set, and
the browser writes nothing else: a resize PUTs `{ uiTblClmId, uiTblClmPx }` to
`/ui-table-masters/column-width`, and every full-row save (stock/item grids, which POST whole
columns to `/ui-table-masters/create`) hands the stored fraction back untouched and carries px
alongside. The endpoint now writes each width only when the body names it, so the browser and
the Qt client stop overwriting each other. Same contract as grids — see
[[erp-grid-columns-config]]. The conversion below still applies to any column nobody has
dragged yet (every row is px NULL until then), which is why it is kept.

`fixed.ui_table_columns.ui_tbl_clm_column_width` (served by
`GET /ui-table-masters/get?uiTableId=N`) is a **percent-of-the-Qt-grid** number, and the
values do **not** sum to 100: ui table 18 ("Quotation") has 76 visible columns summing to
~460%, ui table 21 ("CHARGES") ~180%.

Rendering it as a CSS `%` in a `<colgroup>` on a `table-layout: fixed` table whose width is
`min-width: max-content` produces a **33,554,432 px wide table** — the percentages resolve
against a width that is itself derived from them, so it runs away. Symptom: the grid looks
empty (one column fills the viewport, headers appear blank), and Playwright reports
"the sticky serial cell intercepts pointer events" for every cell to its right.

Fix (used by Quotation Entry, `features/sales/quotation/quotation.utils.ts`):
- convert to **pixels**: `px = max(34, round(configuredWidth * 11))` — 11 px per unit
  reproduces the Qt proportions (9.6 → ~106 px for a Description column);
- set the table's own `width` inline to the sum of the column pixel widths plus the row-action
  column, and keep `min-width: 100%` in CSS so a narrow grid still fills its shell;
- never leave `min-width: max-content` on a fixed-layout table with sized columns.

FLUID UNITS (2026-08-22): those pixels are still what the layout STORES, but the quotation
screen no longer DRAWS them as px. `page.module.scss` declares `--erp-q-u` on `:root`
(clamp 14px…20px, `0.2rem + 0.5vw + 0.35vh`) and every length in the file is `u(n)` — an SCSS
helper emitting `calc(n * var(--erp-q-u))` — so the whole screen scales with the window
(measured: labels 10.1px @1366 → 11.9 @1920 → 14.4 @2560; it passes through the old fixed size
at ~1850 wide). The grids render `<col>` widths in `em` via `scaledWidth(px)` (quotation.utils,
CONFIG_PX_PER_UNIT = 16) and `.grid { font-size: var(--erp-q-u) }` makes the table's `em` that
unit, so columns scale with the type. Consequence for use-column-resize: a mouse delta is
SCREEN px, the stored width is CONFIG px — the drag divides by `unitScaleOf(table)` (the table's
computed font-size / 16) or a drag at 2560 would widen the column 25% too fast. Verified in
Chromium: a 100px drag grows the column exactly 100px at unit 20px. The var must stay on
`:root`: modals/menus are portaled to body and an unresolvable var makes every calc invalid.
The sale-order screen imports this stylesheet, so it scales too, except its own local px
accents (credit panel, chips) in features/sales/sale-order/page.module.scss.

Related: [[erp-grid-columns-config]] (the *other* config system, `fixed.grid_columns`, whose
`grid_column_width` the client normalizer already renders as `"<n>%"` — that one is only used
for list/picker grids with few columns, where the runaway does not show).
