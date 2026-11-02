---
name: erp-stock-voucher-refno-series
description: "ERP stock voucher refnos (OPENING, PHYSICAL) come from accounts.acc_voucher_types via refnoVchrTypeId; PHYSICAL=id 6 (PHY0001), OPENING=id 1 (OPN0001)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 272b4d7c-aa7f-4ad6-a475-2899a9c30282
  modified: 2026-09-09T11:17:13.912Z
---

ERP stock voucher printed refnos have two sources (see `stock-voucher-numbering.helper.ts`): a type with `refnoVchrTypeId` in its rules draws `PREFIX + zero-padded(n, width)` from `accounts.acc_voucher_seq` (e.g. `OPN0001`, `PHY0001`); a type WITHOUT one falls back to `{typeCode}/{accYear}/{deviceCode}/{slno}`, and when the device has no `dev_device_uid`/`dev_device_name` the middle segment is the raw device uuid (the ugly `PHY/2026-2027/<uuid>/1`).

Mapping of `refnoVchrTypeId` (pinned in each controller's RULES) to `accounts.acc_voucher_types.vchr_type_id`: OPENING = 1 (`OPN`), PHYSICAL = 6 (`PHY`, prefix PHY width 4). Physical was added 2026-09-09: code const `PHYSICAL_VCHR_TYPE_ID = 6` in `physical-stock-voucher.controller.ts` + migration `20260909110000_physical_stock_refno_prefix_phy` (INSERT id 6, ON CONFLICT DO NOTHING). Ids are pinned (not identity-assigned) so the rules can name them.

**Why:** adding a new stock voucher type that should print PRETTY refnos means BOTH a code change (`refnoVchrTypeId`) AND an `acc_voucher_types` row, or it silently gets the device-uuid fallback.

**How to apply:** the local live server is `node dist/src/main` (HTTPS :3011, run manually, NOT the PM2 `api` app and NOT watch mode). A code change needs `npm run build` then a manual restart of that process to take effect; the DB migration applies via `npx prisma migrate deploy`. Verified end-to-end by `test/physical-stock-http.e2e-spec.ts` (asserts `/^PHY\d{4}$/`). Related: [[erp-server-http-testing-without-credentials]].
