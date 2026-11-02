---
name: erp-frontend-vps-deploy
description: Frontend runs on the same VPS as the API (169.58.213.171) under PM2 as erp-front on :3001, same-origin behind nginx
metadata:
  type: project
---

Deployed 2026-08-21. The Next frontend shares the API's VPS (Ubuntu 24.04,
169.58.213.171) rather than getting its own box — see [[erp-cloudjiffy-deploy]]
for the older, separate CloudJiffy target, which the repo's own
`ecosystem.config.cjs` + `scripts/deploy-on-node.sh` still describe and which is
NOT what runs on the VPS.

Layout (mirrors the server repo's `/opt/erp-server`):
- `/opt/erp-front/app` — clone of `github.com/vksoftnkl/ERP-frontend` branch `dev` (public repo, HTTPS clone, pull-only)
- `/opt/erp-front/app/.env.production` — holds `NEXT_PUBLIC_API_BASE=https://169-58-213-171.sslip.io/api/v1`, mode 600
- `/opt/erp-front/ecosystem.config.js` — PM2 config, **outside** the git tree so `git reset --hard` can't clobber it; runs `npm start` (not `next start`) so `scripts/ensure-build.js` gates the bundle; `PORT: 3001`
- `/opt/erp-front/bin/deploy.sh [branch]` — fetch → reset → conditional `npm ci --omit=dev` → build into `.next-build` → swap → `pm2 reload` → health-gate
- systemd unit `pm2-erp-front.service` (user `erp-front`), enabled; `erp` user's `pm2-erp.service` runs the API separately

Redeploy: `ssh root@169.58.213.171 'cd /opt/erp-front && sudo -u erp-front HOME=/opt/erp-front /opt/erp-front/bin/deploy.sh dev'`

**Why:** frontend and API share one origin, so there is no CORS entry and no
second TLS certificate — the existing sslip.io Let's Encrypt cert covers both.

**How to apply:**
- nginx routes by prefix in `/etc/nginx/snippets/erp-proxy.conf`: `/api/v1` and `/api/docs` → `erp_api` (:3000), everything else → `erp_front` (:3001). The API location MUST stay `/api/v1`, never `/api` — the frontend owns `/api/gst` and `/api/tamil` route handlers. Shared headers live in `erp-proxy-headers.conf`; backups in `/root/nginx-backup/`.
- PM2 commands as `erp-front` must run from an accessible cwd (`cd /opt/erp-front` first). Running them from root's `$HOME` makes the daemon spawn fail with a misleading `spawn /usr/bin/node EACCES`.
- `.env.production` is read by Next, not exported into the shell, so `ensure-build.js`'s NEXT_PUBLIC_* fingerprint hashes an *empty* set at runtime. Editing `.env.production` alone will NOT trigger a rebuild on restart — rerun `deploy.sh` (its scratch dist dir always builds fresh).
- These deploy assets exist **only on the VPS**, not in the client repo.
