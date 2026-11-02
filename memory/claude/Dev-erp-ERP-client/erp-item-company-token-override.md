---
name: erp-item-company-token-override
description: "Server forces item_company_id to the JWT's company while child rows keep the form's company — never compare the two client-side"
metadata: 
  node_type: memory
  type: project
  originSessionId: 5dbe0d3c-1cce-45d5-a65d-e7740473d838
  modified: 2026-07-18T06:28:36.621Z
---

On save, `items-master.service.ts` (`applyOptionalFields`) **overwrites `item_company_id` with the access token's `company_id`**, ignoring whatever the form sent — "the owning company comes from the caller's access token, never the client body". Child tables get no such override: `item_price_master.ipm_company_id` (and siblings) keep the company picked in the form / header company selector.

So on a company-bound token the parent and its child rows legitimately carry **different** company ids (e.g. item = THE SCM SILK from the token, price rows = Acme Foods from the header selector).

**Why:** any client-side filter of the form `child.*_company_id === item_company_id` silently drops every child row. That is exactly what emptied the Item Master price list on edit (fixed 2026-07-18 by dropping the company comparison from `filterItemPriceRowsByScope`; branch-only, blank = item-wide).

**How to apply:** scope child rows by branch (or not at all — they are already scoped by the item FK), never by company. The read comment in the service mentions a `tenant-scope.extension.ts` that does **not** exist, so reads are not auto-scoped either. Related: [[erp-workspace-layout]], [[erp-item-price-unit-id-is-iuc-id]].
