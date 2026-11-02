---
name: session-resume
description: START HERE — where work stands at the end of the last session and what to pick up next; read this first. Overwrite each day; do not let it accumulate history.
metadata:
  node_type: memory
  type: project
---

# Resume: end of 2026-10-30, continue 2026-10-31

Read [[after-move-facts]] first: everything runs on .106, Qt 6.12, the API on :3011, psql as claude via ~/.pgpass. Start with `claude-viji` ([[prathap-profile-agents-hooks]]); today ran under plain `claude`. The memory is ONE whole-product memory ([[project-map]]).

## State at close: ALL THREE REPOS CLEAN AND PUSHED (the user committed and pushed by hand)
- **Qt NexERP main = origin:**
  - `d478d85` "user right changes copy added": the Copy follow-ups.
  - `e782e59` "30-10-26": the PO rework and Copy v1.
  - `80ff85e`, `6739d86`, `131b042` from earlier.
  - origin is now HTTPS (it was SSH; this PC's SSH key is refused by GitHub).
- **ERP server DEV-v1 = origin:**
  - `16074e95` "gh" holds migrations 20261030180000 (purchase agent dropdown), 190000 (theme v48) and 200000 (menu cleanup). All three are APPLIED on dev.
  - Earlier: `9dab98b5` and `41bf7124` (the print grid migrations 150000 / 160000 / 170000).
- **ERP client dev = origin:** `08be847` "30-10-26" holds the header fix (hidden menu rows no longer drawn) and the user Copy port. `9fd1fe1` is the print template grid.
- **NOT known: whether deploy.sh has run for the live VPS.** If not, today's 6 migrations (150000–200000) go live with the next deploy. Check for a running deploy first ([[live-vps-sync-migration-p3009]]).
- Office server detached on :4000 (agent-office-kit/server.log), unless the PC was restarted.

## Done today (details in their notes)
- Printing parity, the print template grid in both clients, edit-published-in-place ([[printing-parity-2026-10-30]], [[erp-qt-131b042-port]]).
- Themes v47 / v48 published on dev, and v48 shipped as a guarded migration ([[theme-shipping-workflow]]).
- Purchase Order 23 rework, the user's 10 points: the `.ui`, pills, the Agent dropdown 'PURCHASE AGENTS', client line math (PurchaseLineMath), quick buttons ([[txn-screens-on-ui-rule]]).
- **Menu Master cut to what the code opens:** 265 → 128 menus; Transport and Attendance gone ([[menu-master-code-only]]). The re-add file is share admin/menu-cleanup-20261030/readd-menus.sql.
- **User Administration 58 Copy in Qt and React:** it copies the rights plus Allow Edit Date / Entry / Rate; SUPER ADMIN is blocked ([[erp-qt-e782e59-user-copy-port]]).
- **React menu "order":** the real cause was hidden rows being drawn. Fixed.

## Open questions for the user (ask early tomorrow)
1. New-user default role: Qt starts at CASHIER, React at User. Which should both use?
2. React's own "Copy rights from…" picker (Menu Permissions tab; Qt has none) can still copy a SUPER ADMIN's rights. Block SUPER ADMIN there, or remove the picker?
3. Should a SUPER ADMIN login itself be allowed to copy a SUPER ADMIN? Today the block applies to everyone.
4. Prathap (notes-164): retire the old PUBLISHED revision on an in-place publish?

## The user still to test
- PO screen 23 (the full 10-point rework).
- Copy on menu 58 (one real save with a ZT- login, then delete it).
- The React header and Copy in a browser (web-tester can do this, but needs a login; never mint tokens).

## Next, offered and not started
- After the PO test, move Purchase Invoice 24, GRN 196, Return 25, Shortage 195 and Supplier Dues 297 to `.ui`: one qt-screen agent each, then qt-tester and the reviewer.
- Reuse PurchaseLineMath on Purchase Entry 24 and GRN 196, which today have a server-only, debounced calc.
- **React owes:**
  - the PO 10 points, after the user tests Qt;
  - Party Outstanding Ctrl+L (branch / cancelled / bill-refs);
  - canvas niceties: crosstab totals, report-tree page labels, the remembered unit, 60-character format rules.
- Remove the agent worktrees `.claude/worktrees/agent-a7919…` and `agent-acd62…` (both at e0c56ef, no longer needed).
- Carried from 10-29: the error-message seed run; Shortage Receipt 195 status; the purchase suite pass counts; Prathap's P1/P2 topology answer (notes-162); notes-147 A / 148 / 142-144.

## Build/test facts
- Build: `cmake --build build/Desktop_Qt_6_12_0_Debug` from the repo root. Never `cp -p` into a build tree.
- Offscreen probes: link the build's objects without main.cpp.o (scratchpad run_probe scripts). For menu ids, use FormManager `isRegistered`, not grep ([[menu-master-code-only]]).
- `tests/printing/run.sh` writes to dev (blocks OUT3/4/5/7/RT); run blocks selectively ([[qt-test-runners-cwd]]).
- The claude DB role is read-only. Migrations are run by the user with `!`; a "ran it" without `!` ran nothing, so verify in `_prisma_migrations` ([[user-run-commands-with-bang]]).
