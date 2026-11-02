# Notes 114: the supervisor approves a closed session's cash, with no separate Cash Office screen (2026-10-14)

**STATUS:** DONE + verified live 2026-10-14. Server (migration 20261014160000): `sessions/approve-cash` + receiptState / cashAwaiting / receivedBy on sessions/get; menu 286 hidden. API check 16/16 (n114.py: ZT1-261014-04 exact, -05 within tolerance, -06 decided in the step by a temporary 100 grant, -07 over the grant refused then sent on and left PENDING for another approver). Qt: Approve Cash popup on the Cockpit, the Cash column + Variance filter TO_APPROVE on grid 144, the Business Day row now jumps to 273, End Shift text, Cash Office unregistered. Live screen test 12/12 (ZT1-261014-08, -09).

## Why

Notes 113 works: Cash Office (286) was verified live 7/7 on bag `tdr00203`. But when the user saw it, they judged it too much for a normal shop. It brings in bags, a blind count by denomination, a receiver PIN, and a third person to decide any difference. It also puts the same session in two places: Till Sessions for approvals, and Cash Office for receiving.

The user's words: *"just an approve is enough — a supervisor verified the session."*

So the control stays the same: cash is not in the safe until a supervisor confirms it. Only the way the supervisor confirms it changes.

- **Before:** receive a bag on a separate screen, with a blind count.
- **Now:** **one Approve on the closed session**, on Till Sessions / the Session Cockpit (273). The supervisor sees the amount handed over.

## What stays from notes 113 (no change)

- The setting `till.handover_receipt` = REQUIRED | NONE. Our company stays REQUIRED.
- CLOSE_HANDOVER and DROP post to `TILL_CASH_AWAITING`, not the safe.
- The receive posting (TRcv: Dr Safe / Cr Cash awaiting), and a difference as a RECEIVE-stage `till_variance` on the closing session.
- days/get `awaitingReceipt{…}`, and day close refusing while cash of the date is unreceived.
- Keep `movements/awaiting-receipt` and `movements/receive` if you like. The Qt client will stop calling them. Please **hide menu 286** (`menu_is_active` false, or whatever you use), so it no longer shows in menus or user rights.

## The user's decisions

1. **Who approves:** anyone with the approval right on Till Sessions (273) except the session's cashier. Under `till.single_operator` the cashier may approve their own session (the owner of a one-person shop).
2. **A difference:**
   - **Within tolerance:** posts at once, as `receive` does now.
   - **Beyond tolerance:** the **same supervisor decides it in the same step** (treatment + reason) **when their Till Approval Authority grant covers the amount**. That means the usual `assertApprover` checks for CASH_VARIANCE: level, `taa_max_amount`, PIN.
   - **When their grant does not cover it:** it goes SENT_FOR_APPROVAL, and another approver decides it with `decide-variance` as now.

   Today `receive` + `decideVariance(tcmId)` refuse the receiver as decider (`requesterId = tcmReceivedBy`). For this route the requester is the **cashier**, not the supervisor.

## Asked

### 1 · Route: approve a session's cash

`POST /till/sessions/approve-cash`

```
{ companyId, branchId, accYear, sessionId,
  approverPin,            // required, unless till.single_operator and caller = cashier
  receivedAmount?,        // omit = received exactly what was handed over
  treatment?, reasonId?, recoverLedgerId?, notes? }   // only for a gap beyond tolerance
```

- **What it covers:** every movement of the session still awaiting receipt: the CLOSE_HANDOVER and any DROPs. One approval per session, with one total. If `receivedAmount` is given, the difference is against that total.
- **Who:** the caller, not the cashier (except under single_operator), with 273 OVERRIDE (or the right you used for decide-variance), and a correct till PIN. A wrong PIN is journalled PIN_FAIL, as now.
- **Outcome:**
  - No gap: `APPROVED`.
  - Gap within tolerance: `WITHIN_TOLERANCE`.
  - Gap beyond tolerance:
    - `treatment` sent and the caller's grant covers the amount: `DECIDED`. It posts as `decide-variance` APPROVE would.
    - `treatment` sent but the grant does not cover it: refuse with the existing APPROVER_LIMIT / APPROVER_NO_GRANT 403, so the screen can say "send it on instead".
    - No `treatment`: `SENT_FOR_APPROVAL`. The RECEIVE variance is left PENDING for `decide-variance`.
- **Response:** amount handed over, amount received, difference, outcome, variance id if any, voucher no.
- **Idempotent:** a second call on an approved session gets a 409 with a clear code (e.g. `TILL_CASH_ALREADY_APPROVED`).

### 2 · Show the state on the session

So the list and Cockpit can show it without guessing, please add a **cash receipt state** for each closed session:

| State | Meaning |
|---|---|
| `AWAITING` | Some movement is awaiting receipt |
| `APPROVED` | All received, no gap |
| `DIFFERENCE` | Received with a gap: decided, or pending a decision |
| `NONE` | Setting NONE, or closed before notes 113 |

Put it on **sessions/get**, together with `cashAwaiting` (the total handed over and not yet received), `receivedBy` and `receivedOn`.

Grid 144 (Till Sessions list) needs the same state as a column. Tell me the column name if you add it in SQL, or I'll add it to the grid query myself from `till_cash_movement`.

### 3 · Business Day

Leave `awaitingReceipt` as it is. The Qt row will read "N sessions not yet approved" and jump to Till Sessions (273), filtered.

## Qt side, once the route exists

- **Till Sessions (273):** an amber **CASH TO APPROVE** pill on closed sessions in AWAITING, and a filter for them.
- **Cockpit:** an **Approve Cash** button on a closed session in AWAITING. It opens a small popup:
  - "VIGNESH handed over ₹500.00 (close ₹500.00)", plus any drops;
  - the amount received, prefilled with that amount;
  - PIN;
  - when the amount differs beyond tolerance, the treatment + reason, as the existing decide popup.
- **Business Day:** the row text and the jump change to 273.
- **End Shift:** the close text becomes "Hand the cash to your supervisor to approve."
- **Cash Office (286):** the Qt screen stays in the code, unregistered from the menu.
