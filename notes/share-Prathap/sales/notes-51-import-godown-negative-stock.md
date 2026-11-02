# notes (51): godown + negative-stock on imported bill lines

2026-09-26. Found while fixing the sale bill's quotation import (client side is
done, see below). Two server asks, both small.

## What the client does now

| Path | Godown | Allow-negative |
|---|---|---|
| Item pick | `/master-lookups/item-price` `godown_id` | `allow_negative_stock` |
| **Quotation import** (fixed today) | `sqiGodownId` / `sqiGodownName` from `/quotations/get` | `sqiAllowNegativeStock`; then a stock-only `item-price` call **with `godown_id` = that godown** refreshes live stock / stock row / loyalty / reorder, quoted figures untouched |
| **Bill load** (fixed today) | `sbiGodownId` | `sbiAllowNegativeStock` (null → Y) |
| Order import | `soiGodownId` | **hardcoded `Y`** — the negative-stock check is skipped for every order line (ask 1) |
| DC / multi-doc picker | godown id only | hardcoded `Y` (right for a DC — the DC already moved the stock; wrong for an order line picked through the same dialog) |

Before today a bill imported from a quotation sent `sbiGodownId: ""` → 400
`sbiGodownId is required for a new bill line`, and every line was treated as
NOT allowing negative stock.

## Ask 1: `soiAllowNegativeStock` on `/sale-orders/get`

The same derived, GET-only field the quotation and bill already serve
(`quotation.service.ts:1779`, `bill.service.ts:1910`): service item, else
blocked only when the line's godown, the company (`comp_negstk_apl`) and the
item all say no. The order line names its own godown, so use that one (as the
bill does), not the branch default. Also on the DC-picker rows
(`bill-read.service.ts` ~311, the `soi_` branch) if cheap. The client will
read it on the order import and drop the hardcoded `Y`.

## Ask 2: which godown is "the default"?

Two routes disagree:

- `/quotations/get` stamps every line with **the branch's `br_default_godown_id`**
  (`resolveDefaultGodown`).
- `/master-lookups/item-price` with no `godown_id` uses **the price row's
  `ipm_godown_id`** (`item-price.lookup.ts:303`).

On Acme / branch `019c8ea7-…` for item of quotation Q0016 they differ
(`019c9935-…` "namakkal" vs `019e9c9d-…`). So a hand-picked line and the same
item imported from a quotation land in different godowns. Please pick ONE rule
for "no godown chosen yet" and use it in both. The client now always passes
the quotation's godown to the lookup, so it is consistent either way, but the
two entry routes should agree.

---

## Checked 2026-09-26 ~12:00: both landed, thank you

- Ask 1: `/sale-orders/get` serves `soiAllowNegativeStock`. The order import reads
  it (the hardcoded `Y` is gone) and runs the same stock-only refresh as the
  quotation.
- Ask 2: `/quotations/get` now stamps the **price row's** godown, the same rule as
  `item-price`. The client follows whatever the server stamps.
- Client checks: `tests/salebill/run-full.sh orderimport` and `quoteimport` both pass.
