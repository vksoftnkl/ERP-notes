---
name: item-tax-to-rate-master
description: "Moving the item master off item_tax_master onto tax_rate_master: the FK that blocks it, the migration file, and the order the three steps must happen in"
metadata:
  type: project
---

The item master's Default Tax must come from `inventory.tax_rate_master` (what
the GST Rate screen, menu 40, maintains — see [[gst-rate-master-screen]]), not
the old `inventory.item_tax_master`. Three steps, and **the order is forced**.

**The blocker is the FK, and it is structural:**
`item_master.item_default_tax_id` still REFERENCES `inventory.item_tax_master`,
while `tax_rate_master` was seeded clean-sheet — so an id from the new master is
refused by the old FK, and `/tax-rates/get` answers **404** for an id from the
old one. Both remain true whatever rows exist.

The row counts quoted in `25_item_tax_to_tax_rate.sql` (10,034 items, the
per-rate split, the 19 cess rows) are **test data** — see [[db-live-connection]].
They shaped the mapping written in that file and are worth re-checking before it
is run anywhere real; they are not an argument for anything.

    1. CLIENT (done 2026-09-12) — safe on its own, degrades silently
       MasterLookups::ItemTaxDetail -> TaxRateDetail, /item-taxes/get ->
       /tax-rates/get. Field renames: tax_gst_rate_total -> tax_rate_perc,
       tax_cess_unit -> tax_cess_per_unit. The STATE cess (tax_acess_perc /
       tax_acess_per_unit) is folded into the same two accumulators — every
       consumer computes cost = costWot*(1+(gst+cess)/100) + cessQty and the
       second cess is the same two shapes, so no new arithmetic. Its error
       path is now qWarning, NOT a modal: it fires while merely OPENING an
       item, and a rate that will not resolve only means the cost split is
       unavailable.
    2. SQL — 25_item_tax_to_tax_rate.sql in the share. Fills in §4c of
       20_tax_rate_master.sql, which was left a comment waiting for the data
       step. Maps BY NAME, verifies before and after, repoints both FKs.
       `claude` has no UPDATE/DDL, so Prathap runs it.
    3. DROPDOWN 36 (TAXES) — LAST, payload ready at
       grids/dropdown_36_taxes_to_rate_master.json. It is the one step that
       breaks saving: flip it first and the screen offers ids the FK refuses,
       and being server config that lands on everyone at once.

**The item entry is the only user of either** — one `setDropdownId(36)` and one
call to `/item-taxes/get`. So the whole conversion is contained.

**The 19 cess items:** two old rows carry cess and no new row does. User's call
2026-09-12 — **the data is test-only, drop the cess**. (They are chilli powder,
turmeric, biscuits, a matchbox, aspirin and eight rows plainly typed to test the
screen; none attracts compensation cess in law.)

**The connect the user asked about is CORRECT, not a bug:**
`connect(ui->ddDefaultTax, &NexDropdownSingle::selected, …)` fires on user
intent only, and both load paths call `setTaxId` explicitly —
`item_entry.cpp:757` and `:1125`. That is [[widget-signal-wiring-rule]] working
as designed.
