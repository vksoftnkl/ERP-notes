---
name: void-bill-design
description: Void bill/line REV 2 (no approval, sale bill only); notes-136 asked 10-17; Qt after Prathap
metadata:
  type: project
---
User 10-17 "yes go ahead with void bill design" (SIX_TXN_CONTROLS §7 unparked). Written: Shared/Prathap/till/plan-void-bill.md.

Facts found 10-17: /bills/cancel needs only menu 12 Cancel, no supervisor, no print check, no BILL_VOID event (assertCancellable only refuses counted-drawer cash). VOID_BILL/VOID_LINE approval rules shipped ALWAYS/SUPERVISOR/COUNTER; reason categories exist but NO reason rows; till_event BILL_VOID/LINE_VOID unused. print_log partitions stop at 2026_2027 (no 2027_2028). Box has 0 sale bills (reset).

Design: void = cancel's unwind + accounts.till_void evidence row (tvd_was_printed from print_log SUCCESS+PRINT at void time, minutes after post, session void no, approval) + BILL_VOID event; status stays CANCELLED (VOID pill from the row); POST /bills/void only while the bill's session is OPEN/SUSPENDED (cancel then 409 SALES_USE_VOID); 428 gate reuse; printed ⇒ supervisor always. LINE_VOID journalled by the client via /till/events/batch, no approval. DC void later.

**User 10-17: agreed all four §7 answers.** Mockups (till_void_ui_mockups.py → till_void_bill / till_void_list / till_report_voids _ui_mockup.png): popup not-printed vs printed (red band with print_log times, supervisor whatever the rule), Ctrl+F8 key states (Ctrl+F8 checked free on sale bill), bill list VOID / VOID · PRINTED pills (status stays CANCELLED), Cockpit VOIDS / AFTER PRINT / LINE VOIDS tiles + timeline, Till Reports 8th tab Voids. Next after approval: notes to Prathap (§3) → Qt.

**REV 2 (user 10-17): "just we want to see, no need to approve — it will complicate the usage."** Overrides Q2: no void asks a supervisor; the printed-bill override is dropped; shipped VOID_BILL rule → NEVER (the gate stays so a company can switch it on in 276 and the 428 popup works). Reason still required. First written as a separate rev2 file; then the user said "alter the notes" → folded into plan-void-bill.md itself as REV 2 (rev2 file deleted; REV 1 copy in that session's scratchpad plan-void-bill.rev1.md). Mockups redrawn: no approval strip, "Approved by" → "Voided by"; REV 1 script copy in that session's scratchpad.
**Why:** the counter must stay simple; visibility (Cockpit, Z, Till Reports Voids tab, Cash Position flag) is the control.
**How to apply:** don't reintroduce approval steps on counter actions without the user asking; prefer record + show.
**Scope (user 10-17, "record only sales txn" → confirmed SALE BILL ONLY):** till_void is SALE_BILL only (CHECK); DC void dropped (not "later"); LINE_VOID only from the sale bill screen; till money docs keep their own void/cancel. In plan-void-bill.md REV 2.
**Notes 136 written 10-17** (Shared/Prathap/till/notes-136-void-bill.md): till_void table, POST /bills/void (409 TILL_VOID_SESSION_CLOSED, 428 only if a company enables the rule), VOID_BILL rule → NEVER, /bills/cancel 409 SALES_USE_VOID on a live-till bill, /bills/get + void / voidable / printed, events/batch accepts LINE_VOID, 6 VOID_BILL reasons, Z print fields, print_log_2027_2028. Qs: expected cash sees the reversal; loyalty/promo unwind.
