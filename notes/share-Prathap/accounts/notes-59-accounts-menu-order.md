# notes (59): the Accounts menu (&5) — names and order by nature

2026-09-28, the user: "change accounts menu names and position according to their nature."
`fixed.menu_master` has no update route, so this is one migration for you. **Ids do not change**,
so no client registration moves. Separators (`menu_separator = true`) draw the line AFTER the
item.

## The order

| pos | id | name (new) | was | visible | separator after |
|---|---|---|---|---|---|
| **Masters** | | | | | |
| 1.00 | 54 | Ledger Groups | Ledger Group Master | ✓ | |
| 2.00 | 53 | Ledgers | Ledger Master | ✓ | |
| 3.00 | 55 | Opening Balances | Opening Balance (0.00) | ✓ | ✓ |
| **Bill-wise settlement (one party, against bills)** | | | | | |
| 10.00 | 99 | Bill-wise Receipt | Receipt | ✓ | |
| 11.00 | 100 | Bill-wise Payment | Payment | — hidden until the payment screen is built | |
| 12.00 | 51 | Received Cheques | (same) | ✓ | |
| 13.00 | 52 | Issued Cheques | (same) | ✓ | |
| 14.00 | 263 | Cheque Books | (same) | ✓ | ✓ |
| **Vouchers (the Voucher Register, Tally order)** | | | | | |
| 20.00 | 104 | Contra | (same) | ✓ | |
| 21.00 | 261 | Payment Voucher | (same) | ✓ | |
| 22.00 | 260 | Receipt Voucher | (same) | ✓ | |
| 23.00 | 103 | Journal | (same) | ✓ | |
| 24.00 | 259 | Sales (Accounting) | (same) | ✓ | |
| 25.00 | 163 | Purchase (Accounting) | (same) | ✓ | |
| 26.00 | 102 | Credit Note | (same) | ✓ | |
| 27.00 | 101 | Debit Note | (same) | ✓ | ✓ |
| 28.00 | 262 | Voucher Register | (same) | ✓ | |
| **Retired / not built, at the end, hidden** | | | | | |
| 90.00 | 48 | Bill wise Receipt (old) | | — (already inactive) | |
| 91.00 | 49 | Bill wise Payment (old) | | **— now hidden** (retired by 100) | |
| 92.00 | 187 | Collection Entry | | — | |
| 93.00 | 188 | Collection Approval | | — | |
| 94.00 | 179 | Claim Management | | — | |
| 95.00 | 185 | Third Party Bills | | — | |

**Why these names:**
- **Bill-wise Receipt / Payment:** 99/100 settle ONE party's bills (the Receipt screen). 260/261
  are the register's many-party vouchers. The old names "Receipt" and "Receipt Voucher" read as
  the same thing.
- **Ledgers / Ledger Groups / Opening Balances:** masters read as plurals of what they hold.
- **Vouchers in Tally's F4–F10 order:** Contra, Payment, Receipt, Journal, Sales, Purchase,
  Credit Note, Debit Note. The register closes the group.

The old 48/49 rows keep an "(old)" suffix, so nobody mistakes them for 99/100 if they're ever
switched on.

## SQL

```sql
UPDATE fixed.menu_master m SET menu_name = v.name, menu_position = v.pos,
       menu_separator = v.sep, menu_visiblity = v.vis
FROM (VALUES
  ( 54, 'Ledger Groups',            1.00, false, true),
  ( 53, 'Ledgers',                  2.00, false, true),
  ( 55, 'Opening Balances',         3.00, true,  true),
  ( 99, 'Bill-wise Receipt',       10.00, false, true),
  (100, 'Bill-wise Payment',       11.00, false, false),
  ( 51, 'Received Cheques',        12.00, false, true),
  ( 52, 'Issued Cheques',          13.00, false, true),
  (263, 'Cheque Books',            14.00, true,  true),
  (104, 'Contra',                  20.00, false, true),
  (261, 'Payment Voucher',         21.00, false, true),
  (260, 'Receipt Voucher',         22.00, false, true),
  (103, 'Journal',                 23.00, false, true),
  (259, 'Sales (Accounting)',      24.00, false, true),
  (163, 'Purchase (Accounting)',   25.00, false, true),
  (102, 'Credit Note',             26.00, false, true),
  (101, 'Debit Note',              27.00, true,  true),
  (262, 'Voucher Register',        28.00, false, true),
  ( 48, 'Bill wise Receipt (old)', 90.00, false, false),
  ( 49, 'Bill wise Payment (old)', 91.00, false, false),
  (187, 'Collection Entry',        92.00, false, false),
  (188, 'Collection Approval',     93.00, false, false),
  (179, 'Claim Management',        94.00, false, false),
  (185, 'Third Party Bills',       95.00, false, false)
) AS v(id, name, pos, sep, vis)
WHERE m.menu_id = v.id AND m.menu_parent = 5;
```

When the payment screen goes live, set `menu_visiblity = true` on 100.

**The client needs no change for the ids.** The menu tree shows the new names as soon as the rows
change. Screen titles are the screens' own (the Voucher Register's "Opened from:" uses the voucher
TYPE's name from `/vouchers/types`), so a screen titled "Receipt" keeps that title until it is
renamed in the client too. That's a follow-up only if you want the titles to match.
