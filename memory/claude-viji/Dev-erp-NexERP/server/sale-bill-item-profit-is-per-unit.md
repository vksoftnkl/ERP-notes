---
name: sale-bill-item-profit-is-per-unit
description: sbi_item_profit / sbi_profit_pre_tax are PER UNIT despite their amount-scale numeric(15,2), and are client-supplied and nullable
metadata:
  type: reference
---

`sales.sale_bill_item.sbi_item_profit` and `sbi_profit_pre_tax` are **per unit**,
not line totals — a bill's margin is `Σ (profit × sbi_net_qty)`.

Easy to get wrong: both are `numeric(15,2)` like `sbi_net_amt` beside them
(only `sbi_sold_price` carries the `(15,4)` rate scale), and the schema's
`// Costing / margin, PER UNIT` comment sits above the whole group. The data
settles it — two lines of the same item at the same rate, quantities 3 and 10,
both carry `sbi_item_profit = 25.23`.

Two more traps:

- **The server never computes either.** They arrive on `/bills/create` from the
  billing screen and are stored as sent.
- **Both are nullable per line**, so a bill can hold costed and uncosted lines
  together. `open-items` therefore answers `billProfit` = null unless *every*
  live line carries the figure: a partial sum understates the margin, and the
  decision it informs is whether a settlement discount can be afforded.

`sbi_item_profit` is tax-inclusive; `sbi_profit_pre_tax` is pre-tax. A
settlement discount comes off the gross, so the first is the one it eats into.
