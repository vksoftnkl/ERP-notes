## No logic in the database (2026-09-21)

`16_stock.sql`, `19_stock_posting.sql` and `20_stock_transfer.sql` declared 33
functions and triggers between them. **They now declare one** —
`stock.fn_create_stock_partitions`, which is DDL. Everything else is
StockPostingService in NestJS; the three files keep the algorithms as
CONTRACTS beside the flow docs (`19_opening_stock_flow.md`,
`19_physical_stock_flow.md`, `20_stock_transfer_flow.md`) and the
`plan-nestjs-*.md` builds.

> **The one that was still a sentence is now a file.** `fn_sml_apply` — the four
> derived phases every posting path in the product runs through — was twenty-two
> words in `16_stock_ledger_columns.md` while everything around it was
> step-by-step. **`17_sml_apply_contract.md`** (2026-09-22) is that contract,
> written from the shipped NestJS engine and checked against the live database,
> not from the `.sql`. It is the file to read before writing anything that posts
> stock, and the one to read before deciding to "install the missing functions" —
> the trigger and the service must never both be installed.

**The rule** is `sales/README.md` "Where logic lives" plus the offline rider:
the till posts offline and pushes to the cloud, and a trigger fires on the
SERVER during that push, on rows written hours earlier, in a language nobody
here debugs. `sales/offline-sync-invariants.md` has the full argument.

**Here it had already happened.** Of those 33 objects, the live database held
FOUR — `fn_create_stock_partitions`, `fn_sml_immutable`, `fn_sml_forbid_delete`,
`fn_sml_freeze_guard`. `fn_svh_post`, `fn_svh_cancel`, `fn_sml_apply` and every
refresh and resolver were **never deployed**; NestJS reimplemented them and the
`.sql` drifted into a spec nobody runs. `fn_sml_forbid_delete` is live and is
in **no file at all** — someone added it by hand.

**`34_stock_drop_db_logic.sql` HAS RUN — checked on the live box 2026-09-22.**
Schema `stock` now holds **zero triggers** and **one function**,
`fn_create_stock_partitions` (DDL). `fn_sml_immutable`, `fn_sml_forbid_delete`
and `fn_sml_freeze_guard` are gone, and so is every name the `.sql` only ever
specified — `fn_sml_apply`, `fn_svh_post`, `fn_svh_cancel`, `fn_slt_resolve`
return nothing from `pg_proc`. **The rule is absolute: no SQL function in the
database, all of it in NestJS.**

What that costs, and what now has to carry it:

* **Append-only moved from the database to the test suite, and it is DONE.**
  Nothing in the DB stops an `UPDATE`/`DELETE` on `stock_ledger` now, so the
  guarantee is structural as it was for loyalty: **one writer
  (`stock-voucher-posting.helper.ts`), INSERT only**, asserted by
  `test/stock-ledger-single-writer.e2e-spec.ts` — raw SQL *and* the Prisma
  model API, with a self-check so an empty scan cannot pass. Migrations
  `20260922060000` / `20260922120000` did the drops.
* **The freeze check is already carried** — `validate()` reports it per line
  before the post, and it compares `sml_doc_datetime`, not `now()`, so the bug
  the trigger had did not come with it.

`17_sml_apply_contract.md` states all of this at the top, for the reader who
arrives at the posting engine first.

The rest of `34` still says what must replace each drop. Two things in it are
not bookkeeping:

* **`fn_sml_freeze_guard` has a bug.** It tests `now()` against the
  physical-count freeze window — the moment the row ARRIVES, not when the goods
  moved. A sale made at 10:00 before the count began, syncing at 14:00 while it
  runs, is refused. The replacement compares `sml_doc_datetime` instead.
* **`fn_sml_immutable` + `fn_sml_forbid_delete` are the append-only
  guarantee.** They do not break the sync (a push only INSERTs); they go only
  because the rule says no logic in the database. **Do not run section 2 until
  the service is the only writer of `stock_ledger` and only ever INSERTs** — a
  correction is a reversing row, never an edit. Dropping them early buys
  nothing and loses the net.

One more drift found on the way: the deployed `fn_create_stock_partitions` has
a different return type from 16's, so re-running 16 fails with *cannot change
return type of existing function*. Section 3 of `34` reconciles it.

---

# schema/stock — the stock module

Everything that makes up the stock engine, moved out of the flat chain on 2026-09-02 so
the module can be read and worked on as one thing. **The filenames did not change**, so
every "Run order" note elsewhere in the chain is still true — only the folder is new.

## Where it slots into the chain

```
00_init → 01 … 15   →   stock/16_stock.sql
                        stock/16s_stock_seed.sql   →   17 → 18   →   stock/19_stock_posting.sql → stock/20_stock_transfer.sql
```

`16` must run after `15` and before `17`. `19` must run after `16`; it has no dependency on
`17` or `18`, so it can go last. Nothing outside this folder depends on `19`.

```bash
cd /home/vk/Shared/Prathap/schema
for f in 00_init.sql 01_*.sql … 15_promotion_scheme.sql; do psql … -f "$f"; done
psql … -f stock/16_stock.sql -f stock/16t_stock_track_preset.sql -f stock/16s_stock_seed.sql
psql … -f 17_printing.sql -f 18_audit_projections.sql
psql … -f stock/19_stock_posting.sql -f stock/20_stock_transfer.sql
```

## What is in here

### Run these — DDL, in this order

| File | What it is |
|---|---|
| `16_stock.sql` | **The engine + pricing.** 17 tables in the `stock` schema: `stock_lot` (the identity of a holding), `stock_ledger` (every movement, append-only), `stock_balance` (the on-hand cache), `stock_item_cost` (the moving average), `stock_voucher` / `stock_voucher_item` (one document for every movement that is not a sale), plus reservation, transit, valuation, demand, reorder, forecast and ABC/XYZ/FSN. Four parents are partitioned by `acc_year`. **§20 is the pricing part** (merged from the former `16p_stock_pricing.sql` on 2026-09-04): `stock_mrp_price` (`smp_`) + `fn_smp_effective` — four price levels per MRP bucket, because `item_price_master` has no MRP in its key; bucket price wins, the headline is the fallback. The file therefore also reads (never ALTERs) `inventory.item_price_master`. |
| `16t_stock_track_preset.sql` | **Tracking presets** (`spt_`) + 8 seeded trades (Plain, MRP & selling, SP-only, Batch, Perishables, Supermarket FMCG, Pharma, Serial). A copy-only stencil for the policy screen: picking one fills the form, Save writes an ordinary policy row — **no FK back, the engine never reads it**. Merge/override scoping like reasons; carries the policy's own CHECKs so an unpostable combination cannot be authored. Runs anywhere after `16`. |
| `16s_stock_seed.sql` | **The reasons, and the default ageing ladder.** Not decoration: `fn_svh_txn_map` refuses an ADJUSTMENT that cites no reason, so with `stock_reason_master` empty an adjustment cannot be posted at all. 16 shared reasons (incl. the RELOT_OUT/RELOT_IN pair) + a 5-slab default ladder, all with a NULL company. Idempotent. |
| `19_stock_posting.sql` | **The posting engine** — the function `16` was missing. `fn_svh_post` turns a voucher into ledger rows; `fn_svh_cancel` reverses one; `fn_svh_txn_map` decides what each voucher type becomes; two post-lock triggers freeze a posted document. Before this, nothing in the chain could bring stock in. |
| `20_stock_transfer.sql` | **Transfers** (2026-09-04, smoke-verified through a full cycle). `fn_svh_post_transfer`: same branch → the OUT+IN ledger pair in one transaction, POSTED; another branch → OUT rows + `stock_transit` rows, IN_TRANSIT. `fn_svh_receive_transfer`: posts the linked TRANSFER_IN (damage into the DAMAGED bucket), settles transit (PARTIAL/RECEIVED), flips the OUT to RECEIVED when nothing is left; the short stays open on the loss report. Cost travels with the stock, the lot is reused never re-resolved, and a cancel guard refuses un-receiving and mid-flight cancels. REPACK still unbuilt. |

### Run this once, when cutting over

| File | What it is |
|---|---|
| `19m_migrate_opening.sql` | Replays the **live balances** — `item_batch_stock` where the holding is batched, `item_stock_balance` where it is not — into OPENING vouchers **through `fn_svh_post`**, so the engine is proved on real data instead of being handed a balance. Dry-run unless `p_commit`; rejected rows are reported in `tmp_opening_plan`, never silently dropped. |

### Do not run these — read queries the API binds

| File | What it is |
|---|---|
| `16q_stock_grid_queries.sql` | 20 SELECTs. Q1–Q16: the billing lookup, batch list, stock summary, by-MRP, supplier-wise, ageing, near-expiry, valuation as-on-date, FSN/dead stock, what-to-order, below-reorder, chain view, item ledger, transfers in flight, negative-stock exceptions, and the nightly reconciliation assertion. Q17–Q20 are the reason master: the picker (with the shared/override rule), the maintenance list (with `usage_count` and `can_delete`), the edit-load, and the where-used drill-down — plus the three write rules the API must enforce because the schema cannot. |
| `19q_opening_stock_queries.sql` | 5 SELECTs for the Opening Stock screen (menu 44): the list, the edit-load, the pre-post check, what has not been opened, and opening-versus-now. |

### Documentation

| File | What it is |
|---|---|
| `change_selling_screen_plan.md` | **The spec for form 1 of the build order** — the bulk price screen: three endpoints (Q24/Q25/Q26+S1–S3+Q27), the Src/scope-switch rules for chain vs branch rows, the four-number cell math, the validation matrix, keys, and the Qt delegate notes. |
| `PRICING_AND_FORMS_PLAN.md` | **Start here for the client forms.** Where prices live (ipm headline + smp bucket, one resolver), why MRP needs no effective dates, why batch never prices, and the six forms in build order with the fix-after-post rules. |
| `plan-nestjs-opening-stock.md` | **The backend plan for Opening Stock** — NestJS module against `16` + `19` only (no legacy tables, no `acc_voucher_header`): Prisma models with no migration, the shared `stock-voucher` service, device-scoped numbering, save/list/validate/post/cancel, the SQLSTATE→HTTP map, and the flow-doc figures as the integration test. |
| `plan-nestjs-stock-transfer.md` | **The backend plan for the two transfer forms** — one module, two controllers (TRANSFER_OUT and TRANSFER_IN) over the same shared service, because the engine picks the shape from `svh_to_branch_id`: payloads, the receive **prefill from `stock_transit`** (not from the OUT's lines), what to refuse before the engine, the cancel matrix, the SQLSTATE→HTTP map, and six open items including the unwritten LR/vehicle columns. |
| `plan-nestjs-stock-engine-port.md` | **(2026-09-26) What is left, and the rest of the SQL moved into NestJS.** Audit of built vs pending; phase 1 fixes (transfers in TS, header recompute, reserved/transit qty, issue strategy, LOT_COST/MRP, stock→accounts legs under PERPETUAL, day-end pending via `txn_status_log`); phases 3-6 pricing, reports, jobs, deferred. Appendix A maps every former SQL function to its NestJS home. **Overrides the transfer plan's "call `fn_svh_post_transfer`".** |
| `plan-nestjs-stock-adjustments.md` | **(2026-09-26) Phase 2: the adjustment family.** One module and one screen for ADJUSTMENT / ISSUE / DAMAGE / EXPIRY_WRITEOFF / re-lot, per-line direction, the pick-stock picker, reason-ledger accounts legs, and reason master maintenance. |
| `spec_prestrip/` | **(2026-09-26) Reference only, do not run.** The 2026-09-04 copies of 16/19/20 with their function bodies, the spec the port plan cites by line. |
| `16_stock_reference.pdf` | The full reference — every table, every field, worked transactions. **Generated from the database, not hand-written** — regenerate with `16_docgen/`, never edit. |
| `16_stock_ledger_columns.md` | `stock_ledger`, column by column: what each is for and why it is there. Ends with the checklist for posting one row. |
| `16_stock_architecture.png` | How the tables relate. |
| `16_stock_tracking_modes.png` | Why batch-wise / MRP-wise / selling-price-wise / none are not four modes. |
| `19_opening_stock_flow.md` | An opening, from an empty database, table by table — every figure captured from a real run. |
| `19_physical_stock_flow.md` | The same for a physical count, and a side-by-side of how the two differ. |
| `20_stock_transfer_flow.md` | A transfer table by table, both shapes: the same-branch OUT+IN pair, and the inter-branch despatch → transit → partial receive with damage → short left open. Continues the opening flow's figures. **Worked from the function bodies, not captured from a run** — re-run and mark it captured when a cluster is up. |
| `16_docgen/` | Regenerates `16_stock_reference.pdf` from a scratch database. See its own `README.txt`. |

| `item_group_ui_mockup.py` / `.png` | The Item Group Master with the "Tracked as" combo: tree with per-group signature pills, the three combo states (inherit / set / locked-by-stock), and the two-table save (Prisma group API untouched + one GROUP-scope policy upsert). Re-render with `python3 item_group_ui_mockup.py`. |
| `track_preset_ui_mockup.py` / `.png` | The Tracking Presets UI: the policy screen driven by the trade combo (checkboxes greyed until Custom), the admin preset master with the merge/override rule, and the three rules (stencil-not-setting, engine never reads it, bad combos refused). Re-render with `python3 track_preset_ui_mockup.py`. |
| `ageing_slab_ui_mockup.py` / `.png` | The Ageing Slabs ladder editor — the one master screen here that must not be a CRUD grid. Shows the three states (on the default ladder, editing-and-broken, valid) and the coverage bar that makes a gap visible. Re-render with `python3 ageing_slab_ui_mockup.py`. |

Three more renders still live **one level up** in the share, beside the other UI mockups
they were written to match: `opening_stock_flow.png`, `physical_stock_flow.png` and
`opening_stock_ui_mockup.png`, each with the `.py` that draws it.

## What this module never touches

Not one `ALTER`, not one `DROP`. The Prisma-owned tables are read-only inputs:

```
inventory.item_stock_ledger (stl_)   inventory.item_stock_balance (isb_)
inventory.item_batch_master (btm_)   inventory.item_batch_stock   (ibs_)
stock.opening_stock_header/_detail   stock.physical_stock_*
```

They become read-only legacy the day the service layer is pointed here, and `19m` carries
`item_stock_balance` / `item_batch_stock` across.

**The legacy opening document is not carried across at all.** Nothing in this folder reads
`stock.opening_stock_header` / `_detail` — not as a source, not as a fallback, not as a
cross-check. It records what a branch *said* its opening was, possibly years ago, and
nothing that happened since is in it; `item_stock_balance` is what the branch actually
holds now, and that is what a new engine should start from.

## Fixed on 2026-09-04 (verified on a fresh PG 18 run)

- **`fn_stp_effective` now falls back to `FEFO`**, matching the column default — an item no
  longer flips issue strategy the moment a policy row is saved for it.
- **`ux_sml_reversal`** (new, in 16): one reversal per ledger row, enforced. `fn_svh_cancel`'s
  `NOT EXISTS` guard was read-then-write; a concurrent or replayed cancel could double-reverse.
- **A reservation or transit can now precede the holding's first movement**: the refreshers
  create the zero-quantity `stock_balance` row (`fn_sbl_ensure_row`) instead of updating nothing
  and leaving the hold invisible.
- **`fn_sbl_rebuild` returns drifted + missing holdings** — it previously overwrote the drift
  count with the missing-row count, so a nightly assertion could report 0 while balances drifted.
- **`stock_item_cost` seeding race**: the `ON CONFLICT DO NOTHING` could swallow the losing
  movement's quantities on an item's first-ever two concurrent postings; now a lock-or-seed loop.
- **`ux_stt_out_line` includes `stt_bucket`** — one voucher can send saleable and damaged units
  of the same lot to the same godown.
- **The freeze window is ENFORCED** — new `tr_sml_freeze_guard` (BEFORE INSERT on the
  ledger) + partial index `ix_svh_freeze_open`: while a DRAFT PHYSICAL with
  `svh_freeze_stock` is inside its window (wall clock), no movement may touch the counted
  godown except the count's own posting; other godowns keep trading, and posting or
  cancelling the sheet lifts the freeze automatically. Verified all four ways.
- **A posted count's header now carries the net variance** — `fn_svh_recompute` gained a
  PHYSICAL branch reading the LEDGER rows the count posted (signed qty, value at each row's
  actual cost; a draft truthfully totals 0). The line write-back in `fn_svh_post` was
  reordered (INSERT … RETURNING first), so issues, damage and count shortages now show the
  stamped COGS on the document line instead of 0. Verified against the flow-doc figures.
- **Chain-wide price rows** (16 §20): `smp_company_id` / `smp_branch_id` are now nullable —
  NULL branch = one row prices every branch (and a new branch starts priced); a branch's own
  row beats it, then company beats every-company (the `fn_stp_effective` resolution pattern).
  `ex_smp_overlap` COALESCEs both so two chain rows for one bucket are refused while chain +
  branch-override coexist. Verified: override, exclusion, and per-scope scheduled reprices.
- **Effective dating on prices: columns in, feature dormant** (16 §20, formerly 16p). `smp_effective_from/to`
  default to 1900/9999 so today's rows are simply always in force; `ex_smp_overlap` (EXCLUDE,
  replaces `ux_smp_bucket` — drop the old index if a DB already has it) refuses overlapping
  schedules; `fn_smp_effective` gained a defaulted `p_on_date`, so existing callers change
  nothing. Verified: scheduled reprice, future-only bucket falling back to MASTER, overlap refusal.

## Known gaps, as of 2026-09-02

- **No cost revaluation.** A purchase cost corrected *after* the goods have moved cannot be
  fixed: the posted receipt is immutable, cancel-and-re-enter produces a wrong average, and
  `ck_sml_qty_nonzero` forbids the value-only ledger row that would express it. Needs a
  `REVALUE` txn type.
- **`fn_sbl_rebuild` checks quantities only.** It never recomputes `sbl_stock_value`,
  `sbl_avg_cost_rate` or `stock_item_cost`, so it can return 0 while a valuation is wrong.
  There is no value-side assertion.
- **REPACK is refused by name** in `fn_svh_post` (transfers now post through `20`); a repack
  — one lot becoming another — is a future file of its own.
- **Price effective-dating has no screen yet.** The schema side is done (see above); until a
  scheduling screen exists, every price change is immediate, exactly as the legacy behaved.

## Company scoping — three tables, three different answers

`stp_company_id`, `sag_company_id` and `srm_company_id` are all nullable, all meaning "every
company", and all resolved **differently**, because what the caller needs differs:

| Table | Caller needs | Rule |
|---|---|---|
| `stock_track_policy` | **one row** | most specific wins. Scope first (ITEM beats GROUP whatever the company), then company as a tiebreak within a level. `fn_stp_effective` |
| `stock_ageing_slab` | **a whole ladder** | **all-or-nothing.** A company with any slab of its own uses only its own; everyone else uses the default set. Merging two ladders would put one holding in two buckets. `fn_sag_effective` |
| `stock_reason_master` | **a list to choose from** | **merge.** Shared and own appear together — more choices are not a contradiction — except a shared row is hidden when the company defines its own with the same code. Query Q17 in `16q` |

Getting these confused is the easiest way to break a report: the ageing one in particular
looks like it wants the reason-master rule, and would silently double every quantity.
