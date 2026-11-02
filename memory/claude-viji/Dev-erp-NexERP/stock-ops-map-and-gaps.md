---
name: stock-ops-map-and-gaps
description: "2026-09-05 whole-schema review: how each stock table gets its data (trigger/batch/job/manual), the scheduled-jobs list for Prathap, and the open gaps found (transit short, wot, promo_qty, snapshot fn)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 28c2bb21-93af-4cd5-a729-6aea4637de01
  modified: 2026-09-05T06:36:00.418Z
---

Result of the 2026-09-05 table-by-table review with the user (follows
[[stock-schema-design]]). How every stock table gets its rows:

- **Trigger-live** (fn_sml_apply + refreshers, server-side, never synced from
  clients): stock_balance, stock_item_cost, slt_total_on_hand,
  sbl_reserved_qty (fn_srv_refresh), sbl_transit_in_qty (fn_stt_refresh).
- **Batch fn, needs a nightly cron**: stock_demand_daily via fn_sdd_rebuild
  (idempotent delete+reinsert per range). Stockout flag is exact ONLY when
  run nightly (walks back from CURRENT balance); backfills fill qty, not
  flags. fn_sbl_rebuild is the nightly assertion.
- **External job writes, maths deliberately NOT in PL/pgSQL**: stock_forecast
  (method in key → models compete; MAPE/RMSE/bias mandatory) and
  stock_movement_class (ABC×XYZ×FSN per period, stored so March's class
  never changes) — one future forecasting job writes both. Report-only UI
  (Q9/Q10).
- **Explicit act**: stock_valuation_snap (period close; fn to take it NOT
  WRITTEN yet), vouchers, smp prices.
- **Hand-maintained masters**: stp policy, srm reasons (seed suffices at
  launch; maintenance UI needs API-enforced 3 write rules), sag ladder (seed
  suffices; editor must be whole-ladder, never row-CRUD — all-or-nothing
  scoping makes one own row discard the default set), spt presets (seed
  suffices; admin CRUD cheap+safe later), srp reorder policy (no seed;
  bulk-grid screen late phase; backfill from Prisma reorder grid; lead-time
  SD later computable from purchase history).

**Gaps found in the review (engine-side, for Prathap):**
1. Transit short never settles: status stays PARTIAL, fn_stt_refresh keeps
   inflating destination sbl_transit_in_qty forever; needs fn_stt_settle_short
   (flip RECEIVED, keep stt_short_qty — Q14's `OR stt_short_qty <> 0` still
   reports it). Also covers wholly-refused consignment paperwork.
2. stock_transit has no wot cost column: receive's wot leg falls back to the
   lot's frozen FIRST-arrival cost (20:508) while with-tax travels via
   stt_cost_rate. Add stt_cost_rate_wot.
3. As-on-date valuation misses goods in transit (in neither branch's
   balance) — year-end needs a goods-in-transit line; historical as-of needs
   reconstruction from OUT/IN dates. Decide before first year-end.
4. fn_sdd_rebuild never populates sdd_promo_qty (needs promotion-usage
   join) — forecasts would treat promo weeks as baseline.
5. fn_svs_take_snapshot + month-end cron: not written. System works without
   (fn_stock_asof degrades to full replay) but must exist before first
   year-end (freezes audited figures + MRP/SP valuations + age_days that
   are otherwise unrecoverable).
6. Lot cost columns are write-once (correct); but LOT_ACTUAL/FIFO valuation
   issues at the lot's FIRST-arrival rate forever (16:3176) — "lot actual"
   means "lot's first actual". Flagged, not decided.
- Pre-existing deliberate gaps unchanged: REVALUE, REPACK, value-side
  rebuild assertion, price effective-dating screen.

**Other decisions this session:** stock_reservation is generic (source
triple, any holder; sales order is the only planned writer; advisory — till
must sell against available_qty). stock_transit is transfer-only by
construction (NOT NULL branch FKs + out-voucher key). Transfer UI = THREE
screens: Godown Transfer / Branch Outward / Branch Inward (inward is
worklist-driven from Q14, online-only, never free-entry). E-way bill:
inter-state branch transfer = tax invoice + IGST, ₹50k threshold;
intra-state same-GSTIN = delivery challan, TN threshold ₹1 lakh; EWB
generation belongs to the GSP layer fed by stt_lr_no/vehicle_no.

**PRE-HANDOFF REVIEW DONE 2026-09-05** — full findings written to
`schema/stock/REVIEW_2026-09-05.md` (engine fixes ranked + screen-readiness
inventory). Read that file first when the stock work resumes; it supersedes
the gap list above. Method: 5 review passes + a fresh PG18 scratch build of
the whole chain + functional smoke + empirical probes. Chain loads clean and
idempotently; smoke passes end-to-end. Six MUST-FIX items, all REPRODUCED:
(1) fn_sml_apply deadlocks on concurrent posting of one item — 24 deadlocks
in 800 inserts from 2 sessions (lock order sbl→sic→branch-wide broadcast);
(2) post-lock holes (IN_TRANSIT regressed to DRAFT by plain UPDATE, POSTED
voucher hard-DELETEd, posted line reparented); (3) ux_stt_out_line aborts a
legitimate 2-source-godown despatch; (4) fn_sdd_rebuild counts a cancelled
30-unit sale as 60 sold; (5) keyed svi_cost_rate bypasses transfer policy
(rate 555, value 0); (6) sic seed clamp overstates after a
sale-before-first-receipt. Two fixes ALREADY APPLIED to the share that day:
19q:41 usr_name→usr_display_name (live user_master has no usr_name — the
only binding error in 29 EXPLAIN-checked statements) and README 14→16
reasons. Screen verdicts: Opening Stock + Change Selling Price READY; the
other four READY-WITH-GAPS (Branch Inward worst — its inbox and
remaining-qty queries don't exist, and the plan doc's Q14 pointer is wrong);
adjustment family NOT SPECCED as screens.

Offline-sync audit verdicts: see conversation of 2026-09-05 — convergent by
construction except (a) svh_refno must be assigned server-side at sync
(device slno is the dedupe key), (b) freeze-guard refusals during sync must
park-and-retry + "sync before you count" rule, (c) BLOCK-negative items need
doc-time-ordered replay. See also [[item-group-change-warning]],
[[change-selling-screen-build-notes]].
