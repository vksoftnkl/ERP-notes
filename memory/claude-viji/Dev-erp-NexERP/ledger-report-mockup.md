---
name: ledger-report-mockup
description: Ledger Statement report (2026-09-25) in share/report/ — mockup PNG + backend plan with 7 NEW routes; the plan found two live server defects (Dr opening read as Cr; closing/carry-forward drop cancelled originals)
metadata:
  type: project
---
`/home/vk/Shared/Prathap/report/`: `ledger_report_ui_mockup.py/.png`, `plan-backend-ledger-statement.md`, `plan-react-ledger-statement.md` (2026-09-25; lives HERE, not in react-plans/), README. No Qt code yet (user: "dont write the code"). Qt plan not written.

**Routes:** 7 NEW GETs under `/api/v1/reports/ledger-statement/` — ledgers, header, vouchers, voucher-legs, daily, monthly, export. See [[new-module-own-urls]]. Balance = acc_opening_balance year set + legs of POSTED+CANCELLED headers; a cancel = CANCELLED original (legs kept) + POSTED mirror (`avh_reversal_voucher_id` / `avh_against_voucher_id`), verified live; "Include cancelled" off hides the PAIR. Decisions L1-L7 open (menu row under 137 + fold 144 Ledger Monthly Summary, All-branches opening = Σ every set, etc.).

**Defects found (plan §3, for Prathap):**
1. `accounts.fn_ledger_book_balance` tests `op_dr_cr = 'DR'` but the column is char(1) 'D'/'C' → every Dr opening counts as Cr (verified: 89 D → -89). Feeds the notes-47 reconcile guard, so probably IS D2 of [[bill-ledger-reconcile]].
2. opening-balance.service closingByLedger + carry-forward.service count POSTED only → drop the cancelled original, keep its mirror → closing wrong, and carry-forward writes it into next year.
Credit limit DOES exist (found 2026-10-09): cus_credit_amt_limit, cus_credit_bill_limit, cus_credit_allowed; suppliers have none.

**DONE BY PRATHAP 2026-09-25 (verified 18:30):**
- both defects fixed: migration 20260925130000 ('D' sign), and closing/carry-forward now use IN ('POSTED','CANCELLED');
- the backend is BUILT: `src/modules/reports/ledger-statement/`, all 7 routes, `ix_avh_reversal`, menu 258 under 137.

What remains is the client (React plan written; Qt not planned).
