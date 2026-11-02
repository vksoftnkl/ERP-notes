# notes (69): a group's "Tracked as" is filed under the login's company, so no item sees it (bug)

2026-09-30. Found while verifying notes 68. With this bug, "an item follows its group" has no effect for most users.

## What happens

`StockTrackPolicyService.syncFromItemGroup` files the GROUP-scope row under `requestContextService.getCompanyId()`. That value is set in `access-token.guard.ts` from `request.user.company_id`, which is **the company in the login token**, not the company the user is working in.

Reproduced live:
1. VKPOS's token carries **CHANDRA SAW MILL** (`019cc885…`).
2. Group `ZT-N68-GROUP` was saved with preset BE, and its GROUP row got `stp_company_id = 019cc885…`.
3. An item of **Acme Foods** (`019c8ea6…`) in that group resolves through `effectivePolicyLateral` with `stp_company_id IS NULL OR = 019c8ea6…`.
4. So the group row is invisible: `/stock/opening/item-lookup` answers `N`, where it should be `BE`.

`item_group_master` has no company column, and the group save DTO carries none, so the server has no way to know the working company.

## Ask (pick one)

- **(a) Recommended:** file the group's derived row with **`stp_company_id = NULL`** (shared by every company). Item groups are not company-owned, and NULL company already means "shared" throughout the schema. The resolver's `(company IS NULL)` tie-break still lets a company-specific GROUP row, authored on the policy screen, override it for one company.
- **(b)** Take an explicit `companyId` on the group save (the client would send the session company), and file the row under that. This only helps if every company re-saves the group.

## Existing data

Retire or re-file any GROUP row written under a login company. On the box today there is 1 GROUP row in total, and it is deleted.

## How to verify

1. Save a group with preset BE while logged in as a user whose token company differs from the item's company.
2. `/stock/opening/item-lookup` for an item of that group with no preset should answer `BE`.

I'll re-run the throwaway-group test when this is in.

## Verified live 2026-09-30 (after Prathap's run)

**Option (a) is in.** The throwaway group's GROUP row is filed with `stp_company_id = NULL`, and no live GROUP row carries a company.

The same test as notes 68, with a VKPOS login (token company CHANDRA SAW MILL) and an Acme Foods item:

| step | ITEM row | effective |
|---|---|---|
| no preset | none | **BE**, from the group ✓ |
| preset NONE | N | N ✓ |
| preset cleared | none | **BE** again ✓ |

"An item follows its group unless it has its own preset" now holds end to end. Notes 50 #8, 67 #8, 68 and 69 are closed.
