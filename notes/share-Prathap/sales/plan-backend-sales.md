# Sales transactions — backend plan (NestJS), module by module, endpoint by endpoint

**For Prathap. 2026-09-20, all defaults.** The spec is `sales_posting_flow.md`
(what is written); this is HOW, in this codebase — read against the
snapshot of `Erp Server` taken 2026-09-20 (`src/modules/sales/bill`,
`sale-order`, `quotation`, `txn-hold`; `src/modules/stocks/stock-voucher/
stock-voucher-posting.helper.ts`; `src/modules/accountsModule/receipt`;
`src/common/Sequence`, `src/common/txn-status-log`). File names below are
the ones there today; the new ones follow their pattern.

**SQL first:** `../29_sales_documents.sql` then `../30_sales_alters.sql`
(after 11, 13 and 15 §6 from `schema/`), then `prisma db pull`. Nothing
here compiles before that.

**The GST provider layer is its own plan:** `../gst/plan-backend-gsp.md`.
This plan only ENQUEUES work for it (§4.9) and READS its result.

---

## 1. What exists, what does not

| piece | today | plan |
|---|---|---|
| `bill.controller` | `/create` `/get` `/delete` | + `/post` `/cancel` `/amend` `/validate` `/open-sources` `/party-context`; `/delete` becomes DRAFT-only (§4.5) |
| `bill-posting.helper.ts` | `postBillToAccounts` writes ONE `acc_voucher_header` with no legs and an `acc_bill_balance` row at CREATE (`sales.auto_post`); `syncBillPosting` / `deleteBillPosting` mirror it | **retired**. A DRAFT writes no accounts row (§2 rule); POST writes everything, once |
| `bill.service.ts` | draft upsert; tenders via `TenderDetailService.syncDocumentTenders`; charges via the charge module; `txn_status_log` per step (line ~1532) | kept for the draft; the post path is a new service |
| `sale-order` | `/create` `/get` `/pending-amount` `/cancel-lines` `/delete`; advance posting complete | + `/post` (CONFIRMED + reservation) `/cancel` `/amend`; `/delete` DRAFT-only |
| `quotation` | `/create` `/get` `/delete` | unchanged (no post, Q22) |
| `txn-hold` | full | `DC_RETURN` doc type added; unchanged otherwise |
| stock engine | `postStockVoucher` — phases `resolveLots` → `attachLotsToLines` → `writeLedger` → `applyBalances` → `applyItemCost` → `assertNegativeStockPolicy` → `refreshLotTotals`, all module-private, all keyed on `stock_voucher(_item)` | phases 3–7 lifted into a shared `StockPostingService` with a line-source adapter (§3.1); the stock-voucher module becomes its first caller |
| receipt | the pattern: `receipt.guards.ts` (`assertAccYearWritable`, `assertVoucherPartitionExists`, `loadParty`, `lockBills`), `receipt-ledger-roles.ts` (`resolveRoleLedgers`), `allocateVoucherNumber`, `BillBalanceRecomputeService`, `appendTxnStatusLog`, `receipt-amend.service.ts` | **reused as is** — the sales services import them; nothing is copied |
| doc register | tables live, no writer | `DocRegisterService` (§3.3) |
| loyalty / promotion | scheme masters only | `LoyaltyLedgerService`, `PromotionUsageService` (§3.4, §3.5) |
| statutory | nothing | `StatutoryService` over `fn_statutory_limit` (§3.6) |
| DC, DC return, sale return | nothing | three new modules (§5, §6, §7) |

---

## 2. Rules the module follows (house)

1. Identity on every `/get` `/post` `/cancel` `/amend` `/delete`:
   `companyId`, `branchId`, `accYear` + the document id (`sbId` / `sdcId` /
   `srId` / `sdrId` / `soId`). No `/list` — grids.
2. One `partyId` = `cus_id` = `led_id`.
3. DRAFT → POSTED → CANCELLED. No approve. `/create` is an upsert on the
   client's uuidv7 id and 409s on a POSTED id ("use /amend").
4. A DRAFT writes the document tables + charges + tenders (as today) and
   NOTHING in `accounts.*`, `stock.*`, the register, loyalty or usage.
   (`postBillToAccounts`-at-create goes; the `sales.auto_post` setting
   now means "the screen calls `/post` right after `/create`".)
5. POST is one `$transaction`, the steps of flow §5.1 in that order,
   status flipped last. Idempotent: a second `/post` on a POSTED id is a
   200 with the same result (`ux_sml_source`, the voucher's
   `avh_src_doc_id`, the register's `gdr_source_doc_id` are the keys it
   checks first).
6. Amounts positive, `av_dr_cr` the side; every non-party leg has
   `av_role`; ledgers by role through `resolveRoleLedgers` — no ledger id
   in sales code, ever.
7. Cancel = reversal rows. Delete = DRAFT only, hard.
8. Server-owned columns are stripped from the DTO (`class-transformer`
   `@Exclude` on input): `*_posted_voucher_id`, `sb_doc_register_id`,
   `*_revision_no`, `*_status`, fulfilment caches, `sb_cogs_amt`,
   `sbi_cogs_amt`, `sbi_lot_id` (the engine sets it; the client may
   PROPOSE `lotId` on a line, the engine validates it).
9. Errors: the module's exception filter, codes `SALES_*` (`SALES_
   RETURN_LOCKS_BILL`, `SALES_IRN_WINDOW_PASSED`, `SALES_CASH_LIMIT`,
   …); every statutory refusal carries `{code, value, effectiveFrom,
   isCompanyOverride}`.
10. Dates are `date` columns after 30 §6 — pass `YYYY-MM-DD`, never a
    midnight instant.

---

## 3. Shared services (new folder `src/modules/sales/posting/`)

### 3.1 `StockPostingService` — the engine's phases, lifted

```
src/modules/stocks/posting/
  stock-posting.service.ts      post(source), cancel(source)   ← moved phases
  stock-line-source.ts          the adapter interface + the three sales adapters
  stock-voucher.source.ts       the stock_voucher adapter (existing caller)
```

`StockLineSource` gives the engine what `postingCte` reads off
`stock_voucher_item` today:

```ts
interface StockLineSource {
  srcModule: 'SALES' | 'STOCK' | 'PURCHASE';
  srcDocType: 'DELIVERY_CHALLAN' | 'SALE_BILL' | 'SALE_RETURN' | 'DC_RETURN' | 'STOCK_VOUCHER';
  srcDocId: string; accYear: string; companyId: string; branchId: string;
  docDate: string; refno: string; partyId?: string; reasonId?: string;
  txnType: 'DC_ISSUE' | 'SALE' | 'SALE_RETURN' | 'DC_RETURN' | ...; direction: 1 | -1;
  lines(): Promise<StockLine[]>;        // item, godown, lotId?, bucket, uom, factor, qty, freeQty,
                                        // batchNo/expiry/serial (lot identity), docRate, docRateWot, mrp, lineNo, splitNo
  attachLot(lineId, lotId, costRate, costRateWot): Promise<void>;   // writes back sdi_/sbi_/sri_lot_id + cost
}
```

`post(tx, source)` runs `resolveLots` (FIFO or the proposed lot; for a
RETURN the bill line's lot if ACTIVE else a new lot at the line's cost)
→ `attachLotsToLines` (through `source.attachLot`) → `writeLedger`
(`sml_src_module/type/id/acc_year` from the source, `ux_sml_source`
idempotency kept) → `applyBalances` → `applyItemCost` → `assertNegative
StockPolicy` (`comp_negstk_apl`) → `refreshLotTotals`, and returns
`{ lines: [{lineId, lotId, costValue, costValueWot}], totalCost }` — the
COGS figure. `cancel(tx, source)` = `writeReversalLedger` + the balance/
cost phases with the sign flipped (exists for stock vouchers today).

The stock-voucher module's `postStockVoucher` becomes
`StockPostingService.post(new StockVoucherSource(svhId))` — one caller
migrated, behaviour identical, its existing tests are the regression.

### 3.2 `SalesPostingService` — legs by role

`postLegs(tx, doc: SalesLegSource)` builds the leg list of flow §5.3 in
the fixed order, resolves every role once (`resolveRoleLedgers` with the
company / branch / `supplyNature` scope), allocates the number
(`allocateVoucherNumber` — series by `sb_bill_mode`: POS → the device
row, WHOLESALE → `seq_device_code 'MAIN'`), writes `acc_voucher_header`
(`avh_src_module 'SALES'`, `avh_src_doc_type`, `avh_src_doc_id`,
`avh_party_id`, `avh_doc_amount`) + `acc_vouchers`, and returns the
voucher id. Totals come from the live header trigger; the service still
asserts Σ DR = Σ CR before writing (a clearer error than
`ck_avh_balanced`).

`reverseLegs(tx, voucherId, reason)` writes the mirror voucher, same type,
sets `avh_reversal_voucher_id` on the original, `avh_voucher_status
'CANCELLED'` + `avh_cancel_reason` on the original.

Leg sources: `BillLegSource`, `ReturnLegSource`, `CogsLegSource` (the
DR COGS / CR INVENTORY pair, used alone by the DC and DC return and
appended by the bill / return).

### 3.3 `DocRegisterService`

`write(tx, doc)` → `acc_voucher_doc_register` + `_doc_detail` rows from
the document header/lines (flow §5.7, §4 step 4, §6.1 step 6); computes
`gdr_is_einvoice_applicable` and `gdr_is_ewaybill_applicable` through
`StatutoryService`; returns `gdrId`. `cancel(tx, gdrId, reason)` → status
CANCELED. `reissue(tx, gdrId)` on amend: the old row CANCELED, a new row
written (the IRN history stays on the old one).

### 3.4 `LoyaltyLedgerService` (flow §5.8 is the contract)

`resolveMember(tx, bill)` (member id → customer → auto-enrol / none),
`resolveScheme(tx, bill)` (one APPROVED scheme by priority, all scope /
window / bill-type filters), `earn(tx, bill)` (base per `lsc_apply_on`
with item rules + slabs, the exclusions, rounding, cap → ONE EARN row,
idempotent on `ux_lld_src_row`), `redeem(tx, bill, tenderRow)` (rate =
scheme's `lsc_redeem_value_per_point`, tender master fallback; every cap;
then `consume(...)` below), `reverseForCancel(tx, bill)`,
`clawbackForReturn(tx, ret, share)` (cap at live balance, shortfall in
`lld_remarks`, never abort), `preview(bill)` for `/validate` and
`party-context`. Setting `sales.loyalty_auto_enrol` (30 §10).

**`schema/13` and `14` contain NO functions and NO triggers at all.** Not the
FIFO, not the sweeps, and — decided 2026-09-21 — not the balance maintenance
either. Everything is this service.

**The reason is the offline app.** The till runs offline and pushes to the
cloud on reconnect. A trigger fires on the *server*, during that push, on rows
written hours earlier under conditions the server cannot see: in a language
nobody here debugs, per statement of a sync batch so a failure points at the
batch rather than the row, and whether or not the service wanted it — so the
push cannot stage, inspect and correct data before it lands. It also forks
quietly from the TypeScript that has to do the same sums for the offline
client, which is exactly the divergence that already happened to the stock
posting engine. One language, one place, one thing to debug when a wallet looks
wrong.

### 3.4a Balance maintenance — `recomputeMembers` / `recomputeLots` / coupons

Three columns are `GENERATED ALWAYS` and Postgres fills them from plain columns
**this service writes** — no trigger involved:

| column | = |
|---|---|
| `loyalty_member.lmb_balance_points` | `earned + adjusted − redeemed − expired − gifted` |
| `loyalty_ledger.lld_lot_balance` | `lld_points − lld_consumed_points` |
| `loyalty_coupon.lcp_balance_value` | `face + topup − used` |

Leave the inputs at zero and every wallet reads zero and every lot looks
untouched. **These recomputes are not housekeeping, they are the feature.**
A generated column may not be written, so a sync push that sends whole rows
must leave all three out of the column list — Prisma omits them already.

`recomputeMembers(tx, memberIds[])` — per member, over `loyalty_ledger` where
`lld_member_id = id AND NOT lld_is_deleted`: `earned = SUM(points) WHERE type IN
('EARN','OPENING')`; `redeemed = −SUM WHERE 'REDEEM'`; `expired = −SUM WHERE
'EXPIRE'`; `gift = −SUM WHERE 'GIFT'`; `adjusted = SUM WHERE type IN
('ADJUST','TRANSFER')`; `lifetime_bill_amt = SUM(lld_base_amount)` and
`lifetime_bill_cnt = COUNT(DISTINCT lld_src_doc_id)` over `EARN` rows with a
`src_doc_id`; `last_earn_on`, `last_redeem_on`, `last_activity_on` as MAX of
`lld_txn_date` over their filters; `next_expiry_on = MIN(lld_expires_on)` over
`EARN` rows with `lld_lot_balance > 0`. Reversals are rows of the **same type
with the opposite sign**, so a plain SUM nets them — never special-case one.
**A member whose every row was deleted must be ZEROED, not skipped:** a
`GROUP BY` returns no row for it, and that is the one way this can leave a
stale wallet behind.

`recomputeLots(tx, [{lotId, lotAccYear}])` — `lld_consumed_points =
−SUM(c.lld_points)` over rows pointing at that lot and not deleted.

`recomputeCoupons` / `recomputeCouponBatches` — `14`'s contract block has the
filters. Note a coupon movement is an **amount** with the sign on the type, so
those are plain sums, unlike the points ledger. The coupon recompute must
**not** set `lcp_status`: ISSUED / PARTIAL / REDEEMED are arithmetic and are
its, but EXPIRED / CANCELLED / BLOCKED are decisions owned by the admin path
and `couponExpiryRun()` — writing a decision from a recompute is how a
cancelled voucher comes back to life.

**Order matters twice.** Lots before members, because `lmb_next_expiry_on`
reads `lld_lot_balance`. Vouchers before batches, because `lcb_redeemed_count`
counts vouchers by status.

### 3.4b The one rule that replaces the trigger

A trigger could not be forgotten; a service call can, and there are **nine**
write paths into the ledger (earn, consume, cancel reversal, return claw-back,
expiry, gift, adjust, transfer, opening). So make forgetting structurally
impossible rather than a matter of discipline:

> **ONE private method** on `LoyaltyLedgerService` is the only code in the
> entire codebase that inserts, updates or soft-deletes
> `sales.loyalty_ledger`. It writes the rows, recomputes the touched lots,
> then the touched members, and only then returns. Every other method goes
> through it — not a repository, not the sync handler, not a migration script.
> The same for `loyalty_coupon_txn` and `loyalty_coupon`.

Nine places to get right becomes one. **Assert it in a test:** no file outside
that service may reference these tables in a write. Both recomputes take
arrays and issue one UPDATE per set, because a sync push is many rows at once.

### 3.4c The reads, also the service's

`lots(memberId, onDate)` — the spendable lots in the ONE order the whole
product agrees on: `type = 'EARN'`, not deleted, `lld_lot_balance > 0`, matured
(`lld_active_from IS NULL OR <= onDate`) and not lapsed (`lld_expires_on IS
NULL OR >= onDate`), `ORDER BY lld_expires_on NULLS LAST, lld_txn_date,
lld_id`. Oldest expiry first, then oldest earned, never-expiring last — the
customer-favourable order, because it spends what would otherwise lapse.
`consume()`, the till and the expiry job all call this one function and none of
them re-implements the ORDER BY.

`redeemable(memberId, onDate)` = `SUM(lld_lot_balance)` over exactly those lots.
**This is not `lmb_balance_points`**, which also counts lots still in their
cooling period and lots lapsed but not yet swept. Redeem against
`redeemable()`, display `lmb_balance_points`.

`balance(memberId)` = `lmb_balance_points` off the wallet row.

`consume(tx, memberId, points, txnType, opts)` → rows written. `txnType` is
REDEEM (money; a tender row must be given), GIFT (stock; no tender) or
EXPIRE (nobody got anything).

1. `points <= 0` → 0 rows, no work.
2. `SELECT lmb_comp_id, lmb_cust_id FROM sales.loyalty_member WHERE lmb_id = $1
   FOR UPDATE` — **not** `SKIP LOCKED`. Two tills redeeming one wallet must
   serialise; in a supermarket that is a real race. Blocking is correct;
   skipping would let both tills spend the same points. The lock has to be in
   the CALLER's transaction — Prisma's interactive `$transaction`, never a
   fresh connection, or the bill's post and the redeem sit in different
   transactions and the lock protects nothing.
3. No member row → throw.
4. `avail = sales.fn_loyalty_redeemable(memberId, txnDate)`; `avail < points`
   → throw naming member, avail, date and points, **before any row is
   written**. `ck_lmb_balance` would catch it anyway; the message must say
   what actually happened.
5. Walk `sales.fn_loyalty_lots(memberId, txnDate)` in its order (oldest
   expiry, then oldest earned, never-expiring last) taking
   `LEAST(left, lot.lot_balance)` from each, and INSERT **one
   `sales.loyalty_ledger` row per lot touched** — `lld_points = -take`,
   `lld_row_no` 1..n within the call, `lld_lot_id` / `lld_lot_acc_year` the
   lot, `lld_money_value = round(take * rate, 2)`, comp / cust from the member
   row and the rest from the caller. Do not collapse the rows: the member
   statement has to say which lot paid.
6. Anything unallocated after the loop → throw (lot balances and wallet
   balance disagree). Unreachable, which is why it must be loud.

`tr_lld_refresh` recomputes `lld_lot_balance` and the wallet cache from those
INSERTs. The service must not write those caches itself and must not trust its
own arithmetic for them — re-read.

`expiryRun(companyId, accYear, on = today, createdBy = 'SYSTEM')` → rows
written. Nightly, per company, on the Nest scheduler with the jobs list (an
expiry is a clock event, and a clock is not a row change — which is why it was
never a trigger).

1. `runId = md5(companyId + '|' + on)` as a uuid — **deterministic**, so a
   re-run on the same date is caught by `ux_lld_src_row` instead of quietly
   writing a second set. Keep the recipe exactly.
2. Lapsed lots: `lld_txn_type = 'EARN' AND lld_is_deleted = false AND
   lld_lot_balance > 0 AND lld_expires_on IS NOT NULL AND lld_expires_on < on`,
   ordered by `lld_expires_on, lld_txn_date, lld_id`.
3. One EXPIRE row per lot: `lld_points = -lot_balance`, `lld_txn_date = on`,
   the lot keys, `lld_src_module 'SALES'`, `lld_src_doc_type 'EXPIRY_RUN'`,
   `lld_src_doc_id = runId`, `lld_src_acc_year = accYear`, `lld_remarks
   'Lapsed on <on>'`, carrying `lld_lsc_id` from the lot.

Idempotent twice over, and both halves must survive the port: a swept lot has
a zero balance and is not selected again, and `ux_lld_src_row` rejects a second
row for the same run and lot.

`couponExpiryRun(companyId, accYear, on = today, createdBy = 'SYSTEM')` → the
voucher sweep from `14`, on the same nightly schedule.

1. `runId = md5(companyId + '|coupon|' + on)` as a uuid — note the `|coupon|`
   segment; it must not collide with `expiryRun()`'s
   `md5(companyId + '|' + on)`.
2. Lapsed vouchers: `lcp_is_deleted = false AND lcp_balance_value > 0 AND
   lcp_valid_upto IS NOT NULL AND lcp_valid_upto < on AND lcp_status IN
   ('ISSUED','PARTIAL')`, ordered by `lcp_valid_upto, lcp_id`.
3. One `sales.loyalty_coupon_txn` row per voucher: `lct_txn_type 'EXPIRE'`,
   `lct_row_no` 1, **`lct_amount = lcp_balance_value`, positive** — a coupon
   movement is an amount and the type carries the sign; this is not the points
   ledger, where a spend is negative. `lct_branch_id` = the voucher's, falling
   back to the batch's `lcb_branch_id`. Src block as above with
   `lct_src_doc_id = runId`.
4. **Then** `UPDATE sales.loyalty_coupon SET lcp_status = 'EXPIRED'`. Movement
   first, status after: `tr_lct_refresh_coupon` recomputes the balance from the
   movement, and the status is the decision laid on top. `fn_lcp_recompute`
   moves a voucher between ISSUED / PARTIAL / REDEEMED only and will never set
   EXPIRED itself — that decision is this job's.

Idempotent twice over as well: a swept voucher has a zero balance **and** a
status that is no longer ISSUED or PARTIAL, so it is not selected again.

**Reads stay in the database** — `fn_loyalty_lots`, `fn_loyalty_redeemable`,
`fn_loyalty_balance`. They write nothing, so they cannot drift, they are the
single opinion about FIFO order that `consume()`, the till and the expiry job
all share, and `party-context` gets the balance in one round trip.

### 3.5 `PromotionUsageService`

`record(tx, bill, schemeIds[])`, `reverse(tx, srcDoc)`. Also
`validateApplied(bill)` — the scheme must be live for date / branch /
party / item (15's rules) or the post refuses `SALES_PROMO_NOT_LIVE`
(the client applied it; the server does not trust that).

### 3.6 `StatutoryService`

`limit(companyId, code, onDate, appliesTo?, aatoClass?)` → the row from
`fn_statutory_limit` or null; cached per request. `assert*` helpers used
by the guards: `assertCashLimit`, `assertPanOrForm60`, `ewayApplicable`,
`einvoiceApplicable`, `hsnDigits`, `withinCancelWindow(kind, generatedOn)`,
`creditNoteCutoff`.

### 3.7 `sales.guards.ts`

The flow §9 list as functions, each returning `void` or throwing the
coded error; WARN-level ones push into `ctx.warnings[]` and throw only
when the code is not in `dto.overrides[]` ∧ `um_can_override`. Reuses
`assertAccYearWritable`, `assertVoucherPartitionExists`, `loadParty`,
`lockBills` from `receipt.guards.ts`. New: `assertRight(userId, menuId,
'post' | 'cancel' | 'amend' | 'override')` reading the four new
`user_menus` flags; `assertDayOpen`; `assertBackdate`; `assertCreditLimit`
(sum of `abl_pending_amount` DR rows + this bill vs `cus_credit_amt_limit`,
count of OPEN bills vs `cus_credit_bill_limit`, oldest OPEN vs
`cus_credit_days`).

**`assertSalesmen(companyId, ids[])` — this one replaces a foreign key, so it
is not optional.** `sb_salesman_id`, `so_salesman_id`, `sq_salesman_id`,
`sr_salesman_id` and `sdc_salesman_id` are all `uuid[]`, and **PostgreSQL
cannot put a foreign key on the elements of an array**. Nothing in the
database will ever catch a salesman id that does not exist, is soft-deleted,
belongs to another company, or is simply a typo — so the service must, on
every create and every amend of all five documents:

* every element resolves in `public.employee_master`, not `emp_is_deleted`,
  and belongs to this company;
* no duplicates in the array (crediting one person twice);
* the array is null or non-empty — never `{}`, which reads as "there is a
  team" while naming nobody;
* refuse with `SALES_SALESMAN_INVALID`.

**Element 1 is the PRIMARY, by convention.** `acc_bill_balance.abl_salesman_id`
stays a single indexed `uuid` and the posting service writes
`sb_salesman_id[1]` into it, so outstanding-by-salesman keeps its btree. The
client must therefore put the primary first. The schema cannot enforce that,
so it is stated in `30` §1d, here, and in the client that builds the array.
The second and any later salesman are recorded on the document and are **not**
credited in the outstanding-by-salesman report.

**Query shape**, unchanged if a third salesman ever appears:
`WHERE sb_salesman_id @> ARRAY[:salesman]::uuid[]`, served by the GIN indexes
`ix_sb_salesman_gin` / `ix_so_salesman_gin` / `ix_sq_salesman_gin` (30 §1c).
A btree cannot answer array containment — do not let a report fall back to
`= ANY(...)` on an unindexed column.

**`loadman` is `uuid[]` for a different reason and keeps no such convention:**
a load crew is genuinely open-ended, nothing joins or indexes it, and there is
no primary. Validate the ids the same way; that is all.

### 3.7a Tender rules (audit of `accountsModule/tenderDetail`, nothing to alter)

`acc_tender_detail` rows keep coming from `/create` through
`TenderDetailService.syncDocumentTenders`; `/post` stamps `td_voucher_id`
and writes the legs. Rules the posting service applies per `td_tender_type_id`:
CASH/CARD/UPI/WALLET/CHEQUE/BANK/VOUCHER → `DR td_tender_ledger_id / CR party`
(surcharge → `td_surcharge_ledger_id`); **CREDIT (9)** → no leg, the party
stays debited, `abl_status OPEN`; **LOYALTY (10)** → `DR` the ledger of role
`LOYALTY_REDEMPTION` / `CR party` — `tnd_ledger_id` on that master is
ignored; **RRN (7)** → refused (`SALES_RRN_RETIRED`: apply the credit note
through `adjustments[]`); **TEMP_CR (8)** → as CREDIT, plus one `accounts.acc_temp_credit` row (31) from the tender's `tempCredit {name, mobile, place, addr, days, notes}` block — mandatory on a TEMP_CR row (`SALES_TEMP_CREDIT_DETAILS_MISSING`); guards `sales.temp_credit_max_days` (REFUSE), `_max_amount` (WARN), `_block_open` (an OPEN row on the same mobile: WARN with its balance, or REFUSE). Bill cancel → `atc_status CANCELLED`. Guards added:
`tnd_daily_limit` (Σ of the day's `td_amount` per master per branch, WARN +
override — stored today, never enforced), `tnd_min_amount` / `tnd_max_amount`
(WARN), the loyalty cap (REFUSE), and the cash limit / PAN (§3.6).

### 3.7b From the `fsales` audit (`fsales-audit.md`)

* `StockPostingService.post` for SALES honours `sales.stock_from_any_godown`
  (D53: FIFO across the branch's godowns when the line's godown is short)
  and `item_auto_break` (D55: a REPACK_OUT / REPACK_IN pair for the parent
  pack before the SALE row — this is the REPACK the engine still owes).
* `SalesPostingService` asserts Σ `sml_base_qty` written = Σ non-service
  line base qty (3.0's stock data verification) — refuse on mismatch.
* `PUT /bills/update-remarks` `{keys, remarks, editRemark}` — text only on a
  POSTED bill, `txn_status_log` row (D57).
* Delivery events gain `VERIFIED` (D54).

### 3.8 `ChargeCarryService`

Flow §7: `propose(orderId, billLines)` → the per-charge proposal;
`consume(tx, billChargeRows)` → `cd_carried_amt` on the source rows;
`release(tx, billChargeRows)` on cancel.

---

## 4. `bill` module — the routes

### 4.1 `POST /api/v1/bills/create` — the DRAFT (exists)

Changes: drop the `postBillToAccounts` / `syncBillPosting` calls and the
`sales.auto_post` branch; add the new columns of 30 §1–§2 to
`save-bill.dto` / `save-bill-item.dto` (`sbBillMode`, ship-to, dispatch,
transport, `sbUsrRefdate`, `sbCustPan`, `sbForm60Ref`, `sbLoyaltyMemberId`,
`sbiSrcItemId`, `sbiBucket`, proposed `sbiLotId`, `sbiPromoUsageId`);
409 on a POSTED id. The `sbCust*` snapshot fields stay writable on the DTO:
for the walk-in customer the client sends what the operator typed (name /
address / GSTIN — printed, kept on the bill, no customer row); for a listed
customer the server copies from the master unless `custOverride: true`
(the "Edit for this bill" tick), and in both cases validates the GSTIN
shape + state code and feeds the register row from the snapshot, not the
master. Multi-order lines (Q26): each line carries its own
`sbiSrcDocId`; the header `sbSrcDocId` is the first.

### 4.2 `POST /api/v1/bills/validate` — the dry run

Same body as `/post`; runs every guard and the source-line checks;
returns `{ ok, warnings[], refusals[], proposals: { charges[], advances[],
creditNotes[], loyalty: {earn, maxRedeem} } }` and writes nothing. The
screen calls it on F-key "check" and before Post; the proposals fill the
charges strip and the set-off panel.

### 4.3 `POST /api/v1/bills/post` — the one-way door

Body: the four keys + `overrides[]` + `printAfter?`. One transaction:

```
 1 lock the bill row FOR UPDATE; 200-return if already POSTED (idempotent)
 2 guards (3.7) — all of them, before any write
 3 source lines: for each sbi with sbi_src_item_id →
      DC line:  lock sdi FOR UPDATE; sdi_open_qty ≥ qty else SALES_DC_LINE_OVER
      SO line:  lock soi; soi_pending_qty ≥ qty (or allow_bill_over_order_qty)
 4 StockPostingService.post(BillDirectLinesSource)      → direct lines' lots + totalCost
 5 SalesPostingService.postLegs(BillLegSource + CogsLegSource(totalCost)) → voucherId
 6 acc_bill_balance SALES row (abl_voucher_id = voucherId); ADVANCE set-offs
   (ADVANCE_ADJUST rows + legs are part of 5's list — 5 and 6 share one builder);
   credit-note set-offs (NOTE_ADJUST rows); BillBalanceRecomputeService
 7 tenders: already rows from /create — now stamp td_voucher_id; cash-limit guard used them in 2
 8 DocRegisterService.write → sb_doc_register_id
 9 LoyaltyLedgerService.earn / redeem (redeem = the type-10 tender's amount → points)
10 PromotionUsageService.validateApplied + record
11 ChargeCarryService.consume
12 sdi_billed_qty / sdc_fulfil_status; soi_delivered_qty / so_fulfil_status; reservation consume
13 sb_status POSTED, sb_posted_on, sb_posted_voucher_id, sb_cogs_amt, caches; appendTxnStatusLog POSTED
-- after commit --
14 GspQueue.enqueue('EINVOICE', gdrId) if applicable ∧ sales.einvoice_auto_fire;
   GspQueue.enqueue('EWAYBILL', gdrId) likewise; print job if printAfter
```

Response: the `/get` payload + `posting: { voucherId, voucherRefno,
registerId, cogsAmt, loyaltyEarned, warnings[] }`.

### 4.4 `POST /api/v1/bills/cancel`

Body: four keys + `reason` (mandatory). Refusals of flow §5.10, in that
order, before any write; the IRN-within-window case calls the GSP
synchronously (`GspClientService.cancelIrn`) and proceeds only on success.
Then the reversal of every step of 4.3 in reverse order; `sb_status
CANCELLED`, `sb_cancelled_*`; log row with the reason.

### 4.5 `POST /api/v1/bills/delete` — DRAFT only

Today this route also cancels a POSTED bill (`deleteBillPosting`). It
becomes: DRAFT → hard delete of bill, items, charges, tenders, hold link;
POSTED → 409 "use /bills/cancel". The order-cancel side effect it carried
moves to `/sale-orders/cancel`.

### 4.6 `POST /api/v1/bills/amend` — R20 on the bill

Body: the full `/create` payload + `baseRevision` + `editRemark`.
`sales.allow_posted_amend` (default TRUE) gates it. Transaction: 4.4's
reversal work WITHOUT the status change → the draft-upsert of the new
payload on the same id/number → 4.3's steps 3–14 → `sb_revision_no + 1`
→ `audit.audit_log` before/after (the receipt's `receipt-amend.service`
does exactly this; copy its shape, not its code — share the audit writer).
Register: `DocRegisterService.reissue`.

### 4.7 `GET /api/v1/bills/get` (exists)

Adds: `posting` block (voucher, register, IRN / EWB status from
`_doc_einvoice` / `_doc_ewaybill`, loyalty, cogs), `sources[]` (the DCs /
orders it drew from with their open quantities), `revisionNo`,
`deliveryStatus`, `locks: { returns: n, allocations: n, irnLive, ewbLive }`
— so the screen can grey Cancel/Amend with the reason before the user
tries.

### 4.8 `GET /api/v1/bills/open-sources`

`?companyId&branchId&partyId&kind=DC|ORDER` → the party's POSTED DCs with
`sdi_open_qty > 0` (with the `DC_RETURN_WINDOW_DAYS` flag) or CONFIRMED
orders with pending lines — the "bill against" picker. Line-level, so the
screen can take some lines of one DC and all of another.

### 4.9 `GET /api/v1/bills/party-context`

The receipt's `party-context` widened: credit position (limit, used, open
bills, oldest), open ADVANCE rows, open credit-note rows, loyalty member +
balance, PAN / Form 60 state, GST reg type, default price level, ship-to
addresses. One call when the customer is picked.

### 4.10 `PUT /api/v1/bills/delivery-status`

`{ keys, event: PACKED|DISPATCHED|DELIVERED, remarks?, vehicleNo?, lrNo? }`
→ `appendTxnStatusLog` + `sb_delivery_status` (menu 226).

---

### 4.10a The bill's shipping block and the GST calls from the bill

`save-bill.dto` carries today only `sbDriverId`, `sbLoadmanId`,
`sbVehicleId`, `sbVehicleNo`. It gains the 30 §1 columns: `sbShipAddrId`,
`sbShipName/Addr/Place/Pin/Phone/Stcd/Gstin`, `sbDispatchGodownId`,
`sbDispatchBranchId`, `sbTransportMode`, `sbTransporterId/Name/Gstin`,
`sbLrNo`, `sbLrDate`, `sbDistanceKm` — validated (GSTIN shape, state code,
distance 0–4000) and copied by `DocRegisterService` into the register's
ship / dispatch blocks, which is what the payload builders read. The bill
screen calls the GST routes DIRECTLY for the loaded bill
(`../gst/plan-backend-gsp.md` §3, `?sync=true`): generate IRN, generate
EWB / Part-B, fetch; each answer is also reflected in `/bills/get`
`posting`. `/bills/post`'s after-commit hook enqueues the same two when
the branch settings say so.

### 4.11 Temp credits — `GET /api/v1/temp-credits/open` and the receipt hook

`/temp-credits/open?companyId&branchId&status=OPEN,PARTIAL&search=` — the
counter's "who owes" list (a grid; also the picker the receipt uses).
`PUT /temp-credits/follow-up` `{atcId, accYear, promiseDate?, remarks}` →
`atc_promise_date / _followup_*`. `BillBalanceRecomputeService` gains one
step: after recomputing an abl row, `UPDATE acc_temp_credit SET
atc_balance_amount = abl_pending_amount, atc_status = CASE …, atc_settled_on
… WHERE atc_abl_id = …`. **Receipt module, one addition:** `/receipts/
open-items` returns `tempCredit {name, mobile, dueDate}` on a bill that has
one, and accepts `?mobile=` to find the walk-in ledger's bills for a
borrower; `/receipts/post` stamps `atc_last_receipt_id`. No other receipt
change — the bill settles like any bill.

## 5. `sale-order` module

* `POST /sale-orders/post` — CONFIRMED + `stock_reservation` rows (WARN
  on shortfall), log. Idempotent.
* `POST /sale-orders/cancel` — refuses when any `soi_delivered_qty > 0`;
  releases reservations; the advance stays an open ADVANCE row (refund is
  a payment, not the order's business).
* `POST /sale-orders/amend` — while OPEN; after partial delivery only
  undelivered lines.
* `/delete` — DRAFT only.
* `/pending-amount`, `/cancel-lines` unchanged.

Fulfilment writers (bill, DC) call `SaleOrderFulfilmentService.consume(tx,
soiId, qty)` / `.release(...)`, which own `soi_delivered_qty`,
`so_fulfil_status`, `so_completed_on` and the reservation consume.

---

## 6. `delivery-challan` module (new: `src/modules/sales/delivery-challan/`)

```
delivery-challan.controller.ts    /create /validate /post /cancel /amend /delete /get /open-for-bill
delivery-challan.service.ts       draft upsert (the bill.service shape)
delivery-challan-posting.service.ts
delivery-challan.guards.ts        purpose, dc_requires_order, order-line open qty
dto/save-delivery-challan.dto.ts, save-delivery-challan-item.dto.ts, cancel-delivery-challan.dto.ts, delivery-challan-response.dto.ts
delivery-challan-exception.filter.ts, types/
```

`/post`: flow §4 — guards → order lines → `StockPostingService.post(
DcLinesSource)` (txn `DC_ISSUE`) → `SalesPostingService.postLegs(
CogsLegSource)` (type `DCh`; skipped under PERIODIC) → `DocRegisterService
.write` (DELIVERY_CHALLAN, no IRN) → fulfilment → status → after commit
`GspQueue.enqueue('EWAYBILL')` when applicable.

`/cancel`: refuses on `sdi_billed_qty > 0` ∨ `sdi_returned_qty > 0` ∨ EWB
live past window; then stock reversal, COGS reversal, register CANCELED,
fulfilment release.

`/open-for-bill` = `/bills/open-sources?kind=DC` (one implementation, two
routes so each screen has its own name).

---

## 7. `dc-return` module (new: `src/modules/sales/dc-return/`)

Same file set. `/create` takes `sdrDcId` + lines each with `sdriDcItemId`;
`/post`: line qty ≤ `sdi_open_qty` (locked) → `StockPostingService.post(
DcReturnLinesSource)` (txn `DC_RETURN`, lot = the DC line's, bucket per
line) → COGS reversal voucher (type `DCR`) → register (DELIVERY_CHALLAN,
INWARD, sign −1) → `sdi_returned_qty`, `sdc_fulfil_status` → status →
EWB when transport filled. `/cancel`: the mirror; refused when the DC is
CANCELLED (cannot happen — the DC refuses first) or the EWB is live past
window. No amend (small, cancel + re-enter). `/delete` DRAFT only.
`GET /dc-returns/open-lines?sdcId` → the DC's returnable lines.

---

## 8. `sale-return` module (new: `src/modules/sales/sale-return/`)

Same file set + `sale-return-settlement.ts` (the three modes of flow §6.1
step 5, built on the receipt's `allocation-engine.ts` for ADJUST).

* `/create`: `srBillId` optional (free return); lines with
  `sriBillItemId`; `srSettleMode`; tenders[] for CASH.
* `GET /sale-returns/bill-lines?sbId&accYear` → the bill's lines with
  `returnable = sbi_bill_qty − sbi_return_qty`, free-item flags, lot,
  cost, scheme — the screen loads from this, never from `/bills/get`.
* `/validate`, `/post` (flow §6.1), `/cancel`, `/amend` (same gate),
  `/delete` DRAFT-only, `/get`.
* Return-and-re-bill (Q28) is the client's gesture: it calls
  `/sale-returns/post` then opens the bill screen with `sourceReturnId`;
  `/bills/create` accepts `sbSrcDocType 'SALE_RETURN'` for the trail.

---

## 9. `txn-status-log` + `txn-hold`

`TxnStatusDocType` gains `DC_RETURN`; `TxnStatusEvent` gains `CONFIRMED`,
`AMENDED`, `PACKED`, `DISPATCHED`, `DELIVERED`, `IRN_GENERATED`,
`IRN_CANCELLED`, `EWB_GENERATED`, `EWB_CANCELLED` (the GSP worker writes
the last four). `txn-hold`'s `convert` learns the two new doc types.

---

## 10. Permissions

`user_menus` gains `um_can_post`, `um_can_cancel`, `um_can_amend`,
`um_can_override` (30 §7). The users module's save-menus DTO carries all
ten flags (memory: the save is a FULL REPLACE — a client sending six
flags would wipe the four). `assertRight` reads them per `menuId`: bill
12, order 11, return 13, DC 182, DC return (new id), e-inv 152, e-way 153,
delivery update 226. `/validate` returns `rights: {post, cancel, amend,
override}` so the screen greys buttons before the user clicks.

---

## 11. Backend gaps found on the way (not blocking, listed)

1. `bill.service` writes `txn_status_log` on every draft save step —
   keep, but the POSTED / CANCELLED / AMENDED rows must come from the
   posting service in the same transaction, not from the controller.
2. `TenderDetailService.syncDocumentTenders` snapshots the party ledger;
   a bill whose customer changes on amend must re-sync tenders (the
   amend path re-runs the draft upsert, which does).
3. `allocateVoucherNumber` has no POS-device series path yet (the
   receipt only used MAIN) — add `deviceId` to `VoucherNumberScope`.
4. The `sales.auto_post` setting: now "call /post after /create" on the
   client; the server no longer reads it.
5. Prisma: the enum-typed register columns (`accounts."GdrDocType"` etc.)
   are enums in the schema; the new sales tables are varchar+CHECK —
   keep them so, do not "fix" them into enums.

---

## 12. Tests (`test/sales/`, live against the test DB, seeded and cleaned)

**A conformance test, first, because this one has already failed in the wild.**
The five crew/vehicle columns are deliberately repeated on all four
goods-moving documents — `*_driver_id`, `*_supervisor_id`, `*_loadman_id`,
`*_vehicle_id`, `*_vehicle_no` on `sale_bill`, `sale_dc`, `sale_return`,
`sale_dc_return`. Repetition is the price of keeping them off the shared band
(a home delivery has a driver and no consignment), and repetition drifts:
`supervisor_id` was on the bill ALONE and `vehicle_id` was missing from the DC
return before anyone looked. So assert it rather than trust it:

```sql
SELECT p.prefix, f.col
  FROM (VALUES ('sale_bill','sb'),('sale_dc','sdc'),
               ('sale_return','sr'),('sale_dc_return','sdr')) AS p(tbl, prefix)
 CROSS JOIN (VALUES ('driver_id'),('supervisor_id'),('loadman_id'),
                    ('vehicle_id'),('vehicle_no')) AS f(col)
 WHERE NOT EXISTS (SELECT 1 FROM information_schema.columns c
                    WHERE c.table_schema='sales' AND c.table_name=p.tbl
                      AND c.column_name = p.prefix||'_'||f.col);
-- must return ZERO rows
```

Extend the same test to the types, not just the names: `loadman_id` must be
`uuid[]` everywhere and the other four scalar, or one document quietly grows a
second shape.


Bill: post direct (legs balance, stock row, COGS pair, abl row, register,
loyalty EARN) · post against DC (no stock, no COGS, `sdi_billed_qty`) ·
mixed lines · multi-order · cash bill (tender legs, abl CLOSED) · advance
set-off (order's own first) · credit-note set-off · idempotent second post
· cancel (every reversal row, caches restored) · cancel refused: return
exists / receipt allocated / IRN past window · amend (revision, audit
row, register reissued) · amend refused on baseRevision · cash limit
REFUSE · PAN/Form 60 · credit limit REFUSE + override · backdate WARN +
override · discount cap · HSN digits WARN by AATO class · draft delete ·
POSTED delete 409 · rights (no `um_can_post` → 403).

Order: post = CONFIRMED + reservation (WARN shortfall) · cancel refused
after delivery · fulfilment through a bill and a DC · charge carry PRORATA
closes exactly across three bills · FULL / MANUAL / NONE.

DC: post (DC_ISSUE, COGS at lot cost, register no IRN, EWB enqueued above
limit — TN 1,00,000 intra vs 50,000 inter) · below limit no EWB · cancel
refused after bill / after DC return · convert purpose · window flag.

DC return: post only open qty · over-qty refused · bucket DAMAGED lands
DAMAGED · COGS reversed at the DC cost · DC locked after.

Return: against bill — qty cap, SALE_RETURN rows in the bill's lot, legs
(SALES_RETURN + OUTPUT_* DR, party CR), COGS reversal, abl CR row · CASH /
ADJUST (original first, then by due date, remainder open) / ADVANCE ·
loyalty REVERSE pro-rata, never below zero · free-item claw-back · free
return (flagged, no COGS) · cross-branch · window WARN · credit-note
cutoff WARN · cancel refused when the CR row was applied.

PERIODIC: flip `accounts.cogs_mode`, re-run the DC and direct-bill tests —
no COGS legs, everything else identical.

---

## 13. Phases (the diagram's P0–P4; P5 is the Qt plan)

| phase | delivers | depends on |
|---|---|---|
| **P0** | 11, 13, 15 §6, 29, 30 run; `prisma db pull`; `ensure_sales_doc_partitions` for the open years | Prathap |
| **P1** | §3 shared services, stock-voucher module migrated onto `StockPostingService`, its tests green | P0 |
| **P2** | bill `/validate /post /cancel /amend`, `/delete` DRAFT-only, `/open-sources`, `/party-context`, `/delivery-status`; order `/post /cancel /amend`; GSP enqueue stubs (the worker is `../gst/`) | P1 |
| **P3** | delivery-challan + dc-return modules | P2 |
| **P4** | sale-return module + settlement | P2 |
| **P5** | Qt — `plan-qt-sales.md` | P2 onward, screen by screen |

---

## 14. Do not port from 3.0

The client-built voucher JSON stored verbatim (`fsales` flag 1) — legs
are the server's; the ±0.50 balance tolerance — a voucher balances to the
paisa or it is refused; approval as the posting gate (`configs 99`) — no
approval step; the "Claim" settlement mode; hard-coded ledger numbers
(100020 …) — roles; anything at all from `fx_sale_chln_entry`.
