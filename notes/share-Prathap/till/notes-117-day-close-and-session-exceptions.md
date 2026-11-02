# Notes 117: Day close, and the session exceptions (force close, reopen, X read) (2026-10-14)

**STATUS:** server DONE, verified live 2026-10-14 on ZT1 test sessions (scratchpad n117.py / n117b.py). close-check lists the RED items and close answers 409 while RED; the real day was NOT closed. force-close by a supervisor → CLOSED, forced, Z 31; X read 200 on a live session and 409 on a closed one; reopen → COUNTING with the hand-over VOIDED (not deleted) and the Z kept; re-close kept Z 31; reopen refused once the cash is approved. Print providers till.session.* and till.day.* exist. One wording point: the cashier refused on force-close reads "does not decide its own variance" (it should say the cashier cannot force-close their own session). Qt in progress.

The user asked what is still pending for the day end, because many buttons are greyed out. On the Qt side these buttons wait for these routes:

| Screen | Button | Route asked |
|---|---|---|
| Business Day (274) | F10 Close day · Close with exception… | `GET days/close-check`, `POST days/close` |
| Business Day (274) | Cancel closing (rename: **Reopen day**) | `POST days/reopen` |
| Business Day (274) | the "Server close-check" row | `GET days/close-check` |
| Session Cockpit | Force close… | `POST sessions/force-close` |
| Session Cockpit | Reopen | `POST sessions/reopen` |
| Session Cockpit | X read | `POST sessions/x-report` |
| Cockpit / Business Day | Print Z · Print day X | the print data (§5) |

All routes take `companyId · branchId` (+ `accYear`) and an id, as the rest of `/till/`. Rights: Business Day (274) for the day routes, and Till Sessions (273) OVERRIDE for the session ones. Approvals go through `assertApprover` with the seeded rule, as `decide-variance` does: the approver's PIN, never the requester unless `tar_allow_self`.

## 1 · `GET /till/days/close-check?companyId&branchId&tbdId`

The day's checklist as the server sees it, so a client can never close a day the server would not. Each item is `{ code, tone, count, detail, sessionIds[] }`, where tone is RED (blocks), AMBER (allowed, shown) or GREEN. The response also carries `canClose` (no RED) and `canCloseWithException` (RED only of the kinds an exception may cover).

| code | tone | when |
|---|---|---|
| `SESSIONS_LIVE` | RED | a session of the day is OPEN, SUSPENDED or COUNTING |
| `SESSIONS_PENDING_APPROVAL` | RED | PENDING_APPROVAL |
| `CASH_TO_APPROVE` | RED | `till.handover_receipt` REQUIRED and a hand-over / drop of the day is AWAITING (notes 113/114) |
| `VARIANCE_UNDECIDED` | RED | a CLOSE- or RECEIVE-stage cash variance still PENDING (e.g. one sent on by approve-cash) |
| `SAFE_NOT_COUNTED` | RED | `till.day_close_safe_count` and no SAFE count of the day (notes 118). Until 118 is live, leave it out. |
| `SLIP_GAPS` | AMBER | a non-cash (SLIPS) gap still PENDING: reported, not enforced (notes 99 §1) |
| `HELD_BILLS` | AMBER | `txn_hold` rows of the branch still parked |
| `SAFE_OVER_INSURED` | AMBER | a safe over `tsf_insured_limit` (remit, notes 118) |
| `MOVEMENTS_UNPOSTED` | RED | any movement / variance of the day without its voucher (should be impossible; a guard) |

**Exception scope.** Only `SESSIONS_LIVE` (a cashier who left) and `SAFE_NOT_COUNTED` may be closed over. The others have a real decision missing, so they keep blocking. The Qt screen reads the codes and keeps its own jump buttons (open the Cockpit, Till Sessions, the safe).

## 2 · `POST /till/days/close` `{ companyId, branchId, tbdId, notes?, exception?: { reasonId, approverPin } }`

- Recomputes the close-check in the transaction. Any RED → 409 `TILL_DAY_CLOSE_BLOCKED`, carrying the items, unless `exception` is given and every RED is of an exception kind.
- **The exception** is the DAY_CLOSE_EXCEPTION rule (STORE_MANAGER, EITHER), with the approver not the caller. It writes a `till_approval` row → `tbd_close_approval_id`, `tbd_exception_count`, and an event naming the items closed over.
- **Live sessions under an exception** are force-closed first (§4) with reason `DAY_CLOSE`: they cannot be left open on a closed day.
- **Freezes the day's totals:** `tbd_session_count`, `tbd_bill_count`, `tbd_net_sales`, `tbd_cash_expected`, `tbd_cash_counted`, `tbd_cash_variance`, `tbd_noncash_variance`, and the safe figures once 118 exists. Then `tbd_status` CLOSED, `tbd_closed_on/by`, and the **store Z** `tbd_z_no` from a per-branch series. Logs `DAY_CLOSE`.
- **After close:**
  - `sessions/open` for that business date → 409 `TILL_DAY_NOT_OPEN`.
  - Bills, returns, receipts, re-tenders and cash cancels into it → `SALES_DAY_CLOSED`. TILL_DESIGN §7.4 already says SALES_DAY_CLOSED reads `till_business_day`.
  - Under `till.day_auto_open` the next business day opens on the next session as today.
- **No CLOSING state** from the client: the close is one transaction. Keep CLOSING in the CHECK for a later two-step close if you need it. The Qt "Cancel closing" button becomes Reopen day.

## 3 · `POST /till/days/reopen` `{ companyId, branchId, tbdId, reasonId, approverPin, notes }`

- The DAY_REOPEN rule (AREA_MANAGER, EITHER).
- Back to OPEN, `tbd_reopen_count + 1`, the frozen totals cleared, the Z number **kept**. A re-close keeps the same Z, and the print says REOPENED ×n.
- Logs `DAY_REOPEN`.
- **Refuse** when a later business day of the branch is already CLOSED. Reopening day N under a closed N+1 would break the safe's day chain.

## 4 · `POST /till/sessions/force-close` `{ key, reasonId, approverPin, lines[], notes }`

For a session whose cashier has gone (OPEN, SUSPENDED or COUNTING): the drawer is counted by the supervisor instead.

- The FORCE_CLOSE rule (STORE_MANAGER, EITHER). The caller is the supervisor; the cashier cannot.
- **One transaction:** end billing (holds per `till.close_with_holds`), then the caller's count (`tct_counted_by` = caller, `tss_count_place` CASH_OFFICE), then the usual outcome:
  - **within tolerance:** closed; the hand-over is done by the caller. Under notes 116 the caller may approve its cash.
  - **beyond tolerance:** PENDING_APPROVAL, decided by `decide-variance` as now.
- Sets `tss_force_closed` and `tss_force_reason_id`; logs `SESSION_FORCE_CLOSE` with the reason.
- The force-close reasons need a category (FORCE_CLOSE) seeded: e.g. "Cashier left", "Shift over time", "Device failure", "Day close".

## 5 · `POST /till/sessions/reopen` `{ key, reasonId, approverPin, notes }`

For a session closed by mistake (counted wrong, a bill posted to the wrong session).

- The SESSION_REOPEN rule (STORE_MANAGER, EITHER).
- **Refused when:**
  - its business day is CLOSED: reopen the day first;
  - its cash is already approved into the safe (`tcm_receipt_status` RECEIVED, notes 114). Undoing a receipt is a safe matter, so refuse with a clear code.
- **Reverses** the close's own vouchers: the hand-over TDrp and any CLOSE-stage TVar, by reversal, never by delete (`tvr_status` REVERSED). The session returns to COUNTING with the close count cleared (as RECOUNT does since notes 112), and `tss_reopen_count + 1`. The Z number is kept.
- Logs `SESSION_REOPEN`.

## 6 · `POST /till/sessions/x-report` `{ key }`

A mid-shift reading of a live session for the supervisor.

- Returns the `sessions/expected` payload, plus `xNo` (a per-session count) and the time.
- Logs `X_REPORT`.
- Refused to the session's cashier while blind close applies (TILL_BLIND_CLOSE), as `/expected` is.

## 7 · The print data for Z and X

Print Z (session), Print day X / store Z (day) and the X read need a print document type each in the printing module: `TILL_SESSION_Z`, `TILL_SESSION_X` and `TILL_DAY_Z`. Their data is the session / day payload, tenders and movements, as sale bills feed their print. Then I lay them out in Print Designer (menu 62) and assign them (248). Please tell me the doc-type codes and field names you expose. Bump `tss_print_count` / `tbd_print_count` on each print.

## Not in this note

The safe count, safe balance and remittance are notes 118. HO review (`till_session_review`) is phase 6.

## Follow-up (2026-10-14): what the print providers need

The three layouts are built on the providers and published: `TillSessionZ80`, `TillSessionX80` and `TillDayZ80` on the 80 mm roll, generated by `Shared/Prathap/print/till_z_x_templates.py`. Four small gaps:

1. **Timestamps arrive as 24-hour text** (`'DD-MM-YYYY HH:mm'`), which the `date` transform cannot parse. The templates rebuild AM/PM from the characters, and the user wants AM/PM everywhere. Please send ISO timestamps (or add ready `*_text` fields in `hh:mm AM` form).
2. **No variance treatment on `till.session.header`** (EXPENSE / RECOVER / SUSPENSE / WITHIN_TOLERANCE of the CLOSE-stage cash variance); only `variance_status`. The Z should say how the gap was booked.
3. **The day's cash expected includes live sessions that have no count yet.** Today: expected 40,480, counted 38,550, variance +70. The 2,000 gap is ZT1-261014-15, still open. Either sum expected over counted sessions only, or add `cash_expected_counted` beside it, so the day X does not look 1,930 short.
4. **The safe figures on `till.day.header`** follow notes 118.
