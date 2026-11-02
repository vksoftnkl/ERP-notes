# Pricing decisions + the six forms — the plan

Written 2026-09-04, after the full engine review and the smoke-verified fixes
(see README "Fixed on 2026-09-04"). This is the reference for starting the
client forms. Every table, function, query and mockup named here exists today.

---

## Part 1 — the five pricing questions, answered

### 1. Where do prices live?

In exactly **two tables**, both of which already exist. Nothing new is needed.

| Table | Role | Key | A row exists… |
|---|---|---|---|
| `inventory.item_price_master` (ipm_) | the **HEADLINE** price — the item's default | company + branch + item + uom | always — one per item × uom × branch (you already have this) |
| `stock.stock_mrp_price` (smp_, 16 §20) | the **BUCKET** override — what stock at THIS MRP / THIS sale price sells for | company + branch + item + uom + **key_mrp + key_sp** — company/branch NULLABLE, NULL = chain-wide, specific row wins | only when the item is MRP- or SP-tracked **and** that bucket has been priced |

One rule: **bucket beats headline.** One function decides:

```sql
stock.fn_smp_effective(company, branch, item, uom, mrp, sale_price)
-- → price_a..d (+wot), min_price, and source = 'BUCKET' | 'MASTER'
```

The till, the bulk screen, the purchase screen and every report call this
function. **Nobody joins the two tables by hand** — that is how two screens
end up disagreeing about a price.

What is NOT a price table:

- `stock_lot.slt_sale_price` is **identity**, not a price list — it says which
  bucket the stock belongs to. It mirrors the bucket's *default* level
  (app setting `sales.default_price_level`); the four levels live on the bucket.
- Never key a price on `slt_id` / batch — see question 3.

### 2. Effective dates for MRP and selling price?

**You do not need them, because for MRP-tracked goods a new MRP is a new
bucket — the calendar is embodied in the stock.**

When the next purchase brings Hamam at MRP 45, that receipt creates a new lot
and (from the purchase screen) a new priced bucket. The MRP-40 stock still on
the shelf keeps its own bucket and keeps selling at 40's prices until it sells
through. Nothing is scheduled, nothing flips on a date, and the packet's
printed MRP always governs — which is also the legal requirement.

The only genuine want for a date is a **scheduled reprice of one bucket**
("from 1 Oct the retail level becomes 36"). **The schema side of this is now
built and dormant** (added 2026-09-04, smoke-verified):

- `smp_effective_from/to`, defaulting 1900-01-01 / 9999-12-31 — a row written
  today is simply always in force, so behaviour is unchanged
- `ex_smp_overlap` (EXCLUDE over the bucket key + daterange, deferrable)
  replaced `ux_smp_bucket` — one price per bucket *at a time*, overlapping
  schedules refused
- `fn_smp_effective(…, p_on_date date DEFAULT CURRENT_DATE)` — existing
  callers unchanged; a back-dated bill passes its own date; a bucket
  scheduled only for the future falls back to MASTER until its day

A scheduled reprice is: end the current row (`smp_effective_to = 30 Sep`),
insert the next (`from 1 Oct`), one transaction. Until a screen exposes the
dates, every price change is immediate — exactly what the legacy did and
what shops actually do.

### 3. Chain store wants prices "by MRP / by batch" — where?

| The client says | What it really is | Where it is set |
|---|---|---|
| "price per MRP" | MRP tracking | `stock_track_policy.stp_track_mrp = true` for the item/group → buckets appear per MRP → price them on the bulk screen |
| "price per batch" | **No.** Batch is traceability + expiry, never money | Two batches at the same MRP sell for the same money (the 16 §20 header explains the drift hazard of per-batch prices). What the client actually has is one of the two rows below |
| "…because the printed MRP changed between batches" | MRP tracking | same as row 1 |
| "…and nothing is printed on it" (garments, hardware) | selling-price tracking (skip_mrp) | `stp_track_sale_price = true` → the sale price itself is the bucket dimension |
| "different price in different towns" | branch scoping | already the design — a branch's own smp row beats the chain row |
| "change it for the whole chain at once" | **one chain row** (2026-09-04) | `smp_branch_id NULL` = every branch: one row prices the chain, a new branch starts priced, and a branch that must differ writes its own row, which wins. Resolution is branch-beats-chain, then company-beats-shared — the `fn_stp_effective` pattern |

### 4. "I already hold prices in item_price_master"

Keep it. Unchanged, Prisma-owned, never ALTERed from the stock module. It is
the headline that `fn_smp_effective` falls back to, and it is what an item
sells at until the day it needs MRP-wise differentiation — which most grocery
items never do. **smp is an override, not a replacement.**

### 5. "Once it grows I can't touch anything"

The things that are expensive to change later are all already settled:

- the bucket **key** (item × uom × MRP × sale-price, null-normalised) — locked
- the resolver being **one function** — locked
- prices **off the lot** — locked
- branch scoping — locked

The things left out (effective dating, cost revaluation) are the additive
kind. That is the difference between a gap and a debt.

---

## Part 2 — the six forms, in build order

Engine truth today: OPENING, PHYSICAL, RECEIPT, ISSUE, ADJUSTMENT, DAMAGE and
EXPIRY_WRITEOFF post through `fn_svh_post`. TRANSFER_* / REPACK_* are refused
by name until `stock_transit` is wired (19's deliberate gap). That decides the
order.

### Phase 1 — start coding today (nothing blocks these)

**Form 5 first: Change Selling Price — Bulk.** It writes ONE table
(`stock_mrp_price`), fires no trigger, moves no stock — the gentlest
introduction to the module. Everything is drawn and written:
`change_selling_ui_mockup.png`, queries 16q **Q23** (billing lookup with
price), **Q24** (MRP picker), **Q25** (bulk load), and the save flow on the
mockup (validate → upsert smp → report buckets that matched no stock).

**Form 1: Opening Stock.** `stock_voucher` OPENING + lines; Post is one call
(`fn_svh_post`). `opening_stock_ui_mockup.png` is the layout,
`19_opening_stock_flow.md` is the table-by-table spec,
`19q_opening_stock_queries.sql` has the five screen queries including the
pre-post check (show every bad line at once, in the function's own words).

**Form 2: Physical Stock.** The count sheet is **generated** from
`stock_balance` (one line per godown × lot × bucket — the Step-1 query in
`19_physical_stock_flow.md`); the operator types `svi_counted_qty` only; only
the diff posts. Screen rules from the flow doc: default `svh_rate_source`
to `AVG_COST`. The `fn_svh_recompute` PHYSICAL branch is BUILT (2026-09-04):
a posted count's header now carries the net variance qty and value from the
ledger rows it posted, and shortage lines show their stamped COGS — the
screen may read `svh_total_qty/_value` freely.

### Phase 2 — the transfer engine (BUILT 2026-09-04), then the two forms

`20_stock_transfer.sql` now exists and is smoke-verified through a full
cycle (pair + cancel, despatch, partial receive with damage, short left
open, rebuild clean). The two forms build directly on
`fn_svh_post_transfer` / `fn_svh_receive_transfer`. The two shapes:

- **Godown → godown (same branch), form 3:** no transit — the goods cross the
  yard. One TRANSFER voucher whose post function writes the TRANSFER_OUT and
  TRANSFER_IN ledger row pairs in a single transaction. `stock_balance` moves
  between godown rows instantly. Simple, and most of the daily volume.
- **Branch → branch, form 4:** the full pair. TRANSFER_OUT voucher posts OUT
  rows and creates `stock_transit` rows (status IN_TRANSIT, out-voucher ref);
  the receiving branch's worklist is `ix_svh_inbound` / Q14; a TRANSFER_IN
  voucher — linked via `svh_link_*`, which `ck_svh_transfer_in_link` already
  enforces — posts IN rows for what arrived, damage into the DAMAGED bucket,
  and closes or partials the transit rows. `stt_short_qty` (generated) is the
  transfer-loss report.

The tables, triggers (`fn_stt_refresh`), indexes and constraints for all of
this already exist in 16 — only the two posting functions are missing.

### Phase 3 — purchase entry (form 6)

A purchase today is a **RECEIPT voucher**: supplier on the header, lines carry
batch / MRP / expiry per the item's policy, `fn_slt_resolve` makes the lots,
and it posts as txn type `PURCHASE` — `fn_svh_txn_map` maps it that way
precisely so a real purchase/GRN module later reclassifies nothing.

**Selling price from the purchase screen:** at Post, after `fn_svh_post`
returns and in the same transaction, the screen upserts the smp bucket for
each line's (mrp, sale_price) — and updates the ipm headline if the shop wants
the new price as the default. This is a *separate write to a price table*, not
part of the stock posting.

### "Posted, but the user missed a value" — the answer, by kind of value

The dilemma dissolves once you see that **Post freezes stock facts, and
selling prices were never stock facts:**

| What was missed | Stock fact? | The fix |
|---|---|---|
| selling price, any of the four levels, min price | **No** — lives in smp/ipm, not on the voucher | just edit it: form 5, or reopen the purchase in a prices-only mode. No cancel, nothing reverses — no stock moved |
| qty, free qty, godown | **Yes** — ledger rows | `fn_svh_cancel` (writes reversals) and re-enter. Safe until the stock is sold; correctly refused after, under BLOCK |
| batch no / MRP / expiry typed wrong | **Yes** — lot identity | small quantities: the seeded **RELOT_OUT / RELOT_IN** adjustment pair moves stock from the wrong lot to the right one, with an audit trail. Whole document: cancel + re-enter |
| cost rate | **Yes** — entered the moving average | soon after: cancel + re-enter. After sell-through: the known-gap `REVALUE` txn type (planned engine addition); until then an adjustment pair with a reason. Do not let this block the form |

Design the purchase form to show it: the goods section locks at Post (the
triggers refuse edits anyway); the price columns stay editable forever,
because they write to a master, not to history.

---

## Tally export (future module — decisions recorded 2026-09-04)

The stock schema already holds every figure Tally's import XML needs (actual
vs billed qty = qty vs qty+free; batches, godowns, rates, HSN all present).
When built, it is an additive `21_tally_export.sql`: a name-mapping table
(Tally keys masters BY NAME — our uuid → the Tally name first exported, so a
rename here cannot fork a master there) plus an export outbox (which
documents exported, when, to which Tally company). Two read-side decisions:
lots COLLAPSE to Tally batches (slt_batch_no, or batch/MRP composed when the
distinction must survive), and the append-only ledger makes "everything since
the last export" a clean sml_posted_on question, with reversals exporting as
cancellations. No existing table changes.

## The reframe (why this felt stuck)

The search was for *one* table where a price is finally right, and *one*
moment when it becomes so. The design has **two tables with one resolver**,
and for MRP goods **time is modelled as stock, not as dates** — the old price
dies by selling through, not by expiry of a row. Once those two ideas click,
the six forms reduce to: three buildable today end-to-end, two waiting on one
small SQL file, and one that is a RECEIPT voucher plus a price upsert.
