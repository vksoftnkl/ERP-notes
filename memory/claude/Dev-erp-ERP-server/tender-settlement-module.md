---
name: tender-settlement-module
description: Non-cash tender control (share file 49 plan) built 2026-10-08 — /tender-settlement/* menu 278, card last-4 rule in the tender writer, UPI skipped, OVERRIDE stands in for approvals; tnd_settlement_ledger_id means two things
metadata:
  type: project
---

`src/modules/accountsModule/tenderSettlement` (its README) + `TenderDetailService.guardReference` +
`till/services/till-slip-check.service.ts`; migration `20261008190000_tender_settlement` (menu 278
hidden, grids 138-143). Built on 2026-10-08 from the pasted plan-noncash-tender-control.md (not on disk).

The user's calls (2026-10-08):
- Card money-in rows need the last 4 digits. A 4-digit `td_ref_no` (what today's client keys) is
  accepted and copied into `td_card_last4`.
- UPI is "skip for now" (rush hours): no ref rule and no duplicate check. The `noRefCount` per tender
  on the session payload is the watch figure instead.
- OVERRIDE on menu 278 stands in for the NONCASH_WRITE_OFF and SETTLEMENT_RESOLVE approvals until
  till phase 3.
- The statement import is CSV only, driven by `tnd_statement_format` (set via `POST /tender-settlement/format`;
  the Tender master does not carry it).

Non-obvious facts:
- `tnd_settlement_ledger_id` means TWO things. The receipt and voucher register treat it as the
  instrument's clearing ledger and debit it directly. Plan 49 treats it as the bank the payout reaches.
  TSet therefore credits `COALESCE(td_settle_ledger_id, td_tender_ledger_id)` (what the row's own leg
  debited), which nets correctly either way. Never "fix" it to `tnd_ledger_id`.
- Since notes 99 §2 (2026-10-09) TSet writes `td_mdr_amt` = line fee + tax ONLY on sales rows
  (`MDR_FROM_STATEMENT_DOC_TYPES` SALE_BILL/SALE_RETURN/SALES_ORDER) and a void zeroes it. Never on
  receipt/payment/expense rows: the receipt rebuilds its BANK_CHARGES leg from that column on an amend
  (and receipt amend has no SETTLED guard), so the fee would be booked twice.
- Notes 99 (migration 20261009170000): grid `TENDER SETTLEMENT - PICK ROW` (153 on dev; tokens
  itd_company_id, itd_branch_id, itender_id, iline_kind, ifrom_date, ito_date) is the Match/LINKED
  picker — grid 140 only lists overdue money-in rows; grid 140 now treats a NULL expected date as the
  doc date; dropdown `INCOME LEDGERS` (69) mirrors resolve INCOME (group nature Income, any level).
- TENDER_SUSPENSE is the house's only suspense role (ledger "Till Variance Suspense"), shared by the
  till variance and the TSet by design — confirmed for notes 99 §8, no separate Tender Suspense ledger.
- `ux_asl_td_matched` ignores the import's status. A void must release its lines to UNMATCHED (the old
  link goes into `asl_notes`), or a re-import cannot match the same rows.
- Multipart DTOs with a Swagger-only `file?: unknown` need `@IsOptional()`. Under
  forbidNonWhitelisted the own `undefined` field (see [[dto-hasownproperty-always-true]]) is refused
  with "property file should not exist".
- Not built: the closed-session pairing (`asl_tvr_*`), and the NONCASH_VARIANCE decision at close.
  Both wait for till phase 3. See [[till-management-schema]].
