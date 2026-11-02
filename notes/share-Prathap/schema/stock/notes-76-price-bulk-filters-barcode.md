# Notes 76 — Change Selling Price: more filters and a barcode column (2026-10-01)

**The user's ask:** "move the filter into a popup … filter by the item entry's basic info, all
optional, only active items, include tax … add barcode as a column before item."

`GET /stock/price-bulk` filters today on `itemGroupId`, `itemBrandId`, `itemSectionId`,
`supplierId` and `itemId` only, and its row carries no barcode. The Qt screen now has a filter
popup that sends the keys below. Until they exist, the DTO's forbidNonWhitelisted turns any of
them into a 400, so the screen says "needs the server update (notes 76)" instead of loading.
The four existing filters keep working unchanged.

## A. New optional query keys on `GET /stock/price-bulk`

All are optional and ANDed with the existing ones, inside `gridStatement()`'s `items` CTE (the
`itemFilter` fragment).

| Key | Type | Predicate |
|---|---|---|
| `search` | string, trimmed, ≥ 1 char | `i.item_code ILIKE %s% OR i.item_name_en ILIKE %s% OR i.item_alias ILIKE %s% OR i.item_default_barcode ILIKE %s%`. Escape `%` / `_` in the input. The user said "contains". |
| `itemCategoryId` | uuid | `i.item_category_id = $` |
| `trackPresetId` | uuid | the **effective** preset: `COALESCE(i.item_track_preset_id, grp.itg_track_preset_id) = $`. An item follows its group unless it has its own preset (notes 67/68), so "Tracked as" must match what the item entry shows. |
| `taxId` | uuid | `i.item_default_tax_id = $` (dropdown 36, the item entry's tax) |
| `activeOnly` | boolean, **default true** | `i.item_is_active = true` when true. Today an inactive item is listed, because only `item_is_deleted` is checked. |

`itemId` (notes 74) stays as it is.

## B. `barcode` on every row

Add `barcode` = `i.item_default_barcode` to the grid row (`gridStatement`) and the F12 row
(`listBuckets`), on both UNION branches. The screen shows it as the column before Item. It is
display only and never sent back on save.

## C. Optional: `costWot` on the grid / F12 row

The screen's new "Cost B.Tax" column is derived as cost ÷ (1 + tax%), which is wrong for an item
with cess. `loadRowCosts()` already computes the true `costWot`, so returning it on the row would
let the screen show it exactly. The screen uses it when present.

## Test I will run when you say it is in

- search on a code fragment, a name fragment and a barcode fragment
- each id filter alone, and two together
- `activeOnly` default (an inactive ZT item hidden) and `activeOnly=false` (shown)
- `barcode` on the rows, with ZT-ITEM-A's default barcode

---

## Verified 2026-10-01 — applied and live

- **search:** a barcode fragment (784529), a name fragment (iscui), any case, padded spaces and a code (ZT-ITEM) all find the item. `100%` / `a_b` match nothing, so the escaping holds.
- **Filters:** `itemCategoryId`, `taxId` and `trackPresetId` each filter correctly, including the effective preset (ZT-ITEM-B / SUGAR / ZZ TEST N match through their own preset). The filters combine.
- **activeOnly:** by default, the inactive Oreo biscuit and corn flavour are hidden; `activeOnly=false` shows them. A bad uuid or bool gives a clean 400.
- **Row fields:** `barcode` and `costWot` are on the grid and F12 rows.
- **Client:** the Qt screen drives all of this live (suite C9).
- **costWot exposed a separate engine defect,** notes 77. Sale and DC returns post wot = the inclusive rate, so ZT-ITEM-A reads 7.90. The screen keeps deriving Cost B.Tax except for cess items until notes 77 is fixed.
