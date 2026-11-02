---
name: grid-look-theme-only
description: User rule 2026-10-02 — grid alternate rows, row-number gutter and corner are styled by the THEME TEMPLATE only, never painted in code; how Qt makes that possible
metadata:
  type: feedback
---
Grid colours (alternate rows, vertical header / row numbers, QTableCornerButton) come from the theme template only, not from code.

**Why:** the user, 2026-10-02: "the alternate row color based on theme only not in code, do the same for qtablecornerbutton also". 3.0 painted its gutter in code (TableHeaderView::paintSection) and used an "S.No" QLabel over the corner. NexERP had copied both, so no template rule could reach them.

**How to apply:**
- NexTblHeaderView::paintSection calls QHeaderView::paintSection; only a TICKED row (pick dialogs) is drawn over it.
- The corner QLabel is gone; setCornerButtonEnabled(false) keeps the corner unclickable.
- QSS facts, measured offscreen:
  - A vertical section with NO border/background takes the table's alternate-background-color, so the gutter alternates with its rows.
  - Giving the section a border switches it to box mode and loses the alternation.
  - So borders go only on `:checked` (the current row).
- 3.0 grid rules + colours: see [[theme-template-decision]] (v12 seed).

**2026-10-02 revision (v13 LIVE, md5 dab6223a…):** the native gutter drew heavy bevels, and a bordered QSS section cannot alternate. So NexTblHeaderView paints the gutter again, but with NO colour of its own: it reads the TABLE's palette (Base / AlternateBase / Highlight / HighlightedText / Text, which the QSS fills) and SH_Table_GridLineColor (= gridline-color). It draws a thin right + bottom line, current row included. The user accepted "theme decides every colour" as the rule.
- **Corner "#":** NexTable::eventFilter paints the corner button as a header section through horizontalHeader()->style(), so the section:horizontal rule styles it.
- **Lines:** gridline #DCDFE4, borders #CBD0D7.

**NexApiTableView too (2026-10-02):** gutter + "#" corner now come from the shared `NexGridLook` (nex_table_header.h): paintRowNumber() + installHashCorner(), used by NexTable and NexRowHeader. NexJsonModel no longer bakes #f5f5f5 alternation (view uses alternatingRowColors + the theme); its checked row is the table.checked token. Grid-config column/condition colours still win via BackgroundRole.

**v14 LIVE 2026-10-02:** blank grid area grey #E1E4E8 via `NexTable, NexApiTableView { qproperty-emptyAreaColor }` + their QHeaderView background (NexGridLook::paintEmptyArea fills below the last row / right of the last column). NOT done with ::item backgrounds: a stylesheet ::item background overrides model BackgroundRole (grid condition/column colours, free rows).
