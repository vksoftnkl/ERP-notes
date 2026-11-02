---
name: gst-rate-master-screen
description: "The GST rate master screen (menu 40): its files, the server ids registered for it, and where it stands"
metadata:
  type: project
---

`src/modules/inventory/master/tax_rate/` — built 2026-09-11, compiles clean.
NexCrudForm, not a Designer `.ui`: every inventory master uses it, and
LedgerEntry + NexBankGrid already prove header-plus-child-grid.

    tax_rate_list.{h,cpp}            NexCrudMasterPage, grid 103
    tax_rate_entry.{h,cpp}           NexCrudForm, tabs Rate | Ledgers
    tax_rate_ledger_grid.{h,cpp}     Ledgers tab, NexCrudFieldWidget -> lines[]
    tax_rate_ledger_delegate.{h,cpp} role / nature / ledger editors
    tax_rate_ledger_entity.h         column indices + local fallback layout

**Registered live via Swagger** (all landed on the ids the client hard-codes):
dropdown **51** ledgers legal for one role (filters in SQL against
`acc_ledger_role`, takes `itrl_role`), **52** rate-wise roles — also the role
CATALOGUE, carrying `alr_by_supply` so the grid never hard-codes which roles take
a nature, **53** GST rates for Supersedes; **ui_table 29**; **grid 103**;
menu **40** (pre-existing "Tax Master") registered in `form_inventory_menu.h`.
Same content also written as `24_tax_rate_ui_registration.sql`.

**Three payload rules the API documents and the screen honours:** `lines` is
ALWAYS sent — `[]` means delete every override, omitting the key means leave the
grid alone; blank supply nature is sent as `null`, not `""` (the unique index is
NULLS NOT DISTINCT); CGST/SGST/IGST are GENERATED, shown disabled and stripped in
`beforeSubmit`.

**Role and Ledger are POPUP GRID PICKERS, not dropdowns (user decision,
2026-09-12)** — the same gesture as the Description cell on a sales line, and
one idiom for both cells. The delegate returns nullptr for both columns;
`NexTable` opens the popup from its `PopupConfig` on any printable key, and the
grid opens it on a double click and on Add. Two new grids, **registered live
2026-09-12** (SQL + payloads kept as `grids/tax_rate_ledger_pickers.sql` and
`apply_tax_rate_pickers.sh`, which now refuses to re-run — it sends no id, so
a second run would create two MORE grids):

    104  POPUP - POSTING ROLES      alr_by_rate only; carries alr_by_supply as
                                    a hidden column, so a pick teaches the row
                                    whether Supply Nature opens
    105  POPUP - LEDGERS FOR ROLE   same rule as dropdown 51, parameter
                                    `itrl_role` — re-stamped on every cursor
                                    move, since PopupConfig is per COLUMN and
                                    the role is per ROW

Dropdown 52 is still read once per screen, now only as metadata (whether a
loaded row may carry a nature); dropdown 51 is no longer used by this screen.
Navigating onto a role-less Ledger cell only writes the hint on the summary
bar — it does not open the picker, because arrowing across a row is not a
request.

**What was actually broken (2026-09-12):** `loadRoles()` hand-built
`/dropdown-details/run?dropdown_id=52&limit=200` and the runner caps `limit` at
100, answering 400 — so the catalogue never arrived and the old combo held
nothing but its blank entry. Fixed to `NexDropdownApi::run(52, {}, 1, 100)`.
See [[api-endpoint-docs]] for the limit rule.

**The API cannot fix §7f, and now there is a structural reason rather than an
observation (2026-09-12):** `SaveAccountLedgerMasterDto.ledCompanyId` is
declared `type: string, format: uuid` with **no `nullable: true`**, so the DTO
has no way to express "make this global" — which is why sending
`ledCompanyId: null` answers "updated successfully" and changes nothing.
`/account-ledger-masters` exposes only create (upsert) / get / delete, no
update route. It needs the SQL run directly, or a backend change (nullable on
that field plus a service that writes the explicit null).

**React port plan written 2026-09-12:**
`react-plans/plan-react-tax-rate-master.md` — whole screen (list + Rate tab +
Ledgers tab), the three `lines` payload rules, why grid 105 owns the ledger
rule, `/tax-rates/resolve` as a read-only "where this rate posts" panel, and
the NexTable workarounds that must NOT be ported.

**Posting Ledgers screen (acc_ledger_map) — DEFERRED by the user 2026-09-12.**
The read side is already registered: **grid 106 SETUP - POSTING LEDGER MAP**,
one row per active role with its current local/interstate mapping (LATERAL
joins), verified returning all 20 roles. The endpoints + a menu row (250, parent
5) are requested in notes.txt 2026-09-12 (2). A screen was started and deleted
on the user's word — do not rebuild it unasked.

**RESUME HERE:** the screen is finished and the only thing stopping it is §7f in
[[posting-ledger-schema-files]] — until those 20 ledgers go global, dropdown 51
returns 0 rows and the Ledger picker opens empty. Everything else on the tab
works. Still owed API-side: a `/ledger-roles/list` is NOT needed (dropdown 52
covers it), but `acc_ledger_map` has no endpoint at all, so the Posting Ledgers
screen (mockup: `/home/vk/Shared/Prathap/ledger_map_ui_mockup.png`) cannot be
built yet; `chgTaxId` and `ledTaxId` are missing from their DTOs.

Login DTO changed 2026-09-11 to `usrLoginName` / `usrPassword` (was
username/password) — see [[api-endpoint-docs]].
