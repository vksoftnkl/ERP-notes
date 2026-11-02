# Notes 82 — The ledger delete removes a customer's or supplier's ledger and leaves the party active

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap · **Follows:** notes 81

## What happens

`DELETE /account-ledger-masters/delete?ledId=<id>` soft-deletes the ledger even when that id is also a
row in `sales.customers` or `purchase.suppliers` (cus_id / sup_id IS led_id). The customer or supplier
stays **active** but now points at a deleted ledger.

Reproduced live on 2026-10-03:

| Step | Result |
|---|---|
| `POST /customers/create` "ZT-LEDDEL-C" | 201, cusId `01a0fff0-50f0-…` |
| `DELETE /account-ledger-masters/delete?ledId=<that id>` | **200** "Account ledger deleted successfully" |
| `acc_ledger_master.led_is_deleted` | `true` |
| `sales.customers.cus_is_deleted` | **`false`** ← the customer is still active |

That customer can still be picked on a sale bill, but its ledger is gone. The posting, the statement and
the outstanding all key on that ledger. (The test row was cleaned up through `/customers/delete`.)

## What the client does now

The Qt Ledger list refuses Delete on such a ledger and tells the user to delete it from the Customer or
Supplier master. Since notes 81, those masters drop the ledger together with the party's last role.
The Ledger Entry form also opens such a ledger view-only.

A direct API call still gets through, though, so the rule belongs on the server as well.

## Ask

In the ledger delete, refuse with **400** when an active customer or supplier row has
`cus_id` / `sup_id` = `ledId`, for example:

> "This ledger belongs to customer ‹name›. Delete it from the Customer master."

This is the same pattern as the group delete's "is used by N ledger(s)". No change to the customer or
supplier deletes; they already handle the shared ledger correctly (verified for notes 81).

## Acceptance

1. Create customer C. Then `DELETE /account-ledger-masters/delete?ledId=C` gives 400, and the ledger and
   customer both stay active.
2. The same for a supplier, and for a ledger that is both.
3. A ledger with no customer or supplier row still deletes as today.
