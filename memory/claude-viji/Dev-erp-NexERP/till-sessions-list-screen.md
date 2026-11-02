---
name: till-sessions-list-screen
description: S6 Till Sessions list (menu 273) BUILT 2026-10-09 — TxnMainView over grid 144, Ctrl+H grid 145; cockpit (S5) next
metadata:
  type: project
---
**Files:** src/modules/sales/till/sessions/txn_doc_types_till.{h,cpp} (TxnDocTypes::tillSessions()), fm.reg(273, new TxnMainView(...)). Grids registered by share/grids/till_sessions_registration.py (--test / --apply [--update] NAME): **144 MAIN LIST - TILL SESSIONS** (icompany_id, ibranch_id, ifrom_date, ito_date, istatus, ivariance), **145 TILL SESSION EVENTS** (itss_id).
**Facts learnt:** the grid runner IGNORES grid_sort_column — it sorts only on a client sort_by, so the SQL ends with ORDER BY. TxnMainView row actions with no iconPath get **delete.svg** (a bin) — app-wide wart (temp credit Receive too); set iconPath. Frozen totals on till_session are written at close; before it the grid reads bills live from sales.sale_bill and expected/counted/variance from the last CLOSE-stage cash till_variance row.
**Actions:** Enter/Ctrl+Enter = info "cockpit is next"; F5 Cash movement (OPEN/SUSPENDED → S3 in Pickup mode); F6 Count / close (COUNTING → S4); Ctrl+H = grid 145 timeline. Status/treatment pills via badgeOverrides (CLOSED success, not Locked-dim).
**Checked live 10-09:** 7 ZT1 sessions, newest first; OPEN row → movement on/count off; COUNTING row → movement off/count on; history dialog shows SESSION_OPEN + MOVEMENT_POSTED. Harness SNAP_SRC=snap_list. Pickup by a real supervisor NOT tested — no second user with 273 OVERRIDE on the box.
**Not built:** totals footer row, counter/cashier filters (search covers), Print Z, Export.
**Next:** S5 Session Cockpit (Enter on the list).

**Live cash (2026-10-13, user: "how many sessions live and what's the cash in till"):** TillLiveCash (src/modules/sales/till/sessions/till_live_cash.*) = grid 144 once per live status (OPEN/SUSPENDED/COUNTING/PENDING_APPROVAL, date tokens blank = any day) + GET /sessions/expected per session (server's formula only, D4 — never compute expected client-side; 403 → "cash hidden"). 273 is now TillSessionsView (subclass of TxnMainView): toolbar line "● N live session(s) · cash in tills ₹ X" + live figure written into Cash exp. via new NexApiTableView/NexJsonModel::setRowValue. 274 Business Day: same line under the title, "Live now (N, any day)" in CASH IN TILLS, RED check "Drawers still out from an earlier day". Test SNAP_SRC=live_cash (LIVE_SESSION_NO, LIVE_CASH) 7/7 read-only.
