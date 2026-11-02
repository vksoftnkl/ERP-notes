---
name: node-port-of-go-dbsync
description: "This repo is a file-by-file Node.js port of the Go tool /home/vk/Dev/vknex-dbsync; Go changes must be ported, last synced 2026-10-22"
metadata:
  node_type: memory
  type: project
  originSessionId: 1e708f40-8b41-4218-acf1-21db9d6ab8c7
  modified: 2026-10-22T05:45:40.720Z
---

`/home/vk/Dev/vknex-dbsync nodejs` is a port of the Go tool in `/home/vk/Dev/vknex-dbsync` (same flags, env vars, SQL, sync state, slog log format; one JS file per Go file). The Go repo was being edited while the port was made (uncommitted changes, binary rebuilt 2026-10-22 10:46); the port matches that state, including the hash-join `uniqueMatchSQL` and `ensureMeta`.

**Why:** both versions must stay interchangeable on the same databases, and the Go side is the reference.

**How to apply:** before changing behaviour here, check the Go repo (`git status`, file mtimes) for newer changes and port them. Verify by running both side by side against scratch databases on the local PostgreSQL 18 (localhost:5432) and comparing per-table checksums and logs, from a folder without a `.env` (the Go folder's `.env` points at the live server).
