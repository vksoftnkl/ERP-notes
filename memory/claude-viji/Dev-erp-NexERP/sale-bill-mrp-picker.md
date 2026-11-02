---
name: sale-bill-mrp-picker
description: Sale bill MRP popup (TxnPriceBucketDialog) built 2026-10-06 — /item-price buckets[], Ask/Keep/Default modes, re-ask with mrp; tests/mrpbucket 13/13
metadata:
  type: project
---

**User ask 2026-10-06:** "build it with popup for multiple MRPs".

**What I built:**
- `src/shared/ui/txn/txn_price_bucket_dialog.{h,cpp}`: MRP / Sale price / Rate / Stock. The first row with stock is pre-selected; Enter picks, Esc cancels.
- `ItemPriceDetail::Params` gains `mrp` / `salePrice`.
- `SaleBillEntry::fetchItemPrice(row, item, unit, PriceBucket mode, mrp, sp)`:
  - **Ask** = item pick / scan. One bucket is taken silently; several open the popup; Esc → `dropBillItemRow`.
  - **Keep** = level step / apply level / unit switch. Re-sends the line's own MRP.
  - **Default** = promotion free line. First bucket with stock, no popup.
  - Either way it **re-asks /item-price with mrp** (the server's documented contract).

**Server facts:**
- `buckets[]` comes only for MRP / SP-tracked items, ordered by MRP DESC.
- With `mrp` stated and a price row for it → BUCKET answer (MRP ITEM 300 → 220).
- With NO price row for that MRP (MRP STOCK ITEM's 4 MRPs, all unpriced) → the headline answers with max_price 0, so the client keeps the chosen MRP on the line and the rate is keyed.

**Not done:** DC (delivery_challan_entry) has no picker yet. With MaxPrice null its stock goes by FEFO.

**Test:** `tests/mrpbucket/run.sh` (read-only, offscreen), 13/13.

Related: [[lot-cost-valuation]] (notes 93: MRP 0 = not stated).
