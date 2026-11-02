---
name: move-to-prathap-pc
description: 2026-10-28 plan to move all NexERP dev (Qt repo, share, Claude memory) onto Prathap's PC .106 — checklist + PNG in share move/; nothing moved yet
metadata:
  type: project
---
2026-10-28 the user decided to move the Qt project to Prathap's computer (vk-MS-7C95, 192.168.0.106) so Qt, NestJS, React, DB, share and Claude memory are in one place; the user drives it from their PC via VS Code Remote-SSH + RustDesk/AnyDesk.

Written: `share move/checklist-move-nexerp-to-prathap-pc.md` (§0 facts, §1 decisions D1–D6, §2–§10 steps, rollback) + `move_plan.png` (+ .py). **Nothing moved yet. D1 DECIDED 10-28: option (a) — my Claude uses CLAUDE_CONFIG_DIR=/home/vk/.claude-viji (alias claude-viji) on his PC; ~/.claude stays Prathap's; never run plain `claude` in NexERP there. D3 DECIDED 10-28: share lives on his PC at the same path, reached over SSH (Remote-SSH workspace, optional sshfs at same path), no Samba; share-under-git still unanswered. D2, D4, D5 open.** **SCOPE (user 10-28): move ONLY the Qt project + the plans/share (+ my memory, app settings); NestJS, React and DB are NOT moved — originals are on Prathap's PC (D6: nexerp-api-ro / nexerp-client left on the user's PC).**

Key facts found: his PC has Qt 6.12.0 + 6.8.3 only (no 6.10.2 — 69 test files hard-code /home/vk/Qt/6.10.2); 20 of 31 GB RAM used; Prathap's own `claude` + ~/.claude exist under the SAME Linux user vk (D1: give mine its own CLAUDE_CONFIG_DIR); no ~/.pgpass for vk there; React = ~/Dev/erp/ERP client (branch dev); `react` in the Qt repo is an empty file; build/ is 36 GB (rebuild, don't copy).

**How to apply:** keep paths identical (/home/vk/Projects/Qt6/NexERP, /home/vk/Shared/Prathap) so memory folder name, tests and scripts need no edits; move first, Qt 6.12 later. After the move, update [[db-live-connection]], [[prathap-server-access]], [[nestjs-source-readonly]] (api-ro retired).

**10-28 later:** user narrowed it again — I move ONLY the plans (share); the user copies the Qt repo manually. Share COPIED 10-28 to vk@192.168.0.106:/home/vk/Shared/Prathap (rsync -a --mkpath, 866 files, md5 of all files identical). Local copy untouched → TWO copies exist; until the user picks the master (plan: his copy, local renamed Prathap.old + sshfs mount at the same path), writes to the local share do NOT reach his copy.

**10-28 18:35 SWITCHED — Prathap's copy is the MASTER.** User's PC: `/home/vk/Shared/Prathap` is an sshfs mount of vk@192.168.0.106:/home/vk/Shared/Prathap (-o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3,BatchMode=yes); old local copy = `/home/vk/Shared/Prathap.old` (fallback, do NOT write there). Mount is NOT in fstab → lost on reboot: if the folder is empty, check `findmnt /home/vk/Shared/Prathap` and remount with the same sshfs line. .106 off = share unreachable. Verified: write via mount → visible on .106.

**10-28 Qt repo COPIED by me (user asked) to vk@192.168.0.106:/home/vk/Dev/erp/NexERP** (rsync -a, excluded /build/; 7613 files md5-identical; git clean e0c56ef). PATH CHANGED vs the same-path rule → memory must go to ~/.claude-viji/projects/-home-vk-Dev-erp-NexERP/. /home/vk/Dev/erp is Prathap's own git repo: NexERP/ shows as `?? NexERP/` — a `git add -A` there would swallow it; FIXED 10-28 (user OK): `NexERP/` added to /home/vk/Dev/erp/.git/info/exclude, git status no longer lists it. Local repo on the user's PC is still the working one until they switch; two copies now.

**10-29 claude-viji SET UP on .106 (user asked).** `~/.claude-viji` created; alias appended to his `~/.bashrc` (backup `~/.bashrc.bak-2026-10-29`): `alias claude-viji='CLAUDE_CONFIG_DIR=/home/vk/.claude-viji /home/vk/.local/bin/claude'` (full path: `claude` is not on PATH in non-login shells there). Whole project folder (memory 207 files md5-identical + session history, 750 MB) copied to `~/.claude-viji/projects/-home-vk-Dev-erp-NexERP/`; `settings.json` copied with the trusted-repo path changed to /home/vk/Dev/erp/NexERP (the stale nexerp-api-ro deny rule kept, harmless). NOT done: sign-in (user must run `claude-viji` once and log in with their account), VS Code extension env, §6 deny rule for Prathap's `ERP server`. Memory notes still name /home/vk/Projects/Qt6/NexERP — fix as found. Until the user switches, THIS PC's memory is the live one; the .106 copy is a 10-29 snapshot.

**10-29 SWITCH FINAL (user).** .106 is the only workplace from now on. Decisions: stay on Qt 6.12, test on :3011, DB as the claude role. See [[after-move-facts]].
