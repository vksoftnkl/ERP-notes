# Receipt — `POST /receipts/delete`, the route that is missing (2026-09-17)

**For Prathap.** One route. It is graded A in `gap-vs-3.0.md` because it is a
daily operation with no workaround at all — not a slow path, not an ugly path,
none.

Found while building the receipt register (menu 99, `TxnDocTypes::receipt()`)
by calling the live server, not by reading the plan.

---

## 1. What is wrong

A DRAFT receipt cannot be got rid of. Ever.

    POST /receipts/cancel   on a DRAFT   ->  409
      "01a0adea-d2ee-… is DRAFT. Only a POSTED receipt is cancelled;
       a DRAFT is simply not posted."

and there is no delete route to fall back on — the whole surface is
`open-items`, `party-context`, `get`, `create`, `post`, `update-header`,
`cancel`, `regularise-pdc`.

That refusal is **correct**: cancelling means posting a reversal, and there is
nothing to reverse. The gap is that nothing else was built to take its place.

So every abandoned draft is permanent. On Acme Foods on 2026-09-17, keyed only
by testing:

| status | count |
|---|---|
| POSTED | 28 |
| **DRAFT** | **7** |
| CANCELLED | 1 |

Seven that will still be there in five years. A real branch abandons drafts all
day — the customer changes their mind at the counter, the cheque is not in the
bag, the phone rings — and each one becomes a permanent row in the register
that an operator has to learn to read past.

---

## 2. Why deleting a draft is safe

R10 already settled it, in the plan's own words: **a draft touches no bill.**
Concretely, a DRAFT receipt owns:

| written at create | written at post |
|---|---|
| `acc_voucher_header` (DRAFT, **no number, no slno**) | `avh_voucher_no` + `avh_voucher_slno` |
| `acc_tender_detail` rows, `td_voucher_id` **NULL** | `td_voucher_id` filled in |
| `avh_draft_lines` (the other-ledger lines) | `acc_vouchers` legs |
| — | `acc_bill_adjustment` rows |
| — | `acc_pdc_register` rows |
| — | the ADVANCE bill |

**Finding a draft's tender rows.** Not by `td_voucher_id` — it is NULL until
post, which is the whole point. They are reached by the source pair:

```sql
WHERE td_src_module  = 'ACCOUNTS'
  AND td_src_doc_type = 'RECEIPT'
  AND td_src_doc_id   = :avhVoucherId
  AND td_is_deleted   = false
```

Verified 2026-09-17 on draft `01a0adea-d2ee…`: four cheque tender rows, all
with `td_voucher_id` NULL and `td_src_doc_id` pointing back at the header.
Live count of rows in that state: 11 for RECEIPT, 16 for SALE_BILL — so the
sale bill has the same shape and whatever is written here should read the same
way there.

Everything in the right-hand column is a post-time write. **A draft carrying
four cheque tender rows still writes nothing to `acc_pdc_register`** — verified
2026-09-17 by creating exactly that draft and watching the register stay empty
until `/post` ran. So there is no cheque cleanup, no adjustment reversal, no
bill to reopen and no number to burn.

It is the one end of the lifecycle where a hard delete leaves nothing behind.

---

## 3. The route

```jsonc
POST /api/v1/receipts/delete
{ "avhVoucherId": "…", "avhCompanyId": "…",
  "avhBranchId": "…",  "avhAccYear": "2026-2027" }
```

The four keys and nothing else. **No `reason`** — a reason is for a reversal
that stays in the books; a draft leaves no trace to annotate, and asking for
one is a field the operator has to fill in to throw away a blank form.

In one transaction:

1. Lock the header `FOR UPDATE`.
2. **Refuse anything that is not DRAFT**, with the same shape `/cancel` uses in
   reverse — POSTED gets *"is POSTED. A posted receipt is cancelled, not
   deleted"*, CANCELLED gets *"is already cancelled"*. 409, field
   `avhVoucherId`.
3. Soft-delete the `acc_tender_detail` rows, then `avh_draft_lines`, then the
   header. Soft (`*_is_deleted = true`), not hard: the tables carry the flag
   everywhere else and a partitioned hard delete is the one that finds a
   foreign key nobody remembered.
4. `txn_status_log`: DRAFT → DELETED, with who and when. The row is invisible
   to the register afterwards but the log still says a draft was raised at 4pm
   and dropped at 4:02 — which is the only thing anybody would ever ask.

Returns `{ avhVoucherId, avhAccYear }` and a message naming what went.

**Guard:** the grid SELECT and `/receipts/get` must both filter
`avh_is_deleted = false`. Worth checking they already do — a deleted draft
reappearing in the register is the failure mode this route would otherwise
introduce.

---

## 4. What the client does the day it lands

One branch in `receiptRowPolicy()`
(`src/modules/accounts/common/txn_doc_types_accounts.cpp`) goes away.

F3 on the register is already two verbs behind one key, which is what an
operator expects:

| row status | F3 means | today |
|---|---|---|
| DRAFT | **delete** | greyed, with a reason that says there is no route |
| POSTED | cancel (reversal + reason) | works |
| CANCELLED / — | nothing | greyed |

The entry screen needs nothing: `btnCancelVoucher` is already
`setEnabled(posted && …)`, so it has never offered the doomed call.

Until the route exists the register greys F3 on a draft and says so, rather
than firing a call that is guaranteed to come back 409 — a button that always
fails is worse than a button that is not there.

---

## 5. One decision to take with it

**R19 — abandoning a draft.** Delete it, or leave it and let the register filter
drafts out by default?

Recommend **delete**.

One argument for it that does NOT hold, so nobody spends a morning on it: a
draft does **not** burn a voucher number. Checked 2026-09-17 — of 35 receipts
on Acme Foods, the 28 POSTED and the 1 CANCELLED each carry a number and an
`avh_voucher_slno`, and all 7 DRAFTs carry neither. The numbering is allocated
at post, exactly as §4.3 says, so abandoned drafts leave no gap in the
sequence and there is no numbering reason to clear them.

The reason that does hold is simpler: filtering only moves the pile somewhere
the operator cannot see it, and it still grows. Every future "why are there
400 receipts this month" starts here, the party's `lastReceipts` panel has to
learn to skip them, and a register that quietly hides rows is a register
somebody eventually stops trusting. A draft is a scrap of paper, and the
answer to a scrap of paper is a bin.
