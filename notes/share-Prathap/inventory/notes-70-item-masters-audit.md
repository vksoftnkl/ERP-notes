# Notes 70 — item-related masters: client + backend + live audit (2026-09-30)

Scope: every master under menu 34 (Group 35, Brand 36, Section 37, Unit 38, Godown 39, Tax 40,
Category 240, Qty-wise Price 244, Tracking presets 249) plus the Item LIST (29). Each one was read on
both sides and round-tripped live in the Qt client's exact payload shape with `ZT-` rows, all
soft-deleted afterwards (there is no hard delete, so the rows remain with `*_is_deleted = true`).
Client-side defects found on the same pass are already fixed in the Qt tree; this file is only what
is yours. Ordered by what blocks a screen.

## A. Three list screens are dead or wrong at the grid layer (blockers)

| # | Grid | Screen | What is wrong | Fix |
|---|---|---|---|---|
| A1 | **45** | Unit master list (38) | Stored SQL has `WHERE unit_is_deleted = :unit_is_deleted`. The grid runner cannot parse a `:` bind, so **every run is 400** `syntax error at or near ":"`, with or without params. Menu 38 shows an empty list. | `WHERE unit_is_deleted = iunit_is_deleted` (the bare-token convention grid 55 uses; the client sends `iunit_is_deleted`) |
| A2 | **50** | Section master list (37) | Same fault: `WHERE sec_is_deleted = :sec_is_deleted` → every run 400. | `WHERE sec_is_deleted = isec_is_deleted` |
| A3 | **67** | Item master list (29) | SQL is `SELECT item_id,item_code,item_name_en,item_name_ta FROM inventory.item_master ORDER BY item_name_en` — **no WHERE at all**. `iitem_is_deleted` true/false/absent all return 10,251 rows (10,241 live + 10 deleted): "Show Only Deleted" is inert and a deleted item stays in the default list. Grid 1 has the intended predicate under a different token name. | add `WHERE item_is_deleted = iitem_is_deleted` |

## B. Hierarchy is half-built on every tree master (defects)

Group, Brand, Section, Category and Godown all carry `*_parent_id`, `*_level` and a path cache.

| # | Master | Finding (live-verified) |
|---|---|---|
| B1 | Group, Brand, Category, Godown | `*_level` is **never computed**: root and child both come back `null` (godown: `0`). Grid 55 (godowns) orders by `gdl_level`, so it cannot show the tree. |
| B2 | Section | Level IS computed (1/2/3) but only for the row being saved: re-parenting the middle node to root leaves its child at level 3. Recompute the subtree on re-parent. |
| B3 | Group, Section, Category | **A cycle is accepted** (parent.parent = child → 201); only self-parent is refused. After undoing it both rows' path caches are `{}` (own id lost). Add a descendant check on re-parent. |
| B4 | all five | **Deleting a parent with live children succeeds** (200); the child keeps `parent_id` pointing at the deleted row, `parent_name` still resolves, and the child then appears as a root-looking orphan in the parent dropdowns (17/19/20/26). Refuse (409) while active children exist. Nothing checks items referencing a group / brand / section / category either (`item_master` FKs are RESTRICT only for hard delete). |
| B5 | Godown | Any update that **omits `gdl_parent_id` sets the parent to NULL** (the DTO's `@Transform` materialises the key, so `hasOwnProperty` is true) — the child is silently detached and pulled out of every ancestor's path cache. The Qt client is safe only because it always sends the key. Same effect makes a partial update without `gdl_name` a 400. |
| B6 | Godown | `gdl_type` is not validated (`"BOGUS"` stored); only the client's combo keeps it to WAREHOUSE/ZONE/AISLE/RACK/SHELF/BIN. |

## C. DELETE is a toggle, and nothing is guarded (defects)

| # | Finding |
|---|---|
| C1 | On Group, Brand, Section, Category, Unit, Godown and Qty-price, **a second DELETE on a deleted row RESTORES it** (200 "restored successfully"). `/items/delete` stopped doing this in notes 67 and got `/items/restore`; the masters should follow: DELETE refuses (409) an already-deleted row, and restore is its own route. Until then the client labels the action Restore in the deleted view. |
| C2 | **Unit**: a base unit referenced by a pack unit (and, by the same code, a unit used by items / conversions) soft-deletes fine; the pack unit still resolves the deleted base's name on GET while dropdown 15 no longer lists it. |
| C3 | **Tax rate**: `softDelete` checks only "exists"; 21 FKs point at `inventory.tax_rate_master`. Live: a ZT- item with `item_default_tax_id` = ZT- rate → DELETE = 200, `/items/get` still returns that tax. An item / ledger / charge left pointing at a rate the picker no longer offers is a silent posting hazard. Refuse (409) when referenced by live rows. |
| C4 | **Item**: `softDeleteComposite` checks only `itemIsDeleted`; no `stock_balance` / `stock_ledger` / `sale_*_item` lookup. Decide whether an item with stock or transactions may be deleted. |
| C5 | **Godown**: no child or stock guard on delete (B4). |

## D. Contract gaps (decide)

| # | Where | Finding |
|---|---|---|
| D1 | Unit | `unit_uqc` (+ `unit_tally_name/guid/master_id/alter_id`) exist on `inventory.item_unit_master` but are in neither the DTO nor the GET, so nothing can ever set them; sending `unit_uqc` is a 400. The GSP flow reads `Unit = unit_uqc` for the e-invoice and the analysis lists it for GSTR-1 HSN summary + Tally export. Note `unit_code` is already an FK to `item_gst_units` — it IS a UQC, so the two columns duplicate one fact. Either expose `unit_uqc` in DTO + GET, or make the e-invoice map read `unit_code`. |
| D2 | Category | DTO accepts `category_tax_claim / default_tax_id / default_hsn / default_uom_id`, the service DROPS them and `toPayload` hard-codes null; the table has no such columns. Dead contract — remove from the DTO or add the columns. |
| D3 | Group | `itg_alias, itg_tax_claim, itg_default_tax_id/hsn/uom_id` are stored but **nothing reads them** (items-master service: only audit-log labels). If a new item is meant to inherit the group's defaults, that read is missing; otherwise they are dead columns. |
| D4 | Brand | `brand_alias / brand_parent_id / brand_level` are in the DTO, the table has a path cache, but no dropdown exists for a parent brand. Say whether brands are hierarchical; if not, the columns are dead. |
| D5 | Section | `sec_position` is stored and indexed (`idx_sec_position`) but the grid orders by name and the client never sends it. |
| D6 | Section | **No unique index on `sec_name`**: a duplicate name is 201 (the other three masters give 409). |
| D7 | Qty-wise price | **No uniqueness on the slab key** (item, unit, company, branch, party, level, from, to): a re-POST creates a duplicate slab and the service's "Duplicate qty price slab" handler can never fire. |
| D8 | Qty-wise price | The screen's Mode combo is P / Q (Rs) / F; the DTO accepts `^[PRF]$`. The client now maps Q → R on the wire (and shows R as "Q (Rs)"). Nothing to do unless you would rather accept Q. |
| D9 | Tax rate | Swagger says "on update only the keys present are written", but `tax_name` is a required `@TrimmedString`: `{tax_id, tax_sort_order}` is a 400. Harmless for the client (full form always sent). |
| D10 | Dropdowns 17 / 19 / 20 / 26 (parent pickers) | Offer every live row including the one being edited and its descendants; 26 (godown parent) also lists every branch's godowns although the server refuses a cross-branch parent. The client now refuses self-parent before posting; a `dropdown_param` for the branch on 26 and a self-exclusion would close the rest. |
| D11 | ui_table 19 (Qty-wise price grid) | Registered with 23 columns: `#` at 0, `item_id` at 21, `item_unit_id` at 22. The screen's entity has 22: `iqp_id` (hidden) at 0, ids 17-20, `item_unit_id` at 21. Result: the uuid shows under a `#` header and column 22 is dropped with a warning. Wanted layout: col 0 = `Id` hidden, 17-20 hidden, 21 = `item_unit_id` hidden, col 22 removed. DONE 2026-09-30 from the client side (re-registered to 22 columns, old col 22 soft-deleted); nothing for you here. |
| D12 | Menu 249 | `fixed.menu_master` names it "Stock Track Policy"; the screen is the tracking-preset list ("Tracking Presets"). Cosmetic. |

## E. Verified clean (no action)

- Group / Brand / Section / Category / Unit / Godown / Tax / Qty-price: create, get, update-merge (omitted keys untouched, photo kept), delete, restore all 200/201 in the client's payload shape; duplicate name 409 (except D6); unknown key 400.
- Group preset → GROUP-scope policy (notes 68/69): re-verified, `stp_company_id` NULL, clearing the preset retires the row.
- Tax rate: `/resolve` returns the 17 roles with OVERRIDE / DEFAULT / UNMAPPED; supersede cycle is refused; generated cgst/sgst/igst correct.
- Tracking presets: 8 seeded rows load; GET-only is fine (a preset is a stencil copied at save time).
- Qty-price: bulk create, partial update, delete one, restore, and `""` ids stored as NULL.

## F. Snapshot note

Resolved 2026-09-30: the read-only copy was re-synced after notes 69 and now matches the box (shared GROUP policy slot). Nothing to do.
