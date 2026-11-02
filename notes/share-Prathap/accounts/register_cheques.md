# Register cheques: the Receipt Voucher takes instruments (rev 1, 2026-09-26)

The picture is `register_cheques_ui_mockup.png`. The user confirmed Q1–Q7 on it
("ok") before this file was written. Companion files:

- `voucher_register.md` (the register; not repeated);
- `../cheques/` (Received Cheques, menu 51, which is **unchanged** by this);
- `../receipt/` (the Receipt, menu 99, whose tender and cheque writer this reuses).

> **In one line:** the operator types the customers; each customer line carries
> its instrument; the server works out the debit legs from the instruments, and
> a cheque becomes a HELD row in menu 51. The bank is chosen at deposit.

## 1. Why

- A cheque is received before anyone knows which bank it will be paid into. That is
  what **Cheques In Hand** is for; Suspense is not the place.
- The register refused Cheques In Hand (S16) because menu 51 owns the cheque
  rows. So a salesman's collection run (Receipt Voucher, MANY since notes 53)
  could not take cheques at all.
- This keeps S16 as it is: the operator still never *types* Cheques In Hand. The
  ledger only ever arrives as a **generated** leg, from a CHEQUE instrument,
  and the cheque row is written by the same code the Receipt uses.

## 2. Decisions (all confirmed on the PNG)

| # | Decision |
|---|---|
| Q1 | The instrument sits on a **customer (party) line**. One line, one instrument, one party. Part cash plus part cheque from one customer is two lines. A bounce reopens only that line's bills. |
| Q2 | The **debit side is generated**, one leg per instrument, on the **tender master's ledger**. CASH → Cash In Hand, CHEQUE → Cheques In Hand, BANK / UPI → their ledger. A voucher with no instruments works exactly as today (typed Dr cash/bank line). |
| Q3 | **No bank is asked at entry.** It is chosen at Deposit in menu 51. |
| Q4 | A cheque dated **after** the voucher date is **post-dated**. Its party line, its allocations and its Cheques In Hand leg post as **their own voucher on the cheque date**, and its bills settle then (the Receipt's rule, menu 99). |
| Q5 | **Cancel is refused** once any cheque of the voucher has left HELD (deposited, cleared, bounced …). Reverse it through menu 51 first. |
| Q6 | **The Payment Voucher is next (#2):** the same column; a CHEQUE takes the next leaf of the bank's cheque book → Issued Cheques (52). It needs the `share/payment` cheque-book and issued-cheque routes first. Out of this rev. |
| Q7 | **Collected by** on the header: a salesman (dropdown 38), stored in the existing `avh_employee_id uuid[]` as the Receipt does. Optional. |

## 3. The wire (proposed; Prathap may rename, and the client follows the live Swagger)

### 3.1 `GET /vouchers/types`: one more rule column

`instruments: boolean` per type, from a new `acc_voucher_types.vchr_instruments`
(true for **RcpV** only in this rev). The screen shows the Instrument column,
the Cheques card and Collected-by **only** where it is true. There is no client
rule.

### 3.2 `GET /vouchers/instruments?companyId` (new, own URL)

These are the tenders usable as an instrument on a register line. They are active
and not TEMP_CR / CREDIT / LOYALTY / RRN; those are sale-bill settlements, not money
received.

```jsonc
{ "instruments": [ { "tenderId": "…", "name": "CHEQUE", "type": "CHEQUE",
                     "ledgerId": "…", "ledgerName": "Cheques In Hand",
                     "isCash": false, "isCheque": true, "needsRef": true } ] }
```

### 3.3 The payload: an optional `instrument` on a typed line

The names mirror `SaveReceiptTenderDto` / `SaveReceiptChequeDto`, so one writer serves
both.

```jsonc
{ "rowNo": 1, "drCr": "CR", "ledgerId": "<Anand Stores>", "amount": "10000.00",
  "instrument": {
    "tenderId": "<CHEQUE tender>",
    "refNo": "456789",                 // cheque / UTR / UPI ref
    "instrumentDate": "2026-09-26",    // after header.date = post-dated (Q4)
    "bankName": "State Bank of India", // drawee
    "cheque": { "drawerName": "Anand Stores", "bankBranch": "Musiri", "ifsc": null, "micr": null }
  } }
```

`header.employeeIds: uuid[]` (Q7) → `avh_employee_id`.

Refusals (they join the `VCH_` list):

- `VCH_INSTRUMENT_NOT_ALLOWED`: the type's `instruments` is false, or the line
  is not a party line;
- `VCH_INSTRUMENT_TENDER`: an unknown or unusable tender;
- `VCH_CHEQUE_DETAILS`: a CHEQUE without `refNo`, `instrumentDate` or `bankName`;
- `VCH_CHEQUE_MOVED` (cancel, Q5): names the cheque and its status.

### 3.4 `/validate`: what it derives

- **One generated leg per instrument line:**
  - side opposite the line, the tender's ledger, the line's amount;
  - `source: "INSTRUMENT"`, `role` = the tender type (CASH / CHEQUE / BANK / UPI), `fromRows: [line]`;
  - `instrument: {tenderName, refNo, instrumentDate, postDated}`.
- **Post-dated** (Q4): the cheque's leg **and** its typed party line carry
  `postDated: true, postsOn: "<date>"`.
  - `totals` covers **today's** voucher only.
  - A new `derived.postDated[]` lists `{postsOn, lineRowNos, debit, credit}`, one entry per future voucher.
- Allocations on a post-dated line are validated against the bills as they stand
  (they settle on `postsOn`).

### 3.5 `/post`

1. **Today's voucher** as now, plus the generated instrument legs.
2. **For each instrument:**
   - an `acc_tender_detail` row (`src ACCOUNTS/VOUCHER`);
   - for a CHEQUE, an `acc_pdc_register` row **HELD**, with `apd_party_id` = the line's party, `apd_tender_id`,
     `apd_voucher_id`, and `apd_posting_mode ON_RECEIPT`.
   - Write them with the **Receipt's writer**, not a copy.
3. **Allocations of a cheque line** carry `abj_cheque_id` (so a bounce in 51 reverses
   exactly them).
4. **Post-dated:** one voucher per cheque date, same type, its own number, with
   `avh_against_voucher_id` → today's voucher. It is written in the same
   transaction, exactly as the Receipt's PDC voucher is.

### 3.6 `/cancel`, `/get`, the draft

- **cancel:** refuse with `VCH_CHEQUE_MOVED` unless every cheque is HELD. HELD
  rows → CANCELLED; the post-dated voucher(s) are reversed with it.
- **get:** each typed leg returns its `instrument` (as sent), plus `pdcId`, `pdcStatus`,
  and for a post-dated one `postDatedVoucherId / Refno`.
- **draft:** `avh_draft_lines` keeps the `instrument` verbatim.

### 3.7 Menu 51

Menu 51 itself needs no change. One client fix is owed: Enter on a cheque whose voucher is a
**register** voucher must open the Voucher Register, not the Receipt screen. The
row's voucher type decides.

## 4. The Qt client (coded once §3 answers on the box)

- **Legs grid:** a new **Instrument** column.
  - It is appended to ui_table 39 as column 12 and positioned after Group, so no index shift. Re-register, post twice.
  - A chip (CASH / CHEQUE / …) plus `no · date · bank`, and a PDC chip.
- **Ctrl+Q:** the instrument dialog on a party line.
  - The tender combo comes from `/vouchers/instruments`.
  - The cheque fields; drawee from grid 111 "POPUP - BANKS", as the Receipt does.
  - A post-dated note when the date is after the voucher date.
- **Generated instrument legs** paint grey with their chip; post-dated ones are amber, "its own voucher, dated …".
- **Cheques card** in the lower band (shown when `instruments`): line · party · no · date ·
  bank · drawer · amount · HELD / PDC chip, plus the three hint lines from the PNG.
- **Balance card:** today's totals, plus "post-dated, own voucher on …".
- **Collected by:** NexDropdownSingle (dropdown 38) on the header when `instruments`.
- **Menu 51 drill-through** by voucher type (§3.7).

## 5. Tests

- **Server** (erp_dry, then ZT data):
  - three customers — a cheque today, cash, a post-dated cheque. It posts. Check the legs, one HELD `apd_` row per cheque, the PDC voucher on its date, and `abj_cheque_id` on the cheque lines only;
  - bounce the first cheque in 51: only that customer's bills reopen;
  - deposit one, then cancel the voucher → `VCH_CHEQUE_MOVED`.
- **Client:** `tests/register/run.sh cheques`, the same three-customer case through the screen.
  - Check the generated legs and the Cheques card, and that the draft round-trips the instruments.
  - Post, then check menu 51 lists the cheques, then cancel.
