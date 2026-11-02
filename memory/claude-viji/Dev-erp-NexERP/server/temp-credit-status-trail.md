---
name: temp-credit-status-trail
description: Since notes 90 (2026-10-05) a temp credit has its own txn_status_log trail (doc type TEMP_CREDIT keyed by atc_id) — CREATED on bill post, PARTIAL/SETTLED/WRITTEN_OFF/REOPENED from the bill-balance recompute (before-read + UPDATE … RETURNING, only real moves), CANCELLED from cancel/amend/retender; WRITTEN_OFF is derived, not stamped; the post also writes an insert audit row
metadata:
  type: project
---

Notes 90 (Qt client, 2026-10-05): Ctrl+H on Temp Credits (menu 257, grid 132) reads
`txn_status_log` by `atc_id` and found nothing. Now every writer of `atc_status` appends a
TEMP_CREDIT step through `src/common/txn-status-log/temp-credit-status.ts`
(`appendTempCreditStatus`, `settlementEventOf`, `movementRemark`): `BillLifecycleService.postCore`
step 7 (CREATED → OPEN + an `insert` audit row with `log_entity_id = atc_id`), `unwind` and
`BillRetenderService` (CANCELLED), and `BillBalanceRecomputeService.recomputeBills`, which reads
the credits before, runs the status UPDATE with RETURNING, and logs ONLY rows whose status
changed — remark "received 10.00 by RCT/0012, balance 200.00", the voucher being the bill's last
live adjustment. Migration `20261005140000` widened `ck_tsl_src_doc_type`.

**WRITTEN_OFF is derived (notes 90 C):** pending ≤ 0 and the bill's LAST movement — by date, then
`abj_created_on`, WRITEOFF winning a tie because one receipt's money and write-off rows share a
timestamp — was a WRITEOFF. A cancelled write-off reopens the credit. The old sticky
`WHEN atc_status IN ('WRITTEN_OFF','CANCELLED') THEN keep` is gone; only CANCELLED is sticky.

**Why:** the client built grid 132 as a UNION over four tables and asked for the fifth branch; a
recompute that logged every run would flood the trail, so transitions are diffed.

**How to apply:** a new writer of `atc_status` must append a step too. `/bills/cancel` refuses a
bill that ever had an allocation (SALES_ALLOCATION_LOCKS_BILL), reversed or not — so an e2e
that posts receipts cannot end with a bill cancel (temp-credit-history.e2e-spec.ts stops at the
reopen; bill-tender-http.e2e-spec.ts asserts CANCELLED on a receipt-free bill). Receipt suites
need menu 99 via grantMenuRights.

Related: [[txn-status-log-stock-vouchers]], [[pre-existing-test-failures]]
