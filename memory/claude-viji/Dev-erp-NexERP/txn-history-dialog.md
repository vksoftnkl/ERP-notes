---
name: txn-history-dialog
description: History / Ctrl+H on every txn list + opening/physical entry — TxnHistoryDialog over grid 126 (txn_status_log); notes 80 asks for the actor, device and doc-type fixes
metadata:
  type: project
---
**Built 2026-10-02 (user ask):** `src/shared/ui/txn/txn_history_dialog.{h,cpp}`, opened with `TxnHistoryDialog::showFor(TxnDocKey, docLabel, parent)`.
- **Where:** TxnMainView adds the `btnHistory` button (Ctrl+H, history.svg — the same name, icon and key as the masters' History) for EVERY document list. Opening and Physical Stock have a History - Ctrl+H button, enabled once the document is saved.
- **Data:** the log is `public.txn_status_log` (not "txn_log_status"). It is read through registered **grid 126 "POPUP - TXN HISTORY"** (house rule: no /list route). Params `idoc_id` / `iacc_year` / `icompany_id`. Files: share/grids/txn_history_grid.{sql,json}. The user name falls back to `tsl_created_by` when the actor is the zero uuid.
- **Sort:** the client sends sort_by=tsl_seq_no through the new `NexApiTableView::setSort()`. The runner wraps the SQL, so an inner ORDER BY is lost, and grid_sort_column is honoured by no one.
- **Look ("make it elegant"):** title + subtitle + summary card ("Created by X on … · now POSTED — … · N steps").
  - The Event column uses TxnRowDelegate pills with PER-DIALOG overrides (posted green, edits amber, cancelled/deleted red, converted blue, created grey). The lists keep POSTED = Locked slate.
  - The dialog height fits the rows (3–12).
  - Template block `QDialog#TxnHistoryDialog` = seed v16.
- **Server gaps (notes 80, share/admin):** A) 253 CREATED/CONVERTED rows have the zero actor (login name passed instead of usr_id); B) accounts docs never set tsl_device_id; C) all stock vouchers log as STOCK_ADJUSTMENT. audit_log field-level detail is not linkable (log_entity_id NULL).
