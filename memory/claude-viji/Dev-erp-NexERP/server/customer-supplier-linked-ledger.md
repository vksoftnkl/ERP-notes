---
name: customer-supplier-linked-ledger
description: "customer & supplier masters share their PK with a linked account_ledger_master row, synced via AccountLedgerMastersService on create/update/softDelete"
metadata: 
  node_type: memory
  type: project
  originSessionId: d2efb5a2-3f81-4035-8bbf-cf327f65d203
---

**Customer** (`cus*`) and **supplier** (`sup*`) masters each provision a linked `acc_ledger_master` row whose `ledId` **equals the master PK** (`cusId === ledId`, `supId === ledId`). This is the ledger analogue of the account-group link used by state/city/area — see [[sales-masters-linked-account-group]].

The master→ledger field copy is encoded in `CUSTOMER_TO_LEDGER_FIELD_MAP` / `SUPPLIER_TO_LEDGER_FIELD_MAP` and applied by `buildLinkedLedgerDto` (present-fields-only). Required ledger fields: `ledName` (=master name), `ledStateName`, `ledStateCode`. `ledGroupId` = customer's `cusAreaId` (area shares PK with an account group) / supplier's fixed `SUPPLIER_LINKED_LEDGER_GROUP_ID`.

Sync goes **through the ledger service**, not raw writes:
- create → `accountLedgerMastersService.createLedgerWithinTx(dto, tx)`, then `masterId = ledger.ledId`.
- update → `updateLedgerWithinTx(dto, tx)` (made public for this), guarded by a `findFirst` so legacy masters with no ledger are a no-op. Set `dto.ledId = masterId` first.
- softDelete → raw `tx.accLedgerMaster.updateMany({ where:{ ledId, ledIsDeleted:false }, data:{ ledIsDeleted:true, ledIsActive:false, ... } })`.

Gotchas: `updateLedgerWithinTx` enforces company-scoped ledger-name uniqueness — customer has NO customer-table name check, so a customer rename's uniqueness is enforced only via the ledger; the customer service catches that `ConflictException` and re-throws as a `cusName` conflict. `cusName` is nullable but `ledName` is required → the sync falls back to the ledger's current name when `cusName` is empty. Customer/supplier controllers already dispatch create-vs-update via `save()` on PK presence (no controller change needed). `syncBankAccounts` no-ops when no bank items are passed (customer/supplier never send them).

**Since notes 81 (2026-10-03) one ledger can back BOTH roles** (`cus_id = sup_id = led_id`), via `supLinkLedId` / `cusLinkLedId` on create: no ledger write at all on a link create. Any code that assumes "this ledger belongs to one role" is now wrong: role edits keep the group (`ledIsActive = either role active`) and role deletes keep the ledger while the other role row is live (`findCustomerRole` / `findSupplierRole`). A deleted role row on a shared ledger is revived in place on re-link (PK clash otherwise). Spec: test/party-shared-ledger-notes-81.e2e-spec.ts. The ledger master's own delete refuses (400) while a live customer/supplier/sale agent owns the ledger (notes 82 ledger-delete, ensureLedgerHasNoOwner; spec test/ledger-delete-owner-guard.e2e-spec.ts).

