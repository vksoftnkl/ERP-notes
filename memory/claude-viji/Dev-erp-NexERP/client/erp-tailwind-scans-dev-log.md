---
name: erp-tailwind-scans-dev-log
description: "Tailwind v4 scans .next/dev/logs, so one bad CSS candidate wedges the dev server in a self-feeding 500 loop"
metadata: 
  node_type: memory
  type: project
  originSessionId: 7e179271-d1e4-4f58-8734-25ac61ad3a11
  modified: 2026-09-19T08:49:14.349Z
---

The client is on **Tailwind v4 with automatic source detection** (`app/globals.css` does
`@import "tailwindcss"` and never `@config`, so `tailwind.config.ts` is dead — its own comment says
so). That scan reaches **`.next/dev/logs/next-development.log`** and any build directory that is not
gitignored.

That makes a **self-feeding failure**: something puts a malformed class-like string where the scan
can see it → Tailwind emits a broken rule → `Parsing CSS source code failed` at `app/globals.css` →
Next **logs the error, ANSI colour codes and all, into that same log** → the next scan finds the
mangled token again. Every route then 500s, and restarting `next dev` does NOT clear it.

Seen 2026-09-19: running a second dev server with `NEXT_DIST_DIR=.next-verify` (not in
`.gitignore`) put coloured compile output where the scan could read it, and the error string
`z-[var(--erp-z<garbage>!)]` kept regenerating itself.

**Why:** the error names `app/globals.css:382`, a line that does not exist in that file — the file
is fine and the candidate came from somewhere else entirely, so the obvious hunt goes nowhere.

**How to apply:** only `rm -rf .next` **while the server is stopped** breaks the loop; truncating
the log or touching sources does not, because the candidate set is already in memory. Never point a
second dev server at a dist dir that `.gitignore` does not cover (`/.next/`, `/.next-build/`,
`/.next.previous/` are the covered ones). Related: [[erp-workspace-layout]],
[[erp-client-ui-browser-verification]].
