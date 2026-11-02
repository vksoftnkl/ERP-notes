---
name: prathap-server-access
description: "10-27 user granted direct SSH access to Prathap's machine (vk@192.168.0.106, key auth) to build the purchase backend; rules: own worktree/branch/port, migrations after user reviews SQL"
metadata:
  node_type: memory
  type: project
  originSessionId: 5f84b1e7-06ed-44c0-b0af-5205a5dbd71c
  modified: 2026-10-27T08:17:32.848Z
---

2026-10-27 the user gave direct access to Prathap's computer for the purchase module ("use backend and front end simultaneously"; "apply the migrations directly no problem i can review the sqls"). SSH works with key auth: `ssh vk@192.168.0.106` (host vk-MS-7C95). Never store or ask for the password.

Server repo: `/home/vk/Dev/erp/ERP server/` (space in path), remote github vksoftnkl/ERP-Server, Prathap's branch **DEV-v1** with ~128 UNCOMMITTED files; he works there in VS Code; the live API = pm2 `node dist/src/main.js` on :3000 from that folder; prisma migrations in `prisma/migrations` (last 20261027110000_transfers_notes_160).

**Rules (user 10-27, final):** NO separate branch and NO clean-up first — work in his folder on DEV-v1 as it is. Never touch/stash/commit his uncommitted files; commit only my files by explicit path (never `git add -A`); if a file also has his open changes, tell the user before committing. nest build / prisma generate rewrite dist + client there; pm2 API keeps old code until a restart, which only happens on the user's / Prathap's word; my test instance on :3012. Show each migration SQL to the user, apply after their OK. Supersedes the read-only part of [[nestjs-source-readonly]] for purchase work.

Related: [[purchase-module-analysis]].

**10-28 traps on his folder:**
- His index often has dist/ + logs/error.log ALREADY STAGED. A plain `git commit` sweeps them in (95e20707 did; left as is). Always `git commit -F msg -- <my paths>` (pathspec commit), then check `git show --stat`.
- :3011 is his `nest start --watch`. On rebuild the child sometimes closes its port but never exits, so the watch never restarts it → live down. User OK'd once: kill -9 the hung child — but 10-28 12:2x the auto-mode classifier DENIED a later kill (Interfere With Workloads); don't retry, tell the user. It also recovers by itself after a few minutes (12:26). Registration scripts use :3011 via tests/itemmasters/api.py — a shim pointed at :3012 works (same DB). Upload once per finished step.
- Never put an apostrophe inside a single-quoted ssh commit message; scp a message file instead.

**10-29: now LOCAL.** Claude runs on .106 itself, so there is no ssh or scp; edit the folder directly. The commit rules above still hold: only my files, by path, `git commit -F msg -- <paths>`. The :3012 instance is retired; test on :3011, which I started detached on 10-29 (recipe in [[after-move-facts]]). The `nest --watch` compile rewrites TRACKED `dist/` files: never commit them unless asked.
