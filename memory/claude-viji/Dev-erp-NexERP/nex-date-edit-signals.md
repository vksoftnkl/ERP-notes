---
name: nex-date-edit-signals
description: NexDateEdit has dateEdited (user only) vs dateChanged (any write) — always connect dateEdited
metadata: 
  node_type: memory
  type: reference
  originSessionId: 02bb4c26-2d49-4279-9c1f-475e2e47df12
  modified: 2026-08-04T05:22:41.849Z
---

`src/shared/widgets/nex_date_edit.h` emits two signals, the same split
`QLineEdit` draws between `textEdited` and `textChanged`:

- `dateEdited` — user keystroke or calendar pick only. **Connect to this.**
  `setDate()` never raises it, so a screen can paint the field freely.
- `dateChanged` — any text change, `setDate()` included. Passive mirrors only.

Both fire per keystroke, so a half-typed date arrives as an invalid `QDate` —
handlers must check `date().isValid()` before counting from it.

Before 2026-08-04 only `dateChanged` existed (emitted from `textChanged`), which
is why every `setDate()` in the codebase used to need a QSignalBlocker around
it. See [[widget-signal-wiring-rule]].
