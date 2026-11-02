---
name: till-build-review-notes-97
description: 2026-10-08 review of Prathap's full till build (phases 1-2, ExpV, settlement, claim) → share/till/notes-97; client tdSettleStatus "NA" bug fixed
metadata:
  type: project
---
Prathap built far past phases 1-2 by 2026-10-08 evening: ExpV (menu 277, /expenses/*), 40A(3), tender settlement (/tender-settlement/*, 12 routes), slip check, open-check, movements/change. All 11 till migrations live.
Review → /home/vk/Shared/Prathap/till/notes-97-till-build-review.md (High: cash bill with no session on unlinked device; PENDING_APPROVAL dead-end locks counter/cashier/device; settled card row re-tenderable).
Our client bug FIXED: sale bill / retender / sale order sent tdSettleStatus "NA" → server kept it → nothing ever PENDING. Field removed client-side.
Also fixed: User Administration rights tree loaded visibleOnly=true → hidden menus (147 of 245, incl. till 271-277) could not be granted AND a save wiped any grant on them (full replace). Now loads all, hidden ones grey.
**Why:** the user hands these notes to Prathap; NestJS source is read-only for us ([[nestjs-source-readonly]]).
**How to apply:** check notes-97 status before building till screens; see [[till-masters-screen]].
