# Notes 139 — Item masters: full live test, server findings (2026-10-22)

Tested on 192.168.0.106: every item master's routes (create / read / update /
clear / refusals / delete / restore / every field read back) and every Qt
screen, with throwaway `ZT-` data that was deleted afterwards (0 left).
Repeatable: `tests/itemmasters/api_*.py` in the Qt repo (each prints PASS/FAIL).

Everything not listed here works as the code claims. That covers levels and
cycles, in-use and child guards, restore, the group tracking policy, the tax
rate arithmetic, ledger lines and supersede loops, and the tracking presets.

The Qt side of what was found is already fixed.

---

## A. Fix first

### A1. A blank name is saved (brand, group, section, category)
`POST /item-brands/create {"brand_name":"   "}` → 201, stored as `''`.
The same happens on `/item-groups/create`, `/item-sections/create` and
`/item-categories/create`. `@IsNotEmpty` runs before any trim. Godown already
refuses it with 400 `GODOWN_NAME_REQUIRED`.

**Ask:** trim the name first, then refuse an empty one.

### A2. A deleted record's name can never be used again (brand, group, category, godown)
Delete "X", then create "X" → 409 `…_NAME_TAKEN`. The user is told the name
exists but can't see it anywhere. These unique indexes include deleted rows:
- `item_brand_master_brand_name_key`
- `uq_itg_name`
- `uq_category_name`
- `uq_gdl_unique_name_per_parent`

`uq_sec_name` (section) is already `WHERE NOT deleted`, and reuse works there.

**Ask:** make the other four partial (`WHERE NOT …_is_deleted`) like section's.
The blank-name rows from A1 are also stuck behind these.

### A3. 500 instead of 400
- `POST /item-groups/create` with an unknown `itg_track_preset_id` → 500
  `INTERNAL_ERROR`.
- `POST /units/create` with `unit_code:"ZZZ"` (no such GST unit) → 500.

**Ask:** check that the referenced row exists, and refuse with a 400 that names
the field.

### A4. Qty-wise price rows accept bad data
- **A unit row of another item:** `iqp_item_unit_id` pointing at another item's
  unit is accepted (201). Only the foreign key is checked.
- **Overlapping qty ranges** in the same scope (1–10 and 5–15, same dates): both
  201. Only exact duplicates are refused.
- **Mode F (fixed price) with no price:** 201.
- **A 150 % discount:** 201.

Note that nothing at billing reads `item_qty_price` yet. These need settling
before it does.

## B. Data rules

- **B1. Godown: unknown branch accepted.** `gdl_branch_id` = a non-existent id → 201.
  There's no foreign key and no existence check.
- **B2. Godown: two top-level godowns with the same name in one branch.** Both 201.
  NULL parents never clash in `uq_gdl_unique_name_per_parent`.
  **Ask:** use `NULLS NOT DISTINCT`, or `COALESCE` on the parent.
- **B3. Godown: a parent moved to another branch leaves its children behind.**
  `{gdl_id: WH, gdl_branch_id: HO}` → 200, and WH's child stays in Coimbatore
  under a Head Office parent. `validateParentAssignment` (~l.315) only checks
  the godown's own parent.
  **Ask:** refuse the move while it has children, or move the children with it.
- **B4. Godown levels are inconsistent.** A top-level godown created by the
  company-create seed path has `gdl_level` 0; normal creates give 1.
- **B5. Units accept values the screen doesn't allow.**
  - `unit_decimal_count: 9` → 201 (the screen allows 0–3).
  - A negative `unit_weight` → 201.
  - `unit_is_pack_unit: true` with no base unit → 201.
- **B6. Case-only duplicate names are accepted** on brand, group, section,
  category and unit. "ZT-…" and "zt-…" are both 201, and the live box already has
  both `Box` and `BOX`.
  **Ask:** use case-insensitive unique indexes (on `lower(name)`), if that's the
  rule wanted.
- **B7. Brand: a cleared optional text is stored as `""`, not NULL.**
  `{"brand_short":""}` → `''`. Godown's DTO already turns `""` into NULL;
  Brand's doesn't.
- **B8. Tax rate: a new rate can supersede a deleted rate.** → 201.
  `assertSupersedesExists` (~l.724) doesn't filter `taxIsDeleted`. The Qt
  dropdown never offers a deleted rate, so only direct API calls are exposed.
- **B9. Tax rate: the documented error codes never fire for negatives.**
  A rate of −1 or a negative cess is refused, but with the generic
  `VALIDATION_MIN`. `TAX_RATE_PERC_OUT_OF_RANGE` and `TAX_RATE_CESS_NEGATIVE`
  can't be reached.

## C. Questions (not treated as bugs)

- **C1. Godown codes:** the same `gdl_code` can be used twice in one branch, and
  there's no unique index. Should a code be unique per branch?
- **C2. Zero-rated rates:** a `ZERO_RATED` rate at 5 % is accepted. The README
  says a zero-rated export is taxable at 0 %. Is a non-zero rate allowed (an
  export with IGST paid)?
- **C3. Grid 98 (tracking presets):** the stored `grid_sql` has none of the six
  track-flag columns that `grids/stock_track_preset_grid.sql` says it carries.
  Nothing reads them from this grid today.
