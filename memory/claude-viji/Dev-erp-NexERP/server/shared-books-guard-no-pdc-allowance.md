---
name: shared-books-guard-no-pdc-allowance
description: "A party holding an un-matured post-dated cheque fails EVERY shared assertBooksReconcile call (sales bill post, retender, SO advance, cheque clear/bounce/deposit) until it matures"
metadata:
  node_type: memory
  type: project
  originSessionId: 56955dac-749e-4490-8721-ee15a5da424c
  modified: 2026-09-29T04:50:56.409Z
---

`fn_party_bill_reconcile` reads `ledger − bills = −(un-matured post-dated amount)` for any party
holding a post-dated cheque. The voucher counts on the ledger today, while its bill row counts only
on maturity. The shared `assertBooksReconcile` (reconcile/books-reconcile.guard.ts) refuses any
|diff| > 0.005, so every caller that puts that party in scope answers 422 ACC_PARTY_OUT_OF_BALANCE
until the cheque matures. Those callers: sale bill post/cancel (bill-lifecycle), retender,
sale-order advance, cheque clear/bounce/deposit/reissue, opening saves.

Only these allow the difference, via `vouchers/voucher-books.helper.ts` `assertVoucherBooksReconcile`:
the Voucher Register (notes 54), issued cheques, and receipt/payment post + cancel (notes 61,
2026-09-29). Verified 2026-09-29: with only the old receipt cancel restored, cancelling a plain
receipt of a party with another un-matured PDC receipt 422'd with a difference of exactly the cheque.

**Why:** until notes 61 a PDC receipt could not post at all, so customers rarely reached this state.
Now any customer who pays by post-dated cheque reaches it. Not fixed because it touches sales and
cheques modules. The user was told on 2026-09-29; the likely fix is to move the allowance into the
shared guard itself.

**How to apply:** a 422 ACC_PARTY_OUT_OF_BALANCE whose diff equals a live post-dated cheque's amount
is this, not a data defect. Check `abj_is_post_dated AND abj_adj_date > current_date` rows on the
party before running reconcile-books. See [[books-reconcile-trial-check]], [[payment-module]].
