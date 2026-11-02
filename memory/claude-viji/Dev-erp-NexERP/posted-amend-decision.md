---
name: posted-amend-decision
description: R20 — editing a POSTED receipt in place behind a company setting, and why the full create+post payload beat field-by-field routes
metadata:
  type: project
---

Client request 2026-09-17: small shops where the owner IS the operator need to correct a
posted receipt (typically a mis-keyed cheque number) without cancelling and re-keying the
whole document. Spec written: `share/receipt/plan-backend-amend-posted.md`, linked from that
folder's README and HANDOVER_endpoints.md as route 9.

**R20 taken: both models.** `accounts.allow_posted_amend` (BOOL, COMPANY scope, **default
false**) leaves today's cancel-and-re-enter untouched; on, `POST /receipts/amend` restates the
document in place.

**The shape decision.** A tier of narrow routes (header text / instrument detail) was proposed
first and rejected by the user — an operator does not think in tiers, and a route per field
re-implements the posting rules in N places. Amend takes **the whole create+post payload**,
which is the object the screen already assembles, plus the four keys and a mandatory
`editRemark`. Transaction = §5.3's money unwind **without writing a reversal voucher and
without CANCELLED status**, then §5.2 steps 2–12 from the new payload, against the SAME header,
SAME voucher number, `avh_revision_no + 1`. Numbering is never re-allocated: the customer holds
a slip with that number.

**What flipped the argument.** In-place edit of posted money was argued against, then accepted,
on one fact: **`audit.audit_log` already exists and is live** (10,777 rows on 2026-09-17) with
`log_original_record` / `log_modified_record` / `log_changed_fields` as jsonb plus user, branch,
device, IP and screen. It already covers `receipt tender`, `sale_bill`, `stock_voucher` and the
masters — only `acc_vouchers`, `acc_bill_adjustment` and `acc_pdc_register` still need
registering. With a before/after snapshot per row, an amend is BETTER evidence than
cancel-and-re-enter, which leaves two documents and makes the reader infer the difference.

**Two sub-decisions (2026-09-17).** (a) Its OWN route, not a `mode` on `/create` — `/create` is
what every draft save and every timeout-retry goes through, and it must keep answering 409 for a
POSTED voucher; an explicit-mode single URL was considered and is sound, but amend is the only
route gated by a company setting and a route you can switch off beats a flag checked on every
request. (b) **`baseRevision` is mandatory** — the amend carries the WHOLE document, so
last-writer-wins would silently undo another user's correction on ledger legs. `/receipts/get`
must return `avhRevisionNo` for the round trip to close.

**The limit no setting can lift.** Amend refuses exactly what `/cancel` refuses, verified live:
*"Cheque 55491 is CLEARED. Once an instrument has left the drawer the receipt behind it cannot
be unmade."* Also a spent on-account advance, a non-POSTED voucher, and a locked period (to be
added now, though the lock does not exist yet). The window amend serves is a cheque still HELD,
minutes after entry — which is where the reported pain is.

Client side is nearly free: with the setting on and `canEdit` on menu 99, a loaded POSTED
receipt unlocks and **Post & Print** becomes **Save changes**, calling `/amend` instead of
`/create` + `/post`. Cancel and the register's F3 are unaffected. See
[[receipt-screen-revision]] and [[receipt-draft-delete-gap]].

## Shipped and tested 2026-09-17 evening

Prathap built both routes the same day. Retested against the live server:
**`/receipts/delete` 10/10**, **`/receipts/amend` 14/16**, and all three client suites still
green (receipt 56, cheques 33, cheques-ops 34). Everything the spec asked for landed —
`avh_revision_no`, the COMPANY setting default false, `avhRevisionNo` on `/get`, and all four
money tables now in `audit.audit_log` with real before/after payloads.

Verified in the DB, not from the response: restating 500 → 1200 moved the bill's pending by
exactly 700, left TWO live legs at 1200/1200 (`av_is_deleted` filters the superseded ones —
forgetting that filter makes it look like legs are doubling), rewrote the cheque's amount on
`acc_pdc_register`, kept `rct00049` across revisions 0→1→2, and left adjustments
reversal-paired netting exactly the receipt. The deposited-cheque refusal fires with the
sentence from §4.

**THREE defects — SENT as notes (28), ALL FIXED AND VERIFIED the same day.**
Re-tested live 2026-09-18: (a) a missing `avhVoucherId` is a 400 naming the field, and it is in
`required` now; (b) a party change is a **409** — *"rct00259 was received from Deepan and an
amend cannot move it to MADHAVAN. The receipt keeps its number, and that number is on a slip
the first customer is holding"* — with the first party's bill untouched; (c) amend-then-cancel
works (201, "1 voucher(s) reversed") and every ledger nets to 0.00, the bill returning to its
full OPEN amount. **The caution about not using Correct receipt on live data is lifted.**

The three as originally reported:

- **(b) an amend really does move a posted receipt to another customer.** The earlier note
  guessed this "would likely" happen; it does. `rct00111` went Deepan → MADHAVAN with a 201,
  CR leg reassigned, Deepan's bill reopened, MADHAVAN's settled, adjustments netting. The work
  is internally CONSISTENT, which is what makes it dangerous — and the receipt **keeps its
  number**, so a slip Deepan holds now names MADHAVAN's money. Fix = refuse a changed
  `avhPartyId` outright (§6 implies it).
- **(c) NEW — a receipt that has been amended can never be CANCELLED.** `/receipts/cancel` on
  an amended receipt is a 500 with an empty `errors` array, twice. Control: post + cancel
  without amending works (201). So amend is a one-way door, and combined with (b) the receipt
  that landed on the wrong customer cannot be undone either. `rct00111` is left in that state
  on the box as the repro.
- **(a) still open** — `/amend` with no `avhVoucherId` is a 500, because `avhVoucherId` is
  missing from `AmendReceiptDto.required`.

What is RIGHT and was leaned on: the number survives, `avh_revision_no` increments,
`baseRevision` rejects a stale document, superseded legs soft-delete, adjustments net exactly.

**`accounts.allow_posted_amend` is ON for Acme Foods** — enabled for testing, not turned back
off.

## WIRED INTO THE CLIENT 2026-09-18 (commit dc9ea9c)

The user asked "how to edit posted receipt there is no option provided" — correctly: only
`Edit header` existed, and it changes four text fields and no money.

Now on the receipt screen: **`btnAmend` "Correct receipt"**, HIDDEN (not greyed) unless the
receipt is POSTED *and* `accounts.allow_posted_amend` is on — a permanently dead button is a
question the operator keeps asking. `beginAmend()` confirms, sets `m_amending`, unlocks the
grids, pills **AMENDING**, and turns **Post & Print** into **Save changes**; `post()` then
routes to `sendAmend()`.

Three things the wiring needed that were easy to miss:
- `post()` starts with `if (m_saving || !m_header.isEditable()) return;` and `isEditable()` is
  FALSE on a POSTED receipt — right for every path but this one. It is now
  `(!m_header.isEditable() && !m_amending)`.
- the payload is `toDraftJson()` **merged with** `buildPostPayload()`, plus `avhVoucherId`,
  `baseRevision` and `editRemark`. `ReceiptHeader` gained `revisionNo`, read from
  `avhRevisionNo` on `/get`.
- `startNewDocument()` must clear `m_amending` and put the button's caption back, or the next
  receipt is keyed on a screen that still thinks it is restating.

`AppSession::allowPostedAmend()` added, fed by `accounts.allow_posted_amend` in the settings
map beside the other `accounts.*` keys.

**Still not exercised by a test** — the user asked me to stop running the suites on every
change, so this went in on a clean build only. The two defects recorded above (`/amend` with no
`avhVoucherId` is a 500; a changed `avhPartyId` is caught by the wrong rule) were never sent to
Prathap and are still open.
