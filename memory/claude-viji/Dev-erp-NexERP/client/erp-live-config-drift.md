---
name: erp-live-config-drift
description: "Designer-made config rows (fixed.dropdown_details, grid_details, ui_tables) exist only in the dev DB — live (169.58.213.171) drifts and new screens show \"No matches\"; how to diff and sync"
metadata:
  node_type: memory
  type: project
  originSessionId: e4275984-6044-4b69-9a39-7187ba20cfc4
  modified: 2026-10-05T11:09:46.754Z
---

Found 2026-10-03: the Payment screen's Payee picker showed "No matches" on live
because `fixed.dropdown_details` there had no **PAYMENT PAYEES** row (live ids
stopped at 49; dev had 49–62). No migration ships these rows — screens are built
against dropdowns/grids/ui_tables authored through the designer screens in the
dev DB, and nothing carries them to live. Same commit on both sides, so code
parity proves nothing; the config tables are the thing to diff.

Live DB access: `ssh root@169.58.213.171 'cd /tmp && sudo -u postgres psql erp -Atc "…"'`
(db is lowercase `erp`; the `cd /tmp` avoids a permission moan). Frontend is
`/opt/erp-front/app` (branch dev, use `git -c safe.directory=* -C …`), API is
`/opt/erp-server/app` (branch DEV-v1). See [[erp-frontend-vps-deploy]].

**Why:** the registries resolve ids BY NAME from the master lists
([[erp-configured-dropdowns]], [[erp-configured-grid-registry]],
[[erp-ui-table-id-registry]]); a name missing on live falls back to the dev id,
which does not exist there, so the fetch returns nothing and the screen looks
empty with no error anywhere.

**How to apply:**
- Diff by name, not id: dump `id|name|device_type` from both DBs and `diff`.
  Hash SQL + columns per name to find rows that exist on both but differ
  (`md5(row(…)::text)`, `string_agg` over `dropdown_columns`). Ids differ for
  the same name (CUSTOMERS BY AREA was 49 live / 54 dev).
- Live has been edited directly too (TAXES got `ORDER BY tax_sort_order` on
  live only, 2026-09-12) — check `modified_on` on both sides before overwriting.
- Generate INSERTs from dev with `format('INSERT … VALUES (%L, …)')` (`%L` for
  every column; quoted literals coerce to int/bool fine), renumber live
  collisions first (`dropdown_columns` FK is ON UPDATE CASCADE), then `setval`.
- The auto-mode classifier refuses to run writes against the live DB and to
  mint an auth session there: generate the script, back up the tables
  (`pg_dump -t fixed.dropdown_details -t fixed.dropdown_columns`), and hand the
  user the commands to run.
- Rows can exist on BOTH sides with identical SQL and still break search: live grid
  107 (POPUP - LEDGERS, Opening Balance picker) had every `grid_column_filter=false`
  and Title_Case `sql_field_name`s (`Led_Name`). The runner matches search columns
  by EXACT key against `row_to_json` (lower-case), so BOTH must be fixed. Diff
  `grid_column_filter` + `sql_field_name` per grid name too (concat'd booleans print
  `true`/`false`, not `t`/`f`). Restore script: `/root/grid107-columns-restore-20261005.sql`.
  2026-10-05: handed the user the UPDATE (classifier blocked it).
- Also still missing on live as of 2026-10-03: ~30 grids (100–106, 108–112,
  116, 119, 125–131), ~20 ui_tables (27–44 incl. PAYMENT - BILLS/TENDERS 42/43),
  and the `20260923120000_offline_cloud_sync` migration is unapplied there.
- 2026-10-15: reusable generator pattern in `docs/live-sync/2026-10-15-till-grids.sql` (client repo): temp
  tables of `%L` VALUES + a DO-block upsert by name+device, dev id kept when free, sequence `setval` before and
  after, guard on prerequisite columns. Test on a throwaway DB (`pg_dump -t` the 4 fixed tables, `create schema
  fixed` first) — dev grids have DUPLICATE grid_column_number, so compare with a full-row sort.
