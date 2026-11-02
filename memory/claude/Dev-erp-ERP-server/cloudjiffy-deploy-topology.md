---
name: cloudjiffy-deploy-topology
description: "Deployed API topology on CloudJiffy — nginx LB terminates TLS, app on plain HTTP port 3000 via PM2 ecosystem.config.js; mkcert cert was once uploaded as Custom SSL"
metadata: 
  node_type: memory
  type: project
  originSessionId: 1e29cb27-25d1-4475-a3cc-a0be68376150
  modified: 2026-08-19T04:48:07.283Z
---

The backend deploys to CloudJiffy (Jelastic) at vknext-timber.cloudjiffy.net (45.252.190.120). Topology: an nginx load balancer terminates TLS and proxies to the Node app, which must listen plain HTTP on port 3000 (the only port the Jelastic nft firewall whitelists). Runtime env comes from `ecosystem.config.js` (PM2), which pins `HTTPS_ENABLED=false`, `PORT=3000`, `REDIS_ENABLED=false`, and the production `CORS_ORIGINS` list — dotenv does not override values already in process.env, so deployed overrides belong there, not in `.env`.

**Why:** In Aug 2026 the local mkcert cert (`certs/localhost.crt`) had been uploaded as the environment's Custom SSL cert, so browsers got ERR_CERT_COMMON_NAME_INVALID; the fix is dashboard-side (remove Custom SSL, use Let's Encrypt add-on or Built-In SSL), not repo-side.

**How to apply:** For deployed-behavior changes (ports, TLS, CORS, env flags) edit `ecosystem.config.js`; for certificate/domain issues check the CloudJiffy dashboard SSL settings first — the repo has no deploy scripts or Jelastic CLI credentials. Related: [[local-https-mkcert-setup]].
