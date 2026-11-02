---
name: erp-server-500-debugging
description: "How to debug opaque 500s from the ERP server, plus schema-drift gotchas (unapplied migrations, stale Prisma connections, leftover DB check/enum constraints)"
metadata: 
  node_type: memory
  type: reference
  originSessionId: 26047153-64ea-43e1-be9c-6f0da8504cf1
---

ERP server module exception filters (`ModuleExceptionFilter` in `src/common/utils/module-shared.utils.ts`) return a generic `{"success":false,"message":"Internal server error","errors":[]}` for any non-`HttpException` and **do not log the underlying error** — so DB/Prisma failures show nothing in `logs/app.log` (only `RequestLoggerMiddleware ... 500`). To surface the real cause, temporarily rethrow it in the service `catch` (e.g. `throwAccountsBadRequest('DEBUG', [{field:'debug', message: \`${(error as Error)?.name}: ${(error as Error)?.message}\`}])`) and read it from the HTTP response, then revert.

Server runs the **compiled `dist/`** under `nest start --watch`. A pure `touch` (mtime) does NOT trigger a rebuild — make a real content change. After an `ALTER TYPE`/column-type migration, the live server keeps **stale Prisma pooled connections with cached plans** and still 500s until it rebuilds/reconnects (`column "x" is of type ... but expression is of type text`, or "cached plan must not change result type").

Configured-grid SQL is another 500 source: the grid designer (`features/masters/settings/grid-designer/[id]`) free-texts `grid_sql` and per-column `sql_field_name` into `fixed.grid_details`/`grid_columns`, and save-time validation is syntax-only **by design** (no LIMIT 0 probe — `GridDetailsService.normalizeGridSql`), so a typo'd column (e.g. `itg_position`; the real "position" concept is the `*_sort` column, `itg_sort`) breaks the master page list at run time. Since 2026-07-04, `configured-grid-sql/run` maps pg SQLSTATE class-42 errors to a 400 naming the grid and DB message instead of an opaque 500; fix = correct the grid row in `fixed.grid_details`/`grid_columns` (verify the SELECT directly in psql first).

Schema-drift class of bugs (caused "cannot save/update account ledger"): (1) a pending migration simply wasn't applied — check `npx prisma migrate status` / `_prisma_migrations`; apply with `prisma migrate deploy`. (2) This codebase is moving allowed-value validation from DB (native enums + CHECK constraints) to the **app layer** (`@IsEnum(...)` in DTOs); leftover DB constraints can contradict the app enum values (e.g. `chk_lba_account_type` allowed Title-Case 'Savings' but the app sends UPPERCASE 'SAVINGS'). Fix = new migration dropping the stale constraint, mirroring `*_remove_*_check_constraints_to_app_layer` / `*_move_*_enums_to_app_layer`. Related: [[erp-auth-endpoint-verification]], [[erp-workspace-layout]].
