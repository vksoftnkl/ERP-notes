---
name: stock-negative-average-wot
description: A 0-wot legacy receipt drains an item's ex-tax pool below zero; the posting engine treats a ≤0 average/lot wot as "not stated" since 2026-10-15; CHECKs fire before ON CONFLICT
metadata:
  type: project
---

2026-10-15: every sale of ZT-ITEM-B 500'd with `ck_slt_prices` on `stock.stock_lot`.
Its only inward, opening OPN0067 (09-24, before notes 77), carries cost 40 but
`cost_rate_wot` 0, so the item's ex-tax pool started at 0 and later outwards
relieved it negative (Σ −10.29 / 401 = −0.025661 on sic/sbl). `resolveLots` INSERTs
a lot row for every plain ('N') outward line and relies on ON CONFLICT DO NOTHING —
but **Postgres evaluates CHECK constraints before the ON CONFLICT arbiter**, so a
negative rate on a row that would only have conflicted still aborts the post.

Fix in `stock-voucher-posting.helper.ts` (priced/lotted CTEs): a negative average or
lot wot is "not stated" like 0 (`NULLIF(GREATEST(x, 0), 0)`), so `costed` derives
cost ÷ (1 + tax). Only ZT-ITEM-B was negative; its sic/sbl ex-tax figures stay
negative and drift further with each sale until the legacy opening is repaired
(not done — repairing OPN0067's wot alone gives an avg_wot ~47 > the 40 inclusive,
because ~85 units were relieved at 0 wot).

**How to apply:** a negative `*_avg_cost_rate_wot` is legacy pool damage, not an
engine bug in the rebuild; a ledger/voucher-item CHECK on wot does not exist, only
stock_lot's. See [[lot-actual-costing-notes-92]], [[stock-engine-ddl-lives-outside-repo]].
