---
name: sale-bill-godown-selector
description: Sale bill header Godown (ddGodown, dd 26) built 2026-10-06 — new lines sell from it, changing it moves every line, remembered per company/branch in QSettings; tests/salebill run-full.sh godown 12/12
metadata:
  type: project
---

**User ask 2026-10-06:** "add godown selector on sale bill". Before this, every line sold from whatever `/item-price` resolved, which on the box was namakkal for everything: `query.godown_id ?? price row's ipm_godown_id ?? the branch default (br_default_godown_id)`. Since notes 93, stock held in another godown was refused at post.

**Built:**
- `ddGodown` is form row 7 in grpBillInfo, under Price Level. Placeholder "Branch default" = blank = the server resolves it.
- `fetchItemPrice` sends `godownId` = the line's own (on Keep) else the header's.
- `applySaleGodown` (the `selected` signal only): saves to QSettings `salebill/godown/<co>/<br>` and moves every line (GodownId / Name cells + a Keep re-price, MRP kept).
- `restoreSaleGodown` runs at construction and on a new bill. `applyLoadedItems` shows the first line's godown on a loaded bill.
- Per-line godown is still only the hidden-ish GodownName column (no per-line editing).
- **DC / quotation / order have no selector yet.**

**Test:** `tests/salebill/run-full.sh godown`, 12/12 (posted and cancelled bil00977 from Coimbatore via the header). Snapshot checked: the row fits, and the header height is unchanged.

Related: [[sale-bill-mrp-picker]], [[lot-cost-valuation]].
