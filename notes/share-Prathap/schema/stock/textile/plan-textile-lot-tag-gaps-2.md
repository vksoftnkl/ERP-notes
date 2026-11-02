# Textile lot tag — more tag fields (addendum 2)

Written 2026-10-17. **This is a NEW file. It edits no existing plan.** It adds to
`plan-textile-lot-tag.md` ("the base plan") and `plan-textile-lot-tag-gaps.md` ("gaps 1",
G1–G19, all accepted 2026-10-17). Decisions continue as **G20–G23**. The user asked for G20–G22 on
2026-10-17 ("yes"); G23 is the answer to "do we need male/female?".

Mockup: `textile_tag_ui_mockup_rev3.png` (+ `.py`) in this folder. REV 2 is kept.

---

## 0. The rule for a tag field

A field goes on the tag **only when it changes between purchase lines of the same generic item**.
Anything the item already decides stays on the item.

| on the item (once) | on the tag (per purchase line) |
|---|---|
| gender / department, category, HSN, unit, tax, tracking preset | brand, size, colour, model, MRP, price, supplier, **fabric, season, supplier's article no.** |

Checked live 2026-10-17: `inventory.item_master` already has `item_group_id`, `item_category_id`
and `item_section_id`. All three masters are trees (`*_parent_id`, `*_level`, `*_sort`).

## 1. G20 — Fabric / material

One generic "Shirt" comes in cotton, linen and poly-cotton; customers ask by fabric.

- New `inventory.item_fabric_master` (prefix `ifb_`): `ifb_id`, `ifb_company_id` (NULL = shared),
  `ifb_code`, `ifb_name`, `ifb_sort`, `ifb_is_active`, audit + sync columns.
  `UNIQUE (coalesce(ifb_company_id, nil-uuid), upper(ifb_name)) WHERE NOT ifb_is_deleted`.
- Tag column `ilt_fabric_id uuid NULL` → FK. Optional.
- Quick-add exactly as size and colour (gaps 1 G5).
- Shown on the label after colour ("6.2 m · Maroon · Pure silk") and in the sale-bill details.
- **Default:** as above.

## 2. G21 — Season / collection

End-of-season markdowns and sell-through by season.

- New `inventory.item_season_master` (prefix `isn_`): `isn_id`, `isn_company_id` (NULL = shared),
  `isn_code`, `isn_name` ("Diwali 2026", "Summer 2027", "Wedding 2026"), `isn_from_date`,
  `isn_to_date` (both NULL allowed), `isn_is_closed` (a closed season is not offered on new
  tags but stays filterable), `isn_sort`, audit + sync.
- Tag column `ilt_season_id uuid NULL` → FK. Optional.
- **Entry:** season is a **header field** on New tags (and on the purchase later); every line
  takes it. A line can change its own (the column is hidden by default, shown by the grid's
  column chooser). One season per inward is the normal case.
- Default season on the header = the open season whose date range covers today, else blank.
- **Default:** as above.

## 3. G22 — Supplier's article no.

The supplier's own design code (on their invoice, their catalogue). Needed to reorder, to return
goods, and to match the purchase bill. `ilt_model` is OUR name for the design and is not always
the same.

- Tag column `ilt_supplier_article varchar(60) NULL`. Free text (it is the supplier's code).
- Index `(ilt_company_id, ilt_supplier_id, upper(ilt_supplier_article))`, not unique (a supplier
  reuses a code across seasons).
- Searchable in the Register search box; shown on the Register and in the purchase-return pick
  list (gaps 1 G6). **Not printed** on the customer label by default (a label-design choice).
- **Default:** as above.

## 4. G23 — Gender / department: on the item, not the tag

"Men's Shirt" and "Boys Shirt" are different generic items, each created once. Gender goes on the
item through **item section** (a tree, e.g. Men › Shirts, Women › Sarees, Kids › Boys), which
already exists. **No tag column.** Reports by gender read the item's section.

- Setup note for the go-live checklist (beside gaps 1 G18 HSN): build the section tree before
  creating the textile items.
- **Default:** no tag column; section tree at setup.

## 5. Left in `ilt_attr_note` (free text)

Fit, sleeve, pattern, occasion, "blouse included", length notes. Printed on the label when set;
not filtered or reported, so a list would only slow entry.

## 6. Effect on the routes and reports

| piece | change |
|---|---|
| `POST /textile-tags/reserve` line | adds `fabricId?`, `seasonId?`, `supplierArticle?` |
| `POST /textile-tags/update` | the same three are editable (details, any status but VOID) |
| `GET /textile-tags/scan` | returns fabric name, season name, supplier article |
| label dataset `TEXTILE_TAG` | adds `fabric`, `season`, `supplier_article` (design decides what prints) |
| Register grid | Fabric and Season columns + filters; search also matches supplier article |
| reports (gaps 1 G13) | sell-through **by season**; stock and sales **by fabric**; by gender through the item section |
| offline sync | both new list tables sync down to every till (scan needs the names) |

## 7. Build order

All three columns and both list tables go in **T-1** with the rest of the table (nothing is built
yet). Screen work in T-2; report work in T-4.

## 8. Decisions

| # | question | default |
|---|---|---|
| G20 | fabric from a list, optional, with quick-add | yes |
| G21 | season list; header field on New tags; open season by date as default | yes |
| G22 | supplier's article no., free text, searchable, not on the customer label | yes |
| G23 | gender on the item (section tree), not on the tag | yes |
