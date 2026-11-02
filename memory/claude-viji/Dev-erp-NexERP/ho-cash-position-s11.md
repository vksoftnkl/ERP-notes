---
name: ho-cash-position-s11
description: S11 Cash Position BUILT + registered menu 289 (10-17) — grids 170–174, Qt 13/13, HO review popup LIVE 15/15 (real save 10-17)
metadata:
  type: project
---
**Mockups (approved 10-17):** Shared/Prathap/till/till_ho_cash_position_ui_mockups.py → till_ho_cash_position_rev2_ui_mockup.png + till_ho_review_ui_mockup.png (old 10-07 mockup left as is). Story = Till Reports' ([[till-reports-mockup]]).

**Grids** (Shared/Prathap/grids/till_cash_position_registration.py, `--test [--show]`, `--apply [--update]`; update sends columns + replace_columns — the server retires old rows, count only `NOT grid_column_is_deleted`): 170 TILL CASH POSITION (per branch with a counter/safe, as at END of iday: day status + tbd_id, old day open, live/over hours, awaiting = receipt AWAITING movements, safe book = grid 158 rule at t1, insured, oldest safe count, in transit REMIT unconfirmed + age, CLOSE cash var day/MTD, approvals, last till_event, last review, flags text), 171 LIVE sessions (screen calls /till/sessions/expected each, today only), 172 EVENTS, 173 CASHIERS MTD, 174 REVIEWS. Tokens icompany_id, ibranch_id, iday.

**PIN_FAIL person:** change/movement witness PIN_FAIL has tev_user_id = CALLER, the wrong-PIN person in payload.witnessBy → grids use COALESCE(witnessBy, tev_user_id); grids 161–163 updated 10-17.

**Qt:** src/modules/sales/till/report/till_cash_position_entry.{h,ui,cpp} (NexDialog). Shared helpers moved out of Till Reports into till_report_table.{h,cpp} (namespace TillReportTable; + kBarRole fill bar). Drill = child dialogs on the stack with ctx.menuId + applyMenuGovernance(): TillBusinessDayEntry::openOn(branch, tbdId, accYear) (answers guarded by branch), TillSessionsView::setBranch (TxnMainView::setScopeBranch), TillReportsEntry::setScope. Theme block share/theme/block-till-cash-position.qss → template **v28 LIVE** 10-17 (md5 8d0e43ad…). Test tests/till/snap_cashpos.cpp (SNAP_QSS=<file> previews a template) 13/13; snap_reports still 7/7.

**Notes 134 DONE (10-17):** menu **289** (fm.reg in form_sales_menu.h), POST /till/reviews/create (+ GET /till/reviews/list, HO_REVIEW event). Popup = till_ho_review_dialog.{h,cpp} (code-built QDialog parented to the screen, so its theme block styles it): facts, target = day (tbd_id) or a session (grid 150 for the date + live grid 171), 7 outcomes, note unless OK, journal for WRITTEN_OFF/SUSPENSE_CLEARED from grid 117 Jrl (Acme has NO journals yet → empty list), history grid 174. Test tests/till/snap_hoReview.cpp: 12/12 without HR_SAVE; HR_SAVE=1 writes a REAL QUERY review → 15/15 live 10-17 (row + HO_REVIEW event checked; one test review now sits on Coimbatore's 17-10 day). VKPOS granted 289 V/C/E by the user 10-17 (first User Admin attempt didn't save — check user_menus after a grant). Claude cannot write a rights script (classifier blocks even writing it) — the user grants via User Admin. Still open: optional one-call /till/cash-position.
