---
name: bill-ledger-reconcile
description: Trial-mode rule (user, 2026-09-25) — every post checks bill-wise balance = ledger balance for bill-by-bill parties; the ledger side MUST include CANCELLED vouchers; notes 47 + the 3 defects its dry run found
metadata:
  type: project
---

Decided 2026-09-25: during trial, every post touching a `led_is_bill_by_bill`
party checks LEDGER (this year's acc_opening_balance + live legs of every
NON-DRAFT voucher) = BILLS (Σ signed abl_pending_amount, all years),
company-wide; a mismatch refuses the post (422), behind a company setting
switched off after go-live. Spec for Prathap: `share/sales/notes-47-bill-ledger-reconcile-on-post.md`.

**Why:** we are in trial and have no other way to catch posting bugs that
leave the two books apart.

**How to apply:** the ledger side MUST count CANCELLED headers — a cancel keeps
the original legs live and posts a POSTED mirror; leaving CANCELLED out makes
every cancelled bill a false mismatch. Only bill-by-bill ledgers (user's scope).

The first dry run found three server defects: D1, a bill's PARTIAL counter
tender is not written to its abl row (bil00685, bil00695); D2, the opening
sign differs between acc_opening_balance and the opening bill (Freshora); D3,
a bill cancel's mirror voucher takes the NEXT sale-bill number (bil00694,
bil00596), which leaves holes in the invoice series.

Related: [[sale-bill-upgrade]], [[opening-balance-concept]], [[offline-sync-invariants]]

Addendum (same day): C2 Cheques In Hand ledger = register HELD+DEPOSITED
(ON_RECEIPT only), dry run diff 2,000 = bil00696's unregistered cheque. C3
(voucher balances) DROPPED — the DB already enforces Dr = Cr
([[voucher-balanced-check-vacuous]]). Receipts need only C1 + C2.

Addendum 2: advance set-off removed on amend / cancel → advance restored:
verified OK on the current build (harness mode `advamend`). The user's
bil00702 kept 1000 because the SCREEN sent it (audit.audit_log
log_modified_record holds the exact amend payload — use it to settle "what did
the client send"). C1 can't catch that; proposed C4 = header vs own rows
(sbTenderAmt ≠ Σ tender rows on 14 posted bills).

**Notes 47 deployed 2026-09-25:** fn_party_bill_reconcile / fn_cheques_in_hand_reconcile
/ fn_books_reconcile(company, year) + setting accounts.reconcile_on_post (true).
D1 fixed for new bills; the old SATHEESH / Freshora rows were not repaired, so
those two parties' posts will be refused. D3 is NOT fixed (bil00713, bil00714
burned). C4 is unconfirmed. Run `select * from accounts.fn_books_reconcile(co, year)
where abs(diff)>0.005` to see the drift.

**Tender excess releases set-off (client, same day):** typing more counter money
than is due while a credit is set off now gives the set-off back first
(SaleBillTender::releaseAdjustedForExcess → PartyAdjustPanel::releaseAdjusted,
newest first); change only after that. Root cause of the user's "advance stays
2000" (bil00702, bil00715): they typed full cash with the set-off still on, and
the 300 became change.

**Notes 49 verified 2026-09-25 17:50:** D3 FIXED (cancel posts `rev000nn`,
voucher type 23, with no bill-series gap; 224 old gaps remain). The old D1 rows
are repaired, and fn_books_reconcile returns 0 rows company-wide. Bounce sign
fixed. **C4 NOT in** (validate accepts sbTenderAmt 500 vs rows 118). Loyalty
amend is still unverifiable.

**2026-09-25 evening:** D3 is FIXED by Prathap's migration 20260925150000_reversal_voucher_type. Every reversal now uses type `Rev` (id 23 on the box, series rev00001…, dated the ORIGINAL's date), and the old mirrors were renumbered into it. D2's root cause, `fn_ledger_book_balance` testing 'DR' against a 'D'/'C' column, is fixed in 20260925130000.
