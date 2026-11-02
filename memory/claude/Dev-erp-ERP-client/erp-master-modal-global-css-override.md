---
name: erp-master-modal-global-css-override
description: Master-modal styling comes from TWO sheets; global _master-shell.scss overrides the CSS module at higher specificity
metadata: 
  node_type: memory
  type: project
  originSessionId: 55169fea-bb1c-44e3-b718-a04855e2497e
---

Master modal (ERPDynamicModalForm) field/checkbox styling is set in **two** places, and edits to the CSS module often "don't take" because a global sheet wins:

1. `components/design-system/ui/dynamic-modal-form.module.scss` — the CSS-module (hashed class names).
2. `styles/library/_master-shell.scss` — GLOBAL overrides scoped under the `.erp-ms-modal` panel class, targeting the modal's hashed classes via substring attribute selectors like `div[class*="checkboxField"] > label[class*="checkboxWrapper"]`. These have higher specificity (e.g. `.erp-ms-modal` + `div` + `[class*=]` + `label` + `[class*=]` = 0,3,2) than the module's own `.checkboxField > .checkboxWrapper` (0,2,0), so **_master-shell.scss wins**.

When a master-modal style change has no effect, grep `_master-shell.scss` for the same target first. Dump what's actually served with playwright: iterate `document.styleSheets[*].cssRules` and match `cssText` (see [[erp-client-ui-browser-verification]]).

Concrete example fixed 2026-07-15: item Rules & Status checkboxes were misaligned because `_master-shell.scss` forced every checkbox wrapper to `grid-column: 2 / -1` (fine for standalone checkboxes, but in the 6-col section grid it dropped each box into a content-sized implicit column that drifts with label length). Fix = scope an override to checkbox cells carrying an inline `grid-column-start` (only multi-column-section fields have it, set from `gridColumnStart`): `div[class*="checkboxField"][style*="grid-column-start"] > label[class*="checkboxWrapper"] { grid-column: 1 / -1; }`. Same _master-shell-outranks-module principle as [[erp-master-shell-modal-table-styles]] (tables). Related: [[erp-widget-masters-config]].
