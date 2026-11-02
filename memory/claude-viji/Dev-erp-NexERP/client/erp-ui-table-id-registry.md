---
name: erp-ui-table-id-registry
description: Grid layouts are found by ui_table NAME (Desktop rows) through lib/ui-tables, never by a hardcoded ui_tbl_id
metadata:
  type: project
---

Since 2026-09-18 no screen hardcodes a `fixed.ui_tables.ui_tbl_id`. `lib/ui-tables`
holds the registry: each grid is keyed by the `ui_tbl_name` it is configured under
(`ITEM MASTER - PRICE`, `QUOTATION - LINES`, `SALE BILL - LINES`, `SO - ITEM`,
`CHARGES`, `OPENING STOCK - LINES`, `PHYSICAL STOCK - LINES`, `ITEM QTY WISE PRICE`,
`ITEM MASTER - REORDER`, `ITEM MASTER - ALT BARCODES`), plus a `fallbackId` used only
until the directory loads.

- Directory source: `getUiTableDirectory` in `store/api/metadataApi.ts` →
  `/configured-grid-sql/run?grid_id=35` (the UI Table Master list; its SQL already
  filters `ui_tbl_device_type='Desktop'`, and the registry filters again).
  `/ui-table-masters/get` with no id would drag ~900 column rows along.
- React: `useUiTableId(key)`. Off React (payload builders): `getUiTableId(key)`,
  which reads what `transformResponse` primed.
- `CrudMasterPage` takes `uiTableKey`; an explicit `uiTableId` still wins.

**Why:** ids are per-deployment. EAN was 4, opening stock 5, physical stock 6,
quotation 23, charges 26 — none of those rows exist any more, and a dead id fails
silently (the endpoint returns no table, the grid quietly paints its hardcoded
fallback columns), so screens ran for months on the wrong layout.

**How to apply:** adding a grid means adding a `UI_TABLES` entry, not a number.
Web-only duplicate ui tables are gone — browser and Qt share one Desktop row, so a
width/visibility save from the browser edits what the desktop screen shows. Related:
[[erp-ui-table-layout-settings]], [[erp-ui-table-width-is-not-a-percent]],
[[erp-grep-skips-binary-files]] (item-master-page.tsx and quotation-entry-view.tsx
are the two files plain `grep -r` skips — both hold ui-table call sites).

Caveat carried forward: Quotation still reads its widths as `px`
(`ITEM_COLUMN_WIDTH_UNIT`) while its layout is now a Desktop row storing Qt
fractional percents — Sale Bill/Sale Order use `qtPercent`. Left as-is on request.
