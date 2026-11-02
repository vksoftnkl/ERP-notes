---
name: voucher-register-module
description: The Voucher Register (/vouchers/*) went in 2026-09-26 on the shared posting layer — where its choices departed from the plan and the traps its e2e hit
metadata:
  node_type: memory
  type: project
  originSessionId: 0e035175-bfb5-45ff-a049-1f0d0ce8f144
  modified: 2026-09-26T06:19:38.986Z
---

Built 2026-09-26 under `src/modules/accountsModule/vouchers/` from the
"Voucher Register: the whole design in one file" plan (2026-09-25). The DDL
(20260925160000), Rev type (20260925150000) and tds_rates (20260925170000)
were already live; this session added the module, migration
`20260926100000_voucher_register_grids` (grids 117/118, ui_tables 39/40), the
unit spec `voucher-derive.spec.ts` and `test/vouchers-register-http.e2e-spec.ts`.

**Decisions the plan left open (all in the module README):**
- RAISE types raise the bill for the FULL party amount and pair allocations
  (raised ↔ existing) — `ck_abj_against` only lets ADVANCE_ADJUST / NOTE_ADJUST /
  TRANSFER carry an opposite bill. DEMAND types write single ALLOCATION rows.
  `abl_alloc_amount` is re-derived by `BillBalanceRecomputeService`, never incremented.
- `gdr_doc_type` has no PURCHASE/SALES member: PurA/SalA write INVOICE with
  `gdr_tran_nature` PURCHASE/SALE; `RegisterTranNature` gained PURCHASE.
- Instrument-controlled ledgers exclude a tender's ledger only when it is NOT
  under Cash-in-Hand / Bank Accounts / Bank OD (on the box UPI/CARD/BANK tenders
  all point at "Kvb Current A/c", which a Contra must move).
- PmtV TDS grosses up the typed net bank line (`net / (1 − r)`); a zero TDS base
  is silent, only a base under threshold WARNs `VCH_TDS_BELOW_THRESHOLD`.
- `acc_tds_register.atd_deductee_type` is COMPANY | NON_COMPANY (not the ledger's six).
- Period lock is judged on the voucher DATE (`fy_lock_date >= date`), year status must be OPEN.

**Why:** these are the seams the client and the Payment module will build on;
changing any of them later means reversing posted rows.

**How to apply:** a live DRAFT holds `ux_avh_doc_refno` — an e2e that creates a
draft with a supplier bill number then posts the same number gets
`VCH_DUP_DOC_REFNO` (correct, and it cost a run). tester1 has NO rows on menus
101–104/163/259–262; the spec grants and revokes them. The shipped 194C
thresholds (30,000 single) mean the PNG example deducts nothing unless a
company-scoped 0-threshold row exists (the spec seeds one for Acme, permanent).
RCM_* roles were unmapped; the spec maps them company-scoped to E2E ledgers.
See [[sales-e2e-specs-rights-fixture]], [[books-reconcile-trial-check]],
[[money-decimal-rule-sales]].

**notes (54), 2026-09-26 evening:** `vchr_instruments` (migration
20260926120000) — a RcpV line carries `instrument{tenderId, refNo,
instrumentDate, bankName, cheque}`; the Dr leg is generated from the tender's
ledger, a cheque becomes a HELD `acc_pdc_register` row, a post-dated one gets
its own voucher (`avh_against_voucher_id`), all through `TenderDetailService`.
The instrument is OPTIONAL per line (a sibling session's notes-53 test types a
Dr cash line the old way). `fn_ledger_book_balance` is date-blind, so a
post-dated voucher would fail `assertBooksReconcile`; the register runs its own
`voucher-books.helper.ts` that allows exactly the un-matured post-dated rows.
`/cheques/bounce` takes no `bankLedgerId`; `BounceChequeDto` = keys + bounceDate,
reason, bankCharge, partyCharge. `tnd_needs_ref` on the cheque tender is false on
the box (the type's flag says true) — a cheque is still required to carry its
number. ESLint 9 was installed by a sibling with only `.eslintrc.js` present:
run `ESLINT_USE_FLAT_CONFIG=false npx eslint …`.
