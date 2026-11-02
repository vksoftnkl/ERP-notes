---
name: erp-quotation-entry-port
description: Quotation Entry React port (2026-07-30) — where the pieces live and the deliberate decisions that are not derivable from the code
metadata: 
  node_type: memory
  type: project
  originSessionId: febff91c-b70f-4e0c-9cf0-588ab8852047
  modified: 2026-07-31T06:06:17.331Z
---

Ported the Qt "Quotation Entry" screen to React on 2026-07-30. Layout:

- `domain/pricing/*` — the framework-agnostic engine (`recalcDocument`, `distribute`,
  `priceRowGoods`). No React, no API import. 37 golden tests in `pricing.test.ts`,
  run by **vitest** (added this session: `npm test`, `vitest.config.ts`, `decimal.js-light`
  promoted from a transitive dep to a direct one).
- `features/sales/quotation/` — `quotation.{types,constants,utils,state,payload,validate}.ts`,
  `use-quotation-draft.ts` (all network), `components/*`. `page.tsx` is a **router between two
  views on one route**: `components/quotation-list-view.tsx` — which IS a master page, i.e. the
  shared `CrudMasterPage` shell — and `components/quotation-entry-view.tsx` (the voucher form,
  opened by the shell's Add / Edit through `onCreateAction` / `onEditAction`, left through a
  guarded "‹ Quotations"). One route, not two, because the form is one long-lived draft and a
  quotation is keyed by four fields, not one id.
- **`CrudMasterPage` gotcha:** it defaults `listResponseStyleArrayKey` to `"styles"`, the
  response-driven column mode. On a configured grid that carries no `styles` array the table then
  renders a serial-number column and NOTHING else. Every master page on a configured grid passes
  `listResponseStyleArrayKey=""`. Also: `createLabel` sets the Add button's tooltip, not its text;
  Edit/Delete live in the toolbar and need a row selected first; and `grid_column_filter` doubles
  as "searchable" AND "sortable".
- `store/api/quotationApi.ts` + a `"Quotation"` entry in `baseApi.tagTypes`.

Decisions that the code cannot explain on its own:

- **The charge picker is `GET /charges/get?chgModule=S`, not configured popup grid 82.**
  Grid 82 selects only `chg_id, chg_name`, and `cd_ledger_code` is NOT NULL with no server
  default — a row saved without it 404s. The master row carries `chgLedgerCode`, `ledHsnSac`,
  `ledGstRate` and `chgAutoApply` (which is what pre-seeds a new document's charge rows).
- **No virtualisation.** Nothing in this client virtualises (`@tanstack/react-virtual` is in
  package.json with zero imports); the configured layout hides enough of the 89 columns that a
  plain table is fine. Row rendering is isolated so it can be dropped in later.
- **`sqAccYear` comes from `activeFiscalYear.name`** (`fiscal_years.fy_year_name`, already
  9 chars), not from the voucher date — it is immutable after create and feeds the voucher
  sequence's period key. Validation refuses a quote date whose derived year differs.
- **`user_master.usr_language` is `'en'`, an ISO-ish code, not a language name.** The regional
  flag compares against a set of English codes; a naive `!== "english"` test made every lookup
  ask for the Tamil name.
- **`usr_edit_rate`** (via `GET /user-administration/get?usrId=<v7>`) gates the rate and
  price-level cells. There is **no** skip-MRP column anywhere in `user_master` — that stays a
  named constant in `quotation.constants.ts`.
- **The voucher policy has no session source and is therefore operator-visible.**
  `sq_freight_calc_type` / `sq_loading_calc_type` / `sq_disc_alter_base` exist on
  `sale_quotation` and **nowhere else** — no company column, no settings table, nothing in the
  session (verified by an `information_schema` sweep). Left implicit they pinned every new
  quotation to `manual`, which made the per-line FreightPerQty / CoolyPerQty columns, the
  distance-band lookup and the validation role gate all unreachable. They are now three
  controls in the Quotation block, lower case on the wire because they double as
  `/item-price`'s `freight_type` / `loading_type` (closed sets; freight has no `auto`).
- The browse list is configured grid **84** ("Quotation"; grid 83 is an older variant of the same
  SQL). Its config as shipped was unusable and was **fixed in the DB on 2026-07-30** (approved):
  widths `100` → `10` (the table reads a bare number as a percent and renders `100%` as 800px, so
  20 columns made an 8,048px table), `grid_column_filter = true` on
  `sq_quote_refno / sq_usr_refno / sq_cust_name / sq_cust_phone` (with none set, the runner injects
  `1 = 0` and every search returns nothing), and the raw-id columns hidden. `sq_is_deleted` is
  deliberately left VISIBLE: the grid's SQL has no `WHERE`, so deleted rows are listed and that
  column is the only thing distinguishing them.
- Deliberate gaps, all commented at the point of impact: no HSN on the item-price lookup, no
  agent/salesman NAME on the GET, no per-line unloading column (so the UNLOADING role override
  can never fire), no print endpoint on the server.

A third trap, worth remembering: an **RTK Query lazy trigger fired from a mount effect resolves
with `undefined` and never touches the network** (its subscription does not exist yet). The
list→form load silently produced a blank document until it was switched to
`dispatch(api.endpoints.getQuotation.initiate(key, {subscribe: false, forceRefetch: true}))`.
Lazy triggers are fine from event handlers, which is why the F8 path worked all along.

Two other whole-class traps this screen paid for, worth checking on the next voucher screen:
`store/api/*` modules all inject into ONE `baseApi` with `overrideExisting: true`, so a
generically-named endpoint (`getItemByBarcode`) silently replaces another feature's; and a grid
cell that keeps a local edit buffer must (a) re-seed when the draft changes under a focused
editor and (b) never commit an untouched buffer — otherwise walking the one-of-three discount
columns with Enter wipes the discount just keyed.

Verified live: a new quotation saved (`quo00008`), reloaded with identical stored figures, and
the engine reproduced the stored per-line values to the paisa. Delete and "remove a row then
save" are blocked by a server bug — see [[erp-audit-delete-action-blocks-soft-delete]].

Related: [[erp-ui-table-width-is-not-a-percent]], [[erp-item-price-unit-id-is-iuc-id]],
[[erp-configured-dropdowns]], [[erp-client-ui-browser-verification]],
[[erp-quotation-entry-legacy-ui-redesign]].
