---
name: erp-stale-dist-404
description: New ERP server route 404s locally despite correct code — the running dist build is stale; rebuild + restart the process bound to the API port
metadata: 
  node_type: memory
  type: project
  originSessionId: 97302a74-f73d-4e69-ac14-727accfd6235
---

Symptom: a freshly added ERP server route (e.g. `GET /api/v1/item-unit-conversions/get`) returns **404** locally even though the controller/module exist and are registered in `app.module.ts`.

Cause: the running process is a **stale build**. On this machine, port 3011 (the client's API target) is served by a manually launched `node --enable-source-maps dist/src/main` (= `npm run start:prod`) that does **not** auto-reload. A `nest start --watch` process is often also alive but **not bound to any port** (it lost the 3011 bind), so it doesn't help. If the prod process started before the new module was compiled into `dist`, it serves old in-memory code → 404.

Diagnosis: probe unauthenticated — an existing route returns **401** ("Missing or invalid Authorization header"), a missing one returns **404**. Compare the broken route against a known-good sibling (e.g. `/item-prices/get`).

Fix: `npm run build` (in `ERP server`) then restart the process bound to 3011 (`node dist/src/main`). Verify with [[erp-auth-endpoint-verification]] (expect 200). Companion to [[erp-server-500-debugging]].
