# Textile tags: what the Qt screen sends and reads (2026-10-17)

The Qt tag screen is built against the plan (`../plan-nestjs-textile-tags.md` §4) before the
routes exist. Where the plan leaves the JSON open, these are the choices the client made. If your
routes differ, tell me and I change the client.

Code: `src/modules/inventory/stock/textile/` in the Qt repo. Test: `tests/textiletag/run.sh`
(37/37, with reserve and suggest-price answers faked).

## Registered on the box

- **ui_table 53** "TEXTILE TAGS - NEW TAG LINES": the New tags grid, 29 columns. Script:
  `textile_new_tags_ui_table.py` in this folder.
- **Menu: not yet.** The screen class is `TextileTagEntry`. Please give it a menu id with B-1;
  it goes in `form_inventory_menu.h`.

## `POST /textile-tags/reserve`

Exactly as plan §4. The client leaves out any optional key it has no value for, and never sends
`""`. `lineNo` is the grid line number (1-based). A refusal's `errors[].field` should be
`lines.<index>` (the index into the `lines` array that was sent). The screen tints that line.

Read from `data.tags[]`: `lineNo`, `tagId`, `barcode`, `lotId`, `labelQty`.

## `POST /textile-tags/lists/{size|colour|fabric|season}/save`

Body: `{companyId, name, sizeSet}`. `sizeSet` is sent for size only. When the client sends it,
the value is one of `ALPHA`, `WAIST`, `KIDS` or `FREE`.

The client reads the id from `data.<prefix>Id` (`iszId`, `iclId`, `ifbId`, `isnId`), or from
`data.id`. **On a duplicate, return 409 with the existing row in `data`**: the screen takes the
existing row's id as the answer. Until the list dropdowns exist, every typed name goes through
this route, so a known name always comes back as 409.

## `GET /textile-tags/suggest-price`

Query and answer as plan §4. The client reads `data.rule.pct`, `data.rule.round`,
`data.salePrice` and `data.mrp`. A 404 means "no rule": the line is then priced by hand.

## Dropdowns the screen will use (ids from you, B-2)

Size, colour, fabric and season. The client reads the columns
`<prefix>_id`, `<prefix>_name`, `isz_size_set`, `isn_from_date` and `isn_to_date`, and passes
`icompany_id`. The season dropdown lists open seasons only.
