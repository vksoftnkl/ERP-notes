# Notes 136: Void a sale bill, and journal line voids (2026-10-17)

**STATUS:** asked.

The user's problem: a cashier bills, says "the printer didn't work", takes the cash and cancels the bill. Today `/bills/cancel` allows that with only menu 12 Cancel, and it leaves **no trace that the bill had been printed**. Nobody sees it unless they go looking.

**The user's decisions (2026-10-17):**
- **No supervisor approval.** A void is **recorded and shown**: on the Session Cockpit, the session Z, Till Reports and Cash Position.
- **The sale bill only.** No other document is voided.
- **Only while the bill's till session is still live.** After that it's a sale return (counted drawer) or a cancel (no till).

Design: `till/plan-void-bill.md` (REV 2). Mockups: `till/till_void_bill_ui_mockup.png`, `till_void_list_ui_mockup.png`, `till_report_voids_ui_mockup.png`. Everything you need is below, so you don't have to read the plan.

**What already exists and is reused:**
- `BILL_VOID` / `LINE_VOID` in `ck_tev_code`;
- `VOID_BILL` / `VOID_LINE` in the approval event list;
- the reason categories `VOID_BILL` / `VOID_LINE`;
- cancel's unwind;
- `assertCancellable`;
- the 428 gate;
- `events/batch`.

## 1 · Table `accounts.till_void`, one row per voided sale bill

Partitioned by year like the other till tables, including 2027-28.

| Column | What |
|---|---|
| `tvd_id`, `tvd_company_id`, `tvd_branch_id`, `tvd_tenant_id`, `tvd_acc_year` | the usual |
| `tvd_src_doc_type`, `tvd_src_doc_id`, `tvd_src_doc_refno` | always `SALE_BILL`, the bill and its number |
| `tvd_session_id`, `tvd_device_id`, `tvd_cashier_id` | the bill's session, the device the void was keyed on, and the session's cashier |
| `tvd_voided_by`, `tvd_voided_on` | the caller, now |
| `tvd_amount`, `tvd_cash_amount` | the bill's amount, and the CASH part of its tenders |
| `tvd_reason_id`, `tvd_note` | a `VOID_BILL` reason, and a note (required when `trs_needs_note`) |
| `tvd_posted_on`, `tvd_minutes_after_post` | when the bill was posted, and whole minutes from post to void |
| **`tvd_was_printed`** | **the point of all this.** True when `public.print_log` has a row with `plg_src_doc_id` = the bill, `plg_status = 'SUCCESS'` and `plg_output_mode = 'PRINT'` **at the moment of the void.** Stored, **never recomputed** |
| `tvd_print_count`, `tvd_last_printed_on` | from the same rows |
| `tvd_print_failed` | a `FAILED` print row exists for the bill (then the "printer didn't work" claim was true) |
| `tvd_session_void_no` | this is the session's 1st, 2nd, 3rd … void |
| `tvd_approval_id` | `till_approval`, only when a company switched the rule on (§3); else null |
| `tvd_reversal_voucher_id`, `tvd_reversal_acc_year` | cancel's reversal voucher |
| sync / created / modified | the usual |

**CHECKs (no triggers):**
- `tvd_src_doc_type = 'SALE_BILL'`;
- `tvd_minutes_after_post >= 0`;
- `tvd_amount >= 0` and `tvd_cash_amount >= 0`;
- the reversal pair is both set or both null;
- one live row per bill: unique (`tvd_src_doc_type`, `tvd_src_doc_id`, `tvd_acc_year`).

**The bill's status stays `CANCELLED`.** Reports, GSTR-1 and registers already handle CANCELLED. The void row is what makes it a void; our grids show a **VOID** pill from it, and the number stays visible.

## 2 · `POST /bills/void`

**Body:** `{ sbId, sbCompanyId, sbBranchId, sbAccYear, reasonId, note?, approvalId? }`

**Steps, in one transaction:**
1. **Lock the bill.** It must be POSTED (DRAFT → 409 as cancel does). A bill that is already voided answers idempotently, with its void row.
2. **Check the session.** `sbSessionId` must be OPEN or SUSPENDED. Otherwise **409 `TILL_VOID_SESSION_CLOSED`**:
   - the session is counted or closed → "The drawer is counted: make a sale return";
   - there is no session → "Not in a till: cancel it".
3. **Check who** (403 `TILL_RIGHT_…` style). Either the session's **cashier on the session's own device** (the lockOwn rule of end-billing), or anyone with **Till Sessions (273) OVERRIDE** of their own. **And** menu 12 **Cancel**, because a void is a cancel.
4. **Check the reason.** It must be a `VOID_BILL` reason of this company or shipped, and active; a note when `trs_needs_note` (400 `TILL_NOTE_REQUIRED`).
5. **Work out the evidence:** `print_log` (§1), minutes after post, the session's void number, the cash part.
6. **Ask the gate** on `VOID_BILL` (§3). It ships NEVER, so normally nothing happens. If a company switched it on:
   - needed and no `approvalId` → **428 `TILL_APPROVAL_REQUIRED`**, the same body as Expense / Payment;
   - an `approvalId` must be APPROVED, for this bill, and not expired.
7. **Run cancel's unwind**, unchanged, with the reason's name (+ note) as the remark. That covers `assertUnwindable` (GST first), the reversal, CANCELLED, `txn_status_log` and the audit row.
8. **Insert `till_void`.**
9. **Journal `BILL_VOID`** in the bill's session. Payload: `{ sbId, refno, amount, cash, reasonId, reason, note, printed, printCount, printFailed, minutes, voidNo, approvalId }`.
10. **Answer** with the bill (`sbStatus` CANCELLED), the `till_void` row (camelCase), and `reversalVoucherRefno`.

Please add the new codes to `fixed.error_message` (notes 120) with plain words; we'll send wording if you want it.

## 3 · The `VOID_BILL` rule ships `NEVER`

Change the shipped `till_approval_rule` row for `VOID_BILL` from ALWAYS to **NEVER** (seed and live row). That way nothing is asked out of the box. A company that wants a supervisor sets it in Approval Rules (276) to ALWAYS / OVER_AMOUNT / OVER_COUNT (OVER_COUNT = voids already in this session).

There is **no printed-bill override**. A printed bill is voided like any other, and only recorded as printed.

`VOID_LINE`: leave as is; nothing asks it.

## 4 · `/bills/cancel`: refuse a live-till bill

A bill whose `sbSessionId` is OPEN or SUSPENDED → **409 `SALES_USE_VOID`**: "This bill is in an open till. Void it instead." Nothing else about cancel changes. Bills with no session cancel as today, and counted drawers are refused by `assertCancellable` as today.

## 5 · `/bills/get`: tell the screen

Please add to the `/bills/get` answer:
- **`void`**: the `till_void` row (camelCase) when the bill was voided, else null. The loaded bill shows VOID / VOID · PRINTED.
- **`voidable`**: `{ can: bool, code: null | 'TILL_VOID_SESSION_CLOSED' | 'NO_SESSION' | 'NOT_POSTED' | 'ALREADY_VOIDED', sessionStatus }`. The sale bill's Ctrl+F8 then turns on or off and says why without a second call, as `tender-context` does for re-tender.
- **`printed`**: `{ count, lastOn, failed }` from `print_log`. The Void popup shows "Printed 2 times — 10:12 AM, 10:13 AM" before the cashier commits.

## 6 · `events/batch`: accept `LINE_VOID` from a device

The sale bill screen will report lines **removed or reduced** after they were added, on a bill keyed in a till session, draft or posted, through `POST /till/events/batch`:
- `tev_event_code` = `LINE_VOID`;
- the device's live session;
- `(device, clientSeq)` as now.

**Payload:**

`{ docId (null until the first save), refno, itemId, itemName, qtyBefore, qtyAfter, rate, value, wasPrinted, lineAgeSeconds }`

where `value` is the line value removed.

**Please:**
- accept `LINE_VOID` from a device (today a device may not report server events; this one must not be on that list);
- refuse it when the session isn't the device's live one, as for the other device events.

No approval and no reason for lines.

## 7 · Seed the `VOID_BILL` reasons (shipped, company NULL)

| Code | Name | Needs note |
|---|---|---|
| CUST_LEFT | Customer left | no |
| WRONG_ITEM | Wrong item billed | no |
| PRICE_DISPUTE | Price dispute | yes |
| PRINTER_FAIL | Printer failure | yes |
| OPERATOR_ERR | Operator error | no |
| SYSTEM_ERR | System error | yes |

`VOID_LINE` has no reasons; none are asked at the counter.

## 8 · Print data

- **`till.session.header`** (session Z / X): `void_count`, `void_amount`, `void_printed_count` (those with `tvd_was_printed`), `line_void_count`, `line_void_value`. We add a VOIDS line under SALES on the Z.
- **`till.day.header`** (day Z): the same, for the day.
- Please list them in the README's "Printing" section, as for notes 135.

## 9 · `print_log_2027_2028`

`public.print_log` has partitions up to **2026_2027** only; the till tables already have 2027_2028. Please add `print_log` to the year roll (`ensure_acc_year_partitions` or wherever the till tables get theirs). Without it, prints stop being recorded on **1-04-2027**, and with them the printed-before-void evidence.

## 10 · What we do (no action from you)

- **Sale bill:**
  - the Void popup on Ctrl+F8 (reason, the printed / not-printed band, Void; the existing approval popup on a 428);
  - the `LINE_VOID` journal;
  - the 409 `SALES_USE_VOID` in plain words.
- **Grids:**
  - VOID / VOID · PRINTED pills on the bill list and the Cockpit's Bills tab;
  - Cockpit tiles VOIDS / AFTER PRINT / LINE VOIDS;
  - a **Voids** tab on Till Reports;
  - a "voids after print today" flag on Cash Position.
- **Live test:**
  - an unprinted void;
  - a printed void;
  - one run with the rule switched to ALWAYS (428 → popup → retry), then back to NEVER;
  - cancel refused on a live-till bill;
  - line voids from a draft;
  - the Z.

## Questions

1. **The cash in the drawer.** After a void, the session's expected cash drops by `tvd_cash_amount`, through cancel's reversal tender rows carrying the same `session_id`. Please confirm `sessions/expected` sees the reversal in the voided bill's session, so the drawer reconciles.
2. **The loyalty and promotion unwind** is cancel's. Is there anything a void within minutes of posting should do differently? We think not.
