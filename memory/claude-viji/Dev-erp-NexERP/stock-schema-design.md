---
name: stock-schema-design
description: "16_stock.sql — the new stock engine: ledger + balance cache + null-normalised lot identity, superseding the Prisma-owned inventory stock tables"
metadata: 
  node_type: memory
  type: project
  originSessionId: e4646e35-4f88-4e8a-8943-cad6a7935e42
  modified: 2026-09-05T05:26:27.473Z
---

`16_stock.sql` (+ `16q_stock_grid_queries.sql`, two PNG diagrams) in
/home/vk/Shared/Prathap/schema, written 2026-08-21. 16 tables in the `stock`
schema, all-new prefixes: stp_ srm_ sag_ slt_ sml_ sbl_ sic_ svh_ svi_ srv_
stt_ svs_ sdd_ srp_ sfc_ smc_.

**The load-bearing idea:** batch-wise / MRP-wise / selling-price-wise / none are
NOT four modes. `stock_track_policy` carries six independent booleans, and
`stock_lot` has a nullable column per dimension plus a GENERATED null-normalised
companion (`slt_key_*`, sentinels `'~'` / `-1` / `0001-01-01` / nil-uuid). The
unique index is built over the companions only — a UNIQUE index treats each NULL
as distinct, so indexing the raw columns would give an untracked item a fresh lot
on every receipt. `fn_slt_resolve()` blanks whatever the policy doesn't track
before looking up, so there is no second code path anywhere.

**Why it supersedes rather than extends:** the live stock tables
(`inventory.item_stock_ledger` stl_, `item_stock_balance` isb_,
`item_batch_master` btm_, `item_batch_stock` ibs_) and `item_master` are
**Prisma-owned** by the NestJS backend at /home/vk/Projects/ERP/ERP-Server/prisma.
They are never ALTERed. Tracking flags could not go on item_master for that
reason — hence stock_track_policy. `item_master.item_batch_config` stays dead.

**Traps found the hard way (all fixed, all verified on PG 18.1):**
- `public.fn_create_year_partitions` (00_init.sql) scans only public/sales/accounts,
  so the file carries its own `stock.fn_create_stock_partitions`. Editing 00 was
  avoided deliberately.
- PostgreSQL refuses `ON UPDATE CASCADE` on an FK over a GENERATED column — this
  is why files 12/15 spell their scope FKs `ON UPDATE NO ACTION`.
- `int4range(x, 2147483647, '[]')` overflows (inclusive upper normalises to +1);
  use `'[)'` with `COALESCE(to+1, ...)`.
- `MIN(uuid)` is not an aggregate; use `(array_agg(col))[1]`.
- `stp_effective_from` must default to a far past date, NOT CURRENT_DATE, or a
  policy authored today silently fails to apply to back-dated receipts.
- A moving average is a property of the ITEM, so a receipt must revalue *every*
  lot of that item in the branch, not only the lot that moved.

**Engine fixes applied 2026-09-04** (found in a full review, all verified by a
smoke run on a throwaway local PG 18 cluster using the docgen stub-masters
pattern): fallback issue strategy FIFO→FEFO; new `ux_sml_reversal` unique index
(one reversal per ledger row — fn_svh_cancel's NOT EXISTS was read-then-write);
`fn_sbl_ensure_row` so a reservation/transit *before* the holding's first
movement creates the zero-qty balance row instead of updating nothing;
`fn_sbl_rebuild` now accumulates drifted+missing counts (was overwriting);
`stock_item_cost` first-movement seeding is a lock-or-seed loop (ON CONFLICT DO
NOTHING swallowed the losing movement); `ux_stt_out_line` gained `stt_bucket`;
the race-loser balance path now also writes date anchors + identity cache.
README.md "Fixed on 2026-09-04" section lists them. Same day, 16p gained
dormant price effective-dating at the user's request: smp_effective_from/to
(1900/9999 defaults = always in force), ex_smp_overlap EXCLUDE replacing
ux_smp_bucket (drop old index on any DB that has it), fn_smp_effective(...,
p_on_date DEFAULT CURRENT_DATE) with both old overloads dropped. Verified:
scheduled reprice txn, future-only bucket→MASTER-until-its-day, overlap
refusal; ck_smp_not_above_mrp also proven in the same run. 16p was then MERGED
into 16_stock.sql as §20 (PART 2 — PRICING) at the user's request; the old
file is 16p_stock_pricing.sql.bak, run order is now 16 → 16t → 16s → 19, and
16_docgen/docreset.sh stubs item_price_master since 16 now reads it. Then
smp_company_id/smp_branch_id were made NULLABLE at the user's request (NULL =
chain-wide row; branch-specific beats chain, then company beats shared — the
fn_stp_effective ORDER BY pattern; ex_smp_overlap COALESCEs both) — this
SUPERSEDES the earlier "fan out per branch, no company row" advice; verified
override/exclusion/per-scope dated reprices. fn_smp_effective then gained
bucket_id + bucket_scope (BRANCH/CHAIN/NULL) outputs — return-type change,
so the file DROPs the old signature before CREATE; Q23/Q25 select the new
cols. 16q gained rule 4 (scope switch semantics), the S1–S3 save statements
(find-then-write; IS NOT DISTINCT FROM for nullable scope; no ON CONFLICT —
EXCLUDE can't be its target), Q26 pre-save check and Q27 priced-but-no-stock
report. change_selling_screen_plan.md in schema/stock is the form-5 spec
(endpoints, Src/scope rules, 4-number cell math, validation matrix; headline
rows with neither dimension route to the ipm endpoint, never smp). Also same
day:
16t_stock_track_preset.sql (spt_) — admin-editable tracking presets for the
policy screen, COPY-only stencil (no FK from policy; screens match on the
generated signature, no match = "Custom"), merge/override scoping like
reasons, policy CHECKs duplicated so bad combos can't be authored; 8 shared
trades seeded (N/M/S/B/BE/BME/BMEP/R), verified idempotent. Pricing doctrine
settled in discussion: ipm=headline, smp=bucket, bucket beats master via
fn_smp_effective only; batch NEVER prices (batch-wise SP = track_sale_price
as identity); "MRP & selling" = track_mrp alone (levels live on the bucket);
issue strategy vs pricing are independent axes. Still open
(deliberate):
WAVG drift when sic_total_qty goes negative; free goods enter the average at
full cost (fn_sml_apply recomputes value from rate, ignores sml_cost_value); no
qty-1 guard on serial-tracked lots; the file is in schema/stock/ (moved
2026-09-02), not flat in schema/.

**2026-09-05:** user paused the Change Selling Price screen build ("plan and
code later") right after context-gathering. Found that day: the three
endpoints the screen needs (GET/POST /api/v1/stock/price-bulk, GET
/stock/price-buckets/:item) are NOT in the live Swagger yet — Prathap hasn't
built them; the client screen must be coded against the plan's contract and
wired when they land. Forms-coverage review done the same day: the six-form
plan does NOT cover ADJUSTMENT / DAMAGE / EXPIRY_WRITEOFF / ISSUE screens
(engine posts all four; ADJUSTMENT requires a reason and is also the
RELOT fix path), nor the setup screens with mockups in the share (tracking
policy via presets, item-group "Tracked as", ageing slab ladder) nor Reason
Master maintenance (Q17–Q20). User is aware; those may need slotting into
the build order.

**STATE AT END OF 2026-09-04 (resume here):** the stock engine is
schema-complete for all six planned forms. Run order: 00–15 → stock/16
(engine + §20 pricing) → 16t (presets) → 16s (seed) → 17 → 18 → stock/19
(posting) → stock/20 (transfers). Every engine gap that blocked forms is
closed and smoke-verified (PHYSICAL recompute, COGS line write-back, freeze
enforcement, transfers). Remaining deliberate gaps: REVALUE txn type,
value-side rebuild assertion, REPACK. NOTHING deployed to the live
192.168.0.106 DB yet — all verification on throwaway local PG18 clusters.
NEXT STEP agreed with user: start CODING the client forms, first the Change
Selling Price bulk screen — its spec is schema/stock/change_selling_screen_plan.md
(endpoints Q24/Q25/Q26+S1–S3+Q27, Src/scope-switch rules, 4-number cell
math). Overall plan: schema/stock/PRICING_AND_FORMS_PLAN.md, also published
at https://claude.ai/code/artifact/80ff7e44-2d96-46b1-a92b-fd3c2ed10114.
User's decisions this session: presets as combo (copy-only), policy set on
item GROUP screen (item card overrides; admin screen for the rest), chain
price rows (nullable smp scope), 16p merged into 16, dormant effective
dates, Tally export deferred (notes in plan doc).

See [[sales-backend-schema-design]] for the layer conventions this follows,
[[db-schema-share-location]] for where the files live, [[stock-posting-engine]]
for 19, and 16p_stock_pricing.sql (smp_ MRP-bucket price levels) beside it.
