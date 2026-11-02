# notes (65): Stock Adjustment screen (menu 264), server findings

2026-09-30. The Qt screen for menu 264 is built and tested live against 192.168.0.106 as VKPOS:
- Adjustment, Move stock, Re-lot, Expiry and Issue, 50/50 checks;
- one adjustment and one move were POSTED, then cancelled;
- the holding ends where it started.

Notes 60 (Move stock) works as asked.

Found while building it: two defects, one wrong list and one doc mismatch. Nothing below is blocking: the screen works around each one and says so in its code.

## 1. Re-lot into the SAME lot passes the check and posts (defect)

The README says the IN half must be a *different* lot. `check()` compares `inLots` with `outLots`, but an inward line never names a lot, because the identity fields resolve it at post. So `inLots` is always empty and the rule never fires.

Reproduced 2026-09-30 on ZT-ITEM-A, an untracked item with one lot per bucket:
- RELOT_OUT −1 with the lot picked, RELOT_IN +1 with no identity;
- `/validate` answered "All 2 lines are clean";
- `/post` wrote 2 ledger rows, −1 and +1 on the **same lot**;
- the document was cancelled afterwards (`ADJ/2026-2027/WEB-cda79513-…/4`).

**Ask:** at post, after the IN line's lot is resolved, refuse when it equals the OUT lot of the same item. Better still, refuse at save when the IN line's identity (batch / expiry / MRP / sale price / serial / supplier, blanked by the item's policy) equals the OUT lot's.

A re-lot of an untracked item can never be valid, because it has exactly one lot per bucket.

*Screen meanwhile:* it refuses an untracked item, and an IN identity equal to the OUT's, before the round trip.

## 2. Header totals refuse a negative NET (defect)

`SaveStockAdjustmentHeaderDto` extends the shared `SaveStockVoucherHeaderDto`, whose totals are `@OptionalNumber(0)`. The README says "Header totals are the NET of the lines", and an adjustment's net is negative whenever more goes out than comes in:

```
header.totalQty must not be less than 0
header.totalValue must not be less than 0
header.totalValueWot must not be less than 0
```

The physical count's own header DTO allows signed totals (`@OptionalNumber()`).

**Ask:** give the adjustment header signed totals, the way physical has them.

*Screen meanwhile:* it sends the totals only when the net is ≥ 0, and otherwise leaves them to the post's recompute. So a net-out DRAFT shows 0 / 0.00 in grid 122 until it is posted.

## 3. Grid 122 lists the sales shadow ISSUE vouchers

`svh_voucher_type IN ('ADJUSTMENT','ISSUE','DAMAGE','EXPIRY_WRITEOFF')` also picks up every sale bill, DC and return that posted stock through the shadow ISSUE / RECEIPT voucher. Examples: `SALE_BILL/bil00937/r1` and `DELIVERY_CHALLAN/dc00056/r1`.

On the box today these are most of the rows, so the Stock Adjustment list is flooded with sales.

**Ask:** exclude the shadow vouchers in grid 122's WHERE. Whatever marks them works: `svh_link_src_module = 'SALES'`, or the column the sales posting sets.

## 4. `pick-stock` `bucket` is documented optional but required

`PickStockQueryDto.bucket` has `@IsIn(STOCK_BUCKETS)` without `@IsOptional()`. Leaving it out is a 400: "bucket must be one of the following values…". The README and Swagger say it is optional.

**Ask:** add `@IsOptional()`. Absent should mean every bucket. That would also let the damaged list and a "where is this lot" view run as one call.

*Screen meanwhile:* it always sends a bucket.

## 5. `costRate` unit: the DTO text vs the column (clarify)

`SaveStockAdjustmentItemDto.costRate` says "what the stock is worth per **document unit**". But `svi_value` is `GENERATED ALWAYS AS (base_qty + free_base_qty) * cost_rate`, as on the opening, which makes the stored rate per **base** unit.

*Screen:* it sends the keyed inward cost (MANUAL only) divided by the line's factor, i.e. per base unit, like the opening screen.

**Ask:** confirm, and fix the Swagger text if base unit is right.

## Changed on the box (display only, via the API)

- **ui_table 41** "STOCK ADJUSTMENT - ITEM" was re-registered with 31 columns:
  - the 21 you registered, plus hidden ids 21–30 (ItemId, UomId, BaseUomId, Factor, ReasonId, SupplierId, Signature, Sale price, Serial, Id);
  - visible widths sum to ~100;
  - Dir is shown, placed after Available;
  - Enter stops on Item / Qty / Reason / Remarks / To.
- **Grid 122** visible widths scaled from 113 to 100 (`PUT /grid-details/column-width`).

## Still open (not server)

VKPOS has no `user_menus` row for 264 yet, so the menu does not show in its tree. The grant is pending with the user.

## Verified live 2026-09-30 (after Prathap's fix)

All five items are closed.
1. A same-lot re-lot is refused at save, on `lines.1`, with the untracked-item message.
2. Signed totals are accepted: a draft saved with −1 / −20.
3. Grid 122 shows no shadow sales rows.
4. `pick-stock` works without a bucket.
5. The Swagger text now says per BASE unit.

The screen now sends signed totals and no longer forces a bucket. The live suite passes 50/50.
