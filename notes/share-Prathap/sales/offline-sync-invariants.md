# Invariants that survive offline sync — which guards may be CHECKs, and which may not

**2026-09-21.** Posting is ONLINE this phase. This note is written now anyway,
because §8c.3 of the analysis promises that *adding offline later needs no
ALTER* — and the constraints this note removes would each have had to be
dropped the day a till first posts on sync. Deciding it now also means the
services are built with the reconciliation path in them, instead of retrofitting
one after the first wall of failed syncs.

It is the companion to "Where logic lives" in `README.md`. That rule says where
behaviour goes; this one says which **guards** the database may still enforce
once writes can arrive late, out of order and from more than one device.

---

## 1. The rule

> A `CHECK` may guard **one row's own arithmetic**.
> A `CHECK` may **not** guard a **contended shared resource** — a balance, a
> quantity or a count that more than one device can draw from.

The test is a question: *could two tills, both offline, each legitimately
consume the last of this?* If yes, the constraint is a sync-time abort waiting
to happen.

### Why a write-time CHECK cannot survive offline

Two tills are offline. One wallet holds 600 points. Each till redeems 500.
Both bills are real; both customers have walked out with the goods. The first
batch syncs and the wallet reads 100. The second batch syncs and
`ck_lmb_balance` — `earned + adjusted − redeemed − expired − gifted >= 0` —
**aborts the transaction**.

That is not a guard. It is a refusal to record a sale that already happened,
and it fails the whole batch, not just the offending row. Offline means the
decision was taken at the till, with the information the till had. The
server's job on sync is to **record** what happened and surface the conflict —
claw back, write off, or chase the customer — never to pretend it did not.

### The second reason: aggregates arrive in pieces

Even the per-bucket `>= 0` guards fail, and for a subtler reason. Reversals in
this schema are rows of the same type with the opposite sign. If a REDEEM
reversal syncs from one device before the REDEEM it reverses syncs from
another, `lmb_redeemed_points` is transiently **negative** — through nobody's
fault. Any `CHECK` over a column that is a running aggregate of a ledger which
arrives in fragments will eventually fire on correct data.

---

## 2. Where the guard went instead

Three places, and all three are needed:

| | who | when |
|---|---|---|
| **Policy** | `LoyaltyLedgerService` (and the other posting services) | at post, **online**, where the true balance is visible — `consume()` still throws before it writes a single row |
| **Record** | the database | always — the generated balance columns are simply allowed to go negative |
| **Detect** | the reconciliation report | nightly, off the partial indexes below |

Nothing is lost for the online phase: an operator still cannot overdraw a
wallet, because the service refuses it. What changes is that the *database*
stops being the thing that refuses, because under sync it would refuse the
wrong things.

---

## 3. What changed, object by object

Every balance column involved is already `GENERATED ALWAYS … STORED`, so
removing the guard clause is all that is needed — the column computes the
negative number by itself, and a partial index makes finding it free.

| file | was | now | the index that replaces it |
|---|---|---|---|
| `13` `loyalty_member` | `ck_lmb_balance` (wallet ≥ 0) and `ck_lmb_points` (each bucket ≥ 0) | both dropped; `lmb_balance_points` may go negative | `ix_lmb_overdrawn` |
| `13` `loyalty_ledger` | `ck_lld_lot_not_overdrawn` (consumed ≤ lot) | dropped; `lld_lot_balance` may go negative | `ix_lld_lot_overdrawn` |
| `14` `loyalty_coupon` | `ck_lcp_used` (used ≤ face + top-up, and the aggregate ≥ 0 clauses) | narrowed to `ck_lcp_face` (`lcp_face_value > 0` — set once at issue, not an aggregate) | `ix_lcp_overdrawn` |
| `14` `loyalty_coupon_batch` | `ck_lcb_counts` (redeemed ≤ issued) | dropped | `ix_lcb_overdrawn` |
| `29` `sale_dc_item` | `ck_sdi_qty` incl. `billed + returned ≤ dc qty` | narrowed to the challan's own `dc_qty ≥ 0 AND free_qty ≥ 0`; billed and returned are written by *other* documents | `ix_sdi_overdrawn` |
| `31` `acc_temp_credit` | `ck_atc_amounts` incl. `0 ≤ balance ≤ credit` | narrowed to `atc_credit_amount > 0` — set once when the tender is taken | `ix_atc_overdrawn` |

**What stayed, and why it is not the same thing:** `ck_sri_return_qty`,
`ck_lct_amount_nonzero`, `ck_lcb_min_bill`, `sr_refund + adjusted + credit ≤
sr_return_amt`. Each is one row's internal arithmetic, decided by one device at
one moment. No second till can contend for it, so no sync batch can be aborted
by it.

---

## 4. The recompute triggers — REMOVED (decided 2026-09-21)

An earlier draft of this note argued to keep them, batched at statement level,
because a recompute is idempotent under replay and order-independent while an
*incrementing* cache double-counts on a re-sent batch. That argument about
**recompute vs increment** is still right, and the service must honour it. What
it got wrong was *where* the recompute runs.

**It is an offline app.** The till runs offline and pushes to the cloud on
reconnect. A trigger fires on the SERVER, during that push, on rows written
hours earlier under conditions the server cannot see:

* in a language nobody on this team debugs, in a stack frame no TypeScript
  trace reaches;
* per statement of a sync batch, so a failure is attributed to the batch
  rather than to the row that caused it;
* whether or not the service wanted it, so the push cannot stage, inspect and
  correct data before it lands;
* and it forks quietly from the TypeScript that must do the same sums for the
  offline client — the divergence that already happened to the stock engine.

So `13` and `14` now contain **no functions and no triggers at all**. Balance
maintenance is `LoyaltyLedgerService.recomputeMembers/recomputeLots/
recomputeCoupons/recomputeCouponBatches` (plan-backend §3.4a), and the reads
went with it (§3.4c).

**What replaces the guarantee a trigger gave.** A trigger could not be
forgotten; a service call can, and there are nine write paths into the ledger.
So the rule is structural, not disciplinary: **one private method is the only
code in the codebase that writes each of these tables**, and it recomputes
before it returns — lots before members, vouchers before batches. A test
asserts no file outside the service writes them. Nine places become one
(§3.4b).

**What stays in the database, and none of it is code:** the three `GENERATED
ALWAYS` columns (`lmb_balance_points`, `lld_lot_balance`, `lcp_balance_value`),
which Postgres fills from plain columns the service writes; every CHECK,
unique index and foreign key; and the six reconciliation indexes of §3. One
consequence for the sync writer: **a generated column may not be written**, so
a push that sends whole rows must leave those three out of the column list.
Prisma omits them already; hand-rolled SQL must too.

## 5. Verified on `erp_dry`, 2026-09-21

`13` and `14` rebuilt from scratch, then one transaction:

**While the triggers still existed**, the arithmetic they encoded was
exercised end to end, and those results are what the service now has to
reproduce — they are the acceptance test for §3.4a:

| step | expected | got |
|---|---|---|
| three EARN lots (100/200/300) | earned 600, balance 600, next expiry the earliest | ✅ 600 / 600 / 2027-09-01 |
| a spend of 150 across **two** lots | balance 450, lot 1 empty, lot 2 at 150, **next expiry advances** | ✅ 450, lots 0 / 150 / 300, next expiry 2027-09-02 |
| a second till syncs a 600 spend the wallet cannot cover | **recorded, not refused** | ✅ balance −150, no error |
| the reconciliation predicates | find the wallet and the lot | ✅ 1 and 1 |

The advancing next-expiry is the one to keep: it only comes out right if the
**lots are recomputed before the members**, because `lmb_next_expiry_on` reads
`lld_lot_balance`. That ordering was the trigger's to guarantee and is now the
service's.

**After the functions were removed**, `13` and `14` were rebuilt from scratch
on `erp_dry`: both run clean, the schema `sales` holds **no functions at all**
and the loyalty tables carry **no triggers**, while all three `GENERATED`
columns and the six reconciliation indexes are present.

---

## 6. Still owed

* **The reconciliation report itself** — one screen or job per overdrawn
  resource, reading the six indexes. Not built; it is the replacement for what
  the CHECKs used to catch, so it is not optional once offline lands.
* **The conflict policy per resource** — when a wallet syncs overdrawn, is it
  clawed back from future earnings, written off, or chased? That is a business
  decision per company, not a schema one, and it is not yet taken.
* Nothing from the `19`–`28` sweep — see §7, it came back clean.
* The stock findings of §7b are APPLIED — see the note there.

---

## 7. The `19`–`28` sweep, 2026-09-21

### 7a. The accounts / posting files — CLEAN, nothing to change

`19_posting_roles`, `20_tax_rate_master`, `21_accounts_posting_setup`,
`22`–`28`. Sixteen objects in total, all of them in `19`, `20` and `21`;
`22`–`28` create none.

| object | kind | verdict |
|---|---|---|
| `fn_tax_rate_ledger_guard` + `tg_trl_guard` · `fn_charge_tax_guard` + `tr_charge_tax_guard` · `fn_ledger_map_guard` + `tg_alm_guard` | guards on `inventory.tax_rate_ledger`, `public.charge_master`, `accounts.acc_ledger_map` | **stay** — all three fire on MASTER tables, configured centrally and online. No till contends for them |
| `fn_ledger_for`, `fn_tax_ledger_for`, `fn_tax_ledger_for_strict`, `fn_charge_tax_id`, `fn_check_role_ledger` | `LANGUAGE sql STABLE` resolvers | **stay** — pure reads, and the ledger-role resolution order is precisely the "one opinion several callers must not disagree about" case |
| `vw_charge_tax_setup`, `vw_tax_ledger_setup`, `vw_bill_lines_without_tax` | read projections / exception reports | **stay** |
| `fn_ledger_tax_bridge` + `tr_ledger_tax_bridge` | back-fills `led_gst_rate` ↔ `led_tax_id` on `acc_ledger_master` | **stays**, and this one is worth saying out loud: it is the expand→migrate→contract shim, and its entire job is to catch writes from code that does not yet know about `led_tax_id`. Move it into a service and the old write path bypasses it, which is the one thing it exists to prevent |

Every CHECK in these files is master data — rate between 0 and 100, taxability
and supply-nature enums, effective-date ordering, cess basis agreement. Not one
compares a consumed figure against an available figure. **No change.**

### 7b. The stock engine — APPLIED 2026-09-21

`schema/stock/16_stock.sql`, `19_stock_posting.sql`, `20_stock_transfer.sql`
share those numbers and are a different matter: 38 objects, and they are where
both rules actually bite.

1. **Four posting functions are units of work written in plpgsql** —
   `fn_svh_post`, `fn_svh_cancel` (19), `fn_svh_post_transfer`,
   `fn_svh_receive_transfer` (20). Exactly the shape of `fn_loyalty_consume`,
   and this is the divergence that already happened: NestJS reimplemented this
   engine and the `.sql` is now a spec rather than what runs. Applying the
   "Where logic lives" rule here means saying so in the files, as `13` and `14`
   now do.

2. **`fn_sml_freeze_guard` (16, ~line 3288) refuses late-arriving movements.**
   It aborts a `stock_ledger` insert while a godown is frozen for a physical
   count, and it tests **`now()`** against the freeze window — the moment the
   row ARRIVES, not the moment the goods moved. A sale made at 10:00, before
   the count began, that syncs at 14:00, during it, is refused although the
   shelf was already short when the counter started counting.
   **The fix is narrow and the column already exists:** test
   `NEW.sml_doc_datetime` (the movement's own timestamp) instead of `now()`,
   and let a movement that *happened* before the freeze through, flagged for
   the count to reconcile. `stock_ledger` also carries `sml_sync_date`, so the
   offline shape was anticipated here — only this guard was not.

3. **Two contended CHECKs on `stock.stock_balance`** — `ck_sbl_reserved`
   (`sbl_reserved_qty >= 0`) and `ck_sbl_transit` (`sbl_transit_in_qty >= 0`).
   Both columns are running aggregates: a reservation release, or a transit
   receipt, arriving before the reservation or dispatch it answers drives them
   transiently negative and aborts the batch. Same class as the six in §3.
   The design is already inconsistent with itself here — there is **no**
   `sbl_qty >= 0`, because on-hand stock is deliberately allowed to go negative
   and `comp_negstk_apl` decides what to do about it in the service. Reserved
   and transit should be recorded on the same terms.

**Done:** all 33 objects are out of `schema/stock/16`, `19` and `20` (only the
DDL partition helper stays); the three that were actually DEPLOYED are dropped
by `34_stock_drop_db_logic.sql`, which carries the freeze-guard fix
(`sml_doc_datetime`, not `now()`) and the warning not to drop the append-only
guards until the service is the only writer of `stock_ledger`. The two
contended CHECKs on `stock_balance` (`ck_sbl_reserved`, `ck_sbl_transit`) are
still there — they are §3's class and were not part of this pass.

Also noted, not a finding: **`fn_slt_resolve` is not a pure read** — it
`INSERT`s into `stock.stock_lot` when no lot matches (get-or-create). Two
devices syncing can race to create the same lot identity; worth confirming a
unique index covers the natural key before offline lands.

