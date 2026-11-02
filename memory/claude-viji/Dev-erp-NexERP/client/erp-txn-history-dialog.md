---
name: erp-txn-history-dialog
description: "Transaction History = status trail from grid 126 (txn_status_log) via components/txn/TxnHistoryDialog + CrudMasterPage `onHistoryAction` (button + Ctrl+H); sale bill, quotation, expense, receipt, payment and the voucher register use it; sale order still opens the audit-log modal"
metadata:
  node_type: memory
  type: project
  originSessionId: 9b19ae92-ba1a-4dae-9e22-354f499c3f38
  modified: 2026-10-08T05:17:13.258Z
---

Port of Qt `TxnHistoryDialog::statusTrail()` (2026-10-08, user asked for the Qt look on the Sales Bills list).

- `components/txn/txn-history-dialog.tsx` — generic: `target {docId, companyId, accYear, refNo}` + `docLabel`. Reads grid 126 "POPUP - TXN HISTORY" (registry key `txnHistory`) with `grid_param {idoc_id, iacc_year, icompany_id}`; columns from the grid's own config (`useGetGridColumnsQuery`), fallback list in `txn-history.ts`. Rows sorted client-side on `tsl_seq_no` numerically (runner drops inner ORDER BY).
- Event pills coloured by event TEXT (Qt statusTrail overrides: Posted green, Cancelled red, Retendered/Amended amber, Converted blue, Created grey); pill colours = Qt `TxnBadgePalette` HSL formula.
- Summary line: "Created by X on T · now STATUS — Event by Y on T · N steps" (status = text after `→` of the last row).
- Shell: `CrudMasterPage` prop `onHistoryAction(row)` replaces the audit modal, shows the History button even without `auditHistory`, and binds Ctrl+H ONLY for pages that pass it (42 audit-log masters deliberately untouched).
- Wired on sale bill + quotation (2026-10-08; quotation list = grid 83, logs Created/Converted/Deleted), expense vouchers, and on 2026-10-14 receipts, payments and the voucher register (every features/accounts/vouchers page; docLabel = the row's grid-117 `type_name`). Sale order still passes `auditHistory` — switching one is `onHistoryAction` + `<TxnHistoryDialog docLabel=…>` (Qt gives every TxnMainView doc type this trail).
- Why the accounts lists switched: their `auditHistory` modal was empty for nearly every row. audit.audit_log keys voucher writes to TENDER ids (log_table_name "receipt tender"/"payment tender"/"voucher register tender"); only ~47 receipts / 4 payments have header-keyed rows, and the register logs under screen "Voucher Register" while the page asked for "Contra"/"Journal". The trail is keyed by avh_voucher_id (doc type RECEIPT/PAYMENT, JOURNAL for every other register nature).
- Coverage gap (dev DB, 2026-10-14): ~112 Rct, 74 Pmt, 47 RcpV rows have no trail at all (no obvious source pattern — likely seeded/test-written); the dialog shows "No history is recorded for this document." for them. Not a client bug.
- Temp credits keep their own grid-132 dialog ([[erp-temp-credits-register]]).
- Verified with the fake-API CDP recipe ([[erp-client-ui-browser-verification]]); fixtures from grid 86/126 SQL with tokens substituted.
- Styling: another session converted the dialog from a .module.scss to Tailwind class constants in the .tsx (same day) — edit those, there is no scss file.
- Verifying quotation in the browser: the route opens on the ENTRY form; click "‹ Quotations" to reach the list.
