---
name: erp-qt-75c682b-port
description: "Port of Qt NexERP 74b755c..75c682b (2026-10-27) — Godown Transfer 41 / Branch Outward 192 / Branch Inward 193, Stock Reasons 293, Ageing Slabs 294, Tracking Presets 249, UI batch; server notes 153–160 were UNCOMMITTED in the server tree at port time"
metadata:
  node_type: memory
  type: project
  originSessionId: f758b293-d3c1-4595-84b0-6619df03a5d5
  modified: 2026-10-27T05:34:21.304Z
---

Ported 2026-10-27 by five parallel agents on a shared layer written first (MENU_ROUTES 41/192/193/249/293/294,
grids 183–188, ui_tables 54/55, dd 72 `employeeByDesignation`). Follows [[erp-qt-74b755c-port]]. Not committed.

- **Server state**: notes 153–160 (transfer loading charges, `/stock/transfer/recall`, `/receive*`, ageing-slabs module,
  track-preset save/deactivate, count-sheet brand/category/section/negativeOnly) sat UNCOMMITTED in the server
  working tree; nest --watch served them. All config rows (grids 183–188, ui_tables 54/55, dd 72, menus 293/294)
  are dev-DB only → [[erp-live-config-drift]].
- **Transfers**: `features/stocks/stock-transfer/` is ONE screen, `mode: "godown" | "branch"` (Qt Mode::BranchOut).
  Recall is ON (Qt `kRecallLive = false`); server refuses it 409 while ANY receipt exists against the despatch,
  a DRAFT one included — delete the draft first. Branch Inward (`features/stocks/branch-inward/`) is separate and
  does NOT import stock-transfer; `stock-transfer/components/handling-boxes.tsx` could replace its unloading box later.
  Pick-stock is judged on menu 264 (`ADJ_RIGHT_VIEW`) — a transfer user without Stock Adjustment view can't pick.
- **Live round trip verified** (despatch 2 → inward draft SALEABLE 1 + DAMAGED 1 → reopen via
  draftFromHeader+applyPrefill+overlaySavedLines → re-save → delete → recall, stock restored). Recipe: a vitest
  file + config in the scratchpad with `root` = client and `@` alias (config must NOT import `vitest/config` —
  it won't resolve from /tmp), `NODE_TLS_REJECT_UNAUTHORIZED=0`. Dev data: company 019c8ea6…, branch 019c8ea7… →
  019d611f…, godown 019daae6… → 019f0812…, lot 6e59a9c2… (SALEABLE).
- **Employee list**: grids 77 and 187 are BOTH Desktop "MAIN LIST - EMPLOYEES"; registry `newestWins: true` on
  `employeeList` makes the highest live id win for that name only.
- **Opening Stock** Size/Colour are now `choice-picker-cell.tsx` pick-lists (commit only on explicit pick); Model
  stays a datalist. Physical Stock F9 now has a filter dialog and MERGES loads (Qt) instead of asking to reload;
  empty holdings default OFF.
- **"No access" after a write**: see [[erp-server-usermenu-cache-cross-user]] — not the screens.
