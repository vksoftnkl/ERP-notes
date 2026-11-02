---
name: grid-entry-delegate-preference
description: Entry grids are NexTable + delegate + popup cells, never QTableWidget cell widgets (setCellWidget)
metadata:
  type: feedback
---
Every entry grid uses NexTable + a QStyledItemDelegate (ReceiptDelegateBase) with popup cells for masters (NexTable::setPopupConfig → a POPUP grid), like the quotation's item cell — never `setCellWidget` combos / spin boxes / line edits.

**Why:** user, 2026-10-11, on the Expense Voucher: "the delegate is the most comfortable for selection and ui i prefer"; with cell widgets an amount cell was not selected on focus and pickers were plain combos.

**How to apply:** register a ui_table (focus flags on editable columns) + POPUP grids, keep a trailing blank row, bare +/− (see [[grid-line-keys-plus-minus]]), Enter on an empty picker cell opens the popup. Watch the combo-commit trap in [[nextable-grid-traps]]. Reference build: [[expense-voucher-screen]].
