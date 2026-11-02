---
name: till-management-design
description: "Till/cashier session architecture REV 1 (2026-10-07) — share/till/TILL_DESIGN.md + 47_till_management.sql + 48/49 — LIVE since 2026-10-08 (Prathap ran them); 3.0 defects, D1–D12 decisions, open Qs"
metadata:
  node_type: memory
  type: project
  originSessionId: 437b9134-899f-4f46-94f2-a058376f179c
  modified: 2026-10-07T04:30:00.061Z
---

Written 2026-10-07 on the user's ask: "refer 3.0 session management … compare with gofrugal … work for a chain store like approvals … no need to carry 3.0 legacy … design completely new". This un-parks the session half of [[six-txn-controls]] (session first, then void).

**Files:** `/home/vk/Shared/Prathap/till/TILL_DESIGN.md` (771 lines: 3.0 analysis, GoFrugal matrix, architecture, accounting, routes, screens S1–S11, build order) and `/home/vk/Shared/Prathap/47_till_management.sql` (run after 46).

**Validated, not applied:** a throwaway PG 18 cluster (port 55432, TCP only — the scratchpad path is too long for a unix socket) loaded with the live DDL + data of the touched tables. It applies, re-runs idempotently, and 13 probes pass. Two seed bugs were caught that way: `acc_ledger_role.alr_group` is NOT NULL with a CHECK (use SHARED), and `app_setting_def` has no STRING type (use TEXT). Nothing is on 192.168.0.106, and no NestJS or Qt code exists yet.

**Key decisions (why):**
- Tables go in schema `accounts` with prefix `till_`, because `fn_create_year_partitions` only sweeps public/sales/accounts.
- No TILL_CASH ledger role. The till cash ledger IS the branch CASH tender's `tnd_ledger_id`; the session is the sub-ledger via `avh_session_id`.
- Blind close by default; the server alone computes expected; every count attempt is kept.
- Approvals = rule × authority × approval (generic exception gate, tiered by amount). This is NOT a voucher approve step, so the [[api-house-conventions]] no-approve rule still holds.
- Money moves are booked in the session where the cash physically moves. Cancelling a cash bill from a closed session is refused (use a return).
- One writer per row: the store writes everything except `till_session_review`, which only the cloud writes. Sync is push-only and upserts by PK.

**3.0 facts worth remembering:**
- `pos_sess_pays` (cash in/out) never posts to the books.
- "Approve All" overwrites counted with expected.
- A close DELETEs and re-INSERTs the counts.
- Approval is the global AdminUser flag.

**User decisions 2026-10-07 (§14):**
1. Till masters are authored on the STORE server.
2. Shortage recovery is booked to the loan ledger only, no payroll link.
3. PNG mockups drawn on request (NOT yet reviewed by the user — once approved they are the visual spec, [[mockup-fidelity]]): `till/till_open_session_ui_mockup.png` (S1), `till_end_shift_ui_mockup.png` (S4), `till_session_cockpit_ui_mockup.png` (S5), `till_day_close_ui_mockup.png` (S8). Helpers are in `till/till_mock.py` on top of `sales/mock_lib.py`. One worked story runs through them: C01-261007-02 / RAVI K, expected 13,720, counted 13,640, −80, and the session's till legs net to 0.

**Master mockups (2026-10-07, user said "only mockup", so there is no plan file):**
- `till/till_masters_ui_mockups.py` draws counter, safe, reason, denomination, user_employee, approval_rules and approval_authority PNGs.
- **till_operator DROPPED 2026-10-07** (see [[till-operator-to-user-employee]]): there is no Till tab; picture 5 is now User + Employee fields.
- Counter, Safe, Reasons and Denominations are standard CRUD.
- Approval Rules is an event matrix with branch > company > shipped layers.
- Approval Authority is a per-approver grants list with a "who can approve?" test and a gap check.
- None of these have been reviewed by the user yet.

**Remaining screens drawn 2026-10-07** by `till/till_screens_ui_mockups.py`: S2 strip + F11 menu, S3 cash movement, S6 sessions list, S7 approval gate + inbox, S9 cash office, S11 HO cash position. **Every §8 screen now has a mockup** (none reviewed yet). The shared helpers live in `till_mock.py`. The figures reconcile across all of them; S8's cash-in-tills was corrected to 93,930 expected / 93,350 counted, and the safe closes at 2,08,350.

**Receipt / Payment / Expense in the session (2026-10-07):**
- The user wants bill-wise receipts and payments to be part of the till session, plus a new "expense by tender" form.
- The user confirmed that bill-wise pay/receipt already put all their money in `acc_tender_detail`.
- The design is a separate plan: `till/plan-till-receipt-payment-expense.md`, with `48_till_receipt_payment_expense.sql` (validated on the throwaway PG, not applied).
- The session's expected cash = tender rows by session + till movements (receipts DR +, payments CR −, expenses CR −).
- Only CASH rows of payments/expenses reduce the drawer. Bank/UPI-out rows are information-only.
- PAID_OUT is RETIRED. An expense goes on the new ExpV voucher; a supplier or staff member gets a bill-wise Payment.
- ExpV uses no new table: voucher header + legs + tender rows (EXPENSE CR) + GSTR2 rows like PurA. Routes are `/api/v1/expenses/*`.
- New approval rule CASH_PAYMENT (shipped 1,000). The PAID_OUT rule is renamed EXPENSE (500).
- A 40A(3) row (10,000, WARN) is seeded in statutory_limits.
- A Jrl/Con touching the till cash ledger on a till device is refused.
- Mockups: `till_expense_voucher_ui_mockup.png` and `till_money_docs_in_session_ui_mockup.png` (F11 rev 2, Payment on a till, cockpit rev 2).

**Next:** build phase 1 (§13) once Prathap applies 47 and builds the core session routes.

Also noted: `AppSession::sessionId()` is a random uuid per app run today, and the DC sends the device id as `sdcCounterId`.

**Review rev 2 (2026-10-07):** `till/plan-till-review-rev2.md` overrides all the till plans. Four review passes; SQL 47/48/49 corrected in place (they were unapplied) and re-validated.

Real bugs it fixed:
- 48's EXPENSE tender rows were refused by live `ck_td_src_doc_type`.
- The closed-session LINKED reversal broke the books. Now a pairing via `asl_tvr_id`, no voucher.
- Card gaps settled to the wrong ledger. Now `tvr_rows`.
- Double final counts were allowed.
- The approval table had no event/role lists.
- `acc_year` could disagree with the business date.

Added: `till.single_operator` (single-shop self-approval), the sync check at end billing, cut-off ends billing, BANK import source, `GST_ON_CHARGES_PENDING` role.

Re-run contract: each file re-runs on its own, never an earlier file after a later one.

Open: the IT Act 2025 renumbering of 269ST / 40A(3) needs confirming against the Act text; TCS 206C(1H) live rows are stale (still to raise with Prathap).
