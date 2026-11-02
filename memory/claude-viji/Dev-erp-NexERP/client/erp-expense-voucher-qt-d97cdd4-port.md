---
name: erp-expense-voucher-qt-d97cdd4-port
description: "Expense Voucher (menu 277) re-ported to Qt d97cdd4 + 5e25760 on 2026-10-12 — popup grids 155/156, ui_tables 51/52, derived auto tender, lines-key freshness, clear-after-save, cancel & re-enter; deliberate web deviations"
metadata:
  node_type: memory
  type: project
  originSessionId: 8df33ffa-a948-4773-9af1-a588cb6fd19e
  modified: 2026-10-12T05:21:55.185Z
---

Ported 2026-10-12 into `features/accounts/expense` (the d5c3637 port is [[erp-qt-d5c3637-till-expense-settlement-port]]).

- **Grids:** layouts from ui_tables 51 "EXPENSE VOUCHER - LINES" / 52 "EXPENSE VOUCHER - TENDERS"
  (`columns.ts`, joined via receipt's `resolveReceiptColumns`). Widths are drawn as SHARES of a
  `width:100%` table — both layouts sum to 100 and the grids sit side by side; safe only because the
  table never sizes from content (see [[erp-ui-table-width-is-not-a-percent]]).
- **Reason / Expense ledger** are popup cells on grids 155 / 156 via opening-stock's `GridPickerModal`.
  Both ids are past the 100-row grid-directory cap, so they always run on their FALLBACK ids — wrong
  on live until that house bug is fixed. `/expenses/ledger-pick` and `quick-reasons` are no longer called.
- **First tender follows the balance** (`autoAmount`), DERIVED each render (`withAutoTender` +
  `autoTenderTotalPaise`), never stored. Each answer also carries a `linesKey` (body minus tenders).
  Without it, every rebalance would blank the tax cells for a round trip.
- **Deliberate deviation:** bare +/− are line keys in every cell EXCEPT free text (Description, Ref),
  where they type. Qt's cells have no editor open, so it never had that conflict.
- 5e25760: GST bill needs a party (client problem, before EXPENSE_GST_INCOMPLETE); Save/Post clear the
  screen; Cancel & re-enter = cancel + `reenterCopy` (needs Cancel AND Create rights); header Enter walk.
- Verified by CDP driver with a faked API + all captured bodies through the server DTOs (12/12 valid).
  Trap: `innerText` returns CSS-uppercased captions/titles, and the global capitalization
  ([[erp-global-uppercase-inputs]]) title-cases typed text ("Wrong Amount") — compare case-insensitively.
- After a save on the keyed route `/accounts/expense-voucher/:c/:b/:y/:id` the screen clears but the URL
  keeps the old keys (a reload reopens that voucher). Left as is.
- **UI matches the user's Qt screenshot (2026-10-12):** money-from card beside the title, sentence-case
  labels, Voucher no as a box, dd-mm-yyyy `components/date-input.tsx` (Qt QDateEdit look), GST card blue,
  grids flex to fill the height (grey empty area), equal-height feet so both grids end on one line,
  Posts only once there are legs, blank cells where Qt paints nothing. The grid scopes
  `--table-header-text` / `--erp-radius` ([[erp-global-th-color-radius-important]]).
- **Admin settings (2026-10-12):** right-click either grid → house `useGridSettings` dialog (all configured
  columns, hidden included; Focus note says it is unused here). Drag-to-widen is NOT the house
  `useColumnResize`: on a share layout that runs ahead of the mouse on a wide grid and lags on a narrow one.
  The local `components/use-share-resize.ts` instead keeps the sum fixed, so the last visible column absorbs
  the drag, as Qt's stretched last section does. It saves the px of BOTH changed columns.
  Drawing in the house px unit made the tenders grid (1100 config px) scroll sideways at 1900 px, so shares stay.

Print added 2026-10-14 (EXPENSE_VOUCHER, F11) — see [[erp-expense-print-template]].
