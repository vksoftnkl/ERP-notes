---
name: after-move-facts
description: "FROM 2026-10-29 the switch to Prathap's PC is FINAL. Qt 6.12.0, test/run against :3011, DB as the claude role on localhost. Supersedes 6.10.2 / :3012 / nexerp-api-ro / ssh facts in older notes"
metadata:
  type: project
---

**The user, 2026-10-29: "switch is final".** All work happens on vk-MS-7C95 (192.168.0.106, Prathap's PC), driven from the user's PC through VS Code Remote-SSH and RustDesk/AnyDesk. The user's PC copies (`/home/vk/Projects/Qt6/NexERP`, `nexerp-api-ro`, `Prathap.old`) are retired.

- **Paths, all local:**
  - Qt `/home/vk/Dev/erp/NexERP` (branch main);
  - NestJS `/home/vk/Dev/erp/ERP server` (DEV-v1);
  - React `/home/vk/Dev/erp/ERP client` (dev);
  - share `/home/vk/Shared/Prathap`.
  - No ssh or scp to .106 any more. "Upload to Prathap's folder" is now a plain local edit.
- **Qt 6.12.0 ONLY (user 10-29: "all move to 6.12.0 … no more 6.10.2").** Build dir `build/Desktop_Qt_6_12_0_Debug` (underscore before Debug). Build: `/home/vk/Qt/Tools/Ninja/ninja -C build/Desktop_Qt_6_12_0_Debug` (full rebuild ~2.5 min, 367 steps, clean on 10-29).
  - On 10-29 every runner in `tests/` and `scripts/deploy-linux.sh` was switched to default to `/home/vk/Qt/6.12.0/gcc_64` and the 6.12 build dir (`tests/till/snap.sh` private includes too). Uncommitted until the user says commit.
  - `build/Desktop_Qt_6_12_0_Debug/certs/rootCA.pem` was added 10-29: a copy of `ERP server/certs/rootCA.pem` (mkcert CA, same md5), which `appconfig.ini CaCertFile=certs/rootCA.pem` and the live tests need.
- **API: :3011 only. :3012 is gone (user: "no need of 3012").**
  - 10-29: the 20 Qt purchase tests no longer swap `:3011`→`:3012`.
  - The 19 share `purchase/test_*_api.py` files and `print/purchase_doc_templates.py` now use `https://192.168.0.106:3011/api/v1`.
  - Pre-edit copies are in that session's scratchpad `share-backup-2026-10-29/`.
  - :3011 is `npm run start:dev`, started detached 10-29: `NODE_EXTRA_CA_CERTS=/home/vk/.local/share/mkcert/rootCA.pem setsid nohup npm run start:dev > logs/api-3011-dev.log 2>&1 < /dev/null &`. After a src edit the old child often hangs: check that `ss -ltnp | grep ':3011 '` shows a NEW pid. pm2 `api` stays on :3000.
- **psql in tests = `-h localhost -U claude`.** pg_hba REFUSES `claude` from host 192.168.0.106 (this box's own LAN IP), while localhost asks for a password. On 10-29 every `psql -h 192.168.0.106 … -U claude` was rewritten to `-h localhost`: 53 Qt test/script files plus 36 share files (purchase API tests, grids/*_registration.py, print script). Without `~/.pgpass` (`localhost:5432:*:claude:<pw>`, which the user types) every psql read returns EMPTY, and live suites fail ONLY on DB checks. On 10-29, doc-keys Qt (10 fails) + API (IndexError) failed that way, while all screen and key checks passed.
- **DB: inspect as the `claude` role** (user), `psql -h localhost -U claude -d ERP`. The role is read-only (`pg_read_all_data`), and its password goes in `~/.pgpass`, which the user types themselves; never ask for it. Migrations run through Prisma with the server `.env` `postgres` URL ("postgres is fine").
- **Claude config:** start `claude-viji` (alias in `~/.bashrc`, `CLAUDE_CONFIG_DIR=/home/vk/.claude-viji`) from any of the repos. Plain `claude` = `~/.claude` = Prathap's config.
  - **Since 2026-10-30 this memory is the WHOLE product.** Prathap's ERP server and ERP client memories were copied into `server/` and `client/` here, and the cross-repo ones into `workspace/`.
  - The claude-viji memory dirs for ERP server, ERP client and `~/Dev/erp` are symlinks to this folder ([[project-map]]).

**10-29 later: `~/.pgpass` DONE.** The user ran a script that gave the `claude` role a new random password, stored only in `~/.pgpass` (`localhost:5432:*:claude:…`, mode 600). Old `claude` passwords no longer work. The classifier refuses Claude writing secret stores or its own settings, so for those, hand the user a script to run with `!`. Verified: purchase doc-keys suites ALL PASS on 6.12 + :3011 (API + Qt), DB checks included.

**Why:** older notes describe the two-PC setup (ssh, `nexerp-api-ro`, :3012 test instance, 6.10.2 kit), and following them now builds in a folder that doesn't exist or tests a server that isn't running.

**How to apply:** where an older note conflicts with this one, this one wins; fix the older note when you touch it. Related: [[move-to-prathap-pc]], [[prathap-server-access]], [[db-live-connection]].
