---
name: item-masters-full-test
description: 2026-10-22 full backend+frontend live test of all 9 item masters (tests/itemmasters/); Qt fixes applied; server findings = notes-139
metadata:
  type: project
---

User asked 2026-10-22 "complete code test for all item masters (tax/godown etc) backend and front end, with test data". Run by 4 parallel forks + me (item card suites).

- **Suite:** `tests/itemmasters/` — `run.sh` (SRC=<screen test>), `api.py` helper (login, get/post/…, db(), check/summary), API tests `api_{unit,track_preset,qty_price,brand,godown,taxrate}.py` + `api_hierarchy.py [group|section|category]`, screen tests `test_{unit,track_preset,qty_price,brand,godown,taxrate,hierarchy}_screen.cpp`. All data `ZT-…`, deleted after (0 left verified).
- **After Qt fixes:** all screen suites pass except brand "cleared Short Name stored as ''" (server B7). API suites fail ONLY on server findings (notes-139).
- **Qt fixes made:** ApiClient::handleError(error, parent, refusedTitle) → lists say "Cannot delete"/"Cannot restore" (8 lists); unit numbers `.blankAs(QJsonValue())` (weight/conversion/loading/unloading/attach); qty price unused mode cells → null + Barcode lookup implemented (ItemByBarcode → itemId/itemName); parent dropdowns `iexclude_id` in edit (group 17/section 19/category 20); godown dropdown 26 `ibranch_id` follows the branch + `iexclude_id` (scopeParentList, fillFromJson override); tax rate self-supersede extraValidate (dropdown 53 has no exclude param).
- **Not done (feature gaps, ask user):** group Alias field; group/category default Tax/HSN/UOM/tax-claim fields (server's inheritClassDefaults fills new items from them — unusable from Qt). NexCrudForm Text fields don't support blankAs (cleared text = "").
- **Server:** Shared/Prathap/inventory/notes-139-item-masters-test.md (A1 blank names, A2 deleted names block reuse, A3 500s, A4 qty-price rules, B1-B9, C1-C3). Not answered yet.

**2026-10-22: notes-139 LIVE (Prathap ran it).** `tests/itemmasters/verify_139.py` gives 28/28 for A1–A4 and B1–B9 (B3 refuses with 409 GODOWN_BRANCH_CHANGE_HAS_CHILDREN). C1 (a duplicate godown code is still accepted) and C2 (ZERO_RATED at 5 % is still accepted) remain open questions. The api_brand, api_godown and api_taxrate expectations were updated to the fixed behaviour, and all suites pass.
