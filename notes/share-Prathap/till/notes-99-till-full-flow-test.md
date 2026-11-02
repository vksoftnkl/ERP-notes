# Notes 99: full till flow test on the live box (2026-10-09)

**STATUS:** done (server migration 20261009170000, checked live 2026-10-09). The client now uses all of it: §1 slipCheckRequired, §4 PICK ROW grid 154, §5 dropdown 69, §7 the RETENDER tile. Checking §7 turned up notes 100.

**Order:** independent of notes 97 and 98. Read it after them.

## What was run

Everything below ran as VKPOS on device VKPOS1, counter ZT1, safe ZTS, at Acme / Coimbatore. The flow went through the real Qt screens wherever a screen exists. Bills went straight through `/bills/create` + `/bills/post` (copied from the sales e2e harness body).

| Step | Through | Result |
|---|---|---|
| Open session ZT1-261009-10, float 2,000 issued, counted 1,990 | S1 Open Till | OPEN; the −10 posted at open (EXPENSE) |
| 6 bills: cash 590 · UPI 1,180 · card 2,360 · cash 500 + UPI 680 · card 1,475 · UPI 354 + card 826 | API | all stamped with the session; UPI rows PENDING, expected tomorrow |
| Drop 500, paid in 100, pickup (refused: VKPOS is the cashier) | S3 Cash Movement | tdr00085, tpi00024; the pickup was refused correctly |
| Slip check while COUNTING, as the cashier | API | 403 TILL_BLIND_CLOSE, which is correct |
| End billing; count cash 2,680 exact + card 3,835 (826 short) | S4 End Shift | RECOUNT_REQUIRED (see §1) |
| Recount: cash 2,680 + card 4,661 | S4 | ACCEPTED; closed, Z 10 |
| Slip check after close, one row ticked "no slip" | Slip check dialog | SLIP_CHECK event written |
| UPI tender configured (see §0); column map saved and tested | Statement format dialog | 4 lines, 0 errors, 1 payout |
| Import 3 UPI lines + 1 line with no bill | Import dialog | 3 MATCHED by REF, 1 UNMATCHED |
| Post payout | Settlement 278 | tst00013 (see below); rows SETTLED |
| Resolve the line with no bill as INCOME | Settlement 278 | tst00014: Dr Till Variance Suspense / Cr Balances Written Back 250 |
| Expense voucher: post, cancel, draft paid by UPI, reload | Expense 277 | exp00031; exp00032 → rev00468; draft kept UPI on reload |
| Session ZT1-261009-11: float reason SAFE_SHORT, short count, reopen, recount with 40 loose coin, close with no floatLeft | S1 + S4 | reason filed SAFE_SHORT; recount accepted; float left 0 by default |

### Books for session ZT1-261009-10 (all correct)

| Ledger | Amount |
|---|---|
| Cash In Hand | 0.00 |
| Store Safe Cash | +1,180 (2,000 out, then 500 drop and 2,680 close back in) |
| Cash Short & Excess | +10 |
| UPI Clearing | 2,214, then 0 after the TSet |
| Sales | 6,750 |
| Tax | 1,215 |

### TSet tst00013

| Side | Ledger | Amount |
|---|---|---|
| Dr | Kvb Current A/c | 2,452.36 |
| Dr | Bank Charges | 9.86 |
| Dr | GST on Payment Charges (Pending) | 1.78 |
| Cr | UPI Clearing | 2,214.00 |
| Cr | Till Variance Suspense | 250.00 |

## 0 · Test-box change: the UPI tender (asked for by the user)

UPI (`019fbbd2-…`) was changed as follows:
- **Ledger:** from Kvb Current A/c to **UPI Clearing**.
- **Settlement ledger:** **Kvb Current A/c**.
- **Settlement days:** 1.
- **Statement format:** a KVB UPI CSV map.

This follows plan §5.5: the clearing ledger is credited when the payout posts. Leaving the bank as the tender's own ledger would have counted the money twice.

The record before the change is saved in `till/preview/upi_tender_before_2026-10-09.json`.

**Asked:** `test/sales/bill-tender-http.e2e-spec.ts:284` asserts that a UPI bill debits `TENDER_LEDGER.UPI` (= Kvb). It will now see UPI Clearing. Please read the ledger from the tender instead, or tell me to revert the tender.

## 1 · A card gap makes the cashier recount the cash

The cash was counted exact and the card batch total was 826 short. The answer was RECOUNT_REQUIRED, because `count()` puts every counted tender into `outOfTolerance`. The cashier is told to count again but never told what is wrong, so they recount cash that was already right. With max_recounts = 1, the next answer is SENT_FOR_APPROVAL, and that session would sit in PENDING_APPROVAL until phase 3.

Plan §4.2 says a non-cash gap goes to the supervisor's slip check, then re-tender, then NONCASH_VARIANCE.

**Asked:** should SLIPS lines be left out of the recount decision, so a card gap goes straight to the slip check or the approval and never back to the cashier?

The client wording now says "count the cash and the slip totals again" until this is decided.

## 2 · `td_mdr_amt` stays 0 after a TSet post

The three settled UPI rows have `td_settle_amount` = gross, but `td_mdr_amt` = 0.00. Their lines carried fees of 4.72 + 0.85, 2.72 + 0.49 and 1.42 + 0.26.

Plan §5.5 says each matched row gets `td_mdr_amt` = the line's fee + tax.

## 3 · Grid 141 (Unexplained) shows the time in UTC

The statement line was "09/10/2026 15:10:00" (IST, and the format reads IST). The server stored `2026-10-09T09:40:00Z`, which is correct. Grid 141 shows **09:40 AM**.

**Asked:** please format `asl_txn_on` at Asia/Kolkata in the grid SQL. Grid 140 may need the same check for its dates.

## 4 · Grid 140 (Not received) cannot show some rows

- **Rows without an expected date never appear:** `td_expected_settle_on + grace < current_date` drops rows whose expected date is NULL. The 3 PENDING `UPI123` rows of 2026-09-24 are such rows: PENDING, with no settle ledger and no expected date. They can never appear or be written off.
- **Refunds and chargebacks can't be picked:** the Match tab's "Pick row" and Resolve LINKED use this grid as their picker. The grid holds only DR PENDING / PARTIAL rows, so a REFUND line (which needs a money-out row) or a CHARGEBACK line (which needs a SETTLED row) can never find its row.

**Asked:** one of the following:
- a picker grid of candidate rows per line kind, or
- tokens on grid 140 for `idr_cr` and `isettle_status`.

The client now passes `itender_id` (the payout's tender) to the picker.

## 5 · Dropdown 66 lists expense ledgers for the INCOME resolution

Dropdown 66 (TILL REASON LEDGERS) lists `led_ledger_type IN ('EXPENSE', 'INCOME')`. Picking an expense ledger for Resolve INCOME gets 422 SETTLEMENT_LEDGER_INVALID.

**Asked:** an INCOME-only ledger dropdown. I can't add dropdowns.

## 6 · `GET /expenses/get`: `derived.safeName` is null on a saved voucher

Validate fills `safeName`; `get` doesn't. The client now falls back to the SAFE tender row's `ledgerName`, so it shows "Store Safe Cash" (the ledger) rather than "ZT S1 test safe" (the safe).

## 7 · No till event for a re-tender

`TillEventCode` has no RETENDER code, so the cockpit's RE-TENDERS tile cannot count. It now shows "—".

**Asked:** log one from bill-retender, or tell me where to read re-tenders of a session from.

## 8 · TENDER_SUSPENSE posts to "Till Variance Suspense"

Unmatched statement money in tst00013 went to **Till Variance Suspense**. Please confirm the TENDER_SUSPENSE role is meant to share that ledger with till variance suspense, rather than having a "Tender Suspense" ledger of its own.

## Not tested (no data)

- **Write-off (Not received):** no row is past its expected date. See §4.
- **SENT_FOR_APPROVAL / PENDING_APPROVAL:** deliberately avoided, because with no approval gate the session would block ZT1.
- **The rest of phase 3:** approvals, Day Close, cash office, force close, reopen and X read are not on the server.
