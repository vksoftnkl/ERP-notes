# Physical stock, table by table — and how it differs from an opening

Companion to `19_stock_posting.sql`, and the twin of `19_opening_stock_flow.md`. Every figure
was **captured from a real run** on a fresh PostgreSQL 18 cluster with `00_init.sql` →
`16_stock.sql` → `19_stock_posting.sql` loaded.

A physical count is `svh_voucher_type = 'PHYSICAL'` on the same two tables an opening uses.
Four things make it behave differently, and all four are worth knowing before the screen is
written:

1. **It cannot start from nothing.** An opening creates stock; a count corrects stock. The
   count sheet is *generated* from `stock_balance`, not typed.
2. **Only the difference posts.** `svi_qty` stays 0 for the whole document.
3. **A line that agrees writes nothing** — and the document still closes POSTED.
4. **It moves stock in both directions at once**, so one document exercises both the inward
   and the outward cost path.

---

## Step 0 — there has to be something to count

After the opening in the twin document:

```
 item  | godown         | batch  | bucket   | book_qty   | avg_rate  | stock_value
-------+----------------+--------+----------+------------+-----------+------------
 MILK  | Main Warehouse | B-2604 | SALEABLE |  55.000000 | 28.000000 |     1540.00
 SALT  | Main Warehouse |        | SALEABLE | 120.000000 | 20.000000 |     2400.00
 SUGAR | Main Warehouse |        | SALEABLE |  60.000000 | 42.000000 |     2520.00
```

**This is the only starting point a count can have.** An item with no `stock_balance` row has
no book quantity, so it cannot have a variance — finding it on the shelf is an
`ADJUSTMENT`, not a count line.

---

## Step 1 — the count sheet is generated, not typed

The grain is `stock_balance`'s own: **one line per godown × lot × bucket**, not per item.
Two batches of MILK are two lines to count, because they are two holdings.

```sql
SELECT b.sbl_item_id, b.sbl_lot_id, b.sbl_godown_id, b.sbl_bucket,
       b.sbl_base_uom_id, b.sbl_batch_no, b.sbl_expiry_date,
       b.sbl_on_hand_qty AS book_qty
  FROM stock.stock_balance b
 WHERE b.sbl_company_id = :company AND b.sbl_branch_id = :branch
   AND b.sbl_godown_id  = :godown  AND b.sbl_is_deleted = false;
```

This is why a count line already knows its `svi_lot_id`, where an opening line does not:
the lot is where the book figure came from. `fn_svh_post` uses it as given and never calls
`fn_slt_resolve` for these lines.

---

## Step 2 — the header · `INSERT stock_voucher` · **1 row**

```
        svh_refno        | type     | doc_date   | status | freeze | freeze_from      | freeze_to        | rate_source
-------------------------+----------+------------+--------+--------+------------------+------------------+------------
 PHY/2026-2027/TILL-01/1 | PHYSICAL | 2026-06-30 | DRAFT  | t      | 2026-06-30 20:00 | 2026-06-30 23:00 | MANUAL
```

Three header columns only a count uses: `svh_freeze_stock`, `svh_freeze_from`,
`svh_freeze_to` — *a count that does not freeze the stock it is counting is counting a
moving target.*

> ✔️ *Fixed 2026-09-04*: the freeze is ENFORCED — `tr_sml_freeze_guard` (BEFORE INSERT on
> the ledger) refuses any movement touching the counted godown while this sheet is DRAFT
> and now() is inside the window, except the count's own posting. Wall clock, not document
> date — a back-dated entry still changes today's shelf. Other godowns keep trading, and
> posting or cancelling the sheet lifts the freeze by itself. Verified: refusal in-window,
> other-godown pass, own-post pass, auto-lift.

`svh_reason_id` is worth setting on the header — every variance line inherits it, so the
whole count is explained by "Shrinkage" without touching a line.

---

## Step 3 — the counted lines · `INSERT stock_voucher_item` · **3 rows**

```
 ln | item  | batch  | book_qty   | counted_qty | diff_qty  | qty      | base_qty | cost_rate | value
----+-------+--------+------------+-------------+-----------+----------+----------+-----------+-------
  1 | MILK  | B-2604 |  55.000000 |   55.000000 |  0.000000 | 0.000000 | 0.000000 |  0.000000 |  0.00
  2 | SALT  |        | 120.000000 |  118.000000 | -2.000000 | 0.000000 | 0.000000 |  0.000000 |  0.00
  3 | SUGAR |        |  60.000000 |   63.000000 |  3.000000 | 0.000000 | 0.000000 |  0.000000 |  0.00
```

**`svi_qty`, `svi_base_qty` and `svi_cost_rate` are zero for the entire document.** A count
does not state a quantity of stock; it states what was found. The operator types one number
per line — `svi_counted_qty` — and everything else is either loaded or derived.

`svi_diff_qty` is **GENERATED**: `COALESCE(counted, 0) - COALESCE(book, 0)`. It cannot be
edited to zero to make a variance disappear, and it is signed where every other quantity in
the engine is a magnitude — because it is not a quantity, it is a *difference*.

`svi_book_qty` is a snapshot. It is kept forever so the variance can be defended a year
later, when `stock_balance` has moved on.

---

## Step 4 — Post, first attempt · it is refused

```
ERROR:  line 3/1 of voucher PHY/2026-2027/TILL-01/1 is an inward with no cost rate;
        set one or set svh_rate_source
```

The SUGAR overage is an **inward of 3 bags with no cost**. What are three bags found on a
shelf worth? The schema will not guess, because guessing zero drags the item's moving average
toward zero for every future sale.

**A PHYSICAL screen should default `svh_rate_source` to `AVG_COST`** — found stock is worth
what the rest of that item is worth. `MANUAL` is the wrong default for this voucher type,
though it is the right one for an opening.

```sql
UPDATE stock.stock_voucher SET svh_rate_source = 'AVG_COST' WHERE …;
SELECT stock.fn_svh_post(…);   -- returned 2
```

**Three lines counted, two rows written.**

---

## Step 5 — what Post wrote

### 5a · `stock_ledger` · **2 rows**

```
 ln | item  | txn_type       | dir | qty      | base_qty | signed    | cost_rate | cost_value | reason
----+-------+----------------+-----+----------+----------+-----------+-----------+------------+-------
  2 | SALT  | PHYSICAL_MINUS |  -1 | 2.000000 | 2.000000 | -2.000000 | 20.000000 |      40.00 | t
  3 | SUGAR | PHYSICAL_PLUS  |   1 | 3.000000 | 3.000000 |  3.000000 | 42.000000 |     126.00 | t
```

- **`sml_line_no` is 2 and 3.** Line 1 is missing, deliberately — the ledger keeps the count
  sheet's own numbering, so a variance row can be pointed back at the line it came from.
- The **magnitude** convention is restored: `sml_qty` is 2, not −2. The sign is in
  `sml_direction` alone, and `fn_svh_txn_map` derived both from `sign(svi_diff_qty)`.
- **Neither cost rate was typed.** SUGAR's 42.00 came from `svh_rate_source = AVG_COST` in
  `fn_svh_post`. SALT's 20.00 came from somewhere else entirely: an outward is left at 0 on
  purpose, and the BEFORE trigger `fn_sml_cost_default` stamped it from the item's valuation
  policy. A shortage is valued at **what it cost us**, by policy — never by the counter.

### 5b · `stock_balance`

```
 item  | in_qty     | out_qty  | on_hand    | avg_rate  | stock_value | last_out_date
-------+------------+----------+------------+-----------+-------------+--------------
 MILK  |  55.000000 | 0.000000 |  55.000000 | 28.000000 |     1540.00 |
 SALT  | 120.000000 | 2.000000 | 118.000000 | 20.000000 |     2360.00 | 2026-06-30
 SUGAR |  63.000000 | 0.000000 |  63.000000 | 42.000000 |     2646.00 |
```

MILK is untouched — no row, no timestamp, nothing. A count that agrees leaves no trace on the
balance at all.

### 5c · `stock_item_cost`

```
 item  | total_qty  | total_value | avg_rate
-------+------------+-------------+----------
 SALT  | 118.000000 |     2360.00 | 20.000000
 SUGAR |  63.000000 |     2646.00 | 42.000000
```

The shortage **relieved** value at the average (2400 − 40) and the overage **added** at it
(2520 + 126). Neither moved the average, which is right: a count finds a quantity error, not
a price error. Correcting a *price* is a different problem, and one this engine cannot
express yet.

### 5d · `stock_lot` and the document

`slt_total_on_hand` follows: SALT 118, SUGAR 63, MILK 55.

```
        svh_refno        | status | line_count | total_qty | total_value
-------------------------+--------+------------+-----------+------------
 PHY/2026-2027/TILL-01/1 | POSTED |          3 |  0.000000 |        0.00
```

> ✔️ *Fixed 2026-09-04*: `fn_svh_recompute` now has a PHYSICAL branch that reads the LEDGER
> rows the count posted — the header carries the **net variance** (this example: qty +1,
> value 86.00 = −2×20 + 3×42). A DRAFT count truthfully totals 0. The same fix reordered
> the line write-back in `fn_svh_post` (INSERT … RETURNING first), so a shortage line now
> shows the COGS the trigger stamped — SALT's line reads 20.00, not 0. The captured run
> above predates the fix.

---

## Step 6 — the reconciliation

```
fn_sbl_rebuild → 0 holdings differed
```

```
 item  | opened     | variance  | on_hand
-------+------------+-----------+------------
 MILK  |  55.000000 |         0 |  55.000000
 SALT  | 120.000000 | -2.000000 | 118.000000
 SUGAR |  60.000000 |  3.000000 |  63.000000
```

Because the ledger is append-only, the opening figure and the count's correction both survive
as separate rows for ever. Last year's count can still be explained.

And the line that agreed:

```
 ln | item  | book_qty   | counted_qty | diff_qty  | ledger_rows
----+-------+------------+-------------+-----------+------------
  1 | MILK  |  55.000000 |   55.000000 |  0.000000 |           0
  2 | SALT  | 120.000000 |  118.000000 | -2.000000 |           1
  3 | SUGAR |  60.000000 |   63.000000 |  3.000000 |           1
```

The evidence that MILK was counted and agreed lives on the **document**, not the ledger —
which is exactly why the count sheet is kept and not thrown away after posting.

---

## Opening versus physical, side by side

| | OPENING | PHYSICAL |
|---|---|---|
| starting point | an empty branch | a populated one |
| the grid | typed by the operator | generated from `stock_balance` |
| operator types | item, qty, cost, batch… | **one number per line** — the count |
| `svi_qty` | the quantity | **0** for the whole document |
| what posts | `svi_qty` | `abs(svi_diff_qty)` |
| `svi_lot_id` at draft | NULL — resolved at Post | already known — it came from the balance |
| lines → ledger rows | 1 : 1 | 1 : 1 **or 1 : 0** when the count agrees |
| direction | always +1 | +1 and −1 in the same document |
| txn type | `OPENING` | `PHYSICAL_PLUS` / `PHYSICAL_MINUS` |
| cost on the row | from the line, or `svh_rate_source` | overage: `svh_rate_source`; shortage: `fn_sml_cost_default` |
| sensible `svh_rate_source` | `MANUAL` | **`AVG_COST`** |
| effect on the average | sets it | **leaves it alone** |
| can post zero rows | no — raises | **yes**, and still closes POSTED |
| header totals | correct | **0 — see the warning above** |
| guard | one opening per holding per year | none; a holding may be counted any number of times |
