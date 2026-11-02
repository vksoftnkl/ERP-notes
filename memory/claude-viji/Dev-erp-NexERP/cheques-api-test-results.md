---
name: cheques-api-test-results
description: Live test of all 10 /api/v1/cheques routes on 2026-09-17 — what works and the three defects found
metadata:
  type: project
---

All 10 routes of `/api/v1/cheques` exist on the live API (192.168.0.106:3011) and were
exercised end-to-end on 2026-09-17 against Acme Foods (company `019c8ea6…`, branch
`019c8ea7…`, year 2026-2027) with a self-seeded fixture: one posted receipt carrying four
1000.00 cheque tenders, party Deepan `01a0aa0b…`, cheque tender `019fef38…`, bank ledgers
`019ef849…` (Kvb Current A/c) and `019ef870…` (Test Bank Account). `accounts.acc_pdc_register`
was EMPTY before the run, so any cheque testing needs its own fixture.

~60 cases. Reads, deposit, clear, bounce (1st), re-present and replace all behave to
`share/cheques/plan-backend-received-cheques.md`. Postings verified in the DB: clear =
DR bank / CR Cheques In Hand with `av_recon_date` on the bank leg only; bounce = ONE voucher
(`ChqBnc`) holding all five legs with `apd_bounce_voucher_id` = `apd_charge_voucher_id`
(per §4.4 step 8) plus a JOURNAL bill `BNC/<no>`; re-present auto-FIFO correctly settles that
bounce-charge bill first.

Three defects, in severity order:

1. **A second bounce is a 500.** `deposit → bounce → re-present → bounce` returns
   `Internal server error` with no body. Reproduced on two separate cheques; the transaction
   rolls back cleanly (row stays DEPOSITED, no orphan voucher). Strong candidate: §4.4 step 5
   reverses every `abj` row with this `abj_cheque_id` and only skips rows that ARE reversals —
   it does not skip rows already REVERSED, so it re-reverses the original receipt allocation and
   hits the unique index `ux_abj_reversal (abj_reversal_of_id, abj_acc_year)`. It should reverse
   the re-issue allocation instead.

2. **A replacement cheque is a dead end.** §4.6 says the new row gets `apd_tender_id` NULL, but
   `/return` and `/replace` resolve Cheques In Hand FROM the tender row and refuse a null one
   with "Cheque is not linked to its receipt … written by hand or by an import that skipped it."
   So a replacement can never be returned, cancelled or replaced again. The ledger is already
   knowable from `apd_voucher_id`, which the replacement row does carry.

3. **`/deposit-slip` returns 200 with an empty slip** for a slipNo/date/bank that matches nothing,
   instead of 404 — it will happily print a blank slip. Minor, same shape: `/list` accepts an
   unknown `status` value and silently returns 0 rows rather than 400.

See [[received-cheques-design]] for the design these were tested against, and
[[api-house-conventions]].

## Payload field names that are NOT what the columns suggest (2026-09-19)

Cost me a full retest cycle of phantom "failures". The columns are `apd_*`;
the DTOs are not.

| route | what it takes |
|---|---|
| `/cheques/bounce` | `reason`, `bankCharge`, `partyCharge` — **not** `bounceReason` / `bounceCharges` |
| `/cheques/return` | `action` (`RETURNED` \| `CANCELLED`) + `reason` — **no** `returnDate` |
| `/cheques/replace` | a nested `newCheque: {instrumentNo, instrumentDate, amount, bankName, bankBranch, drawerName}` — **not** flat fields |
| `/receipts/duplicate-check` | `partyId, companyId, accYear, voucherDate, amount` — **not** the `avh*` names |
| `/receipts/adjacent` | `voucherId, companyId, branchId, accYear, direction` — **not** `avh*` |
| `/receipts/update-header` | `editRemark` is **required** |

A wrong payload is a 400 that is easy to miss if the caller does not check the
response — and for `/cheques/bounce` the cheque then silently stays DEPOSITED,
which reads exactly like a product bug.

**When seeding a receipt against a bill, pick a `abl_dr_cr = 'DR'` bill.** A
CR-side bill is correctly refused ("sits on the CR side of this party's
account"), and a test fixture that grabs the largest pending bill will
eventually grab one.

## Full-coverage retest, 2026-09-19

All **22** receipt + cheque routes exercised in one pass
(`scratchpad/full_api.py` pattern: every route hit, coverage asserted against
the swagger at the end so a route cannot be silently skipped). Plus all five
client suites. See [[represented-cheque-guard-hole]] for what that retest was
guarding.
