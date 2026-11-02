---
name: erp-loose-search
description: "Search normalisation (2026-10-02) — fixed.fn_search_norm + token-AND matching in every configured grid/dropdown and 8 raw searches; status, remaining paths (Prisma contains routes, pg_trgm fuzzy) in ERP server plan/loose-search.md"
metadata:
  node_type: memory
  type: project
  originSessionId: e0290841-c622-4b94-bead-23b4bafdf3d4
  modified: 2026-10-02T06:20:29.923Z
---

User's complaint 2026-10-02: "chillipowder" found nothing (stored "CHILLI POWDER"); wanted the
backend to make any typed form match, across the whole application.

Built (uncommitted at the time of writing; dev DB migrated):
- Migration `20261002130000_search_norm_function`: `fixed.fn_search_norm(text)` = lower + strip
  `[[:space:][:punct:]]`. IMMUTABLE. Applied to dev via `prisma migrate deploy`.
- `src/common/search/loose-search.ts`: split typed text on spaces; each piece must match some column,
  both sides through the SQL function (never normalise the term in TS — glibc punct ≠ JS \p{P}).
- `ConfiguredGridSqlService.buildSearchSql` rewritten → covers ALL configured grids + dropdowns
  (`/configured-grid-sql/run`, `/dropdown-details/run`) in Qt and React. Params are now
  `[string[] columns, ...tokens]` (specs updated).
- 8 `$queryRaw` ILIKE spots converted with `looseSearchSql([...cols], search)`.
- `test/loose-search-http.e2e-spec.ts` (credential-free, read-only) proves it over HTTP.
- React: `lib/search-text.ts` used by dynamic-modal-form / searchable-select / -multi-select.

**Why / how to apply:**
- `~100 Prisma contains/insensitive` list routes still search strictly — Prisma can't call the SQL
  fn; plan R1 = ids via $queryRaw then findMany. Don't claim those are fixed.
- Typos are NOT handled (no pg_trgm yet); R2 in the plan. pg_trgm is available & trusted on dev PG 18.
- Grid 1 "item master" binds `wantdelete` with no grid_param → 400; use grid 67 "MAIN LIST - ITEMS"
  with `grid_param {"iitem_is_deleted": false}` in tests. Dropdown 42 ITEMS is param-free.
- The configured-grid run route is `/configured-grid-sql/run` (NOT `/grid-details/run`).
- Qt `NexGridProxy::stripSpecialChars` already strips spaces locally — no Qt change needed.
Related: [[erp-server-http-testing-without-credentials]], [[nexerp-qt-client-source]].
