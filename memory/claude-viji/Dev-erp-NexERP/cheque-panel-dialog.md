---
name: cheque-panel-dialog
description: "the cheques action panel is a modal QDialog lifted out of vboxRoot — and both test modal-closers must skip it by name"
metadata:
  type: project
---

**The Received Cheques action panel (Deposit / Clear / Bounce / Re-present /
Replace / Return) opens as a modal QDialog**, not inline under the register
(`a368f10`). Asked for by the user 2026-09-19; it also fixes what the old
comments already admitted — the panel grew freely and squeezed the grid to a
single row at the exact moment the operator wanted to re-read the cheques they
were banking.

**`framePanel` is LIFTED, not rebuilt.** `buildPanels()` removes it from
`vboxRoot` and makes it the dialog's only content, so `stkAction`, the
OK/Cancel pair, `lblPanelTitle` and `lblPanelWhy` all travel with it.
`showPanel()` / `closePanel()` / `submitPanel()` read the same widgets they
always did, and `findChild` still reaches them because the dialog is parented
to the screen.

- **Width: `kPanelMinWidth` 480, and the measuring order matters.**
  `sizeStackToCurrentPage()` has just flipped the stack's size policies, and a
  layout does not recompute until asked — so a bare `adjustSize()` measured a
  page Qt had not laid out and opened the dialog **narrower than its own
  sizeHint** (Deposit 256 against a hint of 379). Activate the layout chain
  from the page outwards FIRST.
- **Height must be `totalHeightForWidth(w)`, not `sizeHint().height()`.** Every
  panel ends in a word-wrapped `QLabel`, whose width hint is tiny and which
  grows tall instead. `sizeHint().height()` is the height wanted while still
  narrow; applied at a wider dialog it keeps the tall shape and adds dead
  space — Re-present grew 533 → 589 *because* it got wider.
- **`show()`, never `exec()`** — the submit is an async POST whose callback
  closes the panel; a nested loop would have to unwind from inside it.
- `QDialog::rejected` → `closePanel()`, so Esc clears `m_page`. Without it the
  next submit acts on a page nobody can see.
- `adjustSize()` AFTER the page is current, or the dialog opens at the tallest
  panel's height — the same trap `sizeStackToCurrentPage()` exists for.
- Centred on `window()`, not the desktop: it is an MDI child, and a dialog on
  the other monitor reads as "the button did nothing".
- `vboxRoot` index 3 (the register's stretch) is unaffected: framePanel sat at
  a later index, so removing it shifts nothing before it.

**BOTH cheque test harnesses must skip it.** `installModalCloser()` dismisses
whatever modal is up and cannot tell an AppMessage box from a panel — it was
rejecting Deposit within 120ms of the click. Both
`tests/cheques/test_received_cheques.cpp` and `test_cheque_operations.cpp` now
`continue` on `objectName() == "panelDialog"`. Miss the ops one and every
operation that suite performs is slammed shut. See
[[qt-offscreen-test-harness]].
