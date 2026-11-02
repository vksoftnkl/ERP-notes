---
name: price-change-log-notes-144
description: "Notes 144 price change log BUILT 2026-10-23 — PriceMasterWriter is the ONLY writer of item_price_master / item_qty_price (guard spec fails the build otherwise); /item-price-log/* on menu 290; plan file never on this box"
metadata:
  type: project
---

`src/modules/Inventory/item-price-log/` (uncommitted at hand-off): `PriceMasterWriter` holds every
write statement on both price tables — item card (ItemsPriceMasterService), bucket re-key
(PriceBucketService.rekeyItem → REKEY), Change Selling Price (the 6 statements MOVED out of
PriceBucketGateway, SQL byte-identical; SellingPriceBulkService calls the writer), qty slabs.
Each: snapshot `to_jsonb` before → statement → after → `PriceChangeLogService.record()`.
`price-master-writer.guard.spec.ts` scans src/ for Prisma writes, nested relation writes (relation
names from Prisma.dmmf) and raw SQL on the two tables outside the writer.

Rules: action read off before/after (deleted flag set = DELETE kept as before only, cleared =
RESTORE); no-op = only stamps/sync/ipm_key_* changed → no row; one change set per transaction
(WeakMap on the tx object); who = token user/device + request IP; app DESKTOP/WEB/API/SYSTEM from
dev_device_type (Mobile → API). Item-card price edits also log the re-derived *_wot / markup.

Reads `/item-price-log/list|get|days|item-history|gaps` need VIEW on menu 290 "Price Change Log"
(Reports, 1.50; migration 20261023130000 + Menu_Master.sql); codes IPCL_RIGHT_VIEW / NOT_FOUND /
BAD_RANGE with catalogue rows (Error_Messages.json 2026-10-23.2). `gaps` = plan §2.6: rows touched
since the log's first row whose latest log row is missing or ≠ the row's business columns.

**Why:** user approved a separate append-only log incl. qty slabs (notes 144).
**How to apply:** a new price write must be a writer method. Unit specs build a REAL writer over
their mocked tx with a stub change log; hand-built e2e suites use `test/helpers/price-writer.ts`.
Live proof: `test/item-price-log-notes-144.e2e-spec.ts` (rolled back). A server built before this
(PM2 :3000 dist) writes prices unlogged — those show up in `gaps`.

**Quick rate popup plan (2026-10-23) server side, built same day:** `POST /stock/price-bulk` and
`/stock/price-schedule/cancel` now need **Edit on menu 30** (403 STOCK_PRICE_RIGHT_EDIT; before,
NO right was checked — any login could change prices; only 4 users hold menu 30 on the dev box).
Rolled-back e2e suites call `grantPriceEdit(tx, userId)` (test/helpers/price-writer.ts).
`originMenuId` on the save → log reason "via <menu name> (menu N)". New `GET
/stock/item-inward-history` (src/modules/stocks/item-inward-history): unreversed inward
OPENING/PURCHASE ledger rows. The popup's grid route is `/stock/price-buckets/:itemId`, not
`/stock/price-bulk?itemId=` as the plan said; Sale Bill is menu 12, not 31.

**Notes 145 (2026-10-23, built):** every list / item-history / get row has `fields[]`
{column,label,before,after} for the changed columns, built in SQL from `unnest(ipcl_changed)`
(`changedValues`) — never the whole before/after rows. `orderColumns()` (price-field-labels.ts):
label-map order, each level's price then its wot + markup, row id (ipm_id/iqp_id) dropped; applied
on write (changedColumns) AND on read (rows logged earlier hold jsonb key order = shortest name
first). A DELETE's deleted flag reads after = true. Filters `deviceId`, `field` (= ANY ipcl_changed).
