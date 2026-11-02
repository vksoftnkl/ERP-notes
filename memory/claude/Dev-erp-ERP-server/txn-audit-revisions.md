---
name: txn-audit-revisions
description: Notes 89 (2026-10-03) — transaction saves write ONE audit_log revision (before/after /get snapshots, log_rev_no); stock vouchers done, 17 other transaction services still to convert
metadata:
  type: project
---

`AuditLogService.logDocumentRevision(input, tx)` writes ONE `audit.audit_log` row per save of a
transaction document: `log_original_record` = /get-shaped document BEFORE (SQL NULL on create — unlike
`logEntityChange`, which files an insert's record in log_original_record), `log_modified_record` =
AFTER, `log_changed_fields` = header-only diff, `log_rev_no` (new smallint, migration
20261003100000, partial unique `ux_audit_log_revision` on (screen, pk, rev)), `log_device_name` from
device_master. Not projected through audit screen fields. `/audit-log/list` now also returns
log_rev_no / log_entity_id / log_device_name.

**Done (step 1 + same code path):** every `StockVoucherService.save` (opening, physical, adjustment
family, transfer) — before/after via `getById(..., tx)`, AFTER read after an optional post; lines
upserted by `sviId` (DTOs of opening/physical/adjustment/base carry it; 422 on create/foreign/dup id;
renumber parks at 1_000_000+n because ux_svi_line is not deferrable). Old header + "lines replaced"
audit rows removed. Spec: `test/txn-revision-notes-89.e2e-spec.ts`.

**Still to do (plan step 3):** the other transaction services (sale bill, quotation, sale order, DC,
returns, receipts/payments/vouchers — audit screens 72–106) still write the old per-entity rows; Q2:
AMENDED should also write a revision, pure status steps should not. Qt side (steps 2/4) sends ids.

**How to apply:** any e2e that hands StockVoucherService a mock AuditLogService must mock
`logDocumentRevision` too (five stock suites were updated). See [[stock-adjustment-family]],
[[txn-status-log-stock-vouchers]].
