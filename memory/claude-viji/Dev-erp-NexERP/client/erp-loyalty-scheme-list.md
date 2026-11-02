---
name: erp-loyalty-scheme-list
description: Loyalty scheme list (promotion-loyalty-points page) data source — configured-grid-sql grid 40
metadata: 
  node_type: memory
  type: project
  originSessionId: f0e4ef84-196d-4387-ace6-145a47300a81
---

The promotion-loyalty-points page lists schemes via `configured-grid-sql/run?grid_id=40` (constant `LOYALTY_LIST_GRID_ID = 40`). It sends `grid_param` JSON `{p_comp_id, p_branch_id, p_status, p_type}` (empty string = no filter) plus `search`/`page`/`limit`, and reads `response.data.items` + `response.data.meta`. Rows are flat `SchemeListRow` (counts come back as strings → coerced); the "Rules" column uses `points_count`/`gifts_count`/`parties_count` (no nested arrays).

Grid 40 is provisioned by ERP server migration `prisma/migrations/20260612071500_seed_loyalty_scheme_grid` (idempotent; SQL over `sales.loyalty_sch_list` with subquery counts + `grid_param` filter placeholders, plus 5 `grid_columns` rows; `ls_name`/`ls_code` are filterable for search). The client hardcodes the id 40, so the migration must run in every environment.

**Why:** server commit `e5db192` removed all per-module list endpoints (incl. `/promotion-loyalty-points/list`) in favor of `configured-grid-sql/run?grid_id={id}`, but the loyalty list was never given a grid — the client was misrouted to `/get` and 400'd. Related: [[erp-grid-columns-config]], [[erp-auth-endpoint-verification]].
