---
name: sales-actor-name-vs-actor-id
description: "Text *_modified_by columns on sales headers (sb_/so_modified_by) take SalesCallContext.actorName (login name), never ctx.actor (uuid); uuid audit columns keep ctx.actor"
metadata:
  node_type: memory
  type: project
  originSessionId: 419e6c09-abb0-4b79-b609-bd0e7ad0cd6e
  modified: 2026-09-24T12:00:19.036Z
---

Since 2026-09-24, `SalesCallContext` carries `actorName` (`user_master.usr_login_name`, falling back to the id), and `SalesContextService.actorName(tx)` returns it outside `resolve()`. `/bills/create` stores the NAME the client sends as `sbCreatedBy`, so any server-side verb that stamps `sb_modified_by` / `so_modified_by` must use `actorName`, or the audit line reads "modified 019e441b-…".

**Why:** Qt ticket item #3 (bil00155). The column is text, so a uuid gets in without complaint.

**How to apply:** use `actorName` only for the text header columns. Keep `ctx.actor` for uuid columns (abl_*, atc_*, txn_status_log changed_by, audit userId). Still on the uuid as of that date: the fulfilment rollups in sale-order.service.ts (softDelete, cancelOrderOpenLines, syncOneOrderFulfilment), which read `requestContextService.getUserId()` directly.
