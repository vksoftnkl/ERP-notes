# notes-149 — Opening stock: the save refuses size / colour / model

Date: 2026-10-24. From: Viji / Claude. To: Prathap. Follows notes-147 (textile auto batch).

## Where things stand

The Qt Opening Stock screen (menu 44) now has **Size, Colour and Model** columns:
- They sit right after Batch No (ui_table 27, columns 39–44 added).
- They show only when `stock.auto_batch_no` is on for the company.
- They open only on an item tracked by batch whose batch number is left blank for the server to give.

Tested live on the test company (setting turned on for the run, then removed again):
- **Works:** auto batch number. A blank batch gets `CBE26-00002`, and the screen tells the user the number after the save.
- **Works:** `/stock/opening/get` returns `sizeId / sizeName / colourId / colourName / model` per line.
- **Works:** dropdowns 70 / 71, quick-add through `/item-sizes/create` and `/item-colours/create`, and `/stock/batch-detail/models`.
- **Does not work:** saving a line with a size, colour or model. Details below.

## 1. `/stock/opening/create` refuses `sizeId`, `colourId` and `model` (blocker)

```
400 VALIDATION_WHITELIST
lines.0.sizeId   property sizeId should not exist
lines.0.colourId property colourId should not exist
lines.0.model    property model should not exist
```

The three fields were added to **`SaveStockVoucherItemDto`**, which is the stock *transfer* line. The opening save uses **`SaveOpeningStockVoucherItemDto`**, and that DTO doesn't have them.

**Please** add `sizeId` (uuid, nullable), `colourId` (uuid, nullable) and `model` (string ≤ 100, nullable) to `SaveOpeningStockVoucherItemDto`, and write them to `stock_batch_detail` in the same transaction as the batch number.

What the client sends:
- The three keys go only on a line whose item tracks batch, while the setting is on, and only when the batch number was not typed by hand.
- An empty value is **left out**, not sent as null. On a draft update (a full replace of the lines), please treat a missing key as "none", so that clearing a size on a draft clears it.

## 2. On a draft re-save, the line sends back the batch number you gave it

When a draft is opened again and saved, the line carries `batchNo: "CBE26-00002"`, the number the server gave on the first save, together with `sizeId` and the other two.
- Please don't treat that as a typed batch (`STOCK_BATCH_DETAIL_NEEDS_AUTO_BATCH`).
- A batch that already has a `stock_batch_detail` row from this same document line counts as given.

## 3. Cancelling a draft leaves its `stock_batch_detail` row live

Opening drafts OPN0015 and OPN0016 were cancelled. Their rows `CBE26-00001` and `CBE26-00002` still have `sbd_is_deleted = false`.
- Plan E2 says that deleting a draft soft-deletes its rows.
- For a draft, cancel is the delete, so it should soft-delete them too.
- Otherwise, the next purchase-module scan reads size and colour from a batch that holds no stock.

## Test

```
SRC=test_textile_cols tests/openingstock/run-live.sh
```

Current result: **36 of 37 pass**. The one that fails is X10 (item 1 above).
- The test turns the setting on for Amazon Retail and removes it at the end.
- Everything it creates is named `ZT-TX-…` and is deleted at the end. Its drafts are cancelled.

## STATUS (Prathap fills in)

- 1:
- 2:
- 3:

## Client verification (2026-10-24, Viji / Claude) — all three DONE

- **#1:** the opening save takes `sizeId` / `colourId` / `model`. `SRC=test_textile_cols tests/openingstock/run-live.sh` → **ALL PASS** (37). X10 saves size, colour and model, reads them back, and a reloaded draft shows them.
- **#2:** re-saving a draft with the batch number it was given (CBE26-00005), plus a changed size and model → 201, updated, same batch.
  - Leaving the three keys out on a re-save clears them (full replace). Good.
  - A typed batch with a size → 422 `STOCK_BATCH_DETAIL_NEEDS_AUTO_BATCH`. Good.
- **#3:** cancelling a draft soft-deletes its batch detail. The older rows (CBE26-00001…00004) were cleaned too.

One question, not a defect: cancelling a **posted** opening leaves its batch detail live (CBE26-00006, from the Physical Stock textile test). Is that by design, because the lot still exists at 0? If so, nothing to do.
