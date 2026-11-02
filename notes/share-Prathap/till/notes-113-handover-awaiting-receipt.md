# Notes 113: a session's cash reaches the safe only when a supervisor receives it (2026-10-14)

**STATUS:** server DONE (migration 20261014100000): routes `movements/awaiting-receipt` + `movements/receive`, menu 286 Cash Office, role TILL_CASH_AWAITING (mapped), voucher TRcv, RECEIVE-stage variance decided via decide-variance + tcmId. Setting `till.handover_receipt` defaults to NONE; set REQUIRED for the company for the live test. Qt Cash Office built and verified live 7/7 on 2026-10-14.

**SUPERSEDED IN PART by notes-114 (2026-10-14):** the user replaced the Cash Office receive screen with a single supervisor Approve Cash on the closed session. The ledger, the setting and the awaiting state stay.

## Why

Live, `VIJI PC-261014-02` (VIGNESH, Coimbatore): open 12:17:17 → end billing → count ₹500 (ACCEPTED, within tolerance) → close 12:17:42. **CLOSE_HANDOVER ₹500 was posted into the safe** (TDrp Dr Safe / Cr Till cash). The only person on every step was the cashier: `tss_counted_by`, `tss_closed_by` and `tcm_done_by` are all VIGNESH, `tss_witness_by` is NULL, and the session needed no approval.

So the books say the safe holds the money, but nobody confirmed receiving it. A cashier who counts "correctly" and keeps the cash never meets a supervisor. Approval only triggers when the count is **out** of tolerance.

What exists today does not close this gap:

- `CountTillSessionDto.witnessBy` is optional and a bare uuid. It takes no PIN, and nothing requires it. The `till.close_witness` setting named in its description does not exist.
- `till.count_place = CASH_OFFICE` still lets the cashier count it (`lockForCount`: operator on any device).

## The user's choice

The user chose this option over two others: a supervisor PIN on close, or the supervisor doing the count.

> The cashier closes as today, but the cash goes to **awaiting receipt**, not the safe. A supervisor later **receives** it with their PIN (counting it) to move it into the safe. Cash not yet received shows on Business Day as a red check.

This is TILL_DESIGN §8 S9's "drop bags awaiting verification (count each bag → DROP_VERIFY; a bag difference becomes the dropping session's variance … never an edit)", brought forward and applied to the close hand-over.

## Asked

### 1 · Posting

| When | Voucher | Dr | Cr |
|---|---|---|---|
| Close hand-over (cashier) | TDrp | **Cash awaiting receipt** (new) | Till cash |
| Receive (supervisor), the amount received | TDrp (or a new type, your choice) | Safe | Cash awaiting receipt |
| Receive: received ≠ handed over | TVar via the session's variance (see §4) | Cash short & excess / recovery / suspense | Cash awaiting receipt (reverse for excess) |

- **Ledger role.** A new one (e.g. `TILL_CASH_AWAITING`, "Till cash awaiting receipt"), seeded and mapped like `SAFE_CASH`. I'd **not** reuse `CASH_IN_TRANSIT`: that one is the bank / CIT remittance leg (CRem / Con), and mixing them would muddle the remittance register's "age of cash in transit". Your call if you disagree.
- **The till cash invariant is unchanged.** Close still credits till cash. Only the safe moves later.
- **Safe balance, insured-limit checks and float issue** count received cash only.

### 2 · Data

Either add the receipt to the hand-over row, or give it its own row; your choice:

- **(a) Columns on `till_cash_movement`:** `tcm_received_by`, `tcm_received_on`, `tcm_received_amount`, `tcm_receive_count_id`, `tcm_receive_voucher_id` (+ acc year), with CHECK `tcm_received_by <> tcm_done_by`.
- **(b) A new kind `RECEIVE`**, pointing at the hand-over (`tcm_source_id`), with `done_by` = the supervisor and `witness_by` = the cashier.

### 3 · Routes

- `GET /till/movements/awaiting-receipt?companyId&branchId[&dayId]` lists the hand-overs not yet received. Each row: session no, counter, cashier, closed on, bag/seal if any, age. **The amount handed over is hidden from the receiver** (blind receive); show it only after the receive, as blind close does for the cashier.
- `POST /till/movements/receive` takes `{ tcmId, lines[] (denominations, as a count), receiverPin, notes }`.
  - The caller is the receiver, and needs a till PIN (notes 95).
  - The receiver is not the session's cashier: refuse self, unless `till.single_operator`.
  - **Right:** a new **Cash Office** menu (S9), Receive verb, or Till Sessions (273) OVERRIDE for now. Please tell me the menu id.
  - The response gives the amount received, the difference, and the variance id if one was raised.
- Optional: `POST /till/movements/receive` taking several `tcmId`s with one count each, so a supervisor can receive a stack of bags in one sitting.

### 4 · A receive difference

The difference becomes a variance on the **closing session**, new stage `RECEIVE`, reusing `till_variance`. It goes through the same approval rules (`CASH_VARIANCE`, tolerance, decide-variance APPROVE / treatment) as a close variance. The session stays CLOSED. Its `tss_variance_status` should show the open receive variance, so the Cockpit and Till Sessions list flag it.

### 5 · Settings and scope

- Add a setting `till.handover_receipt` = `REQUIRED` | `NONE` (company / branch). Leave `NONE` as the default, so shops that don't need this keep today's posting. The user wants `REQUIRED` for their company.
- **DROP.** A cashier's drop into the drop safe has the same gap. Should it follow the same receipt when the setting is `REQUIRED`? I'd say yes. PICKUP already has the supervisor's two signatures (notes 110), so it goes straight to the safe as now.
- **Day close.** While any hand-over of the business date is unreceived, `till/days/close` (when built) refuses, and the checklist shows it red.
- **Existing rows.** CLOSE_HANDOVERs posted before this: treat them as received (no back-fill), or say if you'd rather list them.

## Qt side, once the routes exist

- **Receive screen** (Cash Office): the awaiting-receipt list, then a blind count popup by denomination + the receiver's PIN.
- **Business Day (274):** a red row "Cash awaiting receipt (N, ₹—)" with a jump to the receive screen.
- **End Shift:** after close it will say "Hand the cash to a supervisor — it stays *awaiting receipt* until received."
- **Cockpit / Till Sessions:** a receipt state pill (AWAITING / RECEIVED / DIFFERENCE).
