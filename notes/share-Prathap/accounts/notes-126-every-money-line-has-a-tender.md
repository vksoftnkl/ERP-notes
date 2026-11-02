# Notes 126: every line on a cash or bank ledger carries a tender (2026-10-16)

**STATUS:** asked. Server rule + two reports + a small master clean-up + **§6 the agreed screen design (user, 2026-10-16): the money side of every voucher is a TENDER list, never a ledger.** Qt side listed at the end.

## Why (the user's goal)

> Day Book **by tender** (cash, UPI, card, cheque, bank), so a missing transaction can be found.

The user's own words: *"we want to achieve accounts by tender types, which will be very helpful to find a missing txn"*. This also closes the till's last gap: cash that moves in the books without the till or safe knowing.

**Accounts do not change.** A tender is a label on a money line (like a cost centre), not a ledger. The legs, the Trial Balance, P&L, Balance Sheet and ledger balances all stay the same. Bill-wise Receipt (99) and Payment (100), which pick the tender first, are already right and do not change.

**How other ERPs do it.** In Tally, a bank ledger line carries a *Transaction Type* (Cheque, e-Fund Transfer, Cash, …). Odoo has *payment methods* per bank journal. SAP B1 has *payment means*, and Zoho Books a *payment mode*. The pattern is the same everywhere:
- **ledger = where** the money sits; only cash and bank ledgers count;
- **tender = how** it moved, on **every** line that touches such a ledger.

None of them puts a tender on every ledger, and neither do we.

## What already exists (no change)

- **Tender master:** tender → ledger, ledger mandatory. **It stays the only tender↔ledger map.** No tender list on the ledger master, so there are not two maps to drift apart.
- **`led_ledger_type`** CASH / BANK. The Qt ledger form derives and locks it from the group (Cash-in-Hand → CASH, Bank Accounts → BANK). Live: Cash In Hand, Cashin Hand, Cash in Transit, Till Cash Awaiting Receipt, Store Safe Cash = CASH; Kvb Current A/c, Test Bank Account = BANK.
- **Tender rows** are written by bills, returns, bill-wise receipt / payment, ExpV, temp credit, re-tender, and issued cheques (CR the cheque book's bank).
- **`loadInstrumentLedgers` (voucher-facts.ts §3):** the register already refuses a direct leg on Cheques In Hand and on every non-cash tender's clearing ledger (UPI Clearing).

## The gap (live evidence)

Lines on a CASH or BANK ledger with **no tender row**, posted on or after 2026-10-08, the day the till went live:

| Where | Example | Why there is no tender |
|---|---|---|
| Voucher Register RcpV / PmtV with no instrument | `rcv00140`, `rcv00146` (15-10): Dr Cash In Hand 15,000 each, Coimbatore (a till branch), no session, no tender row | `voucher-register.service.ts` ~423: `!d.legs.some(l => l.instrument)` skips `routeMoneyDoc` |
| Any call with no device id | all 10 RcpV on the till cash ledger since 10-08 (tester1) | `routeMoneyDoc` and `assertNoTillCashLeg` return early when `!caller.deviceId` |
| Journal / Contra on Cash or Kvb | allowed from a back-office device; the safe ledger has no check at all | `assertNoTillCashLeg` only looks at a device holding a live session, and only at the till cash ledger |
| Till transfers TFlt / TDrp / TRcv / TVar / CRem | 53 + 55 + 22 + 13 + 1 vouchers since 10-08, none with a tender row | cash moves between till and safe as plain legs |

These are test posts, but they prove the paths are open.

## Asked

### 1 · One rule, every voucher, every module

**Every leg on a ledger whose `led_ledger_type` is CASH or BANK carries a tender row** (`acc_tender_detail`, `td_voucher_id` set, same amount and side).

**The row always names a tender MASTER (`td_tender_id`), never only a type.** The user wants the Day Book by tender master too, e.g. "CARD – HDFC machine 1" apart from "CARD – ICICI machine 2". When the server fills the tender itself, it picks the tender master row of that company/branch whose `tnd_ledger_id` is this ledger:
- for a CASH line, the branch's CASH tender;
- for a BANK line, the BANK / CHEQUE tender the user picked.

If more than one tender master of the chosen type points at the ledger, the user picks one; the server never guesses. If none fits, refuse (`VCH_NO_TENDER_FOR_LEDGER`, naming the ledger) so the master gets fixed. Put the check in the shared posting path (`postLegs` or the step just before it), so present and future modules get it without remembering: payroll, purchase, branch transfer and the rest.

| Ledger type | Tender on the line | Who picks |
|---|---|---|
| CASH | CASH | **the server, automatically.** The till / safe routing (`routeMoneyDoc`) then always runs. |
| BANK | BANK (NEFT / IMPS / statement entry, ref no.) or CHEQUE (cheque book of *that* bank). UPI-out if you allow it. | the user; refused if missing (`VCH_MONEY_LINE_NO_TENDER`, field `lines.N`) |
| clearing / Cheques In Hand | as today (§3 lock): only the bill, the settlement (TSet) or the cheque routes | none |

- **A tender that doesn't fit the ledger** is refused, e.g. CARD on Kvb: "Kvb Current A/c takes Bank or Cheque".
- **Received** cheques keep using the tender master's ledger (Cheques In Hand). **Issued** cheques keep using the cheque book's bank. The direction tells them apart, as today.
- **Opening balances** are not transactions. They are out of scope.
- **No new DB function or trigger** (house rule): the rule lives in NestJS. A CHECK or index is fine.

### 2 · "No device" is not "no till"

`routeMoneyDoc` / `assertNoTillCashLeg`: a call with no device id is a **back-office** call (cash from the safe, or `TILL_SESSION_REQUIRED` under REFUSE). It is never a pass. The same goes for a Journal / Contra touching the **till cash or a safe ledger** from any device: refuse it (`TILL_CASH_LEDGER_DIRECT`) unless it is one of the till's own voucher types. A real correction goes through Cash Office / a till movement, so the safe book and the books move together.

### 3 · Till transfers get tender rows

TFlt, TDrp, TRcv, TVar, CRem (and any later till movement): one CASH tender row per cash leg (till, safe, in-transit, awaiting-receipt ledgers), with the session where there is one. CRem's bank leg gets BANK. Then the tender Day Book shows the full cash path: drawer → safe → bank.

### 4 · Two reports (own routes, notes "new module = own URLs")

1. **Day Book by tender:** tender rows by date / branch / voucher type, with opening and closing per tender-ledger.
   - **Group by: tender master** (each machine / QR / cash tender on its own) **or tender type** (all CARD together). Both come from the same rows (`td_tender_id`, `td_tender_type_id`).
   - Filters: company, branch, date range, tender master(s) or type(s), session.
2. **Money lines with no tender:** every leg on a CASH / BANK ledger with no matching tender row. **This list should always be empty.** Anything on it is the missing transaction the user is looking for. Grid-registrable SQL is enough; Qt will list it.

Please also run 2 once over the data since 2026-10-08 and send us the count. The rows above should all appear.

### 5 · Tender master clean-up (data, plus one relaxation)

| Tender | Today | Should be | Why |
|---|---|---|---|
| CARD | Kvb Current A/c | new ledger **Card Clearing** (group like UPI Clearing), settles to Kvb, 1 day | card money is not in the bank until payout; the bank's fee (MDR) shows only through the settlement |
| WALLET | Kvb Current A/c | new ledger **Wallet Clearing**, settles to Kvb | same |
| TEMP.CR | Kvb Current A/c | **no ledger** | `buildTenderLegs` writes no leg for it: the link is unused and misleading |
| CREDIT | Sales Freight Collection | **no ledger** | same |
| LOYALTY | Cash In Hand | **no ledger** (or the redemption ledger) | posting ignores it and uses LOYALTY_REDEMPTION |
| BANK, CHEQUE, CASH, UPI | as now | as now | |

The tender master's ledger is mandatory today. Please make it **not required for CREDIT / TEMP_CR / LOYALTY types** (they post no tender leg), and required for the rest. The 40 card rows already posted to Kvb are test data: changing the master is enough.

Also please check the two cash ledgers **"Cash In Hand"** and **"Cashin Hand"**. If the second is a duplicate, mark it inactive.

### 6 · Agreed design: the money side is picked from the TENDER list (user, 2026-10-16)

The user's call, after weighing it against the rule in §1: *"on money selection area user must select tender master list only"*. On every voucher screen the user **never picks a cash or bank ledger**. They pick a **tender master**, and the ledger comes from it. This is how the counter, Receipt (99) and Payment (100) already work, and how SAP B1 (payment means), Odoo (payment methods) and Zoho (payment mode) work. Every screen then works the same way, and every money line has a tender by construction. §1 stays as the server's safety net, for a direct API call.

```
Money side ▾
  CASH                 → Cash In Hand  (till / safe routing)
  UPI – Paytm QR       → UPI Clearing
  KVB – NEFT/IMPS      → Kvb Current A/c
  CHEQUE               → pick a cheque book → its bank (pay) / Cheques In Hand (receive)
```

**Per voucher type:**

| Voucher | Money side | Server change |
|---|---|---|
| **RcpV / PmtV** | tender list (the existing instrument: tender on the party line, money leg generated from it) | Already supported. **Make the instrument mandatory for money:** refuse a typed leg on a CASH / BANK ledger, the same way §3 `loadInstrumentLedgers` already refuses Cheques In Hand and clearing ledgers. In practice, add every CASH / BANK-type ledger to that set for these types. |
| **Contra** (cash ↔ bank, bank ↔ bank) | tender on **both** sides ("from" tender, "to" tender) | **New:** Contra has `vchr_instruments = false` today. Let it carry a from-instrument and a to-instrument, both generating the legs, with §1/§2 routing on the CASH side. Cash → bank keeps its own path too (Cash Office remittance, CRem). |
| **Jrl, DrN, CrN, PurA, SalA** | **no money side.** CASH / BANK ledgers are not offered and are refused. | **New:** extend the refusal to these types. Bank charges / interest from the statement go on a **Payment / Receipt with tender KVB – NEFT**, not a Journal. Please give the refusal a `code` and a helpful sentence ("Bank charges go on a Payment voucher: pick tender KVB – NEFT"), because accountants used to Tally will try a Journal first. |

**Tender master for this:**
- **One BANK tender per bank account** ("KVB – NEFT/IMPS"; add "HDFC – NEFT" when that bank is opened). Today's single BANK tender (ledger Kvb) is fine.
- **One CHEQUE tender only:** received cheques go to its ledger (Cheques In Hand); paid cheques go to the bank of the cheque book picked. This already works (167 issued cheques post CR Kvb).
- **A new bank or cash ledger** can't be used in a voucher until its tender exists: refuse with `VCH_NO_TENDER_FOR_LEDGER` (§1), naming the ledger and saying "create its tender in Tender Master". (Qt: the tender master's ledger field now has Alt+C → New Ledger, so both are made in one go.)
- **A rare genuine correction** on a cash/bank ledger: cancel and re-enter. If you think a controlled exception is needed, use an approval rule (e.g. CASH_LEDGER_ADJUST, store manager). Your call; the user hasn't asked for one.

## Future modules: checked, no conflict

| Case | Tender |
|---|---|
| Payroll: salary payable journal | none (no money ledger) |
| Payroll: salary paid | BANK (Kvb) or CASH (from safe) |
| Purchase bill on credit | none |
| Purchase paid in cash at once | CASH → till / safe routing |
| Stock branch transfer | none |
| Cash sent to another branch | CASH out (branch A safe) + CASH in (branch B safe), one row each end |
| Cash deposit to bank | CASH out + BANK in (CRem) |
| Bank → bank | BANK + BANK |
| Bank charges / interest from the statement | Payment / Receipt with tender KVB – NEFT, statement ref (never a Journal, §6) |
| GST / TDS / loan / capital | BANK or CHEQUE |
| Card / UPI settlement | clearing → BANK (TSet) |
| Depreciation, provisions, year-end | none |

No case breaks the accounts. The rule only stops a cash or bank line from posting without saying *how*.

## Qt side (after 1 is live)

- **Voucher Register (§6):**
  - **RcpV / PmtV:** the money-side picker shows the **tender master list** (not ledgers); Cheque opens the existing instrument popup (cheque book → bank).
  - **Contra:** "from" and "to" tender pickers, once the server accepts instruments on Contra.
  - **Jrl, DrN, CrN, PurA, SalA:** cash and bank ledgers are left out of the ledger picker (grid 119).
  - Refusals show in plain words.
- **Tender master:** Alt+C on the ledger field opens New Ledger (built 2026-10-16, dropdown 43 → menu 53).
- **Ledger master:** no new field. Ledger Type stays derived and locked from the group.
- **Reports:** Day Book by tender and the "no tender" list, under Accounts.
- **Business Day (274):** one check row, "Books = drawers": till cash ledger balance vs live drawers' expected, and safe ledger vs the safe book.

Tell Viji when 1–3 and §6 are deployed. Claude will rerun the till suites and post one of each case above on the ZT test data.
