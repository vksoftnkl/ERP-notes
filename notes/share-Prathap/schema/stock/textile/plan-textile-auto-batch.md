# Textile — auto batch no on purchase, scan the batch at the sale

Date: 2026-10-23. New file; it edits no earlier plan.

## Why this plan

The shop finds the separate tag page too heavy. What they want instead:

- **Purchase line:** item, batch no (given automatically), MRP, selling price.
- **Sale bill:** a Batch No column. Scanning the batch fills the line from stock.

The tag plan (plan-textile-lot-tag*.md, plan-nestjs-textile-tags.md, notes-136) is **PARKED, not cancelled**.
- Nothing in it is built on the server.
- The Qt "New tags" tab and ui_table 53 stay in the code. They have no menu.

## Verdict: the current stock structure carries this with no new table and no new column

| Need | Where it already lives (live 192.168.0.106) |
|---|---|
| batch no, MRP and selling price per purchase line | `stock.stock_lot` has `slt_batch_no`, `slt_mrp` and `slt_sale_price`, and all three are part of the lot identity (`ux_slt_identity`). |
| one lot per purchase line | The item's policy tracks Batch + MRP + Sale price. The CHECKs allow this combination, and valuation = LOT_ACTUAL. |
| the sale takes stock out of that lot | The sale bill already sends `sbiBatchNo` / `sbiLotId`, then `sales-stock.service` → `svi_batch_no` → lot pick narrowed by batch (`lotlessOutwardLine` + narrowing). |
| cost per batch | Notes 92 (LIVE): each lot is costed at its own rate. |
| batch lookup speed | `ix_slt_batch_no (company, batch_no)`. |

## Decisions (mine; stock section)

**D1. Batch number format: `<branch code><YY>-<NNNNN>`, for example `CBE26-00001`.**
- Branch code = `br_bill_prefix`, else `br_code`. If both are empty, the save is refused: "Set a bill prefix on this branch to number batches."
- YY = the first year of the accounting year.
- NNNNN = a running number, 5 wide. It grows past 99999 by itself.
- The text is upper-case, because posting upper-cases the batch key.
- Why branch + year are in the text: the counter is kept per branch per accounting year. With both in the text, a number can never repeat across branches or years, even when branches work offline (offline-sync rule).

**D2. The counter reuses the existing allocator, not a new table.**
- Use `allocateVoucherNumber()` (common/Sequence/voucher-sequence.helper.ts) with a new counter-only row in `accounts.acc_voucher_types`:
  - name BATCH; no prefix; width 5; reset NONE;
  - affects_accounts = false;
  - device code MAIN.
- The helper takes an advisory lock, so two saves at once never get the same number.
- `common/utils/batch-number.util.ts generateNextBatchNo` (MAX of existing + 1) must NOT be used. It races, and it is unsafe offline.

**D3. When a number is given.**
- At the first save of the inward document (draft or post), the server gives one number to each line whose batch is blank, on an item whose policy tracks batch.
- Edits keep the number. A deleted draft simply leaves a gap.
- A batch the user typed is kept as typed. The pharma case still works.
- Turned on by the company setting `stock.auto_batch_no` (app_setting_def, default false). When it is off, a blank batch on a batch-tracked inward line is refused, as today.
- Two lines with the same item, MRP and price still get two batches. Each line is its own label run.

**D4. New tracking preset `TEXTILE_BATCH`: "Batch + MRP + selling price (textile)".**
- track_batch, track_mrp and track_sale_price all true; signature BMS.
- LOT_ACTUAL valuation, FIFO issue, negative ALLOW (the counter is never blocked).
- It is set on the textile item groups.
- MRP is required on the purchase line. Where the goods carry no printed MRP, the user enters MRP = selling price.

**D5. Scan route (new, read-only).** `GET /api/v1/stock/lots/by-batch?companyId=&branchId=&batchNo=[&includeEmpty=false]`
- Matches upper(trim(batch)) on `ix_slt_batch_no`.
- Returns per lot: lotId, itemId, item code and name, base unit, batchNo, mrp, salePrice, and on hand per godown in that branch.
- Lots with no stock are left out unless `includeEmpty=true`.
- Zero rows → 404 with the plain text "No stock with batch CBE26-00001".

**D6. Sale bill (Qt).**
- The Batch No column already exists (SalesItemCols::BatchNo, LotId). Make it visible through the ui_table.
- Scan: the EAN lookup is tried first, unchanged. If nothing is found, try the D5 route.
- Typing in the Batch No cell does the same lookup.
- **Exactly one lot found:** fill item, base unit, MRP (MaxPrice), rate = the lot's selling price, sbiBatchNo and sbiLotId. Promotions still apply afterwards. Tax comes from the item, as for any line.
- **Several lots found:** same picker style as the MRP popup (TxnPriceBucketDialog).
- **Also:** DC and sale return accept the same scan. A return then comes back into the same lot.

**D7. Changing the price of an old batch** is a Re-lot adjustment (menu 264, which exists). The new lot keeps the same batch no with the new price, so the old label still scans; reprint the label for the new price. The old lot drops to 0 and leaves the scan list.

**D8. Labels.** Print from the inward document through the printing module (17_printing): a new document kind BATCH_LABEL, with copies = line qty. Built after D1–D6.

## Build order

1. **Server S1:** the D4 preset; the D2 voucher-type row; the setting; number allocation in the **opening stock** save. Opening stock goes first because it is the go-live path for existing textile stock, and because no purchase module exists yet.
2. **Server S2:** the D5 route.
3. **Qt Q1:** the opening stock screen shows the given batch no after save, and the batch cell may be left blank.
4. **Qt Q2:** the sale bill column and scan (D6), then DC and return.
5. **Purchase module.** It does not exist yet (only suppliers, on both the server and the DB), so it gets its own plan. Its line carries item / batch (blank = auto) / MRP / selling price / qty / rate, and it reuses D3.
6. **Labels (D8).**

## Rule checks

- **Offline:** numbers are unique by construction (branch + year + per-branch counter); the lot id is already deterministic (uuid-v5); no running-total CHECK.
- **No DB functions or triggers:** all of this is in NestJS.
- **No new table:** the counter uses acc_voucher_seq, and the lot holds the price.
- **Code read, not assumed:** posting.helper lotlessOutwardLine / narrowing; voucher-sequence.helper scope = (type, company, branch, acc_year, device, period).
