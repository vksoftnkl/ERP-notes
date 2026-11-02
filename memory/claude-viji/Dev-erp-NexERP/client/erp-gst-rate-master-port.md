---
name: erp-gst-rate-master-port
description: GST Rate Master (menu 40) React port — API/grid/dropdown contract traps that are not visible from the code
metadata:
  node_type: memory
  type: project
---

Ported 2026-09-12 to `features/masters/inventory/tax-rate/`, taking over the
`/master/tax-master` route (menu 40). The old `item_tax_master` screen was deleted:
`inventory.item_tax_master` is dead — `item_master.item_default_tax_id` already FKs
`inventory.tax_rate_master`.

Contract facts the code does not show:

- **Grid 103 (`MAIN LIST - GST RATES`) 400s without a `grid_param`.** Its SQL ends
  `AND (NOT itax_is_active OR t.tax_is_active)` and the runner substitutes named
  tokens textually, so an unbound token fails the statement (`column
  "itax_is_active" does not exist`) rather than returning everything. Send
  `grid_param={"itax_is_active":true}` for live rates only, `false` for all. Same
  trap class as dropdown `dropdown_param` — see [[erp-nex-dropdown-component]].
- **`/tax-rates/resolve` answers `OVERRIDE | DEFAULT | UNMAPPED`**, not
  `RATE | MAP | UNMAPPED` as the port plan's prose says. Everything reads UNMAPPED
  until `accounts.acc_ledger_map` has rows, and nothing can write that table yet —
  that is the true answer, not a screen defect.
- **`tax_cgst_perc`/`tax_sgst_perc`/`tax_igst_perc` are GENERATED ALWAYS** and
  rejected on write. The generation is `(tax_rate_perc/2)::numeric(7,3)`, so a
  client-side mirror must round half-away-from-zero at scale 3 or it disagrees with
  the reloaded row (0.25% is a real slab → 0.125).
- **`lines` must ALWAYS be sent.** `[]` = delete every override; omitting the key =
  leave the grid alone. A blank supply nature goes as `null`, never `""` (the unique
  index is `NULLS NOT DISTINCT`).
- Role/ledger pickers are **dropdowns 52 and 51** driven by `NexDropdownSingle`
  (not grids 104/105). Dropdown 51 takes `itrl_role` and applies the role's own
  `alr_want_type`/`alr_want_duty`/`alr_want_nature` in SQL — never restate that
  client-side. Verified: OUTPUT_CGST offers exactly 1 ledger, SALES offers 6.
- The port plan's blockers §8.1 (twenty ledgers company-scoped) and §8.4 (item FK)
  were **already fixed in the DB** by the time of the port; do not re-apply them.

Gotcha hit during the port: `.erp-ms-modal .erp-ms-modal-body table :is(input, select, …)`
forces `height: 24px`, which clips a `<select>` cell editor. No class of ours can
outrank it — set the height **inline**. See [[erp-master-shell-modal-table-styles]].

Item Master repointed 2026-09-12 (`features/masters/inventory/item/`): the eager tax
records now come from **`/tax-rates/list`** (whole rates, no params — `active_only`
defaults true and the DTO 400s on unknown params), not grid 5. Grid 103 cannot feed
that math: it carries the cess BASES but neither per-unit amount. Field renames in
`resolveItemPriceTaxContext`: `tax_gst_rate_total`→`tax_rate_perc`,
`tax_cess_unit`→`tax_cess_per_unit`, the `tax_*_pur_perc` purchase columns are gone
(one rate both directions), and the state cess (`tax_acess_perc`/`tax_acess_per_unit`)
folds into the same two accumulators. Dropdown 36 (TAXES) was already repointed at
`tax_rate_master` server-side, so the picker needed no change.

**Still on the retired master (2026-09-12), each now resolving nothing for the ids
items actually carry:**
- server `items-master.service.ts` resolves `item_default_tax_name` (and the
  item-price-details `tax_perc`/`cess_*`) from `itemTaxMaster` → `/items/get` returns
  a **null tax name** for every item. The item screen works around it by seeding the
  Default Tax label from the rates it already loaded; nothing else does.
- client `store/api/lookupsApi.ts`: `getTaxOptions` (grid 5) and `getItemTaxById`
  (`/item-taxes/get`) — used by Opening Stock and the Item Template settings screen.
