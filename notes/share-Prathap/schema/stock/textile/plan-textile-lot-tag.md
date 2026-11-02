# Textile lot tag — one label per purchase line, scanned at the counter

Written 2026-09-27. **This is a NEW plan. It edits no existing plan.** Target: the NestJS
server (`/home/vk/Dev/erp/ERP server/`, read here through `nexerp-api-ro`) and the Qt client.

Picture: `textile_lot_barcode_fit.png` (+ `.py`) beside this file.

| existing plan | what this file does to it |
|---|---|
| `plan-nestjs-stock-engine-port.md` | nothing. Uses §1.4 (`MANUAL` issue strategy refuses a lotless line) and waits on phase 6 (purchase) for its main path |
| `plan-nestjs-stock-adjustments.md` | nothing. §7 here says what a re-lot does to a tag |
| `plan-nestjs-opening-stock.md` | nothing. Opening is how a shop's existing stock gets tags at go-live (§5.2) |
| `16_stock.sql` / the live `stock` schema | **nothing. No stock table, column, index or key changes.** |

---

## 0. The problem and the decision

A textile shop buys "Deepa Kanchi 402" once and never again. Making an item master per design
means one new item for every purchase line. The shop wants what textile software does instead:
the purchase line (item, brand, size, colour, model, MRP, selling price) gets an **automatic
number printed as a barcode label**, and the counter **scans the label** to get every detail back.

**Decided with the user on 2026-09-27:**

1. **Items stay generic** ("Silk Saree", "Cotton Saree", "Shirt"), created once. A design is a
   **lot**, not an item. No size or colour goes on the item master.
2. **Every textile detail, the MRP, the selling price and the supplier live on a TAG row** in a new
   table **outside the stock schema**. The stock structure does not change.
3. **The tag's barcode is the lot's batch number.** The stock engine only ever sees an ordinary
   batch, which it already handles.
4. **Price has ONE home: the tag.** Textile items use a batch-only policy, so the engine leaves
   `slt_mrp`, `slt_sale_price` and `slt_supplier_id` NULL (NestJS `resolveLots` blanks every
   untracked dimension; checked in `stock-voucher-posting.helper.ts:318,416`). Never store a
   price in both places.
5. **Cost stays on the lot** (`slt_purchase_rate`, `slt_cost_rate`, `slt_landed_rate`). The
   engine values stock and COGS from it. Never move cost to the tag.

## 0.1 The rules this plan is written under

These are the house rules from `plan-nestjs-stock-engine-port.md` §0, restated where they bite:

- **No SQL function or trigger.** The table, CHECKs, unique and partial indexes and FKs are
  declarative and allowed (Prisma migration; raw SQL only for what Prisma cannot express).
- **One stock engine.** Tags never write `stock_ledger`, `stock_balance` or `stock_lot`.
- **New routes get their own URL family** (`/api/v1/textile-tags/...`). No old route, DTO or
  lookup is reused or widened, **including `getItemByBarcode`**.
- **No `/list` route.** The tag register is a grid (`ui_table` + grid registration, bare params
  always sent).
- **Offline.** The till creates and scans tags locally and syncs by primary key, so the number
  series must be per device and nothing may need the cloud to answer.

## 0.2 What already exists (checked 2026-09-27, live DB + code)

| piece | state | evidence |
|---|---|---|
| A lot per distinct batch, with cost | ✅ | `stock.stock_lot`; `ux_slt_identity` |
| Lookup by batch number | ✅ indexed, **not unique** | `ix_slt_batch_no (slt_company_id, slt_batch_no)` |
| The sale line carries its lot | ✅ | `sale_bill_item.sbi_lot_id`; client sends `sbiLotId` (`sale_bill_entry.cpp:6449`); `save-bill-item.dto.ts:98`; `sales-stock.service.ts` posts `lotId` |
| Returns and DC carry the lot | ✅ | `sri_lot_id`, `sdi_lot_id`, `sdri_lot_id` |
| `BATCH` preset (signature B) | ✅ seeded, but issue strategy **FIFO** | `stock.stock_track_preset` |
| `MANUAL` issue strategy | ✅ allowed by the CHECK | `stock_track_policy` constraint |
| Brand master | ✅ | `inventory.item_brand_master` |
| Size / colour / model anywhere | ❌ | no column, no master |
| A number series for labels | ❌ | — |
| Scan → lot | ❌ | the sale-bill scan goes to `getItemByBarcode` → `item_ean_codes` only (item, no lot) |
| Purchase | ❌ no module | DB: `purchase.suppliers` + `supplier_groups` only; engine plan phase 6 |

---

## 1. The table: `inventory.item_lot_tag` (prefix `ilt_`)

Prisma migration. Not partitioned: it is a master-like row per purchase line, not a transaction
row, and a shop makes tens of thousands a year, not millions.

| column | type | rule |
|---|---|---|
| `ilt_id` | uuid PK, default `uuidv7()` | offline-safe id |
| `ilt_company_id` | uuid NOT NULL | FK `companys` |
| `ilt_tenant_id` | uuid | as every table |
| `ilt_barcode` | varchar(30) NOT NULL | the printed number = the lot's `slt_batch_no` (§2) |
| `ilt_series_id` | uuid NOT NULL | FK `item_lot_tag_series` (§2): the device series that issued it |
| `ilt_seq_no` | integer NOT NULL | position in that device's series |
| `ilt_item_id` | uuid NOT NULL | FK `item_master`; the generic item |
| `ilt_lot_id` | uuid NULL | FK `stock.stock_lot`; NULL until the inward posts (§5) |
| `ilt_brand_id` | uuid NULL | FK `item_brand_master` |
| `ilt_size` | varchar(30) NULL | free text first (decision T3) |
| `ilt_colour` | varchar(40) NULL | free text first (T3) |
| `ilt_model` | varchar(100) NULL | design / model name, e.g. "Deepa Kanchi 402" |
| `ilt_attr_note` | varchar(250) NULL | anything else printed on the label (fabric, length) |
| `ilt_mrp` | numeric(18,6) NULL | printed MRP; NULL when nothing is printed |
| `ilt_sale_price` | numeric(18,6) NOT NULL | **the** selling price |
| `ilt_tax_id` | uuid NULL | FK `tax_rate_master`; NULL = the item's tax (decision T5) |
| `ilt_supplier_id` | uuid NULL | FK `purchase.suppliers` (the lot does not keep it) |
| `ilt_src_module`, `ilt_src_doc_type`, `ilt_src_doc_id`, `ilt_src_acc_year`, `ilt_src_line_no` | | the document line that issued it (purchase, opening, adjustment) |
| `ilt_label_qty` | numeric(18,6) NOT NULL DEFAULT 1 | how many pieces the label covers |
| `ilt_print_count` | integer NOT NULL DEFAULT 0 | labels printed so far |
| `ilt_last_printed_on` | timestamptz NULL | |
| `ilt_status` | varchar(20) NOT NULL DEFAULT `'RESERVED'` | `RESERVED` → `ACTIVE` → `SOLD_OUT`, or `VOID` (§4) |
| audit + sync | | `ilt_is_active`, `ilt_is_deleted`, `ilt_created_on/by`, `ilt_modified_on/by`, `ilt_sync_date`, `ilt_row_version` |

**Constraints and indexes (declarative only):**

- `ux_ilt_barcode UNIQUE (ilt_company_id, upper(ilt_barcode)) WHERE ilt_is_deleted = false`:
  **one scan = one tag**, company-wide. This is where uniqueness lives. `ix_slt_batch_no`
  stays as it is.
- `ux_ilt_series UNIQUE (ilt_series_id, ilt_seq_no)`.
- `ck_ilt_prices`: `ilt_sale_price >= 0 AND (ilt_mrp IS NULL OR ilt_mrp >= 0)
  AND (ilt_mrp IS NULL OR ilt_sale_price <= ilt_mrp)`.
- `ck_ilt_status`: the four values.
- `ck_ilt_lot_when_active`: `ilt_status = 'RESERVED' OR ilt_status = 'VOID' OR ilt_lot_id IS NOT NULL`.
- Indexes: `(ilt_company_id, ilt_lot_id)`, `(ilt_company_id, ilt_item_id, ilt_status)`,
  `(ilt_company_id, ilt_brand_id)`, `(ilt_src_doc_id)`, and a sync-pending partial index like
  every table.

**Price history.** A markdown is an UPDATE on `ilt_sale_price`. What a customer actually paid is
on the sale line (`sbi_rate`), so the tag never needs history for billing. The audit trail comes
from `audit.audit_log`, as for any master.

## 2. The number series

**`fixed.device_master` has no short code** (checked live: `dev_device_uid`, `dev_device_name`,
no code column), so the series gets its own small table rather than a change to the device
master:

**`inventory.item_lot_tag_series`** (prefix `ilts_`): `ilts_id` uuid PK, `ilts_company_id`,
`ilts_device_id` (FK `fixed.device_master`), `ilts_code` varchar(4) (uppercase letters and digits),
`ilts_is_active`, plus audit and sync columns.
`UNIQUE (ilts_company_id, upper(ilts_code))`, `UNIQUE (ilts_company_id, ilts_device_id)`.
An admin assigns a code per device on the tag screen, once.

- **Format:** `<series code><running number>`, e.g. `T1` + `004127` → `T1004127` (T1). Only
  uppercase letters and digits, so Code 128 prints it short and every scanner reads it.
- **Per device**, so an offline till never collides with another. `ux_ilt_barcode` enforces
  company-wide uniqueness anyway.
- **Allocation:** `MAX(ilt_seq_no)+1` for the series under an advisory lock on the series id. This
  is the same pattern as `stock-voucher-numbering.helper.ts`. It is **never** a bare `MAX+1`
  (engine plan §1.6 is the bug this avoids).
- A device with no series row cannot reserve: `NO_TAG_SERIES` (T7).
- A reserved number that is later voided is **not reused**. Gaps are fine; labels may already be
  printed.

## 3. The tracking policy for textile items

Seed one new preset (data, in the migration):

| `spt_code` | name | flags | `issue_strategy` | allow negative |
|---|---|---|---|---|
| `TEXTILE_TAG` | Textile tag (label per purchase line) | batch only (signature `B`) | **`MANUAL`** | no |

- **Why MANUAL and not the BATCH preset's FIFO.** Engine plan §1.4 picks lots by issue strategy
  when an outward line arrives without a lot, and its default D1 is "pick at post". For a textile
  item that would move **a different design** than the one sold. `MANUAL` makes the engine
  refuse a lotless outward line, so every textile line must come from a scan (or an explicit lot
  pick).
- Set it on the textile item **group** (the item group "Tracked as" combo), so every generic
  item under it inherits it.
- **Never turn on MRP, selling-price or supplier tracking for these items.** The price would
  then be stored twice, and one barcode could resolve to several lots.

## 4. Tag lifecycle

| status | meaning | how it gets there |
|---|---|---|
| `RESERVED` | number issued, details keyed, label may be printed, **no stock yet** | `POST /textile-tags/reserve` from a draft document line |
| `ACTIVE` | the inward posted; `ilt_lot_id` set | the issuing document's post (§5) |
| `SOLD_OUT` | the lot's `slt_total_on_hand` reached 0 | set by the tag service after a sale or return posts; **informational only**, since a return brings it back to `ACTIVE` |
| `VOID` | the draft line was deleted, or the label was spoiled before any stock moved | `POST /textile-tags/void`; **refused once any ledger row cites the lot** |

A `RESERVED` tag older than N days with no lot is an orphan. It is shown on the tag register as a
filter. It is never auto-deleted, because a label may be on a shelf.

## 5. Where tags are born

### 5.1 Purchase (the main path; engine plan phase 6)

The purchase entry does not exist yet. When it is built:

1. Each textile line has the tag fields: brand, size, colour, model, MRP, selling price,
   and label qty (defaults to the line qty; 1 = one label per piece).
2. On save (draft), the line calls reserve (§6). The returned barcode goes into the line's
   batch number. Labels can print from the draft, since goods are labelled while unpacking.
3. On post, the purchase's inward line goes through the ONE engine with `batch = barcode`. The
   engine resolves or creates the lot. **In the same transaction** the purchase service sets
   `ilt_lot_id = svi_lot_id`, `ilt_status = 'ACTIVE'` for its tags. That is one UPDATE joined
   on (`ilt_src_doc_id`, `ilt_src_line_no`). It is the document's own unit of work, not engine
   logic.
4. Purchase cancel: the lot's stock is reversed by the engine; the tags go back to `RESERVED`
   (or `VOID` if the user deletes the draft afterwards).

### 5.2 Opening stock at go-live (built module, used as is)

A shop moving onto the system has stock on its shelves. It:
1. reserves tags in bulk from the tag screen (§8), keyed like purchase lines;
2. prints and sticks the labels;
3. enters an **OPENING** document whose lines carry `batch = barcode`. The existing opening
   module needs no change.

Because the opening service does not know about tags, **the link is made lazily** (§6 scan):
a tag with `ilt_lot_id` NULL resolves its lot by (`company`, `ilt_item_id`,
`slt_key_batch = upper(ilt_barcode)`). Under the `TEXTILE_TAG` policy that is exactly one lot.
The first write route that touches the tag (print, price, or the nightly job in §10) stores the
id and flips it to `ACTIVE`. **The scan route itself never writes.**

### 5.3 Adjustment inward (adjustments plan)

Same as opening: the line carries `batch = barcode`, and the link is lazy.

## 6. Routes — `/api/v1/textile-tags`

Every route takes `companyId` and `branchId`. Rights sit on the tag menu (client registration).

| method + path | purpose |
|---|---|
| `POST /textile-tags/reserve` | `{companyId, branchId, deviceId, src?: {module, docType, docId, accYear}, lines: [{lineNo?, itemId, brandId?, size?, colour?, model?, attrNote?, mrp?, salePrice, taxId?, supplierId?, labelQty}]}` → the barcodes, in line order. One transaction, one advisory lock per device |
| `GET /textile-tags/scan?companyId&branchId&godownId?&barcode` | **the counter lookup.** Returns tag + item name + unit + lot id + batch no + MRP + selling price + tax + brand/size/colour/model + on hand (branch, and godown when given) + status. `404 NOT_A_TAG` when no tag matches, so the client falls through to the EAN lookup. Read-only |
| `GET /textile-tags/get?companyId&tagId` | one tag for the edit form |
| `POST /textile-tags/update` | details only (brand, size, colour, model, note, supplier). **`itemId` is immutable once the tag is ACTIVE**: a wrong item is a re-lot (§7) |
| `POST /textile-tags/price` | `{companyId, tagIds[], salePrice?, mrp?, percentOff?}`: markdown, one tag or many. `ck_ilt_prices` still applies. Refused on `VOID` |
| `POST /textile-tags/print` | `{companyId, tagIds[], copies?}` → print payload for the label dataset (§9); bumps `ilt_print_count`, `ilt_last_printed_on` |
| `POST /textile-tags/void` | `{companyId, tagId, reason}`; refused once any `stock_ledger` row cites the lot |

**Scan resolution order in the service:** tag by `(company, upper(barcode))` → lot by
`ilt_lot_id`, else by (company, item, `slt_key_batch`) → balance for (branch, godown?, lot). If
the lazy join finds more than one lot, return `409 TAG_AMBIGUOUS`. That can only happen if
someone broke §3 by turning on another tracked dimension.

## 7. What each existing flow does with a tagged lot

| flow | what happens | change needed |
|---|---|---|
| **Sale bill / DC / order** | scan → `/textile-tags/scan` → row gets item, `lotId`, batch = barcode, rate = `ilt_sale_price`, MRP = `ilt_mrp`, tax (T5), qty 1. The lot cells are read-only on that row | client only (§8.2) |
| **Sale price** | the tag price **replaces** the price-master / bucket price for that row. The engine plan's phase 3 `PriceResolverService` is **not** consulted for a tagged row | client + the scan route |
| **Promotions / loyalty** | the promotion binder runs on top of the tag price as usual. **Brand-based schemes match the item's brand, not the tag's** (decision T4) | client binder, only if T4 = tag brand |
| **Sale return** | scan the same label → the same lot → `sri_lot_id`; stock comes back; the tag leaves `SOLD_OUT` | client only |
| **Transfer** | the lot moves (engine plan §1.1: "never re-resolved"); a tag is company-wide, so the label scans at the other branch | none |
| **Physical count** | scan labels; the line's batch = barcode, item from the tag | client lookup on the count screen (optional) |
| **Re-lot** (adjustments §2) | a re-lot moves stock to a new identity, i.e. a new barcode. **The re-lot's post sets the old tag `VOID` and the new tag `ACTIVE`**; the new tag is reserved first like any other | adjustments service, one UPDATE in its transaction |
| **Change Selling Price screen** | reprices the item master and the MRP buckets, **not tags**; textile goods are repriced through `/textile-tags/price` | none; say so on the screen |

## 8. Client (Qt) work

### 8.1 Tag screen (menu id: box assigns)

- A **register grid** of tags: filters on item, brand, status, date, supplier, and "reserved,
  no stock" (the orphan view). Columns: barcode, item, brand, size, colour, model, MRP, price,
  on hand, age days. `ui_table` + grid registration; bare params always sent.
- **Entry form:** the purchase-line fields, with a "labels" count. Save = reserve; Print.
- **Bulk reserve** for go-live (§5.2): a grid of lines, then reserve all, then print all.
- **Markdown:** select rows, then "price" (new price or % off), then reprint.
- Its own `.ui` on `NexDialog`, per the house pattern for a screen with a grid plus actions.
  The **PNG mockup comes first**, before any client code (house rule).

### 8.2 Sale bill scan (`SaleBillEntry::resolveScannedBarcode`, `sale_bill_entry.cpp:3130`)

Order, when the company setting `stock.textile_tags_enabled` is on (a row in
`public.app_setting_def`, value per company in `app_setting_value`, seeded off):
1. the weight / price label rule (exists);
2. **`/textile-tags/scan`**; on `NOT_A_TAG` fall through;
3. `getItemByBarcode` (EAN, exists).

With the setting off, step 2 is skipped, so a supermarket pays nothing. The same helper serves
the DC, the sale return, the order and the quotation (a quotation fills the price and details but
moves no stock).

A tagged row sends `sbiLotId` and `sbiBatchNo` exactly as a picked lot does today. No DTO
change: the `shared-payload-per-dto-keys` trap does not apply, because no new key is sent.

## 9. Label printing

A printing-module dataset (`17_printing.sql`, stored-SQL datasets, data-driven): `TEXTILE_TAG`
over `item_lot_tag` ⨝ `item_master` ⨝ `item_brand_master`, one row per label copy. Fields:
barcode (Code 128), item, brand, size, colour, model, MRP ("MRP ₹2,450 incl. of all taxes" when
set), selling price, month/year packed. The design lives in the printing Designer. This plan
fixes only the dataset's columns.

## 10. Reports and the nightly job

- **Stock by design / brand / size / colour:** `stock_balance` ⨝ `stock_lot` ⨝ `item_lot_tag`
  (on `ilt_lot_id`, or the lazy key). A read service under the engine plan's phase 4 family:
  `GET /stock/reports/textile?…`.
- **Value at selling price / MRP:** on hand × `ilt_sale_price` / `ilt_mrp`. The engine plan's
  §5.3 snapshot columns `svs_value_at_sp` / `svs_value_at_mrp` read the lot and will be **0 for
  textile**. The textile report uses the tag; the snapshot is left alone.
- **Ageing per design:** from `slt_first_inward_date`, so slow designs show on their own.
- **Nightly job** (BullMQ, beside the engine plan's §5): link lazy tags (§5.2), set
  `SOLD_OUT` / back to `ACTIVE` from `slt_total_on_hand`, and list orphans. Detect and link only;
  it never touches stock.

## 11. Offline

- Reserve runs on the till against the till's own series. Rows sync up by PK.
- Tags must **sync down** to every till of the company, or a till cannot scan a label printed
  at the back office. Add `item_lot_tag` to the cloud-sync table list, going both ways, with
  last-writer-wins on details and price (a markdown made at HQ reaches the tills).
- The scan route's logic is a pure read, so the till runs the same query locally.

## 12. Validation

1. `salePrice` is required, ≥ 0, and ≤ MRP when an MRP is set.
2. The item's effective policy must have the signature `B` with issue strategy `MANUAL`.
   Otherwise reserve refuses: `ITEM_NOT_TEXTILE_TRACKED`.
3. `labelQty` > 0.
4. Brand, supplier, tax and item must be active and belong to the company (or be shared).
5. Void: refused once the lot has any ledger row.
6. Update: `itemId` is immutable once `ACTIVE`.
7. An inward line whose batch **looks like** a tag (matches a `RESERVED`/`ACTIVE` barcode) but
   whose item differs from the tag's item is refused by the issuing document's validate:
   `TAG_ITEM_MISMATCH`.

## 13. Tests

- reserve 3 lines on two devices concurrently: no duplicate barcode; the series has no reuse.
- opening with `batch = barcode`, then scan: lazy join gives the lot; on hand = opening qty.
- sale by scan: `sbi_lot_id` = that lot; on hand −1; the rate = the tag price, not the price master.
- a sale line of a `TEXTILE_TAG` item with no lot: refused by the engine (§1.4 `MANUAL`).
- return by scan: stock back; `SOLD_OUT` → `ACTIVE`.
- markdown: the price changes on the next scan; a posted bill keeps its old rate.
- transfer to branch B, then scan at B: found, B's on hand.
- void after a sale: refused.
- re-lot: old tag `VOID`, new tag `ACTIVE`, qty and value carried.
- setting off: a scan never calls the tag route.
- the engine plan's §5.1 balance assertion = 0 after each.

## 14. Build order

| step | what | depends on |
|---|---|---|
| T-1 | migration (both tables, indexes, preset seed, setting def) + reserve / get / update / price / print / void + number series | nothing |
| T-2 | tag screen (mockup first) + label dataset. **A shop can go live here with OPENING** (§5.2) | T-1 |
| T-3 | scan route + sale bill / DC / return scan branch | T-1; engine plan §1.4 `MANUAL` refusal |
| T-4 | nightly job + textile stock report | T-1 |
| T-5 | purchase entry lot-first (§5.1) | engine plan phase 6 |

## 15. Open decisions

| # | question | default if nobody answers |
|---|---|---|
| T1 | barcode format: series-code length, zero padding, any company prefix? | 2-char series code + 6 digits (`T1004127`) |
| T2 | label per piece (qty 1 each) or per line (one label covering qty N)? | per line, `labelQty` = qty, printed N times; the scan adds 1 per scan |
| T3 | size / colour as free text or small masters (`item_size_master`, `item_colour_master`)? | free text now; masters later, when reports need clean grouping |
| T4 | brand on the tag: should brand-based promotions / loyalty match the **tag** brand? | yes: the binder uses the tag brand when the line has a tag |
| T5 | GST for garments depends on the per-piece sale value (a slab threshold, which is a law param: per company, via `statutory_limits`, never a constant). Resolve `ilt_tax_id` at reserve and on every markdown from the price, or take the item's tax? | resolve from the price by the company's slab rule; sarees and fabric use the item's flat rate |
| T6 | may a markdown go **below cost**? | allowed, with a confirm on the screen (same as the change-selling screen's below-cost confirm) |
| T7 | a device with no `item_lot_tag_series` row | refuse reserve (`NO_TAG_SERIES`) until an admin assigns a code |
