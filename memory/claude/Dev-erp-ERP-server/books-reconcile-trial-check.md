---
name: books-reconcile-trial-check
description: Since 2026-09-25 every party/cheque post runs a books check that 422s on mismatch (accounts.reconcile_on_post) — a pre-existing mismatch blocks all that party's posts
metadata:
  type: project
---

notes (47), live 2026-09-25: `assertBooksReconcile` (src/modules/accountsModule/reconcile/) runs LAST in sale bill post/cancel/amend/retender, order advance, receipt post/cancel/amend, all /cheques verbs, opening saves. It refuses 422 ACC_PARTY_OUT_OF_BALANCE (bill-by-bill party: ledger ≠ open bills) or ACC_CHEQUES_OUT_OF_BALANCE (Cheques In Hand ≠ HELD+DEPOSITED register). Setting `accounts.reconcile_on_post`, COMPANY scope, default true; meant to go OFF at go-live.

**Why:** the user wants defects to surface at the counter during the trial, not at year end.

**How to apply:** a 422 with these codes in a test or at the counter is usually an OLD mismatch on that party, not the post in hand — run `npx tsx scripts/reconcile-books.ts` first. As of 2026-09-25 afternoon every party on the box reconciles (D1 fixed and backfilled, [[sale-bill-alloc-is-seeded]]). "D2" was never data: fn_ledger_book_balance tested op_dr_cr = 'DR' but the column is 'D'/'C' (fixed by migration 20260925130000). An e2e fixture that writes acc_bill_balance / vouchers directly must add a COMPANY app_setting_value override `accounts.reconcile_on_post = false` for its throwaway company (done in test/opening-balance-http.e2e-spec.ts) — never weaken the check.

**Post-dated cheques (2026-09-29):** the shared `assertBooksReconcile` has NO allowance for a
post-dated voucher (POSTED today, so the ledger counts it; its bill row counts only on maturity),
so a receipt with a post-dated cheque 422s. `vouchers/voucher-books.helper.ts`
`assertVoucherBooksReconcile` allows exactly the un-matured difference; receipt and payment post AND
cancel use it since notes (61). Every other caller still uses the shared guard — see
[[shared-books-guard-no-pdc-allowance]]. **Fixture pattern that keeps the check ON:** give each
fixture party ONE `acc_opening_balance` row (unique per company/branch/year/ledger; `op_dr_cr` is
'D'/'C') equal to the NET of the OPENING bills you create on it — `test/helpers/payment-e2e.ts`.
