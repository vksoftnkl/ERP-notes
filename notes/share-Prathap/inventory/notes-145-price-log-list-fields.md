# Notes 145 — Price change log: before → after on every list row (2026-10-23)

notes-144 is live and verified: the writes, menu 290 and the 5 routes. The Qt
report (menu 290) is built against them. One small addition would make it
read fully.

## The gap

`GET /item-price-log/list` (and `/item-history`) rows carry old/new figures
only for the headline columns: `oldPriceA/newPriceA`, `oldMrp/newMrp`,
`oldCost/newCost` and `oldSlabPrice/newSlabPrice`. Every other changed field
comes back only as a name in `changedLabels`.

So the 23-10 change to ZT-ITEM-A's **Price B** (118 → 119) shows in the report
as just "Price B". The user has to open the detail to see the numbers, and the
CSV and the printout can't show them at all. Price B, C and D, Min, disc %,
cess and so on are exactly what the customer asks about.

## Ask

Put the same `fields[]` the `get` route already builds on each list and
item-history row, for the CHANGED columns only:

```json
"fields": [
  { "column": "ipm_sales_price_b",       "label": "Price B",            "before": 118, "after": 119 },
  { "column": "ipm_price_b_wot",         "label": "Price B (w/o tax)",  "before": 100, "after": 100.847458 },
  { "column": "ipm_price_b_markup_perc", "label": "Markup B %",         "before": 48.82, "after": 50.08 }
]
```

It's the `fields: row.changed.map(...)` mapping from `get()`
(item-price-log.service.ts ~188), applied in `list()` and `itemHistory()`. The
before/after JSON is already selected for the generated columns, so it's no
extra read. Keep sending `changedLabels` and the headline pairs as they are.

## Client

Already built to use it. `PriceLogView::changeChips()` prefers `fields[]` when
a row has it, so the chip becomes "Price B 118 → 119 ▲0.8%", with the
w/o-tax and markup companions folded into it. Without `fields[]` it falls back
to the name, as today. No client change is needed when this lands.

## Optional (from plan §3, not in the build)

- `deviceId` filter on list / days.
- `field` filter (any `ipcl_changed` member, e.g. "only Price B changes").
