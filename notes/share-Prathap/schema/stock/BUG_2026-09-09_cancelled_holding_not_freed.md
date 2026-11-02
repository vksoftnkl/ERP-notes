# Bug — a cancelled opening never frees its holding

> **STILL PRESENT as of 2026-09-09 evening**, re-confirmed after the day's other backend
> fixes landed (draft cancel, supplierName, physical header totals, and the supplier /
> sale-price validation all went in — this one did not). Fresh reproduction below on a
> holding created minutes before the test, so no historical data is involved.

**Severity:** blocks the correction workflow entirely. Cancel a wrong opening and the
right one can never be entered. There is no workaround short of inventing a fake batch.

**Where:** the "already has an opening in this year" check, in the NestJS layer — it is
raised by both `GET /stock/opening/validate` and `POST /stock/opening/post`.
(`fn_svh_post` is not deployed, so the SQL in `19_stock_posting.sql` is not what runs.)

---

## Reproduce

```
1. Open a holding and POST it.                  -> ok
2. CANCEL that opening.                         -> ok, reversing rows written
3. Open the SAME holding again and POST.        -> REFUSED:
     "Line 1 (…): this holding already has an opening in this year"
```

Verified twice on 2026-09-09 against 192.168.0.106:
* `ZZ TEST N (NONE)` in Coimbatore (historical data), and
* `ZZ TEST BE` batch `CANCELPROBE-01` in namakkal — a holding created, posted, cancelled
  and re-opened inside one minute. Its ledger is one forward row and one reversal, net
  zero, every forward row reversed. `/validate` still answers
  "this holding already has an opening in this year" and the post refuses.

## It is NOT "any voucher referencing the holding"

A DRAFT that was cancelled without ever posting does **not** block: re-opening that same
holding posts cleanly. So the check reads the LEDGER, not `stock_voucher`. Isolated with
batch `ISOLATE-01` in namakkal — cancelled draft, 0 ledger rows, re-open POSTED.

## The defect: the reversal exclusion is missing

Every forward row for the blocked holding carries a matching reversal:

```
OPN0005  is_reversal=false  has_reversal=true      <- still blocks
OPN0005  is_reversal=true   has_reversal=false
```

Run against that holding, the two candidate predicates disagree:

```
without the exclusion -> 1 row  (blocks)
with    the exclusion -> 0 rows (free)      <- correct
```

## The fix

The check must exclude a forward row that has been reversed. This is exactly what
`19q_opening_stock_queries.sql` (Q2) and `fn_svh_post` both specify, and what the
NestJS reimplementation dropped:

```sql
AND l.sml_txn_type    = 'OPENING'
AND l.sml_is_deleted  = false
AND l.sml_is_reversal = false
AND NOT EXISTS (                      -- ← THE MISSING CLAUSE
      SELECT 1
        FROM stock.stock_ledger r
       WHERE r.sml_reverses_id = l.sml_id
         AND r.sml_acc_year    = l.sml_acc_year
         AND r.sml_is_deleted  = false)
```

Apply it in BOTH places — `/validate` and the post guard — or the preflight will pass a
line that the post then refuses.

`sml_reverses_id` is populated correctly by the cancel route, so no data repair is
needed: the clause starts working the moment it is added.

## Regression test

`schema/stock/19t_verify_opening.sql` check 43 ("CANCELLED: holding is free for a new
opening") already asserts the ledger side and PASSES today — it is the API that
disagrees with the ledger. Worth adding an end-to-end case: post -> cancel -> re-open,
which is the sequence above.
