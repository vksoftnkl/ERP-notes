---
name: erp-widget-masters-config
description: Where master-form widget config (re-label/reorder/visibility) lives and how to build the form-field bridge map
metadata: 
  node_type: memory
  type: reference
  originSessionId: 56f053fd-5d5c-443b-81cd-9991d057273a
  modified: 2026-08-25T12:59:13.290Z
---

Master pages re-label / re-order / show-hide their hardcoded form fields from a
backend "widget-masters" config, plumbed via `features/masters/shared/widget-config.ts`
+ `widget-visibility-tree.tsx`. Endpoints: `/widget-masters/get` (live form config),
`/widget-masters/config` (right-click tree), `/widget-masters/visibility` (PATCH save).

Config source of truth is Postgres: `fixed.form_section` (section_menu_id, section_platform,
section_name/gui_name, section_position/visibility) and `fixed.form_field` (field_section_id,
field_name, field_gui_name, field_secondary_text, field_position, field_visibility).
Sections are scoped by `section_menu_id` + `section_platform` (usually "Web", enum
Mobile|Desktop|Web, case-sensitive). To wire a master page, query the menu's section +
fields, then build `WIDGET_FIELD_NAME_BY_FORM_FIELD` mapping each form field `name` →
the backend `field_name` (matched case-insensitively).

`applyWidgetFieldConfig` takes an optional 4th arg `{ mode }`. Default ("reorder") re-orders
bridged fields by config order + overrides labels — so a NON-EMPTY server config for a menu
OVERRIDES the page's hardcoded tab/subheading/order/label structure at runtime (verified: it
injected a spurious "Section 1" tab and scattered new subheadings on the customer form). Pass
`{ mode: "visibility-only" }` to make the PAGE authoritative: keeps code order/labels/grouping
exactly, config only drops fields whose visibility=false. The CUSTOMER page (menu 10) uses
visibility-only as of 2026-07-16 (3-tab redesign Identity/Notes/Regional Details authored in
buildCustomerFormFields); its Visible Settings popup still show/hides, but Secondary Text
re-label is now inert there. Also added form fields cusContactPerson/cusCountry/cusCreditAllowed
(the checkbox is OR'd with the derived value in buildRequestPayload).

`fieldSecondaryText` is the user's custom re-label: when non-empty it WINS over
`fieldGuiName` as the rendered form label (applyWidgetFieldConfig reorder mode, since 2026-07-08),
and the customer page mirrors popup Secondary Text edits into widgetFieldConfig live. Bridge-map values
must match `fixed.form_field.field_name` EXACTLY (incl. DB typos like `cus_birt_date`) — a stale
name silently renders the input as trailing and shows "not on form" in the popup; verify with a
SQL diff against the map (9 stale customer keys fixed 2026-07-08: comp→comp_name,
cus_credit_amt_limit→cus_credit_amount_limit, cus_distance_km→cus_distance,
cus_disc_perc→cus_discount, cus_birth_date→cus_birt_date, cus_marriage_date→cus_anniversary_date,
cus_tcs_applicable→cus_tcs_allowable, cus_itcoll_exempted→cus_it_collection_exempted,
cus_is_active→cus_active).

NOT only master modals: menu 14 QUOTATION ENTRY wires the same config into a full-page
transaction screen (2026-08-08). Web section 61 "Quoation entry" (gui name; section_name
"Quoation" — both typos are the DB's), 21 fields whose field_name IS the label with spaces
("Existing Customer", "POS State Code", "Contact No"…), covering exactly the three header
blocks. Plumbing differs from the masters and is worth copying for other transaction screens:
config comes through RTK Query (quotationApi getQuotationWidgetConfig / saveQuotationWidgetVisibility,
"WidgetConfig" tag) not useApi; the header blocks are hand-laid JSX, so instead of
applyWidgetFieldConfig they take a `fields: HeaderFieldConfig` prop and gate each control on
`fields.isVisible(key)` / `fields.labelFor(key)` (visibility + label only — reorder is
meaningless across three designed columns); the bridge map QUOTATION_HEADER_FIELD_NAMES doubles
as the fallback label, since the shipped config names every field after the label the screen
already had. Right-click opens it, scoped to `.headerRow` — both grids already own right-click
for their ui_table column settings, so the two never collide. useVisibleSettingsContextMenu does
NOT work here (it requires an open [role=dialog]). Section visibility gates its fields on this
screen (the one section is the whole header; a toggle that changed nothing would read as broken).
GOTCHA: `.comboMenu` is `--erp-z-popup` 1200 and the modal overlay `--erp-z-modal` 1000, so
right-clicking a combobox focuses it (Chromium focuses on right-click), its menu opens and
floats OVER the dialog — the fix is `document.activeElement?.blur()` in the contextmenu handler.
Lives in features/sales/quotation/components/visible-settings.tsx.

MENU 14 SECOND SECTION — TERMS (wired 2026-08-22): the quotation's Terms panel is now
configured too, from Web section 65 "Quotation-terms"/gui "Terms" (created by the user in the
widget-master admin UI, so field_name IS the label): Remarks / Payment Terms / Delivery Terms /
Other Terms. Client side: QUOTATION_TERMS_FIELD_NAMES in quotation.constants.ts (keys are
`QuotationTerms`' own), a `terms: TermsFieldConfig` on the useVisibleSettings return, and
TermsBlock gates each row on it and returns null once `anyVisible` is false — so unchecking the
section hides the whole panel (WidgetVisibilityTree's section toggle already cascades to every
field, and `resolved` ANDs section visibility in). `fields` is optional there because the sale
order reuses TermsBlock without any config. Right-click on the Terms fieldset opens the same
dialog (GroupBox gained an onContextMenu prop). GOTCHA fixed on the way: field 503 was created
named "Other Terms" (a duplicate of 506) where the panel's first row is Remarks — the config map
is keyed by lowercased field_name, so the two collapsed into one entry; renamed 503 to Remarks in
fixed.form_field. SEED DIVERGENCE like menu 22: section 65 exists only in this DB — ERP server
prisma/seed/Form_Section.sql has just section 61 for menu 14 and its WHERE NOT EXISTS guard skips
menus that already have any section, so a fresh DB needs the export regenerated
(`npm run seed:export:ui-config`) or the Terms panel is simply uncontrollable (fails open, all
four rows visible).

MENU 22 SUPPLIER (wired 2026-08-22, features/masters/purchase/suppliers): Web sections 62
"suppliers-Identity"/Identity, 63 "supplier-notes"/Notes, 64 "suppliers-regionaldetails"/Regional
Details — created by the user in the widget-master admin UI, so field_name IS the label
("GST_No", "Supplier_name", "Cash Disc %"), 38 fields matching the form 1:1. COLLISION GOTCHA:
Identity and Regional Details both had Address 1/2/3, City, District, State, Country, and the
config map is keyed by lowercased field_name alone — the regional rows silently overwrote the
identity ones and the two sets would have hidden together. Fixed by renaming field_name (NOT
field_gui_name, which the popup displays) on section 64 to "Regional Address 1" etc.; a fresh DB
needs the same rename. Plumbing lives in suppliers/widget-config.ts (bridge map +
buildSupplierWidgetFieldConfig, which folds section visibility into its fields, +
applySupplierWidgetConfig); page.tsx copies the customer wiring verbatim (right-click hook,
tree modal, PATCH, Escape/F5 capture). Two deliberate differences from customer: the popup's
Secondary Text is re-applied as the label after the visibility-only pass (so re-label works here),
and pruneEmptyGroups drops a heading/subheading left with no field under it — needed because
headings are TABS on this screen, so hiding a whole section would otherwise leave an empty tab
(a "custom" field counts as content, keeping Bank Details). SEED DIVERGENCE: ERP server
prisma/seed/Form_Section.sql + Form_Field.sql provision a DIFFERENT legacy menu-22 config
(sections 37-41, sup_* names, missing fields) that this bridge map would NOT match; the seed's
WHERE NOT EXISTS guard skipped it on this DB. A fresh DB needs a seed matching 62-64 or the
wiring is a silent no-op.

Known menus: 10 customer — REBUILT 2026-08-25, the old comp_*/cus_* config (sections 24/25/26/
31/34…, 66 fields, INSERTed 2026-07-07) is GONE. The user re-authored it in the widget-master
admin UI as Web sections 66 "Customer-identify"/Identify (47), 67 "customer-notes"/Notes (8),
68 "Customers-Region details"/Region Details (8) — so field_name IS the label ("Customer Name",
"GST No", "Distance (km)"), matching the form's 63 inputs 1:1. Symptom before rewiring: the
Visible Settings popup showed every row "not on form" and no checkbox changed anything (a stale
bridge map fails open, silently). Same COLLISION as menu 22 — section 68 repeats Address 1/2/3,
City, District, State, Country from 66, and the config map is keyed by field_name alone — fixed
by renaming section 68's field_name to "Regional Address 1"/"Regional City"/… (field_gui_name,
what the popup shows, left alone). LIVE (169-58-213-171.sslip.io, section gui "Regional Details")
never got that rename, so on 2026-10-01 Address 1→Country on the Regional tab would not hide
(the toggle hid the Identity input instead). FIXED CLIENT-SIDE: shared
qualifyRegionalSectionFieldNames prefixes "Regional " on read for any section whose
name/gui name matches /region/i; customer + supplier run BOTH /widget-masters/get and the
popup tree (/config) through it (normalizeCustomerWidgetSections / normalizeSupplierWidgetSections),
so the DB rename is no longer needed. PATCH goes by fieldId, so the in-memory rename is safe. Plumbing was pulled out of
page.tsx into features/masters/sales/customer/widget-config.ts, copying suppliers: section
visibility folded into fields + pruneEmptyGroups (headings ARE the tabs, so hiding a section must
drop its tab) + Secondary Text re-label after the visibility-only pass. Verified headlessly:
field off → input gone, section off → tab gone, save → PATCH + refetch, reload → still hidden.
The two supplier-only helpers now live in shared/widget-config.ts
(buildWidgetFieldConfigFromSections `{foldSectionVisibility:true}`, pruneEmptyGroups).
SEED DIVERGENCE (unfixed): ERP server prisma/seed/Form_Section.sql + Form_Field.sql still carry
the LEGACY menu-10 sections, so a fresh DB re-provisions comp_*/cus_* and the wiring no-ops —
regenerate with `npm run seed:export:ui-config -- --only=Form_Section.sql,Form_Field.sql`),
29 item MASTER (features/masters/inventory/item, page.tsx→item-master-page.tsx) —
REBUILT like menu 10/22, REBRIDGED 2026-08-29. The seeded sections 55-58 with snake_case
field_names (item_name, item_allow_sales…, from ERP server
prisma/seed/Item_Master_Widget_Config_Menu29.sql) are GONE; the config is now Web sections
69 "Item Master-Core"/Core Details (19 fields, absorbing the old Reference Links AND
sort_order) + 70 "Item Master-Ean Table"/Ean Table (the 27 Rules & Status checkboxes/flags
+ expiry days) + 71 "Item Master-Inventory&Notes" (storage/image/photo/notes), 50 fields
matching the form 1:1, each field_name IS its label ("Item Name", "Hsn Code", "Allow
Negative So", "Service Item"). SYMPTOM of the stale map (reported as "visibility settings
widget cannot access the input fields"): all 50 rows badged "not on form", 0 of 49 bridge
values matched, no toggle reached the form, no re-label/reorder — a stale bridge map fails
open, silently. item_sku now HAS a config row ("Sku"). Same SEED DIVERGENCE as menus 10/22:
the seed still provisions 55-58, so a fresh DB needs the export regenerated or the wiring
no-ops again. GRID-POSITIONED CAVEAT:
the 27 rules checkboxes use explicit gridColumnStart/gridRowStart, so config re-label +
hide work but REORDER is visually inert for them (they stay in their grid cell); inventory
fields flow normally so order applies there. LESSON (2026-07-15): the user had already
created sections 57/58 via the widget-master ADMIN UI with label-derived field_names that
don't bind (item_Retail Item w/ space, item_allow_negative_stock vs form item_allow_neg_stock,
item_batch_based vs item_is_batch_based, item_service_item vs item_is_service, item_expiry_item
vs item_is_expiry_item, item_weigh-scale vs item_weigh_scale, item_image_URL). ALWAYS re-query
fixed.form_section for the menu RIGHT BEFORE inserting — I inserted duplicate sections off a
stale earlier read; had to delete them and normalize the admin names to real binding keys. bridge
map + apply live
in item-master-page.constants.ts WIDGET_FIELD_NAME_BY_FORM_FIELD and .tsx
applyItemWidgetFieldConfig. NOTE: the item form is too complex for the shared
applyWidgetFieldConfig (which trails every unconfigured field to the end) — it uses its
OWN section-aware in-place apply (buildItemFormSections+reorderConfiguredItems) that only
reorders configured fields within their form section and leaves flags/linked-tables/notes
put. Non-obvious bridge values (the rest are the label of the form field): item_name_ta→
"Item Name (Local)", item_hsn_code→"Hsn Code", item_default_tax_id→"Default Tax",
item_supplier_id→"Default Supplier", item_cust_group→"Item Customer Group",
item_allow_neg_stock→"Allow Negative Stock", item_is_batch_based→"Batch Based",
item_is_service→"Service Item", item_is_expiry_item→"Expiry Item".
Wired 2026-07-15, REPLACING a silent no-op: the old code parsed /widget-masters/get with a
legacy widgetGroupId/widgetName shape (toItemWidgetConfigRecords, ITEM_MASTER_WIDGET_GROUP_ID
=5) that the section/field response never contains, so nothing ever applied. The right-click
"Visible Settings" tree editor (WidgetVisibilityTree + /widget-masters/config + PATCH
/widget-masters/visibility) was ALSO added to the item modal 2026-07-15, same as unit/godown.
Item-specific note: the linked-record tables (price/EAN/reorder) own their OWN right-click
column-settings menu (handleBodyRowContextMenu preventDefault+stopPropagation when column
layout enabled), so right-click on a table row = column menu, right-click on a form field =
Visible Settings — the two coexist, no conflict),
37 item-section (sec_*), 38 unit (item_unit_*), 39 godown (section 17,
fields gdl_name/gdl_short/gdl_branch/gdl_parent/gdl_delivery/gdl_split/gdl_negative_stock/
gdl_volume/gdl_active/gdl_remarks), 54 account-ledger-groups (section 19, fields
group_name/group_short/group_parent/group_sort/group_description), 70 employee-department
(section 21, fields department_name/department_code/department_remarks/department_status),
71 employee-designation (section 22, fields designation_name/designation_code/
designation_remarks/designation_default/designation_status), 239 device-list-master
(section 23, fields device_name/device_type/device_platform/device_user/device_company/
device_branch/device_mac_address/device_blocked/device_block_reason/device_status —
note device_status is positioned before device_block_reason), 241 gsp-company-service
(section 20 "gsp company service", page features/masters/settings/gsp-company-service/page.tsx,
fields csgCompanyId→gsp_company / csgGspProviderId→gsp_provide / csgServiceType→gsp_service_type
/ csgEuserName→gsp_service_euser_name / csgEuserPassword→gsp_user_password; csgIsActive trails).
DB: postgres://postgres@localhost:5432/ERP.

Before wiring, confirm BOTH the page identity AND that config fields match the form.
Menu 241's real page is gsp-company-service (the complete, canonical page — lazy Company
dropdown 8, eager gspProviders, EINV/EWAY/BOTH service-type, auth-token fields, on
`/gsp-company-services` + grid 27). There is a SEPARATE gsp-service page (route
`/master/gsp-service-master`) that is just a tender-type-master placeholder — NOT menu 241.
An earlier mistake rebuilt gsp-service onto the gsp-company-service backend (duplicate) before
realizing gsp-company-service already existed; that was reverted. Lesson: when a path is given
for a menu, check for sibling pages (`find features -type d -iname '*<thing>*'`) and the app
route that imports each before editing. If the bridge map maps nothing, the wiring is a silent no-op.
Related: [[erp-grid-columns-config]], [[erp-workspace-layout]], [[erp-configured-dropdowns]].
