# Notes 93 — an outward line must never open a lot; MRP / price 0 is "not stated" (2026-10-06)

Found while checking notes 92 live. **Not caused by notes 92.** Both gaps were there before; notes 92's
deterministic lot id only made the phantom lot easy to date.

## What happened (live, Acme / Coimbatore)

Sale bills **bil00964** (12:10, 3 units) and **bil00967** (12:48, 100 units) of **MRP ITEM** (signature `M`):
- Each was billed from godown `019c9935…`, which holds no MRP ITEM.
- Each line went to stock with `svi_mrp = 0` (bill `sbi_max_price` 0, price 0).
- The engine did not issue from the item's real lot (MRP 300, 38 on hand in two other godowns). It **opened a new lot "MRP 0"** (`d7ee6f25-ba28-5e02-…`), sold it to **−103** and costed it at 252.63.

Two things let that through:

1. **`lotlessOutwardLine()`** sends a line to the FEFO pick only when a tracked dimension is NULL: `track_mrp AND svi_mrp IS NULL`. A line that says `svi_mrp = 0` counts as "MRP stated", so it goes to identity resolution with `key_mrp = 0`. The same holds for `svi_sale_price = 0`. No real MRP or selling-price lot is ever 0.
2. **`resolveLots()`** refuses to open a lot from an outward line only when `rules.lineDirection === 'REASON'` (the adjustment family). A sale, DC or any fixed-OUT shape whose stated identity matches no lot silently **opens** one, and the line then drives it negative.

The client half is fixed on our side: the sale bill and DC now send `sbiMaxPrice` / `sdiMaxPrice` as **null**, never 0, when the line has no MRP (Qt, 2026-10-06). Other callers (React, offline sync, imports) can still send 0, so the engine must not depend on that.

## Ask

**A. 0 is "not stated" for MRP and selling price on an outward line.** In `lotlessOutwardLine()`:

```sql
OR (keyed.track_mrp        AND COALESCE(keyed.svi_mrp, 0)        = 0)
OR (keyed.track_sale_price AND COALESCE(keyed.svi_sale_price, 0) = 0)
```

Such a line then gets the FEFO pick across the item's real lots in the godown. Batch / serial already use `COALESCE(…, '') = ''`; this is the same rule for the two numeric keys.

**B. An outward line never opens a lot for a TRACKED item, in any shape.** In `resolveLots()`, replace the REASON-only guard with:

```sql
AND (c.line_direction > 0 OR c.track_signature = 'N')
```

- **Plain items (`N`) keep today's behaviour:** they have exactly one lot per item, and selling one before its first receipt is the negative-stock case `comp_negstk_apl` already rules on. A shop must be able to bill it.
- **For a tracked item,** an outward line whose stated identity matches no lot is treated as **lotless**: add `OR NOT EXISTS (the lot by its identity keys)` to the pick's selector for outward lines. It then goes to `pickIssueLots`, narrowed by whatever it DID state:
  - stock of that identity elsewhere in the godown → issued from it;
  - none → the existing refusal, *"names no lot and this godown holds no stock of the item to issue from"*.

  A clear refusal at post is right here. The alternative is a lot that never existed, with negative stock, a cost borrowed from another lot, and a count sheet line nobody can find on the shelf.
- **Count shortages and re-lot / bucket-move OUT halves** always carry a lot from the sheet or the pick, so B does not touch them.

## Offline (notes 92 §7)

Both rules run at the BRANCH, at post, where the till's own stock is known. The centre never re-resolves a synced document, so neither rule can refuse a sale the branch already accepted.

## Acceptance (I will run it)

1. A sale bill of MRP ITEM from a godown that holds MRP 300 stock, with `sbiMaxPrice` 0 (an old client): it issues from the MRP 300 lot and opens no lot.
2. The same from a godown with NO stock of it: refused with the pick message (today: a phantom "MRP 0" lot).
3. A plain item sold before any receipt: still posts, and goes negative on its one lot, as today.
4. No tracked lot is opened by a sale or a DC. This must return 0 after the fix (today it returns **1**, the phantom):
   ```sql
   SELECT count(*) FROM stock.stock_lot
    WHERE slt_track_signature <> 'N'
      AND slt_inward_src_module = 'SALES'
      AND slt_inward_src_doc_type NOT IN ('SALE_RETURN', 'DC_RETURN')
      AND slt_created_on > '<deploy time>';
   ```
   A tracked lot is only ever opened by an inward.

## Existing data (test box only)

The phantom lot `d7ee6f25-ba28-5e02-b07b-4ba1c1cc1da8` (MRP ITEM, MRP 0, −103 in godown `019c9935…`) comes only from bil00964 / bil00967. Cancelling those two bills reverses it to 0; there is nothing to repair by hand.
