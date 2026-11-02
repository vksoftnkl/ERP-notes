# Notes 71 — one price table: live check of the 2026-09-30 build

Against https://192.168.0.106:3011 after the restart (Swagger 442 routes, the four new lookup
params present). Method: a `ZT-` group with the MRP_SELLING preset, a `ZT-` item under it with
two price rows (MRP 40 → 38, MRP 45 → 44), then the lookup, the bucket list and the bulk save.
Both ZT- items and both ZT- groups are soft-deleted again. Plan: `plan-nestjs-one-price-table.md`.

## Works (verified)

| # | Check | Result |
|---|---|---|
| 1 | Migration `20260930160000_one_price_table` | applied: bucket columns, generated keys, dates, `ex_ipm_overlap`, the four checks, `ix_ipm_lookup`; old unique index gone; all 48 rows headline |
| 2 | Bucket derivation on `POST /items/create` (§4) | two rows under an MRP-tracking group came back `ipm_bucket_mrp` 40 and 45, `ipm_key_sp` −1; a client-sent `ipm_bucket_mrp` is a 400 "should not exist" |
| 3 | Lookup `mrp=45` / `mrp=40` (§3) | 200, `price_source` BUCKET, `price_scope` BRANCH, `max_price` and `sales_price` of that bucket |
| 4 | Lookup on an untracked item with `mrp=999` | 200, MASTER — the policy blanked the key, as §2 requires |
| 5 | `lot_id` validation | a non-UUID is a 400 with the field named |
| 6 | `GET /stock/price-buckets/:item` | both buckets, BUCKET/BRANCH, stock 0, level prices |
| 7 | `GET /stock/price-bulk` | 200, `data.items[]` with priceSource / priceScope / bucketId / levels — the 503 is gone |
| 8 | Bulk save above MRP (45 → 46) | 422 "These prices cannot be saved" (Q26 + `ck_ipm_not_above_mrp`) |
| 9 | Bulk save for a deleted item | 422 naming the unit — fine |

## Broken

| # | Finding | Evidence |
|---|---|---|
| **B1** | **Every VALID bulk save is a 400.** Update of an existing bucket (with or without `bucketId` / `priceScope`) and insert of a new bucket (MRP 50) all fail with `{"field":"request","message":"originalRecord and modifiedRecord are required for update audit log"}` and nothing is written. The audit-log call in the save path is passed without the two records. Menu 30 cannot save anything until this is fixed. | three POSTs to `/stock/price-bulk`, all 400; DB rows unchanged (44.00 stayed) |
| B2 | An MRP-tracked item whose rows are ALL buckets has no headline: the lookup with an unknown MRP (50) or with no MRP at all is a **404 "Item price not found"**, and `buckets[]` is empty because it lists only buckets WITH STOCK. A new item priced on the item card but not yet received therefore cannot be put on a bill or a quotation at all, and the screen has nothing to offer. Two options: (a) `buckets[]` lists every live bucket row when the item has no stock (the price-buckets route already does), so the screen can offer them; (b) the item card also writes a headline row. (a) is one condition in the lookup and matches the bulk screen's own rule (Q25 adds the headline row for items with none). | lookup `mrp=50` → 404; lookup with no mrp → 404, buckets [] |
| B3 | **Deleting a group does not retire its GROUP-scope policy.** Both ZT- groups are soft-deleted; both policy rows are still `stp_is_deleted = false, stp_is_active = true`. Clearing the preset retires it (verified in notes 70), so the delete path just misses the same call. Belongs with notes 70 B4. | `stock_track_policy` for scope ids 01a0f257-a17a…, 01a0f259-c563… |
| B4 | Rows created through the item card carry `ipm_price_*_wot` and `ipm_price_*_markup_perc` = 0 when the client did not compute them (my payload sent prices only). The Qt item card always computes them, so this is a note, not a defect: the server does not derive wot/markup on the composite save the way the bulk save does (§5.1 "price wins"). Decide whether the composite should. | `ipm_price_a_wot` 0 on both ZT rows |

## Not covered

- Sale price buckets (`track_sale_price`): only MRP buckets were exercised.
- The chain-row override path of the bulk save (`scope: CHAIN` needs an HQ user; VKPOS is not one) — blocked by B1 anyway.
- The re-key on a policy flip (`PriceBucketService.rekeyItem`): not exercised.
- The API dropped twice during the run (18:08 and ~18:40); every result above is from a window in which it answered.
