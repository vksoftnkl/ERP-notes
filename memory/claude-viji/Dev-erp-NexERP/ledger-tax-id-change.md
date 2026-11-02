---
name: ledger-tax-id-change
description: ledGstRate/ledTaxability/ledTaxRate were DROPPED from the ledger create DTO for a single ledTaxId — sending any of them is a hard 400
metadata: 
  node_type: memory
  type: project
  originSessionId: 710f791a-1472-4ec1-9cd3-20c6a06e4ac6
  modified: 2026-09-17T12:09:30.636Z
---

Verified live 2026-09-17 on `/api/v1/account-ledger-masters/create`.

`ledGstRate`, `ledTaxability` and `ledTaxRate` no longer exist. Sending ANY of
them is a **400 "property X should not exist"**, so the old client could not
save a ledger at all — this was breaking, not cosmetic.

They are replaced by one **`ledTaxId`** → `inventory.tax_rate_master(tax_id)`
(`accounts.acc_ledger_master.led_tax_id`, FK `fk_led_tax`). The rate master
already holds the rate, the CGST/SGST/IGST split and the taxability, so nothing
was kept for symmetry.

Picked **exactly like the item master's default tax**: `F::apiDropdown(...,
36, "ledTaxName")` — dropdown **36 "TAXES"** = `tax_id` / `tax_name`. No
`.withDisplayText()`: the save DTO has no `ledTaxName`.

The GET resolves three companion keys for display: `ledTaxName`,
`ledTaxRatePerc`, `ledTaxTaxability` (e.g. `GST 18%` / `18` / `TAXABLE`).
A tax id not in the rate master is a 400 on `ledTaxId`.

On the form the picker is its own **"Tax Rate"** section on the GST & Tax tab,
shown for SalesPurchase *and* Tax profiles — for a duty ledger it is what
replaced "Percentage of Calculation". Supply Details moved onto that tab too;
before, a sales ledger opened an EMPTY "GST & Tax" tab.

It is a **one-field section, folded onto its own heading row** —
`Tax Rate [ GST 18% ] ────` — via the new `NexCrudField::inHeadingRow()` engine
verb (see [[crud-form-heading-row-field]]), so the word does not appear twice on
two lines.

Also fixed in passing: the bank grid was looked up as `control("banks")` but its
field/payload key is **`ledgerBankAccount`**, so edit mode never loaded it.

See [[ledger-type-derivation]], [[ledger-group-tests]].
