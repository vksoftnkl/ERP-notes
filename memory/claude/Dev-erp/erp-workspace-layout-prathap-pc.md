---
name: erp-workspace-layout-prathap-pc
description: "Where the three ERP codebases, the DB and the plan share live on vk-MS-7C95 (192.168.0.106, Prathap's PC) after the 2026-10-28 move, and what was still unsettled on 2026-10-29"
metadata:
  node_type: memory
  type: project
  originSessionId: 07782315-3c11-4ad7-950e-539b79a9ab2b
  modified: 2026-10-29T16:12:43.676Z
---

Machine: `vk-MS-7C95`, 192.168.0.106 (Prathap's PC), Linux user `vk`. After the 2026-10-28 move, everything is on this one machine:

- `/home/vk/Dev/erp/ERP client`: React / Next.js (`features/`, `app/`), branch `dev`, github vksoftnkl/ERP-frontend.
- `/home/vk/Dev/erp/ERP server`: NestJS 11 + Prisma 6, branch `DEV-v1`, github vksoftnkl/ERP-Server. pm2 `api` :3000. The `nest --watch` :3011 and `erp-purchase` :3012 servers are started by hand.
- `/home/vk/Dev/erp/NexERP`: Qt 6 desktop app, branch `main`, github vksoftnkl/NexERP. It was copied here 2026-10-28 from `/home/vk/Projects/Qt6/NexERP` on Viji's PC and is excluded from the parent repo via `.git/info/exclude`.
- DB: local PostgreSQL 18.6, db `ERP`. Login roles are `postgres` (credentials in the server `.env`) and `claude`. vk has no `~/.pgpass`.
- Plans / notes / mockups / API tests: `/home/vk/Shared/Prathap`, which is the master copy. Numbered `notes-NNN-*.md` files are filed by module folder. `git init` was run there, but nothing has been committed.
- Move checklist: `/home/vk/Shared/Prathap/move/checklist-move-nexerp-to-prathap-pc.md`. Agent-team design: `Prathap/agents/*.png`.

**User decisions (2026-10-29). The switch to this PC is FINAL.** On 10-29 I started :3011 detached; the recipe is in the Qt memory `after-move-facts.md`.
- **Stay on Qt 6.12.0.** The build dir is `build/Desktop_Qt_6_12_0_Debug` (underscore before Debug). 69 test files still default to `/home/vk/Qt/6.10.2` and `Desktop_Qt_6_10_2-Debug`; their runners accept a `BUILD=` override.
- **Test and run against :3011** (the HTTPS `nest start --watch`, which Qt's `appconfig.ini` uses), not :3012 `erp-purchase`. On 10-29 morning, nothing was listening on :3011; only pm2 `api` :3000 was up.
- **DB inspection uses the `claude` role.** It is read-only (`pg_read_all_data`, no CREATE). It needs `~/.pgpass`, which the user types themselves. Migrations go through Prisma with the server `.env` `postgres` URL ("postgres is fine").

**Viji's Qt-side Claude memory** is at `/home/vk/.claude-viji/projects/-home-vk-Dev-erp-NexERP/memory` (207 files, copied 10-29), with `alias claude-viji` in `~/.bashrc`. It loads automatically only when `claude-viji` is started from `/home/vk/Dev/erp/NexERP`. As of 10-29 it was not signed in (no `.credentials.json`). Its notes still say 6.10.2, :3012, `~/.pgpass` at .106, and `nexerp-api-ro`. Read its `session-resume.md` for where the purchase work stopped.

**After a reboot (seen 2026-10-29 21:36):**
- Only pm2 `api` :3000 comes back. :3011 (`nest --watch`) and the shared Next dev server :3001 do not, and a running `claude-viji` session dies with its agents.
- `/tmp` is wiped, which takes every scratchpad, task-output file and test harness with it. Subagent transcripts survive under `~/.claude-viji/projects/-home-vk-Dev-erp-NexERP/<session>/subagents/`.
- To resume, read the dead session's `.jsonl` and find the `Agent` launches that have no completion notification.
- Keep browser harnesses outside /tmp; `~/.cache/erp-webtest/` is used.

**How to apply:** ask the DB, not the migration files ([[erp-pasted-design-notes-need-verification]]).
