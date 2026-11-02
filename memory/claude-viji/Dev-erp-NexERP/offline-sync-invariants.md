---
name: offline-sync-invariants
description: Which guards may be CHECKs once posting goes offline — the six over-draw constraints converted to reconciliation indexes, and why the recompute triggers stayed and got batched
metadata:
  node_type: memory
  type: project
---

Decided 2026-09-21 when the user asked why `fn_lmb_recompute` was still in the
database "if we run in backend as nestjs" and then said **"its may be a
offline -> online sync thats why im asking"**. Full note:
`share/sales/offline-sync-invariants.md`. Companion to
[[sql-vs-nestjs-rule]] — that one says where behaviour lives, this one says
which *guards* may still enforce it.

**The rule:** a `CHECK` may guard **one row's own arithmetic**, never a
**contended shared resource**. The test: *could two tills, both offline, each
legitimately consume the last of this?*

**Why:** two tills offline, one 600-point wallet, each redeems 500. Both bills
are real and the customers have left with the goods. At sync the second batch
would hit `ck_lmb_balance >= 0` and **abort** — refusing a sale that already
happened, and failing the whole batch. Second reason: reversals are rows of the
same type with the opposite sign, so if a reversal syncs before the row it
reverses, a bucket is transiently negative through nobody's fault. Any CHECK
over a running aggregate of a ledger that arrives in fragments eventually fires
on correct data. Also: §8c.3 promises "adding offline needs no ALTER", and each
of these would have had to be dropped on that day.

**The guard split three ways:** policy → the posting service, refusing at post
while online; record → the database (the balance columns are all
`GENERATED … STORED` and simply go negative); detect → a reconciliation report
off partial indexes.

**Six constraints converted**, each replaced by a partial index:
`13` `ck_lmb_balance` + `ck_lmb_points` → `ix_lmb_overdrawn`;
`13` `ck_lld_lot_not_overdrawn` → `ix_lld_lot_overdrawn`;
`14` `ck_lcp_used` (narrowed to `ck_lcp_face`) → `ix_lcp_overdrawn`;
`14` `ck_lcb_counts` → `ix_lcb_overdrawn`;
`29` `ck_sdi_qty` (narrowed to the challan's own qtys) → `ix_sdi_overdrawn`;
`31` `ck_atc_amounts` (narrowed to `credit > 0`) → `ix_atc_overdrawn`.
Kept, because they are single-row arithmetic: `ck_sri_return_qty`,
`ck_lct_amount_nonzero`, `ck_lcb_min_bill`, the sale-return settlement sum.

**The recompute triggers STAYED — offline is the reason.** They recompute
rather than increment, which is idempotent under replay, order-independent and
self-healing after a partial batch; an incrementing cache double-counts on a
re-sent batch, and an application-maintained one must be remembered on all nine
write paths into the ledger. They are now **statement-level with transition
tables** (PG 13+; verified 18.1): `13`'s recomputes became set-based
(`fn_lmb_recompute(uuid[])`, `fn_lld_recompute_lot(uuid[], char(9)[])`, one-id
wrappers kept), `14`'s are called once per DISTINCT id. Three triggers per
table because a transition-table trigger cannot serve more than one event.
**Trap the set-based form introduced:** `GROUP BY` returns no row for a member
whose last ledger row was deleted, where the per-member aggregate returned
zeroes — a second UPDATE zeroes those explicitly.

**Verified on `erp_dry`** (schema-only clone; fixtures needed FK drops inside a
rolled-back transaction, and `lld_cust_id` is NOT NULL, and REDEEM needs a
tender so the test used GIFT): 3 EARN in one statement → 600; a 150 spend
across two lots in one statement → balance 450 and **next_expiry advanced**,
which only comes out right because the lot recompute runs before the wallet
recompute in the same statement; a 600 overspend → balance −150 **recorded, not
refused**; both reconciliation predicates found their row. 13, 14, 29 and 31 all
rebuild clean from scratch.

**`19`–`28` swept 2026-09-21 (§7 of the note).** The accounts/posting files
(`19_posting_roles`, `20_tax_rate_master`, `21_accounts_posting_setup`,
`22`–`28`) are **CLEAN** — 16 objects, all in 19/20/21, all master-data: three
guard triggers on `tax_rate_ledger` / `charge_master` / `acc_ledger_map`, five
`LANGUAGE sql STABLE` resolvers, three views, and `fn_ledger_tax_bridge` (the
expand→migrate→contract shim, which MUST stay in the DB because its job is to
catch writes from code that does not know about `led_tax_id`). Every CHECK is
config — rates 0–100, enums, date ordering. No change made.

**`schema/stock/` 16, 19, 20 — THREE FINDINGS, not yet applied:**
1. `fn_svh_post`, `fn_svh_cancel` (19), `fn_svh_post_transfer`,
   `fn_svh_receive_transfer` (20) are units of work in plpgsql — the known
   divergence ([[stock-engine-not-deployed]]); the files should say so the way
   13/14 now do.
2. **`fn_sml_freeze_guard`** (16 ~line 3288) tests **`now()`** against the
   physical-count freeze window, so a sale made before the freeze that SYNCS
   during it is refused. Fix is narrow: test `NEW.sml_doc_datetime` (the
   movement's own timestamp, already a column) instead of `now()`.
   `stock_ledger` also has `sml_sync_date` — offline was anticipated, this
   guard was just missed.
3. `ck_sbl_reserved` (`sbl_reserved_qty >= 0`) and `ck_sbl_transit`
   (`sbl_transit_in_qty >= 0`) on `stock.stock_balance` are contended running
   aggregates. Note there is deliberately NO `sbl_qty >= 0` — on-hand may go
   negative and `comp_negstk_apl` decides in the service — so the file is
   already inconsistent with itself.
Also flagged: `fn_slt_resolve` is not a pure read (INSERTs into `stock_lot`,
get-or-create); check a unique index covers the natural key before offline.

**Still owed:** the reconciliation report itself (not optional once offline
lands), the per-company conflict policy (claw back / write off / chase), and
applying the three stock findings.
