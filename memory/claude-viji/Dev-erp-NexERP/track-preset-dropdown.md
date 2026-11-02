---
name: track-preset-dropdown
description: "Dropdown 49 TRACKING PRESETS (stock.stock_track_preset) for the item group 'Tracked as' combo — script written 2026-09-07, registration blocked on DB write access, and itg_track_preset_id is missing from the live table"
metadata:
  node_type: memory
  type: project
---

Built 2026-09-07: `grids/stock_track_preset_dropdown.sql` in the share — the
configured-dropdown definition (`fixed.dropdown_details` id **49**, name
`TRACKING PRESETS`, Desktop) over `stock.stock_track_preset`, with the same
shared/own company merge as `stock_track_preset_grid.sql` (gridId 98). Both
param cases were run against the live DB and return the 8 seeded presets.

**Three things that are not obvious:**

- **The `claude` DB role is SELECT-only on `fixed.dropdown_details` /
  `dropdown_columns`** (`has_table_privilege` → f). The registration block at
  the bottom of the script could not be applied from here; someone with write
  access runs it, or it is typed into the admin Dropdown master screen. Same
  will be true of any future dropdown/grid registration.
- **Dropdown placeholders are BARE and i-prefixed** — `ispt_company_id`, never
  `:ispt_company_id` (a leading colon is a syntax error in the runner). The
  key passed to `NexDropdown::setFilterParam()` is that same bare name.
  Precedents: dropdown 38 EMPLOYEES (`iemp_branch_id::uuid`), 45 SALES AGENTS.
- **The live backend is AHEAD of the local checkout** at
  /home/vk/Projects/ERP/ERP-Server (272 Swagger paths vs 267; the local tree
  has no trace of either of these): `/api/v1/stock-track-presets/get` exists
  and is documented as feeding "the preset combo on Item Master and Item Group
  Master", and `SaveItemGroupDto`/`ItemGroupPayloadDto` already carry
  **`itg_track_preset_id`** — but `inventory.item_group_master` has **no such
  column** in the live DB. Until Prathap's migration lands the combo displays
  and submits but nothing persists.

**Client wiring done the same day** (builds clean): `NexCrudField` gained
`dropdownParams` + `.withFilterParam(key, value)`, applied in
`nex_crud_form.cpp`'s `ApiDropdown` case before `setDropdownId` — the crud-form
config had no way to send `dropdown_param` at all before this, which is why
only hand-built screens (sale bill, sale order, quotation) could use a
parameterised dropdown. `item_group_entry.cpp` now has a "Tracked as" field on
dropdown 49 with `ispt_company_id` from `AppSession`.

**Item master, 2026-09-07 (same session):** `item_entry.ui` gained
lblTrackPreset/ddTrackPreset at gridBasicInfo row 3 col 3/4, dropdown 49 with
the same `ispt_company_id` param, `item_track_preset_id` in collectJson and
populateForm. It is **conditionally mandatory**: on every ddItemGroup change
(and on edit load, since setSelection doesn't emit selected) the screen GETs
`/item-groups/get?itg_id=` and treats a non-empty `itg_track_preset_id` as
"the group is tracked" → field optional (an ITEM row is then just an override,
per fn_stp_effective's ITEM-beats-GROUP order); empty or a failed lookup →
required, label flips to "Tracked as *". `SaveItemCompositeDto` and
`ItemPayloadDto` already carry `item_track_preset_id` on the deployed server,
and `inventory.item_master` has no such column either. The proxy is deliberate
and imperfect: it cannot see a policy row authored outside the group screen or
a COMPANY-scope row, so it asks for a preset that is not strictly needed —
the safe direction, because a lot keyed while untracked can only be fixed by a
RELOT. **A real `stock-track-policies/effective?item_group_id=` resolver
endpoint would replace it.** Neither GET returns `*_track_preset_name`, so
edit-mode prefill shows the combo blank (same pre-existing gap as
itg_parent_name / item_group_name is fine but parent isn't).

**Still open:** the item_group mockup's *locked-by-stock* combo state is not
implemented, and nothing yet copies the chosen preset's flags into a
`stock_track_policy` row (`stp_group_id`) — the preset is a COPY-ONLY stencil,
so a group whose policy matches no preset signature must read "Custom".

See [[stock-schema-design]] for the preset/policy doctrine,
[[item-group-change-warning]] for the hazard this combo sits next to, and
[[api-endpoint-docs]] for the Swagger location.
