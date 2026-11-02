---
name: payment-module
description: "/payments/* (menu 100) live 2026-09-29 — the receipt mirrored; Pmt type, grid 123, tdAmount includes the bank charge; notes 61 ported its fixes to receipts, notes 62 review applied"
metadata:
  node_type: memory
  type: project
  originSessionId: 69f8d120-dca7-4e3e-92c1-a0d4f1fcd266
  modified: 2026-09-29T02:41:17.277Z
---

`src/modules/accountsModule/payment/` = `receipt/` mirrored, money OUT (plan rev 2,
2026-09-28). Migration `20260929090000_payment_voucher` (applied to dev by psql +
`prisma migrate resolve --applied`, because `20260928210000_stock_bucket_move` of another session
was pending): voucher type `Pmt` (resolved BY CODE; id 33 on dev), roles ADVANCE_PAID /
DISCOUNT_RECEIVED / BALANCES_WRITTEN_BACK in a new `PAYMENT` alr_group and in
`fn_ledger_map_catalogue()` (42 roles), setting `accounts.payment_salesman_mandatory`, menu 100
visible, grid **123** 'MAIN LIST - PAYMENTS'. Engine: `AllocationInput.direction` + result
`partyLegSide`; OUT spec in `receipt/allocation-engine.out.spec.ts`.

Non-obvious: **`tdAmount` INCLUDES the bank charge** (what leaves our bank; party gets
`tdAmount − tdMdrAmt`). ADVANCE_PAID is mapped but nothing posts to it (the remainder stays in the
party leg, as the receipt's ADVANCE_RECEIVED). tester1 has no user_menus rights on menu 52, so
e2e tests cannot drive `/issued-cheques/*` without a rights fixture.

**Why:** the three defects the payment found were ALSO in the receipt; notes (61), 2026-09-29,
ported all three (`/receipts/post` + both cancels → `assertVoucherBooksReconcile`; `/receipts/adjacent`
binds `::date`, see [[db-session-timezone-ist]]; `/receipts/amend` returns the post-bump header).
Test: `test/receipt-payment-parity.e2e-spec.ts` (debtor party on the payment harness).

notes (62), 2026-09-29, same day: PPD comes from the SUPPLIER (`sup_cash_disc_perc` within
`sup_credit_days`, `sup_id` = ledger id), never `accounts.ppd_slabs`; the engine lets an unpinned
settling deduction overflow to on account (TDS on an advance); TDS base = tenders − every DR line;
a keyed TDS line on an unflagged party is a 400; amend CANCELS old leaves (never soft-deletes);
summary names are the receipt's (`creditsHeld`, `totalCredits`); a DR ROUND_OFF other-line is how
a payment rounds UP; menu rights enforced, see [[receipt-payment-menu-rights]].

**How to apply:** when one of receipt/payment is fixed, check the twin — they are copies. e2e
harness: `test/helpers/payment-e2e.ts` (own party + opening row); subclass `PaymentFixtures` for a
Sundry Debtors party. See [[books-reconcile-trial-check]], [[shared-books-guard-no-pdc-allowance]],
[[issued-cheques-design]].
