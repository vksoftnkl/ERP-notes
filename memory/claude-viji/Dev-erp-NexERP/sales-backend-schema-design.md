---
name: sales-backend-schema-design
description: "Agreed DB architecture for sale bill/return/tenders/accounts — db/schema/*.sql, offline-first chain, all txn tables partitioned by acc_year"
metadata: 
  node_type: memory
  type: project
  originSessionId: 8383960f-32da-4bdc-bbdb-6fd9825b5841
  modified: 2026-08-01T05:45:28.465Z
---

Final backend schema (confirmed with user 2026-07-31) lives in `db/schema/00_init.sql … 05_acc_doc_register.sql`; `db/sale_bill.sql` is a throwaway test file, do not build on it.

Locked decisions (user said these must never need ALTER later):
- **3 layers, each fact owned once**: sales docs (sale_bill/_item, separate sale_return/_item) → statutory register (acc_voucher_doc_register + detail + einvoice + ewaybill) → ledger (acc_voucher_header + acc_vouchers + acc_voucher_bills + acc_voucher_cheques). Headers keep roll-up caches only; outstanding truth = acc_voucher_bills.
- **Returns are a separate table** (`sales.sale_return`, doc_type CREDIT_NOTE) with cross-year composite FK to the bill (`sr_bill_id, sr_bill_acc_year`).
- **Generic tender table** `accounts.acc_tender_detail` (src_module + src_doc_type + src_doc_id, td_voucher_id filled at posting) shared by sales order / sale bill / sale return / receipt / payment. Replaces legacy acc_sess_tenders; acc_temp_credits dropped too.
- **Generic charge table** `sales.sale_doc_charge` (same src-doc pattern) for quotation/order/bill/return.
- **All transaction tables PARTITION BY LIST (acc_year char(9))**, composite PK (id, acc_year); every unique index includes acc_year; helper `public.fn_create_year_partitions('YYYY-YYYY')` creates a whole year's set.
- **Offline-first counters**: uuidv7 ids generated at counter; bill slno is per-counter (unique incl. counter_id, voucher slno unique uses NULLS NOT DISTINCT); printed refno embeds counter series; accounts posting happens ONLY server-side after sync.
- House style: varchar + CHECK (no enums), created_by/modified_by uuid, is_deleted + sync_date everywhere, PostgreSQL 18+ (uuidv7).

Client side (built 2026-07-31): `src/modules/sales/salebill/` — SaleBillEntry mirrors QuotationEntry, reuses QuoteItemCols (+StockId/BatchDate/SerialNo appended) and QuotationItemDelegate and ChargeGridController {'S',"SALES"}. Temporary decisions to revisit: item grid id 18 and list grid id 83 reused from quotation (user will assign dedicated ids), `/tenders/list` endpoint pending (dialog falls back to Cash+Credit rows), holds + power-failure autosave are local JSON per counter under AppLocalData/salebill (swap for /holds API later), tenders array sent in bills/create but backend persistence pending, counter id = AppConfig deviceId, save status DRAFT unless AppSession::salesAutoPost.

Ids assigned by the user (2026-08-01, backend-confirmed — not guesses): sale bill menu id **12**, tender master menu id **95** + list grid id **85** (`db/tender_master_list_grid.sql`), tender master endpoints `/tender-masters/{create,get,delete}` with idKey tndId. `TenderMasterEntry` is a 2-tab NexCrudForm under `src/modules/sales/master/tender_master/`.

**Why:** chain-store ERP, tables cannot be renamed/altered after go-live.
**How to apply:** any new transaction table must follow the same rules (partitioned, composite PK, counter-aware numbering, src-doc pattern for shared detail tables). See also [[charge-distribution-model]], [[txn-main-view-architecture]].
