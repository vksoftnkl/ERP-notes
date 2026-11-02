---
name: widget-signal-wiring-rule
description: "Subscribe only to user-intent signals, never ones a programmatic write emits — the rule that removes the need for QSignalBlocker"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: 02bb4c26-2d49-4279-9c1f-475e2e47df12
  modified: 2026-08-04T05:22:28.795Z
---

Connect handlers ONLY to signals the user raises — `clicked`, `activated`,
`textEdited`, `editingFinished`, `NexDateEdit::dateEdited`, a delegate's
`cellEdited`. Never to `textChanged`, `dateChanged`, `valueChanged`,
`currentIndexChanged`, `toggled`, or `QTableWidget::itemChanged`/`cellChanged`,
which also fire when code writes the widget. Recompute by direct call.

**Why:** verified against the user's legacy 3.0 POS (`Sales/saleentry.cpp`,
6449 lines, and `quotationentry.cpp`) — **zero** QSignalBlocker in either. Not
because they're simpler: all 63 auto-slots are `clicked` (45), `textEdited` (6),
`activated` (4), `editingFinished`/`editingCompleted` (2). Nothing listens to a
signal a programmatic write can emit, so `setText()`/`setDate()` are inert and
there is nothing to block. Grid edits reach logic only via
`SaleDelegate::cellEdited`; `SaleEntry::cellChanged(row,col)` is a plain method
the code calls directly at the ~10 places that want a recompute.

**How to apply:** a `QSignalBlocker` around a widget write is a symptom — find
the subscription that made it necessary and repoint it, don't add the blocker.
Blockers on a *controller* signal to batch two recomputes into one (e.g.
`QSignalBlocker block(m_charges)` in `applyPlaceOfSupply`) are legitimate and
stay. Related: [[nex-date-edit-signals]], [[grid-delegate-edit-pattern]].
