---
name: prod-db-access-rules
description: How to query the prod Postgres without materializing credentials, and that writes to the prod DB / deploys need the user's explicit go-ahead (auto-mode classifier blocks them)
metadata:
  type: feedback
---

On the prod VPS use `sudo -u postgres psql -d erp` (peer auth) for reads, and `SET ROLE erp_app` inside a transaction when the app role's privileges matter. Do not paste the DB password from /opt/erp-server/app/.env into commands.

**Why:** on 2026-10-06 the auto-mode classifier denied a command that inlined the erp_app password ("Credential Materialization"), and separately denied applying seed SQL to the prod DB ("Production Deploy"). Read-only inspection and rolled-back dry runs (BEGIN ... ROLLBACK) were allowed.
**How to apply:** do all analysis, regenerate files, and dry-run with ROLLBACK autonomously; then hand the user the exact apply/deploy command instead of running it. Related: [[nexerp-prod-and-dev-topology]].
