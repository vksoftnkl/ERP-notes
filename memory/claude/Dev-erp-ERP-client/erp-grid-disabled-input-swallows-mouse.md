---
name: erp-grid-disabled-input-swallows-mouse
description: "Row-level mouse handlers (double-click, context menu) die over read-only grid cells — Chromium suppresses mouse events on disabled inputs entirely; fix with a scoped pointer-events:none"
metadata: 
  node_type: memory
  type: reference
  originSessionId: a756e224-1e96-4bb6-95f6-738f98b7336b
  modified: 2026-09-16T12:48:36.369Z
---

Every entry-grid cell in this repo (`components/grid-cell.tsx` and its Sale Order / Sale Bill
twins) is an `<input>` filling its whole `<td>`, and a read-only column renders it `disabled`.

**Chromium does not fire mouse events on a disabled form control at all — they do not even
bubble.** So a handler on the `<tr>` (or `<td>`) works over the editable half of a row and
silently does nothing over Code, Hsn, GodownName, StockQty, SchemeName… It reads as a broken
feature rather than a rule, and it is invisible in tests that happen to click an editable cell.

Fix, verified 2026-09-16 with playwright on the Quotation item grid:

```scss
.rowSizable .cellInput:disabled { pointer-events: none; }
```

**Scope it to rows where the row handler has something to do.** A blanket rule costs every
disabled input its `title` hover, and in browse mode EVERY cell is disabled — hovering a
truncated value is how a saved document is read. `ItemGrid` puts `.rowSizable` on a row only
when `editable && onOpenSizeEntry && line.itemId`.

Probe that finds it: `td.dblclick()` per column, logging whether the inner control is
`:disabled`, and whether the dialog opened.

Related: [[erp-quotation-multi-size-entry]], [[erp-client-ui-browser-verification]].
