# NestJS — Opening Stock on the new stock engine

Target repo: `/home/vk/Projects/ERP/ERP-Server` (NestJS + Prisma, PostgreSQL 18).
Schema it is written against: `schema/stock/16_stock.sql` + `19_stock_posting.sql`,
read queries `schema/stock/19q_opening_stock_queries.sql`, worked example
`schema/stock/19_opening_stock_flow.md`, screen `opening_stock_ui_mockup.png`.

## Context

**There is no opening-stock table.** Opening stock is a `stock.stock_voucher` row
with `svh_voucher_type = 'OPENING'` and its `stock.stock_voucher_item` lines —
decided 2026-09-02 over a dedicated header/detail pair, because eleven document
types differ in three columns and share forty. Everything downstream of the
document — the lot, the ledger row, the balance, the moving average — is written
by `stock.fn_svh_post()` and by triggers. The API writes **two tables and calls
one function.**

`19_opening_stock_flow.md` captured a real run from an empty database: three rows
of application writing (1 voucher + 2 lines), six rows that maintained themselves.
That file is the acceptance test for this module, figure for figure.

**Scope of this plan (decided):** a new module built **only** on `schema/stock/`
— draft save, load, list, preflight, post, cancel, plus the two go-live reports.

**Nothing legacy is used, extended, called or copied.** Not
`src/modules/stocks/opening-stock/` (which writes `stock.opening_stock_header` /
`_detail` and, by its own plan, *"stock ledger posting and stock balance posting
are out of scope"*), not `item-stock-ledger.service.ts`, not
`inventory.item_stock_ledger` / `item_stock_balance` / `item_batch_stock`, not
`accounts.acc_voucher_header` and its numbering helper. Those tables are
Prisma-owned, they are frozen, and they are not read or written from here. §0.2
says what to do with the module that is already mounted.

**Database state:** the `stock` schema tables come from the share's `.sql` files.
Prisma does not know them. So: **models, no migrations** — the same rule as
`plan-nestjs-adjustments.md`. `stock` is already in the `schemas` list of
`prisma/base.prisma`, so nothing changes there.

---

## 0. Blockers to settle first

### 0.1 The schema is not on the live database

As of 2026-09-07 nothing from `schema/stock/` has been deployed to
192.168.0.106 — all of it was verified on throwaway PG 18 clusters. Run order,
whole chain, in one go:

```
00_init … 15  →  stock/16_stock.sql  →  stock/16t  →  stock/16s  →  17  →  18
                 →  stock/19_stock_posting.sql  →  stock/20_stock_transfer.sql
```

`16_stock.sql` carries its own `stock.fn_create_stock_partitions` — the public
one in `00_init.sql` scans only public/sales/accounts and will not create the
`stock.*` year partitions. **Call it for the current accounting year before the
first INSERT**, or every write fails with "no partition of relation".

Nothing below can be built or tested until this is done.

### 0.2 Retire the module that is already mounted

`/api/v1/opening-stocks` and its `OpeningStockModule` post nothing to stock and
must not keep taking openings once this lands: two opening-stock paths into one
database, writing two unrelated sets of tables that nothing reconciles, is how a
branch ends up with an opening that exists twice and neither figure wrong on its
own.

Decide before the first live opening:

- **Recommended:** unregister `OpeningStockModule` from `app.module.ts` in the
  same release. Nothing in the Qt client calls it — there is no stock screen
  there yet — so retiring it costs nothing.
- If it must stay mounted for an external caller, mark it `@ApiExcludeController`
  and stop pointing anything new at it. It is not a fallback for this module.

Existing balances are brought over by `19m_migrate_opening.sql`, which is part of
`schema/stock/` and runs in psql, not through the API.

### 0.3 Masters that must exist before the screen opens

| Table | Why |
|---|---|
| `fixed.device_master` | `svh_device_id` is **NOT NULL**. The device *is* the counter (§3). No device row, no document. |
| `inventory.godown_locations` | every movement needs one; `br_default_godown_id` fills the picker |
| `inventory.item_unit_conversion` | `svi_uom_id` is **`iuc_id`, not `unit_id`** — see §4.3 |
| `stock.stock_track_policy` | optional per item; no row = "track nothing, WAVG, FEFO, ALLOW" and that is a complete answer |

### 0.4 Client-side rows

`menu_master` 44 (Opening Stock) and the `ui_table_master` / grid rows the Qt
grid reads do not exist yet. Same gap the Change Selling Price screen hit. Not
this module's work, but the screen cannot open without them.

---

## 1. Prisma models — no migration

`prisma/schema.prisma` is **auto-generated** by `scripts/build-prisma-schema.js`.
Never edit it; add source files and rebuild.

New folder `prisma/stocks/Stock_Voucher/`:

| File | Model | Table | Written? |
|---|---|---|---|
| `Stock_Voucher.prisma` | `StockVoucher` | `stock.stock_voucher` | **yes** |
| `Stock_Voucher_Item.prisma` | `StockVoucherItem` | `stock.stock_voucher_item` | **yes** |
| `Stock_Lot.prisma` | `StockLot` | `stock.stock_lot` | read only |
| `Stock_Balance.prisma` | `StockBalance` | `stock.stock_balance` | read only |
| `Stock_Item_Cost.prisma` | `StockItemCost` | `stock.stock_item_cost` | read only |
| `Stock_Track_Policy.prisma` | `StockTrackPolicy` | `stock.stock_track_policy` | read only |

`stock.stock_ledger` needs **no model at all**. Nothing in TypeScript may insert
into it, and the two queries that read it (§10) are raw. Leaving it unmapped is
the cheapest possible guard.

House style for these files: camelCase fields with `@map("snake_case")`,
`@db.Uuid` / `@db.Decimal(18,6)` / `@db.VarChar(n)`, `@@map` + `@@schema("stock")`,
no relation fields to Prisma-owned masters (`item_master`, `godown_locations`,
`device_master` are read by raw joins, not by Prisma relations — a relation here
would make this module depend on models it must not write).

Six things then differ from every other model in the repo, and each one is a bug
if missed:

1. **Both written tables are PARTITIONED BY LIST (acc_year), with a composite
   primary key.** `@@id([svhId, svhAccYear])` and `@@id([sviId, sviAccYear])`.
   Every `findUnique`/`update`/`delete` therefore needs **both** values —
   an id alone will not compile, which is exactly the protection wanted.
2. **`character(9)`, not varchar.** `String @db.Char(9)` for `svh_acc_year` /
   `svi_acc_year`. `bpchar` space-pads anything shorter than 9, and
   `ck_svh_acc_year` then rejects it; always send the full `YYYY-YYYY`.
   (`AccVoucherSeq` maps its `accYear` as `VarChar(9)` — do not copy that here.)
3. **Generated columns.** `svi_value`, `svi_value_wot` and `svi_diff_qty` are
   `GENERATED ALWAYS … STORED`. Prisma has no concept for them and will happily
   put them in an INSERT. Map them as read fields, comment them, and **never**
   include them in a `create`/`update`. `svi_value` is
   `(base_qty + free_base_qty) × cost_rate` — the flow doc's line 1 is
   120 × 20 = 2,400.00, **not** 10 × 20.
4. **Header totals are trigger-maintained.** `svh_line_count`,
   `svh_total_qty`, `svh_total_value`, `svh_total_value_wot` are re-summed by
   `tr_svi_refresh_header` on every line write, and recomputed again by
   `fn_svh_recompute` at post. They are not generated columns, so Prisma *can*
   write them — which is the danger. Never send them; never compute them in TS.
   **The screen never sums its own grid** either.
5. **No Prisma enums.** House rule for these tables is `varchar + CHECK`.
   `svh_voucher_type`, `svh_status`, `svi_bucket`, `svh_rate_source` →
   `String @db.VarChar(n)` with TS union types. A Prisma enum over a CHECK column
   breaks silently the moment a value is added — and this CHECK has eleven.
6. **`@default(dbgenerated("uuidv7()"))`** on both ids, as everywhere else here.

Then:

```bash
node scripts/build-prisma-schema.js && npx prisma generate
```

---

## 2. Module shape — one service, one controller per document type

```
src/modules/stocks/stock-voucher/            <- shared, type-agnostic
  stock-voucher.module.ts
  stock-voucher.service.ts                   <- save / load / list / post / cancel
  stock-voucher-numbering.helper.ts
  stock-voucher-exception.filter.ts
  types/stock-voucher.types.ts               <- unions, SQLSTATE map
  dto/save-stock-voucher.dto.ts

src/modules/stocks/opening-stock-voucher/    <- OPENING only
  opening-stock-voucher.module.ts
  opening-stock-voucher.controller.ts
  dto/save-opening-stock-voucher.dto.ts      <- extends the shared DTO, pins the type
  dto/list-opening-stock-voucher-query.dto.ts
  dto/opening-stock-voucher-response.dto.ts
```

**Why the split.** `stock_voucher` already serves OPENING, RECEIPT, ISSUE,
ADJUSTMENT, TRANSFER_*, DAMAGE, EXPIRY_WRITEOFF, PHYSICAL and REPACK_*, and five
more screens are planned against it. Written as one `OpeningStockService` it gets
copy-pasted five times and the fifth copy disagrees with the first. The service
takes `voucherType` and a small per-type rule record; the controller pins the
type so no payload can ever change it.

**The type is never in the payload.** `@Controller('stock/opening')` sets
`'OPENING'`. A save DTO carrying `voucherType: 'TRANSFER_OUT'` must be rejected
by `@IsIn(['OPENING'])`, not silently honoured — a transfer posted through this
route would leave a `stock_transit` row uncreated.

House controller shape: `@ApiTags` + `@ApiBearerAuth('access-token')` +
`@UseFilters(<module>ExceptionFilter)` on the class, `@Version('1')` on every
method, and the envelope `{ success: true, message, data }` on every response.
Routes resolve under the global `v1` versioning and auth guard as
`/api/v1/stock/opening…`.

---

## 3. Numbering — the device is the counter

```
svh_slno   bigint, unique per (company, branch, acc_year, voucher_type, DEVICE)
svh_refno  varchar(100), unique per (company, branch, acc_year)
```

Observed format, from the flow doc: `OPN/2026-2027/TILL-01/1`
— `{typeCode}/{accYear}/{deviceCode}/{slno}`.

**Do not reuse `SequenceService` / `accounts.acc_voucher_seq`, and do not create
an `acc_voucher_header` row.** The sequence table is keyed by `seq_vchr_type_id`,
an FK to `accounts.acc_voucher_type`; a stock voucher type is not an accounting
voucher type, and inventing rows in the accounting voucher-type master to satisfy
a foreign key is how the two masters start disagreeing. A stock voucher is not an
accounting document: it moves quantity and cost, it posts no debit and no credit,
and `stock_voucher` has no column pointing at one.

Numbering is therefore self-contained, in
`stock-voucher-numbering.helper.ts`: a transaction-scoped `pg_advisory_xact_lock`
over a hash of `(company, branch, acc_year, voucher_type, device)`, namespace
`'stock.stock_voucher.slno'`, then `MAX(svh_slno) + 1` within that same scope —
all inside the caller's `$transaction`, so the lock releases with it.

**Accept a client-supplied `slno`/`refno` when present.** The whole reason the
serial is per-device is that a warehouse tablet must number its own document
offline and sync later. When both are absent, generate. When they collide,
`ux_svh_slno` / `ux_svh_refno` raise `23505` → **409**, naming the refno. Do not
"fix" a collision by renumbering: the device's copy is already printed.

---

## 4. `POST /api/v1/stock/opening` — save a draft

Create when `svhId` is absent, update when present. Update is **full replace of
the lines** — merge-by-row-number over a grid the user can insert into is where
line numbers drift, and a DRAFT has no history worth preserving.

### 4.1 Payload

```ts
{
  header: {
    svhId?, accYear, companyId, branchId, tenantId?,
    deviceId, sessionId?,
    slno?, refno?, usrRefno?,           // §3
    docDate,                            // 'yyyy-MM-dd'
    toGodownId,                         // REQUIRED for OPENING — §4.2
    rateSource?,                        // AVG_COST|LAST_PURCHASE|LOT_COST|MRP|MANUAL
    remarks?, userId
  },
  lines: [{
    lineNo, splitNo?,                   // splitNo defaults 1
    itemId, uomId, baseUomId,           // iuc_id — §4.3
    godownId, bucket?,                  // bucket defaults SALEABLE
    batchNo?, mfgDate?, expiryDate?, mrp?, salePrice?, serialNo?, supplierId?,
    qty, freeQty?,
    costRate, costRateWot?, taxPerc?,
    remarks?
  }]
}
```

### 4.2 What the service must refuse before the engine does

`fn_svh_post` raises on the first bad line and rolls the whole document back —
correct for the database, useless as a screen message. Everything catchable at
save time is caught at save time, as a **422 with a per-line list**:

- `toGodownId` missing → the engine raises `not_null_violation`
  ("an OPENING voucher must name svh_to_godown_id"); refuse it here instead.
  `ck_svh_godowns` will not catch it — an ISSUE satisfies that check with
  `from_godown` alone.
- `lines` empty → the engine raises at post; refuse at save (a draft with no
  lines is allowed to *exist*, but a post of it is not — see §7).
- `qty` and `freeQty` both 0 on a line.
- `svi_qty >= 0` etc. — negatives are `ck_svi_qty_sign`; a negative opening is
  an ADJUSTMENT, not an opening.
- `splitNo > 1` with a blank `batchNo` → `ck_svi_batch_split`.
- `expiryDate < mfgDate` → `ck_svi_expiry_order`.
- duplicate `(lineNo, splitNo)` in the payload → `ux_svi_line`, and it is a
  cleaner message from TS than from Postgres.

### 4.3 What the service computes and must not trust

| Column | Rule |
|---|---|
| `svi_to_base_factor` | read from `inventory.item_unit_conversion` for `(item, uom)`. Never from the payload — a wrong factor multiplies the opening quantity of the whole branch. |
| `svi_base_qty` | `qty × factor`, server-side |
| `svi_free_base_qty` | `freeQty × factor`, server-side |
| `svi_company_id/_branch_id/_acc_year/_tenant_id` | copied from the header, not the line |
| `svi_lot_id` | **always NULL on save.** `fn_slt_resolve` owns lot identity; a client-chosen lot on an OPENING would let two documents open the same holding under two lots. |
| `svi_cost_rate_wot` | leave 0 if not sent — the engine derives it from `svi_tax_perc` at post and writes it back (20 ÷ 1.05 = 19.047619 in the flow doc). |
| `svh_status` | `'DRAFT'`, always. Post is §7, not a status field. |
| totals | never (§1.4) |

**`svi_uom_id` and `svi_base_uom_id` are `inventory.item_unit_conversion.iuc_id`,
not `item_unit_master.unit_id`.** The schema warns about this twice and the FK
will catch it — but only after the user has typed forty lines. Validate at the
DTO boundary that the `iuc_id` belongs to `svi_item_id` (`iuc_item_id` match);
Q3 reports it as *"the unit does not belong to this item"*.

### 4.4 Editing a posted document

`tr_svh_post_lock` and `tr_svi_post_lock` refuse every edit once
`svh_status <> 'DRAFT'`, and `tr_sml_immutable` refuses every UPDATE and DELETE
of the ledger. Check the status in the service and return **409** with the
status in the message; do not let it arrive as a 500 from a trigger.

The only things a posted opening accepts are cancel (§8) and print.

---

## 5. `GET /api/v1/stock/opening` — list and load

`?svhId=&accYear=` → one document; otherwise the list. Both are
`19q_opening_stock_queries.sql` verbatim, as parameterised `$queryRaw`.

- **List = Q1**, scoped by company + branch + acc_year, `svh_voucher_type =
  'OPENING'`, `svh_is_deleted = false`, ordered `svh_doc_date DESC, svh_slno DESC`
  (index `ix_svh_list`). It reads the trigger-maintained counters, so the list
  never aggregates lines.
- **One = the header row + Q2** for the lines (index `ux_svi_line`).

Not a configured grid — there is no grid id for these, so they do **not** go
through `ConfiguredGridSqlService`.

Response keys are camelCase and are contract for the Qt screen:

```
header: svhId · accYear · refno · usrRefno · docDate · godownId · godownName
        status · lineCount · totalQty · totalValue · totalValueWot
        postedOn · postedByName · rateSource · remarks
lines:  sviId · lineNo · splitNo · itemId · itemCode · itemName · unitName
        uomId · baseUomId · toBaseFactor · godownId · godownName · bucket
        batchNo · mfgDate · expiryDate · mrp · salePrice · serialNo · supplierId
        qty · baseQty · freeQty · freeBaseQty
        costRate · costRateWot · taxPerc · value · valueWot · lotId · remarks
```

`lotId` is **NULL while DRAFT and filled by the post**. That is not missing data;
it means "not posted yet", and the screen may render it as the tick that says
this line reached the ledger.

`numeric` comes back as Prisma `Decimal` — return JS numbers. Dates as ISO
`yyyy-MM-dd` (`NexJson::date` reads the first ten characters).

---

## 6. `GET /api/v1/stock/opening/validate` — the preflight

`?svhId=&accYear=` → **Q3**, unchanged.

This is the query that makes the screen usable. It resolves the lot exactly the
way `fn_slt_resolve` would — blanking whatever the effective policy does not
track, then matching on `stock_lot`'s **generated key columns** — *without
creating anything*, which is the only way to answer "has this holding already
been opened" before the lot exists.

Return every line, `problem` null on the clean ones; the screen shows the failures
and ticks the rest. The wording is already in the SQL and is the same wording the
engine raises — do not paraphrase it in TypeScript.

Nine problems it detects: no quantity · unit does not belong to this item ·
batch/expiry/MRP/serial blank on a tracked item · inward with no cost rate and no
rate source · rate source `AVG_COST` with no average cost yet · **this holding
already has an opening in this year**.

The screen calls this on demand and again immediately before Post. It is advisory
— §7 still has to handle the engine raising, because another till can post the
same holding between the check and the call.

---

## 7. `POST /api/v1/stock/opening/post` — the whole engine, one statement

```ts
await tx.$queryRaw`SELECT stock.fn_svh_post(${svhId}::uuid, ${accYear}::bpchar, ${userId}::uuid) AS rows`;
```

Returns the number of ledger rows written. Inside a `$transaction`; nothing else
belongs in that transaction.

**What the service must not do around it.** No lock, no pre-UPDATE of the status,
no ledger insert, no balance touch. `fn_svh_post` takes `FOR UPDATE` on the header
itself, so two simultaneous posts are already serialised and the loser gets
*"voucher X is POSTED; only a DRAFT voucher can be posted"* → 409. An application
lock on top of that buys nothing and can deadlock against the function's own.

What happens inside, in order — it is worth knowing when reading a stack trace:
`fn_slt_resolve` creates the lots → `stock_ledger` rows are inserted →
`tr_sml_apply` builds `stock_balance`, `slt_total_on_hand` and the moving average
in `stock_item_cost` → the lines get their `lot_id` and the resolved rates written
back → `fn_svh_recompute` → status `POSTED`.

Call **Q3 first** and return its problems as 422 without calling the function when
any row has one. Post is the button users press forty lines in.

Response: `{ rowsPosted, status: 'POSTED', postedOn, header, lines }` — reload and
return the document, because the post changed `lotId`, `costRateWot` and the
totals on rows the client is still holding.

---

## 8. `POST /api/v1/stock/opening/cancel`

```ts
await tx.$queryRaw`SELECT stock.fn_svh_cancel(${svhId}::uuid, ${accYear}::bpchar, ${reason}, ${userId}::uuid) AS rows`;
```

Reversal rows, never a delete. Two properties to hold on to:

- A reversal carries the **original** `doc_date` and `acc_year`, so an
  as-on-date report reads "this document never moved stock". `sml_posted_on` is
  the audit trail of when the cancellation happened.
- **It can legitimately fail.** Cancelling an opening after stock has been sold
  from it drives the holding negative, and `fn_sml_apply` refuses that under
  `stp_allow_negative = 'BLOCK'`. That is correct behaviour, not a bug to route
  around: surface it as **409** with the item named. The fix is an ADJUSTMENT
  with a reason.

`reason` is required by the API even though `svh_cancel_reason` is nullable — a
cancelled opening with no reason is unanswerable three months later.

---

## 9. `DELETE /api/v1/stock/opening`

`?svhId=&accYear=` → soft delete, `svh_is_deleted = true`, **DRAFT only**.

A POSTED voucher is cancelled, never deleted: `svh_is_deleted` is not read by
`fn_svh_cancel`'s ledger scan, and `fn_svh_post` refuses a deleted voucher — so
soft-deleting a posted document hides it from every list while its ledger rows
keep affecting stock forever. Return **409** and point at cancel.

Lines cascade on the FK, but soft delete is not a delete: set `svi_is_deleted`
on the lines in the same transaction, or `ix_svi_voucher` keeps serving them.

---

## 10. The two go-live reports

- `GET /api/v1/stock/opening/pending-items` → **Q4**. Every stockable item with
  no OPENING movement in this branch and year. On go-live day this is the work
  list; a week later it should be the items that genuinely started at zero.
- `GET /api/v1/stock/opening/reconcile` → **Q5**. What the branch started with,
  what it holds now, the difference. The opening figure comes from the **ledger**,
  not from the document, so a cancelled opening correctly reads as zero.

Both are raw, both scoped by company + branch + acc_year, both potentially large
— page them (`limit`/`offset`, default 200) rather than streaming a 40,000-row
item master into a Qt table.

---

## 11. Import from file

The mockup has an **Import from file** button. Scope it explicitly rather than
letting it arrive as an undocumented endpoint:

`POST /api/v1/stock/opening/import` — multipart CSV, columns matching §4.1's
line shape by item **code** and unit **name**, resolved server-side to
`item_id` / `iuc_id`. It creates or replaces the lines of an existing DRAFT and
returns the same per-line problem list as §6. It never posts.

Defer it to a second pass if the first release is tight — a 400-line opening
typed by hand is painful but correct, whereas a silent code-resolution mismatch
in an importer is 400 wrong lines.

---

## 12. Errors

Per-module exception filter, as every module here has. The engine raises with
deliberate SQLSTATEs; map them rather than letting a 500 through:

| SQLSTATE | Raised for | HTTP |
|---|---|---|
| `P0002` no_data_found | voucher / year not found | **404** |
| `23001` restrict_violation | deleted; not DRAFT; not POSTED | **409** |
| `23505` unique_violation | *"this holding already has an OPENING in this year"*; refno/slno collision | **409** |
| `23514` check_violation | line has no quantity; inward with no cost rate; voucher has no lines | **422** |
| `23502` not_null_violation | OPENING with no `svh_to_godown_id` | **422** |
| `0A000` feature_not_supported | TRANSFER_*/REPACK_* posted through this route | **409** |
| `23503` foreign_key_violation | bad item / uom / godown / device | **422** |
| negative-stock refusal on cancel | `fn_sml_apply` under `BLOCK` | **409** |

**The trap:** a raise from inside a function reaches Prisma as
`PrismaClientKnownRequestError` with **code `P2010`** — the real SQLSTATE is in
`error.meta.code` and the text in `error.meta.message`. A filter that switches on
`error.code` sees `P2010` for all eight rows above and returns one useless
message. Switch on `meta.code`, and pass the engine's own message through: it
already names the refno and the line number.

---

## 13. Wiring

- `StockVoucherModule` **exports** `StockVoucherService`; `OpeningStockVoucherModule`
  imports it. The next five screens import the same one.
- Both into `src/app.module.ts` imports; **`OpeningStockModule` comes out** in the
  same commit (§0.2). `PhysicalStockModule` is left alone here — it is the next
  screen's problem, and it will be replaced the same way, by a
  `PHYSICAL` controller over this service.
- `@ApiTags('Opening Stock')` on the new controller; check Swagger at
  `https://192.168.0.106:3011/api/docs` shows exactly one Opening Stock group
  after the old module is unregistered.
- This module imports nothing from `src/modules/stocks/opening-stock/`. If a
  helper there looks reusable it is because it solves the legacy table's problem,
  not this one.
- **No `@CacheTTL` anywhere in this module.** These are live balances.

---

## 14. Tests

Unit (`*.service.spec.ts`, mirroring the existing specs):

- `to_base_factor` is read from `item_unit_conversion`, not from the payload:
  send a deliberately wrong factor and assert the stored `svi_base_qty`.
- generated columns never appear in a create/update payload.
- header totals never appear in a create/update payload.
- `voucherType` other than `OPENING` → 400.
- update of a POSTED voucher → 409 without touching the database.
- `DELETE` on POSTED → 409.
- the SQLSTATE map, driven off `meta.code`, one case per row of §12.

Integration — **run `19_opening_stock_flow.md` end to end** on a throwaway PG 18
cluster and assert its captured figures, which is why they were captured:

> SALT 10 BOX @ 12/box @ 20.00 (untracked) + MILK 50 PIECE @ 28.00, batch B-2604
> exp 30/06/2026, 5 free.
>
> After save: `svh_line_count` 2, `svh_total_qty` **175.000000** (120 + 50 + 5 —
> free goods are stock), `svh_total_value` **3940.00**, status DRAFT,
> `stock_lot` / `stock_ledger` / `stock_balance` / `stock_item_cost` all **0 rows**
> — a draft moves no stock.
>
> After post: returns **2**; `stock_lot` 2 rows with SALT's key columns at the
> sentinels `('~', -1, -1, 0001-01-01, '~', nil-uuid)`; `stock_ledger` 2 rows,
> `sml_cost_rate_wot` **19.047619** on line 1 (nobody typed it); `stock_balance`
> MILK on-hand **55**, SALT **120**; `stock_item_cost` MILK avg 28.000000,
> `sic_total_value_wot` 1466.67; `svi_lot_id` now set on both lines.
>
> **`SELECT stock.fn_sbl_rebuild(...)` → 0 holdings differed**, after the post
> and again after a cancel.

Then the negative assertions, each of which must refuse cleanly:

1. post twice → 409 "is POSTED".
2. a second OPENING for the same holding → 409 "already has an OPENING".
3. edit a line after post → 409 from `tr_svi_post_lock`.
4. an inward at cost 0 with `rateSource` unset → 422.
5. cancel, then post again → 409 (status is CANCELLED, not DRAFT).

Concurrency: two parallel posts of the same voucher — one returns 2, one 409.

> `fn_sbl_rebuild` re-derives **quantities only**. It never recomputes
> `sbl_stock_value`, `sbl_avg_cost_rate` or `stock_item_cost`, so it can return 0
> while a valuation is wrong. Assert the value columns explicitly; there is no
> value-side assertion in the engine yet.

---

## 15. Verification

```bash
T="Authorization: Bearer $TOKEN"; B=https://192.168.0.106:3011/api/v1

curl -H "$T" "$B/stock/opening?companyId=$C&branchId=$BR&accYear=2026-2027"
curl -H "$T" -X POST "$B/stock/opening" -d @opening.json -H 'Content-Type: application/json'
curl -H "$T" "$B/stock/opening/validate?svhId=$V&accYear=2026-2027"
curl -H "$T" -X POST "$B/stock/opening/post" -d "{\"svhId\":\"$V\",\"accYear\":\"2026-2027\"}" -H 'Content-Type: application/json'
curl -H "$T" "$B/stock/opening/reconcile?companyId=$C&branchId=$BR&accYear=2026-2027"
```

Then, in psql, the only check that matters:

```sql
SELECT stock.fn_sbl_rebuild(:company_id, :branch_id);   -- must return 0
```

---

## Open items to close with the DB owner / client

1. **Deployment (§0.1).** Nothing exists on the live database yet, and
   `fn_create_stock_partitions` must be called for the year before the first row.
2. **Retiring `/api/v1/opening-stocks` (§0.2).** Confirm nothing outside the Qt
   client calls it, then unregister it in the same release. Two opening paths
   into one database is the one failure this plan cannot defend against.
3. **Sale prices on the opening line.** The legacy `osl_` line carried four
   sale-price levels; `svi_` deliberately does not — prices stay in
   `inventory.item_price_master` and `stock.stock_mrp_price` (16 §20). If the
   screen is to capture them during go-live, that is a **second** call to the
   pricing endpoints, not a column here. Confirm whether the client wants it in
   the same save.
4. **`svh_rate_source` on the screen.** The mockup has a Rate source control.
   `AVG_COST` and `LAST_PURCHASE` read `stock_item_cost`, which on a go-live day
   is empty — Q3 reports that, but the default should probably be `MANUAL`.
5. **Reason master.** `svh_reason_id` is unused by OPENING but required by
   ADJUSTMENT, which is the RELOT fix path and the thing users will reach for
   when an opening is wrong. Q17–Q20 in `16q` are its screens; not planned yet.
6. **REVALUE / REPACK** remain deliberate gaps in the engine. An opening entered
   at the wrong cost cannot currently be revalued — it is cancel and re-enter, and
   §8 says when that is refused.
