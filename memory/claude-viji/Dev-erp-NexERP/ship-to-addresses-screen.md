---
name: ship-to-addresses-screen
description: "Ship-to addresses moved OFF the ledger master onto the Customer list (menu-less, grid 110): the two screens, the beforeSubmit traps, and the test-harness rebuild trap"
metadata:
  node_type: memory
  type: project
---

Built 2026-09-18, replacing the read-only grid that had been on the ledger
party tab for one day (see [[ship-to-grid]], now obsolete).

**The shape the user asked for.** Ship-to addresses hang off the **CUSTOMER**,
not the ledger — a supplier or a tax head has nowhere for goods to go, so a
grid on the ledger master asked every ledger a question only a customer can
answer. 3.0 did the same (`pos/Accounts/shippingaddrentry.cpp`, reached from
the party). Reached now by a **"Ship Address" toolbar button on the Customer
list** (`addActionButton`, `requiresSelection = true`, Ctrl+H,
`transport.svg`) → `ShipAddrList` → `ShipAddrEntry`.

**Deleted:** `nex_ship_addr_grid.{h,cpp}`, `nex_ship_addr_entity.h`, the
`ledgerShipAddress` field + `sec("ship", { Party })` on LedgerEntry.

**Both screens are config-driven, no .ui.** The user first said "separate ui,
don't use NexDialog", then reversed it: **list = `NexCrudMasterPage`, entry =
`NexCrudForm`**. Files live in `src/modules/sales/master/ship_address/`.

**grid 110 "CUSTOMER - SHIP TO ADDRESSES"** registered on the box via
`POST /grid-details/create` (15 columns, scoped `WHERE saa_ledger_id =
isaa_ledger_id::uuid`). Passing `grid_id` to that route means UPDATE and 404s
if it does not exist — **omit it to create**, the server assigns the next id.
ui_table 34 ("LEDGER - SHIP TO GRID") is now orphaned; its DB rows are
duplicated per device_type but the API returns one clean set.

**Two traps that cost a cycle each**

1. **`beforeSubmit` runs in `onSaveClicked()`, AFTER `collectJson()`.** A test
   asserting on `collectJson()` alone tests the wrong half of the save — it
   will not see `saaLedgerId` (stamped, never a field: nobody picks the party
   on that screen) or `saaCountryCode`. Drive the real save with
   `QMetaObject::invokeMethod(form, "onSaveClicked", Qt::DirectConnection)`.
2. **The engine submits an untouched QLineEdit as `""`, not null.** `saaEmail`
   is format-checked, so a blank one is a hard 400 "must be an email". Every
   optional string is nulled in `beforeSubmit`. This is the text-field twin of
   the combo fix in [[ledger-master-gap-analysis]] — expect it on any form
   whose endpoint validates a format.

**State follows the GSTIN.** `saa_gstin` is NOT NULL and its first two
characters ARE the GST state code, so `applyStateFromGstin()` sets the picker
(dropdown 21) from it; the CODE is authoritative and the NAME is fetched for
display only. The picker stays editable — an address can sit in a state the
party's GSTIN does not name.

**Tests:** `tests/shipaddr/run.sh` (+ `test_ship_addr.cpp`), 36 live checks,
seeds and deletes its own party + address. `tests/ledger/run.sh` updated: it
now asserts the grid is GONE. Both pass.

**Harness trap now fixed in `tests/shipaddr/run.sh`:** these runners link
ninja's own `.o` files, so an edit that has not been compiled is simply not in
the binary — the run silently tests the PREVIOUS version and the failure looks
like a bug in the fix. The runner now calls `ninja` itself first;
`tests/ledger/run.sh` and the others still do not. See
[[qt-offscreen-test-harness]].
