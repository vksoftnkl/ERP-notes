---
name: receipts-api-test-results
description: Live test of all 8 /api/v1/receipts routes on 2026-09-17 — what works and the five gaps found
metadata:
  type: project
---

All 8 routes of `/api/v1/receipts` exist and were exercised live on 2026-09-17, same fixture
company as [[cheques-api-test-results]] (Acme Foods `019c8ea6…` / branch `019c8ea7…` /
2026-2027, party Deepan `01a0aa0b…`, cash tender `019fbbd0…`). ~59 cases.

`/create` and `/post` are the strongest surfaces in the codebase: every rule in
`share/receipt/plan-backend-receipt.md` is enforced with a message that names the figure
(tender-type mismatch, cash received−change, cheque needs an instrument date, role vs DR/CR,
the party as its own other-line, ONE salesman, R3 no-edit-when-POSTED, and the balance proof
"Received 2500 + deductions 500 + credits 0 = 3000, but allocated … Out by −999"). Verified in
the DB: legs DR Cash 2500 / DR TDS Receivable 500 / CR party 3000; onAccount raises a CR
ADVANCE bill that `open-items` then returns as a credit; applying a credit writes NO voucher
legs (bill-to-bill within the party, per [[party-adjustment-design]]); cancel reverses and
reopens the bill (4720 → 7720) and refuses a second time.

**ALL CLOSED 2026-09-18 (second round).** Prathap did the whole handover:
`/receipts/regularise-pdc` now requires `companyId` and reports rows CHANGED of
rows considered; two collectors are refused on `update-header` with `/create`'s
own sentence; `party-context` 404s like `open-items`; **`/receipts/adjacent`**
and **`/receipts/duplicate-check`** are new; `open-items` bills carry
`usrRefno`, `billProfit`, `billProfitPreTax`; `party-context.summary` carries
`totalBalance / totalOutstanding / totalCredits / chequesOutstanding`; the three
stale header totals are backfilled. **The PDC-cancel hole (notes 26) is fixed** —
cancelling a receipt with a post-dated cheque now reverses BOTH ("2 voucher(s)
reversed") and nets every ledger to 0.00. Client side shipped in `adeaaa3`.
**Still open: the 9 legacy orphans (5,400 against Deepan) predating the fix —
no route reaches them, Prathap's call.** Note `avh_against_voucher_id` means
BOTH "PDC child of" and "reversal of", so any orphan query must exclude
reversals or it reports false positives.

**RE-VERIFIED LIVE 2026-09-18 — three of the five are CLOSED.** Gap 4 is closed
(`accounts.v_bill_tcs` exists and `open-items` bills now carry `tcsAmount` +
`tcsPending`, so the bill-wise TCS column is a CLIENT job now). Gap 3 is closed
(`update-header` 404s on a DRAFT). Gaps **1 and 2 are still open exactly as
written**, and gap 5 too. Separately, [[voucher-balanced-check-vacuous]] is
CLOSED: `tr_av_refresh_totals` → `fn_avh_recompute_totals` is deployed on
`acc_vouchers` + all four year partitions and derives both header totals from
the legs; only three receipts posted in the ten minutes before it went on
(rct00049/50/52) still carry stale stamped totals, and one
`fn_avh_recompute_totals` sweep clears them.

Also new since the test: **`/receipts/amend` and `/receipts/delete` exist and
have NOT been exercised.** Handover written as
`share/receipt/HANDOVER_receipt_backend.md` + `notes.txt` **(25)**, which also
carries the six backend asks from the 3.0 comparison
([[receipt-screen-revision]]): `/receipts/adjacent`, per-row deposit account
(the `edit_ledger` override), `/receipts/duplicate-check`, `billProfit`, the
bill's own `usrRefno`, and a party total balance.

Five gaps, none of them in `/create` or `/post`:

1. **`/regularise-pdc` has no company, branch or tenant scope at all** — body is `{asOf?}`
   only. Any authenticated user regularises post-dated settlements across EVERY company in the
   database. It is the only mutating route on the module without the house four keys
   ([[api-house-conventions]]). Effect is idempotent (verified: pending stayed 4990 over two
   runs) but it reports "1 bill(s) regularised" on the second run too — it counts rows
   considered, not rows changed.
2. **`/update-header` accepts two `avhEmployeeId`s** and writes both to the array column, while
   `/create` refuses the identical payload with "A receipt is collected by ONE person."
3. **`/update-header` edits a DRAFT** despite being documented as "on a posted receipt". Low
   stakes since `/create` upserts a draft anyway.
4. **`open-items` bills[] is missing `tcsAmount` and `tcsPending`** — §4.1 of the plan requires
   both from `accounts.v_bill_tcs`. Everything else in §4.1 matches exactly. Blocks the
   bill-wise TCS column in [[tcs-billwise-design]].
5. **`party-context` returns 200 with empty lists for a party that does not exist**, where
   `open-items` 404s on the same id.

Note `onDate` on `open-items` is NOT an as-of filter — per §4.1 it only ages the PPD slab, so
the bill list not moving when you back-date it is correct, not a bug.
