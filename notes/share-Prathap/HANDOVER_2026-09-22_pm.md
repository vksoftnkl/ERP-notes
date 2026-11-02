# Handover — 2026-09-22, afternoon.  Two fixes, then you are blocked on us.

**Standalone. You do not need the morning file.** Everything you need is here,
with the exact file:line, and the SQL to prove each fix yourself.

---

## 0 · Where things stand — you built it and it WORKS

A3, A4 and A5 are all in and deployed: **373 routes**. I verified the whole
chain against the database this afternoon, not the logs.

| check | result |
|---|---|
| `sbl_on_hand_qty` = Σ `sml_signed_base_qty` per holding | **0 mismatches / 28 holdings** |
| Σ `sbl_stock_value` = `sic_total_qty × sic_avg_cost_rate` | **0 drifted / 28 items** |
| `slt_total_on_hand` = Σ its balance rows | **0 mismatches / 28 lots** |
| every voucher balanced — summed from `acc_vouchers`, not the stamped header | **0 unbalanced / 63** |
| negative on-hand anywhere | **0** |
| any ledger row with zero cost rate or value | **0, across all six txn types** |
| cancel reverses | POSTED 11 bills / 11 rows / net −110 · CANCELLED 8 bills / **16 rows** (8 + 8 reversals) / net **0** |
| bill posts its legs | **8 balanced, role-driven** — COGS, INVENTORY, SALES, OUTPUT_CGST, OUTPUT_SGST + party + tender |

The stock engine and the posting engine are correct. **This file is two
defects, not a rebuild.**

Also done, and good: the stale `test/bill-posted-http.e2e-spec.ts` is fixed —
the comment naming the old defect is exactly right.

---

## FIX 1 · `svh_total_value` is 0.00 on every generated stock voucher

### What is wrong

| voucher | count | header `svh_total_value` | real value in the ledger |
|---|---|---|---|
| ISSUE (bill / challan) POSTED | 11 | **0.00** | 2,200.00 |
| ISSUE CANCELLED | 12 | **0.00** | 4,000.00 |
| RECEIPT (returns) POSTED | 6 | **0.00** | 420.00 |
| RECEIPT CANCELLED | 6 | **0.00** | 1,320.00 |

`svh_total_qty` rolls up fine. OPENING and PHYSICAL — the vouchers a user keys —
are **correct**. Only the service-generated path is wrong.

### The exact line

**`src/modules/sales/posting/sales-stock.service.ts:355`** — the INSERT that
creates the generated stock voucher. Its column list is:

```
svh_line_count, svh_total_qty, svh_status, svh_rate_source, svh_remarks, …
```

`svh_total_value` and `svh_total_value_wot` **are not in the list at all**, so
they take the column default of 0.00 and nothing ever writes them.

(`stock-voucher.service.ts:1001` does set them — that is the user-keyed save
path, which is why openings are fine.)

### Do NOT fix it by adding the column to that INSERT

At that point the line carries `${l.costRate ?? 0}`. **The cost is not known
yet** — the engine resolves it at POST (FIFO / AVG_COST) and stamps it onto the
ledger row. Writing a total there would store zeros with extra steps.

### The fix

Roll it up **after posting, from `stock_ledger`, in the same transaction** —
the same place the post already runs:

```sql
UPDATE stock.stock_voucher svh
   SET svh_total_value     = COALESCE(t.v,  0),
       svh_total_value_wot = COALESCE(t.vw, 0)
  FROM (SELECT SUM(sml_cost_value)     AS v,
               SUM(sml_cost_value_wot) AS vw
          FROM stock.stock_ledger
         WHERE sml_src_doc_id  = :svhId
           AND sml_acc_year    = :accYear
           AND sml_is_deleted  = false
           AND sml_is_reversal = false) t
 WHERE svh.svh_id = :svhId AND svh.svh_acc_year = :accYear;
```

**`sml_is_reversal = false`** matters: without it a cancelled voucher sums its
own reversals back in and lands on zero for a different reason.

### Why it matters even though the costing is right

Nothing is mis-valued — the ledger is correct. But **any stock voucher list,
report or print reading `svh_total_value` shows zero for every sale issue and
every return receipt.** Someone reconciles against it and concludes the goods
went out free. It is wrong in a place a human reads, which in a money system is
the worst kind of wrong.

### Prove it — run this after the fix

```sql
SELECT svh.svh_refno, svh.svh_total_value AS header,
       SUM(sml.sml_cost_value) AS ledger
  FROM stock.stock_voucher svh
  JOIN stock.stock_ledger sml
    ON sml.sml_src_doc_id = svh.svh_id
   AND sml.sml_is_deleted = false AND sml.sml_is_reversal = false
 WHERE svh.svh_voucher_type IN ('ISSUE','RECEIPT')
 GROUP BY 1, 2
HAVING svh.svh_total_value IS DISTINCT FROM SUM(sml.sml_cost_value);
```

**Must return zero rows.** Run it now, before the fix, and you get the defect —
this is real output from the box this afternoon:

```
DC_RETURN/dcr00002/r1        | header 0.00 | ledger 240.00
DC_RETURN/dcr00005/r1        | header 0.00 | ledger 240.00
DC_RETURN/dcr00008/r1        | header 0.00 | ledger 240.00
DELIVERY_CHALLAN/dc00001/r1  | header 0.00 | ledger 600.00
DELIVERY_CHALLAN/dc00003/r1  | header 0.00 | ledger 600.00
```

The existing rows are all from the 16:33 batch and will stay 0.00 whatever you
change — the fix only affects vouchers generated after it. **So post one bill,
then re-run the query and look for that bill's row.**

---

## FIX 2 · The float64 leak — three sites found, do one sweep

**Money and quantity must never pass through a JavaScript `number`.** JSON
numbers on the wire are fine (money `numeric(15,2)` = 15 digits, float64 holds
~15.95, so it round-trips exactly). **Arithmetic is not fine**, and neither is
round-tripping a value through `Number()` on its way into a Decimal column.

### 2a · `stock-voucher.service.ts:3173` — writing three stored decimal columns

```ts
private toDecimalNumber(value: string | number | null | undefined): number {
  const parsed = typeof value === 'number' ? value : Number(value);   // ← float64
  return Number.isFinite(parsed) ? parsed : 0;
}
```

used at 997–1005:

```ts
data.svhTotalQty      = new Prisma.Decimal(this.toDecimalNumber(header.totalQty));
data.svhTotalValue    = new Prisma.Decimal(this.toDecimalNumber(header.totalValue));
data.svhTotalValueWot = new Prisma.Decimal(this.toDecimalNumber(header.totalValueWot));
```

The value goes through float64 **before** becoming a Decimal, so the Decimal
faithfully stores whatever float64 already lost. `svh_total_qty` is
`numeric(18,6)` — 18 significant digits, past float64's limit.

**`Prisma.Decimal` accepts a string and keeps it exactly:**

```ts
data.svhTotalValue = new Prisma.Decimal(header.totalValue ?? 0);   // no Number() in between
```

Keep `toDecimalNumber` for the *validation* comparisons (`qty < 0`, `=== 0`) —
those are fine. It is the **column writes** that must not use it.

### 2b · `sales-stock.service.ts:369` **and `:162`** — the qty rollup sums in float64

```ts
round4(lines.reduce((s, l) => s + l.qty + (l.freeQty ?? 0), 0))
```

Float64 addition across every line, then rounded. The same expression appears
twice — line 369 writes `svh_total_qty`, line 162 computes `expected` for a
reconciliation check, so the two agree only because they are wrong the same way.
Sum with `Decimal`, round
once at the end.

### 2c · `sale-order.service.ts` — the original, and the worst

`asNumber()` → float64; `roundAmount()` = `Math.round(v*100)/100`; and **line
186 `const AMOUNT_EPSILON = 0.005`**.

* **An epsilon on money is the system admitting its arithmetic drifts.**
* `Math.round(1.005 * 100) / 100` is **1.00**, not 1.01 — float64 stores 1.005
  as 1.00499999…

`temp-credit.service.ts` repeats it with `num(r.atc_bill_amount)`.

### The sweep — it is a habit, not one bad file

Three separate modules now. Rather than us finding them one at a time:

```bash
grep -rnE "Number\(|parseFloat\(|asNumber\(|toDecimalNumber\(|num\(" src \
  --include=*.ts | grep -viE "spec|page|limit|offset|index|length|slno|lineNo"
```

and check each hit against: **is this a money or quantity value?**

### The five rules

1. Keep `Prisma.Decimal` through every add, subtract, multiply, divide and
   compare.
2. **No epsilon comparisons on money.** With decimals, equality is equality.
3. Round **once**, at the column, with a stated mode (`ROUND_HALF_UP`) to the
   column's scale.
4. **Distribute by largest-remainder.** Split a charge across lines: compute
   each share, round down, then hand the remaining paise to the largest
   remainders. Rounding each line independently loses or invents money.
5. A total the client sends is a **proposal**. The server recomputes and the two
   must agree to the paise. Add totals to the §10 ignore list in
   `sales/HANDOVER_endpoints.md`.

---

## Run these

```bash
npm run typecheck
npm run lint
npm run test:e2e
```

Then **post one bill** and run the FIX 1 proof query above.

---

## Do NOT run

**`37_drop_remaining_db_logic.sql`.** It is in the share root and it executes
cleanly, which is the trap: §1 drops four rules that are enforced today and that
**nothing else enforces** — the group-nature inheritance, both cross-scope name
guards, and the voucher totals recompute. Dropping them first does not move the
rule, it deletes it, and the gap is silent: bad rows simply start saving.

Write the service-side replacement, then drop, one at a time. §3 must not run at
all — `app-setting-value.service.ts:232` depends on it today. §4 is decided:
the partition DDL helpers **stay**.

---

## After these two fixes — §11 is the work. You are NOT blocked.

**Updated 2026-09-22 evening: §12 (void + till session) is DEFERRED by the
user** — *"no need void bills problem and session issue we can handle later."*
It is parked, not cancelled, and the three questions that were open all belonged
to it. **Nothing is waiting on us. Do not build §12.**

**The live work is `sales/HANDOVER_endpoints.md` §11**, and it needs no answers
from anyone:

| §11 | what |
|---|---|
| 11.1 | `GET /sales/resolve-scan` — scan a printed ref no, get back `{ docType, docId, accYear, refno, status, sameBranch, sameYear }`. It RESOLVES; the client then calls that document's own `/get`. `sameBranch` / `sameYear` are the point: a scan landing in another branch or year must offer to switch, never return nothing silently |
| 11.2 | `GET /sales/resolve-item-scan` — item / WEIGHT / LOT barcode, one parser on the server so the prefix rules live in one place. The **increment-not-append** rule is the client's |
| 11.3 | the **GST-cancel-first gate** on `/cancel` — an IRN or live e-way bill must be cancelled BEFORE the document. Note the case that matters: **past the IRN window there is no button at all**, because the correction is then a credit note. Windows from `fn_statutory_limit`, per company — never a constant |

11.3 is the one to do first: it is a correctness gate on a route you have
already shipped, and it is the only one of the three that can lose money.

## ✅ VERIFIED — both fixes are in and correct (checked 2026-09-22, evening)

Files touched 17:08–17:22, exactly the four this file named. Checked against the
code AND the database, not the timestamps.

### FIX 1 · `svh_total_value` — DONE and proven

Rolled up at `sales-stock.service.ts:193`, after posting, from `stock_ledger`,
with `sml_is_reversal = false`. The comment beside it gets the reasoning right,
including why it could not go in the INSERT.

| era | vouchers | still wrong | correct |
|---|---|---|---|
| **POST-FIX (17:23 run)** | 33 | **0** | **33** |
| PRE-FIX (16:33 batch) | 46 | 35 | 11 |

`SALE_BILL/bil00124/r1` now carries 200.00, `DELIVERY_CHALLAN/dc00015/r1`
600.00, `DC_RETURN/dcr00020/r1` 240.00 — all 0.00 before.

**The 35 pre-fix rows are historical test data and need no backfill.** They were
generated before the fix and will stay 0.00; that is expected, not a miss.

### FIX 2 · the decimal sweep — DONE, and they are real fixes not renames

| site | before | after |
|---|---|---|
| `stock-voucher.service.ts:3196` | `new Prisma.Decimal(Number(v))` | **`toDecimalColumn`** — `new Prisma.Decimal(value)` direct, with an `isFinite` guard. No `Number()` in between |
| `sales-stock.service.ts:488` | `lines.reduce((s,l) => s + l.qty …)` in float64 | **`totalQtyOf`** — sums with `.plus()` from `new Prisma.Decimal(0)`, `decQty` rounds `ROUND_HALF_UP` at `QTY_SCALE` |
| `sale-order.service.ts` | `AMOUNT_EPSILON = 0.005`, `asNumber()` | **both gone.** The only surviving mention is a comment explaining why an epsilon was wrong |
| `temp-credit.service.ts` | `num(r.atc_bill_amount)` | gone |

### No regression — invariants re-run at double the volume

| check | result |
|---|---|
| balance = Σ ledger | **0 bad / 54 holdings** |
| lot totals | **0 bad / 54 lots** |
| vouchers balanced (summed from legs) | **0 bad / 143 vouchers** |
| negative on-hand | **0** |
| zero-cost ledger rows | **0** |

Volume roughly doubled since the morning run (54 holdings vs 28, 143 vouchers vs
63) and every invariant still holds.

**Nothing is outstanding on this file.** Next is `sales/HANDOVER_endpoints.md`
§11 / §12 — blocked on the three questions in §13.

