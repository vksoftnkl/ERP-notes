---
name: erp-global-ui-scale
description: Whole-app resolution scaling via `zoom` on <html>; the two invariants any new CSS or measurement code must respect
metadata:
  type: project
---

Since 2026-08-25 the client resizes the entire UI to the screen: `html { zoom: var(--erp-ui-scale) }` in `app/globals.css`, with the factor computed in `lib/ui-scale.ts` (reference viewport 1536x864 = scale 1, floor 0.8, ceiling 1.35, damped growth above the reference). It is written pre-paint by a blocking inline script in `app/layout.tsx`, maintained by `components/layout/ui-scale-controller.tsx`, and overridable from the header's "Interface size" select (`ui-scale-control.tsx`, localStorage `erp.ui.scale`).

`zoom` was chosen over a root font-size because the codebase is ~5,400 px literals vs ~1,200 rem, plus DB-driven column widths in raw px — only `zoom` reaches all of it.

**Why:** three things `zoom` does NOT scale, and each is easy to get silently wrong.

**How to apply:**
1. Any new `vw`/`vh`/`dvh` must be written `calc(<n>vh / var(--erp-ui-scale))`. Undivided, it overflows the screen at any scale above 1. (Dart Sass strips the inner `calc()` inside `min()`/`clamp()`; the emitted `min(860px, 96vw / var(--erp-ui-scale))` is valid and was verified in Chromium.)
2. `getBoundingClientRect()`, `clientX/clientY` and `window.innerWidth/Height` are VISUAL pixels; a CSS length written back into the document is a LAYOUT pixel. Convert with `toLayoutPx` / `layoutRect` / `layoutViewportSize` / `layoutPointer` from `lib/ui-scale.ts` before positioning a portal or storing a width. `offsetWidth`/`clientWidth`/`scrollWidth`/`getComputedStyle().width` are ALREADY layout pixels — do not convert those.
3. `@media (max-width: N)` is blind to the scale too — it only ever sees the raw device viewport, while every metric in the app is written against the *logical* width (`100vw / var(--erp-ui-scale)`), which `auto` scale holds at ~1536px on 1280- and 1366-wide screens. Use a **container query** for a layout threshold: containers are measured in their own zoomed pixels, so `@container <name> (max-width: 1365px)` compares the right number. (`erp-header.module.css` `.topHeader` carries `container-type: inline-size; container-name: erp-header`.)

Containers that exist today: `erp-header` (`erp-header.module.css` `.topHeader`) and **`erp-txn-page`** (the quotation stylesheet's `.page`, worn by Quotation, Sale Order and Sale Bill entry — every breakpoint on those three screens queries it). Fixed 2026-09-15: `@media (max-width: 1400px)` on `.headerRowFour` dropped the bill's credit column onto a second row at 1368x768, although the page had ~1537 logical px — a ~135px band stolen from the items grid. Verified 1368/1280 = 4 columns, 1024 = the 3-column fallback.

`features/print-designer` `.viewportInner` deliberately opts out with `zoom: calc(1 / var(--erp-ui-scale))` so mm<->px stays true; see [[erp-print-designer-ui]].

Fixed 2026-09-05 in `components/layout/erp-header.module.css`, both from this blind spot:
- Module icons vanished on 1280/1366-wide screens (and flickered as a window crossed 1366px) — a `@media (min-width: 1024px) and (max-width: 1365px)` rule hid them although the header had ~1520 logical px. Now `@container erp-header (max-width: 1365px)`.
- The menu row overlapped the date (Settings lost its chevron) below ~1600px: `.primaryMenu { min-width: min(100%, max-content) }` is **invalid CSS** — `max-content` is a sizing keyword and cannot go inside `min()`, so the declaration was dropped and `min-width: 0` stood. The nav shrank under its own non-shrinking items and `.headerRight` (`justify-content: flex-end`) painted over the tail. Fixed with plain `min-width: max-content`.
