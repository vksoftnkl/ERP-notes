# Notes 92 — stock cost: plain stock on the item average, EVERY tracked item at its own lot cost (2026-10-06)

> **Rev 2, same day:** checked against branch-wise offline → online sync (§7).
> §3.1 maintenance and §3.3.1 changed from "old + this document" to a REBUILD FROM THE LEDGER.
> §4 now repairs through a revaluation table instead of overwriting values. §7 is new.
> §7 also asks for `applyBalances` to become a rebuild and for deterministic lot ids.
>
> **Rev 3, same day:** NO NEW TABLES.
> - The lot's branch cost lives on `stock_balance` (`sbl_avg_cost_rate`), not in a `stock_lot_cost` table. A rebuilt-from-the-ledger row holds nothing a balance row cannot.
> - The revaluation table in §4 is needed ONLY IF the backfill's step 0 finds tracked lots already sold at the old average. On the box today it finds none.
> - Changed: §3.1, §3.2 (the `lc` join), §3.3.2, §3.4 K, §4, §5, §7. Nothing else.

**The user's rule:** "only for plain stock keep total average, otherwise all should [be] in [this] rule."

- **Plain stock** (track signature `N`, preset `NONE`): one moving average per company / branch / item, exactly as today.
- **Every tracked item** (batch, expiry, MRP, selling price, serial, supplier, and any mix of them): each lot carries its OWN cost. Every movement, screen and report uses that cost.

This is not one screen. Physical stock, stock adjustment, transfer, sale bill COGS, returns and the
stock reports all read the same figure. So the fix is in the posting engine, and every reader follows
from it.

---

## 1. What happens today (live, Acme / Coimbatore, 2026-10-06)

**MRP STOCK ITEM** (preset FMCG_MRP, signature `BME`): four batches, each opened at its own cost.

| Batch | MRP | Qty | Inward cost (ledger) | `sbl_avg_cost_rate` (every screen shows) |
|---|---|---|---|---|
| A1251 | 40 | 10 | 25.00 | **340.909091** |
| HJ785 | 500 | 10 | 250.00 | **340.909091** |
| L-815 | 650 | 15 | 400.00 | **340.909091** |
| KL8525 | 550 | 20 | 500.00 | **340.909091** |
| | | 55 | 18,750.00 | 18,750 ÷ 55 = 340.909 |

What that does once stock moves:
- **Physical count:** a shortage of 10 × A1251 is posted at 3,409.09. Those goods cost 250.
- **Write-offs:** an expiry or damage write-off of the 40-MRP batch is posted at 13.6× its cost.
- **Sale bill:** selling 1 × A1251 (MRP 40) books COGS 340.91, so the bill shows a loss that did not happen. Selling 1 × KL8525 (MRP 550) books 340.91 on goods that cost 500.
- **Transfer:** the goods arrive in the other branch at 340.91, whatever they cost.

**MRP ITEM** (signature `M`): its one lot has two openings, which average to **252.63** from the ledger.
`stock_lot.slt_cost_rate` says **200.00**, because it is written once, when the lot is created
(`resolveLots`), and never updated. So the existing `LOT_COST` rate source reads a stale figure too.

On the box today, no tracked item has an outward movement yet. Nothing has been mis-posted so far.
The totals are therefore unchanged by this fix (tracked: 18,750.00 and 9,600.00 both ways); only the
split across batches is wrong.

### Where it comes from (nexerp-api-ro, read 2026-10-06)

| # | File / function | What it does today |
|---|---|---|
| A | `stock-voucher-posting.helper.ts` `postingCte()` → `priced.line_cost_rate` (~l.716-735) | **Every outward line and every count shortage is relieved at `sic.sic_avg_cost_rate`**, whatever the document says. This one CASE decides COGS for sale bill + DC (`sales-stock.service.ts` posts through this engine and sums `sml_cost_value` into `costByLine` / `cogsAmount`), and also for adjustment OUT, physical MINUS and transfer OUT. |
| B | same, `stated_cost_rate_wot` (~l.743-757) | The without-tax pair, from the same average. |
| C | same, inward `AVG_COST` rate source (~l.729) | Count overage, adjustment IN, re-lot IN and transfer default all come in at the ITEM average. |
| D | same, `LOT_COST` (~l.731, lateral `slc` ~l.762) | Reads `slt_cost_rate`, which is the FIRST receipt's cost only. |
| E | `applyItemCost()` (~l.2078-2215) | Maintains `stock_item_cost`. Outward value is taken off as `qty_out × sic_avg_cost_rate` (~l.2153), not as the value actually relieved. |
| F | `applyItemCost()` stamp (~l.2222-2235) | Writes the ITEM average onto EVERY holding of the item: `sbl_avg_cost_rate`, `sbl_stock_value`. |
| G | `stock-voucher.service.ts` `countSheet()` (~l.2866) | Sends `sbl_avg_cost_rate` → the physical stock screen. |
| H | `stock-adjustment.service.ts` pick-stock (~l.452/492) | Sends `sbl_avg_cost_rate` → the adjustment screen's outward cost. |
| I | `itembatchstock/itemBatchStockService.ts` (~l.70/164) | Per holding: `sbl_avg_cost_rate`, `sbl_stock_value`. |
| J | `itemstockbalance/itemstockBalanceService.ts` (~l.173-174, 451, 455) | `MAX(sbl_avg_cost_rate)` and `first.sbl_avg_cost_rate`. These are correct today only because every lot carries the same rate. |
| K | `selling-price-bulk/price-bucket.gateway.ts` `mrp_cost` CTE (notes 75) | Already per MRP: the ledger-derived inward cost weighted by on-hand. Keep it, but see §3.4. |
| L | `stock_track_preset.spt_valuation_method`, `stock_track_policy.stp_valuation_method` | **All 8 presets and all 37 policies are `WAVG`.** `LOT_ACTUAL` is allowed by `ck_stp_valuation` / the preset CHECK. No posting code reads the column: the policy service only copies it from the preset (`stock-track-policy.service.ts` ~l.300/498). |

---

## 2. The rule

**Valuation method = the effective policy's `valuation_method`:**

| Policy | Method | Cost of one unit of a holding |
|---|---|---|
| tracks nothing (`N`) | `WAVG` | the item's branch moving average (`stock_item_cost`), as today |
| tracks anything | `LOT_ACTUAL` | that **lot's** moving average **in this branch** (§3.1) |

1. **Data:**
   - `UPDATE stock.stock_track_preset SET spt_valuation_method = 'LOT_ACTUAL' WHERE spt_code <> 'NONE'` (7 presets).
   - For existing policies, set `stp_valuation_method = 'LOT_ACTUAL'` wherever the policy tracks any dimension; `WAVG` otherwise.
2. **Enforce it on save** in the policy service: a policy that tracks any dimension is stored as `LOT_ACTUAL`, and one that tracks nothing as `WAVG`, whatever the payload says. The user's rule has no exception, so the screen does not need a choice. If you would rather keep the column editable, say so, and I will put the combo on the item group / item card.
3. **The engine reads it:** `effectivePolicyCte()` / `effectivePolicyLateral()` should expose `valuation_method` next to `track_*` and `issue_strategy`, so `keyed` carries it.

---

## 3. The ask

### 3.1 A lot's cost per branch — kept on `stock_balance`, NO new table

**Grain:** one cost per (company, branch, lot). It is written onto EVERY `stock_balance` row of that lot in that branch: every godown and every bucket gets the same `sbl_avg_cost_rate` / `_wot`, and `sbl_stock_value = ROUND(sbl_on_hand_qty × rate, 2)`.
- **Same rate on every row:** a godown-to-godown or bucket move inside one branch cannot change value. That is the same reason the item average is per branch, not per godown (16_stock.sql §7). A lot's cost per branch is the item rule, one grain finer.
- **Why no `stock_lot_cost` table:** the maintenance below rebuilds the figure from the ledger and never reads its previous value. A separate row would only be a second copy of what the balance rows already carry, and every reader (count sheet, pick-stock, batch stock) already reads the balance rows.
- **A sold-out lot keeps its cost:** balance rows stay at zero quantity, so their rate stays. A branch that has NEVER held the lot has no rows, and falls back to the item average (§3.2).
- **Why not `slt_cost_rate`:** `stock_lot` is company-wide (it has no branch), and `slt_cost_rate` is the first receipt only. Keep it as that, "the cost on the first receipt", and say so in its comment.

**Maintenance: REBUILD, never increment.** Add `applyLotCost()`, run right before `applyItemCost()` in `postStockVoucher`, in the cancel path, and in the sync receiver (§7). For every (company, branch, lot) of a `LOT_ACTUAL` item the document touched, it RE-DERIVES the rate from the ledger and writes it onto that lot's balance rows in the branch:

```sql
-- per (company, branch, lot); every live row, forward and reversal alike
qty        = Σ sml_signed_base_qty                                    -- generated column, already signed
value      = Σ sml_direction × sml_cost_value      [+ Σ scr_value]    -- scr only if §4 needs it
value_wot  = Σ sml_direction × sml_cost_value_wot  [+ Σ scr_value_wot]
rate       = value / qty                       when qty > 0
           = the LAST inward row's sml_cost_rate,
             by (sml_doc_datetime, sml_id)     when qty <= 0           -- a lot sold out still knows its cost
rate_wot   = the same with the _wot columns

UPDATE stock.stock_balance b
   SET sbl_avg_cost_rate     = rate,
       sbl_avg_cost_rate_wot = rate_wot,
       sbl_stock_value       = ROUND(b.sbl_on_hand_qty * rate,     2),
       sbl_stock_value_wot   = ROUND(b.sbl_on_hand_qty * rate_wot, 2)
 WHERE (company, branch, lot) = that lot's        -- every godown, every bucket
```

- An outward row adds `−sml_cost_value`: what was ACTUALLY relieved (§3.2), not qty × the current average. That is what makes the sum correct.
- A reversal row carries the opposite direction of the row it reverses, so a cancel needs no special case.
- **Why a rebuild and not "old + this document":** the result depends only on WHICH rows exist, never on the order they arrived in or how many times one was applied.
  - A re-sent sync batch, a reversal that lands before its original, or a retry after a half-applied batch all give the same answer.
  - An incrementing figure gets each of those wrong, and has to be remembered on every write path into the ledger.
  - This is the rule we already use for loyalty (share/sales/offline-sync-invariants.md): recompute, do not increment.
- **Cost:** one grouped read of a lot's own ledger rows per touched lot. `ix_sml_item_date` (company, branch, item, …) covers it; add (company, branch, lot) if the plan says otherwise.
- **Locking:** the per-item lock `lockHoldingItems()` / `applyItemCost()` already take covers these rows. No new lock.
- **Rounding:** Σ `sbl_stock_value` of a lot can differ from `qty × rate` by a paisa per row. That is the same as today with the item average.
- `WAVG` (plain) items are not touched here. They keep the item-average stamp (§3.3.2).
- **Negative stock:** while a lot's qty is below zero (sold before its receipt synced, or oversold), its value is not a real cost, so the rate takes the qty ≤ 0 rule (last inward rate). Once the receipt lands and qty is positive, value ÷ qty is right again. This is the same limitation the item average has today. The reconciliation report (offline-sync-invariants) lists lots with qty < 0.

### 3.2 Posting — `postingCte()` `priced`

`priced` needs one more join: the lot's current rate in this branch, read from its balance rows. They all carry the same rate (§3.1). `MAX` simply ignores a row this document has just created at 0.

```sql
LEFT JOIN LATERAL (
  SELECT NULLIF(MAX(b.sbl_avg_cost_rate), 0)     AS lot_rate,
         NULLIF(MAX(b.sbl_avg_cost_rate_wot), 0) AS lot_rate_wot
    FROM stock.stock_balance b
   WHERE b.sbl_company_id = keyed.svh_company_id
     AND b.sbl_branch_id  = keyed.svh_branch_id
     AND b.sbl_item_id    = keyed.svi_item_id
     AND b.sbl_lot_id     = COALESCE(keyed.svi_lot_id, slc.slt_id)    -- the named or resolved lot
     AND b.sbl_is_deleted = false
) lc ON keyed.valuation_method = 'LOT_ACTUAL'
```

- Place it after the existing `slc` lateral, which resolves the lot by identity.
- **The lot is always known by the time the line is priced.** `postStockVoucher` runs `pickIssueLots` → `resolveLots` → `attachLotsToLines` BEFORE `writeLedger` prices the line. So `svi_lot_id` is set on every outward line, the FEFO-picked ones included.
- `ux_sbl_scope` leads with company, branch, godown, item, lot. If the plan scans, add an index on (company, branch, item, lot).

| Line | Today | `LOT_ACTUAL` |
|---|---|---|
| **Outward** (sale, DC, adjustment OUT, damage / expiry, transfer OUT, re-lot OUT) and **count shortage** | `sic_avg_cost_rate`, then keyed, then 0 | `COALESCE(lc.lot_rate, NULLIF(sic.sic_avg_cost_rate,0), NULLIF(keyed.svi_cost_rate,0), 0)` |
| its wot | the average's wot | `lc.lot_rate_wot` **when the cost came from `lc`** (the notes-77 rule: wot from the same source as the cost) |
| **Inward, rate source `AVG_COST`** (count overage, adjustment IN) | `sic_avg_cost_rate` | the lot's own `lc.lot_rate` when this branch already holds that lot. Only a lot new to the branch falls back to `sic_avg_cost_rate` |
| **Inward, `LOT_COST`** | `slt_cost_rate` (first receipt, stale) | `lc.lot_rate`, then `slt_cost_rate` |
| **Re-lot IN** | forced `AVG_COST` = item average | the paired OUT row's `sml_cost_rate` / `_wot`. The goods are the same goods; the new lot inherits what they cost |
| **Bucket move** | same lot, same cost | unchanged: same lot, so the lot average does not move (value out = value in) |
| **Transfer IN** | `stt_cost_rate` | unchanged. The OUT row now carries the lot cost, so `stt_cost_rate` is the lot cost too |
| **Sale / DC return** | the quoted bill cost (`sri_cost_price` / `sdri_cost_price`) | unchanged. The bill's `sbi_cost_price` is now the lot cost, so the goods come back at what they left at. A return with no quoted cost should use `lc` before `sic` |
| `WAVG` (plain) lines | — | **unchanged in every row of this table** |

Nothing in sales code changes. `costByLine`, `cogsTotal` and `buildCogsLegs` already read `sml_cost_value`.

### 3.3 `applyItemCost()` and the stamp

1. **`stock_item_cost` becomes a rebuild too.** Today's MERGE is "old + this document", and its outward side is `ROUND(m.qty_out * c.sic_avg_cost_rate, 2)` (~l.2153). Replace both with the same derivation as §3.1, at item grain, per touched (company, branch, item):
   - `sic_total_qty` = Σ `sml_signed_base_qty`;
   - `sic_total_value` = Σ `sml_direction × sml_cost_value` [+ Σ `scr_value`, only if §4 creates that table] (and `_wot`);
   - `sic_avg_cost_rate` = value ÷ qty when qty > 0, else the last inward rate;
   - last purchase / last sale stamps: the latest qualifying row by (`sml_doc_datetime`, `sml_id`), not "this document's last line".

   What that gives:
   - **WAVG item:** the same numbers as today, because its outward rows were relieved at the running average, and Σ in − Σ out at that average is the moving-average value.
   - **LOT_ACTUAL item:** `sic_total_value` = Σ of its lot values, always. `sic_avg_cost_rate` is then the item's SUMMARY (total ÷ qty), right for item-level reports and as the fallback for a lot new to the branch.
   - **WAVG under offline:** the moving average a branch used for each sale is already stamped on that sale's ledger row. The rebuild never needs to replay the order; it only sums.
2. **The stamp** (~l.2227): per holding, by the holding's method:
   - `WAVG` → today's `c.sic_avg_cost_rate` (unchanged);
   - `LOT_ACTUAL` → **skipped here.** `applyLotCost()` (§3.1) has already written the lot's own rate onto those rows; the item stamp must not overwrite it.

   So the existing stamp gains one condition: the holding's item is `WAVG`.

### 3.4 Readers

| # | Reader | Change |
|---|---|---|
| G | count sheet | **none.** It reads `sbl_avg_cost_rate`, which `applyLotCost()` (§3.1) now writes per lot. Physical stock then shows 25 / 250 / 400 / 500. |
| H | adjustment pick-stock | **none**, same reason |
| I | item batch stock | **none**, same reason |
| J | item stock balance | `MAX(b.sbl_avg_cost_rate)` → `SUM(b.sbl_stock_value) / NULLIF(SUM(b.sbl_on_hand_qty), 0)` (and wot). Also l.451/455: `first.sbl_avg_cost_rate` → the same Σ value ÷ Σ qty over the item's rows. With different rates per lot, MAX and "first" are just one lot's rate. |
| K | price-bulk `mrp_cost` (notes 75) | Switch it to `stock_balance`: Σ `sbl_stock_value` ÷ Σ `sbl_on_hand_qty` over the MRP's lots in the branch (on-hand > 0). When nothing is on hand, use the plain average of those lots' `sbl_avg_cost_rate`. Same figure as today's ledger lateral, but then there is ONE definition of a lot's cost instead of two that can drift. |
| — | `/master-lookups/item-price` | none: it reads `ipm_cost_price`, which Change Selling already writes per MRP |

`stock-balance-assertion.ts` checks quantity only, so nothing breaks there. Suggestion: add a value check, Σ `sbl_stock_value` per item vs `sic_total_value` (to within 0.01 × holdings), so the two can never quietly part again.

---

## 4. One-off backfill (send it to me first; I will dry-run it on erp_dry)

**Step 0 decides whether anything beyond the rebuild is needed.** Under §3.1 a lot's value is Σ of its ledger rows. A tracked lot that has ALREADY had an outward has rows relieved at the old item average, and the rebuild would carry that error forever. So first look for such lots:

```sql
SELECT sml.sml_company_id, sml.sml_branch_id, sml.sml_item_id, sml.sml_lot_id, count(*) AS outward_rows
  FROM stock.stock_ledger sml
  JOIN stock.stock_lot slt ON slt.slt_id = sml.sml_lot_id
 WHERE slt.slt_track_signature <> 'N'
   AND sml.sml_direction = -1
   AND sml.sml_is_deleted = false
 GROUP BY 1, 2, 3, 4;
```

**On the box today this returns 0 rows** (checked 2026-10-06). Then:
1. **Policies:** apply the data change in §2.1.
2. **Run the §3.1 / §3.3 rebuild** for every (company, branch, item) that has stock. It writes each tracked lot's own rate onto its balance rows and re-totals `stock_item_cost`.
3. **Report, don't post:** list every (company, branch, item) whose `sic_total_value` moved, with old, new and difference. With step 0 empty, the totals should not move. Only the split across batches changes.

**Only if step 0 returns rows.** That is a database where tracked items were already sold under the old average. The ledger cannot hold a value-only fix (`ck_sml_qty_nonzero` refuses a 0-qty row, and there is no revaluation txn type), so add one append-only table, which the rebuild adds in (the `[+ Σ scr_value]` in §3.1):

```sql
CREATE TABLE IF NOT EXISTS stock.stock_cost_revaluation (
    scr_id            uuid          NOT NULL DEFAULT uuidv7(),   -- made where it happens: syncs like any document
    scr_company_id    uuid          NOT NULL,
    scr_branch_id     uuid          NOT NULL,
    scr_item_id       uuid          NOT NULL,
    scr_lot_id        uuid          NOT NULL,
    scr_doc_date      date          NOT NULL,
    scr_value         numeric(18,2) NOT NULL,                    -- signed
    scr_value_wot     numeric(18,2) NOT NULL,
    scr_reason        varchar(30)   NOT NULL,                    -- 'LOT_COST_BACKFILL'
    scr_remarks       varchar(250),
    scr_is_deleted    boolean       NOT NULL DEFAULT false,
    scr_sync_date     timestamptz,
    scr_created_on    timestamptz   NOT NULL DEFAULT now(),
    scr_created_by    varchar(100),
    CONSTRAINT pk_stock_cost_revaluation PRIMARY KEY (scr_id),
    CONSTRAINT ck_scr_value_nonzero CHECK (scr_value <> 0 OR scr_value_wot <> 0)   -- one row's own arithmetic
);
CREATE INDEX IF NOT EXISTS ix_scr_lot
    ON stock.stock_cost_revaluation (scr_company_id, scr_branch_id, scr_lot_id) WHERE scr_is_deleted = false;
```

For each lot step 0 found:
- **Target:** `target_avg` = Σ `sml_cost_value` ÷ Σ (`sml_base_qty + sml_free_base_qty`) over unreversed inward rows (the notes-75 filter: `sml_direction = 1`, not a reversal, not deleted, no live `sml_reverses_id` pointing at it); the same with `_wot`. `target_value` = on-hand × `target_avg`.
- **Ledger value:** Σ `sml_direction × sml_cost_value` (the §3.1 sum).
- **Where they differ,** insert ONE row with `scr_value = target − ledger` (and `_wot`), reason `LOT_COST_BACKFILL`. Then run steps 2-3 above.
- **Booking:** the difference is a revaluation of Stock-in-Hand. The user decides how to book it; it must not be posted silently. Historic COGS on posted bills stays as posted; only the stock left on the shelf is restated.
- **Idempotent:** run it twice and the second run finds no difference.

If step 0 is empty, **do not create this table.**

Read-only check of step 2 on the box (no outward on a tracked lot today):

| Kind | Items | Holdings | Value now | Value at lot cost |
|---|---|---|---|---|
| plain (stays WAVG) | 112 | 112 | 237,403.78 | (not changed) |
| tracked `BME` | 1 | 4 | 18,750.00 | 18,750.00 |
| tracked `M` | 1 | 2 | 9,600.00 | 9,600.00 |

---

## 5. Acceptance — I will run these from the Qt side once it is in

**MRP STOCK ITEM, Coimbatore:**
1. **Physical stock** (Load Sheet F9): Avg Cost shows **25.00 / 250.00 / 400.00 / 500.00** per batch (today 340.909 on all four).
2. **Physical stock, shortage:** count A1251 as 9 (book 10) and post. Diff Value is **−25.00**, the MINUS ledger row is 1 @ 25.00, and the A1251 lot cost stays 25.00.
3. **Stock adjustment, expiry write-off:** 1 × KL8525 is valued at **500.00**.
4. **Sale bill:** 1 × L-815. Line `sbi_cost_price` **400.00**, and the COGS leg is DR 400 / CR Inventory 400.
5. **Sale return** of that bill: comes back at 400.00, and L-815 is back to 15 @ 400.00.
6. **Item stock balance** after 2 + 3 + 4 (before 5): item value = 18,750 − 25 − 500 − 400 = **17,825.00** on 52 units → item avg **342.79**; each batch still at its own cost.
7. **Transfer** of 1 × HJ785 to another branch: OUT 250.00, IN 250.00 there.

**MRP ITEM:** its lot shows **252.63**, the ledger average (today `slt_cost_rate` 200).

**Any plain item:** every figure is identical before and after.

**Rebuild = idempotent (§3.1, §7):**
- run the rebuild twice on the same lot / item: the rows are byte-identical;
- post → cancel → post the same document: the lot ends where it would after one post;
- zero `sbl_avg_cost_rate` on one branch's tracked holdings and rebuild: the same rates come back.

I will cancel every test document afterwards.

## 6. Client side (mine, after yours)

No Qt change is needed for the numbers: every screen shows what the server sends. After this is in, I will change only the wording in Stock Adjustment, which says "An outward line is always valued at the branch average" and "A re-lot carries the OUT line's average". Both will say "at what the stock cost (the batch's own cost for tracked items)".

## 7. Branch-wise offline → online sync

Checked against the plan for each branch to post offline and sync to the centre later. The cost model holds, for these reasons and on these conditions.

**Why it holds:**
- **Every cost is branch-scoped.** `stock_item_cost` is per (company, branch, item); a lot's cost is per (company, branch, lot) on that branch's own `stock_balance` rows. Only a branch's own documents move its cost rows, so two offline branches never contend for one cost.
- **A branch posts its own documents in order**, so the cost each sale, write-off or shortage took is fixed AT THE BRANCH, on that ledger row.
- **A transfer carries its cost** (`stt_cost_rate`). The receiving branch never uses its own average for it, and can only receive after the OUT has reached it.

**Conditions:**

1. **The posted cost is final; the centre never re-costs.**
   - A synced document arrives with its ledger rows, `sml_cost_rate` / `sml_cost_value` included.
   - The receiver stores them and runs only the REBUILDS (§3.1, §3.3, balances, lot totals).
   - It must not run `postingCte` pricing again. Its averages are at a different moment from the branch's, so a re-costed bill would no longer match the COGS legs already posted from it.
2. **Caches are rebuilt, never incremented** (§3.1, §3.3.1).
   - Already rebuilds, so fine as they are: `refreshReserved` (Σ `stock_reservation`), `refreshTransitIn` (Σ `stock_transit`), and `refreshLotTotals`' on-hand (Σ `stock_balance`).
   - Still "old + delta", so make them rebuilds before offline:
     - `applyItemCost` (§3.3.1);
     - **`applyBalances`**: its `ON CONFLICT` ADDS this document's in/out to `sbl_in_qty` / `sbl_out_qty` / free qty. Rebuild them per touched holding as Σ of the holding's live ledger rows; the generated on-hand / available follow.
   - `refreshLotTotals`' CLOSED / ACTIVE switch is decided from "this document had an out / an in". Derive the status from the rebuilt on-hand instead (≤ 0 after any outward → CLOSED, > 0 → ACTIVE).
3. **Lot identity must be decided the same way in every branch.**
   - Today `stock_lot` is company-wide, `slt_id` defaults to `uuidv7()`, and `ux_slt_identity` makes the natural key unique.
   - Two branches offline that both receive batch KL8525 / MRP 550 / exp 2027-07-25 of one item each create a lot with a different id. At sync the second insert violates `ux_slt_identity`, and the whole batch fails.
   - **Ask:** make `slt_id` DETERMINISTIC — uuid v5 over (company, item, key_batch, key_mrp, key_sp, key_expiry, key_serial, key_supplier). Both branches then mint the same id, and the sync is an upsert (`ON CONFLICT (slt_id) DO NOTHING`). No remapping of ledger, balance or bill rows.
   - Existing lots keep their ids. The v5 rule applies to new lots, and the resolver looks a lot up by key before it mints one, as today.
   - Because a lot's cost is kept per branch on the balance rows (§3.1), sharing one lot row across branches is safe: no branch can move another's cost through it.
4. **Company-wide fields on the lot are rebuilt at the centre.** `slt_total_on_hand` and `slt_status` / `slt_closed_on` are summed across branches. A branch writes them from what IT can see, so the centre must re-derive them from `stock_balance` after each sync. Otherwise the last branch to sync wins.
5. **Cross-branch writes inside a transfer.**
   - Posting a TRANSFER_OUT refreshes the DESTINATION branch's `sbl_transit_in_qty` (`refreshTransitIn`). Offline, the sending branch does not hold that row.
   - `refreshTransitIn` is already a rebuild from `stock_transit`, so the only change is WHERE it runs. Also run it in the sync receiver: at the centre, and at the destination once the OUT reaches it.
   - The cost needs nothing: it travels on `stt_cost_rate`.
6. **Freeze window by the movement's own time.** A physical count freeze must test the movement's `sml_doc_datetime`, not the sync moment. A sale made before the freeze that syncs during it is a real sale (flagged in offline-sync-invariants for the old `fn_sml_freeze_guard`; please confirm the NestJS port does the same).
7. **No new CHECK over a running total.** This plan adds no table. The conditional `stock_cost_revaluation` (§4) carries only a single-row CHECK. A lot's qty may go negative offline (two tills sell the last unit); the rebuild records it, and §3.1's qty ≤ 0 rule keeps its cost. `comp_negstk_apl` refuses at post while online, and a reconciliation report finds the rest.

**What stays order-dependent, and why that is fine:** the moving average itself, for WAVG and per lot. It is evaluated once, at the branch, in that branch's own posting order, and frozen onto the ledger row. Everything after that is a sum.
