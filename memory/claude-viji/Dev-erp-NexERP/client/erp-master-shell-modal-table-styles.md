---
name: erp-master-shell-modal-table-styles
description: Global master-shell CSS overrides component-module styles for any <table> inside a master modal body
metadata: 
  node_type: memory
  type: reference
  originSessionId: 8a281876-5294-4afa-b3de-3eb124c8c4be
---

`styles/library/_master-shell.scss` styles **every** `<table>` inside a master modal generically via `.erp-ms-modal .erp-ms-modal-body table …` (e.g. `… thead th { text-align: left }`, cell padding, input heights). That selector has higher specificity (2 classes + element) than a CSS-module rule like `.table thead th`, so a component's own module SCSS silently loses.

To override for one component's table (e.g. centering the linked-records table headers in `features/masters/inventory/item/item-linked-records-editor.module.scss`), add a rule with **more** specificity, e.g. `.editor .tableWrap .table thead th { text-align: center }`. Verify the winner with `getComputedStyle` + matching `document.styleSheets` rules in a headless browser ([[erp-client-ui-browser-verification]]) — the DOM tells you which selector actually applied.
