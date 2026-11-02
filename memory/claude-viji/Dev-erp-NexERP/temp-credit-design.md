---
name: temp-credit-design
description: "TEMP_CR tender (31_temp_credit.sql) — a counter loan to a non-credit party: the WHO table rides on the bill's acc_bill_balance row, balance maintained from abl_pending_amount, receipt finds it by mobile; menu 257 / grid 131; what 3.0 did and did not do"
metadata: 
  node_type: memory
  type: project
  originSessionId: 0b00a13b-6c14-4cbf-9fa7-d864e9a434dd
  modified: 2026-09-20T09:39:26.747Z
---

Designed 2026-09-20 on the user's ask ("TEMP_CR for who ask credit for
certain days … a table to store temp cr party details to adjust in receipt …
stored on the party chosen in the sales window").

**3.0 facts** (DB `3.0`): `public.acc_temp_credits` (tmcr_*), tender type 7
"Temp.Credit", written at sale by `app_utils.fx_sale_entry_bak` and
`sales.fx_save_sales_entry_v1` (name in `bank`, place in `notes`, mobile in
`card_no`, due date), DELETED on edit/cancel, and **never settled by any
function** — all 13 live rows are "Pending". Client frame `fTempCredit`:
Name, Mobile, Place, Pay-in days, Amount.

**Ours:** `accounts.acc_temp_credit` (partitioned, `31_temp_credit.sql`,
run after 30). The TEMP_CR tender row (acc_tender_types 8) posts exactly
like CREDIT — no leg, the bill's `acc_bill_balance` row stays OPEN for the
amount — so the OUTSTANDING is already on the party the sales screen chose
(the walk-in ledger). The table is only the WHO: name, mobile (mandatory),
place, addr, days → due date, promise/follow-up, `atc_abl_id` → the balance
row, `atc_tender_id` → the tender row. `atc_balance_amount` is MAINTAINED =
`abl_pending_amount` by `BillBalanceRecomputeService`; bill cancel →
`atc_status CANCELLED`, never deleted.

**Why:** the walk-in ledger has a thousand bills; the receipt needs the
mobile to find the right one, and money must stay bill-wise so a receipt,
write-off or credit-note set-off settles it like any bill.

**How to apply:** receipt module gains ONE addition — `/receipts/open-items`
returns `tempCredit {name, mobile, dueDate}` per bill and accepts `?mobile=`;
`/receipts/post` stamps `atc_last_receipt_id`. Settings:
`sales.temp_credit_max_days` (30, REFUSE), `_max_amount` (0 = no cap, WARN),
`_block_open` (same mobile already OPEN: WARN with balance / REFUSE). List
screen = menu 257, grid 131 (`TxnMainView`, no create; actions Open bill /
Receive / Follow-up), picker grid 132 for the receipt. Pictures:
`share/sales/sale_bill_tender_additions_ui_mockup.png` (panel E) and
`temp_credit_list_ui_mockup.png`. Also settled the same day: CREDIT row =
outstanding, RRN retired (credit note is a NOTE_ADJUST adjustment —
[[party-adjustment-design]]), `sales.tender_type` already exists for
"tender popup for all / cash bills". See [[sales-txn-plan-set]].

**2026-10-02: collection wired, with no new form (user: "we don't want a temp credit collection form").**
- **Menu 257 is granted to VKPOS** (full-echo save; 84 → 85 rows, only +257, no flag changed). It is the "report". Before this no user had it.
- **Grid 114** gains `atc_abl_id`, `atc_party_id` and a visible **Party** column `atc_party_name`, through a join `acc_bill_balance` → `acc_ledger_master`. Re-registered with replace_columns=true; files share/grids/temp_credit_list_grid_114.{json,sql}.
- **The list's Receive** = `SessionEvents::requestOpenMenu(99, args)`. MenuWindow opens the form, then calls `NexDialog::applyOpenArgs()`, a new generic virtual. `ReceiptEntry::applyOpenArgs` sets the party, the mobile filter (`/receipts/open-items?mobile=`, `Api::openItems(..., mobile)`) and ticks that billId (= abl_id) at its pending, receiveTyped=true, Amount = pending.
- **Receipt:**
  - the bills header has a `txtTcMobile` "TEMP CR MOBILE" filter, with a banner naming the borrower (cleared with the filter);
  - temp-credit bills show a **TEMP CR** amber chip in Type and a tooltip (name · mobile · due);
  - OpenBill parses `tempCredit`.
- **Probe:** read-only, bil00950 → 1 bill, receive 210, amount 210; unfiltered, 4 bills / 1 TC. Receipt suites NOT run (live; the user wants one run after their testing).
**Same day, user fixes:**
- **No Add on the Temp Credits list:** it opened a blank SALE BILL, not a customer. New `TxnMainViewConfig::allowAdd` (default true) stops deriveEntryHandlers making onAdd.
- **User column** = login name: grid 114 joins public.user_master `cu.usr_id::text = atc_created_by`, under the same field name.
- **Receive opened the receipts LIST** (menu 99 is list-first). `TxnMainView::applyOpenArgs` now opens a NEW entry (openEntryChild) and forwards the args queued. Probe: list → entry → bil00950 ticked 210.

**2026-10-05: History (Ctrl+H) on menu 257 fixed.**
- It was always empty: History read txn_status_log by `atc_id`, which the server never logs.
- New **grid 132** "POPUP - TEMP CREDIT HISTORY" (share/grids/temp_credit_history_grid.*) is a UNION ALL of four sources:
  - BILL: the bill's tsl rows, by atc_src_doc_id
  - CREDIT: the atc row
  - FOLLOWUP: audit_log rows for acc_temp_credit
  - SETTLE / COUNTER: acc_bill_adjustment on atc_abl_id. `abj_tender_id` set = the bill's own till cash, NOT the loan; the balance row is the whole bill.
- Client side: a generic `TxnHistoryTrail` (txn_history_trail.h) on `TxnMainViewConfig::history`, opened by `TxnHistoryDialog::showTrail`. The status trail is now just the default trail (`statusTrail()`).
- Test: tests/tempcredit/run-live.sh, 15/15 pass, read-only.
- Server ask: share/sales/notes-90 (TEMP_CREDIT tsl events from the recompute, an insert audit row, WRITTEN_OFF is never set).
