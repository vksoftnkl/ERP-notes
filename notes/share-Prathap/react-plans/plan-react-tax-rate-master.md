# GST Rate Master (menu 40) — React port plan

Scope: the whole screen — the list, the entry form's **Rate** tab, and its
**Ledgers** tab. Small by line count and unusually dense in rules, because
almost everything this screen appears to decide is actually decided by the
database or by a server-side SQL filter, and the port's main job is **not to
restate any of it in TypeScript**.

Qt source: `src/modules/inventory/master/tax_rate/`

    tax_rate_list.{h,cpp}             NexCrudMasterPage, grid 103
    tax_rate_entry.{h,cpp}            NexCrudForm, tabs Rate | Ledgers
    tax_rate_ledger_grid.{h,cpp}      the Ledgers tab
    tax_rate_ledger_delegate.{h,cpp}  its cell editors
    tax_rate_ledger_entity.h          column indices + local fallback layout

Schema: `20_tax_rate_master.sql` (the rate + its overrides),
`19_posting_roles.sql` (`accounts.acc_ledger_role`), `21_accounts_posting_setup.sql`
(the resolver and the guard). Companion plans: **`plan-react-dropdown.md`** for
the searchable-picker component, whose §s on `dropdown_param` and the runner's
paging cap apply here verbatim.

> **The whole screen in one line:** a rate is one row of
> `inventory.tax_rate_master`, and its Ledgers tab is zero or more rows of
> `inventory.tax_rate_ledger` saying "for THIS rate, role X posts to ledger Y
> instead of the chart-wide default". **No rows at all is a complete and
> normal configuration.**

---

## 1. What this screen is, and what it is not

`inventory.tax_rate_master` replaced `inventory.item_tax_master`. It is the one
master that answers "what is 18% GST", and every posting decision resolves
through it. It is **not** a per-company table: a rate is global, exactly like
the posting ledgers it names, because the model is ONE SHARED CHART OF ACCOUNTS
and company separation lives on `av_company_id` on the voucher rows.

The Ledgers tab is an **override** surface, not the mapping itself. Ledger
resolution is two-level:

    1. inventory.tax_rate_ledger   this rate's override, for one role
                                   (optionally for one supply nature)
    2. accounts.acc_ledger_map     the chart-wide default, and the ONLY home
                                   for ROUND_OFF / DISCOUNT_ALLOWED /
                                   WRITE_OFF / ADVANCE_RECEIVED

Set nothing on this tab and you get one Sales ledger and one Output CGST
ledger. That is a complete configuration and the normal case for most rates.
The tab exists for the two splits a real chart legitimately wants, and they
compose:

- **by rate** — 'Sales @ 18%', and 'Output CGST @ 2.5%' where the chart keeps
  output tax rate-wise (common Tally/VAT practice);
- **by supply nature** — 'Local Sales' vs 'Interstate Sales'.

Do not "simplify" this to one ledger column on the rate row. That argument was
made during the schema work, was wrong, and the reasoning is recorded in
`21_accounts_posting_setup.sql` §7e: a preference about one chart of accounts
is not a structural constraint.

---

## 2. Module layout

```
src/features/taxRate/
  TaxRateList.tsx            // grid 103
  TaxRateEntry.tsx           // tabs: Rate | Ledgers
  components/
    RateTab.tsx              // name, code, taxability, rate %, the three
                             // GENERATED mirrors, cess, state cess, options
    LedgerTab.tsx            // the override rows (§5)
    RolePicker.tsx           // popup grid 104  (§5.2)
    LedgerPicker.tsx         // popup grid 105, filtered by the row's role
  api/
    taxRate.ts               // list / get / create / delete / resolve
  payload/
    buildSavePayload.ts      // §4.4 — the three rules that are not optional
    parseLoadedRate.ts
  derived.ts                 // the CGST/SGST/IGST mirror, pure (§4.2)
  validate.ts                // §6
```

---

## 3. The list

Grid **103** `MAIN LIST - GST RATES`. Nothing unusual: the standard configured
grid, same component as every other master list. Row click opens the entry with
`tax_id`; the create button opens it empty.

---

## 4. The Rate tab

### 4.1 Fields

| field | control | note |
|---|---|---|
| `tax_name` | text, **required** | e.g. "GST 18%" |
| `tax_code` | text | "GST18" |
| `tax_taxability` | select, **required** | TAXABLE / EXEMPT / NIL_RATED / NON_GST / ZERO_RATED — `ck_tax_taxability` |
| `tax_rate_perc` | number | the one figure the three splits come from |
| `tax_cgst_perc` | number, **disabled** | GENERATED |
| `tax_sgst_perc` | number, **disabled** | GENERATED |
| `tax_igst_perc` | number, **disabled** | GENERATED |
| `tax_cess_basis` | select | NONE / PERCENT / PER_UNIT / BOTH |
| `tax_cess_perc`, `tax_cess_per_unit` | number | |
| `tax_acess_basis` | select | the **state** cess — same four values |
| `tax_acess_perc`, `tax_acess_per_unit` | number | |
| `tax_supersedes_id` | searchable select, dropdown 53 | the rate this one replaces |
| `tax_sort_order` | number | drives list order and the role catalogue's order |
| `tax_is_reverse_charge`, `tax_is_active` | checkbox | |

### 4.2 The three splits are GENERATED — show them, disable them, strip them

`tax_cgst_perc` / `tax_sgst_perc` / `tax_igst_perc` are GENERATED columns. The
API **rejects** them on write. They are on the form because the operator needs
to see 18 becoming 9 + 9, and they are disabled because typing in them could
never change anything.

`derived.ts` is a pure function — `rate => { cgst: rate/2, sgst: rate/2, igst: rate }`
— recomputed as the rate is typed so the mirrors track before any save. It is
display only; the database's value is authoritative on reload.

### 4.3 There is a SECOND cess, and it is not decoration

`tax_acess_*` is the state cess. It exists because `sbi_acess_*`,
`gdr_state_cess_value` and the duty head 'State Cess' all expected one. Every
consumer that adds cess to a value must add both: the arithmetic is
`value * (1 + (gst + cess + acess)/100) + cessPerUnit + acessPerUnit`. Both are
zero on all eleven seeded rates today, which is exactly why a port that forgets
the second one will look correct for a year.

### 4.4 Save payload — three rules, none of them optional

1. **Strip the generated splits.** Remove `tax_cgst_perc`, `tax_sgst_perc`,
   `tax_igst_perc` before POST, and the display-only `tax_supersedes_name`.
2. **`ck_tax_cess_agrees`** — a basis of `NONE` must be sent with zero figures
   or the row is refused. Zero them at build time rather than clearing the
   inputs, so switching a basis back to None does not strand what was typed.
3. **`lines` is ALWAYS sent** — see §5.4.

`POST /api/v1/tax-rates/create` with `tax_id` present is the update;
`GET /tax-rates/get?tax_id=`, `GET /tax-rates/list`, `DELETE /tax-rates/delete`.

---

## 5. The Ledgers tab

One row is one `inventory.tax_rate_ledger` record: **role**, optional **supply
nature**, **ledger**, remarks, active.

### 5.1 The columns

    trl_id            hidden
    trl_role          hidden — the code; the label is what the operator reads
    Role              picker, grid 104
    Supply Nature     blank | INTRA | INTER — a plain three-value select
    trl_ledger_id     hidden
    Ledger            picker, grid 105, filtered by the row's role
    Remarks           hidden (2026-09-12) — kept in state and sent back, so
                      hiding the column cannot drop what another client wrote
    Active            checkbox

### 5.2 Role and Ledger are PICKERS, not dropdowns

Searchable popup grids, the same gesture as the item cell on a sales line —
**not** combo boxes. This was a deliberate change on 2026-09-12: eleven roles
fit in a combo, the ledger list does not, and one idiom for both is what the
operator's hands already know.

    104  POPUP - POSTING ROLES      roles with alr_by_rate = true
    105  POPUP - LEDGERS FOR ROLE   takes `itrl_role`, applies the role's own
                                    requirements in SQL

**Grid 105 is where the rule lives, and the client must not restate it.** Its
join against `accounts.acc_ledger_role` applies `alr_want_type` /
`alr_want_duty` / `alr_want_nature`, so Sales offers income ledgers, Output
CGST offers a ledger whose duty head is Central Tax, and the React code knows
none of that. It also requires `led_company_id IS NULL`: a global rate pointing
at a company-scoped ledger would post everyone's revenue into one company's
books.

`PopupConfig` in Qt is per COLUMN while the role is per ROW, so the Qt version
re-stamps the ledger picker's parameter on every cursor move. In React this is
simply `<LedgerPicker role={row.role} />` — the awkwardness does not port.

Grid 104 also returns `alr_by_supply` as a hidden column, so **picking a role
teaches the row** whether Supply Nature may be edited at all. Keep that: it
means the tab still behaves correctly when the separate role-catalogue lookup
(dropdown 52) fails.

### 5.3 Supply Nature is editable only where the role allows it

Blank is a real choice and the DEFAULT one — it means "answers both natures",
not "unset". It is editable only on a role whose `alr_by_supply` is true (Sales,
Sales return, Purchase, Purchase return). A tax role already implies its nature
— CGST and SGST exist only on an intra-state sale, IGST only on an inter-state
one — so narrowing one further could never be the best match.

Where the cell is not editable, **say so**. Grey it and state why; an input
that silently refuses is the single most common way this tab was reported
broken.

### 5.4 The `lines` contract — three rules the API documents

1. **`lines` is ALWAYS sent.** `[]` means *delete every override*, which is
   exactly what an operator who cleared the grid meant. Omitting the key means
   *leave the grid alone*. These are different requests; do not let a form
   library omit an empty array.
2. **A blank supply nature is sent as `null`, never `""`.** The unique index is
   `NULLS NOT DISTINCT`, so null is what makes one row answer both natures.
3. A line **with** `trl_id` is an update, **without** is an insert, and one on
   the rate but absent from the array is soft deleted.

### 5.5 Client-side validation before save

Per row: a role is chosen; a ledger is chosen (*"an override with no ledger
says nothing — delete the row to fall back to the posting ledger map"*); and no
two rows claim the same (role, nature). Report the ROW NUMBER, not a constraint
name — the database will otherwise say `ux_trl_rate_role` and mean nothing to
anyone.

---

## 6. `/tax-rates/resolve` — use it, do not reimplement it

`GET /api/v1/tax-rates/resolve?tax_id=&supply_nature=` (shipped 2026-09-12)
answers **where this rate actually posts**: every role it can influence, with
`ledger_id`, `ledger_name`, and `source` ∈ `RATE` | `MAP` | `UNMAPPED`.

Put it on the Ledgers tab as a read-only "where this rate posts" panel. It is
the only honest way to show the operator the *result* of the two-level
resolution, and it keeps the resolution in one place — the server. Never
compute the fallback client-side.

Today it answers `UNMAPPED` for all 14 roles, because nothing can write
`acc_ledger_map` yet (§8).

---

## 7. Do not port the Qt layout machinery

Three things in the Qt version are workarounds for `NexTable`, not design, and
have no React equivalent:

- **the local fallback layout** (`tax_rate_ledger_entity.h`) — `loadLayout()`
  hides every column until the server names it back, so the grid needs a
  locally seeded layout to survive `ui_table` 29 being unreachable. A React
  table renders its own columns; the server layout is at most a preference.
- **the deferred first edit** — `editItem()` straight after `insertRow()` is
  refused because the view has not laid the row out yet.
- **`setAutoDefault(false)` / `Qt::NoFocus` on the buttons** — Enter handling
  in a QDialog.

What DOES port is the reason all three existed: **every one of them made the
grid look dead under the operator's hand, and none was visible from reading the
code.** The React equivalents are real focus management and never rendering a
control that silently does nothing.

---

## 8. Backend gaps this screen inherits

1. **§7f — twenty posting ledgers are company-scoped and must be global.**
   `accounts.acc_ledger_master.led_company_id` = Acme Foods on the twenty
   posting ledgers, and both dropdown 51 and grid 105 filter
   `led_company_id IS NULL`. **Until the UPDATE in
   `21_accounts_posting_setup.sql` §7f runs, the Ledger picker returns zero
   rows for every role.** The rule itself is verified: drop that one clause and
   OUTPUT_CGST returns exactly one row.
   The API cannot fix it — `SaveAccountLedgerMasterDto.ledCompanyId` is a
   non-nullable uuid string and there is no update route, so `null` is dropped
   on merge.
2. **`accounts.acc_ledger_map` has no endpoint.** The table exists live with 0
   rows. Its screen is deferred; read is already solved by grid 106.
   Requested in `notes.txt` 2026-09-12 (2).
3. **`chg_tax_id` and `led_tax_id`** are in the schema and on neither save DTO.
4. **`item_default_tax_id` still FKs the OLD table.** See
   `25_item_tax_to_tax_rate.sql` — until it runs, the item master cannot point
   at this master at all.

---

## 9. Tests

Pure first, and these are the ones that catch real defects:

- `derived.ts` — 18 → 9/9/18; 0.25 → 0.125/0.125/0.25 (the rounding is real:
  0.25% exists as a statutory slab).
- `buildSavePayload` — the generated splits are absent; `NONE` basis zeroes
  both figures; `lines: []` survives serialisation as an empty array and is
  **not** dropped; a blank nature serialises as `null`, and `JSON.stringify`
  keeps it.
- `validate` — duplicate (role, nature) including two rows both blank, which is
  the case `NULLS NOT DISTINCT` makes illegal and naive dedupe misses.
- Ledger picker: changing a row's role **clears** the ledger already picked.
  Output CGST wants a Central Tax ledger and Sales wants an income ledger, so a
  carried-over ledger is a pair the guard rejects at save.
- A rate with no lines round-trips: load → save → still no lines, and no
  accidental delete-everything.

---

## 10. Phases

1. List (grid 103) + the Rate tab read-only. Proves the endpoints and the
   generated-column display.
2. Rate tab editable + `buildSavePayload` + validation. Shippable on its own:
   rates can be maintained with no overrides at all.
3. Ledgers tab with both pickers. **Blocked on §8.1** for anything but an empty
   list — build it, but expect an empty ledger picker until those twenty
   ledgers go global.
4. The `/tax-rates/resolve` panel (§6).

---

## 11. Open question

The Qt screen has no **company** control because a rate is global. If a company
ever needs its own Sales or Output CGST ledger, the twelve ledger columns stop
being able to answer, and the mapping moves to a `(company, tax_id)` child
table — `fn_tax_ledger_for()` is the only thing posting calls, so the change
stays behind one function. Nothing in the React screen should make that harder:
keep ledger resolution behind `api/taxRate.ts`, never inline in a component.

