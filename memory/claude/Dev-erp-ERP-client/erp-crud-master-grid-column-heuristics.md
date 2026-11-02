---
name: erp-crud-master-grid-column-heuristics
description: CrudMasterPage grid-column accessor heuristics that bite masters with no natural name field or multiple FK-name lookups (found building freight/loading charges masters)
metadata: 
  node_type: memory
  type: feedback
  originSessionId: fd5477f2-46e6-4595-bfe9-273ead651af4
  modified: 2026-07-22T13:00:57.736Z
---

`components/master/crud-master-page.tsx`'s `buildColumnsFromGridColumns` resolves each configured grid column (from `fixed.grid_columns`, driven by `gridDetailId`) to an internal accessor (`masterName`, `masterCode`, `masterActive`, ...) via `resolveMasterAccessorFromGridColumn`, which matches on `normalized.endsWith("name"/"code"/"alias"/...)` as a **substring** check, not a semantic one — a raw SQL alias like `br_name` or `comp_name` matches `endsWith("name")` just as much as a real name column would.

Two gotchas this causes for masters with no natural single "name" field (e.g. `sale_freight_charges`, `sale_loading_charges`, which key on ranges + a charge amount, not a name):

1. **First `_name`-suffixed column in grid-column position order wins the `masterName` accessor slot**, and its cell renders from `row.masterName` (populated by `buildMasterRows` via `lookupKeys.name`) — not from its own SQL field directly. Any other `_name`-suffixed columns still render correctly via a fallback pass-through (reads their own `sqlFieldName` from `row.__source`), so nothing breaks, but the FIRST one goes blank unless `lookupKeys.name` explicitly includes its raw SQL alias, ordered first. This order is grid-specific: freight's grid (`grid_id=74`) lists Branch before Company, so `lookupKeys.name` needs `["br_name", "comp_name", ...]`; loading's grid (`grid_id=76`) lists Company before Branch, so it needs the opposite order. Check `grid_column_position` for the specific grid before assuming an order.
2. **The Active column needs its literal domain SQL alias in `lookupKeys.active`** (e.g. `fr_is_active`, `ilc_is_active`) — the column heuristic itself resolves the "Active" grid column to the `masterActive` accessor via a generic `compact.includes("active")` fallback regardless of `lookupKeys`, but the actual cell VALUE comes from a separate lookup (`buildMasterRows` reading `lookupKeys.active ?? DEFAULT_ACTIVE_KEYS` against `row.__source`), and `DEFAULT_ACTIVE_KEYS` only covers generic names (`active`, `is_active`, `isActive`), not domain-prefixed ones. Omitting the domain key renders the Active column blank even though the column itself appears.

**Why this matters:** both were invisible in code review / TypeScript — they only surface as blank cells in the actual rendered grid. Verified by direct psql inspection of `fixed.grid_columns` (`grid_column_position`) plus a live headless-browser render, not by reading the component alone.

**How to apply:** before wiring `lookupKeys` for any master with `gridDetailId` set, query `fixed.grid_columns` for that grid's columns ordered by `grid_column_position`, and explicitly list every `*_name`/`*_is_active`-style raw SQL alias for that grid — don't rely on `DEFAULT_ACTIVE_KEYS` or column ends-with pattern coincidences.

Related, lower-stakes findings from the same session:
- `gridDetailId` can be passed as a literal number prop (e.g. `gridDetailId={74}`) to bypass editing `features/masters/shared/configured-grid-detail-ids.ts` — [[erp-grid-columns-config]] didn't mention this shortcut; `godown/page.tsx` already used it.
- The "Show deleted records" `wantDelete` checkbox + `grid_param: {wantdelete}` boilerplate (copy-pasted in most master pages) only works if the specific grid's stored SQL has a `wantdelete` bound token in its WHERE clause (e.g. grid 2/state: `WHERE stm_is_deleted = wantdelete`). Some grids hardcode `WHERE x_is_deleted = false` with no token (freight/loading's grids 74/76) — including the checkbox there would be inert dead UI. Check `fixed.grid_details.grid_sql` for the token before adding it.
- (Obsolete since 2026-10-26) nav hrefs used to bind by exact menu label; they now bind by menu_id in `MENU_ROUTES` — see [[erp-menu-href-mapping]].
