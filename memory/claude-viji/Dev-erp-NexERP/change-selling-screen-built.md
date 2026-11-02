---
name: change-selling-screen-built
description: Menu 30 Change Selling Price coded + live-tested 2026-10-01 in stock/sellingprice; contract facts, ui_table 44, and what is untested (bucket rows, CHAIN)
metadata:
  type: project
---

**Built 2026-10-01** in `src/modules/inventory/stock/sellingprice/` (entity / delegate / dialogs / entry .ui). Registered `fm.reg<ChangeSellingPriceEntry>(30)`. Routes are in `ApiEndpoints::SellingPriceBulk`. Test: `tests/sellingprice/run.sh`, 34/34 live. It edits ZT-ITEM-A (01a0d242-5613…), saves, does a below-cost confirm, then restores 118 on every level. `CSP_SNAP=<dir>` writes snapshots.

**Contract facts (live, 2026-10-01):**
- GET /stock/price-bulk returns `data.items[]` + `meta{limit,offset,count}`, limit ≤ 1000. Numbers come back unquoted.
- `uomId` is an iuc_id. `maxPrice` is what the MRP column SHOWS; `mrp`/`salePrice` are the bucket dimensions the save echoes back.
- `/stock/price-buckets/:itemId` lists chain AND branch rows. The grid keeps the winner per (uom, mrp, sp): BRANCH beats CHAIN.
- Save sends CHANGED rows only. `lineNo` = grid row + 1, and a 422 names `rows.<lineNo>`.
- `priceScope` is echoed unchanged and left absent when the row has no price. Price is authoritative.
- A warning-policy below-cost save → 200 `needsConfirm` → BelowCostDialog → re-post with `confirmed:true`. Notes 71 B1 (audit 400) is fixed live.
- VKPOS is SUPER ADMIN, so CHAIN is allowed (the server's HQ_USER_TYPES list is mirrored only to grey the switch).

**Not tested live:**
- No item on .106 has an MRP/SP bucket, so BUCKET chips, NEW rows (re-adding an item that is on the grid → itemLookup trackSignature M/P) and the noStock toast were not exercised.
- A CHAIN save and the 403 were not exercised either.

**ui_table 44** "CHANGE SELLING PRICE - ROWS" registered 2026-10-01 at the user's ask ("same like grid in item entry"). It has 35 columns: 0-16 visible, 17-34 hidden ids, and the Enter stops are the Mkup%/Price pairs + Min. The create DTO key is `uiTblColumns` (+ `replaceColumns`); `columns` is a 400. Menu 30 has `menu_visiblity = false` on the box.

Mockup deviations: the violet card is LIVE (four editable numbers for the current cell's level), and the row card explains the current row. The band reads "one row = one row of the item price table", not the table name. See [[pricing-one-table-analysis]], [[mockup-fidelity]].

**2026-10-01 (later): Add item showed MRP ITEM as stock 0 / MASTER.** The F12 route (`/stock/price-buckets`) lists PRICE rows only, so a stock bucket with no price row of its own was missing. Add item now uses `/items/get` (data.item.item_group_id / brand / section / supplier) → `/stock/price-bulk` narrowed by all four → keeps that item's rows. The MRP column falls back to `mrp` when `maxPrice` is 0. Live test C7 is read-only on MRP ITEM: 18 on hand at MRP 300, NEW. Notes 74 (`schema/stock`) asks for an `itemId` filter on price-bulk, and for F12 to include unpriced stock buckets.
**Notes 74 APPLIED by Prathap + verified live 2026-10-01:** `price-bulk?itemId=` (uuid, ANDed with the other filters; 400 on a bad uuid), and F12 now also lists unpriced stock buckets (MRP ITEM: headline + MRP 300 / 18, maxPrice 0, MASTER). Add item now uses `Filters.itemId` (the /items/get 4-filter workaround is removed). The F12 dialog's MRP falls back to `mrp` when `maxPrice` is 0. Live suite all pass.
**2026-10-01, user asks:**
- No border on the price cells; only the red/amber tint remains.
- **Branch column added** as enum 35 (appended), shown 2nd. ui_table 44 is now 36 columns (Branch at position 1, width 8%; Item 12, Src 6; visible total 97%). It shows brShort ("Acme CBE") for a BRANCH row and "All branches" (violet) for a CHAIN row; a NEW row follows the switch.
- **No Company column:** the grid is one company, so it would show the same value on every row. Add it with the multi-branch mode.

**2026-10-01, user asks: cost per MRP.** For an MRP-tracked bucket row, the Cost column must be the landing cost of THAT MRP's stock, not the item's blended average. Non-MRP rows keep the item average (the user said that is right). The cost is the server's `costRate`, from `stock_item_cost` (one average per item per branch), and the engine stamps it on every holding, so `sbl_stock_value` can't separate MRPs, and `slt_cost_rate` is only the FIRST inward. Written up as **notes 75** (`schema/stock/notes-75-mrp-bucket-cost.md`): a lateral over inward ledger rows per lot at that MRP, weighted by on-hand qty, net of reversals. It was verified read-only live: MRP STOCK ITEM is 25 @ MRP 40 and 500 @ MRP 550, where today both show 341.67. All three readers must change together (gridStatement, listBuckets, loadRowCosts), and the note also asks for `costBasis`. **No client change until it lands.** Then label the cell "Cost @ MRP x" from costBasis and re-run tests/sellingprice.
**2026-10-01, later: notes 75 APPLIED + verified.** The grid, F12 and the save all use the per-MRP cost (MRP STOCK ITEM 25/500, needsConfirm against 25). `costBasis` is held in hidden col 41 (CostBasis). The Cost cell gets an "MRP" tag and the row card names the basis.
**Same day, user asks: before-tax columns.** Appended enums CostWot=36 (shown after Cost) and RateA-D=37..40 (Base Rate, shown before each level's price). ColumnCount is now 42. Price columns are renamed "Net Rate" in ui_table 44, which was re-registered: 42 cols, visible widths 100%, Enter stops on Mkup/Base/Net/Min. A typed base rate sets net = applyRoundOff(base × (1+tax)), and the base is then re-shown from the net. Base rates and cost wot are derived, never sent. Suite: C3b + C8 added, all pass. C7's NEW-row checks SKIP because the MRP ITEM @300 bucket got priced (BUCKET·BR, 220) at 13:00 today.
**2026-10-01, round 3 (user asks):**
- The "MRP" tag in the Cost cell was removed: unwanted. The user's "+10/−10" means the CHANGE SINCE LOAD of a price. `drawDelta()` puts it at the LEFT edge of Net Rate AND Base Rate (base delta = base − loaded net before tax), drawn only when it fits beside the figure.
- Filters moved to the F8 popup `SellingPriceFilterDialog` (dialogs.cpp): contains, group 17, category 20, brand 18, section 19, supplier 35, tracked as 49, tax 36, plus Active only (default on). `m_filter` holds the state and lblFilterSummary shows it. The inline strip dropdowns are gone.
- Scope defaults to **All branches** for HQ types.
- **Barcode** = enum 42, shown before Item; ColumnCount 43.
- ui_table 44 was re-registered: 43 cols, Base 5% / Net 5%, total 100.
- Server side is **notes 76** (search / itemCategoryId / trackPresetId / taxId / activeOnly + a `barcode` row field + an optional costWot). Until it lands, those filters 400 and the screen names the missing key; the Barcode column stays empty.
- Suite all pass; snapshots via CSP_SNAP include 0_filter_popup.png.
**Notes 76 APPLIED + verified 2026-10-01:** search, category, tax, preset and activeOnly work, and barcode + costWot are on the rows. Suite C9 runs it live through the public `setFilter()`. **Notes 77 written:** SALE_RETURN / DC_RETURN (and some PHYSICAL) inward rows post `sml_cost_rate_wot` = the INCLUSIVE rate, which corrupts `sic_avg_cost_rate_wot` (ZT-ITEM-A wot 7.90 vs 67.19). Openings posted today are correct. The client derives Cost B.Tax and takes the server costWot only when hasCess. Switch it to always-server once notes 77 is fixed.
**2026-10-01 scan rework:** Load + Add item buttons REMOVED (load = F8 Apply & Load; + key → spare line's Barcode cell). Barcode is editable on the spare line only; Enter resolves via item-by-barcode (EAN list only — no fallback); an item already on the grid is focused, not re-added. Suite C10 scans biscuit's EAN 34343/343.
**Barcode source = item_ean_codes ONLY (user, 2026-10-01):** all barcodes were moved to the extra-EAN list; item_default_barcode is NOT a scan source. Notes 78 (default-barcode fallback) was written then WITHDRAWN and deleted — never re-propose it. Open question: the grid's Barcode column (notes 76 B) still shows item_default_barcode.
**2026-10-22, user: hide inactive levels here too (same as the item card).** `applyLevelVisibility()` hides an inactive level's Mkup / Base Rate / Price columns and drops its band (priceLvlIsActive from /price-level-masters/get). It runs after the level fetch and on layoutLoaded. Validation skips inactive levels, and the payload still sends all four as loaded. Live: C (Bulk Whole Sale) and D (Hotel) are inactive. The test reads the active levels and runs C4 on the last active one. On 10-22 `inventory.item_batch_stock` was EMPTY on the box, so C7/C8 (MRP bucket rows) fail on data, not code.

**2026-10-22 price-only Enter (user: "they have to enter the complete row").** Enter walks only the active levels' Net Rate columns via the new `NexTable::setFocusStops()` (a screen-level override of the layout's focus flags; ui_table 44 is unchanged). Past the last price it goes to the next row's first price, or to the spare line's Barcode. Mkup %, Base Rate and Min are reached by click or the arrow keys.
**2026-10-22 arrows + profit line (user: "like the price grid").** A changed Net Rate or Base Rate shows a bold ↑ green / ↓ red (the item card's `priceUpArrow`/`priceDownArrow`, theme rate.above/below) instead of the old "+3.00" text; the delegate's helpEvent tooltip says "Was 118.00 (+3.00)". `lblProfitLine` under the grid reads "Profit per PCS (no tax): WS ₹35.35 (52.6%) ↑ · RET …", built by `refreshProfitLine()` from refreshRowCard. Company: the screen always works in the login company (named in the subtitle); its rows are that company's or shared (NULL company, 12 on the box), so no Company column.
**Active levels come from `fixed.price_levels.price_lvl_is_active`** (/price-level-masters/get), NOT inventory.item_price_levels.ipl_status. At 18:34 on 10-22, VKPOS switched levels 3 and 4 ON, so both screens now show all four.
**2026-10-22 violet "four numbers" card (frameFour) REMOVED** (user: "not needed"). Its markup %, price wot and price duplicated the grid's Mkup % / Base Rate / Net Rate cells; the profit line under the grid replaces it. Lost: typing a target margin % (on price). The cess warning moved onto the profit line ("approximate — this item carries a cess"). applyFourEdit, refreshFourCard and currentLevel are gone; the Src legend chips have a Fixed size policy.
**2026-10-22 focus rules (user):** the "Remove row" button is gone ("-" on the grid removes). No QPushButton takes focus: NoFocus, no default/autoDefault; they work by click or F-key. Tab goes grid ⇄ Price from only. `focusGrid()` runs on first show (showEvent → singleShot 0) and after Clear: the first item row's price, else the spare line's Barcode. Note: the .ui was re-saved from Designer on 10-22 (multi-line format).
