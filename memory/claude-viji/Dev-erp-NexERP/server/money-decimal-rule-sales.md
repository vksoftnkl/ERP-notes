---
name: money-decimal-rule-sales
description: "Money and quantity must stay Prisma.Decimal into numeric columns — no epsilons; sale-order/temp-credit/stock-voucher are converted, ~667 float sites remain across src"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: e9c9b22e-6446-4278-a2b8-28a014a35bbc
  modified: 2026-09-22T11:53:44.763Z
---

Rule from the 2026-09-22 afternoon handover (§3a / "the five rules"): keep `Prisma.Decimal` through every add/subtract/multiply/divide/compare on money and quantity; no epsilon comparisons on money; round ONCE at the column with an explicit mode; distribute by largest-remainder; the client's totals are a proposal the server recomputes.

**Why:** `new Prisma.Decimal(Number(v))` puts the value through float64 *before* it becomes a Decimal, so the Decimal faithfully stores what float64 already lost — `svh_total_qty` is numeric(18,6), past float64's ~15.95 digits. And `Math.round(1.005*100)/100` is 1.00, not 1.01.

**How to apply:** converted so far — `sale-order.service.ts` (AMOUNT_EPSILON/QTY_EPSILON deleted; `dec`/`roundQty`/`roundAmount` return Decimal), `temp-credit.service.ts`, `stock-voucher.service.ts` (`toDecimalColumn` for column writes, `toDecimalNumber` kept for comparisons only), `sales-stock.service.ts` (`decQty`/`totalQtyOf`). JSON payloads keep plain numbers — convert with `.toNumber()` at the wire boundary only.

A unit spec asserting a write now needs `new Prisma.Decimal(n)`, not `n` — that is the signal the value reached the column correctly, not a regression.

**Still float64: ~667 money/quantity sites across `src`, ~316 in `src/modules/sales`.** The blocker is structural: `bill-snapshot.ts` normalises every figure to `number`, and `SalesLeg.amount` / `RegisterDoc.*` are typed `number`, so the posting layer cannot go Decimal one file at a time. See [[sales-posting-chain-findings-2026-09-22]].
