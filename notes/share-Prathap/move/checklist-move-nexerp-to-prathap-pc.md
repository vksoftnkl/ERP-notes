# Moving NexERP development to Prathap's computer — checklist

Rev 1 · 2026-10-28 · picture: `move_plan.png` (same folder)

**What moves (user, 2026-10-28): only the Qt project and the plans** (the share), plus Claude's
memory and the app settings they need.

**Status 2026-10-28:** the **share is COPIED** to Prathap's PC (`/home/vk/Shared/Prathap`, 866 files,
394 MB, checksums identical). **SWITCHED 18:35: Prathap's copy is the MASTER.** On your PC the same
path is now an sshfs mount of it; the old local folder is `/home/vk/Shared/Prathap.old` (fallback, §9). **Qt repo COPIED 2026-10-28 to `/home/vk/Dev/erp/NexERP`** (user's choice of
folder — NOT the old path), without `build/`: 7613 files, 878 MB, checksums identical, git clean at `e0c56ef`.
Because the path changed: Claude's memory goes to `/home/vk/.claude-viji/projects/-home-vk-Dev-erp-NexERP/`
(not `-home-vk-Projects-Qt6-NexERP`). Only one tracked file (a `.pyc`) holds the old repo path; the test runners use
`/home/vk/Qt/6.10.2`, which is unchanged. `/home/vk/Dev/erp` is itself Prathap's git repo — `NexERP/` shows there
as untracked (`?? NexERP/`). They go to Prathap's computer (`vk-MS-7C95`, 192.168.0.106).

**What does NOT move:** the NestJS backend, the React code and the database. Their originals are already
on Prathap's computer and stay exactly as they are. Nothing of theirs is copied, re-installed or
restarted. The copies on your PC (`nexerp-api-ro`, `nexerp-client`) are simply left behind.
You control everything from your own PC through **VS Code + Remote-SSH**, and see the running Qt app
through **RustDesk / AnyDesk**.

**The one rule that makes the move easy:** keep every path **exactly the same** as today
(`/home/vk/Projects/Qt6/NexERP`, `/home/vk/Shared/Prathap`). Both machines log in as `vk`, so the
94 test scripts, the deploy script and Claude's memory folder keep working without a single edit.

**The second rule:** this is a **move, not an upgrade**. Build with the same Qt 6.10.2 first.
Moving to Qt 6.12 is a separate step, after everything is green on the new machine.

---

## 0. What was checked today (both machines)

| Item | Your PC today | Prathap's PC today |
|---|---|---|
| OS | Ubuntu (Wayland) | Ubuntu 24.04.4, 16 cores, 31 GB RAM (**20 GB already in use**), 62 GB disk free |
| Qt | 6.10.2 + 5.15.0 in `~/Qt` | **6.12.0 + 6.8.3** — **no 6.10.2** |
| Qt tools | `~/Qt/Tools` (CMake, Ninja, QtCreator) | same set present |
| Qt repo | `/home/vk/Projects/Qt6/NexERP`, 878 MB without build, `build/` = **36 GB**, `dist/` 302 MB | not there |
| NestJS | read-only copy `Projects/Qt6/nexerp-api-ro` | real repo `~/Dev/erp/ERP server` (DEV-v1), pm2 `api`, `erp-purchase` :3012, Prathap's `nest --watch` :3011 |
| React | (no code; `react` in the Qt repo is an **empty file**) | `~/Dev/erp/ERP client` (Next.js, branch `dev`, github ERP-frontend) |
| Database | `~/.pgpass` → 192.168.0.106 (roles `claude`, `postgres` localhost) | PostgreSQL 18.6 runs **here** — no `~/.pgpass` for `vk` yet |
| Share | `/home/vk/Shared/Prathap` (393 MB), Samba share `[Shared]` — **not in git, no history** | not there (Prathap reads it over Samba) |
| Claude | CLI + memory `~/.claude/projects/-home-vk-Projects-Qt6-NexERP/` (748 MB incl. old sessions) | **Prathap's own** `claude` CLI and `~/.claude` already exist — same Linux user `vk` |
| Editors | Qt Creator | VS Code, Qt Creator |
| Remote desktop | — | RustDesk + AnyDesk installed; Xorg and Wayland sessions available |
| App settings | `~/.config/VK Softwares/Nex ERP.conf`, `appconfig.ini` | — |

Found while checking (not part of the move, tell Prathap): pm2 `erp-purchase` shows **153 restarts**.

---

## 1. Decide first (you + Prathap)

- [x] **D1. DECIDED 2026-10-28: option (a), own config folder.** Prathap keeps `~/.claude`; your
      Claude uses **`/home/vk/.claude-viji`**, under the same Linux user `vk`, so all paths stay identical.
      `CLAUDE_CONFIG_DIR` is confirmed in Claude Code 2.1.312.
  - Shell alias on Prathap's PC (`~/.bashrc`): `alias claude-viji='CLAUDE_CONFIG_DIR=/home/vk/.claude-viji claude'`
    — plain `claude` stays Prathap's.
  - VS Code (remote): set the env `CLAUDE_CONFIG_DIR=/home/vk/.claude-viji` for the Claude Code
    extension (in its settings, or in the Remote-SSH server env), then sign in with **your** account.
  - On the day, **before copying memory**: start `claude-viji` once, sign in, check
    `ls /home/vk/.claude-viji` fills up and `~/.claude` (Prathap's) does not change
    (compare `ls -la ~/.claude` before and after).
  - Memory goes to `/home/vk/.claude-viji/projects/-home-vk-Projects-Qt6-NexERP/`,
    settings to `/home/vk/.claude-viji/settings.json` (§5).
  - Never run plain `claude` in the NexERP folder there: that starts Prathap's Claude with an empty
    memory.
  - *(Option (b), a new Linux user, was dropped because every `/home/vk/...` path would change.)*
- [ ] **D2. Who edits what.** Two people + two agents in the same folders will overwrite each other.
      Agree in writing:
  - Qt repo → you (main);
  - the share → both of you (who writes which folder?);
  - NestJS and React are Prathap's and are not part of this move. Purchase work on the server keeps
    today's rule: the agent commits only its own files, by path.
- [x] **D3. DECIDED 2026-10-28: the share lives on Prathap's PC, and you reach it over SSH (not Samba).**
      Its one home is `/home/vk/Shared/Prathap` on his PC, at the same path. Prathap opens it locally.
      From your PC:
  - **VS Code Remote-SSH** shows it directly (add the folder to the workspace). Nothing to install.
  - Optional, to open the PNGs and docs with your own apps: **sshfs**. It is not installed today:
    `sudo apt install sshfs`, then
    `sshfs vk@192.168.0.106:/home/vk/Shared/Prathap /home/vk/Shared/Prathap -o reconnect,ServerAliveInterval=15`.
    The mount goes to the same path, so links and scripts keep working. The old local copy must be
    renamed `Prathap.old` first (§9). Unmount with `fusermount -u /home/vk/Shared/Prathap`.
  - **Samba is not used for the share any more.** Your PC's `[Shared]` export keeps your other files
    and simply no longer contains `Prathap/`.
  - Still open: put it **under git** (private repo)? Recommended, because today it has no history
    (see the 2026-09-24 truncation, 14 of 18 files restored by luck).
- [ ] **D4. Is your PC still a build machine?** Recommended: no (one place only), but keep the old
      copy read-only for 2 weeks as a fallback (§9).
- [ ] **D5. RAM.** 20 of 31 GB is already used there. A Qt build (`ninja -j16`) + Claude agents +
      Prathap's VS Code + pm2 + Postgres may be tight. Agree to build with `-j8`, or add RAM.
- [x] **D6. DECIDED 2026-10-28:** `nexerp-client` and `nexerp-api-ro` on your PC are **not moved**. The
      React and NestJS originals are on Prathap's computer. The local copies can be deleted in §9.

## 2. Prepare Prathap's PC (before move day)

- [ ] **Qt 6.10.2** via `~/Qt/MaintenanceTool`: Desktop gcc_64 + **Qt PDF**, **Qt Multimedia**,
      Qt Print Support, plus Sources/Debug info if you debug into Qt. Keep 6.12.0 as is.
- [ ] Check `~/Qt/Tools/Ninja/ninja` and `~/Qt/Tools/CMake/bin/cmake` work (not on PATH — the test
      runners call them by full path).
- [ ] Build packages: `sudo apt install build-essential libgl1-mesa-dev libxkbcommon-dev libcups2-dev`
      (+ whatever the first CMake run complains about).
- [ ] Fonts: the Inter fonts ship inside the repo (`src/shared/fonts/inter`); DejaVu is used by the
      mockup scripts — `fc-list | grep -i dejavu`.
- [ ] Python 3 + Pillow (mockups) — `python3 -c "import PIL"`; `requests`/`urllib3` if the API test
      scripts need them.
- [ ] `~/.pgpass` for `vk`: lines for `192.168.0.106:5432:ERP:claude`, `…:3.0:claude`,
      `localhost:5432:*:postgres` — **type the passwords yourself**, `chmod 600`. Never into a doc or chat.
- [ ] `psql -h 192.168.0.106 -U claude -d ERP -c 'select 1'` works from that machine.
- [ ] GitHub: an SSH key on that machine that can **push** to `git@github.com:vksoftnkl/NexERP.git`
      (`ssh -T git@github.com`).
- [ ] Git identity **inside the Qt repo only** (not global — global is Prathap's):
      `git config user.name "Viji"` and `user.email vksoftwaresnkl@gmail.com`.
- [ ] Static IP 192.168.0.106 stays (94 files and the app config point at it).
- [ ] Remote desktop: RustDesk or AnyDesk login set up from your PC; decide Xorg session if the
      Wayland session blocks remote control.
- [ ] Disk: 62 GB free; a fresh Debug build is ~36 GB today → **clean old build dirs first**
      (only `Desktop_Qt_6_10_2-Debug` is needed; `6_10_3-Debug` and `Release` are not).

## 3. Your PC (the controller)

- [ ] Install VS Code (`sudo snap install code --classic`), extensions **Remote - SSH** and **Claude Code**.
- [ ] `~/.ssh/config` entry `Host prathap-pc` → `HostName 192.168.0.106`, `User vk` (key auth already works).
- [ ] VS Code: *Remote-SSH: Connect to Host* → open `/home/vk/Projects/Qt6/NexERP`
      and add the share `/home/vk/Shared/Prathap` to the workspace (D3).
- [ ] In the remote VS Code, set the Claude Code environment to D1's choice
      (`CLAUDE_CONFIG_DIR`), sign in with **your** account.
- [ ] RustDesk/AnyDesk client to see the Qt window.

## 4. Freeze (move day, morning)

- [ ] Tell Prathap: no edits in the share for ~1 hour.
- [ ] Qt repo: `git status` clean, `git worktree list` = only main (today: clean, HEAD `e0c56ef`).
      Commit or stash anything open, `git push`.
- [ ] Note what is running on .106: pm2 `api`, `erp-purchase` (:3012), Prathap's `nest --watch` (:3011).
      Nothing changes there — the server does not move.
- [ ] Ask Claude to write today's `session-resume` memory (what is in flight) before copying.
- [ ] Close Claude sessions on your PC.

## 5. Move (copy, never cut)

Run from **your PC**. `rsync` keeps untracked files (`plan.md`, `original.sql`, `quotation_payload.json`,
`.qtcreator/`, `.claude/settings.local.json`) that a fresh `git clone` would lose.

- [ ] **Qt repo** (without the 36 GB build; dist optional):
      `rsync -az --exclude build/ /home/vk/Projects/Qt6/NexERP/ vk@192.168.0.106:/home/vk/Projects/Qt6/NexERP/`
- [ ] **Share:** `rsync -az /home/vk/Shared/Prathap/ vk@192.168.0.106:/home/vk/Shared/Prathap/`
      (only `Prathap/` — the rest of `~/Shared` is your other customers' files and stays).
- [ ] **Claude memory + project history** → into `/home/vk/.claude-viji` (D1):
      `~/.claude/projects/-home-vk-Projects-Qt6-NexERP/` → `/home/vk/.claude-viji/projects/-home-vk-Projects-Qt6-NexERP/`.
      Same repo path ⇒ same folder name ⇒ the memory loads by itself.
- [ ] **Claude user settings** `~/.claude/settings.json` → `/home/vk/.claude-viji/settings.json`
      (model, deny rules, auto-mode notes). Then update it (§6).
- [ ] **App settings:** `~/.config/VK Softwares/Nex ERP.conf` and the `appconfig.ini` the app reads
      (+ `certs/rootCA.pem` if `CaCertFile` points at it).
- [ ] **Not copied:** `build/` (rebuild), `nexerp-api-ro/` (the real server folder is there now),
      `react` empty file (delete from git later), `~/.pgpass` (re-typed in §2).

## 6. Fix-ups on the new machine

- [ ] Claude `settings.json` deny rule: today it blocks edits in `nexerp-api-ro`. On the new machine
      the real `ERP server` folder sits next door → add a **deny rule for Prathap's files you must
      not touch**, or a rule in memory, so the agent keeps "commit only my files by path".
- [ ] Remind Claude of the new facts (it will update memory): DB/API are local now, the share path
      is the same, `nexerp-api-ro` is retired, `ssh` to .106 is no longer needed.
- [ ] Share under git (D3): `git init` in `/home/vk/Shared/Prathap`, first commit, private remote.
- [ ] Share access (D3): on your PC, add `/home/vk/Shared/Prathap` to the Remote-SSH workspace;
      optional sshfs mount (D3) after renaming the local copy. No Samba export on his machine.
- [ ] Qt Creator on that machine: open `CMakeLists.txt`, kit **Desktop Qt 6.10.2**, build dir name
      **`build/Desktop_Qt_6_10_2-Debug`** (the test runners expect exactly that name).

## 7. Verify (do not skip — in this order)

- [ ] **Build:** `~/Qt/Tools/CMake/bin/cmake -S . -B build/Desktop_Qt_6_10_2-Debug -G Ninja`
      then `~/Qt/Tools/Ninja/ninja -C build/Desktop_Qt_6_10_2-Debug` — zero errors.
- [ ] **App runs** (through RustDesk): log in, open Purchase Entry (menu 24), a Sale Bill, a report.
- [ ] **Printing:** print preview of one bill (PDF + CUPS plugin).
- [ ] **Qt tests:** `SRC=test_purchase_invoice EXTRA_LIBS=-lQt6Test tests/purchase/run.sh` green.
- [ ] **API tests:** `python3 -I /home/vk/Shared/Prathap/purchase/test_purchase_invoice_api.py` green.
- [ ] **DB:** `psql -h 192.168.0.106 -U claude -d ERP` works from Claude's shell.
- [ ] **Claude memory:** new session, ask "where did we stop?" — it must answer from `session-resume`.
- [ ] **Git push** of a trivial commit from the new machine works (then nothing else changes).
- [ ] **dist bundle:** `BUILD_DIR=$PWD/build/Desktop_Qt_6_10_2-Debug bash scripts/deploy-linux.sh`,
      `ldd` shows no "not found".
- [ ] **Offscreen:** `QT_QPA_PLATFORM=offscreen` test run works (it is how agents test screens).

## 8. Backups (everything is on ONE machine now)

One disk failure would lose code, DB, share and memory together. Set up before you rely on it:

- [ ] Git repos pushed daily: NexERP, ERP-Server, ERP-frontend, **share** (new).
- [ ] Nightly `pg_dump` of `ERP` to another machine (your PC is fine).
- [ ] Nightly copy of Claude's memory folder to your PC.
- [ ] Prathap's ~128 **uncommitted** server files have no backup at all today — ask him to commit or copy.

## 9. Retire the old setup (after 2 green weeks)

- [ ] Your PC copy → rename `NexERP.old`, read-only; delete only after 2 weeks with no fallback needed.
- [ ] Old Claude memory folder on your PC: keep a zip as backup, then remove.
- [ ] Local `/home/vk/Shared/Prathap.old` on your PC → zip as backup, then delete (the share is reached over SSH now).
- [ ] `nexerp-api-ro` and `nexerp-client` on your PC → delete whenever you like (D6; the originals are on Prathap's PC).

## 10. After the move (separate steps, later)

- [ ] Qt **6.12 LTS**: new kit + `build/Desktop_Qt_6_12_x-Debug`; the 69 test files hard-code
      `/home/vk/Qt/6.10.2` → change them to one `QT_DIR` setting first, then switch.
- [ ] Agent team files (`.claude/agents/`) — only if you want them (see `../agents/`).
- [ ] Remove the empty `react` file from the Qt repo.

## Rollback

If §7 fails and can't be fixed the same day: nothing is deleted before §9 — close the remote window and keep
working on your PC exactly as before. Pull any commits made on the new machine with `git pull`.
