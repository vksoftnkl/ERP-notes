---
name: txn-line-audit-design
description: Proposed per-save txn audit (header + line diff) in Ctrl+H History; mockup in share/admin; APPROVED 2026-10-03; plan + notes 89 written
metadata:
  type: project
---

2026-10-03 user ask: an admin should see exactly what each save of a transaction changed (lines added, dropped or changed). Example: opening stock saved with 3 lines, then edited.

**Live today:** a txn save replaces all its lines. audit.audit_log gets a header `update` with changed_fields NULL, and for stock vouchers one `stock_voucher_item` row that only says `{svhId, lineCount}` "Opening stock lines replaced". sale_bill_item logs full lines on insert only. log_entity_id is NULL everywhere (notes 80).

**Approved design:** ONE audit row per save, with the full BEFORE and AFTER document (header + lines) in log_original_record / log_modified_record, log_entity_id = doc id, plus a revision number. No schema change. Line ids stay stable (the client sends them back; fallback key item+batch+godown). The client diffs when History opens. The Txn History dialog gets a revisions list on the left and the "Rev N: what changed" panel on the right (header from/to, +added / −dropped / ~changed with field-level from→to, unchanged hidden, totals strip). Buttons: View at Rev N (read-only form from that snapshot), Compare… (any two revisions), Export.

Mockup: share/admin/txn_audit_history_mockup.{py,png}. Plan: share/admin/plan-txn-line-audit.md. Server ask: share/admin/notes-89-txn-line-audit.md (A: one snapshot row per save in the /get shape, plus header changedFields and rev no; B: sviId upsert instead of delete+reinsert; C: opening stock first; Q1 log_rev_no column). Facts: no screen SENDS line ids yet (they are only kept in hidden Cols::Id / SalesItemCols::Id); /audit-log/list?screen_id&record_pk already returns original+modified; audit_screen snapshot SQL is disabled in code; opening stock = audit screen 67. Client steps after Prathap: send sviId, shared diff engine shared/ui/txn/txn_doc_diff, History right panel, View at Rev, tests/txnaudit. Related: [[txn-history-dialog]], [[mockup-fidelity]], [[plans-separate-file]].

**Notes 89 server side LIVE + verified 2026-10-03 (opening stock, via API with a ZT doc, cancelled after):** audit.audit_log has `log_rev_no` (unique per screen+pk). Create → rev 1, `insert`, original NULL, modified = {header, lines} in the /get shape (itemName, godownName, sviId…). Edit → rev 2, `update`, before and after both full; changedFields = header only (remarks). The "lines replaced" row is GONE. `sviId` upsert works: changed and unchanged lines keep their id, dropped is deleted, added gets a new id. log_device_name filled; log_entity_id set. The DTO spec shows sviId on opening, physical, adjustment and transfer. Cancel writes a `cancel` row WITHOUT a rev no. Header totals come from the client (an API test without them shows 0). Next: Qt step 2 (opening stock sends sviId → diff engine → History panel → View at Rev → tests/txnaudit).

**Qt step 2 BUILT 2026-10-03 (opening stock), uncommitted:**
- OpeningStockEntry::buildLines sends `sviId` (from Cols::Id; split rows have none). It repaints from the server answer after every save, so ids always come from the server.
- New files `shared/ui/txn/`: txn_audit_spec.h (TxnAuditSpec / TxnAuditField: screenId, headerTable, lineIdKey, matchKeys, header and line fields with compareKey = id-vs-name, viewAt callback); txn_doc_diff.{h,cpp} (pure: id pass → match-key pass, NEVER line order; numbers as numbers, ""==null; Indian grouping); txn_audit_panel.{h,cpp} (revisions left, diff right, filters, Show unchanged, totals, View at Rev / Compare… / Export CSV; DiffCellDelegate paints "old → new").
- TxnHistoryDialog::showFor(key, label, parent, audit = {}): audit mode when the spec is valid, otherwise the old grid-126 trail. TxnMainViewConfig::audit; TxnDocTypes::openingStockAudit() (screen 67, headerTable stock_voucher); OpeningStockEntry::showSnapshot(snapshot, caption) = read-only, all actions off.
- Live facts: /audit-log/list limit ≤ 100 (not 200); insert comes back as action "New"; log_pk is returned as the refno (filter with record_pk = uuid still works); user name "sfkdjfk" = VKPOS's display name. Pre-89 docs: skip rev-0 rows of non-header tables (the "lines replaced" rows); a rev-0 update with no status = EDITED, "not recorded".
- Styling only via the global roles `caption` and `dlgMuted` plus the existing TxnHistoryDialog block; NO template change was needed.
- Tests: tests/txnaudit/run.sh (offline diff, all pass) and run-live.sh (A1–A7 live, all pass, SNAP=dir for PNGs, applies the theme); tests/openingstock/run-live.sh still ALL PASSED.
- Next: other doc types = a spec each + send line ids (server: Prathap step 3).
