# Notes 74 — Change Selling Price: one item's rows, and unpriced stock buckets in F12

2026-10-01. Found on menu 30 with **MRP ITEM** (`01a0f5ff-1e28-75e5-9dab-35fc9b65ff89`).

**Starting state on the box**

- MRP ITEM has the MRP_SELLING preset.
- An opening posted 18 PCS at MRP 300 in Acme Foods / Coimbatore, at cost 200.
- Its only price row is a blank shared headline: company NULL, branch NULL, MRP 0, prices 0.

The two read routes give different answers for the same item:

| Route | MRP ITEM |
|---|---|
| `GET /stock/price-bulk` (grid) | ✓ `stockQty 18, mrp 300, maxPrice 0, MASTER, CHAIN`: the stock bucket, with no row of its own yet |
| `GET /stock/price-buckets/:itemId` (F12) | ✗ only the headline: `stockQty 0, mrp null, MASTER, CHAIN` |

## 1. `/stock/price-bulk` has no item filter

The screen's **Add item** (one item by search) needs exactly what the grid route answers: every unit × live bucket of ONE item, with stock.

The route has no `itemId` filter, so the client currently works around it:

- it reads the item's group / brand / section / supplier;
- it loads the grid narrowed by all four;
- it keeps that item's rows.

That works, but it costs two calls and can page through a large group.

**Ask:** add an optional `itemId` (uuid) to `ListSellingPriceQueryDto`, ANDed like the other four filters. The client will switch to it when it is there.

## 2. F12 (`/stock/price-buckets`) misses a bucket with stock but no price row

The route is documented as "every live price row of one item this branch can see". It does what that says: it lists PRICE rows only.

But the F12 list is where the operator goes to see what an edit hides and which buckets exist. A received MRP that nobody has priced yet is the one bucket that matters most there, and it is missing. MRP ITEM's MRP 300 (18 on hand) does not appear at all; F12 shows only the headline, stock 0.

The mockup (panel 9) and the screen plan expect every bucket that has stock, plus every priced row. A bucket with neither is not expected.

**Ask:** union in the stock buckets that have no row of their own, the way Q25 does for the grid:

- `priceSource MASTER`;
- `mrp` / `salePrice` set, blanked by the policy;
- `stockQty` = on hand for that bucket at this branch;
- the headline's prices and cost, so picking one shows what that stock sells at today.

That also gives `stockQty` on a bucket row the same meaning in both routes.

**Test:** `GET /stock/price-buckets/01a0f5ff-…?companyId=019c8ea6-…&branchId=019c8ea7-…` should list the headline AND the MRP 300 bucket, with `stockQty 18`.

## Client side (done)

- Add item now reads the narrowed grid, not F12. MRP ITEM shows `MRP 300 · stock 18 · NEW`, and Save would insert the bucket.
- The MRP column shows the bucket's own MRP when `maxPrice` is 0, which is the case for an unpriced bucket.
- Live test: `tests/sellingprice/run.sh` C7 (read-only on MRP ITEM).
