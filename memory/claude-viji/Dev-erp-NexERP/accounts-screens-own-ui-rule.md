---
name: accounts-screens-own-ui-rule
description: User rule 2026-09-14 — every accounts screen (opening balance, receipt, cheques, payment, issued cheques) is its own Designer .ui on NexDialog; never NexCrudMasterPage / NexCrudForm / the generic CRUD dialog, and not TxnMainView for the F8 list
metadata:
  type: feedback
---

"use separate ui for all screen dont use Nexcruddialog" — said while the
five Qt plans were being written.

**Why:** the accounts screens are document screens with several bands and
their own keyboard maps; the generic CRUD dialog / master page shape fights
that, and the user wants each screen readable on its own `.ui` the way
`OpeningStockEntry` and `SaleBillEntry` are.

**How to apply:** each screen = `NexDialog` subclass + its own `.ui` +
delegates + entity header; the F8 voucher list is a page inside the same
`.ui` (`QStackedWidget`, `NexApiTableView` on the module's grid), not a
`TxnMainView` (which is a `NexCrudMasterPage`). House rules collected in
`share/openingbal/plan-qt-opening-balance.md` §0 — reuse them for every
later accounts screen (journal, contra, ledger statement). See
[[payment-design]], [[receipt-design]].
