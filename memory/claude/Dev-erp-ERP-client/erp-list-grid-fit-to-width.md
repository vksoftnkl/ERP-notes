---
name: erp-list-grid-fit-to-width
description: "CrudMasterPage lists scale their stored column px to fill the viewport (ReusableTable `fitColumnsToWidth`); stored px are proportions; last-column resize handle used to cause a phantom h-scrollbar"
metadata:
  node_type: memory
  type: project
  originSessionId: 804e6d11-a23c-4b60-b753-3fa48c3ac40d
  modified: 2026-10-13T05:02:20.396Z
---

Since 2026-10-13 every CrudMasterPage list passes `fitColumnsToWidth` to ReusableTable
(`components/ui/table.tsx`). Stored `grid_column_px` widths are treated as PROPORTIONS: the
scalable columns are scaled so the table exactly fills its viewport (serial, actions and
`fixedWidth: true` columns — the multi-select tick box — keep their px). Shrinking stops at
`FIT_COLUMNS_MIN_SCALE` (0.75); past that the table scrolls.

**Why:** user reported the sale-bill list looked different at 1366x768 vs a ~1850 screen. Cause
was the global UI zoom ([[erp-global-ui-scale]]): a 1366x768 browser (~633px tall viewport)
floors at 0.8 → ~1707 logical px, the big screen was 1.04 → ~1780, while the table was a fixed
sum of px — gap on one, scrollbar on the other. The 1366 scrollbar was ALSO the last column's
resize handle (`.columnResizeHandleRight { right:-6px }`) poking 6px past the table; now
`th:last-child > .columnResizeHandleRight { right: 0 }`.

**How to apply:**
- Available width is read from a zero-height `.fitProbe` div inside the viewport
  (`getBoundingClientRect` → `toLayoutPx`, floored) — `clientWidth` rounds up half pixels and
  re-introduces the scrollbar.
- A drag works in UNSCALED px: scale frozen during the drag; on release the stored width is
  solved (`resolveFitDragNaturalWidth`) so the column stays the width it was let go at and the
  others absorb the change. `onColumnResizeEnd.widthPx` is that unscaled value → px stored as-is.
- Verified headless at 1850x920 and 1366x633 with a fake-API CDP driver
  ([[erp-client-ui-browser-verification]]): no h-scroll, identical proportions.
