---
name: txn-screens-on-ui-rule
description: Big transaction entry screens keep their static layout in a .ui file and their behaviour in C++; the purchase screens built in code are to be moved to .ui, as the Purchase Order was on 2026-10-30
metadata:
  type: feedback
---

**Rule (user + me, 2026-10-30):** a transaction entry screen has a `.ui` on `NexDialog` for its static layout and C++ for behaviour.
- The `.ui` owns: the header fields, pills, quick icon strip, sections, grids as promoted NexTable, totals strip and button rows.
- The C++ owns: data loading, the line math, grid columns from ui_tables, key bindings, enabling, and popups built at run time.
- Sale bill, quotation, sale order, receipt, payment, opening stock and expense voucher already work this way.

**Why:**
- It is the house pattern for every big transaction screen.
- `share/purchase/plan-purchase-frontend.md` l.11 asked for it.
- Layout fixes become .ui edits the user can also make in Qt Creator Designer, instead of C++ (the PO's `buildUi()` went 272 → 148 lines).
- The theme styles by objectName.
- DB-driven field config needs `fieldKey` / `sectionKey` properties, which live in the .ui files (55 of them; [[widget-config-ui-screens]]).
- The purchase screens were built in C++ `buildUi()` anyway, and the user asked "why was the .ui removed" (it never existed).

**How to apply:**
- Purchase Order 23 moved first (10-30, worktree agents, uncommitted at the time).
- Next, once the user has tested the PO: Purchase Invoice 24 (2804 lines), GRN 196 (2213), Return 25 (1505), Shortage Receipt 195 (1270), Supplier Dues 297 (1403). One qt-screen agent per screen, keeping member names and objectNames so the theme blocks and tests still hold, then qt-tester once and the reviewer.
- Small masters (agent, supplier group, supplier item) and popups can stay in code.
