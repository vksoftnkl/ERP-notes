---
name: erp-grid-columns-config
description: "How ERP client master-page grid columns are configured (grid_id map, configured-grid-sql/columns)"
metadata: 
  node_type: memory
  type: project
  originSessionId: f0e4ef84-196d-4387-ace6-145a47300a81
---

Master pages (CrudMasterPage) render table columns from `configured-grid-sql/columns?grid_id={id}` (RTK query `getGridColumns` in `store/api/metadataApi.ts`; normalizer in `store/slices/gridColumnsSlice.ts`).

Since 2026-09-18 the `grid_id` per page comes from the **grid registry** — a screen
passes `gridKey` (a name, e.g. `stateList`), CrudMasterPage resolves it through
`lib/configured-grids`, and uses the resolved id for BOTH the rows
(`apiEndpoints.list` is built for it) and the columns. See
[[erp-configured-grid-registry]]. `features/masters/shared/configured-grid-detail-ids.ts`
(`CONFIGURED_GRID_DETAIL_IDS_BY_TABLE`, keyed by `gridTableName`) is now only the
last-resort fallback for screens that name no grid; its ids were moved to the
Desktop rows too. An explicit `gridDetailId` prop still wins over both.
When an id resolves, CrudMasterPage also calls `grid-details/get?gridId={id}` for
grid metadata used by the column-reorder save payload.

**Column widths (2026-09-18):** a list column is sized by `grid_columns.grid_column_px`
("246px") and by nothing else. `grid_column_width` is the desktop client's Qt fraction —
the browser neither sizes from it nor writes it, so the two clients stop overwriting each
other. `PUT /grid-details/column-width` now takes both fields optionally and writes only
what was sent (`hasOwnProperty` guard, same as the ui-table endpoint); the browser sends
`{ grid_column_id, grid_column_px }` alone. A column with no px value takes the table's own
baseline width, so existing rows (all px NULL) look unchanged until dragged. The full-row
save (`/grid-columns/create`, used by visibility/filter/reorder) passes the stored fraction
straight back and carries px along, which is what keeps a drag alive through those saves —
never let a px value reach `grid_column_width`, it is read back scaled.

**Why:** centralizes per-page grid ids and drives columns off the configured-grid-sql endpoint.
**How to apply:** to wire a new master page, create its grid in `fixed.grid_details` (+`grid_columns`) as a **Desktop** row, then add a `CONFIGURED_GRIDS` entry naming it and pass `gridKey` — not a number. Verify with [[erp-auth-endpoint-verification]].
