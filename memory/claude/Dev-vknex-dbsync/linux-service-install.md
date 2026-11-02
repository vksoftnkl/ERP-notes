---
name: linux-service-install
description: How the sync is installed on this machine (systemd user service with the erp-vps SSH tunnel) and how to redeploy without breaking the tunnel
metadata:
  node_type: memory
  type: project
  originSessionId: 5eb807e9-ff14-4960-9a43-004ddcd82534
  modified: 2026-10-23T13:49:40.363Z
---

On this machine vknex-dbsync runs as a systemd user service installed by `deploy/linux/install.sh erp-vps`; the live DB is reached through the `erp-vps` SSH tunnel unit (local port 5433). The status page is served at http://127.0.0.1:7645/ (SYNC_UI default); the desktop icon runs `control.sh open`.

**Why:** Running `install.sh` without the `erp-vps` argument removes the tunnel unit, and the sync then fails to reach the live DB. The installer (since 2026-10-23) also restarts a running service so the new build is used.

**How to apply:** After code changes, redeploy with `sh deploy/linux/install.sh erp-vps`, then check http://127.0.0.1:7645/api/status. Expect a 2–3 minute interval run to be rolled back by the restart; that is safe by design.
