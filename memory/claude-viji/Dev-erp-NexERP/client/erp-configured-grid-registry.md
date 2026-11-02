---
name: erp-configured-grid-registry
description: "Configured lists resolve their grid_id by NAME from Grid Master's Desktop rows (lib/configured-grids); the legacy web grids are unreachable in the UI"
metadata:
  node_type: memory
  type: project
  originSessionId: 504f0b88-1015-4da8-b3f0-998daab58170
  modified: 2026-10-03T10:45:27.025Z
---

Since 2026-09-18 screens name the grid they read instead of numbering it.
`lib/configured-grids` keys each list by its `grid_name` (`MAIN LIST - ITEMS`,
`TXN MAIN LIST - QUOTATION`, `CHARGES MAIN LIST`, …) with a `fallbackId` and a
`deletedParam`. `CrudMasterPage` takes `gridKey` and derives from it BOTH
`apiEndpoints.list` (`gridRunEndpoint`) and the configured-columns grid, so
`list:` and `gridDetailId` are gone from migrated screens.

- Directory: `getGridDirectory` (metadataApi) → `/configured-grid-sql/run?grid_id=34`
  ("GRID MASTER LIST"). Ids 34 (grids) and 38 (ui tables) are the only two numbers
  left in the app — everything resolves through them. Both are primed app-wide by
  `components/layout/configured-directories.tsx` so `getGridId`/`getUiTableId`
  (used inside RTK Query `query:` builders) read a real id.
- React: `useGridId(key)`. Off React: `getGridId(key)`.
- Deleted-rows toggle: `buildGridDeletedParam(key, wantDelete)` sends the grid's own
  token (`istm_is_deleted`, `iled_is_deleted`, …) plus the legacy `wantdelete`;
  unmatched keys are ignored server-side so one call fits either grid.
  The reverse is NOT safe: if the grid's SQL binds a token (`comp_is_deleted =
  icomp_is_deleted`) but the registry says `deletedParam: null`, the token goes
  unsent and the run 400s `column "icomp_is_deleted" does not exist`. Company (52),
  Branch (56) and Items (67) hit this on 2026-10-01 — the user is converting grids
  from `= false` to an `i<prefix>_is_deleted` token in Grid Master one at a time, so
  expect more. Check `grid_modified_on` to see which changed. 2026-10-03: Customers (65),
  Emp Departments (73), Freight (74), Loading (76), Employees (77) — fixed. Department,
  Freight and Loading also had no `buildListQuery` at all (CrudMasterPage's default query
  sends no `grid_param`), so they needed the Designation-style toggle, not just the
  registry. Converted Desktop grids no client screen reaches yet: 45, 50, 98, 127, 131.
- A QUOTED token the caller doesn't send (`'ifr_branch_id'` in grid 74) does NOT error —
  it stays a literal, the filter matches nothing, and the list is silently EMPTY. Send
  `""` for "all". Audit quoted tokens too, not only the `_is_deleted` ones. Audit: regexp
  `_is_deleted\s*=\s*([a-z_]+)` over `fixed.grid_details.grid_sql` vs the registry,
  then grep (`-a`) every caller of the key — non-master callers (e.g. the loyalty
  item picker on grid 67) may send no `grid_param` at all and fail silently.

**Why:** Grid Master's own SQL ends `WHERE grid_device_type = 'Desktop'`, so the app
only ever showed Desktop grids — every screen still on a legacy `web` grid was reading
a grid the user could not see or configure. Half the ids had also drifted (grid 16 no
longer exists at all).

**How to apply:** a Desktop grid whose SQL binds `:name` (Qt style) cannot run — the
runner binds `'name'`/bare `name` only, so grids 45 (MAIN LIST - ITEM UNIT) and 50
(MAIN LIST - ITEM SECTION) 400 with `syntax error at or near ":"` and Unit/Item Section
stay on their web grids until that SQL is fixed. Still on web grids by necessity:
widget-master (Desktop 47 is per-menu), ledger ship-to (110 needs a ledger id), loyalty
schemes (97 has no company filter + a mandatory state token), gsp-company-service (no
Desktop grid exists), opening-stock's unit/godown feeds (POPUP grids drop columns the
screen needs). Related: [[erp-ui-table-id-registry]], [[erp-grid-columns-config]],
[[erp-crud-master-grid-column-heuristics]], [[erp-grep-skips-binary-files]].
