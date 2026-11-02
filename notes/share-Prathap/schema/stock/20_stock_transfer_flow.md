# A transfer, table by table — godown → godown, and branch → branch

Companion to `20_stock_transfer.sql`, the way `19_opening_stock_flow.md` is the companion
to `19_stock_posting.sql`. Same two items, and it picks up exactly where that document
stopped: MILK (batch `B-2604`, expiry 30/06/2026) 55 on hand at 28.00, SALT 120 at 20.00,
both in godown **MAIN** of branch **A**.

> **Provenance.** The figures below are **worked from the function bodies**, not captured
> from a run — unlike `19_opening_stock_flow.md`, whose numbers came off a live cluster.
> `20_stock_transfer.sql` was smoke-verified on 2026-09-04 (pair + cancel, despatch,
> partial receive with damage, short left open, rebuild clean); this document reconstructs
> that cycle line by line so the API and the screen have something to be checked against.
> Re-run it against a real cluster when one is up and mark it captured.
>
> Read `REVIEW_2026-09-05.md` alongside this: six MUST-FIX engine defects were reproduced
> there, four of them on this path, and they are marked ⚠️ where they touch a step below.

Two shapes, one voucher type. The engine decides which by looking at `svh_to_branch_id`:

```
v_same := svh_to_branch_id IS NULL OR svh_to_branch_id = svh_branch_id
```

Same branch → the OUT and IN ledger rows are written as a **pair, in one transaction**, and
the voucher is POSTED. Another branch → OUT rows plus a `stock_transit` row per line, and
the voucher is **IN_TRANSIT** until somebody receives it. Nothing else about the two paths
differs, which is why they are one function and not two.

---

# Shape A — godown → godown (same branch)

## A0 — the starting point

```
 branch | godown | item | lot(batch) | on_hand | avg_rate  | value
--------+--------+------+------------+---------+-----------+---------
 A      | MAIN   | MILK | B-2604     |      55 | 28.000000 | 1540.00
 A      | MAIN   | SALT | (none)     |     120 | 20.000000 | 2400.00
```

## A1 — the screen picks a holding, not an item

A transfer moves **existing** stock, so the grid is filled from `stock_balance` the way a
count sheet is — item, godown, **lot**, bucket, on-hand. The lot is not a nicety:

```
IF ln.svi_lot_id IS NULL THEN
    RAISE 'line %/% of voucher % names no lot — a transfer moves existing stock; pick it from the balance'
```

`fn_slt_resolve` is never called here. A transfer moves identity, it does not create it — so
the destination gets **the same `slt_id`**, and ageing does not reset. That is the point of
the rule, not a side effect of it.

## A2 — the header · `INSERT stock_voucher` · **1 row**

Move 20 MILK from MAIN to COLD.

```
        svh_refno        | svh_voucher_type | from_godown | to_godown | to_branch | svh_status
-------------------------+------------------+-------------+-----------+-----------+-----------
 TRF/2026-2027/TILL-01/1 | TRANSFER_OUT     | MAIN        | COLD      | (null)    | DRAFT
```

`svh_to_branch_id` NULL is what makes this shape A. Both godowns are mandatory
(`ck_svh_transfer_godowns`), and the function refuses a godown transferring to itself.

**One OUT voucher has one destination godown.** Two destinations are two vouchers.

## A3 — the lines · `INSERT stock_voucher_item` · **1 row**

```
 ln | item | godown | lot     | bucket   | qty       | base_qty  | cost_rate
----+------+--------+---------+----------+-----------+-----------+-----------
  1 | MILK | MAIN   | B-2604  | SALEABLE | 20.000000 | 20.000000 |  0.000000
```

`svi_godown_id` on an OUT line is the **source**. The destination is on the header. (At
receipt this flips — see B5, and do not let the two screens share a field name.)

Cost is left at 0 deliberately: the engine stamps it. Anything typed here is honoured
instead, which is not what a transfer wants.

## A4 — Despatch

```sql
SELECT stock.fn_svh_post_transfer(:voucher_id, :acc_year, :user_id);   -- returns 2
```

### A4a · the OUT row — `stock_ledger` · **1 row**

Inserted with `sml_cost_rate = 0`, and `fn_sml_cost_default` fills it BEFORE INSERT from the
item's valuation policy — WAVG here, so the branch's moving average, 28.000000. The INSERT
uses `RETURNING sml_cost_rate, sml_cost_rate_wot`, so what the ledger actually wrote is what
the next two rows carry. **Cost travels with the stock; nobody re-enters it.**

```
 ln | txn_type     | dir | godown | base_qty  | cost_rate | cost_value
----+--------------+-----+--------+-----------+-----------+------------
  1 | TRANSFER_OUT |  -1 | MAIN   | 20.000000 | 28.000000 |     560.00
```

### A4b · the IN row — `stock_ledger` · **1 more row**, same transaction

```
 ln | txn_type    | dir | godown | base_qty  | cost_rate | cost_value
----+-------------+-----+--------+-----------+-----------+------------
  1 | TRANSFER_IN |   1 | COLD   | 20.000000 | 28.000000 |     560.00
```

`sml_batch_no`, `sml_expiry_date` and `sml_mrp` on both rows are read **from the lot**, not
from the line — same rule as every other posting path.

### A4c · `tr_sml_apply` moves the balance

```
 branch | godown | item | on_hand | transit_in
--------+--------+------+---------+-----------
 A      | MAIN   | MILK |      35 |          0
 A      | COLD   | MILK |      20 |          0
```

The COLD row is created by the trigger if that godown never held the item. Value follows the
same rate on both sides, because nothing revalued.

### A4d · `stock_item_cost` — deliberately unchanged

```
 branch | item | total_qty | total_value | avg_rate
--------+------+-----------+-------------+-----------
 A      | MILK |        55 |     1540.00 | 28.000000
```

The moving average is a property of the **item in the branch**, and the stock never left the
branch: −560.00 out, +560.00 in. An internal move is not a revaluation.

### A4e · the document closes out

The lines are written back with the resolved base quantity and the cost the ledger actually
took, `fn_svh_recompute` re-sums the header, and:

```
        svh_refno        | svh_status | line_count | total_qty | total_value
-------------------------+------------+------------+-----------+-------------
 TRF/2026-2027/TILL-01/1 | POSTED     |          1 | 20.000000 |      560.00
```

**POSTED, not IN_TRANSIT.** Shape A never touches `stock_transit` — there is no lorry.

## A5 — cancel

Nothing special: a same-branch transfer is an ordinary POSTED document and the generic
`fn_svh_cancel` reverses both halves symmetrically. The transfer cancel guard does not fire,
because it only refuses IN_TRANSIT / RECEIVED and POSTED TRANSFER_IN.

---

# Shape B — branch → branch

## B1 — the despatch header

Send 30 MILK from branch A (godown MAIN) to branch B (godown STORE).

```
        svh_refno        | svh_voucher_type | from_godown | to_godown | to_branch | svh_status
-------------------------+------------------+-------------+-----------+-----------+-----------
 TRF/2026-2027/TILL-01/2 | TRANSFER_OUT     | MAIN        | STORE     | B         | DRAFT
```

Lines exactly as in A3, quantity 30. **A destination godown of branch B is named on the
sender's document** — the sender decides where it lands, and the receiving screen inherits it.

## B2 — Despatch · `fn_svh_post_transfer` · returns **1** (one row per line, not two)

### B2a · the OUT row only

```
 ln | txn_type     | dir | branch | godown | base_qty  | cost_rate | cost_value
----+--------------+-----+--------+--------+-----------+-----------+------------
  1 | TRANSFER_OUT |  -1 | A      | MAIN   | 30.000000 | 28.000000 |     840.00
```

Branch A's balance drops to 25. **At the source the stock is simply gone** — it is not held
anywhere as "in transit". Only the destination carries transit.

### B2b · `stock_transit` · **1 row per line**

```
 from | to | item | lot    | bucket   | sent | received | damage | short | cost_rate | value  | status
------+----+------+--------+----------+------+----------+--------+-------+-----------+--------+------------
 A    | B  | MILK | B-2604 | SALEABLE |   30 |        0 |      0 |    30 | 28.000000 | 840.00 | IN_TRANSIT
```

`stt_short_qty` is **generated** — `sent − received − damage`. Nobody maintains it, and it is
the whole loss report.

### B2c · `tr_stt_refresh` tells branch B it is coming

```
 branch | godown | item | lot    | on_hand | transit_in
--------+--------+------+--------+---------+-----------
 B      | STORE  | MILK | B-2604 |       0 |         30
```

The row is created by `fn_sbl_ensure_row` when B has never held this item — the commonest
first-transfer case, and without it the inbound quantity would be invisible until the goods
arrived. `sbl_transit_in_qty` counts only transit rows still IN_TRANSIT or PARTIAL.

### B2d · the voucher

```
        svh_refno        | svh_status | total_qty | total_value
-------------------------+------------+-----------+-------------
 TRF/2026-2027/TILL-01/2 | IN_TRANSIT | 30.000000 |      840.00
```

**An inter-branch OUT never reaches POSTED.** It goes DRAFT → IN_TRANSIT → RECEIVED. Any
report or list filtering on `svh_status = 'POSTED'` will silently lose every transfer in
flight.

## B3 — branch B's worklist

Two ways in, both already indexed:

- `ix_svh_inbound` — `svh_to_branch_id`, `svh_status`, `svh_doc_date`: the vouchers sent to
  me and not yet closed.
- **Q14** in `16q_stock_grid_queries.sql` — transit rows with days-in-flight, short quantity
  and `short × cost_rate` as loss value.

## B4 — the receipt header · a **TRANSFER_IN** voucher at branch B

```
 svh_refno               | type        | branch | from_godown | to_godown | link_src_doc_id
-------------------------+-------------+--------+-------------+-----------+-----------------
 TRI/2026-2027/TILL-07/1 | TRANSFER_IN | B      | MAIN        | STORE     | <the OUT's id>
```

Three things the header must carry, none of them optional:

1. **The link** — `svh_link_src_module='STOCK'`, `svh_link_src_doc_type='STOCK_VOUCHER'`,
   `svh_link_src_doc_id`, and `svh_link_src_acc_year` when the receipt lands in a different
   accounting year from the despatch. `ck_svh_transfer_in_link` enforces the id; the function
   checks the module and doc type by hand and refuses anything else.
2. **Both godowns**, because `ck_svh_transfer_godowns` applies to TRANSFER_IN too. Only the
   line godown matters to the matcher; copy the sender's from the OUT.
3. **`svh_branch_id` = the OUT's `svh_to_branch_id`.** Receiving somebody else's transfer is
   refused by name.

## B5 — the receipt lines: 25 arrived good, 3 arrived broken, 2 never arrived

```
 ln | item | godown | lot    | bucket   | qty
----+------+--------+--------+----------+-----
  1 | MILK | STORE  | B-2604 | SALEABLE |  25
  2 | MILK | STORE  | B-2604 | DAMAGED  |   3
```

**`svi_godown_id` here is the DESTINATION** — the opposite of the OUT line. The matcher is
`(out voucher, item, lot, svi_godown_id = stt_to_godown_id)`, and it insists on finding
exactly one transit row:

- none → `no_data_found`, *"nothing of this lot is in transit to this godown"*;
- more than one → `feature_not_supported`, *"shipped in more than one bucket; ship as
  separate transfers"*.

There is no line for the missing 2. **A short is not keyed, it is what remains unkeyed.**

## B6 — Receive

```sql
SELECT stock.fn_svh_receive_transfer(:in_voucher_id, :acc_year, :user_id);   -- returns 2
```

### B6a · the IN rows — at the cost the stock LEFT with

```
 ln | txn_type    | dir | godown | bucket   | base_qty  | cost_rate | cost_value
----+-------------+-----+--------+----------+-----------+-----------+------------
  1 | TRANSFER_IN |   1 | STORE  | SALEABLE | 25.000000 | 28.000000 |     700.00
  2 | TRANSFER_IN |   1 | STORE  | DAMAGED  |  3.000000 | 28.000000 |      84.00
```

The rate comes from `stt_cost_rate` — the figure stamped on the OUT row weeks earlier.
Branch B never enters a cost and cannot revalue by receiving.

> ⚠️ **Known asymmetry — already logged.** `sml_cost_rate` is taken from the transit row, but
> `sml_cost_rate_wot` is read from `stock_lot` *now* (`lot.slt_cost_rate_wot`), i.e. the lot's
> frozen first-arrival figure. The two rates on one ledger row therefore describe different
> moments. The same-branch path has no such gap — it carries both out of the OUT row's
> `RETURNING`. `REVIEW_2026-09-05.md` proposes the fix: add `stt_cost_rate_wot` to
> `stock_transit`. Until then, do not reconcile `cost_value_wot` across a transit.

Damaged units **post in**: they physically exist, broken, so they land in branch B's DAMAGED
bucket rather than vanishing. A line is treated as damage only when `svi_bucket = 'DAMAGED'`
and the transit row was not itself a shipment of damaged stock.

### B6b · the transit row is settled

```
 sent | received | damage | short | status
------+----------+--------+-------+---------
   30 |       25 |      3 |     2 | PARTIAL
```

### B6c · balances at B

```
 godown | bucket   | on_hand | transit_in
--------+----------+---------+-----------
 STORE  | SALEABLE |      25 |          2
 STORE  | DAMAGED  |       3 |          0
```

`transit_in` fell to 2 — the trigger re-sums the remaining quantity of every row still
IN_TRANSIT or PARTIAL.

### B6d · the two vouchers

```
 svh_refno               | type         | status     | note
-------------------------+--------------+------------+------------------------------------------
 TRI/2026-2027/TILL-07/1 | TRANSFER_IN  | POSTED     | the receipt is finished
 TRF/2026-2027/TILL-01/2 | TRANSFER_OUT | IN_TRANSIT | 2 still outstanding — NOT closed
```

The OUT flips to RECEIVED only when **no** transit row of it has anything left:

```sql
IF NOT EXISTS (SELECT 1 FROM stock.stock_transit
                WHERE stt_out_voucher_id = o.svh_id … AND stt_sent_qty - stt_received_qty - stt_damage_qty > 0)
```

So the short keeps the transfer open, on `ix_svh_inbound` and on Q14, at a loss value of
2 × 28.00 = **56.00** — until somebody decides whose it is.

## B7 — how it ends

Either the 2 turn up (a second TRANSFER_IN against the same OUT: remaining 2 → received 27,
damage 3, transit RECEIVED, and the OUT flips to RECEIVED), or the business writes them off
with a DAMAGE / ADJUSTMENT voucher at whichever branch it blames. **The write-off is
deliberately not automated** — the engine will not decide who pays for a missing carton.

> ⚠️ **But nothing closes the short either.** There is no `fn_stt_settle_short`: a transit row
> with an unreceived remainder stays PARTIAL for ever, the OUT stays IN_TRANSIT for ever, and
> `fn_stt_refresh` keeps that remainder in the destination's `sbl_transit_in_qty` — stock the
> branch is told is coming and never will. A write-off voucher at either branch does **not**
> touch the transit row. Top item in `REVIEW_2026-09-05.md`; until it exists, the API needs a
> settle path of its own or the inbound worklist silently accumulates ghosts.

## B8 — what cancel refuses here

| Attempt | Result |
|---|---|
| Cancel a POSTED **TRANSFER_IN** | Refused — *"un-receiving is a reverse transfer back to the sender, not a cancellation"*. Reversing the ledger while the transit row still says received is a silent fork |
| Cancel an **IN_TRANSIT / RECEIVED** TRANSFER_OUT | Refused — *"goods that left cannot be cancelled on paper"* |
| Cancel a same-branch POSTED transfer | Allowed, reverses both halves |
| Edit any of them after post | Refused by `tr_svh_post_lock` / `tr_svi_post_lock` |

A mid-flight mistake is fixed by receiving what arrived and transferring it back.

---

## The tally

| Table | Shape A (1 line) | Shape B (1 line, then a receipt) | Who wrote it |
|---|---|---|---|
| `stock_voucher` | 1 | 1 OUT + 1 IN | **the app** |
| `stock_voucher_item` | 1 | 1 + 2 | **the app** |
| `stock_ledger` | 2 | 1 at despatch, 2 at receipt | `fn_svh_post_transfer` / `fn_svh_receive_transfer` |
| `stock_transit` | 0 | 1, settled in place | `fn_svh_post_transfer`, updated at receipt |
| `stock_balance` | 2 godown rows moved | source −30, destination +transit, then +stock | trigger `fn_sml_apply` + `fn_stt_refresh` |
| `stock_lot` | 0 | 0 | **nothing** — the lot is reused, never re-resolved |
| `stock_item_cost` | unchanged | A drops, B gains, at the same rate | trigger `fn_sml_apply` |

## The order, in one line

```
A:  pick holdings → voucher → lines → [DESPATCH] → OUT row + IN row → balance moves → POSTED
B:  pick holdings → voucher → lines → [DESPATCH] → OUT row + transit row → B sees transit_in → IN_TRANSIT
                                    → receipt voucher (linked) → [RECEIVE] → IN rows + transit settled
                                    → PARTIAL / RECEIVED → OUT closes only when nothing is left
```

## Twelve things that bite

1. **The lot is mandatory** on both documents. A transfer picks from the balance; it never
   resolves identity.
2. **`svi_godown_id` means source on the OUT and destination on the IN.** Same column, two
   meanings, one screen away from each other.
3. **The TRANSFER_IN header still needs both godowns** even though only the line godown is
   used — `ck_svh_transfer_godowns` does not know the difference.
4. **The transit key and the receive matcher disagree.** `ux_stt_out_line` is
   `(out voucher, item, lot, to_godown, **bucket**)` — it deliberately allows one voucher to
   ship saleable and damaged units of the same lot. The receive matcher looks up
   `(out voucher, item, lot, to_godown)` **without bucket**, finds two rows, and refuses with
   `0A000`. So the shipment saves and despatches, and only the receipt fails. Ship two
   vouchers, and validate it at save (see the plan's §3.2).
   The same index also aborts a legitimate despatch: two lines of the same item+lot from two
   *source* godowns collide, because the destination godown is header-level and drops out of
   the key — raw `23505`, no worded message (`REVIEW_2026-09-05.md`, MUST-FIX 3).
5. **An inter-branch OUT is never POSTED.** Status filters must include IN_TRANSIT and
   RECEIVED or transfers disappear from the list.
6. **`cost_rate_wot` at receipt comes from the lot, not the transit row** (see B6a).
7. **Idempotency is `ux_sml_source`'s.** OUT and IN of one line differ in direction and
   godown, so a retry after a timeout cannot move stock twice — retry freely.
8. **`stock_transit` carries `stt_lr_no`, `stt_vehicle_no` and `stt_expected_on`, and
   `fn_svh_post_transfer` never sets them.** The despatch screen has nowhere to record the
   lorry today; the columns exist and are written by nothing. Either the function grows three
   parameters or the API updates the transit rows after the call.
9. **Partial receipts are more vouchers**, never an edit of a posted one.
10. **A DAMAGED shipment received as SALEABLE launders it.** The bucket check only handles the
    other direction (`svi_bucket = 'DAMAGED'` against a non-damaged transit row). Receiving
    damaged stock as saleable posts it clean into the destination, silently
    (`REVIEW_2026-09-05.md`, SHOULD-FIX).
11. **An IN_TRANSIT OUT is not frozen.** `fn_svh_post_lock` exempts IN_TRANSIT and RECEIVED, so
    a plain UPDATE can regress it to DRAFT — and a second despatch then moves the stock twice.
    The API must never write `svh_status` directly; only the two functions may.
12. **The transit row keeps only the LAST receiving voucher's id.** After two partial receipts
    `stt_in_voucher_id` names the second one; the first is discoverable only through the
    ledger. Do not build "which receipt closed this" on that column.

