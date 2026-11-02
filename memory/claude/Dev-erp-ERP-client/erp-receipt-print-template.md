---
name: erp-receipt-print-template
description: "RECEIPT_A5 print template (2026-10-14) — its 5 SQL datasets, the row-selection rules that make cancelled/amended/PDC receipts reconcile, and two print-engine limits found building it"
metadata:
  node_type: memory
  type: project
  originSessionId: 6c807bfc-7fc5-48fd-baeb-30259aabc660
  modified: 2026-10-14T05:30:31.651Z
---

`RECEIPT_A5` ("Receipt A5", shared, ptl_company_id NULL) is the `RECEIPT_VOUCHER`
design, published rev 1 and assigned GLOBAL / PRINT in the DEV DB only (live needs
the rows copied — see [[erp-live-config-drift]]). Saved through
`POST /print-templates/create` with the full header (see
[[erp-print-template-save-clears-header]]).

Files: `/home/vk/Dev/erp/docs/print-templates/receipt-a5/` — `receipt-a5-deploy.sql`
(DO block: purpose found by CODE, fresh ids, approver = `admin` login, skips if
RECEIPT_A5 exists or a GLOBAL PRINT assignment exists; dry-run tested on dev in a
rolled-back transaction), `datasets/*.sql`, `receipt-a5.json`. Handed to the user
2026-10-14 for live (live had the purpose as 01a04859…, no RECEIPT_A5, identical
columns); not known to be applied yet.

Datasets: `header` (MASTER, all totals pre-computed), `instruments`, `bills`,
`credits`, `adjustments`. Verified against all 167 non-draft receipts: received +
role-DR legs + credits = settled + on-account + role-CR legs, every detail sum
equals its header total, and today's money = `avh_doc_amount` on every POSTED one.

Row-selection rules (each one broke reconciliation until applied):
- **PDC vouchers = the `td_voucher_id`s of the receipt's own tender rows**, NOT
  `avh_against_voucher_id`: a cancel's REVERSAL voucher and re-presented-cheque
  vouchers carry the same type + against link (rct00020/29 double-counted).
  The server's `receiptPdcVoucherWhere` has the same over-match.
- **A cancel soft-deletes tenders and ADVANCE bills stamped with exactly
  `avh_status_on`**, so `td_is_deleted AND td_modified_on = avh_status_on` gives
  a cancelled receipt's instruments back.
- Adjustments: drop `abj_reversal_of_id` rows and rows retracted by a reversal on
  the SAME voucher (amend); a cancel files negatives on the reversal voucher, so
  the cancelled receipt still lists what it settled. CR rows = bills settled; DR
  rows with `abj_against_bill_id` = credit notes/advances used.
- `td_amount` INCLUDES the card/UPI surcharge; BANK_CHARGES legs are excluded.

Engine limits found: the `date` transform formats every timestamp in UTC (IST
times print 5h30 early, after-midnight dates land on the previous day) — format
timestamps in SQL `to_char` instead (DB zone is Asia/Kolkata); a ':' inside a SQL
literal is refused by `ck_ptd_sql_no_quoted_param`, hence `chr(58)`. GROUP_HEADER
bands do not repeat on a new page. Related: [[erp-receipt-entry-port]],
[[erp-document-print-button]].
