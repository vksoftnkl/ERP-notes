# notes (60): "Move to damaged": the bucket move (D-A3), please build

2026-09-28. The user decided D-A3 of `plan-nestjs-stock-adjustments.md`: **build "Move to
damaged"**, because the shop has to SEE which damaged items go back to which supplier. A write-off
(DAMAGE) loses that: the stock is gone. A bucket move keeps it, still the company's stock, in the
DAMAGED bucket, until a purchase return (later) sends it back or a DAMAGE write-off clears it.

The stock adjustment screen draws it as a sixth type, **Move stock**
(`stock_adjustment_ui_mockup.png`, panel 5).

## What the database already allows

- `ck_sml_txn_type` includes `BUCKET_OUT` / `BUCKET_IN`;
- `ck_sml_bucket` has SALEABLE / DAMAGED / QUARANTINE / EXPIRED / SAMPLE.

**Nothing in the service handles them yet** (no `BUCKET_*` anywhere in `src/`), and no reason
allows them.

## The ask

1. **DTO:** `header.voucherType` gains **`BUCKET_MOVE`**, stored as `svh_voucher_type =
   'ADJUSTMENT'` the way re-lot is. No new document type, no CHECK change. A line gains
   **`toBucket`** (same enum as `bucket`).

   A bucket-move line is ONE line: `bucket` = from, `toBucket` = to, qty > 0. The server writes the
   pair:
   - `BUCKET_OUT` −1 from (lot, from-bucket);
   - `BUCKET_IN` +1 to (**the same lot**, to-bucket);
   - same godown, same quantity, **at the OUT's stamped cost** (value carried, the average
     unchanged).

   Why one line and not a pair like re-lot: the lot and quantity are identical on both sides, so
   a second line would only be something to key wrong.
2. **Rules** (in `StockAdjustmentService.check`):
   - the lot is **required**: it is what names the supplier;
   - `bucket ≠ toBucket`;
   - available in (lot, from-bucket) ≥ qty, with BLOCK as for every outward line here;
   - no free qty;
   - the freeze check as usual;
   - a document is all bucket moves or none.
3. **Accounts:** **none.** The stock never leaves the company. Same as re-lot.
4. **Reasons** (seed, shared):
   - `MOVE_DAMAGED` "Damaged — hold for return": BOTH, allowed `{BUCKET_OUT,BUCKET_IN}`;
   - `MOVE_SALEABLE` "Back to saleable": BOTH, same.

   The screen defaults the to-bucket from the reason (DAMAGED / SALEABLE) and lets the operator
   change it.
5. **Pick-stock:** please add the **lot's supplier** (`supplierId`, `supplierName`) to each
   `/stock/adjustment/pick-stock` row. With `bucket=DAMAGED` that row set IS the "what goes back
   to which supplier" list. The screen groups it by supplier.
6. **Sales must not sell from DAMAGED.** Please confirm the sale bill's lot pick and
   `sbl_available_qty` for a sale only use SALEABLE, so moved stock can't be billed by accident.
   If it doesn't, that's a fix on the sales side, not here.
7. **Tests:**
   - move 5 SALEABLE → DAMAGED: two ledger rows, average unchanged, no accounts voucher;
   - over-move refused;
   - lotless refused;
   - cancel mirrors both rows;
   - the §5.1 balance assertion = 0 after each.

## Also still owed for the screen (from the adjustments plan)

- **D-A5:** the list grid (a `ui_table` + a grid over `stock_voucher` with the five kinds) and a
  **menu id** for Stock Adjustment. **One menu for all kinds** (user, 2026-09-28: per-kind rights
  can come later).
