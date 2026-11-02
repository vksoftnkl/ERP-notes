# NestJS — textile tags backend (build plan)

Written 2026-10-17. **A new file; it edits no existing plan.** Target: the NestJS server (read here
through `nexerp-api-ro`, paths below are relative to it). It turns three design files into one build
plan for the server:

| design file (same folder) | status |
|---|---|
| `plan-textile-lot-tag.md` ("base") | T1–T7 accepted 2026-09-27 |
| `plan-textile-lot-tag-gaps.md` ("gaps 1") | G1–G19 accepted 2026-10-17 |
| `plan-textile-lot-tag-gaps-2.md` ("gaps 2") | G20–G23 accepted 2026-10-17 |
| `textile_tag_ui_mockup_rev3.png` | the screen this server feeds |

Where this file says something different from the design files, **this file wins** and the row
below says why. Every change comes from a fact found in the code on 2026-10-17 (§0.2).

| design said | this plan does | because |
|---|---|---|
| base §4: stored status `RESERVED → ACTIVE → SOLD_OUT`, flipped by the tag service | store only `OPEN` / `VOID`; **RESERVED / ACTIVE / SOLD_OUT are derived** at read time (§5.6) | the lot id is deterministic (§0.2 row 4), so "does the lot exist / what is on hand" is a PK read; a stored copy can only go stale |
| base §5.2, §10: lazy lot link + a nightly BullMQ job | **no link step, no job.** `ilt_lot_id` is computed at reserve (§5.2) | same; and the server has no cron/repeatable job today (row 9) |
| base §1: `ck_ilt_lot_when_active`, FK `ilt_lot_id → stock_lot` | `ilt_lot_id` is a plain uuid, **no FK** (the lot does not exist yet at reserve) | the FK would refuse every reserve |
| base §11: tags sync **down** to every till | follows the platform: today sync is **push-only** (till → cloud). Down-sync is a platform gap shared with every master (B7) | row 8 |
| gaps 1 G1: slab from `statutory_limits` | needs a **new statutory code + a slab rule table**; neither exists (§7) | row 7 |
| base §6 routes | + hold, unhold, suggest-price, series save, markup save, list quick-add (§4) | gaps 1/2 |

---

## 0. Ground rules

### 0.1 House rules that bite here

- **No SQL function or trigger** written by us. Tables, CHECKs, unique / partial / expression
  indexes and FKs are declarative and allowed. (The platform's own sync triggers are installed
  automatically; §10.)
- **One stock engine.** The tag module never writes `stock_ledger`, `stock_balance` or `stock_lot`.
- **Own URL family** `/api/v1/textile-tags/...`; no old route or DTO widened, `item-by-barcode`
  included.
- **No `/list` route.** Lists are grids (`fixed.grid_details` + `grid_columns`) and dropdowns.
- **Cost stays on the lot.** `ilt_label_cost` is print-only (G2).
- **Price has one home: the tag.** Never MRP / selling-price tracking on these items.

### 0.2 Facts this plan is built on (code read 2026-10-17)

| # | fact | evidence |
|---|---|---|
| 1 | Module shape: controller + service + module + `dto/` (+ `types/`); `@Controller('<path>')`, `@Version(API_VERSION)` per route; envelope `{success, message, data}` | `src/modules/stocks/stock-adjustment/stock-adjustment.controller.ts:46-85`; `main.ts:189-197` |
| 2 | Prisma: one model per file under `prisma/<folder>/`, camelCase + 3-letter prefix + `@map`; ids `@default(dbgenerated("uuidv7()"))`; DB-only objects listed in the model doc-comment and hand-written in the migration | `prisma/stocks/stockTrackPreset.prisma:46-63`; `stockLot.prisma:53-95` |
| 3 | Number lock: `pg_advisory_xact_lock(hashtext(ns), hashtext(scope))` wrapped in `count()` | `src/modules/stocks/stock-voucher/stock-voucher-numbering.helper.ts:100-128` |
| 4 | **Lot id is deterministic**: UUIDv5 of the identity (`lotIdentityUuid()`); identity = company, item, `key_batch = coalesce(nullif(upper(btrim(batch)),''),'~')`, and sentinels for untracked dims (MRP/SP −1, supplier nil uuid) | `stock-voucher-posting.helper.ts:415-424, 1303, 1383` |
| 5 | Batch-only policy stores NULL MRP / SP / supplier on the lot | `stock-voucher-posting.helper.ts:785-790` |
| 6 | `MANUAL` issue strategy already refuses a lotless outward line: `STOCK_PICK_MANUAL_LOT_REQUIRED` (422) | `stock-voucher-posting.helper.ts:1130-1136, 1281-1285`; `types/stock-error-codes.ts:100` |
| 7 | `statutory_limits` + `StatutoryService.limit(code, companyId, onDate, …)` exist; **no garment / value-slab code** is seeded | `prisma/public/statutoryLimits.prisma:1-38`; `src/common/posting/statutory.service.ts:~57, 329` |
| 8 | Cloud sync auto-discovers any table with a PK and a `*_sync_date` column (`public.v_sync_tables`); **push-only** | `src/modules/cloud-sync/cloud-sync.service.ts:268-273`; migration `20260923120000_offline_cloud_sync:78-120` |
| 9 | BullMQ installed, two queues, nobody enqueues; **no `@Cron`** anywhere | `src/common/queue/queue.constants.ts:1-4` |
| 10 | Settings: `AppSettingValueService.resolveEffective({companyId, branchId, deviceId, userId})` then `.find(i => i.asdKey === KEY)`; defs inserted by migration | `src/modules/settings/appSettings/app-setting-value.service.ts:232`; `stock-adjustment.service.ts:1170-1174` |
| 11 | Print: purpose row in `public.print_purpose`; dataset in `public.print_template_dataset` (`ptd_source_kind` SQL or PROVIDER; SQL must use `:company_id`, runs READ ONLY) | `prisma/public/printPurpose.prisma`; `printTemplateDataset.prisma:19-63`; migration `20261015130000_party_outstanding_print_purpose:19-31` |
| 12 | Errors: per-module string enum + `throwBadRequest/Conflict/Unprocessable/NotFound(..., code)`; wording rows in `prisma/seed/Error_Messages.json` | `src/common/utils/module-shared.utils.ts:34-87`; `src/modules/error-messages/error-messages.service.ts:13-20` |
| 13 | Rights: no route guard; services call `assertMenuRight(client, userId, menuId, right)` (403 `<PREFIX>_RIGHT_<RIGHT>`). Stock/Inventory modules check none today | `src/common/posting/rights.ts:67, 121`; `src/modules/gst/config/gst-config.support.ts:101-108` |
| 14 | Audit: `AuditLogService.logEntityChange({action, tableName, screenName, screenType:'master', pk, …}, tx)` | `src/modules/audit-log/audit-log.service.ts:500`; `items-section-master.service.ts:164-177` |
| 15 | Effective policy: `effectivePolicyLateral()` / `effectivePolicyCte()`; group preset copied to `stock_track_policy` by `StockTrackPolicyService.syncFromItemGroup()` | `stock-voucher-posting.helper.ts:323, 363`; `stock-track-policy.service.ts:209` |
| 16 | Today's scan: `GET /api/v1/master-lookups/item-by-barcode` → `item_ean_codes`, no lot | `src/modules/master-lookup/master-lookup.controller.ts:184-194` |
| 17 | Purchase = suppliers + supplier groups only | `src/modules/purchase/` |
| 18 | Item has `itemGroupId` (required), `itemSectionId`; group has `itgTrackPresetId` | `prisma/inventory/itemMaster.prisma:14,17,64`; `itemGroupMaster.prisma:31` |

---

## 1. Migration `2026MMDDhhmmss_textile_tags` (one, hand-written SQL)

### 1.1 `inventory.item_lot_tag` (`ilt_`)

| column | type | rule |
|---|---|---|
| `ilt_id` | uuid PK default `uuidv7()` | |
| `ilt_company_id` | uuid NOT NULL | FK `companys` |
| `ilt_tenant_id` | uuid NULL | as every table |
| `ilt_barcode` | varchar(30) NOT NULL | `<series code><6 digits>`; = the lot's batch no |
| `ilt_series_id` | uuid NOT NULL | FK `item_lot_tag_series` |
| `ilt_seq_no` | integer NOT NULL | |
| `ilt_item_id` | uuid NOT NULL | FK `item_master`; immutable once the lot exists |
| `ilt_lot_id` | uuid NOT NULL | **computed at reserve** = `lotIdentityUuid(company, item, barcode, batch-only)`; no FK (§5.2) |
| `ilt_brand_id` | uuid NULL | FK `item_brand_master` |
| `ilt_size_id` | uuid NULL | FK `item_size_master` (G5) |
| `ilt_colour_id` | uuid NULL | FK `item_colour_master` (G5) |
| `ilt_fabric_id` | uuid NULL | FK `item_fabric_master` (G20) |
| `ilt_season_id` | uuid NULL | FK `item_season_master` (G21) |
| `ilt_model` | varchar(100) NULL | free text |
| `ilt_supplier_article` | varchar(60) NULL | free text (G22) |
| `ilt_attr_note` | varchar(250) NULL | free text, printed |
| `ilt_mrp` | numeric(18,6) NULL | |
| `ilt_sale_price` | numeric(18,6) NOT NULL | **the** selling price |
| `ilt_label_cost` | numeric(18,6) NULL | print-only cost for the cost code (G2) |
| `ilt_supplier_id` | uuid NULL | FK `purchase.suppliers` |
| `ilt_src_module`, `ilt_src_doc_type`, `ilt_src_doc_id`, `ilt_src_acc_year`, `ilt_src_line_no` | | source line; NULL for a standalone reserve |
| `ilt_label_qty` | numeric(18,6) NOT NULL default 1 | pieces / metres the label covers |
| `ilt_print_count` | integer NOT NULL default 0 | |
| `ilt_last_printed_on` | timestamptz NULL | |
| `ilt_hold_party_id` | uuid NULL | the customer's ledger id (house rule: one partyId = led_id) (G9) |
| `ilt_hold_until` | date NULL | |
| `ilt_hold_note` | varchar(100) NULL | |
| `ilt_status` | varchar(10) NOT NULL default `'OPEN'` | `OPEN` / `VOID` only (§5.6) |
| `ilt_void_reason` | varchar(150) NULL | |
| audit + sync | | `ilt_is_active`, `ilt_is_deleted`, `ilt_created_on/by`, `ilt_modified_on/by`, **`ilt_sync_date`** (sync discovery, fact 8), `ilt_row_version` |

**No `ilt_tax_id`** (G1).

Constraints and indexes:

```sql
CREATE UNIQUE INDEX ux_ilt_barcode ON inventory.item_lot_tag (ilt_company_id, upper(ilt_barcode))
  WHERE ilt_is_deleted = false;
CREATE UNIQUE INDEX ux_ilt_series_seq ON inventory.item_lot_tag (ilt_series_id, ilt_seq_no);
CREATE UNIQUE INDEX ux_ilt_lot ON inventory.item_lot_tag (ilt_lot_id) WHERE ilt_is_deleted = false;
ALTER TABLE inventory.item_lot_tag
  ADD CONSTRAINT ck_ilt_barcode  CHECK (ilt_barcode ~ '^[A-Z0-9]{3,30}$'),
  ADD CONSTRAINT ck_ilt_prices   CHECK (ilt_sale_price >= 0 AND (ilt_mrp IS NULL OR (ilt_mrp >= 0 AND ilt_sale_price <= ilt_mrp))),
  ADD CONSTRAINT ck_ilt_cost     CHECK (ilt_label_cost IS NULL OR ilt_label_cost >= 0),
  ADD CONSTRAINT ck_ilt_qty      CHECK (ilt_label_qty > 0),
  ADD CONSTRAINT ck_ilt_status   CHECK (ilt_status IN ('OPEN','VOID')),
  ADD CONSTRAINT ck_ilt_void     CHECK (ilt_status = 'OPEN' OR ilt_void_reason IS NOT NULL),
  ADD CONSTRAINT ck_ilt_hold     CHECK (ilt_hold_party_id IS NOT NULL OR (ilt_hold_until IS NULL AND ilt_hold_note IS NULL));
CREATE INDEX ix_ilt_item_status ON inventory.item_lot_tag (ilt_company_id, ilt_item_id, ilt_status);
CREATE INDEX ix_ilt_brand   ON inventory.item_lot_tag (ilt_company_id, ilt_brand_id);
CREATE INDEX ix_ilt_season  ON inventory.item_lot_tag (ilt_company_id, ilt_season_id);
CREATE INDEX ix_ilt_src     ON inventory.item_lot_tag (ilt_src_doc_id, ilt_src_line_no);
CREATE INDEX ix_ilt_article ON inventory.item_lot_tag (ilt_company_id, ilt_supplier_id, upper(ilt_supplier_article));
CREATE INDEX ix_ilt_hold    ON inventory.item_lot_tag (ilt_company_id, ilt_hold_party_id) WHERE ilt_hold_party_id IS NOT NULL;
```

`ux_ilt_lot` makes "one tag = one lot" a database fact: two tags can never resolve to the same lot.

### 1.2 `inventory.item_lot_tag_series` (`ilts_`)

`ilts_id` PK, `ilts_company_id`, `ilts_device_id` (FK `fixed.device_master`), `ilts_code`
varchar(4), `ilts_last_seq` integer NOT NULL default 0 (display only — allocation reads MAX, §5.1),
`ilts_is_active`, audit + `ilts_sync_date`.
`UNIQUE (ilts_company_id, upper(ilts_code))`, `UNIQUE (ilts_company_id, ilts_device_id)`,
`CHECK (ilts_code ~ '^[A-Z0-9]{2}$')` (T1: 2 characters).

### 1.3 The four lists (G5, G20, G21)

Same shape for each; company NULL = shared by every company (house rule: NULL company = shared).

| table | prefix | extra columns |
|---|---|---|
| `inventory.item_size_master` | `isz_` | `isz_size_set` varchar(20) NOT NULL (ALPHA / WAIST / KIDS / LENGTH / FREE), `isz_sort` int |
| `inventory.item_colour_master` | `icl_` | `icl_sort` |
| `inventory.item_fabric_master` | `ifb_` | `ifb_sort` |
| `inventory.item_season_master` | `isn_` | `isn_from_date`, `isn_to_date` (NULL allowed), `isn_is_closed` bool default false, `isn_sort`; `CHECK (isn_from_date IS NULL OR isn_to_date IS NULL OR isn_from_date <= isn_to_date)` |

Common: `<p>_id`, `<p>_company_id NULL`, `<p>_code` varchar(20), `<p>_name` varchar(60) NOT NULL,
`<p>_is_active`, `<p>_is_deleted`, audit, `<p>_sync_date`. Unique per scope on the name:
`UNIQUE (coalesce(<p>_company_id,'00000000-0000-0000-0000-000000000000'), upper(<p>_name)) WHERE NOT <p>_is_deleted`
(sizes: add `isz_size_set` to the key — "L" exists in ALPHA, not twice).

Seed (shared rows): sizes ALPHA XS..4XL, WAIST 26..44 step 2, KIDS 0-1Y..13-14Y, FREE; a dozen
colours; fabrics Cotton, Linen, Pure silk, Art silk, Poly-cotton, Polyester, Rayon, Georgette,
Chiffon, Wool. Seasons: **none** (company-specific).

### 1.4 `inventory.item_price_markup` (`ipmk_`) (G3)

`ipmk_id`, `ipmk_company_id` NOT NULL, `ipmk_item_group_id` NULL, `ipmk_brand_id` NULL,
`ipmk_sets` varchar(4) (`SALE` / `MRP`), `ipmk_markup_pct` numeric(9,4) `> -100`,
`ipmk_round_rule` varchar(12) (`NONE`, `END_9`, `END_99`, `NEAREST_10`, `NEAREST_50`),
`ipmk_is_active`, audit, sync.
`UNIQUE (ipmk_company_id, coalesce(ipmk_item_group_id, nil), coalesce(ipmk_brand_id, nil), ipmk_sets) WHERE NOT ipmk_is_deleted`.

### 1.5 Seeds in the same migration

| what | row |
|---|---|
| track preset | `stock.stock_track_preset` `TEXTILE_TAG`: batch only (signature `B`), issue `MANUAL`, no negative |
| settings (`public.app_setting_def`, fact 10) | `stock.textile_tags_enabled` bool default false · `stock.textile_cost_code` text default '' (10 distinct A–Z or blank) · `stock.textile_remnant_qty` numeric default 1.0 · `stock.textile_orphan_days` int default 30 |
| item group flag | `inventory.item_group_master.itg_textile_label_per_piece boolean NOT NULL DEFAULT false` (G16) — an expand-only ALTER |
| statutory code | `GST_VALUE_SLAB_APPAREL` (§7) |
| print purpose | `TEXTILE_TAG_LABEL` (§8) |
| error wording | rows in `prisma/seed/Error_Messages.json` for every §9 code |
| grids / dropdowns / menu | ids assigned by the box at build; SQL in §11 |

### 1.6 Prisma

One model file each under `prisma/inventory/`: `itemLotTag.prisma`, `itemLotTagSeries.prisma`,
`itemSizeMaster.prisma`, `itemColourMaster.prisma`, `itemFabricMaster.prisma`,
`itemSeasonMaster.prisma`, `itemPriceMarkup.prisma`; the slab table in `prisma/public/`. The
doc-comment of each lists its hand-written CHECKs and expression indexes (fact 2). Add the
`itgTextileLabelPerPiece` field to `itemGroupMaster.prisma`.

---

## 2. Module layout

`src/modules/stocks/textile-tags/` (it lives with stock because it hangs off the lot; tables are in
`inventory`):

```
textile-tags.module.ts            imports StockVoucherModule (lot helpers), AppSettingsModule, AuditLogModule
textile-tags.controller.ts        @Controller('textile-tags')        — tags
textile-tag-lists.controller.ts   @Controller('textile-tags/lists')  — size/colour/fabric/season save
textile-tag-setup.controller.ts   @Controller('textile-tags/setup')  — series + markup
textile-tags.service.ts           reserve, get, update, price, hold, void, print
textile-tag-scan.service.ts       scan (pure read; the till runs the same SQL)
textile-tag-markup.service.ts     suggest + rule save
textile-tag-series.service.ts     series save + allocation
textile-tags.rules.ts             setting keys, constants, round rules, cost-code cipher
types/textile-tag-error-codes.ts  enum TextileTagErrorCode
dto/                              one file per route body / query
README.md
```

Register in `app.module.ts` beside `StockAdjustmentModule` (fact 1). Route bodies carry
`companyId`, `branchId` (`@RequiredUuid()`), `deviceId` where a series is needed; `userId`
falls back to the token (fact 1).

---

## 3. Rights

Menu ids come from the box (tag screen menu; the four list masters may share it). Every write
route calls `assertMenuRight(client, userId, TEXTILE_TAG_MENU, right)` (fact 13):

| route | right |
|---|---|
| reserve | Create |
| update, hold, unhold, lists save | Edit |
| price | Edit (+ the house "below cost" confirm, §5.4) |
| void | Delete |
| print | Print |
| setup (series, markup) | Edit on the setup menu (admin) |
| scan, get, suggest-price | none beyond the token (the counter must never be blocked) |

---

## 4. Routes — `/api/v1/textile-tags`

| method + path | body / query → result |
|---|---|
| `POST /reserve` | `{companyId, branchId, deviceId, seasonId?, supplierId?, src?: {module, docType, docId, accYear}, lines: [{lineNo?, itemId, brandId?, sizeId?, colourId?, fabricId?, seasonId?, model?, supplierArticle?, attrNote?, mrp?, salePrice, labelCost?, supplierId?, qty}]}` → `{tags: [{lineNo, tagId, barcode, lotId, labelQty}]}` in line order. Header `seasonId` / `supplierId` fill lines that omit them (G21). A per-piece group (G16) returns `qty` tags of `labelQty` 1 for that line; otherwise one tag with `labelQty = qty` |
| `GET /scan?companyId&branchId&godownId?&barcode&partyId?` | the counter lookup (§5.5). `404 TEXTILE_TAG_NOT_A_TAG` → the client falls through to EAN |
| `GET /get?companyId&tagId` | one tag + derived status + on hand per branch |
| `POST /update` | details only: brand, size, colour, fabric, season, model, supplierArticle, attrNote, supplierId, labelCost. `itemId` change allowed only while derived status is RESERVED |
| `POST /price` | `{companyId, tagIds[], salePrice? \| percentOff?, mrp?, confirmBelowCost?: bool}` → per-tag `{tagId, old, new, belowCost}`. Refuses VOID; `ck_ilt_prices` applies |
| `POST /hold` | `{companyId, tagId, partyId, until, note?}` |
| `POST /unhold` | `{companyId, tagId}` |
| `POST /void` | `{companyId, tagId, reason}` (§5.7) |
| `POST /print` | `{companyId, tagIds[], copies?}` → `{printPurpose: 'TEXTILE_TAG_LABEL', params}` for the existing print-render path (§8); bumps `ilt_print_count`, `ilt_last_printed_on` |
| `GET /suggest-price?companyId&itemId&brandId?&cost` | `{rule: {id, sets, pct, round} \| null, mrp?, salePrice?}` (§5.3) |
| `POST /lists/{size\|colour\|fabric\|season}/save` | create or update one list row (quick-add uses create). Returns the row; a duplicate name returns the existing row with `409 TEXTILE_TAG_LIST_DUPLICATE` |
| `POST /setup/series/save` | `{companyId, rows: [{deviceId, code}]}`; a code already used by a tag is immutable |
| `POST /setup/markup/save` | `{companyId, rows: [...]}` full replace of the company's rules, in one transaction |

Lists for screens (register, series, markup, list masters) are **grids**; combos are
**dropdowns** (§11). No `/list` routes.

---

## 5. Service logic

### 5.1 Reserve

One transaction:

1. `textile_tags_enabled` must be on for the company (fact 10) → else `TEXTILE_TAG_DISABLED`.
2. Series: row for `(company, deviceId)` → else `TEXTILE_TAG_NO_SERIES` (T7).
3. Per line validate (§9 codes): item active and in scope; **effective policy of the item is
   signature `B` with issue `MANUAL`** (`effectivePolicyCte`, fact 15) → else
   `TEXTILE_TAG_ITEM_NOT_TRACKED`; brand/size/colour/fabric/season/supplier active and in scope
   (company or shared); season not closed; `salePrice ≥ 0`, `≤ mrp` when set; `qty > 0`; metre
   items (`unit_decimal_count > 0`) are never split per piece.
   Collect all line errors, then refuse the whole batch (the screen shows them per line).
4. Lock: `SELECT count(pg_advisory_xact_lock(hashtext('inventory.item_lot_tag.seq'), hashtext(series_id::text)))`
   (the fact-3 pattern; not in a CTE).
5. `next = coalesce(max(ilt_seq_no),0)+1` for the series; barcode = `code || lpad(seq, 6, '0')`.
   When `seq > 999999` → `TEXTILE_TAG_SERIES_FULL` (admin gives the device a second code).
6. `ilt_lot_id` from the engine's own `lotIdentityUuid(alias)` (posting.helper.ts:1383, checked
   2026-10-17). It is a SQL fragment over an alias carrying the eight identity columns, so select
   it from a VALUES row with the batch-only sentinels of `lotIdentityKeyColumns()` (:410-419):

   ```sql
   SELECT ${lotIdentityUuid('k')} AS lot_id
     FROM (VALUES (:company::uuid, :item::uuid, upper(btrim(:barcode)),
                   (-1)::numeric, (-1)::numeric, DATE '0001-01-01', '~',
                   '00000000-0000-0000-0000-000000000000'::uuid))
          AS k(svh_company_id, svi_item_id, key_batch, key_mrp, key_sp, key_expiry, key_serial, key_supplier)
   ```

   **Never re-implement the hash.** If the engine's identity ever changes, this follows it; test 4
   (§12) proves the two agree.
7. Insert; update `ilts_last_seq`; one `logEntityChange` per tag (fact 14, screen "Textile Tags").

### 5.2 Why the lot id at reserve is safe

- The engine creates the lot on the inward post with `INSERT … ON CONFLICT DO NOTHING` and the
  same deterministic id (fact 4). The tag's id **is** the lot's id before and after.
- It holds only while the item's policy is batch-only. Two guards: reserve refuses any other
  policy (5.1 step 3), and **`StockTrackPolicyService` must refuse to change the policy of an
  item / group that has OPEN tags** → `TEXTILE_TAG_POLICY_LOCKED` (one check added to
  `syncFromItem` / `syncFromItemGroup`, fact 15). This is the server half of the item-group
  change warning the client already has.
- Re-lot (adjustments) moves stock to a **new identity**: the re-lot reserves a new tag first
  (new barcode = new lot id), posts, then voids the old tag in its own transaction (§6.3).

### 5.3 Suggest price (G3)

Pick the most specific active rule for `(company, item.group, brand, sets)`: group+brand > brand >
group > any; on a tie, the latest modified. `base = cost × (1 + pct/100)`; round **up**:
`END_9` → next number ending in 9; `END_99` → next ending in 99; `NEAREST_10/50` → ceil to that
step; `NONE` → 2 decimals. A rule `sets = MRP` fills `mrp` only. No rule → `{rule: null}`.
The same function is reused by purchase in T-5.

### 5.4 Price (markdown)

`percentOff` → `round(old × (1 − p/100), 2)`; then `ck_ilt_prices`. Tags whose new price `<`
the lot's `slt_cost_rate` (or `ilt_label_cost` when the lot does not exist yet) are listed; if any
and `confirmBelowCost` is not true → `409 TEXTILE_TAG_BELOW_COST` with the list (the screen asks once,
then resends with the flag, T6). Audit each change. Posted bills are untouched (their rate is on
the line).

### 5.5 Scan (pure read)

```
tag   := ilt where company and upper(barcode) = upper(:barcode) and not deleted     → else 404 NOT_A_TAG
VOID  → 422 TEXTILE_TAG_VOID
lot   := stock_lot where slt_id = ilt_lot_id                                         (PK read)
bal   := stock_balance for (branch, godown?, lot)
```

Returns: tag id, barcode, item id + name + unit + `unit_decimal_count` (G4: the client asks the
length when > 0), lot id, batch no (= barcode), MRP, sale price, brand / size / colour / fabric /
season names, model, attr note, **item tax id + HSN** (the bill resolves the slab, §7), on hand
(branch, and godown when given), derived status, and `hold: {partyId, partyName, until, note}`
when a hold is live and `partyId` differs (G9 warning; never an error). The till runs the same
SQL on its local DB.

### 5.6 Derived status

| derived | when |
|---|---|
| `VOID` | `ilt_status = 'VOID'` |
| `RESERVED` | no `stock_lot` row with `slt_id = ilt_lot_id` |
| `ACTIVE` | lot exists, on hand (company or filtered branch) > 0 |
| `SOLD_OUT` | lot exists, on hand = 0 |
| + `REMNANT` | ACTIVE, metre item, 0 < on hand ≤ `stock.textile_remnant_qty` |
| + `HELD` | `ilt_hold_party_id` set and (`ilt_hold_until` NULL or ≥ today) |
| + orphan | RESERVED and created more than `stock.textile_orphan_days` ago |

Computed in the grid SQL and in `get` / `scan`. Nothing to keep in step, so no job.

### 5.7 Void

Refuse (`TEXTILE_TAG_IN_USE`) when **any** `stock.stock_ledger` row cites `ilt_lot_id`. Else set
`VOID` + reason; the number is never reused.

---

## 6. Hooks into other modules (small, each in that module's own transaction)

### 6.1 Opening stock and adjustment inward (T-2: the go-live path)

**No change needed.** The line carries `batch = barcode`; the engine creates the lot with the id
the tag already holds (§5.2). One **validation** is added to the opening / adjustment validate
(base §12.7): a batch that matches an OPEN tag's barcode must have the tag's item →
`TEXTILE_TAG_ITEM_MISMATCH`. Read-only lookup on `ux_ilt_barcode`.

### 6.2 Sale bill, DC, return, order, quotation

- Server: **no DTO change** — a tagged line sends `lotId` + `batchNo` like a picked lot today; the
  engine's `MANUAL` refusal (fact 6) already stops a lotless textile line.
- **Tax slab (G1)** is the only sales-side server change: §7.
- Promotions with a brand condition match the **tag's** brand when the line's lot has a tag (T4):
  where the server re-evaluates promotions, look up `ilt_brand_id` by `ilt_lot_id`.

### 6.3 Re-lot (adjustments)

The re-lot's post takes `newTagId`; in its transaction it checks the new tag is RESERVED for the
same item, and after posting sets the old tag `VOID` (reason "re-lot → <new barcode>").

### 6.4 Purchase (T-5, when the module exists) — contract only

Purchase lines call `reserve` with `src` = the purchase draft and the line's rate as `labelCost`;
the barcode goes into the line's batch. Post needs nothing from tags (lot id already known).
Purchase cancel reverses stock; the tag shows RESERVED again by derivation. Purchase return and
consignment (G6, G7) read tags by `ilt_src_doc_id` / supplier / article.

### 6.5 Physical count (G17)

The count screen calls `scan`; the count line carries `batch = barcode`. No server change.

---

## 7. GST value slab (G1) — needed by any garment shop, tags or not

Neither the threshold nor the rule exists (fact 7). Add:

- **Statutory code** `GST_VALUE_SLAB_APPAREL` in `public.statutory_limits` (seed pattern of
  migration `20261008110000`): value 2500.00, effective from 2025-09-22, per company override as
  the table already allows.
- **`public.tax_value_slab`** (`tvs_`): `tvs_company_id NULL`, `tvs_hsn_prefix varchar(8)`
  (e.g. `61`, `62`, `6301`…), `tvs_limit_code` (→ statutory code), `tvs_tax_upto_id` (≤ limit),
  `tvs_tax_above_id` (> limit), `tvs_from_date`, audit, sync.
  `UNIQUE (coalesce(company,nil), tvs_hsn_prefix, tvs_from_date)`. Longest prefix wins.
- **Resolver** `TaxSlabService.resolve(companyId, itemId, netUnitValue, onDate)`: item HSN → longest
  slab prefix → `limit(code, companyId, onDate)` → tax id. No slab row → the item's own tax
  (sarees and fabric stay flat).
- **Sale bill save / post validation** recomputes each line's tax from the **net unit value after
  line, bill and promotion discounts** and refuses a mismatch (`SALES_TAX_SLAB_MISMATCH`) so a
  stale client cannot post the wrong slab. Same in DC (when priced), return (uses the original
  line's tax), quotation.
- Exposed for the client as `GET /api/v1/tax-slab/resolve?companyId&itemId&netUnitValue&date`
  (its own URL family; the bill calls it when a discount changes).

The rule is net value **per piece**; for metre goods the slab table simply has no row.

---

## 8. Label printing

- `public.print_purpose` row `TEXTILE_TAG_LABEL` (fact 11, insert-if-missing).
- A shipped template with one **SQL dataset** `TEXTILE_TAG` (READ ONLY, uses `:company_id` and
  `:tag_ids` (uuid[]) and `:copies`): one row per label copy —
  `generate_series(1, coalesce(:copies, CASE WHEN per-piece OR metre THEN 1 ELSE ceil(ilt_label_qty) END))`.
- Fields: barcode (Code 128), item name, brand, size, colour, fabric, model, attr note, MRP,
  sale price, packed month (`to_char(ilt_created_on,'MM/YYYY')`), season, `cost_code`.
- `cost_code` = `translate(round(ilt_label_cost)::text, '1234567890', :cipher)` where `:cipher` is
  the company setting passed as a parameter by the print route; blank setting → NULL (G2).
  `translate` is a Postgres built-in, so no function of ours.
- Supplier article is a dataset field but **not** on the shipped design (G22).

---

## 9. Error codes — `TextileTagErrorCode` (+ wording rows)

| code | HTTP | when |
|---|---|---|
| `TEXTILE_TAG_DISABLED` | 422 | setting off |
| `TEXTILE_TAG_NO_SERIES` | 422 | device has no series (T7) |
| `TEXTILE_TAG_SERIES_FULL` | 422 | seq > 999999 |
| `TEXTILE_TAG_ITEM_NOT_TRACKED` | 422 | policy not B + MANUAL |
| `TEXTILE_TAG_PRICE_ABOVE_MRP` | 400 | price > MRP |
| `TEXTILE_TAG_SEASON_CLOSED` | 422 | closed season on a new tag |
| `TEXTILE_TAG_LIST_INACTIVE` | 422 | size/colour/fabric/season/brand inactive or out of scope |
| `TEXTILE_TAG_LIST_DUPLICATE` | 409 | quick-add of an existing name (returns it) |
| `TEXTILE_TAG_NOT_A_TAG` | 404 | scan miss (normal; falls through) |
| `TEXTILE_TAG_VOID` | 422 | scan / price / hold on a VOID tag |
| `TEXTILE_TAG_BELOW_COST` | 409 | markdown below cost without confirm |
| `TEXTILE_TAG_IN_USE` | 409 | void after stock moved |
| `TEXTILE_TAG_ITEM_LOCKED` | 422 | item change once the lot exists |
| `TEXTILE_TAG_ITEM_MISMATCH` | 422 | inward batch = a tag of another item |
| `TEXTILE_TAG_POLICY_LOCKED` | 422 | policy change on an item/group with OPEN tags |
| `TEXTILE_TAG_SERIES_CODE_USED` | 409 | changing a code that issued tags |
| `TEXTILE_TAG_RIGHT_<RIGHT>` | 403 | from `assertMenuRight` |
| `SALES_TAX_SLAB_MISMATCH` | 422 | §7 |

Plain-English wording per the simple-error-messages rule, e.g. NO_SERIES: "This counter cannot
print tags yet. Ask the admin to give it a tag code."

---

## 10. Offline and sync

- Every new table has `<p>_sync_date`, so `v_sync_tables` picks it up and the platform installs
  its sync triggers at boot (fact 8). Nothing to register.
- **Push-only today.** Tags reserved on a till reach the cloud. Tags reserved at the back office do
  **not** reach a till's local DB. This is the same gap every master has (items, prices); the
  textile module follows whatever down-sync the platform adds (B7). Until then an offline till
  scans only its own tags; online it scans all.
- Numbers never collide offline: per-device series + `ux_ilt_barcode`.
- Lot ids never collide: deterministic from (company, item, barcode).

---

## 11. Grids and dropdowns (SQL registered by migration; ids from the box)

Grid SQL has **no comments** (runner rule) and always receives every optional token
(grid-runner rule).

| grid | content |
|---|---|
| Tag register | tags ⨝ item, brand, lists, supplier, lot, `stock_balance` (branch); derived status columns; filters item, brand, size, colour, fabric, season, status, supplier, date range, search (barcode / model / article), reserved-no-stock, remnant, held |
| Series | devices of the company ⨝ series, last issued, next, count, state |
| Markup rules | rules ⨝ group, brand, with the example price for a sample cost |
| Size / colour / fabric / season masters | one each, company + shared rows |
| Textile stock report (G13, T-4) | by item section (gender, G23), group, brand, size, colour, fabric, season; on hand, value at cost / price / MRP, age, sell-through |

Dropdowns: size (filtered by size set), colour, fabric, season (open only, default by date), size
set, markup "sets" and "round rule" enums.

---

## 12. Tests (e2e against a scratch company)

1. Reserve 3 lines on two devices concurrently → unique barcodes, no reused seq.
2. Per-piece group: qty 5 → 5 tags, 5 distinct lot ids; metre item never split.
3. Reserve refused for a non-`TEXTILE_TAG` item; for a closed season; for price > MRP; for a device
   with no series — whole batch, all line errors returned.
4. **Lot id agreement:** reserve, post an opening with `batch = barcode`, then
   `stock_lot.slt_id = ilt_lot_id`. (The core assumption of this plan.)
5. Scan: RESERVED before the opening, ACTIVE after, SOLD_OUT after selling all, ACTIVE after a
   return; on hand per branch and godown.
6. Sale by scan posts with the tag's lot; a lotless textile line → `STOCK_PICK_MANUAL_LOT_REQUIRED`.
7. Markdown: below cost refused without confirm, accepted with; posted bill keeps its rate.
8. Void refused after any ledger row; allowed before.
9. Policy change on a group with OPEN tags → `TEXTILE_TAG_POLICY_LOCKED`.
10. Hold: scan for another party returns the hold; same party returns none; expired returns none.
11. Suggest price: each rule level wins in order; each rounding.
12. Slab: 2,799 with 12 % off → net 2,463 → upto-rate; 2,799 no discount → above-rate; saree
    (no slab row) → item rate; bill posted with a wrong slab → `SALES_TAX_SLAB_MISMATCH`.
13. Print dataset: per-line tag with qty 6 → 6 rows; per-piece → 1; cost code with cipher
    KRISHNAMUT and cost 1320 → `KIRT`; blank cipher → NULL.
14. Re-lot: new tag RESERVED → posted → old tag VOID, stock on the new lot.
15. Engine balance assertion = 0 after each (engine plan §5.1).

---

## 13. Build order

| step | content | depends on | unlocks (client) |
|---|---|---|---|
| **B-1** | migration (§1) + Prisma + module skeleton + reserve / get / update / void / lists save / setup save / suggest-price + errors + rights | — | T-2 tag screen (New tags, Series, Markup, lists) |
| **B-2** | grids + dropdowns (§11) + print purpose / dataset (§8) + price / hold / unhold / print | B-1 | Register + Markdown tabs, label printing; **go-live with OPENING is possible here** |
| **B-3** | scan (§5.5) + `ITEM_MISMATCH` on opening / adjustment validate (§6.1) + `POLICY_LOCKED` (§5.2) | B-1 | sale bill / DC / return / count scan (T-3) |
| **B-4** | tax slab (§7): table, statutory code, resolver route, sale-bill validation | — (independent) | bill-time slab (G1) |
| **B-5** | textile stock report grid + re-lot hook (§6.3) | B-2 | T-4 |
| **B-6** | purchase hooks (§6.4) | purchase module | T-5 |

B-4 can be built in parallel with B-1; it is not textile-only.

---

## 14. Decisions (defaults proposed; confirm before B-1)

| # | question | default |
|---|---|---|
| B1 | store only OPEN/VOID and derive RESERVED/ACTIVE/SOLD_OUT (replaces base §4 and the nightly job) | yes |
| B2 | compute `ilt_lot_id` at reserve with the engine's `lotIdentityUuid`, no FK | yes |
| B3 | block policy changes on items/groups with OPEN tags (`POLICY_LOCKED`) | yes |
| B4 | GST value slab as a general `tax_value_slab` table + statutory code, used by every sale, not only tags | yes |
| B5 | sale-bill server refuses a line whose tax ≠ the slab for its net value | yes (refuse) |
| B6 | rights checked on tag write routes (stock module checks none today) | yes |
| B7 | offline down-sync of tags | follow the platform; no textile-only down-sync |
| B8 | series code exactly 2 chars (T1); a full series needs a new code | yes |
| B9 | module folder `src/modules/stocks/textile-tags/`, tables in `inventory` | yes |
