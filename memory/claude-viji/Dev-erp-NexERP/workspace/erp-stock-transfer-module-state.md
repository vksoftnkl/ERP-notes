---
name: erp-stock-transfer-module-state
description: "ERP stock-transfer module: two SQL bugs found+fixed (uuid=text, item_name); despatch/receive blocked by undeployed SQL engine; TRANSFER wrongly gets the opening-uniqueness preflight"
metadata: 
  node_type: memory
  type: project
  originSessionId: 272b4d7c-aa7f-4ad6-a475-2899a9c30282
  modified: 2026-09-09T11:43:01.356Z
---

Tested the whole stock-transfer module (`src/modules/stocks/stock-transfer`) over HTTP on 2026-09-09 via `test/stock-transfer-http.e2e-spec.ts` (18 cases, net-zero: seeds stock with a physical overage, reverses it in teardown). Reads, save, validate, delete, auth and all 404 paths work. Findings:

**Two real SQL bugs found and FIXED in `stock-transfer.service.ts`:**
1. `assertLotsAndStock` compared `uuid` columns (`slt_id`, `sbl_lot_id`) to text params via `IN (${Prisma.join(lotIds)})` → Postgres `42883 operator does not exist: uuid = text`, so EVERY transfer save with a lot 500'd (lots are mandatory). Fixed to `= ANY(ARRAY[${Prisma.join(lotIds)}]::uuid[])`.
2. `loadTransitRows` had `ORDER BY i.item_name` but item_master's column is `item_name_en` (the SELECT aliases `item_name_en AS item_name`; a qualified `i.item_name` forces table resolution) → `42703 column i.item_name does not exist`, so loading ANY single transfer (`GET /stock/transfer?svhId=`) 500'd. Fixed to `ORDER BY i.item_name_en`.

**Despatch + receive-post cannot run on this DB (deployment gap):** `TRANSFER_OUT_RULES.postFunction='stock.fn_svh_post_transfer'` and receipt calls `stock.fn_svh_receive_transfer`; neither exists (schema `stock` has only 4 trigger guards, no `fn_svh_*`). Unlike OPENING/PHYSICAL, transfer has NO TypeScript in-process posting path (`usesInProcessPosting` is only true when postFunction==='stock.fn_svh_post'), so the whole despatch/receive half is dead until the SQL stock engine is deployed.

**Not fixed (design call, flagged):** `TRANSFER_OUT_RULES` omits `allowsRepeatHolding`, so the shared preflight's opening-uniqueness check (`!allowsRepeatHolding` gate, in `stock-voucher.service.ts` ~L1767 `opened` CTE) wrongly fires on transfers — a transfer of any holding that had an OPENING this year is refused with "this holding already has an opening in this year" (422), even a cancelled opening (check filters `sml_is_deleted=false`, not status). Likely wants `allowsRepeatHolding: true` on transfers, or the check gated to voucherType='OPENING'. Doesn't unblock despatch anyway (engine still absent).

Fixes need `npm run build` + restart of the manual `node dist/src/main` :3011 server to reach the live server. Related: [[erp-server-http-testing-without-credentials]], [[erp-transfer-engine-not-deployed]].
