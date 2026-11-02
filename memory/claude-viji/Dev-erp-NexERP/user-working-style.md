---
name: user-working-style
description: How the user (viji, VK Softwares) works on NexERP — Qt first and best known, React follows after Qt is tested, short instructions, never make them repeat context
metadata:
  type: user
---

- **Qt is their strength and main focus.** The user knows Qt well and concentrates on the Qt desktop. React is applied AFTER the Qt version is built and tested (user, 2026-10-30: "I mainly concentrate only on Qt … have to apply React after Qt testing").
- **One person, all layers, through me:** DB (PostgreSQL), NestJS, Qt and React ([[project-map]]).
- **Don't make them remind you** (user: "I can't remember you every time, try to be smart").
  - At the start of any task, read [[session-resume]] and [[after-move-facts]], then the area index (`qt-db-index.md`, `server/INDEX.md`, `client/INDEX.md`, `workspace/INDEX.md`) before asking anything.
  - Search memory for the screen, menu id or table name being touched.
- **Instructions are short and often typed fast** ("do it", "commit all the files", "use agents"). Infer the intent from context, act, verify, and report. Ask only when a choice is genuinely theirs.
- **"Ran the cmd"** may not mean it ran here: give `!` commands and verify the effect ([[user-run-commands-with-bang]]).
- **Commits only on request:** Qt on main ([[feedback-commit-to-main]]), server DEV-v1, client dev. When asked only for a commit message, give plain text ([[commit-message-plain-text]], in `server/`).

**How to apply:**
- After every Qt feature or commit range, write down what React now owes. Put it in [[session-resume]] under "React owes", with the Qt commit ids.
- Once the user has tested the Qt side, propose (or, if asked, do) the React port with the same routes, JSON keys, keys and messages. Record it as `client/erp-qt-<sha>-port.md`, following the existing port notes.
- Never let a Qt-only change silently diverge from React. Note every deliberate difference in the port note.
