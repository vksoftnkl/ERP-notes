---
name: erp-modal-checkbox-cell-gutter
description: Master-shell gives auto-placed modal checkbox fields a 130px label gutter; the grid-column-start exemption never matches React inline styles
metadata: 
  node_type: memory
  type: project
  originSessionId: d0b3126c-7565-4980-83bb-9a51e176c645
  modified: 2026-08-10T07:19:15.566Z
---

In a master entry modal, a `type: "checkbox"` field placed with `fieldStyle: { gridColumn: "span N" }` silently loses ~130px of its cell: `styles/library/_master-shell.scss` re-imposes a two-column field grid (`minmax(7.5rem, --erp-modal-label-width)`) on every `checkboxField` and drops the box into the control column. Long captions then wrap onto two lines.

**Why:** the stylesheet exempts cells carrying an inline `grid-column-start`, but when React sets both `gridColumnStart` and `gridColumnEnd` the CSSOM serialises them back to the `grid-column` shorthand, so `[style*="grid-column-start"]` never matches.

**How to apply:** don't fight it with inline styles (the companion rule pins the wrapper to `grid-column: 2 / -1`). Add a page-scoped panel skin passed via `modalPanelClassName` that sets `grid-template-columns: minmax(0, 1fr)` on `div[class*="checkboxField"]` and `grid-column: 1 / -1` on its `label[class*="checkboxWrapper"]`, with the class repeated 3× so it outranks the global rule — see `features/masters/sales/tender/tender-modal.module.scss`. Related: [[erp-master-modal-legacy-tab-layout]], [[erp-master-modal-global-css-override]].
