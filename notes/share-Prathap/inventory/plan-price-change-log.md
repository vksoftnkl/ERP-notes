# Plan — Item price change log + date-wise report (2026-10-23)

**User's ask (in their words):** "want to record every change of item price
master into separate log so that user side and our side will be easy if someone
ask me i dont know how rate was modified … which item on which company branch
user time device record all the fields into log … a detailed report to show
user by date wise".

The goal: when a customer says "I didn't change this rate", the shop (and we)
can open one report and see exactly who changed it, when, from which machine,
on which screen, and from what to what.

Status: **design for approval.** Nothing is built.

Mockup: `plan-price-change-log_mockup.png` (same folder).

---

## 0. What exists today, and why it is not enough

`audit.audit_log` already gets "item price master" rows (21–22 Oct 2026, ~120
rows, almost all from tests). They hold before/after JSON and the user. But:

| Need | audit.audit_log today |
|---|---|
| Company / branch of the price row | no column; only inside the JSON, as labels ("Branch ID"); `log_branch_id` is **NULL** on every price row |
| Device | `log_device_name` is **NULL** on every price row (the IP is there) |
| Item, for a filter or index | only inside the JSON |
| One row per item price, cheap to report by date | mixed with every other table's audit; no index for this |
| Every write path logged | **no** — the MRP/SP bucket re-key (`price-bucket.service.ts` ~180, `rekeyItem`) updates `item_price_master` with **no** audit row |
| Field names usable by a report | changed fields are stored under display labels ("Sales Price A", "Price  A Markup %"), not columns |
| Kept for years | it is the general audit trail, not built to be the evidence for prices |

So this plan adds a **separate, append-only price log**, written in the same
transaction as every price write. `audit.audit_log` stays as it is.

---

## 1. The table — `inventory.item_price_change_log`

One row per price row changed per save. A save that changes 3 rows writes 3
log rows that share one `ipcl_change_set_id`.

```sql
CREATE TABLE inventory.item_price_change_log (
  ipcl_id              uuid        PRIMARY KEY DEFAULT uuidv7(),
  ipcl_changed_on      timestamptz NOT NULL DEFAULT now(),
  ipcl_change_set_id   uuid        NOT NULL,            -- one save / one request
  ipcl_price_table     varchar(4)  NOT NULL,            -- 'IPM' item_price_master | 'IQP' item_qty_price
  ipcl_row_id          uuid        NOT NULL,            -- ipm_id / iqp_id
  ipcl_action          varchar(10) NOT NULL,            -- CREATE | UPDATE | DELETE | REKEY
  ipcl_source          varchar(30) NOT NULL,            -- see §2
  ipcl_menu_id         integer,                         -- the screen, when a screen did it

  -- WHERE the price applies (copied from the row; AFTER for create/update, BEFORE for delete)
  ipcl_company_id      uuid,                            -- NULL = shared item / chain price
  ipcl_branch_id       uuid,                            -- NULL = all branches
  ipcl_item_id         uuid        NOT NULL,
  ipcl_unit_id         uuid,                            -- iuc_id
  ipcl_godown_id       uuid,
  ipcl_party_id        uuid,                            -- IQP only
  ipcl_price_level     smallint,                        -- IQP only
  ipcl_bucket_mrp      numeric(18,2),
  ipcl_bucket_sp       numeric(18,2),
  ipcl_effective_from  date,
  ipcl_effective_to    date,

  -- names AS THEY WERE, so the report reads right after a rename or delete
  ipcl_item_code       varchar(60),
  ipcl_item_name       varchar(250),
  ipcl_unit_name       varchar(60),
  ipcl_branch_name     varchar(120),
  ipcl_godown_name     varchar(120),

  -- WHO / FROM WHERE
  ipcl_user_id         uuid,                            -- from the TOKEN, never the body
  ipcl_user_name       varchar(150),
  ipcl_device_id       uuid,                            -- X-Device-Id header
  ipcl_device_name     varchar(150),
  ipcl_ip              varchar(60),
  ipcl_app             varchar(30),                     -- DESKTOP | WEB | API | SYSTEM
  ipcl_reason          varchar(500),                    -- operator note / "confirmed below cost …"

  -- WHAT: every column, before and after (raw column names, numbers as numbers)
  ipcl_before          jsonb,                           -- NULL on CREATE
  ipcl_after           jsonb,                           -- NULL on DELETE
  ipcl_changed         text[]      NOT NULL,            -- e.g. {ipm_sales_price_a, ipm_price_a_wot}

  -- the figures the report filters and sorts on, generated from the JSON
  ipcl_old_price_a numeric GENERATED ALWAYS AS ((ipcl_before->>'ipm_sales_price_a')::numeric) STORED,
  ipcl_new_price_a numeric GENERATED ALWAYS AS ((ipcl_after ->>'ipm_sales_price_a')::numeric) STORED,
  ipcl_old_mrp     numeric GENERATED ALWAYS AS ((ipcl_before->>'ipm_max_price')::numeric)     STORED,
  ipcl_new_mrp     numeric GENERATED ALWAYS AS ((ipcl_after ->>'ipm_max_price')::numeric)     STORED,
  ipcl_old_cost    numeric GENERATED ALWAYS AS ((ipcl_before->>'ipm_cost_price')::numeric)    STORED,
  ipcl_new_cost    numeric GENERATED ALWAYS AS ((ipcl_after ->>'ipm_cost_price')::numeric)    STORED,

  CONSTRAINT ck_ipcl_table  CHECK (ipcl_price_table IN ('IPM','IQP')),
  CONSTRAINT ck_ipcl_action CHECK (ipcl_action IN ('CREATE','UPDATE','DELETE','REKEY')),
  CONSTRAINT ck_ipcl_shape  CHECK ((ipcl_action = 'CREATE' AND ipcl_before IS NULL AND ipcl_after IS NOT NULL)
                                OR (ipcl_action = 'DELETE' AND ipcl_before IS NOT NULL AND ipcl_after IS NULL)
                                OR (ipcl_action IN ('UPDATE','REKEY') AND ipcl_before IS NOT NULL AND ipcl_after IS NOT NULL)),
  CONSTRAINT ck_ipcl_changed CHECK (cardinality(ipcl_changed) > 0)
);

CREATE INDEX ix_ipcl_scope_date ON inventory.item_price_change_log (ipcl_company_id, ipcl_branch_id, ipcl_changed_on DESC);
CREATE INDEX ix_ipcl_item_date  ON inventory.item_price_change_log (ipcl_item_id, ipcl_changed_on DESC);
CREATE INDEX ix_ipcl_user_date  ON inventory.item_price_change_log (ipcl_user_id, ipcl_changed_on DESC);
CREATE INDEX ix_ipcl_set        ON inventory.item_price_change_log (ipcl_change_set_id);

-- append-only: the API role may only INSERT and SELECT
REVOKE UPDATE, DELETE, TRUNCATE ON inventory.item_price_change_log FROM <api role>;
```

**Rules for the table:**
- No trigger and no function (our rule). Generated columns, CHECKs, indexes and grants are fine.
- Partitioning is not needed at today's volume: about one row per price row changed. Add yearly partitions on `ipcl_changed_on` later if it grows, the same way the transaction tables are partitioned.
- No purge route. Rows are kept for good, because this table is the evidence.

---

## 2. Every price write goes through ONE writer, and the writer always logs

Since triggers are not allowed, the guarantee has to live in the code:

1. **`PriceMasterWriter`** (new, in `items-price-master/`) is the only code allowed to touch `item_price_master` and `item_qty_price`:
   - `create(tx, row, ctx)`
   - `update(tx, ipmId, patch, ctx)`
   - `softDelete(tx, ipmId, ctx)`
   - `rekey(tx, ipmId, key, ctx)`
   - the same four for qty price

   Each method reads the row FOR UPDATE, writes it, then calls `PriceChangeLog.record(tx, before, after, ctx)`. Both happen in one transaction, so a price change with no log row cannot commit.
2. **`ctx`** carries the source, menu id, reason and change-set id:
   - `source` / `menuId`: who in the code is calling.
   - `reason`: what the operator typed or confirmed.
   - `changeSetId`: one per request.

   The user, device, IP and app come from `RequestContextService` and the token, **never from the body** (see notes-143 #9).
3. **No-op saves write nothing.** `ipcl_changed` is computed over the business columns only. `*_updated_on`, `*_sync_date`, `*_sl_no` and the `*_key_*` generated keys are ignored. If nothing else changed, no row is written.
4. **Sources** (`ipcl_source`):

   | source | written by |
   |---|---|
   | `ITEM_CARD` | item master save: price grid, `prices[]` on `/items/create`; `items-price-master.service.ts` |
   | `CHANGE_SELLING_PRICE` | menu 30, `selling-price-bulk/price-bucket.gateway.ts` (6 writes today) |
   | `QTY_PRICE` | menu 244, `items-qty-price-master.service.ts` |
   | `BUCKET_REKEY` | `price-bucket.service.ts rekeyItem`, which writes NO audit today; `app = SYSTEM`, user = whoever's save triggered it |
   | `IMPORT` | any CSV / bulk import that writes prices, now or later |

5. **Guard test.** An e2e/spec check fails the build if `itemPriceMaster.(create|update|upsert|delete…)`, `itemQtyPrice.(…)` or a raw `INSERT/UPDATE/DELETE … item_price_master|item_qty_price` appears outside `PriceMasterWriter`. This stops a future screen from writing prices without a log.
6. **Gap check (belt and braces).** The report's summary route also counts price rows whose `ipm_updated_on` is newer than their last log row: "changed outside the log". It should always be 0. If it isn't, it shows as a red line on the report and tells us a path was missed.

---

## 3. Routes (new module `item-price-log`, read-only)

All of these need VIEW on the report's menu (new menu id, "Price Change Log").
Company and branch are checked against the token.

| route | what |
|---|---|
| `GET /item-price-log/list` | one row per change, newest first. Params: `companyId`, `branchId?` (empty = all the user may see), `fromDate`, `toDate` (date-wise, inclusive, local day), `itemId?`, `userId?`, `deviceId?`, `source?`, `field?` (any `ipcl_changed` member, e.g. `ipm_sales_price_a`), `search?` (item code/name), `page`, `limit ≤ 200`. Returns the identity and names, the who/where columns, `changed[]`, and a `diff[]` of `{ field, label, from, to }` built from before/after so the client does not re-derive it. |
| `GET /item-price-log/get?ipclId=` | one change, with full `before`, `after` and `diff` (every field, changed or not, flagged). |
| `GET /item-price-log/days` | same filters → per day: changes, items, users, devices; and the gap count (§2.6). This drives the date-wise headers and the day strip. |
| `GET /item-price-log/item-history?itemId=&companyId=&branchId?` | one item's timeline across all its units, buckets and branches, oldest → newest, with the price in force after each change. Answers "how did this rate become 118?". |

Field labels such as "Price A", "MRP" and "Cost" come from one server map, so
the report, the CSV and the printout all name a field the same way.

---

## 4. The report screen (Qt) — "Price Change Log"

See the mockup.
- **Filters:** From / To (default today; ◀ ▶ steps a day), Branch (All / one), Item (picker), User, Device, Source, Field changed, Search. "Only price fields" is on by default; off also shows markup %, remarks and godown changes.
- **Day strip:** one chip per day in the range: date, number of changes, number of items. Click a chip to jump to that day.
- **Grid, grouped by date.** Each day has a header row ("Thu 23-10-2026 · 14 changes · 6 items · 2 users"). Each change row shows:
  - Time (AM/PM), Item (code · name), Unit, Branch, MRP bucket.
  - What changed: chips such as "Price A 118 → 119 ▲0.8%" and "MRP 150 → 160".
  - By (user), Device, Source (Item card / Change selling price / Qty price / System) and Reason.
  - A price going DOWN shows red, UP shows green, and a below-cost confirmation gets a warning chip.
- **Enter / double-click** opens the detail popup: every field in three columns, Before | After | Change, with changed rows highlighted. It also shows the change-set (other rows saved in the same click), the device, IP and app.
- **Item history** (F3 on a row, or pick an item) shows that item's timeline, which is the "how did the rate get here" view.
- **Export CSV** and **Print / PDF** (through the print engine) give the customer a copy. The header lists company, branch, date range, filters and who printed it, and when.
- Keys follow `TxnKeys`: Ctrl+F search, Ctrl+P print, F1 keys.

---

## 5. Order of work

1. **Server (Prathap):**
   - Add the table (§1).
   - Add `PriceMasterWriter` and `PriceChangeLog`.
   - Move all 5 sources onto it (§2).
   - Add the guard test, then the 4 routes (§3).
   - Fill `log_device_name` and `log_branch_id` on the existing audit rows while there (cheap, same context).
2. **Qt (us):** the report screen (§4), a menu, a grid and a live test. The test changes a ZT- item price from the item card, from Change Selling Price and from Qty price, then checks one log row per change, the device, the user and the date grouping.
3. **Backfill:** none. The 120 existing audit rows are test data from 21–22 Oct. The log starts the day it ships, and the report says "recorded from <date>".

## 6. Questions for the user

- **Q1** Include the qty-wise slab prices (menu 244) in the same log? (Planned: yes, `IQP`.)
- **Q2** Should the shop's own users see the report, or admins only? (Planned: a normal menu right, so the shop decides who gets it.)
