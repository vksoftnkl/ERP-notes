---
name: erp-qt-desktop-printing-plan
description: "Qt desktop port plans for printing + print-designer (2026-09-29) — where they live, the core decisions, and the z-order bug found in the web canvas"
metadata:
  node_type: memory
  type: project
  originSessionId: af144e3f-35ed-4e5d-802c-6dc178106483
  modified: 2026-09-29T02:39:13.790Z
---

Written 2026-09-29 at the user's request: `features/printing/QT_DESKTOP_PLAN.md`
(master: server contract, document printing, delivery, config screens, phases) and
`features/print-designer/QT_DESKTOP_PLAN.md` (canvas). Plans only; no Qt code exists yet.

Core decisions in them: the server keeps rendering (Qt only shows/delivers PDF or raw
ESC/POS bytes, never ports the layout engine or jexl); Qt 6.8 Widgets + Pdf/PdfWidgets +
PrintSupport; async `QNetworkAccessManager` client; domain ported to a Core-only lib and
checked against JSON parity vectors exported from the web vitest cases; `ptvBody` model
wraps `QJsonObject` so unknown keys survive.

**Found while planning — a live web bug:** the server's `layout.engine.ts` paints each
band's elements sorted by `z` (stable), but the web canvas (`BandBody`/`ElementBox`)
paints in ARRAY order and ignores `z`. So the canvas can disagree with the print when
`z` and array order differ. Reorder actions move both, which hides it most of the time.

The only Qt code on this machine is a small mock-backed prototype `ErpDesktop` at
`~/Downloads/files` (Qt6 Widgets+Network, C++17, CMake, blocking `QEventLoop` requests).
Whether that or the legacy 3.0 (QtRPT) desktop app is the real host is an open question
in the plan, along with offline printing and `/print` vs `/preview` for the Print button.

Related: [[erp-printing-module]], [[erp-print-designer-ui]], [[erp-document-print-button]],
[[erp-print-template-contract]].
