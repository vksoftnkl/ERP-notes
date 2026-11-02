---
name: configured-grid-param-convention
description: "How configured-grid-sql binds grid_param into stored SQL — named p_* placeholder tokens, not output columns"
metadata: 
  node_type: memory
  type: project
  originSessionId: fa29c825-0d96-459c-820e-589a70cc1813
  modified: 2026-08-08T08:26:25.683Z
---

`configured-grid-sql/run?grid_id=&grid_param=<json>` binds grid_param into the stored
`fixed.grid_details.grid_sql` (Prisma model `GridDetails`, field `gridSql`). The stored SQL embeds
parameters as **bare named identifier tokens** (e.g. `p_comp_id`, `p_branch_id`, `p_status`,
`p_type` — see the grid 40 `loyalty_sch_list` seed), occasionally quoted (`'p_comp_id'`). The
grid_param JSON keys match those token names; the `p_` prefix keeps them distinct from real output
columns like `ls_comp_id`. They are NOT `:named`/`$1` placeholders and NOT output-column names.

Two prefixes are in use, both bound the same way: `p_*` (newer) and the legacy VFP input params
`i*` — `icompany_id`, `ibranch_id`, `iacc_year`, `ifrom_date`, `ito_date` (grids 83/84/86, the
quotation and bill lists). An unbound token is only harmless where it sits in a plain comparison; a
token behind a cast (`sq_company_id = 'icompany_id'::uuid`) makes Postgres reject the literal, so
every caller of such a grid MUST send all of its keys — dates as `""`, which the SQL's
`NULLIF('ifrom_date','') IS NULL OR …` guards read as "no bound". A missing key is now reported as a
400 naming it (`findUnboundParamTokens`), not a 500.

Bind with `ConfiguredGridSqlService.bindGridParams(sql, prm)` — it replaces each token with a
positional `$N` and returns a params array (secure, parameterized). `substituteGridPrm` is the
legacy INSECURE string-substitution version (formats SQL literals) still used by
`dropdown-details.service.ts`; a candidate to migrate to bindGridParams. Do NOT route grid_param
through `buildFilterSql` (output-column `WHERE "key" = $N`) — it leaves the embedded p_* tokens
unbound and errors with "column p_* does not exist".
