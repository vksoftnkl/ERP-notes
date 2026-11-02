# sales — the six sales transactions: Quotation, Sales Order, Sale Bill, Sale Return, Delivery Challan, DC Return

**2026-09-20.** Stock posting, accounts posting, loyalty, promotion,
e-invoice, e-way bill, and Create / Post / Amend / Cancel in the receipt
pattern, for all six — with the two decisions taken (**COGS perpetual**,
**one `sale_bill` with `sb_bill_mode` POS | WHOLESALE**) and the fifty
questions of the analysis answered "default".

**The whole thing in one line:** the challan moves the goods (stock rows +
DR COGS / CR Stock in Hand), the bill raises the debt (DR Party / CR Sales
+ output GST, then tenders against the party), a bill against a challan
raises only the debt, and every return is the same rows with the sign
reversed — never a delete.

## Where logic lives — SQL or NestJS

Every `.sql` file in this chain is read by someone deciding whether to put a
behaviour in the database or in the service. One answer, applied everywhere:

> **The database keeps what must be true NO MATTER WHO WRITES** — CHECK
> constraints, generated columns, cache-recompute triggers, grant-boundary
> views, partition DDL helpers, and pure read functions that several callers
> must not disagree about.
>
> **NestJS owns what happens INSIDE A UNIT OF WORK** — allocation, posting,
> settlement, scheduled jobs. Anything invoked as one step of a larger
> transaction the service already owns.

**Why it matters here.** Both failure modes are already in this codebase. Put
a unit of work in the database and it silently forks: the stock posting engine
(`19`, `20`) was written as `.sql`, NestJS reimplemented it, and the two
drifted apart with nobody the wiser — the `.sql` is now a spec, not what runs.
Put a write-path rule in the service and its constraints go decorative:
`ck_avh_balanced` cannot catch a voucher with no legs, because the totals it
checks are stamped by the writer instead of derived.

**The whole chain, object by object.** These are every function, trigger and
view the P0 files create — nothing else in `11`–`15`, `29`, `30`, `31` has a
side to be on.

| file | object | side | why |
|---|---|---|---|
| `11` | `vw_gst_credential` | **SQL** | it IS the grant boundary: revoke on `gst_company_credential`, grant on the view. In TS the secrets are readable again |
| `13`, `14` | **everything** — FIFO consume, both expiry sweeps, all balance recomputes, their triggers and the three read functions | **NestJS**, all of it | these two files now contain **no function and no trigger**. See the offline rider below |
| `29` | statutory resolver | **NestJS** `StatutoryService.limit()` | the offline till must apply 269ST and the e-way floor with no server in reach, so the resolution order has to be in TypeScript anyway — a second copy in plpgsql is two implementations of one statute |
| `29`, `31` | `ensure_sales_doc_partitions` | **SQL** | DDL, and the one deliberate exception — see below |

`13` and `14` create no function for the three NestJS ones. Each keeps the
algorithm as a comment block beside its tables — the FIFO walk, the row
shapes, the deterministic run ids and the idempotency — so the rule and the
thing it governs are read together. `plan-backend-sales.md` §3.4 is the build
and `sales_posting_flow.md` §5.8 the contract.

**The offline rider, and it overrides the rule above for anything on the sync
path.** This is an offline app: the till posts offline and pushes to the cloud
on reconnect. A trigger fires on the *server*, during that push, on rows
written hours earlier — in a language this team does not debug, per statement
of a batch so a failure points at the batch and not the row, and whether or not
the service wanted it. So on any table an offline device writes, **nothing
executes in the database**: no functions, no triggers. `13` and `14` were
stripped completely on 2026-09-21. What is left there is declarative and cannot
run at the wrong moment — `GENERATED ALWAYS` columns, CHECKs, unique indexes,
foreign keys, and the reconciliation indexes.

A trigger could not be forgotten and a service call can, so the replacement is
structural: **one private method is the only code in the codebase that writes
each of those tables**, and it recomputes before returning. A test asserts it.
`plan-backend-sales.md` §3.4a–c is the contract.

**The two things still written in SQL, and why they are not exceptions to the
offline rider.** Neither runs when a device writes data.

* `ensure_sales_doc_partitions` (`29`, restated in `31`) is **DDL**. It is
  called deliberately, by a person or a year-roll job, and it creates tables.
  It cannot fire during a sync push because nothing calls it there. The live
  database is already built this way — `ensure_acc_year_partitions`,
  `fn_create_year_partitions` and `stock.fn_create_stock_partitions` are all
  database functions — so removing only ours would make it the odd one out
  without making anything safer.
* `vw_gst_credential` (`11`) is a **view**, and it is the GRANT boundary:
  revoke `SELECT` on `gst_company_credential`, grant it on the view. Replace it
  with a TypeScript query and the read-only role can read the credentials
  again. It executes nothing and decides nothing.

Everything else in `11`–`15`, `29`, `30`, `31` is tables, constraints, indexes
and comments.

**A guard is not the same thing as logic.** Where a behaviour lives is this
rule; which *constraints* may still enforce it once writes arrive late and out
of order is the companion note, `offline-sync-invariants.md`. Short version: a
CHECK may guard one row's own arithmetic, but not a contended shared resource —
six over-draw constraints across `13`, `14`, `29` and `31` became recorded
over-draws plus a partial index, because two offline tills can each spend the
last of one wallet and the batch that syncs second must be recorded, not
refused. The recompute triggers stayed, and became statement-level.

**Three details that must not be lost in the port.** `consume()` takes
`SELECT … FOR UPDATE` on the wallet and *never* `SKIP LOCKED` — two tills
redeeming one wallet must serialise, and in a supermarket that is a real race;
the lock has to sit in the caller's interactive `$transaction`, or the bill's
post and the redeem land in different transactions and it protects nothing.
Both sweeps keep their deterministic run ids — `md5(company|date)` for points,
`md5(company|coupon|date)` for vouchers, which must not collide — so a re-run
on the same date is rejected by the unique index instead of quietly writing a
second set.

---

**Building A2 (the shared services)? `A2_BRIEF.md` is the one page** — the
reading order, the nine rules that must survive, and what "done" looks like.

**Handing this to Prathap? `RUN_ORDER.md` is the one page** — what to run,
in what order, what each step unblocks, and the two fixes that come before
any SQL. Verified against the live test box 2026-09-21.

**Building the endpoints? Start with `HANDOVER_endpoints.md`** — every route with
its exact request / response JSON, the `posting / locks / rights` blocks, the
warning shape and the error codes; the client is built to the same text.

**Read in this order:**

| file | what it is |
|---|---|
| `sales_txn_plan.png` | **start here** — the picture: document flow, posting matrix, lifecycle + engine, rules, the two decisions, build order P0–P5 (`.py` beside it regenerates it) |
| `analysis_2026-09-20.md` | the analysis that came first: where each document stands live (§1), the bill posting defect list (§2), stock (§3), accounts (§4), register (§5), loyalty / promo (§6), 3.0 facts for bill / return / loyalty / e-invoice only (§7 — the challan takes nothing from 3.0), Tally / GoFrugal behaviours (§8), compliance per company (§8b), the final sweep (§8c), and the 50 questions (§12) |
| `fsales-audit.md` | 3.0's `fsales` (59 flags) behaviour by behaviour against the plan — ✅ landed / ➕ added by the audit / ⏭ deferred by name / ✖ not carried; decisions D53–D59 |
| `offline-sync-invariants.md` | which guards may be CHECKs once posting goes offline — the six over-draw constraints that became reconciliation indexes, and why the recompute triggers stayed |
| `sales_posting_flow.md` | **the spec** — every table, row and leg each document writes at post / cancel / amend; the worked bill (§5.2); the three return settlements; charges across part deliveries (§7); the law per company (§8); the guard order (§9); the offline contract (§10); rules as checks (§11) |
| `../29_sales_documents.sql` | **P0, "I will run with Prathap"** — `sale_return(_item)`, `sale_dc(_item)`, `sale_dc_return(_item)`, `transporter_master`, `vehicle_master`, `statutory_limits` + the shipped pack, `company_aato`, `acc_party_tax_certificate`, the partition helper |
| `../31_temp_credit.sql` | **P0** — `accounts.acc_temp_credit`: the WHO behind a TEMP_CR tender (name / mobile / place / days) riding on the bill's balance row; partition helper re-stated; three settings |
| `../30_sales_alters.sql` | **P0** — ALTERs on `sale_bill(_item)`, `sale_order`, `txn_charge_detail`; the DC pair in three vocabularies; four date columns; unit UQC, Form 60, AATO class, `user_menus` post / cancel / amend / override; roles COGS / INVENTORY / LOYALTY_REDEMPTION / SCHEME_DISCOUNT + ledgers + map; voucher types DCh / DCR / SRt + series; 21 settings |
| `A2_BRIEF.md` | **A2 in one page** — read-in-this-order, the services to build, the nine rules, and the finish line |
| `RUN_ORDER.md` | **the order to execute** — A0 fixes, A1 P0 SQL, A2 services, A3 the bill (unblocks the client), A4 challan, A5 return; the GST track B beside it |
| `HANDOVER_endpoints.md` | **the wire** — bills (create extended, validate, post, cancel, amend, delete, get, open-sources, party-context, delivery-status, update-remarks), orders, challans, DC returns, returns, temp credits; error codes; what the server must ignore |
| `plan-backend-sales.md` | what Prathap builds — shared `StockPostingService` (the engine's phases lifted), `SalesPostingService` (legs by role), `DocRegisterService`, loyalty / promotion / statutory / charge-carry services, the guards; bill `/validate /post /cancel /amend`, order `/post /cancel /amend`, the three new modules; tests; phases P0–P4; what not to port |
| `plan-qt-sales.md` | the Qt client — ids to reserve, the bill screen's changes (verbs, sources, charges strip, party panel, transport band, identity, badges, loyalty tender, weight barcodes), the DC / DC return / return screens, masters, settings, tests, build order (P5) |
| `plan-react-sale-bill.md` | **the React port of the sale bill (sales entry), REV 2, 2026-09-25**: the whole screen, tender, set-offs, shipping, re-tender, holds, the four lists; supersedes `../react-plans/plan-react-sale-bill.md` and, for the bill, `plan-react-sale-tender.md` |
| `../gst/` | the GSP layer (e-invoice / e-way / GSTIN lookup) — its own README, flow and two plans |

**Run order for P0:** `schema/11_gst_provider.sql`, `13_loyalty_ledger.sql`,
`15_promotion_scheme.sql` §6 (promotion_usage), then `29`, `30`, `31`, then
`SELECT ensure_sales_doc_partitions(y)` for every open year, then
`prisma db pull`.

**Dry-run 2026-09-20:** 29 and 30 ran clean, in order, on a scratch clone of
the live schema (`erp_dry` on localhost, built from a schema-only dump of
192.168.0.106/ERP); `ensure_sales_doc_partitions('2026-2027')` made all
nine partitions; the resolver probes in 29 §7 answered as designed. Checked
read-only against the LIVE chart: the four ledger seeds resolve their
groups (Direct Expenses / Stock-in-Hand / Indirect Expenses), none of the
four ledgers exists yet, and the series clone finds the one Bil row
(MAIN). Not run on the live database — that is P0, Prathap's.

**Two defects in the OLDER prerequisite files, found by the same dry run:**
`schema/11_gst_provider.sql` lines 1362-1363 call `fn_create_year_partitions`,
which does not exist on the box — **run `schema/00_init.sql` first**, which
defines exactly that one function and nothing else (NOT
`ensure_sales_doc_partitions`, which comes from 29 and so runs too late); `schema/15_promotion_scheme.sql` references
`sales.loyalty_coupon`, so `14_loyalty_coupon.sql` must run before it.

**Ids confirmed 2026-09-20** (`plan-qt-sales.md` §0, `../gst/plan-qt-gsp.md` §0).
**Mockups, one transaction at a time** (user: mockup → confirm → code, per
transaction). Drawn so far:

| picture | what it shows | state |
|---|---|---|
| `sale_bill_additions_ui_mockup.png` | the bill screen's additions (§1) | ✅ confirmed 2026-09-20 |
| `sale_bill_tender_additions_ui_mockup.png` | the Settle popup's additions (§1.7) | ✅ confirmed 2026-09-20 |
| `temp_credit_list_ui_mockup.png` | menu 257, the who-owes list (§1.12) | ✅ confirmed 2026-09-20 |
| `delivery_challan_ui_mockup.png` | **the whole new DC screen** (§4) | ⏳ awaiting confirmation (2026-09-21) |

DC return (menu 251) and sale return (menu 13) follow, one confirmation at a
time. Every picture is one `*_ui_mockup.py` beside its `.png`, importing
`mock_lib.py`.
