---
name: txn-status-log-stock-vouchers
description: public.txn_status_log holds one row per status STEP only — never per save; stock vouchers file under INVENTORY/STOCK_ADJUSTMENT via rules.statusDocType
metadata: 
  node_type: memory
  type: project
  originSessionId: a773ba83-f640-4ae6-a9d5-286d3f73f361
  modified: 2026-09-09T00:00:00.000Z
---

`public.txn_status_log` is the append-only status trail every transactional document shares. Write it with `appendTxnStatusLog` from `src/common/txn-status-log/txn-status-log.helper.ts`, always inside the caller's transaction.

**One row per status STEP, never per save.** An edit that leaves a document in the status it was already in adds nothing here — field-by-field change is `audit.audit_log`'s job. Bill, sale-order and (since 2026-09-08) stock vouchers all follow this. Stock vouchers log: `CREATED` (→DRAFT) in `createDraft`, `POSTED` in `post`, `CANCELLED` in `cancel`, and `DELETED` on soft delete — the last passes its event explicitly because a soft delete leaves `svh_status` alone, so from and to are both DRAFT and the event cannot be inferred.

**DECIDED 2026-09-09: for stock vouchers the trail is the ONLY record of a status change.** `stock.stock_voucher` keeps `svh_status` (the current state) and nothing else — migration `20260909120000_stock_voucher_lifecycle_to_status_log` backfilled the trail and dropped `svh_posted_on/_by`, `svh_approved_on/_by`, `svh_cancelled_on/_by` and `svh_cancel_reason`. The user's reason: two copies of one fact drift, and a header column is rewritable while a trail row is append-only. `postedOn`/`postedBy`/`postedByName`/`cancelledOn`/`cancelReason` stayed in the API payload — the list and `getById` raw queries read them back with `LEFT JOIN LATERAL` on `txn_status_log` (`tsl_src_doc_type` = `rules.statusDocType`, `tsl_src_doc_id`, `tsl_acc_year`, `tsl_to_status`, newest `tsl_seq_no` first), and `StockVoucherService.findLastStatusStep` does the same for the already-cancelled 409 message. Never re-add a stamp column to the header.

**`ck_tsl_src_doc_type` is a 14-value vocabulary shared across sales/purchase/accounts**, and only three of them describe a stock document: `STOCK_TRANSFER`, `STOCK_ADJUSTMENT`, `OTHER`. Eleven stock voucher types have to land on three values, so it is pinned per screen as `StockVoucherTypeRules.statusDocType` rather than derived — OPENING and PHYSICAL are filed as `STOCK_ADJUSTMENT`, the two transfer types as `STOCK_TRANSFER`. Adding a rules constant means adding this field (controllers, unit specs and `test/*.e2e-spec.ts` all declare their own).

**Module name differs from the stock ledger's.** Here it is `INVENTORY`; `stock_ledger.sml_src_module` wants `STOCK`. See [[stock-engine-ddl-lives-outside-repo]].

**Partitioned by `tsl_acc_year`** (the SOURCE document's year), so pass the document's year, not today's. A year with no `public.ensure_acc_year_partitions('YYYY-YYYY')` fails every insert — same trap as [[sale-bill-acc-year-partitions]]. 2024-2025 through 2026-2027 exist on localhost.

Unit tests mocking a Prisma client need `txnStatusLog: { findFirst, create }` on it — `findFirst` is the helper reading the last `tsl_seq_no`, and its absence surfaces as `Cannot read properties of undefined (reading 'findFirst')`.
