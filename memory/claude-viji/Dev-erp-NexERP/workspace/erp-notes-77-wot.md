---
name: erp-notes-77-wot
description: "Notes 77 (2026-10-01): returns/counts wrote cost_rate_wot = inclusive rate; code fixed + tested, uncommitted; repair SQL plan/notes-77-wot-repair.sql handed to the user, NOT run by me"
metadata:
  node_type: memory
  type: project
  originSessionId: bc6602ee-a536-4c95-a88a-46994d9f5bfc
  modified: 2026-10-01T12:38:10.755Z
---

Notes 77: SALE_RETURN / DC_RETURN inward ledger rows carried `sml_cost_rate_wot` = the tax-inclusive rate, and PHYSICAL gains got wot = cost. Old OPENING rows have wot 0 or wot = cost. Together these corrupted `stock_item_cost.sic_avg_cost_rate_wot` (ZT-ITEM-A: 7.90 against 67.19).

Fixed 2026-10-01, uncommitted:
- `sales-stock.service.ts` `costRateWot()`: the shadow svi wot is cost ÷ (1 + line tax %), no longer a copy of the cost. Spec: `sales-stock.service.spec.ts`.
- Engine `postingCte` in `stock-voucher-posting.helper.ts`: wot is paired with the source of the cost (average / lot / line), and 0 means "derive it". A COUNT line has no tax % (the save strips it), so it derives from the item's tax as at the doc date. Covered by the e2e case "notes 77" in `test/stock-engine-ts.e2e-spec.ts`; all 15 cases pass.
- Repair: `plan/notes-77-wot-repair.sql` replays the ledger, validates on the with-tax side and skips keys it can't reproduce. It ends in ROLLBACK.

**Why:** the user asked to see the repair SQL first and dry-runs it on `erp_dry` themselves. Auto mode blocked my own dry run on ERP.

**How to apply:** don't run repair SQL against any DB, not even as a rolled-back dry run; hand it over. A peer session ran `git rm --cached` on all of `dist/` that day (3284 files staged as deleted, `dist/` now untracked). That wasn't me, so don't "fix" it without asking. Related: [[erp-a2-posting-services-state]], [[erp-one-price-table]].
