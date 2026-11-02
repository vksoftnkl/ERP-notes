---
name: api-endpoint-docs
description: NexERP backend API endpoints — Swagger UI at https://192.168.0.106:3011/api/docs, machine-readable spec at /api/docs-json; all routes live under /api/v1
metadata:
  type: reference
---

The authoritative list of NexERP backend API endpoints is the live NestJS
Swagger page at https://192.168.0.106:3011/api/docs. The OpenAPI JSON is at
https://192.168.0.106:3011/api/docs-json (~800 KB) — fetch that, not the HTML,
when looking up a route:
`curl -k -s https://192.168.0.106:3011/api/docs-json` (`-k` is required: self-signed cert).

Spec as of 2026-08-24: "ERP Server API 1.0.0", 267 paths, all prefixed
`/api/v1/<resource>` across ~78 resource groups (items, customers, suppliers,
quotations, sale-orders, bills, tender-*, charges, promotion-*, item-stock-*,
gsp-*, master-lookups, grid-details, ui-table-masters, txn-holds, auth, …).
Auth is a bearer `access-token` security scheme.

**Why:** Stated by the user on 2026-08-24 as the URL to use for API endpoints
from now on; endpoint shapes are documented nowhere in the repo.

**Paging limits are enforced, and a breach is a silent dead screen.**
`/dropdown-details/run` declares `limit` as `minimum: 1, maximum: 100` — ask
for 200 "to be safe" and NestJS answers **400**, the lookup's error branch
runs, and the widget it fed shows an empty list that looks exactly like a
broken widget. Found on the tax rate master's Role combo (2026-09-12), which
reported as "I can't choose anything in the grid".
Read the parameter schema in docs-json before hand-writing a query string, and
prefer the shared builders (`NexDropdownApi::run`, `NexUiTableApi::get`) which
already send page / limit / dropdown_param in the shapes the runner expects.
The login DTO is `usrLoginName` / `usrPassword` (changed 2026-09-11 from
username/password).

**How to apply:** Before adding or editing a request path in
`src/core/api/api_client.cpp` (or its callers), pull the route, method, and
payload schema out of docs-json rather than guessing. It is a LAN host, so it
is only reachable while on the user's network — if curl fails, ask the user to
paste the relevant part. Related: [[sales-backend-schema-design]],
[[gsp-integration-schema]], [[promotion-schema-design]], [[stock-schema-design]].
