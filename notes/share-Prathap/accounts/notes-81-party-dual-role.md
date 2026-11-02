# Notes 81 — One party as both customer and supplier (shared ledger)

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap

## The ask

A party we sell to is often also a party we buy from. Tally handles this with **one ledger** that is
used in sales and in purchase vouchers, so the outstanding nets itself: a sale debits the ledger and
a purchase credits it. We want the same here: make an existing customer a supplier too (or the reverse)
**without creating a second ledger**.

## The schema already allows it

- `sales.customers.cus_id` → FK `fk_cus_ledger` → `accounts.acc_ledger_master.led_id`
- `purchase.suppliers.sup_id` → FK `fk_sup_ledger` → `accounts.acc_ledger_master.led_id`

These are two separate tables with their own primary keys, so one `led_id` can have a row in both.
**No DDL is needed.** On the live box no party has both rows today (checked 2026-10-03).

## Why the API can't do it today

1. **Create always makes a new ledger.**
   - Supplier: `suppliers.service.ts:200` calls `createLedgerWithinTx` and line 201 takes its id as `supId`.
   - Customer: `customer.service.ts:345` does the same.
   - Sending an existing id as `supId` / `cusId` doesn't help, because that means *update*
     (`save()` line 69). It answers "No active supplier found".
2. **Ledger names are unique per company** (`uq_led_name_company`, `uq_led_name_shared`). Creating
   supplier "ABC Traders" while customer "ABC Traders" exists is refused with 409 on `ledName`. The only
   way round it today is a different name, which gives **two ledgers and two balances**. That is the
   outcome we're trying to avoid.

## What we're asking for

### 1. Link an existing ledger on create

Add one optional field to each create DTO:

| Route | New field | Meaning |
|---|---|---|
| `POST /suppliers/create` | `supLinkLedId` (uuid) | make this existing ledger a supplier |
| `POST /customers/create` | `cusLinkLedId` (uuid) | make this existing ledger a customer |

When the field is present:

- **No ledger is created.** Insert only the role row, with `sup_id` / `cus_id` = the given `led_id`.
- Refuse with 400 if the ledger doesn't exist, is deleted, or isn't `led_ledger_type = PARTY`.
- Refuse with 409 if the role row already exists (the party is already a supplier / customer).
- The role row's name, address, GSTIN and PAN default from the ledger when the payload leaves them out.
- **Skip the name-uniqueness check against that ledger**, because it is the same party, not a clash.
- **Leave the ledger's group alone.** A customer that becomes a supplier stays under its area group
  (under Customers / Assets), and a supplier that becomes a customer stays under **Suppliers** /
  Liabilities. Tally does it this way too: one ledger, one group.
- The customer side still needs a `cusAreaId` (`cus_area_id` is NOT NULL) for beat and area filters.
  It just must not move the ledger. See item 2.

Same audit row as a normal create.

### 2. A role's edit must not move a shared ledger's group (this breaks things silently)

- Supplier edit always writes `ledGroupId: SUPPLIER_LINKED_LEDGER_GROUP_ID` (`buildLinkedLedgerDto`,
  `suppliers.service.ts:395`, used by the update at line 284).
- Customer edit writes `ledgerDto.ledGroupId = nextAreaId` (`customer.service.ts:464`).

With a shared ledger, the group would flip between Suppliers and the area **every time either master
is saved**. That moves the balance between Assets and Liabilities in the balance sheet.

**Rule:** when the ledger also has a row in the *other* role table, the role edit omits `ledGroupId`.
The group then changes only from the ledger form, or from a customer edit when the ledger is
customer-only. That is today's behaviour for every existing party.

### 3. Deleting one role must not delete the shared ledger

- `suppliers.service.ts:123-133` soft-deletes the ledger together with the supplier.
- `customer.service.ts:254-262` does the same for a customer.

With a shared ledger, deleting the supplier would kill the customer's ledger, and with it its
receipts, outstanding and statement.

**Rule:** soft-delete the ledger only when no active row is left in the other role table. Otherwise
delete only the role row.

### 4. Field sync on edit: no change needed, just noting it

Both masters copy address, GST and contact fields onto the ledger when saved. For a shared ledger
the last save wins. That's acceptable, because it is the same party.

## What stays as it is

- `led_is_bill_by_bill` is already `true` for both roles, so receipts and payments can set bills off
  against the one ledger.
- The ledger statement and bill-wise outstanding already key on `led_id`, so the net balance shows
  with no change.
- No change to vouchers or posting. The party id on every document is already the `led_id`.

## Client side (Qt), once 1–3 are live

- **Customer master:** a "Make supplier" action. **Supplier master:** a "Make customer" action. Each
  opens the other master prefilled, with the link field set.
- The picker lists only PARTY ledgers that don't already have that role.
- Sales screens pick parties from `sales.customers` and purchase screens from `purchase.suppliers`,
  so once the second row exists the party appears in both with nothing else to change.

## Acceptance

1. Create customer C, then `POST /suppliers/create` with `supLinkLedId = C`. The result is
   `sup_id = cus_id = led_id`, there is still exactly one ledger row, and the ledger group is unchanged.
2. Repeating step 1 gives 409.
3. Editing the supplier, then the customer, leaves `led_group_id` unchanged.
4. Deleting the supplier leaves the ledger and the customer active. Deleting the customer after that
   soft-deletes the ledger.
5. The reverse direction (supplier first, then `cusLinkLedId`) behaves the same way.
6. A sale bill and a purchase bill for the party both post to the one ledger, and the statement shows
   the net figure.
