# `fn_sml_apply` — the contract

Companion to `16_stock.sql` §17.3 and to `16_stock_ledger_columns.md`, which until now
carried this whole thing as **one sentence**:

> 6. `INSERT`. `fn_sml_apply` then maintains `stock_balance`, the lot's total and status,
>    `stock_item_cost`, and enforces the negative-stock policy.

Four tables, a valuation policy and a refusal rule in twenty-two words. Everything else in
`schema/stock/` is step-by-step; this was not, and it is the one piece every posting path in
the product goes through. This file is that step-by-step.

**It is written from the shipped implementation, not from the `.sql`.** See
"Where this actually runs" below — the `.sql` is a specification that is not deployed, and a
specification written from a stale specification is exactly the failure this chain has been
cleaning up. Every rule here was read out of
`nexerp-api-ro/src/modules/stocks/stock-voucher/stock-voucher-posting.helper.ts`
(1,258 lines, synced 2026-09-22) and every schema claim was checked against the live
database on 192.168.0.106.

---

## Where this actually runs

There are two implementations of this contract and **they must never both be installed**.

| | |
|---|---|
| `stock.fn_sml_apply` + `tr_sml_apply` | the spec in `16_stock.sql` §17. **Not deployed.** |
| `applyLedgerRows()` in the NestJS helper | **what runs.** Set-based, once per document. |

Installing the trigger on top of the service would apply every ledger row a second time.
The service's own header comment says so; it is repeated here because this file is the one a
DBA is most likely to read before deciding to "just install the missing functions".

**Do not install them.** `fn_sml_apply` is a NAME in this document, kept because it is what
every other file in `schema/stock/` calls this contract. It is not something to deploy, and
the `.sql` that declares it is a specification that has never run.

### The database enforces NONE of this. Verified on the live box, 2026-09-22.

**`stock` holds zero triggers and one function** — `fn_create_stock_partitions`, which is
DDL and is called by a person or a year-roll job, never on a write path. The three guards
earlier drafts of this file leaned on are **gone**:

| dropped | what it used to guarantee | who guarantees it now |
|---|---|---|
| `tr_sml_immutable` | a ledger row is never UPDATEd | **the service, by discipline** |
| `tr_sml_forbid_delete` | a ledger row is never DELETEd | **the service, by discipline** |
| `tr_sml_freeze_guard` | no movement into a godown a DRAFT PHYSICAL count freezes | `validate()`, per line, before the post |

That is the standing rule and not an oversight: **no functions and no triggers in the
database.** A trigger fires on the SERVER during a cloud push, on rows a till wrote hours
earlier under conditions the server cannot see, in a language this team does not debug, per
statement of a sync batch so a failure points at the batch rather than the row — and whether
or not the service wanted it, so the push cannot stage and correct data before it lands. It
also forks from the TypeScript that has to do the same sums offline.

> **The cost, and what carries it instead.** The *database* no longer stops an `UPDATE` or
> `DELETE` on `stock_ledger`, and any such write would silently stop `stock_balance`,
> `stock_item_cost` and `stock_lot` matching the ledger they cache. What replaces the trigger
> is the same structural answer the loyalty tables got, and **it is already in place**:
>
> * **one writer** — this helper is the only file that may `INSERT INTO stock.stock_ledger`,
>   and nothing, the helper included, may UPDATE, DELETE or TRUNCATE it. A correction is a
>   reversing row, never an edit.
> * **asserted, not trusted** — `test/stock-ledger-single-writer.e2e-spec.ts` walks `src/`
>   and fails on the first violation. It covers the raw-SQL forms *and* the Prisma model API
>   (`stockLedger.create` / `update` / `delete` / …), and it asserts that it found source to
>   scan and that the owner really does insert, so a regex that quietly matches nothing
>   cannot pass forever. It needs no database, so it runs wherever the suite runs.
>
> The freeze guard's check is carried by `validate()`. Both guarantees have a home; neither
> depends on a trigger.

What the database does still hold is **declarative only** — it cannot run at the wrong
moment, and none of it is a function:

* `sbl_on_hand_qty`, `sbl_available_qty`, `sml_signed_base_qty` — `GENERATED ALWAYS`.
  (Sync trap: a generated column may not be written, so a push sending whole rows must omit
  them from the column list.)
* `ck_sbl_accum`, `ck_sbl_holds`, `ck_sbl_bucket` — one row's arithmetic only.
* `ux_sbl_scope`, `ux_sic_scope`, `ux_slt_identity`, `ux_sml_source`, `ux_sml_reversal` —
  the identities the phases upsert on, and the two that make posting idempotent.

Note what is **not** in that list: nothing stops on-hand going negative. `ck_sbl_accum`
keeps the four *accumulators* non-negative, and on-hand is generated from them, so it may
legitimately be below zero. Whether *this* item may is phase 6's job — and with the guards
gone, phase 6 is the **only** thing standing between a BLOCK item and a negative shelf.

---

## The input: which ledger rows one run applies

Every phase below is scoped by the same four predicates, and that is deliberate — one
definition, so the balance, the average, the policy check and the lot totals cannot
disagree about which rows a run is applying.

```sql
    sml.sml_src_doc_id  = :docId
AND sml.sml_acc_year    = :accYear
AND sml.sml_is_deleted  = false
AND sml.sml_is_reversal = :reversal      -- false on a post, true on a cancel
```

**`sml_is_reversal` is the load-bearing one.** A cancel's reversal rows carry the same
`sml_src_doc_id` as the post's. Without that predicate a cancel would sum the original
movement straight back in and undo its own undoing.

A run applies **one side of one document in one year**. It never sees anything else, and it
must not: these phases are increments, not rebuilds.

---

## Phase 4 — `stock_balance`

**Grain of the aggregate:** company, branch, tenant, godown, item, lot, base UOM, bucket.
**Identity upserted on:** `ux_sbl_scope` = (company, branch, godown, item, lot, bucket)
`WHERE sbl_is_deleted = false`.

Tenant and base UOM are inserted but are *not* part of the conflict key. They are carried,
not keyed.

### 4a · the four accumulators

Sum the run's rows into four magnitudes and **add** them to the row:

| column | from |
|---|---|
| `sbl_in_qty` | Σ `sml_base_qty` where `sml_direction > 0` |
| `sbl_out_qty` | Σ `sml_base_qty` where `sml_direction < 0` |
| `sbl_free_in_qty` | Σ `sml_free_base_qty` where `sml_direction > 0` |
| `sbl_free_out_qty` | Σ `sml_free_base_qty` where `sml_direction < 0` |

Nothing else is a quantity you may write. `sbl_on_hand_qty` and `sbl_available_qty` are
GENERATED:

```
sbl_on_hand_qty   = (in + free_in) - out - free_out
sbl_available_qty = on_hand - reserved
```

**Free quantity counts as stock on hand.** A scheme give-away is physically on the shelf, so
it is in the on-hand figure; it is tracked separately only so the *value* of what was given
away can be told apart later. Anything that treats on-hand as `in − out` is wrong by the
free quantities.

### 4b · the three ageing anchors

| column | rule | why |
|---|---|---|
| `sbl_first_in_date` | `LEAST(existing, run's MIN inward date)` | the first arrival on this shelf never moves once set — it is what ageing is measured from |
| `sbl_last_in_date` | `GREATEST(existing, run's MAX inward date)` | |
| `sbl_last_out_date` | `GREATEST(existing, run's MAX outward date)` | |

Back-dated documents are the reason these are `LEAST`/`GREATEST` and not assignments.

### 4c · the identity cache

`sbl_batch_no`, `sbl_mrp`, `sbl_sale_price`, `sbl_expiry_date`, `sbl_supplier_id` are copied
from `stock_lot` and **overwritten** on every apply. They are a denormalised cache of the
lot's identity so the stock screens and `ix_sbl_expiry` / `ix_sbl_mrp` / `ix_sbl_supplier`
can answer without joining the lot. Overwriting is safe precisely because the lot's identity
is immutable — they can only ever be rewritten with the same values.

### 4d · bookkeeping

`sbl_row_version + 1`, `sbl_modified_on`, `sbl_modified_by`. **A movement bumps the version;
phase 5's revaluation does not** — see 5c.

> **The value columns are NOT written here.** `sbl_stock_value`, `sbl_stock_value_wot`,
> `sbl_avg_cost_rate`, `sbl_avg_cost_rate_wot` are phase 5's, and they depend on an average
> that has not been computed yet. Phase 4 before phase 5, always.

---

## Phase 5 — `stock_item_cost`, the moving weighted average

**One row per (company, branch, item)**, on `ux_sic_scope`. The average is a property of the
**item in a branch**, never of a lot and never of a godown.

### The definition

* An **inward** adds quantity and value **at the row's own cost**, then re-averages.
* An **outward** removes quantity **at the current average** and leaves the rate alone.

That is what "moving average" means, and it is why an outward can never change the cost of
what remains. The average is **stored and survives quantity reaching zero**: the next inward
of an item that went to nil is not a fresh start.

### 5a · the lock, in its own statement

Before any arithmetic, take a transaction-scoped advisory lock per (company, branch, item),
**in item order**, in a statement of its own:

```sql
SELECT count(pg_advisory_xact_lock(hashtextextended(
         company || ':' || branch || ':' || item, 0)))
  FROM (SELECT DISTINCT ... ORDER BY 1,2,3) t
```

Three things about this, each of which has to be right:

1. **Why an advisory lock and not just `FOR UPDATE`.** `FOR UPDATE` locks a row that exists.
   The first-ever movement of an item has no row to lock, so two documents would both find
   nothing and both insert. This is the exact race the `.sql` trigger's lock-or-seed loop
   exists for.
2. **Why `ORDER BY`.** Two documents touching the same two items must take the locks in the
   same order or they deadlock. (This was `REVIEW_2026-09-05.md` finding 1.)
3. **Why a separate statement.** A statement's snapshot is taken before anything in it runs.
   A lock acquired *inside* the MERGE could not make the winner's committed row visible to
   the loser's MERGE — the loser would still see no row and still insert a duplicate.

The existing row is then additionally locked `FOR UPDATE` inside the CTE, before the
arithmetic reads it, so two documents serialise and the second computes from the first's
committed figures.

### 5b · the arithmetic, outwards first

The trigger applied rows one at a time in insertion order. A set-based phase has to choose
an order, and the choice is **outwards first, at the average the branch carried before this
document, then inwards re-average**.

That is not arbitrary: it makes an outward's relief the same figure that was stamped on its
ledger row as `sml_cost_rate` under `AVG_COST`, so **the ledger and the average agree about
what the shortage cost**. The only shape where the two orders differ is a count that is over
on one lot and short on another of the same item, and there the difference is which side of
a rounding boundary the average lands on.

```
new_qty       = (has_row ? cur_qty - qty_out : 0) + qty_in
new_value     = GREATEST(cur_value - ROUND(qty_out * cur_avg, 2), 0) + value_in
new_avg       = new_qty > 0  ? ROUND(new_value / new_qty, 6)
              : qty_in  > 0  ? last_in_rate
              : !has_row     ? COALESCE(last_out_rate, 0)
              :                old_avg
```

Both quantity sides include free quantity (`sml_base_qty + sml_free_base_qty`). Value comes
from `sml_cost_value` on the inward side only.

`_wot` ("without tax") is the same arithmetic on the parallel columns, and is carried for
every figure. The two must be computed together or they drift.

**A fresh row never starts below zero.** With no current row the outwards are swallowed:
quantity clamps at 0 and the rate is taken from that document's own rows. An item whose
first-ever movement is an outward (ALLOW; a sale keyed before its receipt) therefore leaves
the *branch quantity* at 0 and the negative on the *balance*, and the next receipt starts the
average clean instead of dividing a real value by a quantity dragged down by stock that was
never costed in.

**Quantity at or below zero keeps a rate.** Never NULL, never 0-by-accident — the document's
last inward rate if it had one, otherwise the rate as it stood.

### 5c · the other five columns

| column | rule |
|---|---|
| `sic_max_cost_rate` | `GREATEST(existing, max inward rate)`. **"Highest rate ever RECEIVED at"** — an outward never sets it. This is the one column the outwards-first rule does not touch. |
| `sic_last_purchase_rate` / `_date` | the document's **last inward line by line order**, or unchanged if it had none |
| `sic_last_sale_rate` / `_date` | from `sml_doc_rate` — the **document** rate, not the cost — on the last outward row whose `sml_txn_type = 'SALE'`. No stock voucher posts a SALE, so today this is the sales module's hook and not a live path. |

### 5d · the stamp — carrying the average onto the holdings

A weighted average is a property of the item, so when a receipt moves it, **every lot of that
item in that branch is worth a different amount**. For every balance row of every item the
run touched:

```sql
sbl_avg_cost_rate     = sic_avg_cost_rate
sbl_avg_cost_rate_wot = sic_avg_cost_rate_wot
sbl_stock_value       = ROUND(sbl_on_hand_qty * sic_avg_cost_rate,     2)
sbl_stock_value_wot   = ROUND(sbl_on_hand_qty * sic_avg_cost_rate_wot, 2)
```

Not only the lot that moved. Revaluing one lot and leaving its siblings on the old rate is
how `Σ sbl_stock_value` silently stops agreeing with `sic_total_value` — which is the
assertion in "What must hold afterwards" below.

**Only those four columns.** A revaluation is not a movement, so the row version and audit
columns of a holding that did not move are deliberately left alone. That is what makes
`sbl_row_version` mean "this shelf moved" and not "this shelf was recalculated".

---

## Phase 6 — the negative-stock policy

Re-read every holding the run touched **after the balances have landed**, and judge any that
is now below zero by the item's effective policy **on the document's date**
(`fn_stp_effective`, falling back to `ALLOW` when the item has no policy row):

| policy | outcome |
|---|---|
| `BLOCK` | **refuse the whole document**, as one 409 naming every offending holding. The transaction rolls back with it. |
| `WARN` | let it through and log it. An offline till that must keep billing is a real business choice. |
| `ALLOW` | nothing to say. The exception report reads `ix_sbl_negative`. |

Two things follow from the ordering:

* The check is **after** the write, not before. There is no "would this go negative"
  arithmetic to get wrong — the balance is already there, and the check is a read of it.
  The refusal works by rolling back.
* The fix for a BLOCK is an **adjustment with a reason**, or keying the receipt first. It is
  never a retry: nothing about the second attempt would be different.

`validate()` reports the same condition per line before the post, so the screen sees it
first. Phase 6 is the backstop for the race between the two, not the primary UX.

---

## Phase 7 — `slt_total_on_hand` and the lot's status

**Recomputed, not incremented.** For each lot the run touched:

```sql
slt_total_on_hand = COALESCE(SUM(sbl_on_hand_qty) OVER every live balance row of that lot, 0)
```

The sum is across **every branch and every godown**, because a lot is chain-wide: note that
`ux_slt_identity` is keyed on (company, item, and the six identity keys) with **no branch and
no godown in it**. Re-derivation rather than increment is the definition of the column, and a
re-derivation cannot drift.

### Status

| from | to | when |
|---|---|---|
| `ACTIVE` | `CLOSED` + stamp `slt_closed_on` | the run had an outward **and** the lot now totals ≤ 0 |
| `CLOSED` | `ACTIVE` + clear `slt_closed_on` | the run had an inward and the lot does not end empty |
| `BLOCKED`, `EXPIRED` | unchanged, always | somebody decided those; posting does not get a vote |

The trigger saw one row at a time. For the one shape where a document both receives into and
issues from the same lot, **ending empty wins** — a lot with nothing in it is closed,
whichever line came last.

---

## Cancellation is the same engine over reversal rows

A cancel is not a second code path. It:

1. mirrors every ledger row the post wrote, with `sml_direction` **negated**,
   `sml_is_reversal = true` and `sml_reverses_id` pointing at the original;
2. runs **phases 4–7 over those rows and only those rows** (`reversal: true`).

Everything else on the reversal row is copied verbatim from the original — including
`sml_doc_date` and `sml_doc_datetime`, so the reversal ages with the movement it undoes, and
including `sml_cost_rate` / `sml_cost_value`.

> **A cancel is value-neutral only if nothing moved in between.** The reversal carries the
> original's cost, but phase 5 relieves an *outward* at the **current** average, not at the
> row's own cost. So cancelling an inward removes quantity at today's average. That is
> correct moving-average behaviour and not a defect — but it means post-then-cancel returns
> `sic_total_value` to its exact prior figure only when no other document touched that item
> in between. Quantity always returns exactly.
>
> Verified on the live box 2026-09-21: two OPENING vouchers and one PHYSICAL, all posted then
> cancelled with nothing between them, left `stock_balance` at `on_hand 0.000000`,
> `value 0.00` on both items — in 15 and 10, out 15 and 10.

`ux_sml_reversal` (unique) is what stops a row being reversed twice. Cancelling a **DRAFT**
runs none of this: a draft has written no ledger row, so there is nothing to mirror.

---

## What must hold afterwards

The assertions worth putting in a test, and the ones a reconciliation job should check:

1. `sbl_on_hand_qty` = Σ `sml_signed_base_qty` over every live ledger row of that
   (company, branch, godown, item, lot, bucket). The balance is a cache of the ledger.
2. `sic_total_qty` = Σ `sbl_on_hand_qty` over that item's live balance rows in that branch —
   **except** where a fresh row swallowed an opening outward (5b), which is the one licensed
   divergence and is visible as a negative `sbl_on_hand_qty` with `sic_total_qty = 0`.
3. `Σ sbl_stock_value` over an item's holdings in a branch = `ROUND(sic_total_qty *
   sic_avg_cost_rate, 2)`, to within per-row rounding. This is the assertion 5d exists to
   keep true.
4. `slt_total_on_hand` = Σ `sbl_on_hand_qty` over that lot, chain-wide.
5. A lot with `slt_total_on_hand <= 0` and `slt_status = 'ACTIVE'` that has ever had an
   outward is a phase-7 escape.
6. No live ledger row exists whose document is not POSTED or CANCELLED.

---

## The order, in one line

```
lots → lines → LEDGER → balance → cost + stamp → negative policy → lot totals → header
  1       2       3         4           5              6                7          8
```

Phases 4–7 are one sequence with one scope, run once on post and once again on cancel over
the mirrored rows. **The ledger is the truth; 4 through 7 are all derived from it**, which is
why any of them can be rebuilt from the ledger and none of them can be trusted over it.

---

## Still open

Everything here is work for **NestJS**. None of it is a function to add to the database —
that door is closed.

* **The rebuild has no contract of its own.** Assertion 1 above is its definition, but the
  rebuild path — what it locks, whether it is safe to run against a live branch, whether it
  may run while a till is pushing — is not written down anywhere. It is the repair tool for
  every drift the dropped `tr_sml_immutable` now makes possible, so it matters more than it
  did when a trigger was holding the line.
* **The two transfer paths are not reimplemented.** A transfer also writes `stock_transit`
  and a paired document, and half-implementing that would move stock out of a branch with
  nothing recording that it is on a lorry. The transfer routes still call the old engine
  function names and **still fail — which is the honest outcome**, and the failure must be
  cleared by writing the service, not by deploying the `.sql`. `usesInProcessPosting()` keys
  on the function name for exactly this reason. `20_stock_transfer_flow.md` describes the
  intended behaviour.
* **Append-only is covered** — `test/stock-ledger-single-writer.e2e-spec.ts`. Listed here
  only so it is not re-litigated: it is a SOURCE scan, so it catches a write someone commits,
  not one an operator types at a psql prompt. That residual risk is accepted and is what the
  reconciliation queries in "What must hold afterwards" are for.
* **`sml_txn_type = 'SALE'`** is wired in phase 5c but nothing posts it yet. It goes live with
  the sales bill.
