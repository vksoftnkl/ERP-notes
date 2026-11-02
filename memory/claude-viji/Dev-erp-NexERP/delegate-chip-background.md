---
name: delegate-chip-background
description: Chip/badge cells must draw their background with DelegatePaint::background, never QStyledItemDelegate::paint with opt.text cleared — the raw text shows through when widened
metadata:
  type: feedback
---

For a cell that a delegate draws itself (a chip, a badge), draw the background with
`DelegatePaint::background(painter, opt)` (src/shared/ui/delegate_paint.h). Never use
`QStyledItemDelegate::paint(painter, bg, index)` with `bg.text` cleared.

**Why:** the base paint() runs initStyleOption again and reads the text back from the model. So the raw
value is painted UNDER the chip. It hides while the column is narrow, then shows through garbled once the
column is widened. The user reported it on the Voucher Register Instrument column on 2026-09-26. The same
pattern was fixed at 8 sites: voucher line, receipt bill, receipt tender, ledger map, opening balance and
opening bill delegates.

**How to apply:** use it for any new chip column. TxnRowDelegate was already correct: it calls
style->drawControl itself. `tests/register/run.sh instrshot` renders the cell at two widths, loads
app.qss, and saves PNGs to the scratchpad to look at.
