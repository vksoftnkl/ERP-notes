---
name: till-reports-mockup
description: (+ tabs 5–7 grids 175–180 built 10-17) Till Reports screen 2026-10-17 — mockups approved, grids 159–169 live, Qt screen built (offscreen 7/7); menu 288 (notes-133 done 10-17)
metadata:
  type: project
---
Mockups `Shared/Prathap/till/till_report_{variance,exceptions,approvals,safe_book}_ui_mockup.png` (script till_reports_ui_mockups.py) — user approved 2026-10-17 ("yes do it").

**Grids** (Shared/Prathap/grids/till_reports_registration.py, `--test` read-only): 159 CASHIER VARIANCE, 160 CASHIER SESSIONS, 161 EXCEPTIONS, 162 EXCEPTION HOURS, 163 EXCEPTION EVENTS, 164 APPROVERS, 165 APPROVER DECISIONS, 166 UNDECIDED, 167 SAFE DAYS, 168 SAFE DAY LINES, 169 PEOPLE (pickers). Reuses 157 remittances, 133/136 counter/safe pickers, dd 61 branches. Each grid gets ONLY its own tokens (an extra key could become an unbound $N).
Facts baked in: PIN_FAIL belongs to tev_user_id (whose PIN was wrong), every other event to the session's cashier; pickups/top-ups = MOVEMENT_POSTED payload kind; RECOUNT decision = tap_status REJECTED + tap_rule_snapshot->>'decision'='RECOUNT'; own session = tap_decided_by = tss_operator_id (tap_self_allowed is the RULE's flag, not "did self"); safe opening = last SAFE till_count before the day + flows after it.

**Grid 158 fixed 2026-10-17:** the safe-ledger tender branch now has `td_src_doc_type <> 'VOUCHER'` (server flowsOf rule) — since notes 128 till vouchers carry tender rows and the Cash Office safe book double-counted them.

**Qt:** src/modules/sales/till/report/till_reports_entry.{h,cpp,ui} (TillReportsEntry, NexDialog, QTableWidget + local PillDelegate with sizeHint, totals row, hour heat strip, Enter/double-click → TillSessionCockpit, Ctrl+H → TxnHistoryDialog::showTrail(tillSessions().history), Print = QPrintPreviewDialog of the tab, Export = CSV). Harness `SNAP_SRC=snap_reports tests/till/snap.sh` 7/7 read-only.
**REGISTERED:** menu 288 (Till > Till Reports, pos 2.50, verbs VIEW/PRINT/EXPORT), `fm.reg<TillReportsEntry>(288)` in core/navigation/form_sales_menu.h. No user had a 288 grant on 10-17 — grant it in User Rights.

**2026-10-17 later — tabs 5–7 BUILT (user: "write the pending reports", no mockup — same layout as tabs 1–4):** grids 175 CASHIER PERFORMANCE / 176 PERFORMANCE SESSIONS, 177 LEDGER CHECK (§5.7 invariant: CLOSED session's legs on the branch CASH tender ledger, av_session_id, Dr + = tss_float_left − carried float; idiff) / 178 LEDGER LEGS (itss_id), 179 TENDER RECON (card/UPI/wallet/cheque/bank per session; settled = td_settle_amount GROSS; ipending; HAVING not a wrapper — runner wants top-level FROM a real table) / 180 TENDER ROWS (itss_id, itender_type_id). Script Shared/Prathap/grids/till_reports2_registration.py (imports till_reports_registration; keep _BIND, R.bind is patched). Qt: tabs Cashier performance / Ledger check / Tender reconciliation in TillReportsEntry; currentSessionRow falls back to the MAIN row (ledger/tender rows are sessions). snap_reports 13/13 (ZT2 books 1,500 LIVE; ZT1 ×3 OK). Box has 0 sale bills and 0 non-cash tenders since the reset → performance bill figures and tender recon untested with data. Scan-to-tender time is not recorded anywhere (lines written at save). Uncommitted.

**2026-10-21 tabs 5–7 tested WITH DATA:** tests/till/flow_live.py (FLOW_DIR=scratchpad) opened ZT1-261021-01, 4 bills (CASH 590 / UPI 1180 / CARD 2360 / mixed 500+208+472), counted 3090 + card slips 2832, closed Z 4 — no variance. snap_reports 14/14: performance VKPOS 4 bills 5,310; ledger check OK (TFlt 2000 + 2 Bil − TDrp 3090 = 0); tender recon Card 2,832 / UPI 1,388 PENDING. Tab-7 check now compares the totals row with psql (non-cash td rows of the range) instead of asserting empty. The ZT1-261017 ledger checks fall out of the 7-day window after 10-23 — set REPORT_FROM or rewrite them then.

**10-21 later:** ledger-check test no longer hard-codes ZT2's 1,500 — it reads ZT2's legs on the TFlt ledger from the DB (ZT2 now 1,000 after a Change). snap_reports 14/14.
