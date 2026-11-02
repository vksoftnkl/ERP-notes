# Sales + GST — the order to execute, for Prathap

> **STATUS, 2026-09-22 — read `../HANDOVER_2026-09-22.md` first; it is the
> current one-page instruction and this file is the map behind it.**
>
> **A1 ✅ · A2 ✅ · A3 ✅ · A4 ✅ · A5 ✅ — THE SALES BACKEND IS BUILT.**
>
> Verified against the running API at 11:34 on 2026-09-22: **373 routes**, up
> from 331 that morning. `/bills` carries all fourteen verbs (create, validate,
> post, cancel, amend, delete, get, open-sources, party-context,
> delivery-status, update-remarks, transport, tender-context, retender);
> `/sale-orders` gained post, cancel, amend; `/delivery-challans` (10),
> `/dc-returns` (7), `/sale-returns` (9) and `/temp-credits` (2) are all live.
>
> **The Qt client is UNBLOCKED.** The standing rule was "do not code the client
> until A3's routes answer on the test box" — they answer. The bill screen
> (`plan-qt-sales.md` §1, three confirmed mockups) can be coded now, and the DC,
> DC-return and sale-return screens have their mockups drawn too.
>
> **Still not built, and it is the next backend work:** `HANDOVER_endpoints.md`
> **§11** (scan-to-load, barcode, the GST-cancel-first gate) and **§12** (void +
> the till session). Those are specification, not routes. §13 has three
> questions for the user that change their shape.
>
> **No SQL is waiting.** `37_drop_remaining_db_logic.sql` in the share root
> **must not be run** — see `../HANDOVER_2026-09-22.md` §3.
>
> The ✅ marks inside A1 below were written on 09-21 and are still correct;
> the prose around them ("two steps left") is stale — both steps are done.

**2026-09-21.** One page. The spine (A) is serial: each step is wrong without
the one above it. The GST track (B) runs beside it once A1 is done.

Everything below was verified against the test box **192.168.0.106 / ERP on
2026-09-21**, after each file. **A1 is all but done: `00_init`, `11`, `13`,
`14`, `15 §6`, `29`, `30`, `31`, `32`, `33`, `35` have all run and been
checked, and every partitioned table has 2024-25 … 2027-28.** Two steps left —
the year-roll and `prisma db pull`.

**The SQL changed substantially on 2026-09-21** — no functions or triggers in
the database, the transport band moved to one shared table, transition stamps
moved to `txn_status_log`. If you pulled these files before that, pull again:
see "Take fresh copies of what is still to run" below.

---

## A0. Two fixes first — ten minutes, before any SQL is run  ✅ DONE

| file | the defect | the fix |
|---|---|---|
| `schema/11_gst_provider.sql` | lines **1362–1363** call `public.fn_create_year_partitions`, which **does not exist on the box** (checked 2026-09-21: `pg_proc` count 0). | **Run `schema/00_init.sql` first.** That file defines exactly one thing — this function — and it is self-contained (lines 51–82, no dependencies). Then run `11` unchanged. |
| `schema/15_promotion_scheme.sql` §6 | references `sales.loyalty_coupon` | run `14_loyalty_coupon.sql` before it |

> An earlier draft of this table said to substitute `ensure_sales_doc_partitions`.
> **That was wrong**: it is created by `29`, which runs *after* `11`, so it would
> not exist at that point either. `00_init.sql` is the fix.
>
> `11` has **no `BEGIN`/`COMMIT`**, so its tables commit as they go and the
> failure lands mid-file. What happens next depends on the psql flags: with
> `-v ON_ERROR_STOP=1` it aborts and the **NIC sandbox seed block never runs**;
> without it, psql reports the error and carries on, leaving the tables and the
> seed but **no partition for `public.gst_api_log`** — and the first API log
> insert then fails with *no partition of relation found for row*. Either way,
> run `00_init.sql` first and neither happens.
>
> `gst_api_log` is the only LIST-partitioned table `11` creates.
> `fn_create_year_partitions` is generic: it walks every LIST-partitioned table
> in `public`, `sales` and `accounts` and creates that year's partition with
> `IF NOT EXISTS`, so running it is idempotent and safe on the tables already
> there.

---

## A1. P0 — run the SQL, in this exact order, one transaction at a time

Already live before today, do **not** re-run in full: `12_loyalty_scheme.sql`
and `15_promotion_scheme.sql` (only its §6 was missing, and that is now in).

```
 ✅ schema/00_init.sql            defines fn_create_year_partitions
 ✅ schema/11_gst_provider.sql    9 gst_* tables + view, seeded
 ✅ schema/13_loyalty_ledger.sql  verified: 0 functions, 0 triggers, recon
                                  indexes in, over-draw CHECKs out
 ✅ schema/14_loyalty_coupon.sql  same, verified
 ✅ schema/15_promotion_scheme.sql §6 only — promotion_usage, 4 partitions

 ✅ 29_sales_documents.sql        11 tables, 19 statutory rows.
                                  (its partition helper has since been removed
                                  from the file — see below)
 ✅ 30_sales_alters.sql           verified: sb_bill_mode, 0 band columns,
                                  5 crew kept, stamps gone, 3 GIN indexes,
                                  DCh/DCR/SRt, 4 user_menus flags, 62 settings

 ✅ 31_temp_credit.sql            acc_temp_credit, 4 partitions
 ✅ 32_gst_doc_tables_reconcile.sql  register's 18 ship/dispatch columns gone;
                                  gdw_/gde_ renamed and the 6 missing columns in
 ✅ 33_txn_transport_detail.sql   created, 4 partitions, STOCK_TRANSFER admitted
 ✅ 35_tender_void.sql            5 void columns, TndC + its series row,
                                  um_can_retender
 ✅ partitions — 2024-25 … 2027-28, all ten tables

 1  prisma db pull                ← THE ONLY THING LEFT IN A1

    The year-roll function is deliberately left alone — see below.
```

**A1 is all but done — two steps left.** Everything through `35` is on the box and checked. Schema
`sales` still holds **zero functions and zero triggers**; `sale_bill` has its
five crew columns, no transport band and no transition stamps; the salesman
columns are `uuid[]` with their GIN indexes on bill, order and quotation.

### `ensure_sales_doc_partitions` is gone from the files — deliberately

It never installed. Twice — from `29` and again from `31` — the
`CREATE FUNCTION` silently failed to appear while every other statement in the
same transaction committed. Rather than keep chasing a statement that does
nothing but look like it should, it has been **removed from both files**.

**It was redundant anyway.** `public.fn_create_year_partitions`
(`schema/00_init.sql`) is generic: it sweeps every LIST-partitioned
single-column table in `public`, `sales` and `accounts` — **31** of them today
— which is a superset of the nine `ensure_sales_doc_partitions` listed. That is
exactly how all ten partitioned tables got their 2024-25 … 2027-28 partitions
without anyone naming them.

So the year-roll is the generic call, which is step 1 below. Nothing is
missing and nothing needs creating.

### Use psql for the rest, not a GUI query tool

`31`, `32`, `34` and `35` all contain dollar-quoted blocks. A splitter that
quietly drops one function will keep doing it, and the next one may not be as
easy to spot as this was. **`psql -v ON_ERROR_STOP=1 -f <file>`** — `30` was
fine only because it contains no function or `DO` block at all.

After each file, the one-line check that would have caught this:

```sql
SELECT proname FROM pg_proc p JOIN pg_namespace n ON n.oid = p.pronamespace
 WHERE n.nspname IN ('public','sales','accounts','stock')
   AND proname IN ('ensure_sales_doc_partitions','fn_create_stock_partitions');
```

**Independent of the sales chain, run whenever after 29:**

```
 34_stock_drop_db_logic.sql   takes the last logic out of schema `stock`.
                              ⚠ RUN SECTION 1 ONLY for now — section 2 drops the
                              append-only guards and must wait until
                              StockPostingService is the only writer of
                              stock_ledger. Section 3 then needs
                              schema/stock/16_stock.sql re-run.
```

### Ordering traps — one still live, two now satisfied

1. **`05m_migrate_doc_register.sql` must run BEFORE `32`, or not at all.** It
   inserts the 18 `gdr_ship_*` / `gdr_dispatch_*` columns **by name**, and `32`
   drops them. Run after and it fails; it has been updated, but the ordering is
   the real rule.
2. ~~`33` after `30`~~ and ~~`32`/`33` after `29`~~ — **both satisfied**:
   `29` and `30` are in. Kept here because the reasons still hold if anyone
   rebuilds from scratch: `30 §1b2` settles which crew columns stay on
   `sale_bill` before `33` creates the band they are deliberately not in, and
   both `32` and `33` need `vehicle_master` / `transporter_master` from `29`.

### A failed run rolls back — 29, 30 and 31 are TRANSACTIONAL

`29`, `30` and `31` are each wrapped in `BEGIN; … COMMIT;`. **If one fails
partway, nothing is committed** — fix the cause and run it again. There is
nothing to clean up first. (An earlier draft of this file said to drop what it
made and restart; that was wrong.)

Use `psql -v ON_ERROR_STOP=1` so it aborts at the first error rather than
running on with every later statement failing. Even without it the transaction
protects you — the `COMMIT` turns into a rollback — but the output is far
harder to read.

**What they are NOT is re-runnable after a SUCCESSFUL run.** Their
`ALTER TABLE … ADD CONSTRAINT` statements have no `IF NOT EXISTS`, so a second
pass over a database where they already committed trips
(`fk_sdc_transporter` in 29, `ck_sb_bill_mode` in 30). That only matters if
you run one twice by accident.

`32`, `33` and `35` have no transaction wrapper and do not need one: every
statement is `IF NOT EXISTS` / `IF EXISTS`. All three were verified by running
them twice.

### Partitions — done, and here is the rule for next year

`29` creates its six tables as `PARTITION BY LIST` parents with no partitions,
so an INSERT before they exist fails with *no partition of relation found for
row*. That window has already closed: `fn_create_year_partitions` was run for
the open years and, being generic, picked the new tables up automatically —
2024-25 through 2027-28 on all ten partitioned tables.

The only thing left is the year-roll, which is step 1.

### Take fresh copies of what is still to run

Only **`34_stock_drop_db_logic.sql`** is still to run — everything else in the
chain is in. `schema/05m_migrate_doc_register.sql`,
`schema/05_acc_doc_register.sql` and the three `schema/stock/` files changed
today as documentation and reference; take them when convenient.

**What already landed** (2026-09-21): `00_init`, `11`, `13`, `14`, `15 §6`,
`29`, `30`.
Nine `gst_*` tables + `vw_gst_credential`, `gst_api_log` partitioned
2024-25 → 2027-28, seed took (2 providers NIC + Chartered/TaxPro, 23 endpoints,
32 field-map rows); the loyalty and coupon families with their partitions and
reconciliation indexes; `promotion_usage`; `29`'s 11 tables with 19 statutory
rows; and `30`'s alters — `sb_bill_mode`, the crew columns kept and the
transport band and transition stamps gone, three GIN salesman indexes,
`DCh`/`DCR`/`SRt`, the four `user_menus` flags, 62 settings. Verified after
each: **schema `sales` holds zero functions and zero triggers**, which is the
whole point of the rewrite.

**The year-roll is left as it is — a decision, not an omission.**
`ensure_acc_year_partitions` is maintained by hand, one `EXECUTE format(...)`
per table, and it now covers **34 of the 35** partitioned tables — everything
from this chain included. The one it does not name is `public.print_log`.

That is not a problem now: partitions exist for 2024-25 → 2027-28, so it would
only bite on the roll into 2028-29, when the first print would fail with *no
partition of relation found for row*. The user has taken the view that it is
not worth acting on today, and the partition scheme may well be reshaped before
then.

Recorded here so it is a known, dated choice rather than something discovered
in 2028. If it is ever revisited, the whole hand-maintained body can be
replaced with:

```sql
PERFORM public.fn_create_year_partitions(v_year);   -- sweeps all 35
PERFORM stock.fn_create_stock_partitions(v_year);   -- the stock schema is not swept
```

all 35 key on a `*_acc_year` column, so the generic sweep cannot build a
partition on the wrong key.

**Every file here was dry-run on a schema-only clone of the live database
(`erp_dry`) before it was handed over**, and `29` and `30` have since run on
the box itself and been verified column by column.

**A1 is done when** this returns nothing MISSING and a 1 (`acc_temp_credit`
arrives with `31`, so it is the one still expected to be missing right now):

```sql
select t, coalesce(to_regclass(t)::text,'MISSING') from unnest(array[
  'sales.sale_dc','sales.sale_dc_item','sales.sale_return','sales.sale_dc_return',
  'accounts.acc_temp_credit','sales.loyalty_ledger','sales.loyalty_coupon',
  'sales.promotion_usage','public.gst_provider','public.gst_api_log']) t;
select count(*) from information_schema.columns
 where table_schema='sales' and table_name='sale_bill' and column_name='sb_bill_mode';
```

**A trap that was in the files until 2026-09-21, now fixed — worth knowing it
existed, in case he is working from an older copy.** `13` used to `ALTER TABLE
IF EXISTS sales.sale_return ADD COLUMN sr_loyalty_reverse_points` (and
`sri_loyalty_points` on the line). But `sale_return` is created by `29`, four
files LATER, and `ALTER TABLE IF EXISTS` on a table that does not exist yet is
a **silent no-op** — so those two columns would simply never have appeared, and
nothing would have said so. `29` now declares both in the `CREATE TABLE`
itself, and `13`'s two ALTERs are gone. (`29` also had the header column as
`sr_loyalty_reversed numeric(15,2)`; points are `numeric(18,4)` everywhere else
in `13`, so it is now `sr_loyalty_reverse_points numeric(18,4)`.) The bill side
was never affected — `sale_bill` and `sale_bill_item` already exist, so `13`'s
ALTERs on those land normally.

**One more, minor:** neither `29` nor `30` is re-runnable after a partial
failure — their `ALTER TABLE … ADD CONSTRAINT` statements have no
`IF NOT EXISTS`, so a second pass trips (`fk_sdc_transporter` in 29,
`ck_sb_bill_mode` in 30). On a fresh database they run clean. If one fails
halfway, drop what it made and start it again rather than re-running on top.

**The five crew roles are now identical on all four goods-moving documents**
— bill, sale return, challan, DC return:

| role | shape | why |
|---|---|---|
| `salesman_id` | `uuid[]` | a document is credited to a team; a third is possible |
| `supervisor_id` | `uuid` | one person oversees a load. **Was on the bill only**; added to the other three |
| `driver_id` | `uuid` | a vehicle has one |
| `loadman_id` | `uuid[]` | an open-ended crew |
| `vehicle_id` | `uuid` | one |

`sale_order` and `sale_quotation` deliberately carry **none** of them — the
rule is already stated in `06`: *"Driver / loadman / supervisor / vehicle are
NOT carried: despatch is the bill's fact."* The crew belongs to documents that
MOVE GOODS.

**The singular ones now carry foreign keys** (`fk_*_driver`, `fk_*_supervisor`,
`fk_*_vehicle`) — 29 for the three new documents, 30 §1b2 for `sale_bill`,
which never had them. Each is picked from a dropdown, so a stale id is
otherwise invisible until a despatch report is run. The two ARRAY roles cannot
have one and are validated by the service instead (plan-backend §3.7). Adding
the bill's is safe: 0 rows on the test box have a driver or supervisor set and
there are no orphans.

**`35` — re-tender, the till's commonest correction.** A customer says UPI, it
fails, they pay cash — many times a day. The sale was right, only the payment
changed, and today that forces an edit or a cancel of a posted bill. `35` adds
`td_is_voided` / `td_void_reason` / `td_voided_on` / `td_voided_by` /
`td_replaces_id` to `acc_tender_detail`, the `TndC` **contra** voucher type,
and the `um_can_retender` right. The failed row is VOIDED, never deleted — a
till review has to see it, and `td_is_deleted` would say "keyed wrong" when the
truth is "payment failed". Route: `POST /bills/retender` (HANDOVER §2.13).
**It works on a bill that already has an IRN**, because the tender is not on
the e-invoice, not in GSTR-1 and not on the e-way bill.

**`32 §2b` also DROPS the register's 18 ship / dispatch columns.**
`acc_voucher_doc_register` kept three address blocks; only `gdr_party_*` (13
cols) is its own work, because **GSTR-1 B2B needs the recipient**. Ship-to and
dispatch-from appear in **neither GSTR-1 nor GSTR-3B** — they exist only to
build the e-invoice and e-way payloads, and since `33` those live in
`txn_transport_detail`. What makes it safe is the lock rule: the band **freezes
when an IRN or e-way bill is generated**, so it IS the declared copy, not a
working copy that may have moved on.

`33`'s `ck_ttd_doc_type` therefore widens per module and **already admits
`STOCK_TRANSFER`** — which closes a gap that exists today: `stock_voucher`
carries only `svh_from_godown_id` / `svh_to_godown_id`, so an inter-branch
transfer needing an e-way bill had **nowhere to record a vehicle at all**.
`PURCHASE_BILL` / `PURCHASE_RETURN` join when those modules are built.

**`schema/05m_migrate_doc_register.sql` inserted all 18 by name** and has been
updated; it must run BEFORE `32` or not at all.

**`32` — the two GST document tables drifted and nobody could see it.**
`accounts.acc_voucher_doc_einvoice` and `_ewaybill` were deployed from an older
revision of `05_acc_doc_register.sql` and then diverged. `05` creates them with
`CREATE TABLE IF NOT EXISTS`, so **re-running it does nothing** — the drift is
silent and permanent. Both tables hold 0 rows, so `32` is free. It fixes three
kinds of difference:

* **spelling** — `*_canceled_on` (deployed) → `*_cancelled_on` (05, and what
  every other table in the schema uses). Two spellings of one word in one
  schema is a bug waiting for whoever types the other one.
* **naming** — `*_updated_on/_by` → `*_modified_on/_by`, the house audit pair.
* **missing** — `*_tenant_id`, `*_attempt_count`, `*_last_attempt_on`, and
  `*_request_payload` / `*_response_payload` (jsonb: what we actually sent the
  portal and what it said back — the only answer to "what did we declare on
  that date" once a document has been amended). On the e-way row also
  `gdw_transport_doc_no` / `gdw_transport_doc_date` — Part-B's
  `transDocNo`/`transDocDate`, the LR, which `gsp_flow.md` lists as a field we
  send and which had nowhere to land — plus `gdw_vehicle_id` (FK to
  `vehicle_master`, hence "after 29") and `gdw_extended_upto`.

`gcc_id`, `gpv_code`, `generated_via`, `irp_code` and `next_attempt_on` are
left alone: they are on the deployed tables and absent from `05` because `11`
added them. `05` is the file that is behind, not the database.
`32` is idempotent — verified by running it twice.

**The inward direction rule is now written** (`gst/gsp_flow.md` §6a): the
register's two address blocks are POSITIONAL — `gdr_dispatch_*` is always the
FROM and `gdr_ship_*` always the TO. On a return the **pickup block is the
dispatch block**, which is why `*_pickup_*` exists on both returns instead of
reusing the bill-to snapshot.

**The split: crew stays on the document, the band is shared.**

| on all four documents (`sb_`/`sdc_`/`sr_`/`sdr_`) | in `txn_transport_detail` (33) |
|---|---|
| `driver_id`, `supervisor_id`, `loadman_id uuid[]` | `from_*` / `to_*` addresses + `direction` |
| `vehicle_id`, `vehicle_no` | `transporter_id / name / gstin` |
| | `lr_no`, `lr_date`, `distance_km`, `transport_mode` |

The crew and the vehicle are "who of ours did it" and exist **whenever goods
move, even with no consignment** — a home delivery has a driver and a van and
no transporter, no LR and no e-way bill. Putting them in the band would force a
row there just to record who drove, which is the common case for a POS company.
The band holds the addresses and the carriage and is genuinely absent for POS.

**They are NOT duplicated.** One home per fact: the e-way builder reads the
vehicle off the document, which it has loaded anyway, and the rest from the
band. Two copies of a vehicle number diverge and no report knows which to
trust. `30 §1b2` therefore does **no migration at all** — the bill's five
simply stay, and gain the foreign keys they never had.

The price of repeating five columns on four tables is drift, and it has already
happened here (`supervisor_id` was on the bill alone, `vehicle_id` missing from
the DC return). **A conformance test asserts all twenty exist** —
plan-backend §12, and it passes.

**`33` — the transport BAND is ONE table now, not four copies.**
`public.txn_transport_detail` carries ship-to / dispatch / pickup / receive /
transporter / vehicle / LR / distance / driver / supervisor / loadman for
**every** goods-moving document, keyed `(ttd_doc_type, ttd_doc_id,
ttd_acc_year)`, 1:1, partitioned by year. `sale_bill`, `sale_dc`,
`sale_return` and `sale_dc_return` carry **none** of it.

*Why, and it is not storage.* The band was 22 columns × 4 documents = 88
definitions, and they had **already drifted** — `vehicle_id` was missing from
the DC return and `supervisor_id` existed only on the bill. Every future e-way
field would have been four ALTERs and the fourth gets forgotten. The NULL
argument was measured and does not hold: 22 NULL columns cost ~**8 bytes a
row**, so half a million POS bills a year would have cost about 4 MB.

*FROM and TO are positional; `ttd_direction` says what they mean.* OUTWARD
(bill, challan): FROM is our godown, TO is the party. INWARD (both returns):
FROM is the party's pickup, TO is our godown. So the register maps across with
no per-document branch — `gdr_dispatch_*` ← `ttd_from_*`, `gdr_ship_*` ←
`ttd_to_*`. Same pattern as `txn_charge_detail` and `txn_status_log`, and it
pays the same price: a polymorphic child cannot FK to four partitioned
parents, so the service is what stops a row outliving its document.

**`30 §1b2` keeps the bill's five and drops only the band.** No data moves, so
the 24 test-box bills carrying a loadman array are untouched. It still drops
the ship / dispatch / transport block defensively, in case an earlier draft of
`30` ran somewhere and left a stale second copy beside a `txn_transport_detail`
row.

**Both returns still have the full band — it just lives in `33` (29).** `sale_return` had only
`sr_driver_id` + `sr_vehicle_no`, and `sale_dc_return` had no driver, vehicle
or crew at all — yet returning goods travel and an inward e-way bill needs the
same Part-A/B. Both now carry the challan's band with the direction reversed:
a **PICKUP** block (`*_pickup_addr_id/name/addr/place/pin/phone/stcd/gstin` —
where the customer handed the goods back, which is the e-way FROM and is not
the same thing as the bill-to snapshot), **RECEIVE** (`*_receive_godown_id`,
`*_receive_branch_id`) and transport (mode, transporter id/name/gstin, vehicle
id + no, LR no/date, distance, driver, `loadman uuid[]`). `sr_receive_godown_id`
is NULLABLE — a header default only, because `sri_godown_id` on the line is
authoritative and a return may split across godowns; `sdr_receive_godown_id`
stays NOT NULL because un-billed goods come back off one challan into one
place. Both returns also get `fk_*_transporter` / `fk_*_vehicle` in the same
deferred ALTER block as the DC, and a `ck_*_transport_mode` CHECK.

**Salesman is `uuid[]` on all five documents (30 §1c, 29).** `sale_bill`,
`sale_order` and `sale_dc` already were; `sale_return` (29) and
`sale_quotation` (30 §1c, an `ALTER … TYPE`) now match. **Dropping
`fk_sq_salesman` is part of that ALTER and is not optional** — PostgreSQL
cannot put a foreign key on the elements of an array, which is why the bill
and the order never had one. That integrity moves to
`assertSalesmen()` in the service (plan-backend §3.7): every id must resolve
in `employee_master`, be this company's, not be deleted, not repeat, and the
array must be null rather than `{}`. **Nothing in the database will catch a
bad salesman id, so this check is load-bearing.**
Three GIN indexes come with it (`ix_sb_/ix_so_/ix_sq_salesman_gin`) because a
btree cannot answer array containment; the query shape is
`WHERE sb_salesman_id @> ARRAY[:salesman]::uuid[]`.
**Element 1 is the primary** — `abl_salesman_id` stays a single indexed uuid
and takes `sb_salesman_id[1]`, so outstanding-by-salesman keeps its btree and
credits only the primary. Line-level salesman (`sbi_`/`soi_`/`sdi_`/`sri_`)
stays a single uuid. `loadman` stays `uuid[]` with no primary convention — an
open-ended crew that nothing joins on.

**Transition stamps are gone from the headers (30 §1b, 29's new tables).**
`sb_posted_on`, `sb_cancelled_on`, `sb_cancelled_by`, `sb_cancel_reason` are
dropped from `sale_bill`, `sq_cancelled_on` from `sale_quotation`, and
`sale_dc` / `sale_dc_return` / `sale_return` never carry them. When, who and
why a status changed live in `public.txn_status_log` — which already has
`tsl_changed_on`, `tsl_changed_by` and `tsl_remarks` (the cancel reason), plus
from-status, device and session. A log row is APPENDED, and two devices
appending cannot conflict the way two devices UPDATEing one header row can,
which is what matters on the push. **`*_posted_voucher_id` STAYS** — it is a
relationship to the voucher, not an event, and it is how a document reaches
its voucher without scanning the log. `*_status` stays too, for filtering, the
CHECK and the badge. Client and grid SQL already updated to match.

**Two things to know while running A1:**

* **`11` retires two modules he already has.** `fixed.gsp_provider_master`
  (2 rows) and `fixed.gsp_company_service` (1 row) hold GSP config in
  plaintext today and are served by `/api/v1/gsp-provider-masters` and
  `/api/v1/gsp-company-services`. `11` replaces them with the
  `public.gst_provider*` family (provider · service · account · endpoint ·
  field map · error map · company credential · auth session · api log, with
  encrypted credentials). Migrate the 3 rows, then retire the two modules at
  **B1** — not before, or the GST screens lose their config.
* **`13` and `14` create no function for three behaviours you might expect.**
  Drawing points out of a wallet, the nightly points expiry and the nightly
  voucher expiry are `LoyaltyLedgerService.consume()`, `.expiryRun()` and
  `.couponExpiryRun()`, built at **A2**. Each algorithm is written out in
  `plan-backend-sales.md` §3.4 and again beside its tables in the `.sql`.
  Everything else in the chain — the recompute triggers, the read functions,
  `vw_gst_credential` and the partition helpers — stays in
  the database on purpose. The rule and the full object-by-object inventory
  are at the top of `README.md`, "Where logic lives".
* **Six over-draw CHECKs are deliberately absent** from `13`, `14`, `29` and
  `31` — wallet, lot, voucher, batch, challan line and temp credit. Under
  offline sync two tills can each spend the last of one resource, and the batch
  that syncs second must be recorded rather than refused. Each is now a partial
  index feeding a reconciliation report, and the refusal lives in the posting
  service at **A2/A3**. `offline-sync-invariants.md` is the reasoning; the
  reconciliation report itself is not built yet.
* **`30` adds the four `user_menus` flags** (post / cancel / amend /
  override). Granting a menu is a **full replace** — whoever writes those
  rows must send all ten flags or the existing ones are silently revoked.

---

## A2. P1 — the shared services  (`src/modules/sales/posting/`)  ✅ DONE

> Verified 2026-09-21 and re-checked 2026-09-22 in the read-only snapshot: all
> eight services exist under `src/modules/sales/posting/` (11 files) and
> `src/modules/stocks/posting/` (5); `FOR UPDATE` with the only `SKIP LOCKED`
> mentions being comments explaining why not; `recomputeMembers` /
> `recomputeLots`; `deterministicUuid` with the `|coupon|` segment;
> `assertSalesmen` in `sales.guards.ts`; `test/stock-ledger-single-writer.e2e-spec.ts`
> written; stock-voucher migrated onto `StockPostingService`; and the freeze
> guard tests `sml_doc_datetime`, **not `now()`**.
>
> The engine's contract is now written down: `schema/stock/17_sml_apply_contract.md`.

**`A2_BRIEF.md` is the one page for this phase** — reading order, the services,
the nine rules that must survive, and what done looks like. Start there.

`StockPostingService` (the stock engine's phases lifted), `SalesPostingService`
(legs by role), `DocRegisterService`, `LoyaltyLedgerService`,
`PromotionUsageService`, `StatutoryService`, `ChargeCarryService`,
`sales.guards.ts` — then **migrate the existing stock-voucher module onto
`StockPostingService`** and get its tests green.

Nothing user-visible comes out of this step. Everything below it is wrong
without it, because every document posts through these same services.

Spec: `plan-backend-sales.md` §3.

---

## A3. P2 — the bill and the order   ✅ **DONE 2026-09-22 — the Qt client is unblocked**

| module | routes |
|---|---|
| `bills` | `/validate` `/post` `/cancel` `/amend`, `/delete` narrowed to DRAFT, `/open-sources`, `/party-context`, `/delivery-status`, `/update-remarks`, **`PUT /transport`** (§2.12), **`GET /tender-context`** (§2.12a), **`POST /retender`** (§2.13) |
| `sale-orders` | `/post` `/cancel` `/amend` |
| `temp-credits` | `/open` + the receipt hook |
| grids | **133** POPUP - RECENT BILLS FOR RE-TENDER (this device + session, today). A list is a grid, not a `/list` route |
| GSP | enqueue stubs only (the worker is track B) |

Build to `HANDOVER_endpoints.md` §2, §3, §7 — request and response JSON are
written out there, and the Qt screen is built to the same text. Every
response carries the `posting` / `locks` / `rights` blocks of §1.

**Tell me the day these answer on the test box.** The bill screen
(`plan-qt-sales.md` §1, three confirmed mockups) is coded straight away; it
cannot start before.

---

## A4. P3 — delivery challan + DC return   ✅ **DONE 2026-09-22**

`/api/v1/delivery-challans` (`create` `validate` `post` `cancel` `amend`
`delete` `get` `open-for-bill` `convert-purpose`) and `/api/v1/dc-returns`
(`create` `post` `cancel` `delete` `get` `open-lines`).
HANDOVER §4, §5 · flow §4.

## A5. P4 — sale return + settlement   ✅ **DONE 2026-09-22**

`/api/v1/sale-returns`, with the three settlements (CASH / ADJUST / ADVANCE).
HANDOVER §6 · flow §6. Independent of A4 — either order, or both at once.

---

## B. GST — the second track, any time after A1

| step | delivers | run it |
|---|---|---|
| **B1** | config module + crypto + `GspClientService` + payload builders + unit tests; **`/gst/gstin/lookup` live** | right after A1 — it is the first user-visible win (the GSTIN fetch buttons) and it retires the `fixed.gsp_*` pair |
| **B2** | bill IRN + EWB document routes, the worker, sandbox tests; the bill's post enqueues | with **A3** |
| **B3** | DC / DC-return e-way bill | with **A4** |
| **B4** | credit-note IRN + return e-way bill | with **A5** |

`gst/HANDOVER_endpoints.md` is the wire; `gst/plan-backend-gsp.md` the build.
Credentials: TaxPro **sandbox**, from the environment, never in a file.

---

## Not his — so he does not wait on it, and does not build it

Menus 251–257, grids 113–132, dropdowns 55–57 and ui_tables 38–47 carry **no
SQL**. They are registered through the API as each Qt screen is built (P5,
this side). The only registration in his files is `user_menus` and
`app_setting_def`, both inside `30` and `31`.

---

## If there is time for one thing only

**A1 → A3.** A0 and the first two files of A1 are already done. Finishing A1
is an afternoon and unblocks every plan; A3 is what lets the sale bill — the
screen the business actually runs on — be built.

Stop and tell me when A3's routes answer: the Qt bill screen is coded to
`HANDOVER §2` the same day, and three mockups are already confirmed.
