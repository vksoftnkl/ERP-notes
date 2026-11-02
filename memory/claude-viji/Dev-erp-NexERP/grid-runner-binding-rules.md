---
name: grid-runner-binding-rules
description: How configured-grid-sql binds grid_param tokens and search — every optional token must always be sent, never cast a token, search is column-flag driven; learnt registering grids 113/114/115 on 2026-09-23
metadata:
  type: project
---

`configured-grid-sql.service.ts` (nexerp-api-ro, read only) runs a grid's SQL by
**binding each `grid_param` key as a positional `$N` in EVERY occurrence of the
token, quoted (`'icompany_id'`) or bare (`isaa_ledger_id`)**. Three consequences,
each found the hard way on 2026-09-23:

1. **An unsent token is left as literal text.** `'idelivery_status'` with no such
   key stays the string `idelivery_status`, so `NULLIF('token','') IS NULL` is
   FALSE and the filter silently applies the word itself. The client must send
   every optional token, '' when unset — `TxnFilter::sendWhenEmpty = true`,
   `buildExtraParams` always populated. Dates are always sent by TxnMainView.
2. **Never cast a token.** `'ibranch_id'::uuid` / `'ifrom_date'::date` are folded
   at plan time; with '' or the literal token they raise
   "invalid input syntax for type uuid" before any CASE can skip them. Compare as
   text: `t.atc_branch_id::text = 'ibranch_id'`, `b.sb_bill_date::text >= 'ifrom_date'`
   (ISO dates order correctly as text).
3. **Don't try to detect "unset" by comparing the token to its own name** —
   `'x' IN ('', 'x')` becomes `$1 IN ('', $1)`: always true.

**`search` is not a token.** The runner wraps the base SQL and adds ILIKE over the
columns whose `grid_column_filter` is true (`deriveSearchableFieldNames`). Never
write `'search'` in a grid's SQL; flag the columns instead. Right after flagging
columns on 114/115 the search still answered 0 — suspected preloaded-column cache
(`service.ts:185`), asked in notes (40).

Registration: `POST /grid-details/create` WITHOUT `grid_id` creates (the server
picks the next id — the plan's 131/133/134 became **113/114/115**); WITH
`grid_id` + `replace_columns: true` updates. Same for `/dropdown-details/create`
(55/56 landed as planned) and `/ui-table-masters/create` with
`replaceColumns: false` to APPEND columns (ui_table 22 cols 94–98; position =
column no + 1 on that table). A run: `GET /configured-grid-sql/run?grid_id=&limit=&
grid_param=<JSON>&search=`.

Related: [[track-preset-list-screen]] (the first API registration),
[[nextable-grid-traps]].


**Two more traps (2026-09-26, grid 119 / dropdown 59).**
- **Top-level FROM on a real table.** Both grids and dropdowns must start `SELECT … FROM schema.table`: no top-level `WITH`, and no `FROM (subquery)`. Put recursive CTEs inside WHERE/select-list subqueries. `/grid-details/create` refuses the SQL at create time; `/dropdown-details/create` ACCEPTS it and only `/run` fails, with "Invalid dropdown SQL configuration".
- **No comments in dropdown SQL.** The grid endpoint strips `--` comments on save; the dropdown endpoint stores them raw. An apostrophe inside a comment then reads as a quote, and `/run` says "Multiple statements are not allowed".
- Run a dropdown with `GET /dropdown-details/run?dropdown_id=&page=&limit=&search=&dropdown_param=<JSON>`. Grid rows come back under `data.items`, and the limit is 100 at most.

**Stale-read trap (found 2026-09-26):** the global HttpCacheInterceptor caches `/configured-grid-sql/run` by URL for 1 s, because it reads @CacheTTL(1) as seconds. A reload of the same page within a second of the last load returns the OLD rows. After a write, reload no sooner than 1.1 s after the last load (IssuedCheques::m_lastLoad). The server ask is notes 55 #3. See [[issued-cheques-screen]].

**Updating a registered grid (2026-09-28 trap):** re-POSTing `/grid-details/create` with `grid_id` replaces the SQL but APPENDS `grid_columns` sent without their `grid_column_id` — grid 119 ended up with 18 columns (9 duplicates created 2026-09-28). Send each column's existing `grid_column_id` (from `/grid-details/get`), or omit columns if only the SQL changes. Removing extras is `DELETE /grid-details/column-delete?grid_column_id=` (needs the user's go-ahead; auto mode blocked it).
