---
name: money-lines-tender-notes-126
description: "Notes 126 (2026-10-16) — every CASH/BANK leg carries an acc_tender_detail row; coverMoneyLines after every leg writer (SALES only reported); server rows td_src_doc_type 'VOUCHER' that till readers must skip; register §6 money side = tender; reports/tender-day-book"
metadata:
  node_type: memory
  type: project
  originSessionId: d006fa32-cd91-4155-862e-f20e41c74d33
  modified: 2026-10-16T06:27:11.598Z
---

Built 2026-10-16 (migrations 20261016120000 + 130000, both applied on the dev box):
- `src/common/posting/money-lines.ts` `coverMoneyLines` / `coverPostedVoucher`: after legs are written,
  CASH/BANK lines (led_ledger_type) short of tender rows get a server row (CASH = branch CASH tender;
  BANK only with `bankTender` BANK|CHEQUE or a document-named tender) else 422
  VCH_MONEY_LINE_NO_TENDER / VCH_NO_TENDER_FOR_LEDGER. SALES src module = never refused (user's
  counter rule, option 1). Hooked: postLegs.insertLegs (unless `money.deferred`), register 9d,
  receipt/payment after linkInstruments, writeChequeVoucher (CHEQUE), order advance, till (BANK), TSet.
- Coverage match: td_voucher_id / td_src_doc_id = voucher, document rows (td_src_doc_id = avh_src_doc_id,
  td_voucher_id NULL), retender (td_replaces_id = src + voided row flipped), and a row covers its
  td_settle_ledger_id too (card/UPI payments out post to the settlement bank).
- Server rows: td_src_doc_type 'VOUCHER', td_src_module 'TILL' for till transfers. EVERY reader that
  sums tender rows by session or ledger must skip 'VOUCHER' (expected(), safe flows, slip check, grid 158
  done).
- Register §6: RcpV/PmtV typed money line → VCH_MONEY_LINE_NO_TENDER; Contra lines need `tenderId`
  (CASH/BANK only); Jrl/DrN/CrN/PurA/SalA → VCH_MONEY_LEDGER_NOT_ALLOWED. Till branch: Jrl/Con never on
  till cash/safe/awaiting/in-transit (TILL_CASH_LEDGER_DIRECT, any device). No device = back office.
- Tender master: type↔ledger fit (TENDER_LEDGER_TYPE / _REQUIRED); tnd_ledger_id nullable only for
  TEMP_CR/CREDIT/LOYALTY. Dev data: CARD→Card Settlement Clearing, WALLET→Wallet Clearing (settle Kvb),
  credit types unlinked, "Cashin Hand" inactive.
- Report 2 first run (since 10-08): 403 uncovered lines (till transfers, Rev, ChqBnc, test-residue PmtV).
- Regression found + fixed same day: a CHEQUE label row DR on a BANK ledger (clearing, issued-cheque
  stop/return) made `fn_is_cheques_in_hand_ledger` true → books check refused with
  ACC_CHEQUES_OUT_OF_BALANCE on Kvb. Migration 20261016140000 skips td_src_doc_type 'VOUCHER' there.
  Any new global reader of tender rows must skip 'VOUCHER' too.
- No-device RcpV cash posts to the default safe in a till branch, so receipt/register e2e suites now
  need the ZTS default safe — run them BEFORE expense/tender-settlement suites ([[e2e-suites-steal-default-safe]]).
  /validate previews the tender's own ledger (does not route) — open gap.

Notes 128 (2026-10-16, user's counter rule confirmed): SALES never refused (sale bill / return / re-tender /
order advance all post avh_src_module SALES); accounts vouchers refused. §2 split TENDER_LEDGER_TYPE into
TENDER_CASH_LEDGER_NOT_CASH / _CLEARING_LEDGER_IS_MONEY / _BANK_LEDGER_NOT_BANK / _CHEQUE_LEDGER_IS_MONEY +
TENDER_LEDGER_NOT_ALLOWED (a ledger on TEMP_CR/CREDIT/LOYALTY). §3 audit: every cross-document reader
(drawer expected, slip check, safe flows, 40A(3), grids, print datasets, cheque fns) already filtered;
fixed the per-voucher readers that took ALL rows on a voucher id (receipt/payment rebuildLines,
tenderIdsByRow, loadTenderPayload → tdSrcDocType RECEIPT/PAYMENT; register loadInstruments <> 'VOUCHER')
and receipt/payment amend now call retireMoneyLineRows. Reversal mirrors stay unjudged: the report covers a
mirror by its original's rows, so refusing would only block cancelling pre-126 vouchers.

Notes 129 §4 (user's call): RRN (type 7) = a bill settled against a sale return's credit note, NO money —
`ON_ACCOUNT_TENDER_TYPES` (sales-doc.utils: CREDIT, TEMP_CREDIT, RRN) drives isCreditTender, the counter
allocation's NOT_SETTLING and re-tender; tender master NO_LEDGER [7,8,9,10]; migration 20261016150000 cleared the
RRN master's ledger (was SALES FREIGHT COLLECTION, expense) + ck_tnd_ledger_required. §5: user said NO back-fill of
the ~400 pre-126 gaps — they stay on the no-tender report. Open data issue flagged, not fixed: the CARD tender's
surcharge ledger is Cash In Hand and UPI's is "new customer" (a party).

Notes 130: grid 119 (POPUP - VOUCHER LEDGERS) + GET /vouchers/ledger-pick offer NO cash / bank ledger on any type
(led_ledger_type CASH/BANK or under the money groups — the derive's isMoneyLedger) and NOTHING on RcpV DR / PmtV CR;
Con lists nothing (Qt uses the tender list). Migration 20261016160000 + prisma/seed/Grid_Details.sql (seed is
ON CONFLICT DO NOTHING, so a grid change always needs both). "share/grids/voucher_ledger_picker.sql" exists on no
repo here — the seed is the source.

**Why:** user wants the Day Book by tender to find missing transactions.
**How to apply:** a new leg writer must call coverPostedVoucher (or go through postLegs). Related:
[[till-notes-124-open-list]], [[books-reconcile-trial-check]].
