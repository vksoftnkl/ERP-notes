---
name: party-dual-role
description: customer-as-supplier / supplier-as-customer on one shared led_id — notes 81 SHIPPED + verified 2026-10-03; Qt "Also a Supplier/Customer" button BUILT (tests/party 33/33)
metadata:
  type: project
---

Asked 2026-10-03. The schema needs no change: `cus_id` and `sup_id` each have their own FK to `led_id`, so one ledger can be both. On the box 0 parties have both rows.

The API blocks it. Create always makes a new ledger, and ledger names are unique per company, so a same-name supplier gets 409 `ledName`. Two more problems would break a shared ledger: each role's edit rewrites `ledGroupId` (supplier → fixed "Suppliers" group, customer → area), and each role's delete soft-deletes the ledger.

Written up as `share/accounts/notes-81-party-dual-role.md`. It asks for `supLinkLedId` / `cusLinkLedId` on create, no group write when the ledger is shared, and deleting the ledger only when it is the last role. It includes acceptance steps.

**Shipped 2026-10-03 (Prathap) and live-verified**: `supLinkLedId` / `cusLinkLedId` on create (400 for a missing, deleted or non-PARTY ledger; 409 when the role already exists); a role edit leaves a shared ledger's group alone; a delete removes the ledger only with the last role. Name, phone, etc. default from the ledger when left out.

**Qt BUILT 2026-10-03:** `src/shared/party/party_role_link` puts an "Also a Supplier" / "Also a Customer" footer button on a SAVED customer/supplier form. It is shown only when a probe of the other master answers 404 AND `canCreate(10|22)` is true. Clicking it opens the other entry form in ADD mode, prefilled from the LEDGER GET (one mapping for both directions; the supplier keys differ: supPincode/supPhone/supMailId), with the name locked and the link id sent through `NexCrudForm::setSubmitExtra`; ctx.menuId is the TARGET master's. Test: tests/party/run.sh. Server quirk: the ledger GET serves a cached 200 for about 3 s after a delete.

**How to apply:** never work around this by creating a second ledger with a different name. Related: [[api-house-conventions]], [[customer-creates-ledger]].

**Ledger form lock (built 2026-10-03):** `LedgerEntry::lockIfOwnedByParty` probes `/customers/get` and `/suppliers/get` with the ledger id; a 200 calls `NexCrudForm::lockReadOnly(reason)`, which disables the field PAGES (so it holds against later `setEnabled(true)` calls) plus Save. tests/ledger §7b covers it (fixed ids test2 / GOOGLE INDIA). The ledger LIST can still delete such a ledger; that is not blocked yet.

**Ledger list delete blocked (2026-10-03):** `PartyRoleLink::guardLedgerDelete` runs before the list's confirm. It uses `probeOwners`, the one shared probe that LedgerEntry also uses. Server side FIXED by notes 82 (verified live 2026-10-03): the ledger delete now answers 400 "This ledger belongs to customer ‹name›…" for a customer, supplier or shared ledger; a plain ledger still deletes. The client guard stays as the friendlier first line. Test trap: `#define private public` crashed the test (it changes class layout); expose a public function instead.
