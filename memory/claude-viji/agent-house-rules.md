# NexERP agents — house rules (every agent reads this first)

You are a subagent. You cannot talk to the user; the main session does. When something needs the
user's decision, **stop and return the question** in your report. Never guess a business rule.

## Where things are (one machine: vk-MS-7C95 = 192.168.0.106, everything local)

| What | Path / address |
|---|---|
| Qt desktop app | `/home/vk/Dev/erp/NexERP` (branch `main`). Qt **6.12.0** at `/home/vk/Qt/6.12.0/gcc_64`. Build dir `build/Desktop_Qt_6_12_0_Debug`. Build: `/home/vk/Qt/Tools/Ninja/ninja -C build/Desktop_Qt_6_12_0_Debug` |
| NestJS backend | `/home/vk/Dev/erp/ERP server` (branch `DEV-v1`). Dev API **`https://192.168.0.106:3011/api/v1`** (`nest start --watch`). pm2 `api` on :3000 is not ours. There is NO :3012 any more. |
| React web app | `/home/vk/Dev/erp/ERP client` (Next.js, branch `dev`). Talks to :3011. |
| Share (plans, mockups, notes, API tests, SQL copies) | `/home/vk/Shared/Prathap/<module>/`. NOT under git (no history): read a file before you change it; never delete one. |
| Database | PostgreSQL 18, db `ERP`. Read: `psql -h localhost -U claude -d ERP -At -c "…"` (read-only role, password from `~/.pgpass`). Writes only through migrations (see db-schema). |

Never type, print, echo or ask for a password, token or secret.

## Memory: read before you work
- Qt: `/home/vk/.claude-viji/projects/-home-vk-Dev-erp-NexERP/memory/MEMORY.md`. Start with `after-move-facts.md`, then `session-resume.md`.
- Server: `/home/vk/.claude/projects/-home-vk-Dev-erp-ERP-server/memory/MEMORY.md`
- Client: `/home/vk/.claude/projects/-home-vk-Dev-erp-ERP-client/memory/MEMORY.md`

Read the index, then the notes your task touches. Where an older note disagrees with `after-move-facts.md`, the latter wins: 6.10.2, :3012, `nexerp-api-ro`, ssh and `/home/vk/Projects/Qt6` are retired.

## Never
- `git commit`, `push`, `stash`, `reset`, `checkout`, `restore`, `add -A`. The user says "commit" and the main session commits by explicit path.
- Apply a migration, run seeds, restart pm2, kill a process, or start/stop :3011. Report what is needed; the user gives the OK.
- Touch prod (169.58.213.171, or anything whose name contains prod).
- Edit or revert files outside your task. Prathap edits `ERP server` and `ERP client` at the same time as you. Run `git status` first. If a file you must edit already has changes you did not make, say so in your report.
- Hand-edit or "fix" `dist/` (the watch compile rewrites those tracked files) or `prisma/schema.prisma` (it is generated).
- Treat a data row on this DB as a bug. The schema is authoritative; the data is test data.

## Always
- **Verify against the artefact.** Read the live API answer (curl :3011), the real DTO / `.ui` / file, and the DB catalog (`pg_catalog`, `information_schema`) before writing code that depends on them. Never use names you have not seen.
- **API house rules:**
  - ONE `partyId`: a customer / supplier id IS its `acc_ledger_master.led_id`.
  - No `/list` route on documents: lists are registered grids (configured-grid-sql).
  - `/get` takes companyId + branchId + accYear + id.
  - A new module gets its OWN routes, DTOs and services; never reuse an existing URL.
- **No DB functions or triggers** for business logic. Stock posting is TypeScript (`StockPostingService`).
- **Offline-safe:**
  - Branches post offline and sync later. No uniqueness or CHECK that two offline branches could both legitimately break.
  - Numbers are per device.
- **Money and quantities** stay `Prisma.Decimal` into numeric columns; no floats, no epsilons.
- **Errors:**
  - Every refusal carries `errors[].code`.
  - The user sees simple English (what happened + what to do); wording lives in `fixed.error_message` / `prisma/seed/Error_Messages.json`.
- **Display formats:**
  - Times are 12-hour AM/PM everywhere.
  - React dates are `dd/MM/yyyy`.
  - Grid SQL emits IST text for timestamps.
- **Look comes from the theme template only** (`/home/vk/Shared/Prathap/theme/seed-template-vNN.qss` for Qt, the design system for React). Never hard-code colours or fonts in a screen.
- **Test data:**
  - Names start with `ZT-`.
  - The VKPOS test login on Acme Foods / Coimbatore is what the existing tests use.
  - Cancel what you post. Delete only rows you can prove you created.
- **Report** (short, factual, in this order):
  1. files changed (full paths);
  2. checks run, with exact results (e.g. "ninja: 0 errors", "api suite 41/41");
  3. what you did NOT do and why;
  4. questions for the user.
  Say plainly when something failed.
