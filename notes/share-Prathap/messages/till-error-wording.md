# Till error messages, short and simple (REV 2, 2026-10-15)

**STATUS:** approved by the user 2026-10-15 and CODED in Qt (src/core/api/api_error_text.cpp, one table; change wording here first, then there). Offline test tests/till/err_text.cpp 11/11. Grey line style: theme template v27 (QLabel#msgDetail).
REV 2: every message cut short (user: "many users don't know formal English").

## Rules for every message

- **Title:** 2–4 words.
- **Line 1:** what happened, one short line.
- **Line 2:** "Do:" then one short step.
- Everyday words only: no "variance", "session", "approval authority", "device".
  - till session → **till**
  - variance / difference → **cash short or extra**
  - approve → **OK**
  - device / computer → **this PC**
  - float → **opening cash**
- On bill / return / receipt / payment screens, add: **"Bill not lost."**
- Below that, a **small grey line** with the server's code and its own text, for us (not for the user).

```
┌────────────────────────────────────────┐
│ ⚠  Cannot take payment                  │
│                                        │
│ Your till is waiting for supervisor OK. │
│ Do: Call supervisor.                    │
│ Bill not lost.                          │
│ ────────────────────────────────────── │
│ 409 · TILL_SESSION_NOT_OPEN             │ ← small grey
│ Session VIJI PC-261015-01 is PENDING_…  │
│                               [ OK ]   │
└────────────────────────────────────────┘
```

**No wording yet for a code:** title "Cannot save", line "Something went wrong.", "Do: Send a photo of this to support."

---

## 1 · Billing counter (cashier)

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `TILL_SESSION_REQUIRED` | Cannot take payment | Till is not open. | Open Till first. |
| `TILL_SESSION_NOT_OPEN` · PENDING_APPROVAL | Cannot take payment | Your till is waiting for supervisor OK. | Call supervisor. |
| `TILL_SESSION_NOT_OPEN` · COUNTING | Cannot take payment | Your till is being counted. | Finish End Shift, then open a new till. |
| `TILL_SESSION_NOT_OPEN` · SUSPENDED | Cannot take payment | Your till is paused. | Resume it in Open Till. |
| `TILL_SESSION_NOT_OPEN` · CLOSED | Cannot take payment | Your till is closed. | Open a new till. |
| `TILL_SESSION_NOT_OPEN` · other | Cannot take payment | Your till is not open. | Call supervisor. |
| `TILL_SESSION_CLOSED` | Cannot take payment | Your till is closed. | Open a new till. |
| `TILL_SESSION_DAY_ENDED` | Cannot take payment | Your till is from yesterday. | End Shift, then open a new till. |
| `TILL_SESSION_WRONG_DEVICE` | Cannot take payment | Your till is open on another PC. | Bill on that PC. |
| `TILL_SESSION_NOT_YOURS` | Cannot take payment | This till is another cashier's. | Log in as yourself. |
| `TILL_SESSION_CLOSED_USE_RETURN` | Cannot change bill | This bill's till is closed. | Make a Sale Return. |
| `TILL_CASH_BLOCKED` | Too much cash | Drawer is over the cash limit. | Drop cash in safe, or call supervisor. |
| `TILL_HOLDS_OPEN` | Cannot end shift | You have held bills. | Finish or cancel them first. |
| `TILL_DEVICE_UNSYNCED` | Cannot end shift | Some bills are not sent yet. | Check network, try again. |
| `TILL_CASH_LEDGER_DIRECT` | Cannot save | Till cash goes only through the till. | Use Cash Movement. |
| `SALES_DAY_CLOSED` | Day closed | Today is closed for billing. | Call manager. |

## 2 · Opening the till

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `TILL_OPERATOR_BUSY` | Cannot open till | Your old till is not finished. | Finish it, or call supervisor. |
| `TILL_COUNTER_BUSY` | Cannot open till | This counter is in use. | Use another counter, or call supervisor. |
| `TILL_COUNTER_NOT_YOURS` | Cannot open till | This counter is not for this PC. | Pick another counter. |
| `TILL_NO_FREE_COUNTER` | Cannot open till | No counter is free. | Call manager. |
| `TILL_COUNTER_INACTIVE` | Cannot open till | This counter is switched off. | Call manager. |
| `TILL_COUNTER_NOT_FOUND` | Cannot open till | Wrong counter for this branch. | Pick another counter. |
| `TILL_DEVICE_UNKNOWN` | Cannot open till | This PC is not set up as a till. | Call manager. |
| `TILL_DEVICE_BLOCKED` | Cannot open till | This PC is blocked. | Call manager. |
| `TILL_DEVICE_REQUIRED` | Cannot open till | This PC is not set up as a till. | Call manager. |
| `TILL_DAY_NOT_OPEN` | Cannot open till | Today is not opened yet. | Call manager. |
| `TILL_DAY_CLOSING` | Cannot open till | Today is being closed. | Call manager. |
| `TILL_FLOAT_INVALID` | Wrong amount | Opening cash amount is wrong. | Check the amount. |

## 3 · Counting / End Shift

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `TILL_NOT_COUNTED` | Count first | Drawer is not counted. | Count the drawer. |
| `TILL_COUNT_INVALID` | Wrong count | Something in the count is wrong. | Check the count. |
| `TILL_RECOUNT_LIMIT` | Cannot count again | Too many counts. | Call supervisor. |
| `TILL_APPROVAL_REQUIRED` | Cash short or extra | Cash does not match. | Call supervisor. |
| `TILL_BLIND_CLOSE` | Not allowed | Cashier cannot see this before closing. | Just count the cash. |

## 4 · Cash movement (drop, pickup, top-up, paid-in, exchange)

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `TILL_MOVEMENT_INVALID` | Cannot save | Something in this entry is wrong. | Check the entry. |
| `TILL_MOVEMENT_AT_COUNTER` | Not allowed here | Do this at the cashier's counter. | Go to the counter. |
| `TILL_SUPERVISOR_SELF` | Need another person | Supervisor and cashier must be different. | Pick another supervisor. |
| `TILL_SUPERVISOR_RIGHT` | Not allowed | This supervisor has no right for this. | Pick another supervisor. |
| `TILL_SUPERVISOR_NO_PIN` | No PIN | This supervisor has no PIN. | Pick another supervisor. |
| `TILL_SUPERVISOR_PIN_REQUIRED` | Enter PIN | Supervisor PIN is empty. | Supervisor types PIN. |
| `TILL_SUPERVISOR_PIN` | Wrong PIN | Supervisor PIN is wrong. | Type again. |
| `TILL_WITNESS_PIN_REQUIRED` | Enter PIN | Witness PIN is empty. | Witness types PIN. |
| `TILL_WITNESS_PIN` | Wrong PIN | Witness PIN is wrong. | Type again. |
| `TILL_WITNESS_NO_PIN` | No PIN | This witness has no PIN. | Pick another witness. |
| `TILL_NO_DROP_SAFE` | No safe here | This counter has no drop safe. | Give the cash to supervisor (Pickup). |
| `TILL_SAFE_MISSING` | No safe | This branch has no safe set up. | Call manager. |
| `TILL_MOVEMENT_NOT_FOUND` | Not found | This entry is not there now. | Refresh and try again. |
| `TILL_MOVEMENT_NOT_VOIDABLE` | Cannot cancel | This entry cannot be cancelled. | Make a new entry to correct it. |
| `TILL_MOVEMENT_SESSION_CLOSED` | Cannot cancel | That till is already counted. | Make a new entry in today's till. |
| `TILL_REASON_INVALID` | Reason needed | Reason is missing or not complete. | Pick a reason and fill it. |

## 5 · Supervisor screens

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `TILL_APPROVER_SELF` | Need another person | You cannot OK your own till. | Ask another supervisor. |
| `TILL_APPROVER_NO_GRANT` | Not allowed | You have no right to OK this. | Ask another supervisor. |
| `TILL_APPROVER_LIMIT` | Amount too big | This is above your limit. | Ask a senior. |
| `TILL_APPROVER_PIN` | Wrong PIN | PIN is empty or wrong. | Type again. |
| `TILL_RECEIVER_SELF` | Need another person | You were this till's cashier. | Ask another supervisor. |
| `TILL_RECEIVER_NO_PIN` | No PIN | You have no PIN. | Call manager. |
| `TILL_RECEIVER_PIN_REQUIRED` | Enter PIN | Your PIN is empty. | Type your PIN. |
| `TILL_RECEIVER_PIN` | Wrong PIN | Your PIN is wrong. | Type again. |
| `TILL_RECEIPT_NOT_AWAITING` | Already done | Someone already did this. | Refresh. |
| `TILL_VARIANCE_NOT_PENDING` | Already done | Someone already did this. | Refresh. |
| `TILL_CASH_ALREADY_APPROVED` | Already done | Cash is already in the safe. | Nothing to do. |
| `TILL_SESSION_NOT_OPEN` (Cockpit) | Not now | This till is not in the right state. | Refresh. |
| `TILL_SESSION_NOT_FOUND` | Not found | This till is not there. | Refresh. |
| `TILL_REOPEN_BLOCKED` | Cannot reopen | This till cannot be opened again. | Correct it in today's till. |

## 6 · Manager: business day

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `TILL_DAY_CLOSE_BLOCKED` | Cannot close day | Some work is not finished. | Finish the red items first. |
| `TILL_FORCE_COUNT_REQUIRED` | Count first | A till is still open. | Count that till's cash. |
| `TILL_DAY_CLOSED` | Day closed | This day is already closed. | Reopen the day first. |
| `TILL_DAY_NOT_CLOSED` | Not closed | This day is still open. | Refresh. |
| `TILL_DAY_LATER_CLOSED` | Cannot reopen | A later day is closed. | Call head office. |

## 7 · Setup problems

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `TILL_LEDGER_UNMAPPED` | Setup missing | An account is not set up. | Call manager. |
| `TILL_YEAR_NOT_SET_UP` | Setup missing | This year is not set up. | Call manager. |
| `TILL_MASTER_IN_USE` | In use | This is used by a till now. | Close that till first. |
| `TILL_EVENT_INVALID` | Something went wrong | This PC sent a wrong request. | Send a photo of this to support. |

## 8 · Safe and bank remittance (notes 118, added 2026-10-15)

**STATUS of this section:** approved 2026-10-15 and coded (api_error_text.cpp).

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `TILL_SAFE_CHANGED` | Count again | Safe cash changed while you counted. | Count the safe again. |
| `TILL_REMIT_OVER_BALANCE` | Amount too big | Safe does not have this much cash. | Send less, or count the safe first. |
| `TILL_REMIT_NOT_IN_TRANSIT` | Already done | This cash is not on the way any more. | Refresh. |
| `TILL_APPROVER_SECOND` | Need second person | This needs two people's PIN. | Another person types their PIN. |

---

## Questions

1. Is **"Call supervisor"** / **"Call manager"** right for your stores?
2. Should we keep the words **Drop**, **Pickup** and **Open Till**? Your cashiers already see them on the buttons.
