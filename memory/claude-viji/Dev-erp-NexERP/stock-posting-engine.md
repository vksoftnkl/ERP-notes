---
name: stock-posting-engine
description: "19_stock_posting.sql — fn_svh_post/fn_svh_cancel, the function 16_stock.sql was missing; opening stock is an OPENING voucher, not a table"
metadata: 
  node_type: memory
  type: project
  originSessionId: e4646e35-4f88-4e8a-8943-cad6a7935e42
  modified: 2026-09-04T14:55:43.417Z
---

Written 2026-09-02, now in **/home/vk/Shared/Prathap/schema/stock/** (the whole stock module moved there the same day): `19_stock_posting.sql`,
`19q_opening_stock_queries.sql`, `19m_migrate_opening.sql`, plus
`opening_stock_ui_mockup.py`/`.png` one level up in the share (PIL, 2x supersample,
Inter + #7B1113 — same house pattern as printing_ui_mockup.py). **No new tables and
no new column prefix** — the whole file is functions, triggers and two indexes
on `stock.stock_ledger`.

**The gap it closes:** `16_stock.sql` builds the engine and stops at the
document. It has `fn_svh_recompute` (header totals) and nothing that turns a
`stock_voucher` into ledger rows, so nothing in the chain could bring stock in.

**Opening stock is `svh_voucher_type = 'OPENING'`**, decided 2026-09-02 after
the user asked "which way is right to go" and chose it over a dedicated
header/detail pair. `svi_` already has every column the legacy `osl_` line had
except the four sale-price levels, and those stay in
`inventory.item_price_master` — the user explicitly chose "prices stay
separate".

**Design calls worth remembering:**
- The one-opening-per-holding rule **cannot be a unique index**. The rule has to
  ignore an opening that was later reversed, and a partial index predicate
  cannot see another row; a unique index would make `fn_svh_cancel` a one-way
  door. It is an EXISTS check inside `fn_svh_post`, backed by the non-unique
  partial `ix_sml_opening`. This deviates from the approved plan, deliberately.
- A reversal carries the **original** doc_date and acc_year, so an as-on-date
  report reads "this document never moved stock"; `sml_posted_on` is the audit
  trail for when the cancellation happened.
- An inward left at cost 0 **raises** after `svh_rate_source` has had its
  chance (a free-goods-only line is exempt) — an OPENING at rate 0 would drag
  the branch's moving average toward zero for every future sale.
- A PHYSICAL count where every line agrees posts **zero rows and still closes**;
  only a voucher with no lines at all raises.
- (2026-09-04) `fn_svh_recompute` gained a PHYSICAL branch: a posted count's
  header carries the NET VARIANCE read from the LEDGER rows it posted (signed
  qty, value at each row's stamped cost; draft = 0 truthfully; reversals
  included so a cancelled count re-totals to 0). Companion fix in fn_svh_post:
  line write-back moved AFTER the ledger INSERT with RETURNING sml_cost_rate,
  so outward lines (issues, damage, count shortages) show the trigger-stamped
  COGS on the document instead of 0. Verified against the flow-doc figures
  (net +1 / 86.00 from −2×20 + 3×42).
- (2026-09-04) Freeze window ENFORCED: tr_sml_freeze_guard in 16 §17.5a
  (BEFORE INSERT on stock_ledger) + partial ix_svh_freeze_open. Semantics:
  DRAFT PHYSICAL + svh_freeze_stock + now() in window → refuse any ledger row
  for COALESCE(from,to) godown of that branch, EXCEPT the count's own rows;
  wall clock not doc date; per-godown not per-branch; posting/cancelling the
  sheet lifts it automatically (status leaves DRAFT). Verified: in-window
  refusal, other-godown pass, own-post pass, auto-lift, rebuild 0. Remaining
  deliberate gaps: REVALUE txn type, value-side rebuild assertion, REPACK.
- `TRANSFER_*` and `REPACK_*` were refused by name until `stock_transit` was
  wired — **done 2026-09-04 in `20_stock_transfer.sql`** (REPACK still refused):
  `fn_svh_post_transfer` (same branch → OUT+IN pair in one txn, POSTED, cost
  read back via INSERT..RETURNING after fn_sml_cost_default stamps it; other
  branch → OUT rows + transit rows, IN_TRANSIT) and `fn_svh_receive_transfer`
  (linked TRANSFER_IN; DAMAGED-bucket line = transit damage posted into the
  DAMAGED bucket; settles transit PARTIAL/RECEIVED; OUT → RECEIVED only when
  nothing remains — a short keeps it open on the loss report). Contract: a
  transfer line must carry svi_lot_id (never fn_slt_resolve — identity moves,
  is not created), one transit row per (item, lot, dest godown) so
  multi-bucket shipments of one lot need separate OUTs, and
  tr_svh_transfer_cancel_guard refuses un-receiving and mid-flight cancels
  (same-branch POSTED pairs cancel normally via fn_svh_cancel). Full cycle
  smoke-verified incl. partial receive, over-receive refusal, rebuild 0.

Validated on a throwaway PG18 cluster (never the live DB): 00_init → 16 → 19 →
19m load twice with zero errors, `fn_sbl_rebuild` returns 0 after every posting
and every cancellation, and all the negative assertions (edit after post, second
opening, duplicate source row, negative-stock BLOCK on cancel) refuse correctly.

`19m_migrate_opening.sql` reads **only the live balances** (`item_batch_stock` /
`item_stock_balance`). The legacy opening DOCUMENT `stock.opening_stock_header/_detail`
is deliberately not a source — the user ruled it out on 2026-09-02: it records what a
branch *said* its opening was, and nothing since is in it.

**Backend plan written 2026-09-07:** `schema/stock/plan-nestjs-opening-stock.md`
(indexed in the stock README). NestJS module for Opening Stock built on 16+19
ONLY — the user's instruction was "use schema/stock, don't use any legacy codes",
so it retires `/api/v1/opening-stocks` + `OpeningStockModule` (which writes
`stock.opening_stock_header/_detail` and posts nothing to stock) rather than
coexisting with it, uses no `acc_voucher_header`/`acc_voucher_seq` (a stock
voucher is not an accounting document — self-contained advisory-lock numbering,
per-device slno), and models `svh_`/`svi_` in Prisma with no migration. Shape:
one shared `stock-voucher` service (11 doc types, 6 screens) + a thin
`OPENING`-pinned controller. Two live-DB blockers named in §0: nothing from
schema/stock is deployed yet, and `stock.fn_create_stock_partitions` must be
called for the year before the first INSERT. The SQLSTATE→HTTP map hinges on
Prisma surfacing function raises as P2010 with the real code in
`error.meta.code`.

See [[stock-schema-design]] for the engine this sits on,
[[db-schema-share-location]] for where the files live.
