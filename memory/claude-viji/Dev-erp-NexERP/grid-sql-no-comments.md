---
name: grid-sql-no-comments
description: "fixed.grid_details.grid_sql must never hold \"--\" or \"/*\" — the runner 400s GRID_RUN_SQL_INVALID, the grid lists nothing, and a psql test of the SQL hides it"
metadata:
  type: reference
---
The configured-grid runner (`GRID_SQL_COMMENT_PATTERN` in `ERP server/src/common/configured-grid-sql/configured-grid-sql.service.ts`) refuses any grid SQL holding `--` or `/*`, even inside a string literal. `/configured-grid-sql/run` answers 400 `GRID_RUN_SQL_INVALID`, "Comments are not allowed in configured query".
- The Grid Master save path strips comments first, but a migration that writes `grid_sql` does not.
- On screen a popup just shows "Records : 0", with no error.

**Why:** migration 20261016160000 (notes 130) put four `-- notes 130 …` lines into grid 119's SQL. The Voucher Register ledger cell then showed nothing on every voucher type until notes 132 (2026-10-17, migration 20261017100000) removed them ([[tender-on-every-money-line]]).

**How to apply:**
- Keep explanations in the migration's header comment, never inside the `$grid$…$grid$` / `$mig$…$mig$` replacement text.
- Edit `prisma/seed/Grid_Details.sql` to match.
- Grid SQL sent to Prathap has NO comments: keep them in the share .sql header and strip them for the DB copy.
- Before applying, run `grep -nE -- '--|/\*'` over the new grid SQL. An em dash '—' inside a literal is fine.
- Check a grid through the runner (Qt harness `apiGet(NexGridApi::run(id,…,gridParam))`, or curl), not only psql. See server/[[configured-grid-param-convention]].
