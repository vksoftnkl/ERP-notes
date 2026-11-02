---
name: sales-posting-chain-findings-2026-09-22
description: "Facts the sales e2e proof surfaced that the schema forces — shadow stock voucher needs a registered device uuid, DC/SR/DCR counter columns are NOT NULL, a reversal voucher takes its own number, ledger map rows are global"
metadata: 
  node_type: memory
  type: project
  originSessionId: e9c9b22e-6446-4278-a2b8-28a014a35bbc
  modified: 2026-09-22T11:04:13.302Z
---

Learned proving the A3–A5 chain on 2026-09-22 (fixed in code, nothing migrated):

- `stock.stock_voucher.svh_device_id` is `uuid NOT NULL` FK → `fixed.device_master`, but a bill/DC/return's device id is free text. `SalesStockService.shadowDevice()` uses the document's id if it is a registered device, else the session's counter, else 422 `SALES_DEVICE_UNREGISTERED`.
- `sale_dc.sdc_counter_id`, `sale_return.sr_counter_id`, `sale_dc_return.sdr_counter_id` are `uuid NOT NULL` with no FK and no counter master; `sale_bill.sb_counter_id` is nullable. DTOs mark them optional → the doc store refuses a missing one with 400 (`headerRequired`). A migration making the three nullable is the real fix (not run).
- `ux_avh_voucher_no` keeps a CANCELLED number taken and `ux_avh_src`/`ux_avh_doc_refno` admit one live voucher per document: a reversal voucher (`SalesPostingService.reverseLegs`) takes a NEW number of the same type, carries no `avh_src_*`/`avh_doc_refno`, and links via `avh_against_voucher_id` — the receipt-cancel precedent. The original is marked CANCELLED first.
- `accounts.acc_ledger_map` rows are GLOBAL (`alm_company_id IS NULL`); a probe filtering by company finds nothing.
- `sml_doc_rate` is `NOT NULL DEFAULT 0`; the engine writes 0 for "no rate" and phase 5 reads 0 as NULL for the last-sale stamp.

**How to apply:** don't "fix" these by editing the DTOs or adding SQL without reading this; see [[sales-e2e-specs-rights-fixture]].
