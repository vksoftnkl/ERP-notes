---
name: erp-expense-print-template
description: EXPENSE_A5 print template + Expense Voucher print wiring (2026-10-14) — 4 SQL datasets that redo the server's tax derive, the cancel/rewrite tender rule, and the F11 last-posted reprint on the screen
metadata:
  type: project
---

Built 2026-10-14 the same way as [[erp-receipt-print-template]]: `EXPENSE_A5`
("Expense Voucher A5", shared, purpose `EXPENSE_VOUCHER` 01a11b6f… in dev), published
rev 1, GLOBAL / PRINT assignment — DEV DB only. Handover files + tested deploy script:
`/home/vk/Dev/erp/docs/print-templates/expense-a5/` (not known to be applied on live).

Datasets `header`, `lines`, `taxes` (GST summary, GST bills only), `tenders`. Verified on
all 13 non-draft vouchers: lines = tenders = `avh_doc_amount`, tax = posted `INPUT_*` legs.

- **Lines come from `avh_draft_lines`** (kept after post); tax is recomputed exactly as
  `expense-derive.ts`: round(taxable × rate, 2) once, CGST = round(whole/2), SGST = rest,
  IGST when place of supply ≠ `comp_state_code`; pos = typed, else supplier GSTIN[0:2],
  else company. Place of supply / interstate are NULL/false without a GST bill.
- **Tender rule differs from the receipt:** expense cancel stamps tenders with its own
  `new Date()` a few ms AFTER `avh_status_on`, and post rewrites rows (a deleted
  pre-post row exists) — so CANCELLED = deleted rows with `td_modified_on >= avh_status_on`.
- `avh_remarks` = "paid to — notes" (joinRemarks, " — "); header splits it.
- Engine `fmt('0.##')` still prints 2 decimals (`#` counts as a fixed digit) — use `|num`.

Client: list toolbar Print (+ multi-select, drafts refused); screen **F11 Print** = the
voucher on screen if POSTED/CANCELLED, else the one this screen just posted (Post clears
the screen, so `save()` in `use-expense-voucher.ts` now returns the ExpensePayload, not a
boolean). Post itself does not open the dialog.
