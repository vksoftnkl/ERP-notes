# Void bill and void line: design (REV 2, 2026-10-17)

**Status:** design approved by the user on 2026-10-17. Mockups (§8.1) are drawn: `till_void_bill_ui_mockup.png`, `till_void_list_ui_mockup.png` and `till_report_voids_ui_mockup.png`, made by `till_void_ui_mockups.py`. Server notes written: **notes-136-void-bill.md** (asked 2026-10-17). Nothing is built.

**REV 2, the same day (the user):**
- **No approval, only seen.** *"just we want to see, no need to approve; it will complicate the usage."* No void asks a supervisor. The control is that every void is recorded and shown.
- **The sale bill only.** *"we need to record only sales txn"*, confirmed as the sale bill only. No other document is voided.

**Source:** `sales/SIX_TXN_CONTROLS.md` §7, parked by the user on 2026-09-22 ("session first, then void"). Since then the till session (TILL_DESIGN, live since 10-08), the approval gate (notes 124), the till event journal and the reasons master have all been built. This file turns §7 into something buildable on top of them. §7's reasoning stands and is not repeated here.

## 0 · The problem, in the user's words

> *"user can bill but won't take print — he will say printer not worked, so the customer unknowingly gives the amount to the user, he clears the bill and takes the money."*

And the quieter one (§7d): scan ten items, take the money for ten, then remove three lines.

## 1 · What the box does today (checked 2026-10-17)

| Piece | Today |
|---|---|
| `POST /bills/cancel` | Needs only menu 12 **Cancel**. It reverses the bill and logs a `txn_status_log` row and an audit row with a free-text `reason`. It is refused only when the bill's cash sits in a drawer that is already counted (`assertCancellable`). |
| Evidence | Nothing records whether the bill was printed, how long after posting it was cancelled, or which till session it happened in. **No `BILL_VOID` till event is written.** |
| `till_event` codes | `BILL_VOID` and `LINE_VOID` exist, but nothing writes them. |
| Approval rules | `VOID_BILL` and `VOID_LINE` are shipped as ALWAYS / SUPERVISOR / COUNTER (PIN), not self. Nothing asks them. |
| Reasons | The categories `VOID_BILL` and `VOID_LINE` exist, but **there are no reason rows**. |
| `print_log` | Records every render: `plg_status` SUCCESS / FAILED / QUEUED / CANCELLED and `plg_output_mode` PRINT / PREVIEW / EMAIL / FILE. **Partitions stop at 2026-27:** there is no `print_log_2027_2028`, though the till tables have one. From 1-04-2027 the evidence below would have nowhere to go (§7b's gap, sooner than it said). |

So the theft in §0 is fully open today: a cancel leaves no trace of the print, and nobody sees it unless they go looking.

## 2 · The model: a void is a cancel that is recorded and shown

**The accounting is cancel's, unchanged.** The same unwind runs: stock, ledger legs, tenders, loyalty, promotion, register. There is no second reversal engine.

**What void adds:**
1. a reason from the list;
2. a row of evidence, stamped once and never recomputed;
3. a `BILL_VOID` till event, so the session's timeline, its Z, Till Reports and Cash Position all see it.

**No supervisor is asked (REV 2).** Seeing every void is the control. A company that wants a PIN can still switch the `VOID_BILL` rule on (§3.3).

**The sale bill only (REV 2).** DC, order, quotation and both returns keep cancel only. Till money documents (movements, receipts, expenses) keep their own void and cancel.

**When it's a void and when it's a cancel:**

| The bill | Route |
|---|---|
| Posted in a till session that is **still live** (OPEN / SUSPENDED) | **Void.** `/bills/cancel` refuses with 409 `SALES_USE_VOID`. |
| Posted in a session already **counted or closed** | Neither, for cash: a **sale return**, as today's `assertCancellable` says. |
| Posted with **no till session** (office or credit bill) | **Cancel**, as today. |

So a void exists only while the customer could still be at the counter, which is the window §0 is about.

**Status stays `CANCELLED`.** Every report, GSTR-1 filter and register already treats CANCELLED correctly. A new `VOIDED` status would have to be taught to all of them for no accounting difference. What makes it a void is its evidence row, and the lists show a **VOID** pill from that row (**VOID · PRINTED** when it had been printed). **The number is kept** and the bill stays visible, as §7c asks.

## 3 · Server (for Prathap)

### 3.1 Table `accounts.till_void`, one row per voided sale bill

Partitioned by year, like the other till tables.

| Column | What |
|---|---|
| `tvd_id`, company, branch, tenant, `tvd_acc_year` | The usual. |
| `tvd_src_doc_type`, `tvd_src_doc_id`, `tvd_src_doc_refno` | **`SALE_BILL` only** (a CHECK says so). |
| `tvd_session_id`, `tvd_device_id`, `tvd_cashier_id` | The bill's session, the device the void was keyed on, and the session's cashier. |
| `tvd_voided_by`, `tvd_voided_on` | The caller, and now. |
| `tvd_amount`, `tvd_cash_amount` | The bill's net, and the cash part of its tenders (what left the drawer's expected). |
| `tvd_reason_id`, `tvd_note` | A `VOID_BILL` reason, and a note when the reason needs one (`trs_needs_note`). |
| `tvd_posted_on`, `tvd_minutes_after_post` | When the bill was posted, and whole minutes from post to void. |
| **`tvd_was_printed`** | **The control.** True when a `print_log` row exists with `plg_src_doc_id` = the bill, `plg_status` SUCCESS and `plg_output_mode` PRINT, **at the moment of the void.** Stored, never recomputed. |
| `tvd_print_count`, `tvd_last_printed_on` | From the same rows: how many prints, and the last one. |
| `tvd_print_failed` | A FAILED print row exists, so the "printer didn't work" claim was **true**. |
| `tvd_session_void_no` | This is the n-th void of the session (1, 2, 3 …). |
| `tvd_approval_id` | The `till_approval` row, only when a company has switched the rule on (§3.3); null otherwise. |
| `tvd_reversal_voucher_id`, `tvd_reversal_acc_year` | The cancel's reversal. |

**Checks (CHECKs only, no triggers):**
- `tvd_src_doc_type = 'SALE_BILL'`;
- `tvd_minutes_after_post >= 0`;
- `tvd_amount >= 0`;
- the reversal voucher id and year are both set or both null;
- one live row per bill: unique on (`src_doc_type`, `src_doc_id`).

### 3.2 `POST /bills/void`

**Body:** `{ sbId, sbCompanyId, sbBranchId, sbAccYear, reasonId, note?, approvalId? }`

**Steps, in one transaction:**
1. **Lock the bill.** It must be POSTED. A bill that is already voided answers idempotently.
2. **Check the session.** `sbSessionId` must be a session that is OPEN or SUSPENDED. Otherwise 409 `TILL_VOID_SESSION_CLOSED`, saying "the drawer is counted: make a sale return" (or "no till: cancel it").
3. **Check who.** Either the session's cashier **on the session's own device** (lockOwn, as end-billing), or anyone with Till Sessions (273) OVERRIDE. Otherwise 403.
4. **Check the reason.** It must be a `VOID_BILL` reason, plus a note when the reason needs one.
5. **Work out the evidence:** printed or not, prints, failed print, minutes after post, the session's void number.
6. **Ask the gate** on `VOID_BILL` (§3.3). With the shipped rule (NEVER) nothing is asked. If a company switched it on:
   - needed and no `approvalId` → **428 `TILL_APPROVAL_REQUIRED`**, the same contract as Expense / Payment / Open Till;
   - an `approvalId` must be APPROVED, for this bill, and not expired.
7. **Run cancel's unwind**, with the reason's text as the remark. This covers the GST-first gate (`assertUnwindable`), the reversal, status CANCELLED, the status log and the audit row.
8. **Insert the `till_void` row.**
9. **Journal `BILL_VOID`.** Payload: `{ sbId, refno, amount, cash, reason, printed, printCount, minutes, voidNo, approvalId }`.
10. **Answer** with the bill, the void row and the reversal number.

**Rights:** menu 12 **Cancel** as well. A void is a cancel, so a cashier without the Cancel right can't void.

### 3.3 No approval by default (REV 2)

**Ship the `VOID_BILL` rule as `NEVER`.** Change the seed and the live shipped row from ALWAYS. Out of the box, no void asks anybody.

The gate stays in the route, so a company that wants a supervisor can set the rule in Approval Rules (276) to ALWAYS, OVER_AMOUNT (`tar_threshold_amount`) or OVER_COUNT (voids already in the session). The client's existing 428 popup then asks.

**There is no printed-bill override.** A printed bill is voided like any other. It is stored with `tvd_was_printed = true` and shown as AFTER PRINT everywhere (§4).

`VOID_LINE` stays as shipped and is never asked (§3.5).

### 3.4 `/bills/cancel` change

A bill whose session is OPEN or SUSPENDED is refused with **409 `SALES_USE_VOID`**: "This bill is in an open till. Void it instead." Nothing else about cancel changes.

### 3.5 Void line: `LINE_VOID` from the sale bill screen

Lines are removed and reduced on the **client**, often on a draft that is never saved. So the sale bill screen reports them through **`POST /till/events/batch`**, the existing offline-safe journal, where a re-sent batch is a no-op. A `LINE_VOID` event is sent when a line on a sale bill keyed in a till session (draft or posted) is:
- **deleted** after it was added, or
- has its **qty reduced**.

No other sales screen journals lines.

**Payload:** `{ docId? (null until the first save), refno?, itemId, itemName, qtyBefore, qtyAfter, rate, value (the line value removed), wasPrinted (the bill had a SUCCESS print), lineAgeSeconds (since the line was added) }`.

**Server:**
- accept `LINE_VOID` from a device (today `events/batch` refuses server-only codes; LINE_VOID must not be one);
- check the session is the device's live one.

**No approval.** The control is the **watch**: count and value per cashier per session (§6).

### 3.6 Reasons to seed (shipped, company NULL)

| Category | Code | Name | Needs note |
|---|---|---|---|
| VOID_BILL | CUST_LEFT | Customer left | no |
| VOID_BILL | WRONG_ITEM | Wrong item billed | no |
| VOID_BILL | PRICE_DISPUTE | Price dispute | yes |
| VOID_BILL | PRINTER_FAIL | Printer failure | yes |
| VOID_BILL | OPERATOR_ERR | Operator error | no |
| VOID_BILL | SYSTEM_ERR | System error | yes |
| VOID_LINE | (none) | Lines are a journal, not a form: no reason is asked at the counter | |

### 3.7 Print data and `print_log`

- **Session Z / X (`till.session.header`):** add `void_count`, `void_amount`, `void_printed_count`, `line_void_count` and `line_void_value`. The Z prints a VOIDS line under SALES.
- **Day Z (`till.day.header`):** the same, for the day.
- **`print_log_2027_2028`:** add it to the year roll, as the till tables are. Without it the printed-before-void evidence stops on 1-04-2027.

## 4 · Client (Qt)

### 4.1 Sale bill

- **Void bill (Ctrl+F8; checked free on the sale bill)** works on a loaded POSTED bill whose session is live. The office sale bill already sends `sbSessionId`. Otherwise the key says why it's off: "Not in an open till: cancel it" or "The drawer is counted: make a sale return".
- **The Void popup** (mockup 1). It has no approval strip.
  - It shows the bill, its amount and the cash part.
  - A green band says the bill was never printed, **or** a red band says it was: "Printed 2 times — 10:12 AM and 10:13 AM, both SUCCESS … recorded as AFTER PRINT".
  - It takes a reason (a dropdown of `VOID_BILL` reasons), and a note when the reason needs one.
  - Then Void.
- **On 428** (only if a company switched the rule on): the existing `TillApprovalRequestDialog`, then a retry with `approvalId`, as Expense 277 does.
- **Cancel (existing)** keeps working for bills with no till. On a live-till bill, the server's 409 `SALES_USE_VOID` is shown in plain words (`api_error_text`), naming the Void key.
- **The list (TxnMainView)** shows a VOID / VOID · PRINTED pill instead of CANCELLED when a `till_void` row exists (a grid change). A "Voided" column says when, why and how long after the post. The number stays.

### 4.2 Void line journal (no UI)

With a live till session, the sale bill screen journals a `LINE_VOID` event when:
- **a line is removed** (`-`, Ctrl+Del, the row menu): its value;
- **its qty is lowered:** the difference.

Not journalled:
- a line edited **before** it was committed (the item still being chosen);
- a qty raised;
- a price change (that is `PRICE_OVERRIDE`, a separate control).

**Sending:** queued in memory, sent in batches (on save and every 30 s) and retried, with a `clientSeq` per device. If the device is offline the queue waits; it is never lost silently while the app is open.

### 4.3 Till screens (mockups 2 and 3)

- **Session Cockpit:**
  - tiles **VOIDS** (count · value), **AFTER PRINT** (red when any) and **LINE VOIDS**;
  - the Bills tab shows the VOID pills and the reason;
  - the timeline shows every `BILL_VOID` / `LINE_VOID` as written.
- **Till Reports (288), a new eighth tab "Voids":**
  - per cashier: bills, voids, value, % of bills, after print, median minutes after post, line voids and their value, **voided by**;
  - below it, the picked cashier's bill and line voids, newest first;
  - the Exceptions tab's counts include BILL_VOID / LINE_VOID.
- **Cash Position (S11):** a flag "n voids after print today" per store.

## 5 · What this does NOT do

- **Any document but the sale bill (REV 2).** DC, order, quotation and both returns keep cancel only. There is no DC void, now or later.
- **A supervisor for any void, by default (REV 2).** A company can switch the `VOID_BILL` rule on; line voids are never asked.
- **Blocking billing after n voids.** Never: the counter is never blocked, and seeing the voids is the control.

## 6 · The watch: what the Voids tab answers

"Six voids, ₹4,200, all by one cashier, all within two minutes of posting, five of them after a successful print" (§7e). The tab sorts by printed-before-void, then by count. It has no cut-off figure: HO reads it, and the Cash Position flag points at stores with any voids after print.

## 7 · The user's decisions (2026-10-17)

1. **Void only in the same live session**; otherwise cancel (no till) or sale return (counted). Yes.
2. **Supervisor: none (REV 2).** Every void is recorded and shown instead. The rule ships NEVER; a company can switch it on.
3. **Line voids journalled only**, from the sale bill. Yes.
4. **Sale bill only (REV 2).** No DC void.

## 8 · Order of work

1. ~~Mockups (PNG)~~: drawn to REV 2 on 2026-10-17 (three PNGs, listed under Status).
2. ~~Notes to Prathap~~: notes 136. §3 (table, route, the `/bills/cancel` change, `events/batch` LINE_VOID, the `VOID_BILL` rule NEVER, reasons, print fields, `print_log` 2027-28).
3. Qt: the Void popup, the line journal, the Cockpit tiles, the list pill, the Voids tab.
4. Live test:
   - void an unprinted bill (nothing asked);
   - void a printed bill (stored AFTER PRINT, nothing asked);
   - switch the rule to ALWAYS for one test: 428 → popup → retry, then back to NEVER;
   - cancel on a live-till bill refused;
   - line voids journalled from a draft;
   - the Z and the Voids tab show them.
