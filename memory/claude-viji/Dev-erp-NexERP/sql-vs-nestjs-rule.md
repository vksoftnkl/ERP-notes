---
name: sql-vs-nestjs-rule
description: Where logic lives — 2026-10-05 TIGHTENED: no DB functions/triggers at all, logic in NestJS only; CHECKs, generated columns, indexes stay in the DB
metadata:
  node_type: memory
  type: project
---

**TIGHTENED 2026-10-05 (user, on the Loyalty Status plan): "dont keep any function in db write in nestjs only".** No new DB function, trigger or view. The "cache-recompute triggers" and "pure read functions" allowances below are WITHDRAWN for new work; that logic goes in a NestJS service (as LoyaltyLedgerService already does). CHECK constraints, generated columns, FKs and indexes are still DB-side. Write every backend plan this way.

The user settled this on 2026-09-21, after asking "we are going to run this
in nestjs so is good to convert to backend plan rather than a sql function
right". The rule now sits at the top of `share/sales/README.md`:

> **The database keeps what must be true NO MATTER WHO WRITES** — CHECK
> constraints, generated columns, cache-recompute triggers, grant-boundary
> views, partition DDL helpers, and pure read functions several callers must
> not disagree about.
>
> **NestJS owns what happens INSIDE A UNIT OF WORK** — allocation, posting,
> settlement, scheduled jobs.

**Why:** both failure modes are already in this codebase. Put a unit of work
in the database and it silently forks — the stock posting engine
([[stock-engine-not-deployed]]) was written as `.sql`, NestJS reimplemented
it, and the `.sql` is now only a spec. Put a write-path rule in the service
and the constraints go decorative — `ck_avh_balanced` cannot catch a voucher
with no legs ([[voucher-balanced-check-vacuous]]) because the totals it
checks are stamped by the writer.

**How to apply:** ask whether the behaviour is invoked as one step of a
transaction the service already owns. If yes it is NestJS's. If it must hold
after a correction, a soft delete or a hand fix by anyone, it is the
database's.

**RESTATED FLATLY, 2026-09-22, in the user's words: _"we do not want any sql
function in db all must live in nestjs only."_** No carve-out offered, no
"except invariants". Treat the 2026-09-21 framing below as history and this as
the rule.

**Live state, read off 192.168.0.106 that day:** 15 own functions + 5 own
triggers left. Schema `stock` is DONE — `34` section 2 HAS RUN, zero triggers,
one function (`fn_create_stock_partitions`); `fn_sml_apply`, `fn_svh_post`,
`fn_svh_cancel`, `fn_slt_resolve`, `fn_sml_immutable`, `fn_sml_forbid_delete`,
`fn_sml_freeze_guard` all return nothing from `pg_proc`. `sales`, `inventory`,
`fixed`, `audit` were already clean. Everything left is `accounts` (7 fn / 4 tg)
and `public` (7 fn / 1 tg).

**`37_drop_remaining_db_logic.sql` is the sweep** (share root, written
2026-09-22, sections 1+2 dry-ran clean on `erp_dry`). Four tiers:

1. **Safe now** — the 5 triggers + 4 functions: `fn_acc_group_inherit_nature`
   (32-hop cycle cap must survive the port), the two cross-scope name guards on
   groups and ledgers (**a unique index CANNOT express this** — the rule is
   shared-vs-company-scoped, a predicate over two rows; and **keep the
   grandfather early-return** or existing rows stop saving),
   `fn_avh_refresh_totals` (+ the leg-moved-voucher recompute),
   `fn_asv_check_scope`.
2. **Safe now** — `fn_seed_ledger_map` + `fn_ledger_map_catalogue`, one-shot
   seeds already run. **Trap: `fn_seed_ledger_map` takes (boolean, varchar)** —
   `DROP FUNCTION IF EXISTS ...()` matches nothing and still reports success.
3. **BLOCKED** — `fn_app_settings_effective` / `fn_app_settings` /
   `fn_app_setting`. `app-setting-value.service.ts:232` does
   `SELECT * FROM public.fn_app_settings_effective(...)` TODAY. Port the
   precedence (GLOBAL < COMPANY < BRANCH < DEVICE < USER) into the service
   first — it must exist in TypeScript anyway for the offline till, same
   argument that retired `fn_statutory_limit`.
4. **DECIDED 2026-09-22 — THEY STAY.** The 4 partition DDL helpers
   (+ `ensure_sales_doc_partitions`, in 29/31 but NOT deployed live). Not
   because they are harmless: because moving partition creation into NestJS
   means **the API needs DDL rights on the production database**, a strictly
   larger hole, open all the time instead of once a year.

**So the rule in full: no SQL function in the database, EXCEPT partition DDL
helpers. Final state of `pg_proc` for our schemas is FOUR rows, never zero.**
Anything else that appears there is drift. The rider: the year-roll is a HUMAN
step, and `ensure_acc_year_partitions` does not cover `public.print_log`
(accepted; bites only on the roll into 2028-29).

**The cost of the stock drop, and why it is already paid.** The DB no longer
stops an `UPDATE`/`DELETE` on `stock_ledger` (migrations `20260922060000` /
`20260922120000` dropped `tr_sml_immutable`, `tr_sml_forbid_delete`,
`tr_sml_forbid_truncate`). The replacement is structural and **IS IN PLACE**:
one writer (`stock-voucher-posting.helper.ts`), INSERT only, asserted by
`test/stock-ledger-single-writer.e2e-spec.ts` — raw SQL *and* the Prisma model
API (`stockLedger.create/update/delete/...`), plus a self-check so an empty
scan cannot pass forever. The freeze check is carried by `validate()`, on
`sml_doc_datetime` not `now()`. **I recorded this as missing on 2026-09-22 and
was wrong — check `test/` before calling a guarantee unowned.**

**What still stays, because it is declarative and cannot run at the wrong
moment:** GENERATED columns, CHECKs, unique/partial indexes, FKs, views.
Those are not functions and the rule does not touch them.

---

**SUPERSEDED IN PART, 2026-09-21.** The user's decision, in their words:
*"yes move to nestjs dont keep anyfunction in db because i said you its a
offline app now when we push datas to cloud means something will get issue."*

**The offline rider now outranks the rule** for anything an offline device
writes: **no functions and no triggers in the database at all** on those
tables. `schema/13_loyalty_ledger.sql` and `schema/14_loyalty_coupon.sql` were
stripped completely — FIFO consume, both expiry sweeps, all four balance
recomputes, all six triggers and the three read functions. Both files rebuild
clean on `erp_dry` with schema `sales` holding **zero functions** and the
loyalty tables **zero triggers**.

**Why (the user's reason, and it is the right one):** a trigger fires on the
SERVER during a cloud push, on rows the till wrote hours earlier under
conditions the server cannot see — in a language this team does not debug, per
statement of a sync batch so a failure points at the batch rather than the row,
and whether or not the service wanted it, so the push cannot stage and correct
data before it lands. It also forks from the TypeScript that must do the same
sums for the offline client.

**What replaces the trigger's guarantee** (plan-backend §3.4b): a trigger
cannot be forgotten, a service call can, and there are NINE write paths into
`loyalty_ledger`. So it is structural — **ONE private method is the only code
in the codebase that writes each table**, it recomputes before returning, and a
test asserts no file outside the service writes those tables.

**Two orderings that must survive** (they were the triggers' job): recompute
**lots before members**, because `lmb_next_expiry_on` reads `lld_lot_balance`;
recompute **vouchers before batches**, because `lcb_redeemed_count` counts
vouchers by status. The coupon recompute must NOT set `lcp_status` — arithmetic
statuses are its, but EXPIRED/CANCELLED/BLOCKED are decisions.

**What stays in those files, because it is declarative and cannot run at the
wrong moment:** three `GENERATED ALWAYS` columns (`lmb_balance_points`,
`lld_lot_balance`, `lcp_balance_value`), all CHECKs, unique indexes and FKs,
and the six reconciliation indexes. **Sync trap:** a GENERATED column may not
be written, so a push sending whole rows must omit those three from the column
list (Prisma already does).

---

**`fn_statutory_limit` went too (2026-09-21).** It was a `sql STABLE` read, so
not dangerous the way a trigger is — but it is the one piece of law the OFFLINE
TILL must apply with no server in reach (269ST cash limit, PAN/Form 60, the
e-way floor), so the resolution order has to exist in TypeScript regardless. A
second copy in plpgsql = two implementations of one statute. Now
`StatutoryService.limit()`; `29` keeps the `statutory_limits` TABLE and spells
the precedence out as a comment. **The precedence IS the rule, keep it exactly:**
company override > specific `applies_to` > `aato_class` set > latest
`effective_from`, LIMIT 1. 29's §7 verify probes were rewritten as a plain
query (they now double as the reference implementation) and give the designed
answers: EWAY TN 100000 beats dflt 50000; TCS 206C(1H) answers for 2024-06-01
and correctly returns NULL for 2026-06-01.

**FINAL STATE of the P0 chain — two objects left, both deliberate:**
`ensure_sales_doc_partitions` (29, restated in 31) is **DDL**, called by a
person or a year-roll job, never during a sync push — and the live DB is
already built that way (`ensure_acc_year_partitions`,
`fn_create_year_partitions`, `stock.fn_create_stock_partitions` all exist), so
removing only ours would make it the odd one out without making anything safer.
`vw_gst_credential` (11) is a **view** and IS the grant boundary — revoke
SELECT on `gst_company_credential`, grant it on the view; replace it with a TS
query and the read-only role can read credentials again. Everything else in
11-15, 29, 30, 31 is tables, constraints, indexes and comments.

**`schema/stock/` STRIPPED TOO (2026-09-21, user: "now do the same stripping
for schema/stock files").** 16, 19 and 20 declared 33 functions/triggers; they
now declare **one** — `stock.fn_create_stock_partitions` (DDL). All three files
rebuild clean on `erp_dry`; after the drop + a 16 re-run the schema holds that
one function and **zero triggers**.

* 19 and 20 were **pure spec** — `fn_svh_post`, `fn_svh_cancel`,
  `fn_svh_post_transfer`, `fn_svh_receive_transfer` and the locks were NEVER
  deployed. Their `COMMENT ON FUNCTION` text was excellent and is preserved as
  the contract block in each file.
* Only **four** functions were live: the DDL helper plus `fn_sml_immutable`,
  `fn_sml_forbid_delete`, `fn_sml_freeze_guard`. **`fn_sml_forbid_delete` is in
  NO FILE** — added by hand by someone.
* **`34_stock_drop_db_logic.sql`** drops the deployed three. TWO WARNINGS IN IT:
  (a) `fn_sml_freeze_guard` tests `now()` against the physical-count freeze
  window — the ARRIVAL time — so a sale made before the count that syncs during
  it is refused; the replacement compares `sml_doc_datetime`.
  (b) `fn_sml_immutable` + `fn_sml_forbid_delete` are the APPEND-ONLY
  guarantee and do NOT break sync; **do not run section 2 until the service is
  the only writer of `stock_ledger` and only ever INSERTs** (a correction is a
  reversing row).
* Gotchas hit while stripping: a bare `CREATE CONSTRAINT TRIGGER`
  (`tr_sag_coverage`, DEFERRED — its "validate the ladder once at COMMIT, never
  per row" rule is kept as a comment) escaped the DROP/CREATE-pair pattern; and
  the deployed `fn_create_stock_partitions` has a DIFFERENT RETURN TYPE from
  16's, so re-running 16 failed with *cannot change return type* — §3 of 34
  reconciles it.

**The original rule, still the framing for files OFF the sync path:** the
database keeps what must be true no matter who writes; NestJS owns what happens
inside a unit of work. `19`–`28` were audited against it and are clean —
master-data guards, `sql STABLE` resolvers, views, and `fn_ledger_tax_bridge`
(the expand→migrate→contract shim). Not yet decided: whether the offline rider
should also strip `schema/stock/` (38 objects, incl. 4 posting functions) —
see [[offline-sync-invariants]] §7b.

**Original application (for the record):**
