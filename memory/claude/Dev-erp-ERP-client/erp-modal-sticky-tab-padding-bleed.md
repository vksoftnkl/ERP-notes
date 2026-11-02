---
name: erp-modal-sticky-tab-padding-bleed
description: Master-modal sticky tab bar leaks scrolling content above it because the scroll body has top padding; fix with box-shadow (not a pseudo)
metadata: 
  node_type: memory
  type: project
  originSessionId: 401005cf-2a15-4b04-96ce-cf8d3424a079
---

Master-form modal tab bar (`.erp-ms-modal-tabs` / module `.sectionTabs`) is `position: sticky; top: 0` inside the scroll body `.erp-ms-modal-body`, which has `padding: 14px 16px`. Sticky pins to the body's **content** box, so a 14px strip of that top padding sits *above* the pinned bar — scrolling field/table content bleeds through it and appears to float over the tabs.

**Fix (applied in `styles/library/_master-shell.scss`):** `box-shadow: 0 -14px 0 0 var(--erp-ms-paper)` on the tab bar — an opaque paper-coloured block 14px above the bar that rides with it. A `::before` pseudo does NOT work here: the bar carries `overflow-x: auto` (for horizontal tab scroll), which clips any child sitting above it. A box-shadow paints outside the box and is immune to that clipping. Keep the 14px offset in sync with the body's top padding.

The sticky table headers (item-linked-records-editor `.table thead th`) are NOT the culprit — they stick within their own `overflow-x:auto` `.tableWrap`, never over the tabs. Related: [[erp-master-shell-modal-table-styles]], [[erp-master-modal-global-css-override]], [[erp-modal-dense-grid-heading-bleed]].
