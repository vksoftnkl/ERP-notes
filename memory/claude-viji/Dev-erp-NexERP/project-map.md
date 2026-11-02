---
name: project-map
description: The whole NexERP product on one page — PostgreSQL + NestJS + Qt desktop + React web, where each lives, how a feature moves through them, and where this memory keeps each part
metadata:
  type: project
---

NexERP is ONE hybrid product. A feature is usually a DB change, a NestJS route, a Qt screen, and then the same screen in React.

## The four parts (all on this box, vk-MS-7C95 = 192.168.0.106; see [[after-move-facts]])

| Part | Where | Branch | Runs on |
|---|---|---|---|
| PostgreSQL `ERP` | localhost:5432 | — | `psql -h localhost -U claude -d ERP` (read-only role; writes go through Prisma migrations the user runs) |
| NestJS + Prisma API | `/home/vk/Dev/erp/ERP server` | DEV-v1 | `nest --watch` HTTPS :3011 (dev, the one every client and test uses); PM2 :3000 (dist) |
| Qt 6.12 desktop | `/home/vk/Dev/erp/NexERP` | main | `build/Desktop_Qt_6_12_0_Debug`, menu ids registered in `src/core/navigation/form_*_menu.h` |
| React / Next.js web | `/home/vk/Dev/erp/ERP client` | dev | Next dev server https :3001; routes map menu ids (`components/layout/constants.ts`, MENU_ROUTES) |
| Share (plans, notes, mockups, themes, API tests) | `/home/vk/Shared/Prathap` | — | `notes-NNN-*.md` in area folders, for Prathap |
| Live | VPS 169.58.213.171 | DEV-v1 | `/opt/erp-server/bin/deploy.sh` ([[live-vps-sync-migration-p3009]]) |

**DB schemas:** accounts, audit, fixed, inventory, public, purchase, sales, settings, stock, transport.

**Configuration lives in the DB and drives both clients.** Both clients find these BY NAME, never by a hard-coded id:
- grids: `fixed.grid_details` / `grid_columns`, run through `/configured-grid-sql/run`
- dropdowns: `/dropdown-details/run`
- `ui_tables` for entry grids
- `menu_master` plus user rights
- the theme template `app_theme_template` (Qt) and the theme tokens

## How a feature moves (the user's order)
1. **Plan:** the share plan or mockup is the spec ([[mockup-fidelity]], [[plans-separate-file]]).
2. **DB:** a Prisma migration in ERP server ([[sql-vs-nestjs-rule]]: no DB functions or triggers). The user applies it with `!` ([[user-run-commands-with-bang]]).
3. **Server:** NestJS module and routes ([[api-house-conventions]], [[new-module-own-urls]]).
4. **Qt screen and Qt tests,** then the USER tests Qt.
5. **React port** of the same screen: same routes, JSON keys, keys and behaviour. Write a `client/erp-qt-<sha>-port.md` note per Qt commit range ported. See [[user-working-style]].
6. **notes-NNN for Prathap,** then commit (Qt main, server DEV-v1, client dev; only when asked), then push and deploy.

## Where this memory keeps each part (merged 2026-10-30)
- **Root** = Qt + DB design notes. Full list in `qt-db-index.md`.
- **`server/`** = NestJS / Prisma notes. Full list in `server/INDEX.md`.
- **`client/`** = React notes, including every Qt→React port. Full list in `client/INDEX.md`.
- **`workspace/`** = cross-repo, deploy and early-history notes. Full list in `workspace/INDEX.md`.
- The same folder loads whether a session starts in NexERP, ERP server, ERP client or `~/Dev/erp`: the claude-viji project memory dirs are symlinked here.
- Plain `claude` is Prathap's profile and keeps its own older copies ([[prathap-profile-agents-hooks]]).

**Agents:** 11 in `~/.claude-viji/agents`, one per layer ([[agent-team]]): db-schema, nestjs-backend, qt-screen, react-screen, api-tester, qt-tester, web-tester, reviewer, docs-notes, quick-fix, architect.
