---
name: grid-sql-no-comments
description: "fixed.grid_details.grid_sql must never hold \"--\" or \"/*\" — the runner 400s GRID_RUN_SQL_INVALID and the grid lists nothing"
metadata:
  node_type: memory
  type: project
  originSessionId: 92e1f481-9103-4bba-aa10-0af8f69e7ddb
  modified: 2026-10-17T05:15:29.352Z
---

The configured-grid runner (`GRID_SQL_COMMENT_PATTERN` in
src/common/configured-grid-sql/configured-grid-sql.service.ts) refuses any grid SQL holding `--` or
`/*`, even inside a string literal: 400 GRID_RUN_SQL_INVALID "Comments are not allowed in configured
query". The grid-details save path strips comments first, but a migration that writes grid_sql does not.

**Why:** migration 20261016160000 (notes 130) put four `-- notes 130 …` lines into grid 119's SQL, and
the Voucher Register ledger cell showed nothing on every type until notes 132 (2026-10-17, migration
20261017100000) removed them.

**How to apply:** keep explanations in the migration's header comment, never inside the
`$mig$…$mig$` replacement text, and edit `prisma/seed/Grid_Details.sql` to match. Before applying, run
`grep -nE -- '--|/\*'` over the new grid SQL. See [[configured-grid-param-convention]].
