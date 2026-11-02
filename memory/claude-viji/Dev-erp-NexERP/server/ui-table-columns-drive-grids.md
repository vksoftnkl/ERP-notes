---
name: ui-table-columns-drive-grids
description: "Transaction-screen grids render only columns configured in fixed.ui_table_columns — adding one takes a client meaning, a text-field registration, and a seeded DB row"
metadata: 
  node_type: memory
  type: project
  originSessionId: b5bd0dbd-6a83-4250-ae95-74215812ebf8
  modified: 2026-08-08T06:31:17.431Z
---

The ERP client's transaction grids (quotation item grid, charges grid) do NOT render their local column list. `resolveColumns` (ERP client `features/sales/quotation/quotation.utils.ts`) walks the `fixed.ui_table_columns` rows fetched for a `ui_tbl_id`, matches each row's `ui_tbl_clm_name` against the local meanings list on a **normalized token** (lowercased, non-alphanumerics stripped — `normalizeColumnToken`), and **skips any local meaning with no DB row**. Rows sort by `ui_tbl_clm_no`, and the meanings list is declared in that same order. Only when zero rows come back does it fall back to showing every local meaning.

ui table ids (`fixed.ui_tables`): **23** = "Quotation-item", the quotation item grid this client loads (`ITEM_GRID_UI_TABLE_ID`, widths in px); **21** = "CHARGES"; **18** = "Quotation Item Table", the legacy Qt screen's own layout — this client never loads it and its widths are fractional percents, so do not mirror column changes there without checking who consumes it.

**Why:** a new `sale_quotation_item` column wired only through DTO → payload → draft state looks complete and typechecks, but has no cell to type into and saves NULL forever. The DB config is the missing half, and nothing in the client fails to signal it.

**How to apply:** adding an item-grid column takes three edits, not one —
1. `itemColumn("Token", kind, align, { write: "<DraftLine key>", read: "<DraftLine or PricedLine key>" })` appended to `ITEM_COLUMN_MEANINGS` in `quotation.constants.ts` (last = highest `ui_tbl_clm_no`), plus the "All N configured columns" / `ITEM_GRID_UI_TABLE_ID` doc counts.
2. For a text cell, add the DraftLine key to `TEXT_LINE_FIELDS` in `components/quotation-entry-view.tsx` — otherwise `setLineField` runs `parseCell` and stores a number in a string field. See [[quotation-entry-view-nul-sentinel]] before editing that file.
3. An idempotent `INSERT` into `fixed.ui_table_columns` as a `prisma/seed/*.sql` file in the ERP server repo (the repo's convention for config/seed data — migrations do not carry these rows). Derive `ui_tbl_clm_no`/`position` as `MAX(no) + 1` for the table and copy `ui_tbl_clm_created_by` from an existing row of the same table rather than hardcoding either; guard with `NOT EXISTS` on the normalized name so re-runs insert nothing. Example: `prisma/seed/Quotation_Item_Grid_ItemSize_Column.sql` (2026-08-08, the `sqi_item_size` column).

Distinct from [[configured-grid-param-convention]], which is about `fixed.grid_details.grid_sql` and its `p_*` params — a different mechanism for a different kind of grid.
