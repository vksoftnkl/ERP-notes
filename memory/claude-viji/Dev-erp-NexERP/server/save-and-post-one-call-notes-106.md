---
name: save-and-post-one-call-notes-106
description: "Notes 94 B / 106 built 2026-10-12 — /create with post:true (or the status the post leaves) saves and posts in ONE transaction on bills, orders, challans, receipts, payments and physical stock; a refusal leaves no row and burns no number; receipts/payments judge it on the Post right (94 A)"
metadata:
  type: project
---

Every `/create` of the six documents takes `post: true`, and the module's own posted status is the
same instruction (`sbStatus: 'POSTED'`, `soStatus: 'CONFIRMED'`, `sdcStatus: 'POSTED'`,
`header.status: 'POSTED'`). The draft write and the whole of `/post` run in one transaction:
`BillLifecycleService.saveAndPost` / `postDraftInTx`, `SaleOrderLifecycleService.saveAndPost` /
`postInTx`, `DeliveryChallanService.saveAndPost` / `postDraftInTx`, `ReceiptPostingService` and
`PaymentPostingService.saveAndPost` (composing `saveInTransaction` + `postInTransaction` like
`/amend`), and the stock voucher save's existing `postAfterSave`. The number comes from a row in
`acc_voucher_seq` updated inside the transaction, so a rollback returns it. `/post` stays.

**Why:** the Qt Save & Post was `/create` (number issued) then `/post`; a refusal `/validate` had
not caught left a draft the user never asked for and a burned number (eleven on 2026-10-12). The
user: "i want to stop to create DRAFT". Receipts/payments judge the one-call on the Post right
alone (notes 94 A, VIGNESH had Post without Create); the sales routes never checked Create.

**How to apply:** a one-call refusal must be exactly `/post`'s, so never add a guard to the
one-call path that `/post` lacks — extend the shared `post*InTx`. `soStatus: 'CONFIRMED'` on a
create no longer writes the column directly (it confirms through the lifecycle); other `soStatus`
values on a save still behave as before. Receipts/payments need `onAccount` with `post: true`
(400 on the field). The sales e2e harness now owns device `E2E-SALES-HARNESS` (created/un-deleted
at `bootApp`; `DEVICE_ID` is a live `let`, read after boot) because the real 'raman' device was
deleted through the screen on 2026-10-12. A suite that boots its OWN app but posts through the harness helpers must call
`useHarnessDevice(prisma)` first, or it posts on the zero-uuid placeholder (404 Device not found) —
till-money-paths and till-counter-claim broke that way until 2026-10-12. **Notes 131 exception (payments, 2026-10-16):** 428 TILL_APPROVAL_REQUIRED (CASH_PAYMENT gate,
till.approval_gate REFUSE) KEEPS the draft — `saveAndPost` takes `SAVEPOINT payment_one_call_post`
after `saveInTransaction`, rolls the post back to it on that code only, commits the draft and throws
the 428 naming it (srcDocId = avhVoucherId); `approvalId` rides on SaveDraftPaymentDto. Receipts
have no gate (no receipt rule answers 428). Test: `test/payment-approval-gate-http` (cashier `tester`).
Tests: `test/sales/save-and-post-http`,
`test/payment-save-and-post`, `test/physical-save-and-post-http`. Related:
[[pre-existing-test-failures]], [[till-management-schema]], [[user-menu-transaction-rights]].
