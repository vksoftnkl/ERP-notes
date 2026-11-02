# Notes 130: the voucher ledger list must not offer cash / bank ledgers (2026-10-16)

**STATUS:** asked. Follows notes 126 §6. A small SQL change to one grid and one route. We can't write `fixed.grid_details`.

## Why

Since notes 126 the server refuses a cash or bank ledger typed on a voucher line:
- **Jrl / DrN / CrN / PurA / SalA:** `VCH_MONEY_LEDGER_NOT_ALLOWED`.
- **RcpV / PmtV:** `VCH_MONEY_LINE_NO_TENDER`. The money side comes from the instrument.
- **Con:** each line needs its `tenderId`.

Checked live 2026-10-16 through `vouchers/validate`. But the ledger list the Voucher Register shows (grid **119**, "POPUP - VOUCHER LEDGERS") still offers Cash In Hand, Kvb Current A/c, Store Safe Cash and the rest. So the user picks one, types the amount, and only learns at save that it is wrong. The list should not offer what the server will refuse.

## Ask

1. **Grid 119** (`share/grids/voucher_ledger_picker.sql`):
   - Leave out every ledger whose `led_ledger_type` is `CASH` or `BANK`, on **every** type.
   - Remove the last block (the RECEIPT / PAYMENT clause that offers only cash / bank ledgers on the By side of RcpV and To side of PmtV). That side is now picked from the tender list (the instrument), so it lists no ledgers at all.
   - In SQL terms, replace that last `AND (NOT EXISTS … = EXISTS …)` block with:
     ```sql
     AND COALESCE(l.led_ledger_type, '') NOT IN ('CASH', 'BANK')
     ```
   - Contra no longer uses grid 119: Qt shows the tender list there (built today, below). If you prefer, keep cash/bank for `itype_code = 'Con'`. It does no harm.
2. **`GET /vouchers/ledger-pick`:** the same rule, since its description says it mirrors grid 119.
3. Please send the new grid SQL back into `share/grids/voucher_ledger_picker.sql` too, so the file and the grid stay the same.

## Qt side, built 2026-10-16

- **Contra:** the ledger cell opens the **tender list** (CASH tenders and each bank's BANK tender, from `vouchers/instruments`) instead of grid 119.
  - Picking a tender fills the ledger from it and sends `lines[].tenderId`.
  - The cell shows "BANK · Kvb Current A/c".
  - Checked live through `vouchers/validate`: Dr BANK / Cr CASH 10.00 is accepted; the same lines without the tenders → `VCH_MONEY_LINE_NO_TENDER` on both (tests/register `contra126`, 7/7).
- **RcpV / PmtV:** no change. The customer / supplier line's instrument (Ctrl+Q) already picks the tender.

Tell Viji when it is deployed. Claude will check that grid 119 returns no CASH / BANK ledger on each of the 8 types.
