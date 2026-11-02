# Held bills, payment lines and form checks: short error messages (REV 1, 2026-10-15)

**STATUS:** approved by the user 2026-10-15 and CODED in Qt (src/core/api/api_error_text.cpp). Defaults kept for the two questions: "Send a photo of this to support"; "Fill in all the needed boxes" (no field names). A code with no row and no [APP] prefix (most SALES_ / voucher codes) shows the server sentence under a plain title, the code in grey — never "Something went wrong".
The codes are from notes 119 (Prathap, 10-15: held bills `HOLD_…`, payment lines `TENDER_LINE_…`, form checks `VALIDATION_…`, `INTERNAL_ERROR`).

Same rules as `till-error-wording.md`:
- A title of 2–4 words, one short line, then "Do:" and one step.
- Everyday words.
- The server's text stays small and grey underneath.

Most of these codes mean **the app sent something wrong**, not that the user did. Those all share one message, the **[APP]** row below, so the user isn't asked to fix what they can't.

## 0 · The shared rows

| Name | Title | Line 1 | Do |
|---|---|---|---|
| **[APP]** | Something went wrong | The app sent a wrong request. | Send a photo of this to support. |
| **[SETUP]** | Setup missing | Something is not set up. | Call manager. |
| `INTERNAL_ERROR` | Server problem | The server had a problem. | Try again. If it repeats, send a photo to support. |

## 1 · Held bills (`HOLD_…`)

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `HOLD_NOT_FOUND` | Not found | This held bill is not there now. | Refresh the held bills list. |
| `HOLD_LOCKED_ELSEWHERE` | Open on another PC | This held bill is open on another PC. | Finish it there, or wait a few minutes. |
| `HOLD_NOT_RESUMABLE` | Cannot open | This held bill is already billed or cancelled. | Refresh the held bills list. |
| `HOLD_CLOSED` | Already done | This held bill is already billed or cancelled. | Refresh the held bills list. |
| `HOLD_NOT_LEASED_HERE` | Open on another PC | This held bill is open on another PC. | Finish it there. |
| `HOLD_DUPLICATE` / `HOLD_NO_DUPLICATE` / `HOLD_SLNO_DUPLICATE` | Try again | Two counters saved at the same time. | Save again. |
| `HOLD_DEVICE_BLOCKED` | Cannot hold | This PC is blocked. | Call manager. |
| `HOLD_DEVICE_NOT_FOUND` / `HOLD_DEVICE_REQUIRED` | Cannot hold | This PC is not set up as a till. | Call manager. |
| `HOLD_SESSION_NOT_FOUND` | Cannot hold | Your till is not open. | Open Till first. |
| `HOLD_AMOUNT_NEGATIVE` | Wrong amount | An amount is below zero. | Check the amounts. |
| every other `HOLD_…` | **[APP]** | | |

## 2 · Payment lines (`TENDER_LINE_…`, `TENDER_REF_…`)

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `TENDER_LINE_CHANGE_EXCEEDS_RECEIVED` | Wrong change | Change is more than the cash received. | Check the cash received. |
| `TENDER_LINE_TOTAL_MISMATCH` | Amounts don't match | Payments do not add up to the bill. | Check the payment amounts. |
| `TENDER_LINE_CARD_LAST4_INVALID` | Card number | Card last 4 digits are wrong. | Type the last 4 digits again. |
| `TENDER_LINE_INSTRUMENT_DATE_REQUIRED` | Date needed | Cheque or card date is missing. | Fill in the date. |
| `TENDER_LINE_REFERENCE_INVALID` / `TENDER_REF_REQUIRED` | Reference needed | Payment reference is missing or wrong. | Fill in the reference number. |
| `TENDER_REF_DUPLICATE` | Already used | This payment reference is already used. | Check the reference number. |
| `TENDER_LINE_TENDER_NOT_FOUND` / `TENDER_LINE_TENDER_TYPE_NOT_FOUND` / `TENDER_LINE_LEDGER_NOT_FOUND` | **[SETUP]** | | |
| every other `TENDER_LINE_…` | **[APP]** | | |

## 3 · Form checks (`VALIDATION_…`)

A form check names a field (`errors[].field`, e.g. `sbSessionId`, `lines.0.qty`). The field's internal name is never shown to the user; the grey line keeps it.

| Code | Title | Line 1 | Do |
|---|---|---|---|
| `VALIDATION_IS_NOT_EMPTY` | Fill it in | Something needed is empty. | Fill in all the needed boxes. |
| `VALIDATION_MIN` / `VALIDATION_IS_POSITIVE` | Wrong amount | An amount is too small. | Check the amounts. |
| `VALIDATION_MAX` / `VALIDATION_MAX_LENGTH` | Too long | A value is too big or too long. | Make it shorter or smaller. |
| every other `VALIDATION_…` (`IS_UUID`, `WHITELIST`, `IS_ENUM`, `FAILED` …) | **[APP]** | | |

## Questions

1. Is **"Send a photo of this to support"** right for the **[APP]** row? Or should it name someone, like "Call VK Software"?
2. Section 3: should a missing box only say "Fill in all the needed boxes", or should it also name the box? Naming it needs one label per field per screen in Qt, so it's a bigger job.
