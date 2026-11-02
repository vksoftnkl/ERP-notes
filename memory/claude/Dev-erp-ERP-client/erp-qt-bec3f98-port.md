---
name: erp-qt-bec3f98-port
description: "Port of Qt NexERP c77f3f1..bec3f98 (2026-10-16) — plain-words errors app-wide helper, Cash Office 287 (safe count/remit/confirm/decide), cockpit Change + Void movement, Business Day safe rows, null till id on bills; live-sync for grids 157/158 + dd 68"
metadata:
  type: project
---

Ported 2026-10-16 by four parallel subagents (cash-office / sessions / business-day / session+counter
screens) on a shared layer written first. Follows [[erp-qt-c77f3f1-port]]. Not committed by me.

- **Errors (Qt ApiErrorText + server notes 120):** `lib/api/error-text.ts` (`apiRefusalOf`, `refusalMessage`),
  `toast.apiError`, `components/feedback/refusal-text.tsx`; `tillErrorOf(e).refusal`, `dialogRefusal(e).refusal`.
  Used on till screens + sale bill/testbill ("Bill not lost."), receipt, payment save/post failures only.
  Other screens still `toast.error(getApiErrorMessage(..))` — no central handler like Qt's, migrate per screen.
- **Cash Office** `features/sales/till/cash-office` at `/sales/till/cash-office?tab=count|remit` (menu 287;
  retired 286 has the same name but is inactive). `/till/safes/{balance,count,decide-variance}`,
  `/till/movements/{remit,remit-confirm,void}`; grids 157 TILL REMITTANCES / 158 TILL SAFE BOOK (registry
  `tillRemittances`/`tillSafeBook`), dd 25 + 68. Remit never sends `lines` (Qt parity; an agent had added a
  notes-count option, removed). Decide/Void greyed without 287 `canOverride` (Qt leaves it to the server).
  Grid runner ignores the grid's ORDER BY → sorted client-side. Smoke-rendered headless with a faked API.
- **Cockpit:** Change… (`/till/movements/change`, `sessions/change*.ts(x)`) and Void movement… on the Money
  docs pick (shared `features/sales/till/shared/movement-void-dialog.tsx`, also used for REMIT).
- **Business Day:** SAFE_NOT_COUNTED row + Count safe/Remit links, SAFE · TODAY, exception close with no live
  session (approvers = dd 68). Deliberate deviation: verdict text names the safe when only it blocks.
- **Bills/order/re-tender** send `sessionId: getTillSessionId() ?? null` (login id → server 500).
  Save actor is memoised at mount, so a till opened later is only seen after remount (pre-existing).
- **Live:** `docs/live-sync/2026-10-16-cash-office-grids.sql` (grids 157/158 + dd 68 by name, guarded on the
  notes-118 columns, dry-run on a scratch DB: guard/insert/update/idempotent/identical/id-collision OK). It
  also re-applies dd 68's notes-122 text, which the 2026-10-15 sync would otherwise undo. USER runs it
  ([[erp-live-config-drift]]). Menu 287 itself ships in server migration 20261015100000_till_safe_notes_118.
- Not ported: Qt FormManager focus-on-open / EnterNavigation tab skip (web has window-level keys); Approval
  Rules web screen has no initial focus and grid→form Enter gap (checked, not changed).
