---
name: item-masters-notes-139
description: "Notes 139 (item masters live-test findings) built 2026-10-22 — master names unique among LIVE rows ignoring case (partial lower() indexes, DB-only), unit index exact-case only (box has Box+BOX), qty-price overlap/unit/figure rules, godown branch rules; C1/C2 left as questions"
metadata:
  node_type: memory
  type: project
  originSessionId: f661e9b6-8d8d-46f2-8b49-136f936a1118
  modified: 2026-10-22T11:24:59.113Z
---

Built 2026-10-22 across two sessions (the first, 56dfa405, stopped before testing; the second
finished specs, proof and READMEs). Migration `20261022110000_item_masters_notes_139` (applied 14:03):
uq_brand_name / uq_itg_name / uq_sec_name / uq_category_name on `lower(name) WHERE NOT deleted`,
uq_gdl_unique_name_per_parent `(branch, parent, lower(name)) NULLS NOT DISTINCT WHERE NOT deleted`,
uq_unit_name exact-case partial; re-levelled godowns from 1; blanked '' alias/short/description to
NULL. None of these indexes is declared in the Prisma fragments (partial/expression). Proof:
`test/item-masters-notes-139.e2e-spec.ts` (10 cases, rolled back).

Non-obvious choices: unit names are case-insensitive only in `UnitsMasterService.assertNameFree`
(advisory lock) because the box holds live "Box" and "BOX" — swap uq_unit_name to lower() once
merged. Every master's restore now catches the unique error → `*_NAME_TAKEN`. Godown B3 refuses a
branch change while it has live children OR stock on hand / is a branch default (extra
`GODOWN_BRANCH_CHANGE_IN_USE`). Qty price overlap = `[from,to)` × inclusive dates, inactive slabs
count, judged after the whole batch under a per-item advisory lock.

Live rows the new unit rules now refuse on any edit (left for the user): PCS, tesrs, xd are pack
units with no base; "packs" has 20 decimals. C1 (godown code unique per branch? the stock CSV import
matches name OR code) and C2 (ZERO_RATED above 0% = export with IGST paid?) await the user. C3:
grid 98's six track-flag column rows are soft-deleted on the box and in the seed — the Qt
`grids/stock_track_preset_grid.sql` is stale, not the server.

**Why:** notes 139 live test (Qt `tests/itemmasters/api_*.py`).
**How to apply:** a new master name rule goes in the partial lower() index + handleWriteError on
create, update AND restore. Related: [[item-master-notes-137]], [[item-base-unit-lock-notes-138]],
[[prisma-migration-gotchas]], [[error-codes-notes-119]].
