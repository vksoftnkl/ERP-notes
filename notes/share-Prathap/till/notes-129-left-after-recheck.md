# Notes 129: what is left after the 2026-10-16 recheck

**STATUS:** asked. We rechecked notes 124 / 125 / 126 / 127 / 128 live on 2026-10-16, about 4:30 PM. Everything below is **still open**. The rest is done (list at the end).

## 1 · Supervisor approves their own cash into the safe (notes 127 §3 / 124 §11)

**Not done.** The `sessions/approve-cash` docs still say the caller is never the session's cashier (`TILL_RECEIVER_SELF`) unless `till.single_operator`.

**Ask (unchanged):** also allow the session's own cashier when the company's CASH_VARIANCE rule has `tar_allow_self` (or an APPROVE_CASH rule, as you prefer).
- **Always with their PIN.** The PIN-free path stays only for the owner under `till.single_operator`.
- Keep it journalled (`tap_self_allowed`), so the Cockpit and the day Z can show "approved by own cashier".

## 2 · Receipt surcharge rounding (notes 127 §4 / 107)

**Not seen.** `ReceiptTenderDto` / `SaveReceiptTenderDto` are the same as this morning.

**Ask (unchanged):** work out the surcharge inside the tender, `surchargeAmt = round2(tdAmount × perc / (100 + perc))`. Then 200 received by UPI at 1% is **202.00**, not 202.02, the same as on the sale bill. Please tell us when it is live; we'll post a 200 receipt to check.

## 3 · Cash counted twice after till transfers got tender rows (notes 128 §3)

Till transfers (TFlt / TDrp / TRcv / TVar / CRem) now carry CASH tender rows.

**Ask:** tell us which readers you changed to leave the transfer rows out where they don't belong (or that they never summed tender rows):
- drawer expected (Cockpit, close count);
- safe book;
- 40A(3) check;
- day Z;
- Business Day;
- Account Day Summary.

We will then compare the Cockpit, Business Day and day Z figures for 15-10 and 16-10 against before.

## 4 · New: the RRN tender points at an expense ledger

The tender master **RRN** (type RRN) has the ledger **SALES FREIGHT COLLECTION** (`led_ledger_type` EXPENSE). It is not in the notes 126 §5 table, and the notes 128 save check doesn't cover the RRN type.

**Ask:**
- What is RRN for?
- If it moves money, point it at a clearing ledger and add RRN to the save check. If it moves none, treat it like CREDIT / TEMP_CR / LOYALTY: no ledger (`TENDER_LEDGER_NOT_ALLOWED`).

## 5 · Gaps before the change (information, then one question)

`reports/tender-day-book/no-tender` from 08-10 to 16-10 lists **402 lines, ₹12,65,953.36**:

| Type | Lines |
|---|---|
| TDrp | 120 |
| TFlt | 106 |
| TRcv | 44 |
| Rev | 36 |
| ChqBnc | 30 |
| TVar | 23 |
| PmtV | 18 |
| RcpV | 13 |
| TPIn | 6 |
| CRem | 4 |
| Bil | 1 (bil01062, ₹70, Cash In Hand, 08-10) |
| TSet | 1 |

The only one today is **trc00074** (TRcv ₹2,354, Store Safe Cash / Till Cash Awaiting Receipt), made at 9:57 AM, before your change went live. **No new gap since then.** Thank you.

**Question:** do you plan a one-time back-fill of tender rows for the old till-transfer vouchers (TDrp / TFlt / TRcv / TVar / CRem / TPIn)? Then the Day Book by tender is complete from the start of the till. If not, say so: they are mostly test data, and we'll leave them on the report.

## 6 · Three answers still missing (notes 127 §7)

| From | Question |
|---|---|
| 124 §2 / notes 115 §3 | The till events the e2e cleanup deleted: restored from a backup, or lost for good? |
| 124 §3 / notes 122 §6 | Negative safe opening on the day Z (13-10: −1,426.00; 15-10: −1,346.00): print it as it is, flag it, or count approved hand-overs back to the day they were handed over? |
| 124 §4 / notes 122 §2 | The 500s on "Plain stock" bills, 15-10, 12:30–1:10 PM IST: what does the server log say was the cause? |

## Checked live and DONE (no action)

- **125:** `UserType` has STORE MANAGER / AREA MANAGER / HO FINANCE / CASH OFFICE, MANAGER is gone, and both users were renamed. `USER_ROLE_RESERVED` is in the docs. *Not tested:* it needs a client ADMIN login, so please confirm you tested it. **Qt:** the User Admin role list was switched to the new names today.
- **126:**
  - Both reports are live.
  - Refusals checked through `vouchers/validate`, nothing saved:
    - Journal with a cash or bank line → `VCH_MONEY_LEDGER_NOT_ALLOWED`;
    - Contra with no tender → `VCH_MONEY_LINE_NO_TENDER` on both lines;
    - Payment Voucher with no instrument → `VCH_MONEY_LINE_NO_TENDER`.
- **126 §5 / 128 §2:**
  - CARD → Card Settlement Clearing, WALLET → Wallet Clearing.
  - TEMP.CR / CREDIT / LOYALTY have no ledger.
  - "Cashin Hand" is inactive.
  - Save check (`tender-masters/create`, nothing saved):
    - CARD on Kvb → `TENDER_CLEARING_LEDGER_IS_MONEY`;
    - CASH on Kvb → `TENDER_CASH_LEDGER_NOT_CASH`;
    - CREDIT with a ledger → `TENDER_LEDGER_NOT_ALLOWED`.
- **127 §6:** `TILL_FORCE_CLOSE_SELF` is documented on `sessions/force-close`. Not tested: it needs a live session of the caller.

**Not checked by us yet:** 127 §5, the error codes from notes 119. We'll check them in the next round.

Tell Viji when 1–4 are deployed. Claude will recheck each live.
