---
name: place-of-supply-split
description: "CustStcd vs PosStcd are two separate facts on every sales payload, and why the default POS comes from AppSession"
metadata: 
  node_type: memory
  type: project
  originSessionId: a00e4b3d-7fec-4c75-a81a-54925d7dd66f
  modified: 2026-09-11T04:51:10.313Z
---

As of 2026-09-11, on quotation, sale order and sale bill alike, the customer's
registered state and the document's place of supply are **two different fields
with two different homes**:

- `*CustStcd` ← `m_voucherParam.stateCode`, filled by `fetchCustomerDetail()` from
  the master's `state_code`. Never touched by the dropdown.
- `*PosStcd` / `*StateName` ← `ui->ddPosStcd` (dropdown 21), which the operator
  can override for a ship-to across a state line.

They agree on an ordinary document and differ on a cross-border one. `isLocalSale`
is derived from the **POS**, never from the lookup's `local_sales` flag — that flag
is only the fallback when the customer master has no state at all. Loads read
`*CustStcd` with a fallback to `*PosStcd`, for records written before the split.

**Why:** before this, all three screens wrote both keys from one source, so an
overridden quotation told the bill importing it that the *customer* lived in the
ship-to state. The bill and order had no POS widget at all and
`applyPlaceOfSupply()` was dead code on both.

**How to apply:** when adding a sales document screen, wire `ddPosStcd` to
dropdown 21, seed it silently from `AppSession::companyStateCode()` +
`companyStateName()` (the name exists only because `NexDropdownSingle` cannot
resolve a name from an id, and it keeps the default off a hardcoded 33 /
"Tamil Nadu"), and connect `selected` — never `setSelection` — to
`applyPlaceOfSupply()`. Related: [[widget-signal-wiring-rule]],
[[charge-distribution-model]].

Also fixed in the same pass: `ChargeGridController::clear()` drops rows but
**keeps** `m_interState`, so every reset must call `setInterState()` back or the
next document's own-GST charge rows stay on IGST.
