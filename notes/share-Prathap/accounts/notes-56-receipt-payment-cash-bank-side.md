# notes (56): Receipt / Payment Voucher — the cash/bank side, enforced on the server

2026-09-28. The user's decision: on a Receipt Voucher, **By (Dr) takes only cash and bank
ledgers, and To (Cr) takes everything else, never cash or bank**. A Payment Voucher is the
mirror. The Qt Voucher Register already enforces this in its ledger popup. `/vouchers/validate`
and `/vouchers/post` do not enforce it yet, so a voucher that doesn't come through that popup
can still break the rule.

## What is live today

Here is what `acc_voucher_types` has for the two types:

| type | `vchr_dr_groups` | `vchr_cr_groups` | effect |
|---|---|---|---|
| RcpV (30) | `{}` | `{}` | **any ledger on either side**: cash can be both By and To |
| PmtV (31) | `{}` | `{Cash-in-Hand, Bank Accounts}` | Cr is right; Dr still accepts cash/bank |

`legalOnSide()` (`vouchers/voucher-derive.ts:332`) treats an empty list as "any group", and a
list can only say which groups are **allowed**. It has no way to say "everything except cash
and bank", and that exclusion is what the non-money side needs.

As a result, both of these post clean today:
- a receipt with Dr Cash / Cr Bank, which is a **Contra** (a cash deposit) filed as a receipt.
  It lands in the receipt register and puts the day-close cash figures out;
- a receipt that debits an expense or party ledger, which is a Journal filed as a receipt.

## The ask

**One rule keyed on `vchr_nature`**, applied everywhere a side is checked:

| nature | money side (cash/bank ONLY) | other side (cash/bank NEVER) |
|---|---|---|
| `RECEIPT` | DR | CR |
| `PAYMENT` | CR | DR |
| anything else | no change | no change |

A ledger counts as "cash/bank" when its group, or any ancestor, is **Cash-in-Hand**,
**Bank Accounts** or **Bank OD A/c**. That's the same three-group test the instrument
exclusion already uses. Matching on the reserved group ids is better than on names if you
have them.

Where it goes:

1. **`legalOnSide()`** (validate and post). After the existing group-list test, apply the nature
   rule. A failure is the existing `VCH.LEDGER_SIDE` refusal. Suggested messages:
   - money side: `Row 2: Sales A/c (Sales Accounts) may not be debited on a Receipt Voucher — the DR side takes cash or bank only`
   - other side: `Row 1: Cash In Hand may not be credited on a Receipt Voucher — cash to bank is a Contra`
2. **`ledgerPick`** (`vouchers/voucher-lookups.service.ts:85`). Apply the same filter, so the
   route and the Qt grid offer the same list.
3. **The party checks** (`voucher-derive.ts:560`) need nothing extra, since a party is never under
   cash/bank.

The vouchers affected are **new validates and posts only**. Leave already-posted vouchers as
they are.

## Decide: Bank OD on a Payment

PmtV's `vchr_cr_groups` lists Cash-in-Hand and Bank Accounts but **not Bank OD A/c**. So a
payment from an overdraft account is refused today, while RcpV (empty lists) would accept a
receipt into one. We suggest adding Bank OD A/c to PmtV's Cr list (a data change), so both types
treat the three groups the same. If OD accounts are meant to go through Contra only, say so and
the Qt grid will drop Bank OD too.

## Optional: the RcpV data row

Once the nature rule is in, RcpV's empty `vchr_dr_groups` stops mattering. For symmetry with
PmtV, and for anyone reading the table, you could set it to `{Cash-in-Hand, Bank Accounts}`
(plus Bank OD if you agree to the above) in the same migration. Either way works; the rule is
what enforces it.

## What the client already did (for your tests)

- Grid 119 "POPUP - VOUCHER LEDGERS" has the nature rule in its SQL
  (`share/grids/voucher_ledger_picker.sql`, section dated 2026-09-28). A read-only dry run on
  .106 gave these counts:
  - RcpV: DR 241 → 4 (cash/bank only), CR 241 → 237 (no cash/bank);
  - PmtV: DR 241 → 237, CR 4 → 4;
  - Jrl and Con unchanged.
- The screen now enters a receipt **party first**: line 1 opens on To (the customer), and the By
  line opens pre-filled with the difference. That changes nothing in the payload.

**Tests to add:**
- RcpV with Cr Cash is refused;
- RcpV with Dr Sales A/c is refused;
- RcpV with Dr Bank / Cr customer passes;
- PmtV with Dr Cash / Cr Bank is refused;
- Jrl with Dr Cash / Cr Bank still passes;
- `ledgerPick` for RcpV `side=DR` returns only cash/bank.
