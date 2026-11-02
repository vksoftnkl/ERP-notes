---
name: erp-global-th-color-radius-important
description: "globals.css forces every th's colour/weight and every control's corner radius with !important; override by redefining the variables, not by selectors"
metadata:
  node_type: memory
  type: project
  originSessionId: b214ce54-e494-4c3f-9ab0-746ca07b6366
  modified: 2026-10-03T06:24:20.724Z
---

`app/globals.css` has `:where(th, [role=columnheader]) { color: var(--table-header-text) !important; font-weight: 700 !important }`
and `:where(button, input, select, textarea, table, …) { border-radius: var(--erp-radius) !important }` (4px).
Zero specificity, but `!important` — so a module's `th { color: … }` or `border-radius` on a control silently loses.

**Why:** hit 2026-10-03 restyling Cheque Books — dark-red headers rendered black.

**How to apply:** set `--table-header-text: <colour>` on the table (scoped custom property) and read it back in the th rule; don't write border-radius on controls (dead code) — scope `--erp-radius` if a different corner is really needed. Related: [[erp-master-shell-modal-table-styles]], [[erp-master-modal-global-css-override]].
