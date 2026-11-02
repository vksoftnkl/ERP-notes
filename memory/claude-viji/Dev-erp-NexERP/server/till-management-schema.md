---
name: till-management-schema
description: "Till: migrations 20261008100000-180000, src/modules/till phases 1-2, the receipt/payment/expense plan built (ExpV menu 277, 40A(3) warn, approvals report-only, 269ST deferred); TILL_DESIGN.md is not in the repo"
metadata:
  node_type: memory
  type: project
  originSessionId: d213b8ac-0bd2-425a-87f3-cb80957c3ac4
  modified: 2026-10-08T04:45:39.745Z
---

Applied to the dev DB on 2026-10-08: migration `20261008100000_till_management` = the user's pasted
`47_till_management.sql` (share file). It covers 6 masters (till_safe, till_counter, till_reason,
till_denomination, till_approval_rule, till_approval_authority) and 9 LIST-by-acc_year tables
(till_business_day → till_session → till_variance, till_count → till_count_line, till_cash_movement,
till_approval, till_event, till_session_review). It also adds acc_tender_types.ttm_close_mode /
ttm_handover, 4 ledger roles, voucher types TFlt/TDrp/TPOut/TPIn/TVar/CRem (ids 34–39 locally), 18
`till.*` settings, and seeded reasons, denominations and rules. Prisma fragments are
`prisma/accounts/Till*.prisma`.

How it differs from the share file:
- the 4 roles are added to fn_ledger_map_catalogue (see [[new-ledger-role-needs-catalogue]])
- TENDER_SUSPENSE sits under Current Assets, because 'Suspense A/c' is a Liabilities group here and the role wants Assets
- SAFE_CASH and CASH_IN_TRANSIT sit under Cash-in-Hand, so the Contra vouchers stay valid in Tally
- ensure_acc_year_partitions was restated with the till tables, because the share file relied on fn_create_year_partitions, which is not in the migration history (missing on the VPS)
- Acc_Tender_Types.sql and the exporter now carry the two new columns

**Why:** the share files (46, 47, and 48/49, which rename PAID_OUT → EXPENSE) assume objects that exist only on the dev box.

**How to apply:**
- Phase 1 was built on 2026-10-08 in `src/modules/till` (see its README). The user pasted TILL_DESIGN.md REV 1 in chat; it is still not on disk, and neither are plan-till-*.md or SIX_TXN_CONTROLS.
  - It covers masters CRUD, day auto-open, sessions (open, suspend, end billing, blind count, close), expected(), TFlt/TDrp/TVar postings and till_event.
  - Menus are pinned 271-276 (migration 20261008140000 and Menu_Master.sql). Notes 101 (2026-10-09, migration 20261009190000) split Till Masters: 275 = Till Counters, 280 Till Safes, 281 Till Reasons, 282 Denominations, each master judged on its own menu (TILL_MENU.COUNTERS/SAFES/REASONS/DENOMINATIONS); 275's grants were copied onto 280-282. Notes 102 (20261009200000) moved the four under a VIEW-only group 283 "Till Masters" (TILL_MENU.MASTERS_GROUP) beneath 271; 276 stays under 271. Every till e2e suite's TILL_MENUS must list 280-282 or creating a safe 403s. Notes 111 (2026-10-13, migration 20261013120000) renamed 276 to Till Approval Rules and added 285 Till Approval Authority (5.10 under 271), with 276's grants copied over. `approval-authorities/*` is judged on 285 (TILL_MENU.APPROVAL_AUTHORITY), and the 276 key is now TILL_MENU.APPROVAL_RULES. The TILL_MENUS lists carry 285 too.
  - **Rollout rule:** `TillSessionService.resolveForMoney` stamps or enforces the session ONLY when the token's device is bound to an active till_counter with tcn_requires_session and till.require_session holds. Every other device posts as before, because today's client sends a random session uuid per app run.
  - The rule is wired into bill post, bill retender (new rows use the CURRENT session, D7), sale return post, receipt post and payment post. Bill cancel refuses cash taken in a counted session (409 TILL_SESSION_CLOSED_USE_RETURN), and loadDayClosed also reads CLOSED till_business_day rows.
  - Phase 2 was built the same day.
    - Movements: DROP, PAID_IN and EXCHANGE (cashier, 272 EDIT); PICKUP and TOP_UP (supervisor, 273 OVERRIDE; a pickup's witness is the cashier and never the supervisor).
    - Voids: a reverseLegs mirror plus the row marked VOIDED, 273 OVERRIDE, only while the session is OPEN or SUSPENDED.
    - Cash gauge (state plus 0–4, never the figure); BLOCKED refuses bill and receipt post with 409 TILL_CASH_BLOCKED.
    - Its e2e suite boots TWO apps (tester1 = cashier, 'tester' 019e44fb… = supervisor).
  - REV 2 review (pasted 2026-10-08; it wins over REV 1). Its §1 SQL was already in 47–49 as applied. Built:
    - §2.4: tender writer sets PENDING plus expected date only for CARD/UPI/WALLET rows with SLIPS/STATEMENT close mode and a settlement ledger.
    - §2.6: end-billing outboxCount/lastClientSeq give 409 TILL_DEVICE_UNSYNCED. A late arrival is a bill made before billing stopped AND created on the server after it.
    - §2.7: after the cut-off, 409 TILL_SESSION_DAY_ENDED, judged on sb_bill_datetime.
    - §2.8: voids are allowed in COUNTING until a final count.
    - §2.9: rows voided after tss_closed_on count as live.
    - §2.13: setting label via migration 20261008150000.
    - §2.14: POST /till/movements/change.
    - REV 2 items on unbuilt modules (settlement, day close, approvals, ExpV) are listed in the module README.
  - plan-till-receipt-payment-expense.md (pasted in chat, never on disk) is BUILT, see the till README section "Receipts, payments and expenses in the session" and `accountsModule/expense/README.md`:
    - ExpV = `/api/v1/expenses/*`, menu 277 (20261008160000), grid `MAIN LIST - EXPENSE VOUCHERS` (137 on dev) + print purpose EXPENSE_VOUCHER (20261008170000). Cheques are refused on an expense (plan said register them).
    - 40A(3) = `payment/cash-payment-limit.ts` (WARN `STATUTORY_40A3`, company REFUSE row refuses), called by /payments/post, /expenses validate+post, the register's PAYMENT types; index ix_td_cash_party_day (20261008180000). It uses `resolveStatutoryLimit()` because StatutoryService is REQUEST-scoped and would drag singleton posting services with it.
    - CASH_PAYMENT / EXPENSE approvals are REPORT-ONLY (user's call 2026-10-08) via read-only `TillApprovalService.assess`: `approval` field + INFO TILL_APPROVAL_REQUIRED + `*_POSTED` event payload. Phase 3 must turn it into the gate.
    - 269ST on /receipts/post is deliberately NOT built (user's call 2026-10-08): it waits until the bill's own 269ST/PAN check (commented out 2026-10-06, commit 273a8aa4) is switched back on.
  - Notes 98 (2026-10-09), option (b): an open whose count differs from the float POSTS the OPEN-stage variance at once (TVar, treatment EXPENSE, `reasonId` FLOAT_MISMATCH or the UNKNOWN reason, which needs notes). Option (a), the close gate, was skipped: it waits on notes 97 §1.2 (PENDING_APPROVAL is a dead end), and notes 97 never reached the server. Phase 3 must RE-TREAT these rows (reverse and repost), not post them.
  - Notes 99 (2026-10-09): count() lets only the DRAWER (non-SLIPS lines) decide RECOUNT/SENT_FOR_APPROVAL; an out-of-tolerance SLIPS line makes the count final with `slipCheckRequired`, its variance PENDING, an APPROVAL_REQUESTED NONCASH_VARIANCE event (`enforced: false`), and close skips it (no TVar: SUSPENSE needs the slip check's tvr_rows). `/bills/retender` logs RETENDER (already in ck_tev_code) via `logMoneyDoc` in the money session. Notes 100: its added rows are forced onto that session too — the Qt client sends `tdSessionId: null` / `tdDeviceId: null` on every tender line, and a line's explicit null beats the scope in `TenderDetailService.insertTenderLine`; bill post survives only because `stampTenderRows` re-stamps, so any NEW writer of tender rows in a session must stamp server-side.
  - Not done: approvals (phase 3), cockpit and day close (phase 4), safe (phase 5), HO, the session grid, and stamping of bill amend and order advance. The 49 settlement services were built on 2026-10-08: see [[tender-settlement-module]]. Grids 133-136 (till masters) were made by the user in the grid designer.
  - The e2e suites (till-session-http, till-money-paths-http, till-event-single-writer) each register their OWN device. Never bind a counter to the sales harness's DEVICE_ID 'raman': every sales suite would then need a session.
- Share file 48 shipped the same day as `20261008110000_till_money_docs` (verbatim apart from no BEGIN/COMMIT and a serial setval). It:
  - removes PAID_OUT from till_cash_movement and renames PAID_OUT → EXPENSE for rules and reasons (SUPPLIER and STAFF_ADVANCE retired)
  - adds the ExpV voucher type (id 40 locally; GSTR2/INPUT, party NONE)
  - adds tvr_payment_amount / tvr_expense_amount and tss_payment_count / tss_expense_count, physically last, so the fragments list them last
  - adds the CASH_PAYMENT rule, the 40A(3) statutory row and 'EXPENSE' in ck_td_src_doc_type
  - adds partial indexes ix_td_session and ix_av_session_ledger
  TPOut was then deactivated by `20261008120000_till_retire_tpout` at the user's request (row and its unused seq row kept).
- Share file 49 shipped as `20261008130000_noncash_tender_control`. It:
  - adds acc_settlement_import / acc_settlement_line (Prisma AccSettlementImport / AccSettlementLine, LIST by acc_year, added to ensure_acc_year_partitions)
  - adds tnd_statement_format, tvr_rows, ix_td_ref and ix_td_settle_pending
  - adds NONCASH reasons, NONCASH_WRITE_OFF / SETTLEMENT_RESOLVE rules, 5 event codes and the TSet voucher type (id 41 locally)
  - adds role GST_ON_CHARGES_PENDING, catalogued as 'GST on Payment Charges (Pending)', GENERAL, Current Assets; Duties & Taxes is Liabilities here
  - adds 6 settings (tender.* and till.noncash_variance_default)
- Each later share file restates the CHECK lists (ck_trs_category, ck_tar_event, ck_tap_event, ck_tev_code). Diff the regex-extracted values against the LIVE constraint before shipping one, so nothing live is dropped. All four lists after 49 hold 18/31/31/54 values.
- Partitioned-table rules: see [[sale-bill-acc-year-partitions]].
- Never `require()` a scripts/*.js file to syntax-check it, because the export scripts run `main()` at load. Use `node --check`.
