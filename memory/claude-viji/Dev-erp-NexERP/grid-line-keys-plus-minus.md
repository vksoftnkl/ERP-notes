---
name: grid-line-keys-plus-minus
description: Entry grids add/remove lines with bare + / - (NexTable::setLineKeysEnabled), never Ctrl chords like Ctrl+Delete
metadata:
  type: feedback
---

Line add/remove on an entry grid is bare `+` / `-` via `NexTable::setLineKeysEnabled(true)` + `lineCreateRequested` / `lineRemoveRequested` (the quotation/DC idiom). Do not bind Ctrl+Delete or similar Ctrl chords for it.

**Why:** user, 2026-09-28, on the Voucher Register ("use +/- avoid ctrl keys") after Ctrl+Delete stopped working once the ledger popup had taken focus.

**How to apply:** new grids get setLineKeysEnabled; after a NexPopupWindow pick, call `setFocus()` on the grid (the Qt::Tool popup does not hand focus back). Ctrl+Q (instrument) on the register is still a Ctrl key — ask before changing it. See [[voucher-register-screen]].
