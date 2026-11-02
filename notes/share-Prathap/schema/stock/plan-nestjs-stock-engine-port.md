# NestJS — Stock engine: what is left, and moving the rest of the SQL into the service

Written 2026-09-26. Target repo: the NestJS server on 192.168.0.106
(`/home/vk/Dev/erp/ERP server/`, read here through the `nexerp-api-ro` snapshot).

**This is a NEW plan. It edits no existing plan.** It sits beside:

| existing plan | what this file does to it |
|---|---|
| `plan-nestjs-opening-stock.md` | nothing. Built. §1.9 adds the accounts posting it deliberately left out (its §3, "do not create an `acc_voucher_header` row", is **superseded for POSTED documents only**; drafts still never go there) |
| physical stock plan (api-ro `plan/plan-nestjs-physical-stock.md`) | nothing. Built. §1.2 and §1.9 add header recompute and accounts posting |
| `plan-nestjs-stock-transfer.md` | **§0.1 and every "call `fn_svh_post_transfer` / `fn_svh_receive_transfer`" are OVERRIDDEN by §1.1 here.** The DTOs, routes, validation, the transit worklist and the LR/vehicle write stay exactly as that plan says |
| `change_selling_screen_plan.md` + api-ro `plan-nestjs-change-selling-price.md` | nothing. §3 here is its blocker (the table and the resolver) |
| — | the adjustment family gets its own file: **`plan-nestjs-stock-adjustments.md`** (phase 2) |

**The spec for every former SQL function** is in `spec_prestrip/` beside this file. It holds the
2026-09-04 copies of 16/19/20 with their bodies intact. Every `16:NNNN`, `19:NNN`, `20:NNN` below is
a line in those files. They are `.txt` on purpose. **Nothing here is to be created in the
database.**

---

## 0. The rules this plan is written under

1. **No SQL function or trigger in the database, except the partition DDL helper.** The user
   said it flatly on 2026-09-22. Every behaviour below becomes TypeScript. GENERATED columns,
   CHECKs, unique and partial indexes, FKs and views stay: they are declarative and cannot run
   at the wrong moment.
2. **One stock engine.** Everything posts through `StockPostingService.post/cancel`
   (`src/modules/stocks/posting/stock-posting.service.ts`), which drives the seven phases in
   `stock-voucher-posting.helper.ts`. A new document writes a `StockLineSource`, never a second
   engine. `test/stock-ledger-single-writer.e2e-spec.ts` stays green: one INSERT site on
   `stock_ledger`, no UPDATE/DELETE.
3. **Derived numbers are recomputed from their source inside the same transaction**, never
   incremented. That covers balances, reserved qty, transit qty and header totals. An increment is
   right until one write is lost, and after that it is wrong forever.
4. **New routes get their own URL family** (house rule). No old route, DTO or DB function is
   reused.
5. **Offline.** The till posts locally and syncs by primary key
   (`cloud-sync`, `session_replication_role = replica`). So anything derived must be computable
   by the same TypeScript on the device, and the cloud never recomputes on push.

## 0.1 Where things stand (audited 2026-09-26, code + live DB)

| area | state | evidence |
|---|---|---|
| OPENING (8 routes) | ✅ built | `opening-stock-voucher.controller.ts` |
| PHYSICAL (7 routes, freeze on `sml_doc_datetime`) | ✅ built | `physical-stock-voucher.controller.ts`, `stock-posting.service.ts:105-169` |
| Lot resolve, policy precedence, ledger, balances, WAVG + restamp, negative policy, lot totals, cancel-by-mirror | ✅ built | helper `resolveLots` … `refreshLotTotals`, `cancelStockVoucher`; `test/stock-engine-ts.e2e-spec.ts` cases 1-10 |
| Sale bill / DC / sale return / DC return stock | ✅ built, via a generated ISSUE/RECEIPT voucher | `sales/posting/sales-stock.service.ts` |
| Sale-order reservations | ⚠ rows are written; `sbl_reserved_qty` never is | `stock-reservation.service.ts`; live: 28 OPEN, reserved = 0 everywhere |
| BUG 2026-09-09 (a cancelled opening held its holding) | ✅ **fixed** (not recorded in the bug file) | `stock-voucher.service.ts:1736` uses `unreversedLedgerRow()` |
| **Transfers** | ❌ **broken**: routes exist, posting calls absent SQL functions | `stock-transfer.controller.ts:87`, `stock-transfer-receive.controller.ts:83`, helper `usesInProcessPosting`; live `stock_transit` = 0 rows |
| Header recompute at post | ❌ totals come from the save payload | `stock-voucher.service.ts` `writeHeaderTotals` |
| Issue strategy (FIFO/FEFO/LIFO) in posting | ❌ stored, never applied | `stock-track-policy.service.ts:261` |
| Rate source LOT_COST / MRP | ❌ fall to 0 | helper `postingCte` `costed`; spec 19:413-415 |
| Sales generated-voucher numbering | ❌ `MAX+1` without a lock | `sales-stock.service.ts:378` |
| Stock → accounts (PERPETUAL) | ❌ no stock voucher posts a leg | §1.9 |
| Day-end "what is still unposted" | ⚠ `txn_status_log` has it, with three defects | §1.10 |
| Legacy read routes | ❌ read `inventory.item_stock_*` (0 rows, never written) | `itemstockBalanceService.ts:52,109,256`, `itemBatchStockService.ts:49`, `item-price.lookup.ts:344`, `stock-reconciliation.processor.ts:48,73` |
| Adjustment family, REVALUE, REPACK | ❌ | phase 2 / 6 |
| `stock_mrp_price` + resolver | ❌ table not deployed; gateway returns 503 | `stock-mrp-price.gateway.ts:95-125` |
| As-of, ageing, snapshot, demand, rebuild assertion | ❌ | phases 4-5 |
| Purchase | ❌ no module | phase 6 |

---

## PHASE 1 — Make what exists correct

### 1.1 Transfers move into the TypeScript engine

**Spec:** `fn_svh_post_transfer` 20:87-315, `fn_svh_receive_transfer` 20:316-591,
`fn_svh_transfer_cancel_guard` 20:592-625, plus the worked example `20_stock_transfer_flow.md`.

Remove `postFunction` and `usesInProcessPosting` altogether. Every stock voucher type then posts
in-process, and the type's rules record says *how* through a `postShape`:
`'SIMPLE' | 'COUNT' | 'TRANSFER_OUT' | 'TRANSFER_IN'`.

**Despatch (`POST /stock/transfer/despatch`)**, inside one `$transaction`:

1. Lock the header `FOR UPDATE`, status must be DRAFT, and it must not be deleted.
2. `same := toBranchId IS NULL OR toBranchId = branchId` (20:~110).
3. Every line must carry `svi_lot_id`. The lot moves; it is never re-resolved. Cost must be 0
   on the line. The engine stamps it from the policy (this is REVIEW MUST-FIX 5, already coded
   around in todo 1.3; keep that).
4. **Same branch:** write the OUT rows (`TRANSFER_OUT`, −1, from-godown) and read their stamped
   `sml_cost_rate` back with `RETURNING`. Then write the IN rows (`TRANSFER_IN`, +1, to-godown)
   **at that cost**. Run phases 4-7 once over both sets. The header goes to POSTED.
5. **Other branch:** write the OUT rows only, plus one `stock_transit` row per
   (item, lot, dest godown) carrying `stt_cost_rate` from step 4's `RETURNING`, LR, vehicle and
   expected date. The header goes to IN_TRANSIT. Then run §1.3's `refreshTransitIn` for the
   destination holdings.
6. Write a `txn_status_log` event (`STOCK_TRANSFER`, DRAFT→POSTED or DRAFT→IN_TRANSIT).

**Receive (`POST /stock/transfer/receive/post`):**

1. Lock the TRANSFER_IN and the linked OUT. The OUT must be IN_TRANSIT and not deleted. The
   "must not be deleted" part closes REVIEW hole 2 for soft-deleted OUTs.
2. For each line, match its transit row. The remaining quantity is
   `sent − received − damage`, and over-receipt is refused.
3. Write IN rows at `stt_cost_rate`, never the line's cost. A line with bucket `DAMAGED` lands
   in the DAMAGED bucket (20:~420).
4. Update the transit rows (`stt_received_qty`, `stt_damage_qty`, status PARTIAL/RECEIVED).
   Then run `refreshTransitIn`. The OUT becomes RECEIVED only when every row has nothing left
   (20:~540).

**Cancel.** This is the guard 20:592 ported into `cancel()`:
- a same-branch POSTED pair cancels normally, by mirroring both halves;
- an IN_TRANSIT or RECEIVED OUT is refused: goods on a lorry cannot be un-sent;
- a POSTED TRANSFER_IN is refused, because un-receiving is a reverse transfer;
- a DRAFT of either is deleted, as today.

**Short-settle, which is new. Nothing did it before (REVIEW known gap 1):**
`POST /stock/transfer/receive/settle-short {outVoucherId, accYear, reasonId, remarks}`.
- Allowed only on an IN_TRANSIT OUT whose rows are all RECEIVED or PARTIAL.
- It flips each PARTIAL row to RECEIVED and **keeps `stt_short_qty`**, so the loss report still
  shows it. Then it sets the OUT to RECEIVED and runs `refreshTransitIn`.
- The reason is required (seed a `TRANSIT_LOSS` reason, direction OUT).
- Under PERPETUAL it posts **DR the reason's ledger / CR INVENTORY** for
  `short_qty × stt_cost_rate`. The goods left branch A's stock at the OUT and never arrived at
  B, so without this entry the Stock-in-Hand ledger keeps a value no stock stands behind.

**Column needed: `stt_cost_rate_wot`.** This is a Prisma migration, not SQL logic. Receive
currently takes the without-tax cost from the lot's first-arrival rate (20:508), while the
with-tax cost travels. Stamp both at despatch.

**Tests:** un-skip `stock-transfer(.http).e2e-spec.ts` (drop the `STOCK_ENGINE_REQUIRED`
switch). Add these cases:
- same-branch pair: average unchanged, both godowns right;
- inter-branch: transit row, destination `sbl_transit_in_qty`;
- partial receive, then over-receive refused;
- DAMAGED bucket on receive;
- settle-short: transit qty back to 0, loss visible, the accounts leg;
- mid-flight cancel refused;
- rebuild assertion (§5.1) returns 0 after each.

### 1.2 Header totals are recomputed at post

**Spec:** `fn_svh_recompute` 16:3532-3606. It was called by `tr_svi_refresh_header` 16:3607-3643.

At the end of `post()` and of `cancel()`, and never from the save payload:
- `svh_line_count`, `svh_total_qty`, `svh_total_value`, `svh_total_value_wot` come from the
  **ledger rows the document wrote**, at their stamped `sml_cost_value`, reversals included. A
  cancelled document therefore re-totals to 0, which is REVIEW SHOULD-FIX "cancelled PHYSICAL
  header recompute".
- **PHYSICAL** gets the net variance: the sum of the signed qty and signed value of its
  PLUS/MINUS rows (README:189-193; the worked figures are in `19_physical_stock_flow.md`, net
  +1 / 86.00).
- A draft keeps the payload's figures, flagged provisional. That matches today's behaviour.

Delete the out-of-date DTO comments that say `fn_svh_recompute` overwrites the totals.

### 1.3 Reserved and in-transit quantities on the balance

**Spec:** `fn_sbl_ensure_row` 16:3347, `fn_sbl_refresh_reserved` 16:3390, `fn_srv_refresh`
16:3432, `fn_stt_refresh` 16:3465.

Add two private methods to the one balance writer, the helper that owns `applyBalances`:

- `ensureBalanceRow(tx, holdingKeys[])` inserts a zero row `ON CONFLICT DO NOTHING`, so that a
  reservation or transit can exist before the holding's first movement.
- `refreshReserved(tx, holdingKeys[])` sets
  `sbl_reserved_qty = SUM(open reservations for that holding)` (**recompute, not increment**).
- `refreshTransitIn(tx, destHoldingKeys[])` sets
  `sbl_transit_in_qty = SUM(sent − received − damage)` over non-RECEIVED transit rows.

Callers:
- `StockReservationService` reserve, consume and release call `refreshReserved`;
- despatch, receive and settle-short call `refreshTransitIn`.

Take the per-(company, branch, item) advisory lock in sorted order first. It is the same lock
`applyItemCost` takes (helper ~:764), so a reservation and a posting on one item never
deadlock (REVIEW MUST-FIX 1's lesson).

**One-time data fix**, in a migration: recompute `sbl_reserved_qty` for every holding from
`stock_reservation`. There are 28 OPEN reservations live and the column reads 0.

### 1.4 Issue strategy on an outward line with no lot

**Today:** an outward line finds its lot only from its own identity keys or a supplied
`lotId`. `stp_issue_strategy` is written by the policy service and read by nobody except the
sale-order reservation walk.

**Rule to implement.** When an outward line of a lot-tracked item arrives without a lot, pick
lots in the from-godown by the effective policy's `stp_issue_strategy` (FEFO → expiry ASC,
FIFO → `sbl_first_in_date` ASC (first into THIS godown), LIFO → DESC; `MANUAL` → refuse, the client must name the lot; fallback FEFO per README:171). Take from lots with
available > 0 and **split the line across lots** using `svi_split_no`, which exists for exactly
this. An untracked item has one lot and needs no pick.

Which documents this affects:
- Opening and PHYSICAL never pick: their lines carry identity.
- Transfers never pick: the lot is mandatory.
- The sale bill already sends the lot it picked.

So in practice this is for the adjustment family's outward lines and for a bill line that
arrives lotless from the offline till. **Open decision D1:** pick at post, or refuse a lotless
outward line and make the client pick. Picking at post is what the offline till needs.

### 1.5 Rate source LOT_COST and MRP

19:413-415: `LOT_COST` = the resolved lot's `slt_cost_rate`, and `MRP` = the line's `svi_mrp`.
Add both to the `costed` CTE. Validation already treats them as derivable
(`DERIVABLE_RATE_SOURCES`, types.ts:78), so today they produce a zero-valued inward. After the
resolution, a zero cost on an inward still refuses (19's rule, free-goods lines exempt).

### 1.6 Lock the numbering of the generated sales vouchers

`sales-stock.service.ts:378` does `MAX(svh_slno)+1` with no lock, so two concurrent bills on
one device can collide on `ux_svh_slno`. Call `stock-voucher-numbering.helper.ts`'s allocator
(advisory lock on company, branch, year, type, device) instead of the inline query.

### 1.7 Point the four legacy read paths at the engine tables

`inventory.item_stock_ledger`, `item_stock_balance` and `item_batch_stock` have 0 rows and no
writer. So everything that reads them answers "no stock":

| path | change to |
|---|---|
| `GET /item-stock-balance/get`, `/bulk-list`, `/batch-options` | read `stock.stock_balance` joined to `stock_lot` (available = `sbl_available_qty`, which is GENERATED on-hand − reserved, so §1.3 makes it right) |
| `GET /item-batch-stock/get` | same, one row per lot |
| `GET /master-lookup/item-price`, stock qty (`item-price.lookup.ts:344`) | `SUM(sbl_on_hand_qty)` for the item, branch and godown |
| BullMQ `stock-reconciliation` processor | **replace** with §5.1 |

Keep the response shapes (the Qt and React clients read them). Mark the old tables for the drop
list, and don't drop them in this plan.

### 1.8 Remove the out-of-date comments

- helper:26-32 says the DB owns the append-only and freeze triggers;
- the `stockLedger.prisma` header mentions a delete trigger;
- the stock-voucher README says the ledger has no Prisma model.

All three are false since 2026-09-22.

### 1.9 Stock → accounts posting (PERPETUAL)

**Why.** `accounts.cogs_mode` = PERPETUAL, and the sale bill, DC and returns already write
**DR COGS / CR INVENTORY** (`sales-leg.sources.ts` `buildCogsLegs`). Every sale credits the
Stock-in-Hand ledger. Nothing in the stock module ever debits it, so the ledger runs negative
and never equals the stock valuation.

**The rule.** When a stock document POSTS and `cogs_mode = PERPETUAL`, it writes one accounts
voucher through **`VoucherPostingService.postLegs`**
(`src/common/posting/voucher-posting.service.ts`) inside the same transaction. Its cancel calls
`reverseLegs` (the live `Rev` pattern, dated the original).
- **Drafts never reach `acc_voucher_header`.** Day-end pending is §1.10's job, not this table's.
- Under PERIODIC, no stock voucher posts a leg.

**The link, with no new column:** `avh_src_module = 'STOCK'`,
`avh_src_doc_type = svh_voucher_type`, `avh_src_doc_id = svh_id`, and
`avh_doc_refno = svh_refno`, the same way sales links. `avh_voucher_date = svh_doc_date`.

**The amount always comes from the ledger rows the post just wrote** (`sml_cost_value`), never
from the payload. That way accounts and stock cannot disagree.

| document | legs (PERPETUAL) |
|---|---|
| OPENING | DR `INVENTORY` / CR `OPENING_DIFFERENCE`, value = Σ OPENING rows |
| PHYSICAL | shortage: DR reason ledger (default role `STOCK_SHORTAGE`) / CR `INVENTORY`; excess: DR `INVENTORY` / CR reason ledger (default role `STOCK_EXCESS`). One voucher, both pairs, netted per ledger |
| adjustment family | see `plan-nestjs-stock-adjustments.md` §5 |
| TRANSFER same branch or same company | **none**: one company-level Stock-in-Hand ledger (decided 2026-09-26) |
| transfer settle-short | DR reason ledger / CR `INVENTORY` (§1.1) |
| generated ISSUE/RECEIPT behind a sales document | **none**: the sales document posts its own COGS pair. Posting here too would count it twice |
| inter-state branch transfer | out of scope: that is a GST supply, done later through a branch-transfer invoice |

**Which ledger:**
1. the line's reason, then the header's reason: `stock_reason_master.srm_gl_ledger_id`, when set;
2. otherwise the role, through `acc_ledger_map`.

The resolution sits in `VoucherPostingService.resolveLegLedgers`, as for every other leg.

**What the database needs (a migration, data only):**
- `acc_ledger_role`: add `STOCK_SHORTAGE` (EXPENSE) and `STOCK_EXCESS` (INCOME);
- `acc_ledger_map`: seed both, like `26_ledger_map_seed.sql` did;
- `acc_voucher_types`: set `vchr_affects_accounts = true` on type 1 OPENING and type 6 Phy.
  They keep their own numbering (`refnoVchrTypeId` already points at them).

**The Opening Balance screen (menu 55) must refuse** an opening balance on the ledger mapped to
`INVENTORY` while `cogs_mode = PERPETUAL`. Otherwise the value is counted twice: once from the
balance screen, once from the opening stock. This is a rule in the opening-balance service.

**Go-live step, not a test-box task:** stock documents posted before this lands have no voucher.
Add a one-time admin route, `POST /stock/admin/post-missing-vouchers?companyId&accYear`. It walks
POSTED stock vouchers with no `acc_voucher_header` row for (`STOCK`, `svh_id`) and posts them.
It is idempotent because it reads first. The rows on .106 today are test data. Run it there only
to test the route.

**Tests:**
- post an opening: the INVENTORY ledger is debited by exactly Σ `sml_cost_value`;
- cancel it: the `Rev` voucher nets to 0;
- a PHYSICAL with one short and one over line posts one voucher with two pairs;
- a sale bill after an opening leaves INVENTORY = the stock valuation (§4.1 as-of) to the paisa;
- PERIODIC posts nothing;
- a generated sales ISSUE posts nothing of its own.

### 1.10 Day end: "is anything still unposted?"

**The table is `public.txn_status_log`**, which every module already writes. The pending
documents are the **latest event per `tsl_src_doc_id`** whose `tsl_to_status` is DRAFT, HELD or
CONFIRMED (IN_TRANSIT too, once §1.1 lands).

New route: `GET /api/v1/txn-status/pending?companyId&branchId&accYear&upToDate&srcModule?`. It
returns one row per pending document (module, doc type, refno, doc date, status, since,
changed_by), plus counts per module and doc type for the day-close screen's summary.

**Three defects to fix first, or the answer is wrong:**
1. **Order by `tsl_seq_no`, never by `tsl_changed_on`.** On create+post the CREATED row is
   stamped about 50 ms after the POSTED row (seen live: seq 2 POSTED 15:29:20.489, seq 1
   CREATED 15:29:20.543). Sorting by time calls 109 posted openings "draft". Also fix the
   writer so both events take one timestamp, read once per request.
2. **Sale-order drafts are never logged.** 41 draft orders are in the table and none are in the
   log. Log `CREATED` at order create, the way every other document does.
3. **Vague `OTHER` doc type.** DC return (`sales-stock.service.ts:505`,
   `dc-return.service.ts:82`) and cheques (`cheques.utils.ts:60`,
   `issued-cheques.service.ts:120`) log as `OTHER`. Add `DC_RETURN`, `CHEQUE_RECEIVED` and
   `CHEQUE_ISSUED` to `ck_tsl_src_doc_type` and to `TxnStatusDocType`, and re-label the existing
   rows in the same migration.

Index: `(tsl_company_id, tsl_branch_id, tsl_acc_year, tsl_src_doc_id, tsl_seq_no DESC)` on the
partitioned table, for the latest-per-doc scan.

The generated sales stock vouchers are not logged, which is correct: they belong to their sales
document.

---

## PHASE 2 — Adjustment family

Its own file: **`plan-nestjs-stock-adjustments.md`**. It needs §1.4 (pick) and §1.9 (legs) from
phase 1.

## PHASE 3 — Pricing: the MRP-bucket price table and its resolver

**Spec:** 16 §20 (16:4303 on), `fn_smp_effective` 16:4612; `PRICING_AND_FORMS_PLAN.md`:20-29;
`16p_stock_pricing.sql.bak` for the pre-chain-row version.

1. **Prisma migration** for `stock.stock_mrp_price`, from the restored `16_stock.sql` §20:
   - nullable `smp_company_id` / `smp_branch_id` (NULL = chain-wide);
   - `smp_effective_from`/`to` defaulting 1900/9999;
   - the `ex_smp_overlap` EXCLUDE constraint, which Prisma cannot express, so it goes as raw SQL
     in the migration;
   - `ck_smp_not_above_mrp`.

   This is DDL, not logic, so it is allowed.
2. **`PriceResolverService.effective(company, branch, item, uom, mrp, sp, onDate)`** ports
   `fn_smp_effective`:
   - a BUCKET row beats MASTER (`item_price_master`);
   - a branch row beats a chain row, and a company row beats a shared one;
   - the dated row in force on `onDate` wins;
   - it returns `bucket_id` and `bucket_scope` (BRANCH/CHAIN/NULL).

   It must be **pure TypeScript over rows**, because the offline till needs the same answer.
3. Swap the stub `stock-mrp-price.gateway.ts` for it, then work down the change-selling todo
   (§1-§12: Q24-Q27, S1-S3 find-then-write, the below-cost confirm round-trip).

## PHASE 4 — Read helpers and reports

The seven share queries that call dead functions are rewritten as TypeScript services. They are
reads, so any file may call them.

| former function | NestJS home | used by |
|---|---|---|
| `fn_stp_effective` 16:2608 | exists as `effectivePolicyLateral` (helper:237-276). Keep it and export it as the ONE policy resolver | 16q Q1/Q7/Q23, 19q:60/63/245/275 |
| `fn_stock_asof` 16:3843 | `StockAsOfService.asOf(company, branch, date, item?, godown?)`: nearest snapshot (§5.3), plus the ledger delta since. With no snapshot, a full replay | 16q Q8, §1.9 test, year-end |
| `fn_sag_effective` 16:4036 | `AgeingLadderService.effective(company)`: all-or-nothing, a company's own ladder or the shared one, never mixed | 16q Q21/Q22 |
| `fn_sag_coverage` + `tr_sag_coverage` 16:3938/4016 | `AgeingLadderService.saveLadder(company, slabs[])`: **whole-ladder** replace in one transaction, validated before commit (starts at 0, no gaps, exactly one open-ended slab and it is last) | ladder editor |
| `fn_stock_ageing` 16:4079 | `StockAgeingReport`, by `stp_ageing_basis` (INWARD_DATE / LAST_IN_DATE / MFG_DATE) | 16q Q6 |
| `fn_smp_effective` | §3 | 16q Q23-Q25 |
| `19m_migrate_opening.sql` → `fn_svh_post` | not needed: the legacy tables are empty | — |

Routes: `GET /stock/reports/as-of`, `/ageing`, `/valuation`, `/near-expiry`, `/negative`,
`/item-ledger`, `/transit` (16q Q1-Q16, one route each as the screens are built; not all at
once). Ageing ladder: `GET /stock/ageing-ladder`, `PUT /stock/ageing-ladder` (whole ladder).

**Valuation as of a date must include goods in transit** (REVIEW known gap 3): an extra line
from `stock_transit` rows not RECEIVED as of the date, at `stt_cost_rate`. Year-end needs it,
because those goods are in neither branch's balance.

## PHASE 5 — Scheduled jobs (BullMQ, beside the existing processors)

| job | spec | schedule | notes |
|---|---|---|---|
| **5.1 Balance assertion** (replaces `stock-reconciliation.processor`) | `fn_sbl_rebuild` 16:3644; contract `17_sml_apply_contract.md` assertions 1-6 (:366-381) | nightly per company | **Detect and report, never silently fix.** Compare `stock_balance` to the ledger sum per holding (qty and value), `stock_item_cost` to the sum of balances, reserved and transit to their sources, and `slt_total_on_hand` to its balances. Write findings to a report table or log. Repair is a separate admin route with a dry-run flag, and it must report orphans too (REVIEW: the old rebuild could not zero them) |
| **5.2 Demand daily** | `fn_sdd_rebuild` 16:3768 | nightly, yesterday | **Fix MUST-FIX 4:** exclude reversed rows (`unreversedLedgerRow()`), since a cancelled sale of 30 counted as 60. Fill `sdd_promo_qty` from promotion usage (known gap 4). Stockout flag from the balance walk-back, valid only when run nightly |
| **5.3 Valuation snapshot** | never written; the columns are in 16 `stock_valuation_snap` (16:1988) | month-end + year-end | freezes qty, value, MRP/SP valuation and age days per holding, plus the in-transit line (§4). Needed **before the first year-end** |
| **5.4 Reservation expiry** | implied by `srv_expires_on` (16:1745, table at 16:1714) | hourly | expire OPEN rows past `srv_expires_on`, then `refreshReserved` |
| **5.5 Lot expiry** | "somebody decides" (17:330) | nightly | set `slt_status = EXPIRED` for lots past expiry with on-hand > 0; enforce `stp_block_expired_sale` at the sale-bill validate |

## PHASE 6 — Deferred, listed so they are not lost

- **REVALUE:** a voucher type that changes value, not qty. It needs a design (per-item new rate
  → a restamp of `stock_item_cost` + DR/CR INVENTORY vs a revaluation role).
- **REPACK_IN/OUT:** now required by D55 auto unit-break (`plan-backend-sales.md`:413-416). It
  is refused by name today.
- **Purchase posting:** no module. Under PERPETUAL it posts DR `INVENTORY`, not DR `PURCHASE`
  (sales analysis §9a).
- **Forecast and movement class** (`stock_forecast`, `stock_movement_class`): an external job,
  deliberately not SQL. Report-only UI.
- **Reorder policy (`srp`) bulk screen:** late phase.
- **LOT_ACTUAL/FIFO issuing at the lot's first-arrival rate for ever** (16:3176): flagged, not
  decided.

---

## Appendix A — every former SQL object, and where it lives now

| # | object (spec line) | status | NestJS home |
|---|---|---|---|
| 1 | `fn_create_stock_partitions` 16:184 | **stays in DB** (DDL rule) | — |
| 2 | `fn_stp_effective` 16:2608 | ✅ ported | `effectivePolicyLateral` |
| 3 | `fn_slt_resolve` 16:2702 | ✅ ported | helper `resolveLots` |
| 4 | `fn_sml_apply` + trigger 16:2823/3133 | ✅ ported | helper phases 4-7 |
| 5 | `fn_sml_cost_default` + trigger 16:3157/3208 | ⚠ LOT_COST/MRP missing | §1.5 |
| 6 | `fn_sml_immutable` + trigger 16:3221/3249 | ✅ replaced by the single-writer test | — |
| 7 | `fn_sml_freeze_guard` + trigger 16:3288/3327 | ✅ ported (on doc datetime) | `assertNotFrozen` |
| 8 | `fn_sbl_ensure_row` 16:3347 | ❌ | §1.3 |
| 9 | `fn_sbl_refresh_reserved` 16:3390 | ❌ | §1.3 |
| 10 | `fn_srv_refresh` + trigger 16:3432/3460 | ❌ | §1.3 |
| 11 | `fn_stt_refresh` + trigger 16:3465/3524 | ❌ | §1.3 |
| 12 | `fn_svh_recompute` 16:3532 | ❌ | §1.2 |
| 13 | `fn_svi_refresh_header` + trigger 16:3607/3629 | ❌ | §1.2 (called once at post, not per line) |
| 14 | `fn_sbl_rebuild` 16:3644 | ❌ | §5.1 |
| 15 | `fn_sdd_rebuild` 16:3768 | ❌ | §5.2 |
| 16 | `fn_stock_asof` 16:3843 | ❌ | §4 |
| 17 | `fn_sag_coverage` + constraint trigger 16:3938/4016 | ❌ | §4 `saveLadder` |
| 18 | `fn_sag_effective` 16:4036 | ❌ | §4 |
| 19 | `fn_stock_ageing` 16:4079 | ❌ | §4 |
| 20 | `fn_smp_effective` 16:4612 | ❌ | §3 |
| 21 | `fn_svh_txn_map` 19:93 | ⚠ OPENING/PHYSICAL only | adjustments plan §2 |
| 22 | `fn_svh_post` 19:207 | ⚠ ported for OPENING/PHYSICAL/ISSUE/RECEIPT | helper + adjustments plan |
| 23 | `fn_svh_cancel` 19:586 | ✅ ported | `cancelStockVoucher` |
| 24 | `fn_svi_post_lock` + trigger 19:699/737 | ✅ service discipline (lines are written only while the header is DRAFT) | keep the check in `save()` |
| 25 | `fn_svh_post_lock` + trigger 19:742/798 | ✅ service discipline; **add** IN_TRANSIT/RECEIVED to the "frozen" set (REVIEW hole 2a) | §1.1 |
| 26 | `fn_svh_post_transfer` 20:87 | ❌ | §1.1 |
| 27 | `fn_svh_receive_transfer` 20:316 | ❌ | §1.1 |
| 28 | `fn_svh_transfer_cancel_guard` + trigger 20:592/619 | ❌ | §1.1 cancel |
| — | `fn_sml_forbid_delete`, `tr_sml_forbid_truncate` (migration-only, never in a share file) | ✅ dropped, replaced by the single-writer test | — |
| — | `fn_svs_take_snapshot` (never written) | ❌ | §5.3 |
| — | `fn_stt_settle_short` (never written) | ❌ | §1.1 settle-short |

## Appendix B — open decisions

| # | question | default if nobody answers |
|---|---|---|
| D1 | lotless outward line: pick at post by issue strategy, or refuse? | pick at post (the offline till needs it) |
| D2 | does a transfer force BLOCK on over-issue regardless of the item's negative policy? (transfer plan open item 4) | yes: a lorry cannot carry negative stock |
| D3 | multi-destination despatch | one OUT per destination (transfer plan item 6) |
| D4 | refno prefixes TRF / TRI | agree with the client screens |
| D5 | offline refno: accept the device's slno/refno (opening plan §3), or assign at sync (2026-09-05 offline audit)? | accept the device's; `svh_slno` is per device so it cannot collide |
| D6 | the snapshot table's grain and retention | per holding, monthly, kept for ever |
