# notes-150 — Physical Stock (menu 45): server review

Date: 2026-10-24. From: Viji / Claude. To: Prathap.

This is the same kind of review as notes-143 (Opening Stock), done on `/stock/physical/*`. Code paths are from the 10-23 snapshot (`src/modules/stocks/…`). Every item marked **LIVE** was reproduced on 192.168.0.106 with ZT-ITEM-A in Trichy QUARANTINE. Everything the test made was cancelled, and the holding is back at 0.

The Qt screen was reworked the same day: Opening Stock's layout, the shared keys, ↓ for the item list, and a batch-label scan. That needs nothing from you except items 9 and 10.

## Fix first

1. **Duplicate holding lines are accepted, and each posts its own variance (LIVE).**
   - The duplicate check is `lineNo|splitNo` only (`stock-voucher.service.ts:696`).
   - A draft with two lines of the same lot × bucket, each counted 3 against book 0, saved (201). Validate said "All 2 lines are clean".
   - Posting it would put +6 on a shelf that holds 3.
   - **Please refuse a repeated `lotId|godownId|bucket` at save (422).**
   - The Qt screen never sends a duplicate, but the API must not depend on that.

2. **Two frozen drafts on the same godown block each other for good (LIVE).**
   - Saved PHY0002 and PHY0004, both with freeze on Trichy. Each one's validate says "frozen for physical count <the other>".
   - Posting PHY0002 gave 422 "frozen for physical count PHY0004". Neither can post until one is cancelled.
   - The frozen CTE (`:2149-2183`) and `assertNotFrozen` (`stock-posting.service.ts:172`) exclude only the document itself.
   - **Please refuse a second open freeze on the same godown at save.**

## Should fix

3. **A backdated `docDatetime` gets round the freeze (LIVE).**
   - The window is checked against the movement's own timestamp, which comes from the body (`stock-posting.service.ts:176`, DTO `:279`).
   - A count keyed with now − 3h validated clean while PHY0002's freeze was on.
   - Backdating is needed for an offline till. Please accept it only from a device with an offline session, or cap how far it can trail `now()`.

4. **Re-saving a draft quietly re-bases the book figure (LIVE).**
   - `replaceLines` re-reads the balance on every save (`:1322`, `:1415`).
   - Stock moved 0 → 5 after the sheet was drawn. Post correctly refused the stale draft. But re-saving the same lines turned it into book 5 / counted 3 / diff −2, and nothing told the user.
   - **The Qt screen now says so** after a save: "Stock moved since these lines were drawn … Line 1 … book 0 → 5".
   - Please also return a `bookChanged` flag (or the old book) per line, so the warning doesn't rely on what the client remembers.

5. **Header totals, lineCount and createdBy are stored from the body (LIVE).**
   - lineCount 77 (for 2 lines), totalQty 999999 and createdBy "ZT-SPOOFED-AUTHOR" were all stored as sent (`:1150`, `:1205`).
   - Post recomputes the totals; a draft never does, and createdBy is never corrected.
   - Same ask as notes-143: compute them on the server, and take the author from the token.

6. **No menu-45 rights check.** None of the 7 routes calls `assertMenuRight`. Any logged-in user can create, post or cancel a count.

7. **Tenant and company come from the body.**
   - `svhTenantId: header.tenantId` (`:1115`, `:1239`).
   - `companyId` / `branchId` are never compared with the token's company. The guard sets it (`access-token.guard.ts:46`), but nothing reads it.

8. **An older count can be cancelled after a newer one.**
   - Cancel refuses only a negative result. LIVE: PHY0005 (+5) was correctly refused after PHY0006 (−3) had posted.
   - But when the reversal stays non-negative it goes through. That undoes an old variance on top of a newer count.
   - Please refuse the cancel when a later posted count of the same holding exists.

## Textile (for the batch label scan)

9. **`count-sheet` doesn't return size / colour / model.**
   - `/get` already returns them.
   - Please left-join `stock.stock_batch_detail` on company + `upper(btrim(batch_no))` and add `sizeName`, `colourName` and `model`.
   - The screen shows them after the item name ("Shirt · XL · Maroon · Slim fit"). It is already coded to read the three keys from both routes.

10. **`/stock/lots/by-batch` counts SALEABLE only.**
    - The count screen scans a batch label and asks this route which lot(s) it names, with `includeEmpty=true`.
    - A QUARANTINE or DAMAGED holding of that batch is not resolvable today.
    - Please add an optional `bucket` (or "all buckets") and `godownId`.
    - Could not be tested live: no lot carries an auto batch number yet. That waits on notes-149 #1.

## Minor

- `count-sheet` requires `accYear` but never uses it (`:3035`).
- The frozen message prints "until 2026-10-24 13:30". It should be 12-hour ("01:30 PM"), the house rule.

## Checked and fine (live)

- countedQty negative → 400; null → 422 `COUNT_NOT_COUNTED`.
- Line godown ≠ header → 422. Lot from another godown or item → 422.
- `bookQty` in the body → 400. The stale book is refused at post.
- Re-post or edit of a posted count → 409.
- Save-and-post works, and is refused inside a freeze window. Cancelling a draft lifts its freeze.
- Lock → draft check → validate run in one transaction (no save-vs-post race).
- The holding lookup is scoped to company and branch.

## STATUS (Prathap fills in)

- 1:
- 2:
- 3–8:
- 9–10:

## Client verification (2026-10-24, Viji / Claude)

Probed live on ZT-ITEM-A in Trichy QUARANTINE. Every count was cancelled, and the holding is back at 0.

| # | Result |
|---|---|
| 1 | **DONE.** Two lines on one holding → 422 `STOCK_COUNT_HOLDING_DUPLICATE` |
| 2 | **DONE.** A second freeze on the same godown → 409 `STOCK_FREEZE_GODOWN_HELD` ("already frozen for physical count PHY0021") |
| 3 | **DONE.** docDatetime 3 hours back → 422 `STOCK_DOC_DATETIME_OUT_OF_RANGE` |
| 4 | **DONE** when the line sends its `sviId`: `bookChanged: true`, `previousBookQty: 0`, plus `bookChanges[]`. The Qt screen didn't send `sviId`; it does now, and the save message uses your flag ("Line 1 … book 2 → 3", test T3). |
| 5 | **DONE.** Totals are computed (lineCount 1, qty 2, value 120 when 77 / 999 / 123456 were sent); createdBy = the login |
| 6, 7 | Not verified live: it needs a user without menu 45, and a second company with stock |
| 8 | **DONE.** Cancelling the older count → 409 `STOCK_COUNT_SUPERSEDED` ("PHY0022 has been counted again by PHY0023") |
| 9 | **DONE.** count-sheet carries sizeId / sizeName / colourId / colourName / model; the screen shows "ZZ TEST B (BATCH) · L · BLUE · ZT Slim" |
| 10 | **DONE.** by-batch `bucket=ALL` + `godownId`. The count screen's label scan now uses them and adds a QUARANTINE holding (test T1) |
| minor | **DONE.** The freeze message reads "until 24-10-2026 02:34 PM" |

Tests: `SRC=test_physical_textile tests/physicalstock/run-live.sh` ALL PASS (new). `run-live.sh` ALL PASSED. `test_physical_ux` ALL PASS.
