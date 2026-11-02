# notes-151 — Stock Adjustment (menu 264): server review

Date: 2026-10-24. From: Viji / Claude. To: Prathap.

This is the same kind of review as notes-143 (Opening) and notes-150 (Physical), done on the stock adjustment routes. Code paths are from the 10-23 snapshot: `src/modules/stocks/stock-adjustment/*`, the shared `stock-voucher.service.ts` and the posting helper.

**LIVE** means reproduced on 192.168.0.106 with ZT-ITEM-A / ZZT-B. Every test document was cancelled and every quantity is back where it was. The one exception is item 2.

The Qt screen was reworked the same day: Opening Stock's layout, the shared keys (F2 = Edit, Pick from stock moved to F12), ↓ for the item list, and a batch-label scan through Pick from stock.

## Fix first

1. **No menu-264 rights check, and company / branch come from the body.**
   - Neither the controller nor the service calls `assertMenuRight`; only `AccessTokenGuard` runs.
   - LIVE: `pick-stock?companyId=<another company>` answers 200 "0 holdings" instead of 403.
   - Please add the rights check, and compare company / branch with the token's.

2. **Cancelling an adjustment leaves the moving average at the cancelled document's cost (LIVE).**
   - ZT-ITEM-A QUARANTINE had average 50 and `stock_item_cost` 50 / 42.372.
   - Posting an IN of 10 @60 and then cancelling it left the holding average at 60 and `stock_item_cost` at 60 / 60.
   - Cause: `rebuildItemCosts` (posting helper ~2622) takes the latest row's rate when quantity is 0, and the latest row is the reversal. Reversed pairs should not count.
   - **This is still on the box.** ZT-ITEM-A's average is 60 (it was 50), and needs a DB fix after the code fix.

## Should fix

3. **Draft totals and createdBy are stored from the body (LIVE).**
   - A one-line draft sent with lineCount 77, totalQty 999 and totalValue 123456 stored exactly that (`writeHeaderTotals`, `stock-voucher.service.ts:1195`).
   - `actorFor(header.createdBy)` (`:3613`) let a save be recorded as another user ("ravi").
   - Same ask as notes-143 / 150: compute the totals on the server, and take the author from the token.

4. **`docDatetime` has no limits, and the freeze check runs on it.**
   - LIVE: a draft dated 2026-10-24 with `docDatetime` 2020-01-01 saved fine.
   - `assertNotFrozen` (`stock-posting.service.ts:175`) and the validate freeze check use `svh_doc_datetime`. That comes from the client, or is the draft's creation time, and an update never changes it.
   - So a draft keyed before a physical count, or a backdated one, posts into a frozen godown. Same as notes-150 #3.

5. **The adjustment rules run outside the post transaction.**
   - `stock-adjustment.service.ts:369-373`: `check()` reads the lines before the shared post locks the row.
   - Stock and freeze are checked again under the lock, but re-lot balance, expiry, reason direction and remarks are not. A draft save that lands in between can post lines that never passed those rules.
   - Save with `status: POSTED` skips `assertRelotLandsElsewhere`, which is only wired on `/post` (`:386`).

6. **A re-lot can change the bucket (LIVE).**
   - RELOT_OUT 4 from SALEABLE plus RELOT_IN 4 into QUARANTINE saved, and validate said "All 2 lines are clean".
   - That is a bucket move under a re-lot reason, with no accounts entry.
   - The pair check (`:699-731`) should require the same bucket.

7. **The lot's company is never checked.**
   - `loadLots` (`:1141`) and the shared validate check only that the lot belongs to the item.
   - An inward line naming another company's lot of a shared item resolves through `l.slt_id = svi_lot_id` with no company filter (posting helper `:843`).

## Textile

8. **`pick-stock` and `/get` lines don't return size / colour / model.**
   - Please join `stock_batch_detail` (company + upper batch) and return `sizeName`, `colourName` and `model`.
   - The screen already shows them after the item name ("Shirt · XL · Maroon · Slim fit"), in the grid and in Pick from stock.

9. **Pick-stock has no exact batch filter.** `search` is a loose match on name, code and batch.
   - The screen's batch-label scan opens Pick from stock searched for the label, across all buckets of the godown, so today the user picks the right row.
   - An exact `batchNo` parameter would let a single match be taken straight away.

10. **`/stock/lots/by-batch` cannot serve an adjustment.** It covers SALEABLE only, has no bucket on its rows and no cost.
    - Same ask as notes-150 #10: a `bucket` parameter (or all buckets, with the bucket on each row), a `godownId` filter, and `avgCostRate`.

11. **For information:** a re-lot keeps the batch number only when the item also tracks MRP or sale price.
    - On a batch-only item, the same batch resolves to the same lot and is refused (`RELOT_SAME_LOT`, LIVE on ZZT-B).
    - Where the batch is kept, its detail row still applies through `ux_sbd_batch`, so no copy is needed.

## Checked and fine (live)

- Taking out more than on hand → 422 INSUFFICIENT_STOCK.
- Two lines on one holding are summed → 422.
- Write-off of an item with no stock → 422.
- Line godown ≠ header → 422 ADJ_LINE_OTHER_GODOWN.
- Move to the same bucket → 422 MOVE_SAME_BUCKET.
- Re-lot of an untracked item, or to the same batch → 422 RELOT_SAME_LOT.
- Expiry write-off of a lot with no expiry → 422.
- DAMAGE citing RELOT_OUT → 422 REASON_NOT_ALLOWED.
- A partial re-lot carries the old lot's cost.
- Cancelling a re-lot whose new lot was partly used, or an IN whose stock a later move holds → 409 CANCEL_WOULD_GO_NEGATIVE.
- A move and a re-lot post no accounts entry. An IN or OUT adjustment posts its entry and reverses it on cancel.
- Post and cancel take the actor from the login.
- An IN line uses a keyed cost only when MANUAL.

## Left on the box by the review

- ZT-ITEM-A QUARANTINE average is 60 (was 50), and `stock_item_cost` is 60 / 60 (was 50 / 42.372). See #2.
- Zero-quantity balance rows: ZT-A SALEABLE, and ZZT-B lots ZT-B1 (`8efb019b…`) and ZT-B2 (`a928761f…`).
- A new ZZT-B `stock_item_cost` row at 40.

## STATUS (Prathap fills in)

- 1–2:
- 3–7:
- 8–10:

## Client verification (2026-10-24, Viji / Claude)

Probed live on ZT-ITEM-A and ZZT-B in Trichy. Every test document was cancelled.

| # | Result |
|---|---|
| 1 | **DONE (rights).** All routes check menu 264 (read in the code; it can't be tested live without a user who lacks 264). A pick-stock for another company still answers 200 "0 holdings". Your comment says the company is deliberately not compared, as in notes-150 #7. Accepted. |
| 2 | **DONE.** IN 10 @70 → avg 70. IN 10 @50 → 60. Cancel the second → holding avg and `stock_item_cost` are both back at **70 / 70**. The old drift on ZT-ITEM-A is gone too (rebuilt to 0 with no live rows). |
| 3 | **DONE.** Draft sent with 77 / 999 / 123456 stores lineCount 1, qty 10, value 700. createdBy = the login, not the body's user. |
| 4 | **DONE.** docDatetime 2020-01-01 → 422 `STOCK_DOC_DATETIME_OUT_OF_RANGE`. |
| 5 | **DONE (read in code).** The rules run again inside the post transaction (`beforePost` hook, `STOCK_ADJ_KIND_CHANGED`). Save with status POSTED now runs the same hooks, including `assertRelotLandsElsewhere`. Not testable as a race. |
| 6 | **DONE.** RELOT_OUT SALEABLE + RELOT_IN QUARANTINE → 422 `STOCK_ADJ_RELOT_BUCKET_MISMATCH`. |
| 7 | **DONE (read in code).** `LOT_OTHER_COMPANY` in the adjustment rules and in the shared validate. The box has no other company's lot of a shared item to try it with. |
| 8 | **DONE.** pick-stock and the load both return "ZT-PTX-L-29381 / ZT-PTX-BLUE-29381 / ZT Slim" for CBE26-00006. |
| 9 | **DONE.** `batchNo=cbe26-00006 ` (lower case, trailing space) → exactly 1 row. `batchNo=CBE26` → 0 rows. |
| 10 | **DONE.** by-batch `bucket=ALL&godownId=` rows carry `bucket` and `avgCostRate` (40). |

**Client changes.** The Stock Adjustment screen's batch-label scan now calls pick-stock with the exact `batchNo`. One holding is taken straight away, several open Pick from stock on that batch, and none says "Not found". Test A10 in `test_adjust_ux` covers it.

**Box note (not a defect).**
- After #2, ZT-ITEM-A has no cost history, so stock-in at AVG_COST resolves to 0 and is refused (`STOCK_LINE_COST_UNRESOLVED`). That broke our two live suites.
- To keep them working, I posted a cost anchor: ADJ/2026-2027/debug-void-dev/24, 1 pc DAMAGED in Coimbatore godown @50. Please leave it.

**Question.** A physical count line carries no cost, so a count that finds stock of an item never bought (no average, no last purchase) cannot post. The screen offers MANUAL as a rate source, but there is nowhere to type the cost. Is that intended? If so, the user has to do a stock-in adjustment with a keyed cost first. If not, we'll add a cost cell under MANUAL. Tell us what the count DTO accepts.
