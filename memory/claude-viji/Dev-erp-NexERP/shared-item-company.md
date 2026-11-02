---
name: shared-item-company
description: A blank item company = shared by every company (user rule, 2026-10-01); opening-stock lookup refused it and grid 71 ignores company — notes 73; the item card now guards the company/branch pair
metadata:
  type: project
---

**User rule (2026-10-01): an item with a blank `item_company_id` is shared by every company.** It is the same rule as the ledgers in [[company-scope-nullable]].

**Server (notes 73, `share/inventory/notes-73-shared-item-company.md`, OPEN):**
- **A:** `opening-stock-lookup.service.ts` refuses a blank company ("no company set, so no company can open it"). `stock-voucher.service.ts:2557` and `price-bucket.gateway.ts:797` already treat a blank company as shared.
- **B:** grid 71 (POPUP - ITEMS) ignores `iitem_company_id` / `iitem_branch_id`, which all 8 Qt callers send. An Acme user therefore sees 10,010 CHANDRA SAW MILL items. The proposed SQL compares the tokens as text.

**Client (DONE 2026-10-01), `item_entry.cpp`:** `wireCompanyBranchPair` / `resolveBranchCompany` (via `/branch-masters/get?brId=` → brCompId / brCompName) / `validateCompanyBranch`:
- a branch picked with no company fills in that branch's company;
- a branch of another company is cleared, with a message;
- clearing or changing Company clears a branch that no longer fits;
- Save refuses a branch with a blank company.

Test: `tests/itemcard/run.sh`, 11/11, READ-ONLY on MRP ITEM. SUGAR, MRP ITEM and newlama still have a blank company + a branch on the box; that is for the user to fix, not a data fix.
