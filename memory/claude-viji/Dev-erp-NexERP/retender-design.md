---
name: retender-design
description: change how a posted bill was PAID without editing the sale — 35_tender_void.sql, the TndC contra, and why it works even with an IRN
metadata:
  node_type: memory
  type: project
---

**The user's scenario, 2026-09-21, and it happens many times a day at a
supermarket till:** the bill is rung up, the customer says UPI, the operator
marks UPI and moves to the next customer; the UPI then fails and they pay cash.
The SALE was right, only the PAYMENT changed — but today that forces an edit or
a cancel of a posted bill.

**It works even on a bill with an IRN, and that is the key insight: the tender
is NOT part of the GST document.** Not on the e-invoice, not in GSTR-1, not on
the e-way bill. So changing it declares nothing new and lock 2
([[transport-band-shared-table]], HANDOVER §1.3a) does not reach it. Same
reasoning as `PUT /bills/transport`: a separate concern with its own lifecycle
gets its own verb instead of travelling through `/amend`.

**`35_tender_void.sql`** (after 30) adds to `accounts.acc_tender_detail`:
`td_is_voided`, `td_void_reason` (UPI_FAILED | CARD_DECLINED | CHEQUE_REFUSED |
KEYED_WRONG | CUSTOMER_CHANGED | OTHER), `td_voided_on`, `td_voided_by`,
`td_replaces_id`, plus `ck_td_void_pair` (voided ⇒ reason + date, live ⇒
neither) and a partial index. Also the **`TndC`** voucher type and the
**`um_can_retender`** right on `user_menus`.

**VOIDED, not deleted, and not a negative row.** The failed UPI happened at the
counter and a till review must see it; `td_is_deleted` would say "keyed wrong"
when the truth is "payment failed". A negative counter-row is unavailable —
`ck_td_amounts` enforces `td_amount >= 0`, and that CHECK is correct (single
row's own arithmetic, not a contended resource). Reports filter
`NOT td_is_voided`; audit reads both halves.

**A CONTRA, not a reverse-and-repost.** `DR` the tender that really happened,
`CR` the one that did not, same amount — **the party balance never moves and
the bill stays settled throughout**. Reversing and re-posting would unsettle it
in between, and a failure at that moment would leave it unsettled. `TndC` is
`ACCOUNTING|CONTRA` like `ChqClr`.

**Gotcha hit while writing it:** `chk_acc_voucher_types_cash_bank_exclusive`
forbids `is_cash_voucher` AND `is_bank_voucher` both true. A tender change can
go cash→bank or bank→cash depending on the row, so BOTH are false (like Bil and
Rct); ChqClr can say bank=true only because it always is one.

**Two tender types are not simple swaps:** voiding a **LOYALTY** row must route
through `LoyaltyLedgerService` so the points come back (else the customer is
silently robbed); voiding a **TEMP_CR** row must clear its `acc_temp_credit`
entry (31), which is the WHO on the bill's balance row.

**Refusals:** `SALES_RETENDER_AMOUNT_MISMATCH` (new total ≠ voided total — a
different amount is a part payment, not a re-tender), `SALES_DAY_CLOSED` (the
drawer was already counted), `SALES_RETENDER_PDC_MOVED` (deposited/cleared PDC
— that is a bounce), `SALES_RETENDER_RIGHT`.

**Client (user was explicit): Ctrl+F6 opens the TENDER FORM ONLY — it NEVER
loads the bill.** That is the point of the feature: by the time the customer
comes back the operator is already keying the next bill, so loading the old one
would throw away their half-typed work and put an item grid in front of someone
who only needs to fix a payment. `sale_bill_entry` is not touched, cleared or
navigated; the dialog sits over it.

Flow: Ctrl+F6 → picker **grid 133** (POPUP - RECENT BILLS FOR RE-TENDER: this
device + session, today, newest first, plus a bill-no box for older; default to
the newest, so the common case is Ctrl+F6, Enter) → the tender form → `POST
/bills/retender`, close, no reload.

The dialog reads **`GET /bills/tender-context`** (HANDOVER §2.12a), NOT
`/bills/get` — no items, charges, party context or transport. It returns the
tender rows plus `isLoyalty` / `isTempCredit` / `pdcMoved` / `canRetender` so
the dialog can warn BEFORE the operator commits.
Form = `sale_bill_tender` in re-tender mode (Void tick + reason per row, new
rows below, Voided vs New totals in the footer, OK dead until they match).
**The screen must NOT grey Ctrl+F6 from `locks.editable.document`** — it is the
one thing a declared bill still allows.
Ids: grid **133** reserved (live ceiling 112, sales reserved 113–132).
