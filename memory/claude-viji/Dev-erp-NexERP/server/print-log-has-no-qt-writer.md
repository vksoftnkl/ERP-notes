---
name: print-log-has-no-qt-writer
description: "The Qt client prints via /print-render/preview + local QPrinter and never calls /print-render/log, so public.print_log holds no PRINT rows for Qt-printed documents (0 rows on the dev box 2026-10-17); the web client does call /log"
metadata:
  node_type: memory
  type: project
  originSessionId: 81104474-26c6-4d21-bb34-afa619677ddf
  modified: 2026-10-17T12:00:56.665Z
---

`/print-render/preview` writes no print_log row by design; `/print-render/print` (server-side
render through the ladder) logs every copy; `/print-render/log` records a PRINT / FILE / PREVIEW
made from a preview. The Qt client (`/home/vk/VKNexERP/NexERP`, `src/modules/admin/printing/`)
renders every document through Preview and prints the PDF locally with QPrinter
(`document_preview_dialog.cpp`); its endpoint registry `api_endpoints_printing.h` has no Log
constant and nothing calls `/print-render/log`. The web client does (`features/printing/components/
print-options-dialog.tsx`). `public.print_log` had 0 rows on the dev box on 2026-10-17.

Also: `ensure_acc_year_partitions()` rolls the till tables but NOT print_log, whose partitions stop
at `print_log_2026_2027`; TS `PLG_OUTPUT_MODES` holds REPRINT and `MAX_COPIES` 10 while the live
CHECKs refuse both ([[print-log-reprint-check-mismatch]]).

**Why:** the Void bill design (REV 1, 2026-10-17, reviewed, NOT approved, NOT built) makes
"was this bill printed before the void" its central control, read from print_log. Without a Qt
call to `/print-render/log` the flag is always false.

**How to apply:** any control that reads print_log as evidence needs the Qt to call
`/print-render/log` (PRINT SUCCESS after QPrinter accepts, FAILED on a printer error) first, and
print_log added to the year roll. See [[till-management-schema]], [[till-notes-124-open-list]].
