---
name: stock-to-accounts-posting
description: "Under PERPETUAL an OPENING / PHYSICAL post writes an accounts voucher (STOCK, svh_voucher_type, svh_id) from the ledger rows; transfers post none; settle-short posts DR reason / CR INVENTORY; the opening-balance screen refuses INVENTORY"
metadata:
  node_type: memory
  type: project
  originSessionId: 04f8789e-47a6-43d6-8027-632e2629b5c4
  modified: 2026-09-28T06:08:49.654Z
---

Since 2026-09-28 (`src/modules/stocks/posting/stock-accounts-posting.service.ts`, called
from `StockPostingService.post/cancel`): when `accounts.cogs_mode` = PERPETUAL, OPENING
posts DR INVENTORY / CR OPENING_DIFFERENCE and PHYSICAL posts shortage DR reason ledger
(`srm_gl_ledger_id`, else role STOCK_SHORTAGE) / CR INVENTORY and excess the other way
(role STOCK_EXCESS), one voucher netted per ledger. Amounts are Σ `sml_cost_value` of the
rows just written, never the payload. Link: `avh_src_module='STOCK'`,
`avh_src_doc_type=svh_voucher_type`, `avh_src_doc_id=svh_id`, `avh_doc_refno=svh_refno`;
`avh_voucher_refno` is preset to the stock refno only when it fits varchar(50) (device-
scheme refnos do not, so the voucher then draws OPN0001-style from type 1/6). Cancel calls
`reverseLegs` (a `Rev` mirror, dated the original — a CANCELLED original stays in the
books beside it). Same-company transfers and sales-generated ISSUE/RECEIPT shadows post
NOTHING (the bill posts its own COGS pair) — except, since notes 153 (2026-10-26), a godown transfer's loading + unloading charges: one TRF voucher DR LOADING_CHARGES / CR LOADING_PAYABLE at despatch, under PERIODIC too ([[notes-153-godown-transfer-loading]]). Transit short-settle posts DR reason / CR
INVENTORY for short × `stt_cost_rate` as a Journal ('Jrl') linked to the OUT.

Roles STOCK_SHORTAGE / STOCK_EXCESS + catalogue rows + `vchr_affects_accounts` on types 1
and 6 + the TRANSIT_LOSS reason: migration 20260928100000. `accounts.fn_seed_ledger_map()`
creates the two ledgers on the next boot seed.

**Why:** every sale credited Stock-in-Hand and nothing debited it, so the ledger ran
negative and never equalled the valuation.

**How to apply:** `avh_user_id` is NOT NULL with an FK — the nil actor cannot post; the
document's `svh_created_by` stands in. The Opening Balance screen (menu 55) refuses a
non-zero row on the INVENTORY-mapped ledger under PERPETUAL. Go-live:
`POST /stock/admin/post-missing-vouchers?companyId&accYear` (idempotent).
`GET /stock/admin/balance-assertion` is §5.1 detect-only. Tests need
`test/helpers/stock-posting.factory.ts` (real accounts writer, cogs mode pinned).
