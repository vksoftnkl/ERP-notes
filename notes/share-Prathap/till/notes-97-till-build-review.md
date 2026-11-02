# Notes 97 — Till build review (phases 1–2, money docs, non-cash, claim)

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-08 · **For:** Prathap

Read against TILL_DESIGN, plan-till-review-rev2, plan-till-receipt-payment-expense,
plan-noncash-tender-control, plan-till-counter-claim and plan-till-operator-to-user, on the
source as of 2026-10-08 19:59 and the live DB (all 11 till migrations applied).

**Overall: well built.** Schema, CHECK lists, settings, menus, voucher types, the expected-cash
formula, TSet legs, ExpV legs and the counter claim all match the plans. The claim is now
session-only (nothing written to the counter), open-check is live, and web devices claim but
are never linked. Your declared deferrals (phases 3–6, 269ST on receipts, re-link, Qt screens)
are fine and not repeated here.

Items marked **(verified)** were re-read line by line by us; the rest are from the review and
are worth a look.

---

## 0. Fixed on OUR side — nothing for you to do

**Client sent `tdSettleStatus: "NA"`** on every bill, re-tender and sale-order tender row
(`sale_bill_entry.cpp`, `sale_bill_retender.cpp`, `sale_order_entry.cpp`). `defaultSettlement`
(tender-detail.service.ts:1026) keeps a status the caller sends, so **no card/UPI row could ever
become PENDING** and the whole settlement pipeline was unreachable from the app. The client no
longer sends the field. Your e2e never sent it, which is why it passed.

Suggest you also **ignore** `tdSettleStatus` / `tdSettleVoucherId` / `tdSettledOn` from the bill
DTO entirely (server-owned), so an older client or an amend can't reset a SETTLED row to NA.

---

## 1. High

**1.1 Cash moves with no session on an unguarded device (verified)**
`till-session.service.ts:1106-1153` `resolveForMoney` answers null when the device holds no
live session and is not linked to a counter. Then `bill-lifecycle.service.ts:212-226` keeps the
`sbSessionId` the client sent.
- A tablet that picked C03, counted and closed at 14:00 posts a ₹2,000 cash bill at 14:05:
  accepted, in no drawer. Same for a tablet that never opened a session, and while its session
  is PENDING_APPROVAL.
- If the client sends another device's live session id, the cash lands in **that** drawer's
  expected figure and its Z (false shortage on the other cashier).
- Plan (counter-claim rows 2, 9 and S1 "Not now leaves billing disabled") says: in a branch that
  runs a till, under `till.require_session = true`, a cash document from any device without a
  live session → 409 `TILL_SESSION_REQUIRED`. The README's "rollout rule" is fine for a branch
  with no counters at all; it should not cover a branch that has them.
- Ask: when the branch has an active counter needing a session and `require_session` holds,
  refuse instead of null; and never keep a client-sent session id that is not this device's live
  session.

**1.2 PENDING_APPROVAL locks the counter, cashier and device for good**
Nothing leaves PENDING_APPROVAL in phases 1–2 (close → 428, count refuses it, force-close is
phase 4), but `assertCounterFree` / `assertOperatorFree` / `assertDeviceFree` (~:2000-2050) treat
it as live. With live defaults (cash_tolerance 10, max_recounts 1) a ₹11 gap after one recount
locks that counter, that cashier on every counter (TILL_OPERATOR_BUSY) and that device.
Ask: until phase 3 ships, either let a 273 OVERRIDE holder accept the variance (post it as
SUSPENSE), or do not count PENDING_APPROVAL as holding the cashier/device.

**1.3 A settled card row can be re-tendered / cancelled away (verified)**
`bill-retender.service.ts:166` guards SETTLED/PARTIAL only for `td_is_pdc`. A card row settled
by a TSet, re-tendered to cash: the clearing ledger is credited twice, the money counted twice.
Same for bill cancel and `syncDocumentTenders`' soft delete. Ask: refuse (or require the
settlement unlinked first) for any SETTLED/PARTIAL row or a row with a live match.

---

## 2. Medium

**2.1 Close can never finish after a zero orphan slip line (verified)**
Count accepts a SLIPS line with no tenderId for a tender the session never took (S4 mockup
sends "Gift voucher [0]"). `close()` (:973-976) calls `tenderLedger(tx, null, …)` before looking
at the amount → 422 TILL_LEDGER_UNMAPPED every time; the count is already final.
Ask: skip zero variances before resolving a ledger (or drop zero orphan lines at count).

**2.2 Re-tender writes into a CLOSED session**
`bill-retender.service.ts:128` `moneySessionId = till?.tssId ?? bill.sbSessionId`. A back-office
re-tender of a closed session's bill stamps the new CASH row and its TndC leg with the closed
session → its §5.7 invariant and recomputed expected move. D7: the session the money moves in
NOW, never the original.

**2.3 Same statement rows imported twice**
Duplicate check is only the file sha256 (tender-settlement.service.ts:201/281). A weekly file
holding lines already posted from the daily file imports them UNMATCHED → suspense, bank debited
twice. Ask: a per-line key (RRN / ref + terminal + amount + txn date) across imports.

**2.4 ADJUSTMENT lines lose their direction (verified)**
`settlement-format.ts:246` takes `abs()` of gross/fee/tax. Fine for SALE/REFUND/FEE (kind carries
direction), but a −100 debit ADJUSTMENT books as +100. Also: no setting to skip a preamble or a
"Total" footer row — a total line with no kind column imports as a SALE.

**2.5 Slip check empty for a single-terminal shop**
Count stores `tclTenderId: line.tenderId ?? null` (:2583) though the count logic itself assigns
an unnamed line to the session's only terminal; `till-slip-check.service.ts:65` filters on
`tcl_tender_id = tenderId` → batch total / slips / difference all null. Store the resolved id.

**2.6 Blind close leaks through the 428**
close on a PENDING_APPROVAL session returns `amount` = Σ|counted − expected| to the session's own
cashier (272 EDIT). They know what they counted → they know expected, while the cash is still in
the drawer. Omit the amount for the session's own cashier.

**2.7 Expense GST: intra/inter from the wrong state (verified)**
`expense-derive.ts:127-129` compares place of supply with the **company's** state. IGST Act:
intra = supplier's location and place of supply in the same state. A Karnataka hotel bill
(POS KA, supplier KA) to a TN company books IGST input; it is CGST+SGST. Compare POS with the
supplier GSTIN's state.

**2.8 40A(3) sums an expense ledger as a payee**
`cash-payment-limit.ts:80` sums by `td_party_ledger_id`; a no-party ExpV writes the expense
ledger there (expense.service.ts:243) and a register PmtV's instrument line can be an expense
ledger. Six ₹2,000 courier expenses to different couriers + one ₹500 PmtV "Courier" → 40A(3) at
12,500. REV 2 §2.17: a no-party expense is never summed. Only sum rows whose ledger is a party.

**2.9 Resolve / write-off can't be undone**
Void says "undo those first", but there is no un-resolve or write-off reversal; one mistaken
resolve locks that payout. A SUSPENSE write-off ("still chasing") ends the row as FAILED, so late
money can't be matched. Overpaid REF matches within tolerance stay PARTIAL with negative
outstanding and write-off refuses them.

---

## 3. Low

- Second count after ACCEPTED → generic 500 (P2002 on `ux_tct_final`) instead of a till code (:723).
- `resolveForMoney` reads the session without FOR SHARE; a bill in flight at end-billing can land
  after the count froze. `assertDeviceFree` / `assertOperatorFree` are unlocked reads with no
  index: a double-tap on an unlinked tablet can open two sessions.
- Linked path in `claimCounter` (:1894-1911) never checks `tcnRequiresSession`; `open()` never
  reads `settings.requireSession` — a PC told `requireSession:false` can still open.
- `routeMoneyDoc` (:1413) returns "none" for a token with no device id, before the branch-runs-
  a-till check: such a caller's cash goes to the till cash ledger outside any session. (Web logins
  do carry a `web:` device, so this is only a deviceless token.)
- Cut-off judged on client `sb_bill_datetime`: a till with a wrong clock bills into yesterday.
- `cashLimit` payload sends `alertLimit` / `blockLimit` figures alongside the gauge — drop them for
  a blind cashier.
- Pickup `witnessBy` may be anyone but the caller; plan says the cashier.
- Expense: voucher date from the client, not the session's business date; no ROUND_OFF line
  (a paise-off supplier bill can't be entered); no delete route for a draft though 277 has DELETE.
- Not read anywhere: `till.close_witness`, `till.handover_mode`, `till.session_max_hours`. No
  CASH_ALERT / CASH_BLOCK / CASH_UNBLOCK events logged on a state change.

## Asked

Section 1 before the till screens go to a shop; section 2 when you can; section 3 is your call.
