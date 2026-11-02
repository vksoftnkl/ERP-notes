# Opening stock, from an empty database — what happens, table by table

Companion to `19_stock_posting.sql`. Every figure below was **captured from a real run** on a
fresh PostgreSQL 18 cluster with `00_init.sql` → `16_stock.sql` → `19_stock_posting.sql`
loaded and nothing else. Nothing here is illustrative.

Seven `stock.*` tables end up with rows. **Only three of them are written by the
application.** The rest write themselves.

---

## Step 0 — the starting point

```
stock_track_policy = 0     stock_lot     = 0     stock_balance   = 0
stock_voucher      = 0     stock_ledger  = 0     stock_item_cost = 0
stock_voucher_item = 0
```

---

## Step 1 — masters, before the screen can even open

None of these are `stock.*`. All are Prisma-owned and this module only reads them.

| Table | Why it must exist first |
|---|---|
| `public.companys` | tenant |
| `public.branch_master` | the shop, and `br_default_godown_id` is what fills the godown picker |
| `inventory.godown_locations` | **every** movement needs one; an unlocated movement cannot be reversed, counted or transferred |
| `fixed.device_master` | `svh_device_id` is NOT NULL — the device *is* the counter, so an offline till can number its own document |
| `public.user_master` | who did it |
| `inventory.item_master` | what |
| `inventory.item_unit_conversion` | **`iuc_id`, not `item_unit_master.unit_id`** — this is the mistake the schema warns about twice |

---

## Step 2 — `stock_track_policy`: what each item is tracked *by*

Typed once per item that needs it. MILK is batch + expiry. SALT gets **no row at all**, and
that is a complete answer, not a gap:

```
 item_code | track_batch | track_expiry | track_mrp | signature | valuation | issue | negative
-----------+-------------+--------------+-----------+-----------+-----------+-------+---------
 MILK      | t           | t            | f         | BE        | WAVG      | FEFO  | ALLOW
 SALT      | f           | f            | f         | N         | WAVG      | FIFO  | ALLOW
```

`fn_stp_effective` falls back to "track nothing, WAVG, FIFO, ALLOW" when it finds no row.
An untracked item is not a special case anywhere below — it is an item whose lot identity
happens to be all-NULL.

> ✔️ *Fixed 2026-09-04*: the fallback now returns `issue_strategy = FEFO`, matching the
> column default, so saving a policy row no longer flips an item's strategy. (The captured
> run above predates the fix — SALT's resolved `issue` column would now read FEFO.)

---

## Step 3 — the user types the header · `INSERT stock_voucher` · **1 row**

```
        svh_refno        | svh_voucher_type | svh_doc_date | svh_status | lines | total_qty | total_value
-------------------------+------------------+--------------+------------+-------+-----------+------------
 OPN/2026-2027/TILL-01/1 | OPENING          | 2026-04-01   | DRAFT      |     0 |  0.000000 |       0.00
```

Written by the app. And that is all that happened:

```
stock_lot=0   stock_ledger=0   stock_balance=0   stock_item_cost=0
```

**A draft moves no stock.** It is a piece of paper.

---

## Step 4 — the user types the lines · `INSERT stock_voucher_item` · **2 rows**

Line 1 SALT keyed in **BOX** (12 per box), untracked. Line 2 MILK in PIECE, batch `B-2604`
expiring 30/06/2026, with 5 free.

```
 ln | qty       | base_qty   | free_base | batch  | cost_rate | value   | lot_id
----+-----------+------------+-----------+--------+-----------+---------+--------
  1 | 10.000000 | 120.000000 |  0.000000 |        | 20.000000 | 2400.00 | (null)
  2 | 50.000000 |  50.000000 |  5.000000 | B-2604 | 28.000000 | 1540.00 | (null)
```

`svi_value` is **GENERATED** — `(base_qty + free_base_qty) × cost_rate`. So line 1 is
120 × 20 = 2,400.00, **not** 10 × 20. The cost rate is per *base* unit.

Then something happened without being asked. `tr_svi_refresh_header` fired and re-summed the
header:

```
 line_count | total_qty  | total_value | status
------------+------------+-------------+--------
          2 | 175.000000 |     3940.00 | DRAFT
```

**The screen never sums its own grid.** 175 = 120 + 50 + 5 — free goods are stock.

`svi_lot_id` is still NULL on both lines. That is not missing data: the lot is resolved at
Post, and a NULL here means exactly "not posted yet".

---

## Step 5 — Post

```sql
SELECT stock.fn_svh_post(:voucher_id, :acc_year, :user_id);   -- returned 2
```

One call. Five things happen inside it, in this order.

### 5a · `fn_slt_resolve` creates the lots — `stock_lot` · **2 rows**

```
 item | sig | batch_no | expiry     | key_batch | key_mrp | key_expiry | cost_rate | on_hand | status
------+-----+----------+------------+-----------+---------+------------+-----------+---------+--------
 MILK | BE  | B-2604   | 2026-06-30 | B-2604    |      -1 | 2026-06-30 | 28.000000 |      55 | ACTIVE
 SALT | N   |          |            | ~         |      -1 | 0001-01-01 | 20.000000 |     120 | ACTIVE
```

This is the load-bearing idea of the whole engine. SALT tracks nothing, so its identity
normalises to the sentinels `('~', -1, -1, 0001-01-01, '~', nil-uuid)` — and because
`ux_slt_identity` is built over the *generated* companion columns and not the raw ones, SALT
finds that same single lot on every future receipt instead of getting a fresh one each time.
A UNIQUE index treats each NULL as distinct; the sentinels are what stop that.

`slt_total_on_hand` was 0 when the lot was created a microsecond earlier. The trigger in 5c
filled it.

### 5b · the movements — `stock_ledger` · **2 rows**, append-only

```
 ln | txn_type | dir | qty       | base_qty   | free_base | signed     | cost_rate | rate_wot  | cost_value | batch  | expiry
----+----------+-----+-----------+------------+-----------+------------+-----------+-----------+------------+--------+-----------
  1 | OPENING  |   1 | 10.000000 | 120.000000 |  0.000000 | 120.000000 | 20.000000 | 19.047619 |    2400.00 |        |
  2 | OPENING  |   1 | 50.000000 |  50.000000 |  5.000000 |  55.000000 | 28.000000 | 26.666667 |    1540.00 | B-2604 | 2026-06-30
```

- Quantities are **magnitudes**; the sign lives only in `sml_direction`.
- `sml_signed_base_qty` is GENERATED — `direction × (base + free)`. It is the one column
  every quantity report sums.
- `rate_wot` was derived from `svi_tax_perc` (5%): 20 ÷ 1.05 = 19.047619. Nobody typed it.
- `batch_no` / `expiry` were copied **from the lot**, not from the line — `stock_lot` owns
  them, and `fn_slt_resolve` may legitimately have blanked what the operator typed.

### 5c · `tr_sml_apply` builds the balance — `stock_balance` · **2 rows**

Created from nothing by the trigger. The application never writes this table.

```
 item | in_qty | free_in | out_qty | on_hand | avail | avg_rate  | stock_value | batch  | first_in
------+--------+---------+---------+---------+-------+-----------+-------------+--------+-----------
 MILK |     50 |       5 |       0 |      55 |    55 | 28.000000 |     1540.00 | B-2604 | 2026-04-01
 SALT |    120 |       0 |       0 |     120 |   120 | 20.000000 |     2400.00 |        | 2026-04-01
```

`sbl_on_hand_qty` and `sbl_available_qty` are both GENERATED. The same trigger also set
`slt_total_on_hand` and the lot's ACTIVE/CLOSED status, and checked
`stp_allow_negative` — which an inward always passes.

### 5d · the same trigger builds the cost — `stock_item_cost` · **2 rows**

```
 item | total_qty | total_value | value_wot | avg_rate  | last_purchase | last_purchase_date
------+-----------+-------------+-----------+-----------+---------------+-------------------
 MILK |        55 |     1540.00 |   1466.67 | 28.000000 |     28.000000 | 2026-04-01
 SALT |       120 |     2400.00 |   2285.71 | 20.000000 |     20.000000 | 2026-04-01
```

This is the moving weighted average, and it is a property of the **item in the branch**, not
of the lot — which is why a receipt revalues *every* lot of that item, not only the one that
moved. On an opening there is nothing to average against yet, so the average is simply the
rate keyed.

### 5e · back in `fn_svh_post` — the document is closed out

The lines get their lot, and anything the function resolved is written back so the document
shows what the ledger actually received:

```
 ln | lot_now_set | base_qty   | cost_rate | rate_wot  | value
----+-------------+------------+-----------+-----------+---------
  1 | t           | 120.000000 | 20.000000 | 19.047619 | 2400.00
  2 | t           |  50.000000 | 28.000000 | 26.666667 | 1540.00
```

```
        svh_refno        | svh_status | line_count | total_qty  | total_value | posted
-------------------------+------------+------------+------------+-------------+--------
 OPN/2026-2027/TILL-01/1 | POSTED     |          2 | 175.000000 |     3940.00 | t
```

From this instant `tr_svi_post_lock` and `tr_svh_post_lock` refuse every edit, and
`tr_sml_immutable` refuses every UPDATE and DELETE of the ledger rows. The only thing left
that can be done to this document is cancel it.

---

## Step 6 — the whole tally

| Table | Rows | Who wrote it |
|---|---|---|
| `stock_track_policy` | 1 | **typed by hand**, once per tracked item |
| `stock_voucher` | 1 | **the app** — the screen header |
| `stock_voucher_item` | 2 | **the app** — the grid |
| `stock_lot` | 2 | `fn_slt_resolve`, at Post |
| `stock_ledger` | 2 | `fn_svh_post`, at Post |
| `stock_balance` | 2 | trigger `fn_sml_apply` |
| `stock_item_cost` | 2 | trigger `fn_sml_apply` |

Three rows of application writing. Six rows that maintained themselves.

```
fn_sbl_rebuild → 0 holdings differed
```

The cache and the ledger agree.

> ⚠️ `fn_sbl_rebuild` re-derives **quantities only**. It never recomputes `sbl_stock_value`,
> `sbl_avg_cost_rate` or `stock_item_cost`, so it can return 0 while a valuation is wrong.
> There is no value-side assertion yet.

---

## The order, in one line

```
masters → policy → voucher → lines → [POST] → lot → ledger → balance + lot total + moving average → POSTED
                                       └── one call: fn_svh_post ──┘
                                              └─ everything from "ledger" rightward is triggers ─┘
```
