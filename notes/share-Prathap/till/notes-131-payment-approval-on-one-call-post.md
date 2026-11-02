# Notes 131: the CASH_PAYMENT approval on Payment's one-call post (2026-10-16)

**STATUS:** DONE by Prathap (deployed 2026-10-16; swagger shows approvalId on SaveDraftPaymentDto). Qt Payment screen wired the same day. Receipts: no approval event, never 428 — nothing to do. Follows notes 124 §5 / 106.

## Why

Qt now handles **428 TILL_APPROVAL_REQUIRED** on these screens:
- **Expense Voucher (277):** post → approval popup → post again with `approvalId`.
- **Open Till (272), FLOAT_MISMATCH:** open → popup → open again with `approvalId`.

The popup offers "approver here + PIN" (`approvals/request` with `approverId`), or "send to inbox". The approvers' inbox is on Till Sessions (273).

**Payment (100) can't do the same.** Since notes 106 it posts in ONE call: `/payments/create` with `post: true`. But `approvalId` exists only on `PostPaymentDto` (`/payments/post`). And for a NEW payment there is no voucher id yet to name as `srcDocId` in the request.

## Ask

1. Accept `approvalId` on `/payments/create` when `post: true` (same meaning as on `/payments/post`).
2. When that call answers 428 TILL_APPROVAL_REQUIRED, either:
   - (a) keep the payment as a DRAFT and put its `voucherId` in the error (`srcDocType: PAYMENT`, `srcDocId`), so the request can name it; **or**
   - (b) let the request name the payment by something the client already has.

   Your call. (a) looks simplest, and it is what the expense does: its draft is saved before the post.
3. The same check for `/receipts/create` with `post: true`, if any receipt rule can answer 428. If none can, say so.

Under `till.approval_gate` WARN (as shipped) nothing changes: the payment posts and the request waits in the inbox.

Tell Viji when it is deployed. Claude will wire the Payment screen the same way and test it with a company on REFUSE.
