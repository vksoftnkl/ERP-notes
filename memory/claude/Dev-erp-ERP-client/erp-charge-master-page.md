---
name: erp-charge-master-page
description: Charge Master client page (2026-07-24) — grid 81 list + /charges CRUD; grid 81 SQL is missing the chg_is_deleted filter
metadata: 
  node_type: memory
  type: project
  originSessionId: 7142496d-4df2-49ac-9191-ee59b0b5493f
  modified: 2026-07-24T09:41:48.822Z
---

Charge Master page built 2026-07-24: `features/masters/accounts/charge/page.tsx`, route
`app/master/charge-master/`, nav under **Accounts**. Backend (`src/modules/master/charge-master`,
`/charges/create|get|delete`) and grids 80/81 pre-existed — no server change was needed.
List uses `configured-grid-sql/run?grid_id=81`; there is **no** `/charges/list` endpoint.

**Open defect (DB config, not client code):** grid 81's `grid_sql` has no
`WHERE c.chg_is_deleted = false`, so soft-deleted charges stay in the list as "Inactive".
Fix = add that predicate to `fixed.grid_details.grid_sql` for grid 81 (or via Grid Designer).

`chgRole` and `chgCostAlloc` must go out as **null**, never `""` — the server DTO's
`SkipOnNullish` only bypasses `@IsIn` for null. `chgCostAlloc` is `visibleWhen`/`requiredWhen`
landing-cost is checked.

Related: [[erp-crud-master-grid-column-heuristics]], [[erp-grid-columns-config]],
[[erp-client-ui-browser-verification]] (note: the cached playwright chromium is gone —
`/snap/bin/chromium` works as `executablePath`).
