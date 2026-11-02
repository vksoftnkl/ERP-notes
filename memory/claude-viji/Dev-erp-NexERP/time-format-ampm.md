---
name: time-format-ampm
description: "User rule 2026-10-14 — every displayed time is 12-hour AM/PM, never 24-hour (grids SQL + Qt labels)"
metadata:
  node_type: memory
  type: feedback
  originSessionId: be97f9ff-7dd1-4a20-8a9e-6cc099232ffe
  modified: 2026-10-14T12:57:07.547Z
---

User 2026-10-14: "on time columns use AM/PM dont use 24HR format".

**Why:** the shop staff read 12-hour clock; 18:17 confuses them.

**How to apply:**
- Grid SQL: `to_char(x, 'HH12:MI AM')`, with seconds `'HH12:MI:SS AM'`, with date `'DD-MM-YYYY HH12:MI AM'`. Leave sort keys (`'YYYYMMDDHH24MISSUS'`) and ISO strings (`"T"HH24:MI:SS`) alone; keep a raw timestamp column for sorting, never sort on the formatted text.
- Qt display: `hh:mm AP` (Qt `hh`/`HH` without AP is 24-hour). The generic grid DateTime type already uses `dd/MM/yyyy hh:mm AP`.
- Payload/parse formats stay 24-hour (`HH:mm:ss` sent to the server, bank-file date formats).
- "06:17 PM" is wider than "18:17": recheck column widths after a change.
- Done 2026-10-14: grids 126, 132, 133, 136, 144–150 and 26 Qt display sites. Inputs too (user said yes): loyalty/promotion scheme time edits and physical stock freeze window use hh:mm AP.
