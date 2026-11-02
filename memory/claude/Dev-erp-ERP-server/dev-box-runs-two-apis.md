---
name: dev-box-runs-two-apis
description: "192.168.0.106 runs the API twice — PM2 \"api\" on :3000 (plain HTTP, production env, serves dist/) and `nest start --watch` on :3011 (HTTPS, .env) — an env change needs both restarted"
metadata:
  node_type: memory
  type: reference
  originSessionId: a8132dd9-0a3f-4d9b-b7b5-674d30e9b557
  modified: 2026-10-17T10:43:04.548Z
---

The dev box 192.168.0.106 (this machine) runs two copies of the API, both from
`/home/vk/Dev/erp/ERP server`, both reading its `.env` at boot (Nest ConfigModule, dotenv does not override
values PM2 already set):

- **PM2 `api`** (ecosystem.config.js) — `dist/src/main.js` on **:3000**, plain HTTP, `NODE_ENV=production`.
  It serves whatever `dist/` held at its last restart; the watcher keeps rewriting `dist/` underneath it.
  Restart: `npx pm2 restart api`.
- **`nest start --watch`** — in the user's own terminal, **:3011** HTTPS (`.env` PORT/HTTPS_*). A bare
  `touch` of a src file does NOT restart it (content unchanged → no compile); a real content change does.
  To bounce it for an env change, append a newline to `src/main.ts`, wait for the new pid on :3011, then
  restore the file byte-for-byte (done 2026-10-03, notes 82).

**The Qt client talks to :3011**, not :3000 (notes 109, 2026-10-12): a PM2 restart alone does not
reach it. "property X should not exist" from the client = the :3011 process predates the DTO.

**The watcher can die and leave its app running.** On 2026-10-12 `nest start --watch` exited around
16:34; its child kept serving :3011 (reparented to `systemd --user`) on the 16:34 build for hours.
Fixed by `kill` of the orphan, then `setsid nohup npm run start:dev > logs/api-3011-watch.log 2>&1
< /dev/null & disown` from the repo (detached; logs/ is gitignored). Before calling a change live,
check the route's field in `/api/docs-json` on BOTH ports.
Happened again 2026-10-13: an orphan from 09:29 (parent `systemd --user`; no autostart unit found)
made the user's fresh `npm run start:dev` die with `EADDRINUSE 0.0.0.0:3011`. Find it with
`ss -ltnp | grep ':3011 '`, kill it, then the watcher needs a restart (or a real src change) to relaunch.

**A rebuild can hang the restart (2026-10-16 19:13).** The watcher rebuilt and signalled the old
child, which closed its :3011 listener but never exited (one keep-alive connection, probably the
Qt client's, held its graceful shutdown); SIGTERM did nothing, and the new build waits for the old
child to exit, so :3011 answered nothing (curl 000) for ~5 min. `kill -9 <old child pid>`
(`pgrep -P <nest pid>`): the watcher spawned the new build at once. After every src edit, check
`ss -ltnp | grep ':3011 '` shows a NEW pid before calling it deployed.
On 2026-10-17 it hung on BOTH rebuilds of the day (12:26 and 16:05). With the Qt client connected
it is now the norm, not a fluke: after any src edit, expect to `kill -9` the old child.

**How to apply:** after any `.env` change, restart both, and check `ss -ltnp | grep -E ':3000 |:3011 '`
for new pids. Related: [[local-https-mkcert-setup]], [[gst-config-module]].

**Restart recipe (2026-10-26).** :3011 = `npm run start:dev` (nest --watch, .env PORT=3011 HTTPS) needing
`NODE_EXTRA_CA_CERTS=/home/vk/.local/share/mkcert/rootCA.pem`. A watcher started inside a Claude/IDE
shell dies with it and leaves an ORPHAN child serving stale dist (seen: 09:35 build still live hours
later, source edits never went live). Start it detached: `setsid nohup npm run start:dev >
logs/api-3011-dev.log 2>&1 < /dev/null &`. To stop: `kill -KILL -<pgid>` of the watcher group — SIGTERM
frees the port but leaves both node processes hung. :3000 = `pm2 restart api --update-env` (boot seeds run
there: check "Seeding finished … 0 failed" in ~/.pm2/logs/api-out-0.log).

**The watch child hangs on EVERY recompile (seen 12:01, 2026-10-26):** a source save → "Found 0 errors" →
the old app child closes :3011 but never exits, so the new one never starts and :3011 answers nothing.
Fix: `kill -KILL <the dist/src/main child>`; the watcher then spawns the fresh build. Check after every edit.
Find the child with `ss -ltnp | grep :3011` or `ps -eo pid,cmd | grep -- "--enable-source-maps" | grep -v grep` — NOT `pgrep -f <path>`, which also matches the shell running it, so a `kill -KILL $(pgrep …)` kills your own command (seen 2026-10-26).

**TWO watchers seen at once (2026-10-26 19:27):** `nest start --watch` pid 257066 (started ~11:30, child
367172 hung since 13:25, listening on nothing) AND pid 566760 (child serving :3011). Both compile into the
same dist/ and both relaunch a child per change — one child loses the port race, a likely cause of the
"hangs on every recompile" above (unverified). Left running and reported to the user; if the hangs
continue, kill the older watcher's group and keep one.
