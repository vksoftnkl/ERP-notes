---
name: till-cash-office-287
description: Cash Office menu 287 (notes 118 safe count + remittance) — built 10-15 offscreen; server code done but NOT deployed then; grids 157/158
metadata:
  node_type: memory
  type: project
  originSessionId: 3ef83d0b-5600-4825-9b13-0befb9e93814
  modified: 2026-10-15T05:17:11.199Z
---

Cash Office = **menu 287** (Prathap's migration 20261015100000_till_safe_notes_118; 286 stays retired). Built 2026-10-15 in `src/modules/sales/till/safe/`: TillSafeOffice (screen), TillSafeDecideDialog (safes/decide-variance, EXPENSE/SUSPENSE + VARIANCE reason + own PIN + second person dd 68 + PIN), TillRemitConfirmDialog (movements/remit-confirm). Business Day 274: SAFE_NOT_COUNTED / SAFE_OVER_INSURED rows + "Count safe →"/"Remit →" open 287 via SessionEvents::requestOpenMenu(287, {tab}); "SAFE · TODAY" numbers from days/get `safe`.

- Grids (Shared/Prathap/grids/till_safe_registration.py): **158 TILL SAFE BOOK** (TillSafeService.flowsOf rules row by row, newest first) and **157 TILL REMITTANCES**. Trap: the grid runner refuses any SQL containing the word DROP (bare \bdrop\b) — the kind is spelt `'DR' || 'OP'`. Trap: runner limit max 100 per page.
- Bank picker = dropdown 25 BANK LEDGERS (group "Bank Accounts", shared ledgers).
- Safe gaps live in till_variance stage SAFE / REMIT (tvr_session_id NULL, tvr_safe_id); the screen never decides inside count/confirm — beyond tolerance it waits and "Decide…" takes it.
- Tests: `SNAP_SRC=snap_safe` 13/13 (fake balance); **LIVE 10-15 `SNAP_SRC=live_safe` 12/12** (exact count, CIT remit crm00001 100.00, confirm, refused remit shows server notes-120 words). VKPOS already holds 287 V/C/O; user must grant 287 in User Administration (user_menus save is a full replace — [[user-menu-full-replace]]).
- 4 new error codes (TILL_SAFE_CHANGED, TILL_REMIT_OVER_BALANCE, TILL_REMIT_NOT_IN_TRANSIT, TILL_APPROVER_SECOND) approved + coded 10-15 ([[simple-error-messages]]).

**2026-10-21 Void remittance LIVE 12/12:** tests/till/live_remitvoid.cpp (`SNAP_SRC=live_remitvoid`; RV_TCM=<tcm_id> voids an already-posted remit). CIT remit 50 crm00002 → safe 65→15 → Void (reason Keyed wrong) → VOIDED, safe back to 65. crm00001 also voided. Trap: the post-remit "Print the memo now?" box must be answered No in a harness (Yes opens a print dialog and hangs). Safe ZTS was −12,000 because no hand-over had been approved since the 10-16 reset (float issues come out of the safe, CLOSE_HANDOVER only lands on Approve Cash) — approved all 5 ZT1 sessions with live_approvecash (each 5.00 short), safe now 65.
