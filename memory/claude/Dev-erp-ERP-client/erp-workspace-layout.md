---
name: erp-workspace-layout
description: ERP client (Next.js) and server (NestJS) are sibling repos; server DB/Redis reachable locally for debugging
metadata: 
  node_type: memory
  type: project
  originSessionId: c22ab12e-2b50-4347-992e-02b08faa7e08
---

The workspace `/home/vk/Dev/erp/` holds two sibling git repos: `ERP client` (Next.js) and `ERP server` (NestJS + Prisma). API issues reported against `http://192.168.0.106:3011/api/v1/...` are served by `ERP server`, which runs locally — often as a plain `node dist/src/main` on 3011 that does NOT auto-reload (seen again 2026-10-28; check `ps`, see [[erp-stale-dist-404]]), sometimes under `nest start --watch`. Postgres (`DATABASE_URL` in `ERP server/.env`, db `ERP`, schemas like `fixed`, `public`, `audit`) and Redis (db 0, `http-cache:` key prefix, 300s controller TTL) are reachable on localhost — querying the DB directly is the fastest way to verify what an endpoint should return, since API GETs require a session-validated JWT.

## Several Claude sessions share ONE working tree (2026-09-21)

`ListAgents` routinely shows 4-6 peer sessions on these repos, and they are all
editing the SAME checkout — not worktrees. Consequences:

- **Never `git stash`** (not even a path-scoped `git stash push -- <dir>`).
  It reverts every other session's in-flight edits in those paths for as long as
  it is held, and the `pop` can clobber anything written meanwhile. Used it once
  to get a lint baseline; the tree survived, but the baseline was meaningless
  because it also stashed other sessions' work. To compare against HEAD, run the
  tool on `git show HEAD:<path>` written to a temp file instead.
- `git status` will list dozens of files you never touched. The session-start
  status snapshot goes stale within minutes. Scope every `git diff` to your own
  paths, and never `git add -A` / `git commit -a`.
- The server runs under `nest start --watch` (port **3011**, HTTPS), so `dist/`
  churns constantly and is not yours. A second, stale `node dist/src/main.js`
  sits on port 3000 — see [[erp-stale-dist-404]].

## Peer sessions stage YOUR files (2026-10-13)

Mid-task, another session's `git add` swept my unfinished print-designer edits
into its staged set (index == my working copy, no commit yet). Its next commit
would have shipped my half-done feature under its message. Check
`git diff --cached --name-only` before ending a long task and, if your own paths
are staged by someone else, `git restore --staged <your paths>` — only yours;
leave the other session's staged files alone. Also: nest's `deleteOutDir: true`
means `npm run build` wipes `dist/` under the running watch server, so don't
build — the watcher already compiled `dist/` (compare mtimes to confirm).

**Recurs (2026-10-15):** the same peer re-staged my files twice in one session. For a file BOTH
sessions edited, `git restore --staged` would drop their hunks too — instead rewrite the index
blob: `git show :path` → remove only my hunks → `git hash-object -w --stdin` →
`git update-index --cacheinfo`. Keep the hunk list in a re-runnable script.

