---
name: print-log-reprint-check-mismatch
description: "print_log's live CHECK refuses REPRINT and copy_no 10 though the TS constants allow both; the failure is swallowed, so the row just goes missing"
metadata:
  node_type: memory
  type: project
  originSessionId: e80c1738-78d7-4126-85d0-2834a3e8c28f
  modified: 2026-10-01T11:54:17.867Z
---

`ck_plg_output_mode` on the live DB accepts only PRINT | PREVIEW | EMAIL | FILE, yet
`PLG_OUTPUT_MODES` in print-render.constants.ts lists REPRINT, and `/print-render/print`
with `isReprint: true` writes 'REPRINT'. `ck_plg_copy_no` is 1..9 while `MAX_COPIES` is 10.
`PrintLogService.record` swallows every insert error by design, so either case silently
drops the row (only a server-log line). Seen 2026-09-29.

**Why:** the print_log fragment was "reconstructed, not transcribed" from 17_printing.sql §7.

**How to apply:** never emit REPRINT or copy 10 without a migration widening the CHECK. The
print dialog's rows come from `POST /print-render/log` (PRINT/FILE, added 2026-09-29; PREVIEW
on popup open since 2026-10-01), not from `/print`. The popup hides the browser PDF viewer's
toolbar (`#toolbar=0`) because a print/download from it is invisible to the app and logged
nothing — a "print_log not updating" report was exactly that. Nothing server-side reads
print_log yet, so a PREVIEW row pollutes no count.
