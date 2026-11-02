# Notes 143 — Opening Stock backend review (2026-10-23)

**User's ask:** "check opening stock front end backend codes".

The client-side findings are already fixed in the Qt app. These ten are on the
server.

**How this was checked:** read-only review of the API source snapshot plus the
live schema (SELECT and `\d` only); nothing was run against the API. I re-read
the code myself for #1, #3, #4 and #9. Paths below are relative to
`src/modules/stocks/`. SVC is `stock-voucher/stock-voucher.service.ts`, PH is
`stock-voucher/stock-voucher-posting.helper.ts`. There are no business triggers
on `stock.*`, so no trigger blocks any of these.

**Already sound:**
- Save-and-post is atomic: a 422 rolls the whole save back, draft edits included.
- Double post and double cancel are blocked (`lockHeaderInStatus` takes FOR UPDATE).
- Line renumbering parks lines before giving them new numbers.
- Reversal rows mirror the originals, and `ux_sml_reversal` stops a second mirror.
- Accounts amounts come from the ledger rows, not from the payload.
- notes-141 is live.

---

## 1. A draft save can change a document that was posted a moment earlier (HIGH)

**Where:** SVC `loadForWrite` (~3083) reads the status with a plain `findUnique` (no `FOR UPDATE`). After that, `updateDraft` (~1173) updates by primary key only. `replaceLines` and `writeHeaderTotals` follow.

**Scenario:**
1. `/post` takes the row lock.
2. A `/create` draft save of the same `svhId` passes its DRAFT check, then waits on that lock.
3. The post commits.
4. The save then rewrites the lines (clearing `svi_lot_id`) and the totals of a POSTED document.

The ledger and the document no longer agree.

**Fix:** take `FOR UPDATE` in `loadForWrite`, or add `AND svh_status = 'DRAFT'` to the update and treat 0 rows as 409.

## 2. "Already opened" is not checked under a lock, and `/post` does not re-check what it posts (HIGH)

**Where:**
- `post()` runs `assertPostable` before its transaction (SVC ~2280).
- Save-and-post runs it inside the transaction, but before any lock (SVC ~385).
- The engine (PH ~214-283) never re-runs the preflight.
- No unique index stops a second opening of the same holding.

**Scenarios:**
- Two tills save-and-post the same godown + lot + bucket at the same moment. Both pass, and the holding is opened twice: double stock and a double INVENTORY debit.
- `/post` passes on version N. A draft save commits version N+1 (missing batch, another item's unit, an already-opened holding). Post then writes N+1 unchecked.

**Fix:** run the preflight inside the post transaction, after `lockHoldingItems`, against the locked rows.

## 3. Base quantities are taken from the client unchecked (HIGH)

**Where:**
- SVC ~1332 writes `baseQty` exactly as sent (the comment says so).
- `freeBaseQty` is left out of the write when it isn't sent (~1340).
- The preflight checks `svi_qty`, not `svi_base_qty`.
- The ledger moves `svi_base_qty` and skips rows where it is 0.

**Scenarios:**
- `{qty 5, baseQty 0}` "posts" but writes no ledger row.
- `{qty 1, toBaseFactor 12, baseQty 1000}` opens 1000.
- `freeQty 2` without `freeBaseQty` posts no free goods.
- `baseUomId` is never checked against the item.

**Fix:** derive base = qty × factor on the server (or reject when it differs), and require `baseUomId` to be the item's base conversion.

## 4. Cancelling a posted opening is not refused when stock has since been sold (HIGH, policy question)

**Where:** PH `assertNegativeStockPolicy` (~2835) sets `forceBlock = … && !params.reversal`. The default policy is `COALESCE(stp_allow_negative,'ALLOW')`.

**Scenario:** open 10, sell 8, cancel the opening. Under ALLOW or WARN (the default for an item with no policy row) the cancel succeeds. The balance goes to −8, and the whole opening value is reversed in accounts.

**Question for you:** should a reversal (cancel) always refuse to go negative, whatever the item's policy? The screen's design and our earlier notes assume it does.

## 5. A line's godown and item are not checked against the document's branch or company (HIGH)

**Where:** `replaceLines` (SVC ~1279/1287) stores `godownId` and `itemId` as sent. The only checks for these (godown in branch, item in company, not a service item, active) are in the CSV import (`import.helper.ts:197`) and the item-lookup.

**Scenario:** an opening in branch A with a line in branch B's godown, or another company's item, or a service item, posts stock.

## 6. `docDate` is not checked against `accYear` (MEDIUM)

**Where:** SVC ~1065/1186 store any `docDate`. The already-opened rule keys on `sml_acc_year`.

**Scenario:** open a holding in 2026-2027, then post another opening with `accYear 2027-2028` and `docDate 2026-04-01`. The check passes, and the stock is opened twice for the same date.

## 7. Lot identity: blank batch and MRP 0 (MEDIUM)

- **Blank batch:** the preflight tests `COALESCE(svi_batch_no,'')=''` without trimming (SVC ~2172/2180), while the lot key trims. So `batchNo "  "` passes and opens the no-batch lot. Serial numbers have the same problem.
- **MRP 0:** the preflight refuses only a NULL MRP (~2176). The item-lookup seeds `mrp: seed?.mrp ?? 0` (`opening-stock-lookup.service.ts:246`), so an MRP-tracked item with no price row opens an "MRP 0" lot.

## 8. The item-lookup `alreadyOpened` flag is per item, for any year and any godown (MEDIUM)

**Where:** `opening-stock-lookup.service.ts:134-140` filters only company, branch, item and OPENING. So it says "already opened" when the per-holding rule would allow it: another bucket, another godown, or another year.

**Fix:** filter it the same way the preflight's `opened` CTE does.

## 9. Company, branch and user come from the request body, not the login token (HIGH, applies to every route)

**Where:**
- The controller passes `dto.companyId`, `dto.branchId` and `dto.userId` straight through.
- `AccessTokenGuard` (`src/modules/auth/guards/access-token.guard.ts:46`) knows the token's company and branch, but these routes don't use them.
- `resolveActor` (`src/common/utils/module-shared.utils.ts:269`) prefers the body's userId over the token's.

**Scenario:** any logged-in user can read, post or cancel another company's opening by sending that company's ids. The body's `userId` becomes `postedBy` and `avh_user_id`.

This is a system-wide decision. Raising it here because a post writes accounts.

## 10. A refno sent by the client becomes the stock and accounts voucher number unchecked (MEDIUM)

**Where:** `stock-voucher-numbering.helper.ts:200-206` uses a supplied slno/refno as given. `posting/stock-accounts-posting.service.ts:210` sets `presetRefno = svh_refno`. `acc_voucher_header` has no unique index on `avh_voucher_refno`.

**Scenario:** a device sends `refno "OPN0042"` while the series is at 40. The server-numbered 42 then hits `ux_svh_refno` (409). Because the series update rolls back, it may keep drawing 42 and failing (please confirm against `allocateVoucherNumber`). Two accounts vouchers can also share a printed number.

---

**Priority order suggested:** 1, 2, 3, 9, 5, then 4 (after deciding the policy), then 6, 7, 8, 10.
