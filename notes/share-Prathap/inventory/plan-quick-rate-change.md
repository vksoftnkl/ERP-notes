# Plan — Quick rate change popup (from a sales line / the item list) (2026-10-23)

**User's ask:** "i want to change only price using price grid … user can change
rate instantly from sales entry / item main list". The reference is the 3.0
"Change Selling Price" popup (screenshot 2026-10-23 13-56-45): one item, a
price grid per unit, Save F5, and the item's purchase history below.

Mockup: `quick_rate_change_mockup.png` (same folder).

Status: **analysis + mockup for approval.** Nothing is built.

---

## 1. What 3.0 does, and what to keep

| 3.0 popup | Keep? |
|---|---|
| One item, opened from the bill / item list | **Yes.** That's the whole point: no leaving the bill. |
| Grid: Conv, Packing, Stock, Cost.Wot, Cost, Profit Type, Round Off, Profit A, WS Wot, WS Price, WS Mgn%, Profit B … in one long row with a horizontal scrollbar | **Reshape.** Our levels are A–D with markup and price, grouped under a header band per level, so nothing scrolls sideways. Profit Type is hidden (our rule: profit % on cost **w/o tax**, notes-140). Round Off is applied, not shown. |
| Purchase history: Date, Supplier, Qty, B.Tax Rate, Cost, MRP | **Yes**, as "Recent inward" (§3.2). |
| Save F5 / Cancel | Ctrl+S / Esc (the shared key set, `TxnKeys`). |
| — | **New:** the item's last rate changes, from the price change log (who / when / old → new). This is the "how did the rate become 118" answer, shown right where the rate is changed. |
| — | **New:** "This branch / All branches" and "From today / From a date", the same choices menu 30 has. |

## 2. Behaviour

- **Opened by** a key on the current line of Sale Bill, Quotation, Sale Order and DC, and on a row of the Item list.
  - Proposed key: **Ctrl+R** ("Rate"). Not taken on those screens, and not used for typing. Also listed on F1.
  - Also from a right-click menu on the line.
- **Shows** every price row of that item the branch can see, which is what `/stock/price-bulk?itemId=` already returns:
  - one row per unit × MRP bucket (PCS, BOX ×12, "PCS · MRP 150");
  - each row with stock, cost w/o tax and cost.
- **Edits**
  - Editable: Markup % or Price for A–D, Min price, and MRP (on a new bucket row only, as menu 30).
  - Typing a markup sets the price, and typing a price sets the markup, both on cost **w/o tax**.
  - A changed cell is tinted and shows its delta ("+10").
  - A price below cost turns red and asks to confirm on save (the server's `needsConfirm`).
- **The live line** under the grid explains the current cell. Example: "Retail A 118 → 128 (+8.5%) · profit 61.4% on cost w/o tax 79.29 · below MRP ✓".
- **Save (Ctrl+S)** saves the changed rows only (`POST /stock/price-bulk`, same contract as menu 30), so the price change log records it. The popup then closes.
  - **"Apply the new rate to this line"** (shown only when opened from a bill/quotation, ticked by default): after saving, the line is re-priced through the screen's own price lookup, so discounts, tax and loyalty follow. Other lines keep their prices.
  - From the item list, the list row refreshes.
- **Rights:** the popup edits only if the user has **Edit on menu 30** (Change Selling Price). Without it, the popup opens read-only with a note, and stays useful for looking up cost and history. The server already checks the right on save.
- **Esc** with unsaved changes asks first.

## 3. Backend

### 3.1 Reused as they are
- `GET /stock/price-bulk?companyId&branchId&itemId=` gives the grid rows (one item).
- `POST /stock/price-bulk` saves: `scope`, `effectiveFrom`, `confirmed`, rows. That brings the below-cost confirm, dated prices (notes 140) and the price log (notes 144, source `CHANGE_SELLING_PRICE`).
- `GET /item-price-log/item-history?itemId=` gives the last changes (notes 144).

### 3.2 New — one small read route
`GET /stock/item-inward-history?companyId&branchId&itemId&limit=10`

The last inward rows of the item from `stock.stock_ledger`:
- `sml_direction = 1`, `NOT sml_is_reversal`, `sml_txn_type` in the inward types (OPENING today; PURCHASE / GRN when purchase lands).
- Newest first, with the reversal of a cancelled document netted out.
- Returns: date, txn type, refno, supplier (`sml_party_id` → name), qty and unit, `sml_cost_rate_wot`, tax % (`cost_rate / cost_rate_wot`), `sml_cost_rate`, `sml_mrp`, batch.

There's no purchase module yet, so the ledger is the one place every inward already lands. Purchases will appear here with no client change.

### 3.3 Small addition to the price log
`SaveSellingPriceBulkDto` gains an optional **`originMenuId`**: the screen the popup was opened from (e.g. Sale Bill). `PriceChangeLog.record()` adds it to `ipcl_reason` ("via Sale Bill, menu 31"), so the report can tell a rate changed during billing from one changed on menu 30. The source stays `CHANGE_SELLING_PRICE`.

## 4. Client (Qt)

- One shared popup: `shared/ui/txn/quick_rate_dialog.{h,cpp,ui}` (NexDialog, `EnterNavigation`), opened with `QuickRateDialog::open(itemId, unitId, originMenuId, parent)`.
- The grid reuses menu 30's column logic (`change_selling_price_entity.h`, the delegate's markup ↔ price maths and round-off), so the two screens can't disagree about a price.
- `TxnKeys` gets `Action::ChangeRate` = Ctrl+R (a line action, listed under "Lines" on F1).
- Sale Bill, Quotation, Sale Order and DC bind Ctrl+R on the item grid. The Item list binds it on its table. Each passes its menu id.
- Test: `tests/quickrate/run-live.sh`. On a ZT- item:
  - open from a bill line, change price A, save;
  - check the line re-priced, one price log row "via Sale Bill", below-cost confirm, and read-only without the right;
  - restore the price.

## 5. Questions for the user

- **Q1** Ctrl+R as the key? (Alternatives: Alt+R, F7.)
- **Q2** "Apply the new rate to this line": ticked by default?
- **Q3** Show all four levels (A–D), or only the levels this company uses (from Price Level master)? Planned: only the active levels.
