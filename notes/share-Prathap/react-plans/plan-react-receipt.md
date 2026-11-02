# Receipt Entry (menu 99) — React port plan

Scope: **one screen**, the customer receipt. It covers the header, the party
band with its two context panels, the **bills grid** (bills and credits in one
list), the **instruments grid** (instruments and role lines in one list), the
identity strip, and the full lifecycle: draft → post → cancel, delete draft,
correcting a posted receipt (amend), and walking the register. The **register
itself** is the shared transaction list and is only touched on in §14.

It is the first screen in the product that writes a leg into `acc_vouchers`.
Every rule on it serves one equation, and most of this plan is about keeping
the client's preview of that equation identical to the server's.

Qt source: `src/modules/accounts/receipt/`, about 5,400 lines, built
2026-09-16 and reworked through 2026-09-19:

    receipt_entry.{h,cpp,ui}          NexDialog: the one page
    receipt_entity.h                  both column enums, OpenBill / CreditItem /
                                      TenderRow / OtherLine / ReceiptHeader,
                                      ReceiptRoles, ReceiptAdjType, netAllocations(),
                                      linesThatTravel()
    receipt_identity.h                the equation, pure
    receipt_allocation_engine.{h,cpp} Auto-allocate, pure
    receipt_bill_delegate.*           bills grid: editability, credit tint, chips
    receipt_tender_delegate.*         instruments grid: kind-driven editability
    receipt_delegate_base.*           shared money editor + focused-cell paint
    receipt_cheque_dialog.*           F4, the optional cheque extras
    ../common/txn_doc_types_accounts.*   the register descriptor (grid 108)
    src/shared/txn/tender_master_row.h   TenderMasters::parse / isOfferable / isCheque …

Companion documents that this plan does **not** repeat:

- `../receipt/receipt_flow.md` (the spec: what a receipt writes, table by
  table, and decisions R1–R20) and `../receipt/plan-backend-receipt.md` (the
  server's transaction).
- `../receipt/plan-backend-amend-posted.md` (R20, the amend route) and
  `plan-backend-delete-draft.md`.
- `../receipt/gap-vs-3.0.md` rev 2, graded A–D, with a "where ours is AHEAD"
  section so a port does not restore something that was removed on purpose.
- **`plan-react-sale-tender.md`** for the tender-master rules (offerable on a
  date, gives change, needs a reference, is a cheque). The receipt uses the
  same `tender_master_row` helpers. Import them. Do not copy them.
- **`plan-react-dropdown.md`** for dropdowns 13, 54 and 38 and the popup grids.
  Its section on bare-parameter binding applies to dropdown 54 and grid 108
  verbatim (§4.4).
- **`plan-react.md`** §8 for the one-reducer shape used here as well.

> **The whole screen in one line:** RECEIVED + DEDUCTIONS + CREDITS =
> ALLOCATED + ON ACCOUNT + ADDITIONS. The screen shows it permanently, will
> not post while it is out by a paisa, and the server recomputes it anyway.
> The client's figures are a **preview** of the server's, and they must never
> drift from them.

---

## 1. What the receipt is, and what it is not

1. **A DRAFT touches no bill (R10).** `/create` saves the header, the
   instruments and the role lines. It writes **no** `acc_bill_adjustment` row
   and does not move `abl_pending_amount`. Since notes (30) it *remembers* the
   bill-wise settlement in `avh_draft_lines`, but only as a memo. So a
   half-keyed receipt is a normal state for an afternoon's work, not a lock on
   somebody's ledger, and two operators can hold drafts against the same party.
2. **Post is one gesture and two calls.** Save the draft if it is dirty, then
   post it. There is **no approval step**. It was withdrawn on 2026-09-15 for
   every accounts screen. The lifecycle is DRAFT → POSTED → CANCELLED
   (`api-house-conventions`).
3. **A post-dated cheque is a second voucher.** A cheque dated after the
   receipt date gets its own voucher, dated the cheque and pointing back
   through `avh_against_voucher_id`. The bills it covers do not settle until it
   matures. The screen says so **before** Post (the PDC chip) and names the
   second number after.
4. **An adjustment is never a tender.** Credits the party already holds
   (advances, credit notes, opening credits) are spent **in the bills grid**,
   not as an instrument. The tender list therefore excludes ADJUST, CREDIT,
   LOYALTY and TEMP_CR tenders (§5.1). Otherwise the day's cash-up would be
   wrong by exactly the credits applied (`party-adjustment-design`).
5. **ONE `partyId`, and it is the ledger.** `sales.customers.cus_id` **is**
   `acc_ledger_master.led_id`. Nothing is bridged or resolved. Sending
   `customerId` is a 400.

---

## 2. Module layout

```
src/features/receipt/
  ReceiptScreen.tsx            // layout + wiring, no arithmetic
  components/
    DocStrip.tsx               // number, date, their ref + date, our ref, narration, salesman
    PartyBand.tsx              // beat + customer + six facts + the three party totals
    LastReceipts.tsx           // party-context.lastReceipts
    ChequesOut.tsx             // party-context.pendingCheques
    BillsGrid.tsx              // ui_table 32: credits first, then bills (§6)
    InstrumentsGrid.tsx        // ui_table 33: instruments then role lines (§7)
    ChequeDialog.tsx           // F4, the optional extras (§7.5)
    IdentityStrip.tsx          // the six figures + difference + hint (§8)
    StatusChip.tsx             // the title-row notice (§12.3)
    PaidDetail.tsx             // F7, grid 112 (§6.7)
  state/
    draft.ts                   // ReceiptDraft + reducer (the ONLY mutable state)
    useReceiptDraft.ts         // thunks: load, save, post, amend, cancel, delete, step
  api/
    receipt.ts                 // openItems, partyContext, get, create, post, amend,
                               // cancel, delete, adjacent, duplicateCheck
    ledger.ts                  // /account-ledger-masters/get (mobile, address)
  domain/                      // no React import, no API import
    identity.ts                // ReceiptIdentity.compute (§8)
    allocate.ts                // the Auto-allocate engine (§9)
    roles.ts                   // isMirroredByBillColumn / isCreditSide / isLegSplit
    adjType.ts                 // abj_adj_type → column
    netAllocations.ts          // /get allocations[] (history) → bill rows (§11.2)
    mergeDraft.ts              // clamp a remembered settlement onto open-items (§11.3)
    mergeAmend.ts              // add this receipt back to open-items (§13.2)
    seededLines.ts             // the derived / offered role lines (§7.3)
  payload/
    buildDraftPayload.ts       // §10.1
    buildPostPayload.ts        // §10.2
    buildAmendPayload.ts       // §13.3
    parse.ts                   // every response → draft (pure; the {data:{header}} envelope)
  validate.ts                  // → Problem { message, target } (§10.4)
```

`domain/` is the part tested against the Qt client's numbers (§16). It is the
same idea as `src/domain/pricing` in the quotation plan.

---

## 3. Scope, keys and the date

- Company, branch and year are **captured once**: from the session on New, and
  from the **loaded document** afterwards. A receipt from another branch or
  year opens and is acted on with the keys it was written with. Never re-read
  them from the session mid-life.
- **Four keys address an existing receipt on every route**:
  `avhCompanyId · avhBranchId · avhAccYear · avhVoucherId` (`/adjacent` and
  `/duplicate-check` spell them without the `avh` prefix). Never send a bare id.
- **Every bill reference travels with its own year.** A 2025-26 bill settled
  in 2026-27 is routine, and `acc_bill_balance` is partitioned by
  `abl_acc_year`. So every allocation, credit and pin carries `billAccYear`.
- **The receipt's date decides which tenders are offered**, not the wall
  clock (`TenderMasters::isOfferable(t, master, on)`). A back-dated receipt
  must offer the tenders that were in force then. Re-filter when the date
  changes.
- The default date is the session's **app date**, not `new Date()`.

---

## 4. The wire contract (live Swagger, 2026-09-24)

All routes are under `/api/v1/receipts`. **Assume every response uses the
`{ data: { header, … } }` envelope.** Twice in Qt, reading one level too
shallow produced a blank value that looked like a different failure:

- `/create` returns `data.header.avhVoucherId`. Read as `data.avhVoucherId`,
  the id came back empty and `/post` then complained "avhVoucherId must be a
  valid UUID", which pointed at Post when the fault was in Create.
- `/post` returns `data.header.avhVoucherRefno`. Read shallow, a receipt that
  **had** posted showed a blank number and "Receipt  posted.", which looked
  exactly like a failed post.

`pdcVouchers[]` really is at `data` level. `parse.ts` owns the envelope
**once**, and every thunk goes through it.

| route | when | notes |
|---|---|---|
| `GET /open-items?partyId&companyId[&onDate][&mobile]` | party chosen, F5, reopen draft, begin amend | `onDate` only ages the PPD slab. It is **not** an as-of filter |
| `GET /party-context?partyId&companyId` | party chosen | `summary`, `lastReceipts[]`, `pendingCheques[]`. Failure is silent (§5.3) |
| `GET /get?avhVoucherId&avhCompanyId&avhBranchId&avhAccYear` | open by key | `header` (with `avhRevisionNo`), `tenders`, `otherLines`, `allocations` (**history**), `creditsApplied` |
| `POST /create` | save draft; step 1 of post | `SaveDraftReceiptDto`, upsert with `replace:true` |
| `POST /post` | step 2 of post | `PostReceiptDto` |
| `POST /amend` | Save changes while amending | `AmendReceiptDto` (§13) |
| `POST /cancel` | POSTED only | four keys + `reason` (required) |
| `POST /delete` | DRAFT only | four keys |
| `GET /adjacent?voucherId&companyId&branchId&accYear&direction[&status&fromDate&toDate]` | Ctrl+PgUp/PgDn | `data.voucher` is null at either end |
| `GET /duplicate-check?partyId&companyId&accYear&voucherDate&amount[&branchId][&excludeVoucherId]` | the tender Amount cell | a warning, never a refusal (§7.6) |
| `PUT /update-header` | **not used by the Qt screen** | see §17.4 |
| `POST /regularise-pdc` | **never from this screen** | a daily job's route |

### 4.1 Request shapes

```ts
type DraftBody = {                          // SaveDraftReceiptDto
  avhVoucherId?: string;                    // absent on the first save
  avhCompanyId: string; avhBranchId: string; avhAccYear: string;
  avhTenantId?: string | null;
  avhVoucherDate: string;                   // ISO
  avhPartyId: string;
  areaId?: string;                          // the beat, informational
  avhEmployeeId?: string[];                 // ONE element: "collected by ONE person"
  avhUsrRefno?: string; avhDocRefno?: string; avhDocDate?: string; avhRemarks?: string;
  avhDeviceType: 'PC' | 'WEB' | 'MOBILE' | 'POS' | 'DESKTOP';   // the React app sends WEB
  avhUserId: string;
  tenders: TenderLine[];
  otherLines?: OtherLineBody[];             // only linesThatTravel (§7.3)
  allocations: DraftMemo[];                 // the MEMO (§10.1), not the post shape
  creditsApplied: CreditBody[];
  replace: true;
};
type TenderLine = {                         // SaveReceiptTenderDto
  tdRowNo: number; tdTenderId: string; tdTenderTypeId: number;
  tdTenderLedgerId?: string; tdAmount: number;
  tdReceivedAmt?: number; tdChangeAmt?: number; tdMdrAmt?: number;
  tdRefNo?: string; tdPayerVpa?: string; tdBankName?: string;
  tdInstrumentDate?: string;                // ISO; later than the receipt date = PDC
  cheque?: { bankBranch?: string; ifsc?: string; drawerName?: string; bankLedgerId?: string };
  // tdIsPdc is NEVER sent: the server derives it from the date
};
type OtherLineBody = {                      // SaveReceiptOtherLineDto
  role?: string; ledgerId?: string;         // one or the other, NEVER both (400)
  drCr: 'DR' | 'CR'; amount: number; settlesBill?: boolean; narration?: string;
};
type PostBody = {                           // PostReceiptDto
  avhVoucherId: string; avhCompanyId: string; avhBranchId: string; avhAccYear: string;
  allocations: { billId: string; billAccYear: string; amount: number;
                 discount?: number; writeoff?: number; roundoff?: number;
                 writeoffApprovedBy?: string | null }[];
  creditsApplied?: { billId: string; billAccYear: string; amount: number }[];
  otherLineBills?: { lineNo: number; billId: string; billAccYear: string; amount: number }[];
  onAccount: number;
};
```

The DTO also accepts `tdId`, `tdAuthCode`, `tdCardLast4`, `tdSurchargePerc` /
`tdSurchargeAmt` and `tdNotes`. The Qt screen sends none of them. Port that
behaviour. Card last-4 and auth code are a reasonable later addition for a
card row, but they are not in scope here.

### 4.2 Response shapes the screen reads

```ts
// /open-items → data
{ party:   { ledId, ledName, groupName, isBillByBill, isTdsApplicable, tdsDeducteeType,
             isTcsApplicable, tcsBasis /* RECEIPT | SALES */, tanNo },
  summary: { totalPending, billCount, overdueCount, creditsHeld, pdcHeld },
  bills:   OpenItem[],      // abl_dr_cr DR, in the server's accounts.receipt_bill_sort order
  credits: CreditItem[] }   // CR: ADVANCE | SALES_RETURN | OPENING | JOURNAL
type OpenItem = { billId; billAccYear; billType; docRefno; docDate; dueDate: string | null;
                  billAmount; pendingAmount; status; daysOverdue; pdcHeld; ppdSuggested;
                  usrRefno: string | null; tcsAmount; tcsPending;
                  billProfit: number | null };   // null ≠ 0 (§6.2)
type CreditItem = { billId; billAccYear; billType; docRefno; docDate;
                    billAmount; pendingAmount; srcDocType; srcDocId };

// /party-context → data
{ summary: { totalBalance, totalOutstanding, totalCredits, chequesOutstanding },
  lastReceipts:   { voucherDate, voucherRefno, docAmount, instruments }[],
  pendingCheques: { instrumentNo, instrumentDate, bankName, amount, status }[] }
```

### 4.3 Amounts are positive, and the side is a flag

`ck_av_amount > 0` and `ck_abl_amount > 0`. `abl_dr_cr` and `td_dr_cr` are
spelled `'DR'/'CR'`. (Only the *opening balance* uses `'D'/'C'`. Do not reuse
that helper here.) A minus never travels, and every money input has `min=0`.

### 4.4 The bare-parameter trap (dropdown 54, grid 108)

The configured runner substitutes bare `i`-prefixed tokens as **text**, and
it leaves an **unsupplied** token in the SQL.

- **Dropdown 54** "CUSTOMERS BY AREA" names `iarea_id`. With no beat chosen,
  Qt's filter builder sent `{}`, Postgres received the literal word, and
  choosing *no* beat returned **no** customers ("invalid input syntax for type
  uuid: iarea_id"). **Always send `iarea_id`**, as an empty string when there
  is no beat, and set it before the dropdown first fetches.
- **Grid 108** names `iavh_status`. A register call that omits it is a 400 for
  the whole call. Send `""`. Do **not** model it as a combo filter whose first
  option means "send nothing".

---

## 5. Loading a party

### 5.1 The tender list

`GET /tender-masters/list`, filtered **by the receipt's date**, then with every
ADJUST, CREDIT, LOYALTY and TEMP_CR tender removed (§1.4). Sort by
`displayPosition`, then name.

If the list is empty, or the call fails, **no instrument can be added**. Show
the status chip ("No tender configured" / "Tender master unavailable"). Unlike a
counter, a receipt cannot fall back to a hand-made CASH row: `td_tender_id` and
`td_tender_ledger_id` are what the posting engine turns into the DR leg, and an
invented id would post to a ledger that does not exist.

A fresh receipt starts with **one** instrument row on the default tender, and
that row does **not** mark the document dirty.

### 5.2 Party chosen (beat → customer)

The beat (dropdown 13, `sales.area_master`) only re-filters the customer
dropdown (54, param `iarea_id`). It filters nothing on the screen itself.
"All beats" means an empty string (§4.4).

On a new customer:

1. If anything is allocated, confirm first: *"What is allocated belongs to X's
   bills and will be cleared."*
2. Clear bills and credits, then fire **three requests in parallel**:
   `/open-items`, `/party-context`, and `/account-ledger-masters/get?ledId=`
   (mobile and address, which `open-items` does not carry; cache the result
   per party).
3. **Apply each answer only if it belongs to the current party** (§15.1).

### 5.3 What the answers drive

- **`party` from `/open-items`**: `isBillByBill`, TDS/TCS applicability and
  basis. These decide which role lines are **offered** (§7.3). Keep a
  `loaded: boolean` flag beside it. Every flag on the party defaults to false,
  and false is a claim ("not bill-by-bill", "no TDS"). A POSTED receipt being
  read never calls `/open-items`. In Qt, the hint announced "NOT bill-by-bill"
  over a screen full of that party's bills. **Never read a default-false flag
  as a fact before its fetch** (`loaded-receipt-allocations`).
- **Not bill-by-bill:** say so **once, when the items load**, on the status
  chip: *"…there is nothing to allocate against, and anything received goes on
  account."* Do not raise it from the paint path. In Qt, a notice raised from
  a function that repaints on every keystroke kept overwriting the duplicate
  warning.
- **`/party-context` failing is silent.** Its panels are context, not truth,
  and a receipt can still be taken without them. Empty the two panels and show
  a dash for the totals.

### 5.4 The party band

Group · mobile · address · TDS/TCS (*"TDS 194Q · TCS on RECEIPT"* or "none") ·
Outstanding (Σ pending and bill count) · Overdue (n bills, in red) · Credits
held (green when > 0). Then the three **party-wide** figures from
`party-context.summary`, which cannot be derived from the bills:
**TOTAL BALANCE**, **CHEQUES OUT**, **AFTER THIS RECEIPT**. Under them, a PDC
note when `summary.pdcHeld > 0`: *"PDC held x — those bills do not settle
until maturity."*

**AFTER THIS RECEIPT: do not port the Qt formula.** Qt computes
`totalBalance − Σ bill.settled()`. That subtracts **applied credits**, which
Auto-allocate places inside `receive`. But spending a credit moves nothing
between the party and us: the bill and the credit both shrink, and the net
balance does not change. The Qt formula also **misses** settling deductions
that have no column (TDS, claims), and those **do** reduce what the party
owes. Derive it from the identity instead:

```
afterThis = totalBalance − (received − additions) − Σ settling deductions   // incl. disc / w-off / r-off
```

Show a dash until `party-context` has answered. Check this against one live
receipt with a credit applied and one with TDS before shipping (§17.2).

---

## 6. The bills grid (ui_table 32, "RECEIPT - BILLS")

### 6.1 One list, credits first

`acc_bill_balance` holds both what the party **owes** (`DR`) and what it
**holds** (`CR`). Rev 4 merged the two grids back into one, the way 3.0's
`tableInvoice` was. Since rev 5 the **credits come first**, at the top where
they cannot be scrolled past, because what the party already holds should be
spent before asking for money.

- A credit row is **tinted green the whole way across** and chipped
  **CREDIT / ADVANCE / CR NOTE**, never with the raw `billType`. This matters:
  Deepan's credit is `abl_bill_type = 'OPENING'`, the same word as his nine DR
  bills, and both sides showed "OPENING" with nothing to say which was owed
  and which was held.
- On a credit row, **Receive means Apply**. Disc, W/off and R/off do not open,
  because a settlement discount cannot be given on a credit note.

In React, keep **two lists** in state (`bills`, `credits`) and render them as
one table. Qt needed row-routing helpers (`isBillRow`, `billIndex`,
`creditIndex`) because a single `QTableWidget` held both, and
`billRowForRef()` quietly became a model index rather than a grid row when the
credits moved to the top. Key each row by `billId` and none of that arithmetic
exists.

**Never re-sort the bills.** They arrive in the server's
`accounts.receipt_bill_sort` order (due date or bill date), and Auto-allocate
walks them in list order. That order is what keeps the preview identical to
the post. Column sorting on this grid would be a defect.

### 6.2 Columns (24 in Qt; `useGridColumns(32)` decides what shows)

| key | shown | editable | note |
|---|---|---|---|
| `docDate` | Date | — | |
| `docRefno` | Ref | — | our reference |
| `usrRefno` | Their ref | — | the **customer's** reference; what a remittance advice quotes |
| `billType` | Type | — | chip |
| `dueDate` | Due | — | null for a bill raised with no terms |
| `daysOverdue` | Days | — | |
| `billAmount` | Bill amount | — | "7,490 pending" means one thing on 7,500 and another on 75,000 |
| `paid` | Paid | — | `billAmount − pendingAmount`, derived |
| `pendingAmount` | Pending / Held | — | "Held" on a credit |
| `pdcHeld` | PDC held | — | covered by a cheque that has not matured |
| `tcsPending` | TCS due | — | read-only; TCS is collected, not negotiated |
| `receive` | Receive / Apply | **yes** | |
| `discount` | Disc | **DR rows** | seeded from `ppdSuggested` |
| `writeOff` | W/off | **DR rows** | §6.5 |
| `roundOff` | R/off | **DR rows** | not money (§10.2) |
| `after` | After | — | `pending − (receive + disc + w/off + r/off)` |
| `billProfit` | Profit | — | **blank when null**, never 0.00 (an OPENING bill was typed, not sold) |
| `netProfit` | Net profit | — | `profit − disc − w/off − r/off`, live |
| `note` | Note | yes | a scratch column. **It goes nowhere**: there is no per-bill narration on `acc_bill_adjustment` |
| `billId`, `billAccYear`, `ppdSuggested`, `drCr`, `tcsAmount` | hidden | | |

A zero money cell renders **blank**, not `0.00`. Forty rows of 0.00 hide the
three figures that matter. (In Qt this needed a workaround in `NexTable`. In
React it is one formatter.)

**Adding a column later needs the `ui_tables` layout re-registered.**
`fixed.ui_table_columns` is keyed by column **number**, so inserting a column
mid-grid applies every stored width and heading to the column next to it. The
POST to `/ui-table-masters/create` with `uiTblId: 32` must reuse each
`uiTblClmId`. The column keys are `uiTblClmColumnVisibility` /
`uiTblClmColumnFocus` / `uiTblClmColumnNecessity`, and `uiTblClmNo` is a
**string**.

### 6.3 A typed Receive is an override

Typing into Receive sets `receiveTyped` on that bill. After that,
Auto-allocate **skips** the row and works around the figure the operator
chose, and the row's discount survives with it. Undoing a deliberate figure
is the thing accountants complain about most in the systems they are
migrating from.

### 6.4 The row cap

`receive + disc + w/off + r/off ≤ pending` on a bill. The server locks the
bill `FOR UPDATE` and refuses the excess, and `ck_abl_settled` is the
backstop. When a cell pushes the row over, **give back only the cell just
typed** (the other three were deliberate), clamped to what still fits, and
say *"X has 5,000 pending, and 5,400 has been placed against it."* A credit's
Apply is capped at what the credit holds: *"A credit cannot be spent twice."*

**Qt crashed here, and the lesson carries over.** `AppMessage::showWarning`
runs a nested event loop. An `/open-items` reply landing inside it replaced the
lists, and the reference taken before the dialog was left dangling. In React,
compute the clamp from values inside the reducer, **as one action**, and show
the message as a non-blocking toast. Never `await` a confirm between reading a
row and writing it back.

### 6.5 Write-off above the threshold

`accounts.writeoff_approval_above` (on the session): a write-off above it
needs an approver **name**. It is a recorded decision, not a workflow. Qt
raises a chip telling the operator to put the name in the **narration**.
**Do better in React:** the allocation DTO has a real field,
`writeoffApprovedBy`, which Qt never sends. Show an approver input on the row
(or a small popover) when the threshold is crossed, and send it. Confirm with
Prathap that the service writes it to `abj_approved_by` (§17.3).

### 6.6 Selection gestures

- **Alt+A** runs Auto-allocate (§9). There is also a Clear button.
- **F7** opens the paid detail for the bill under the cursor (§6.7).
- **F5** re-reads `/open-items`, because somebody may have posted against the
  same bills meanwhile.

### 6.7 F7: what has already been paid against this bill

Popup grid **112** "POPUP - BILL PAYMENT HISTORY", parameters `iabj_bill_id`
and `iabj_acc_year`: every receipt, journal and credit note that has touched
the bill. It is the answer to "we paid that in March" while the operator is on
the phone. On a credit row it says *"a credit has no settlement history of its
own"* instead of opening an empty list.

---

## 7. The instruments grid (ui_table 33, "RECEIPT - TENDERS")

### 7.1 One list, two kinds of row

A row is either an **instrument** (cash, card, UPI, cheque, which writes an
`acc_tender_detail` row) or a **role** (which writes one `acc_vouchers` leg
with `av_role`). The Type picker lists the tenders first, then the roles.
**Picking the other kind converts the row**, moving it between `tenders` and
`otherLines`. That is the only way to add a role line, and the reason there is
no second button. This is **presentation only**: the payload still splits
into `tenders[]` and `otherLines[]`, exactly as the server expects.

Keep two lists in the reducer (`tenders`, `otherLines`) and render them as one
table, as with the bills. Converting a row is a single `CONVERT_ROW` action
that removes it from one list and appends it to the other.

### 7.2 Columns

| key | shown | instrument | role |
|---|---|---|---|
| `rowNo` | # | | |
| `type` | Type | tender name | role code (or a picked ledger, see §17.5) |
| `amount` | Amount | `td_amount` | `av_amount` |
| `refNote` | Ref / narration | `td_ref_no` (cheque no., UTR, …) | narration |
| `instrDate` | Instr. date | **cheque only** | — |
| `bankName` | Bank / drawer | **cheque only**, popup grid **111** "POPUP - BANKS" but **free text allowed** (`td_bank_name` is a varchar, not a foreign key) | — |
| `mdr` | MDR | what the bank kept (§7.4) | — |
| `drCr` | Dr/Cr | — | DR / CR |
| `settlesBill` | Settles bill | — | Yes / No |
| `against` | Against | — | optional pin to one bill (§10.3) |
| `pdcVoucher` | PDC / voucher | chip **PDC** before Post, then the voucher number after | **SEEDED** on a server-seeded line |

The kind decides which cells open. Grey out what the kind cannot use, with the
reason in the tooltip, so the operator sees why before clicking.

**Changing a tender drops what the new tender cannot hold.** A non-cheque
loses its instrument date, bank and cheque extras. A tender that gives no
change loses received/change. **Picking a role sets its defaults:** a
credit-side role (TCS_PAYABLE, SURCHARGE_RECOVERED, INTEREST_INCOME) is CR
and settles nothing, BANK_CHARGES is DR and settles nothing, any other role is
DR and settles the bill. The operator can override them.

The Qt grid is pinned to two visible rows. That is a layout choice, not a rule.

### 7.3 Seeded role lines: derived vs offered

`domain/seededLines.ts` rebuilds these after **every** edit that could move
them, and leaves every line the operator added by hand alone:

| role | source | settles | side | operator types it? |
|---|---|---|---|---|
| DISCOUNT_ALLOWED | Σ Disc column | yes | DR | no, **derived** |
| WRITE_OFF | Σ W/off column | yes | DR | no, **derived** |
| ROUND_OFF | Σ R/off column | yes | DR | no, **derived** |
| BANK_CHARGES | Σ MDR on the instruments | **no** | DR | no, **derived** |
| TDS_RECEIVABLE | offered when `party.isTdsApplicable` | yes | DR | **yes**: the amount and the pin survive a rebuild |
| TCS_PAYABLE | offered when TCS applies **and** `tcsBasis === 'RECEIPT'` | no | CR | **yes** |

- **The screen does not calculate TDS.** The certificate the customer holds is
  the fact, and any rate we applied would only be a guess.
- **On the SALES basis, TCS is already inside the bill.** Raising a line here
  would charge the customer twice.
- **A seeded line can be re-amounted but never deleted.** The server seeds it
  again at post and then refuses the receipt as a disagreement. Pressing `-`
  says which figure to change instead.

**The mirrored three never go on the wire (`linesThatTravel`).** They exist
in state because the identity needs them, but the bills grid's columns
already travel inside `allocations[].discount / writeoff / roundoff`. Sending
them as lines as well asks for the same money twice, and the server refuses it
in two different ways depending on `settlesBill` (*"DISCOUNT_ALLOWED cannot
settle a bill"* / *"…Out by 100.00"*). For a while, **any receipt with a
settlement discount was unpostable** because of this. It was verified live
that the post writes the DR Discount Allowed leg by itself.

### 7.4 BANK_CHARGES is on neither side of the identity

A customer pays 15,000 by card and the bank keeps 10. The customer still paid
15,000, and the bill is settled in full. The 10 is a **split of the bank leg**
(bank 14,990, charges 10), not a deduction from the customer. Counting it
would make the identity demand 10 that nobody owes. This is the most common
way to get a receipt screen wrong, so `roles.isLegSplit` is checked in
`identity.ts` and `allocate.ts`, not in a component.

### 7.5 Cheques

The number (`tdRefNo`), the **date written on the cheque** (`tdInstrumentDate`)
and the bank are typed **in the row** and are mandatory there. **F4** opens
the optional extras (branch, IFSC, drawer name, `bankLedgerId`), which go in
`cheque{}`. Nothing in the popup is required, because `SaveReceiptChequeDto`
requires nothing. The popup shows a PDC notice when the date is later than the
receipt date. Every cheque, post-dated or not, gets an `acc_pdc_register` row
server-side. That is for the Received Cheques screen (menu 51) to manage, not
this one.

PDC is computed as **instrument date > receipt date**, compared as dates, not
strings. `tdIsPdc` is never sent.

### 7.6 The duplicate guard

After an Amount is keyed on an **instrument**, call `/duplicate-check` with the
**total** received, not the row. Two 500s keyed as two rows of one receipt are
not a duplicate of anything, and checking per row would raise false alarms on
every split tender. Send `excludeVoucherId` when editing a saved receipt, so
it does not match itself.

- It is a **warning, never a refusal**, shown on the status chip rather than a
  modal, because the operator is mid-entry. It names the receipts that already
  exist.
- **Don't ask twice for the same figure**, but **reset that memo on every new
  or loaded document.** In Qt it was not reset, so the second receipt for the
  same figure was silently skipped, which is exactly the case the guard exists
  for.
- If the check call fails, stay silent. A guard that cannot answer must not
  get in the way.

### 7.7 Instrument-row gestures

- **Alt+T** or `+` adds an instrument on the default tender.
- **Default** resets the current row to the default tender and clears its
  cheque fields (3.0's `tBtnDefTndr`).
- `-` removes the current row, but refuses a seeded line (§7.3).

---

## 8. The identity (`domain/identity.ts`)

```
RECEIVED    Σ tender.amount                    every instrument, a PDC included
DEDUCTIONS  Σ otherLine.amount where settlesBill && !isLegSplit
CREDITS     Σ credit.apply
ALLOCATED   Σ bill.settled()                   receive + disc + w/off + r/off
            + Σ settling deductions that are NOT mirrored by a bill column
ON ACCOUNT  max(0, left − right) computed with onAccount = 0
ADDITIONS   Σ otherLine.amount where !settlesBill && !isLegSplit
difference  left − right;  balances ⇔ |difference| < ½ paisa
```

**A deduction is on BOTH sides.** The customer withheld it, and the bill went
down by it. That is what makes the worked example hold: RECEIVED 70,000 +
DEDUCTIONS 1,400 + CREDITS 4,000 = ALLOCATED 70,750 + ON ACCOUNT 4,650, with
the 1,400 **inside** the 70,750. Qt once counted it on the left only, and
every receipt with TDS was out by the TDS.

**The mirrored three are already inside `bill.settled()`.** Adding them to
ALLOCATED again leaves the strip permanently out by the discount, with nothing
visibly wrong on screen. `isMirroredByBillColumn` must be the **same function**
in `identity.ts` and `allocate.ts`. If the rule is written twice, the two
copies drift.

**Money:** sum in integer **paise** inside `domain/`. Qt uses doubles with a
0.005 tolerance and rounds each placement to paise. Integer paise make both
exact. Convert at the edges (parse and build).

The strip shows the six figures and the difference, green when balanced and
red when not, with a hint phrased as an instruction:

- surplus: *"x received but not placed — allocate it to a bill or leave it on
  account"*;
- deficit: *"x placed but not received — reduce an allocation or add an
  instrument"*;
- balanced with on-account > 0: *"x is not against any bill — it will be kept
  on account as an advance in this party's name."*

**Post is disabled unless** the identity balances, the document is editable
(or being amended), writes are allowed, and a party is chosen. That is not a
nicety: the server recomputes all six figures and refuses a receipt that is
out by one paisa.

---

## 9. Auto-allocate (`domain/allocate.ts`, pure)

This mirrors the server's `allocation-engine.ts`. It exists so the operator
sees **the same split the server will write** before pressing Post. The
server is still the truth.

```
0. clear receive/roundOff on every NON-typed bill; clear every credit.apply
1. CREDITS FIRST — each credit, bills in list order, skip typed rows,
   take = min(creditLeft, room); the credit lands in bill.receive
2. settlingDeductions = Σ settling, non-mirrored, non-leg-split lines   (these take ROOM)
   additions          = Σ non-settling, non-leg-split lines             (these take CASH)
3. cashLeft = available − additions
   roomLeft = Σ room(bill) − settlingDeductions
   walk bills in list order:
     typed row → cashLeft −= bill.receive; continue
     take = min(cashLeft, room(bill), roomLeft); place it
4. onAccount = cashLeft if > ½ paisa
```

`available` is Σ tender amounts. `room(bill) = pending − settled()`, rounded
to paise. **Round at every placement.** Half a paisa left on each of forty
bills is how an identity ends up 0.02 out with nothing visibly wrong.

Why each step has to be this way:

- **A deduction takes ROOM, and an addition takes CASH.** Subtracting a
  deduction from the cash places that cash nowhere: the bill is left short by
  the TDS **and** the strip reports the TDS as a surplus. Placing an addition
  on a bill counts it twice. Both were real bugs in the 2026-09-15 foundation.
- **Credits are spent before cash.** A customer who holds a 4,000 credit note
  and pays 10,000 against a 12,000 bill expects the note to be used.
- **Clear** returns each discount to `ppdSuggested`, not to zero. Clearing an
  allocation is not a decision to refuse the prompt-payment discount.

---

## 10. Save and post

### 10.1 The draft payload (`buildDraftPayload`)

Header + `tenders[]` + `linesThatTravel(otherLines)` + `replace:true`, plus the
**memo** (notes 30):

```ts
allocations:    bills.filter(b => settled(b) > 0)
                     .map(b => ({ billId, billAccYear, amount: b.receive,   // RAW receive
                                  discount: b.discount, writeoff: b.writeOff })),
creditsApplied: credits.filter(c => c.apply > 0).map(({billId, billAccYear, apply}) => …)
```

**The memo sends the RAW `receive`**, not the post-shaped amount. The post
adds settling deductions into `amount` because the posting engine needs them.
A memo has to come back into the same cell it came out of. Otherwise
reopening a draft inflates Receive by a TDS line the operator never typed.

The server stores the memo in `avh_draft_lines`. It writes no adjustment row
and validates nothing, **on purpose**: the client handles staleness (§11.3).

After saving a draft on its own (not on the way to a post): show *"Draft saved
— nothing has been allocated yet"*, then **clear the screen for a new
receipt**. Saving a draft is how a receipt is *put down*. If it stays on
screen, the next receipt gets keyed on top of it, and that is how a draft
acquires somebody else's tenders.

### 10.2 The post payload (`buildPostPayload`)

```ts
allocations[i].amount = bill.receive
                      + (unpinned settling non-mirrored deductions) × bill.receive / Σ receive
// and alongside, NOT inside amount:
  discount, writeoff, roundoff
creditsApplied = credits with apply > 0
otherLineBills = pins, numbered against linesThatTravel()   // 1-based lineNo
onAccount      = identity.onAccount
```

- **`amount` excludes discount, write-off and round-off**, because they travel
  as their own fields. Folding the round-off into `amount` made the server read
  it as cash, and every receipt with a round-off was refused for being out by
  exactly that amount (notes 34).
- The pro-rata share of unpinned deductions is the client's **preview**. The
  server spreads them deterministically itself.
- **Pins are numbered against the filtered list.** `lineNo` is the line's
  position in the array `/create` sent, and the mirrored roles are not in that
  array. Numbered against the full list, a pin points at the wrong line as
  soon as a discount is on the receipt.

### 10.3 The pin (`against`)

A settling deduction with **no** pin spreads pro-rata over the bills this
receipt settles. A **pinned** one lands on exactly one bill. TDS per invoice is
what 26AS matching works on, which is why pins exist at all. The pin does
**not** travel inside the line: it goes in `otherLineBills[]`, because a pin
is a fact about a bill and the line is a fact about a ledger. The Against
picker lists this receipt's bills (DR only), and changing the bill list must
drop any pin that points at a bill no longer listed.

### 10.4 Validation (`validate.ts`), and where it points

Every problem carries **a target**: a header field, or a (grid, row key,
column) cell. The screen **focuses the target before showing the message**,
so dismissing the message lands the operator on the field that needs
changing. Every refusal on an instrument row names the row number.

Before a draft:

- no customer (*"a receipt is money from somebody"*); no date;
- no salesman when `accounts.receipt_salesman_mandatory`;
- per instrument: no tender; amount ≤ 0; a cheque with no number, or **no
  instrument date** (*"that date is what decides whether it is money today or
  a post-dated cheque"*); a tender whose master `needsRef` with no reference
  (use the master's `refLabel`);
- per role line: neither a role nor a ledger; amount ≤ 0.

Before a post, everything above, plus:

- no instrument and no credit → *"a receipt with neither is a journal"*;
- **credit-only** (no instrument) → *"moving a credit from one bill to
  another is a journal, not a receipt"*;
- the identity is out → the hint, with focus on the **first bill that still
  has something pending** (its Receive cell).

### 10.5 The post sequence

```
validateBeforePost
if onAccount > 0: CONFIRM, naming the figure and the party
    "x of this receipt is not against any bill. It will be held on account as an
     advance in P's name…" — No → focus the first unsettled bill's Receive
if dirty || no voucherId: POST /create → take data.header.avhVoucherId
if still no voucherId: refuse locally ("the draft has no id yet")    ← never let /post say it
POST /post
  → number from data.header, pdcVouchers[] from data
  → paint each PDC voucher number on its row (matched by tdRowNo)
  → toast: "Receipt rct00049 posted." + "n post-dated cheque(s) got their own voucher…"
                                     + "x is on account as an advance."
  → CLEAR for the next receipt (the toast has already named the number)
```

On-account money is **confirmed, never assumed**. An advance is a liability,
and it is also the most common way a receipt goes wrong (a missed bill, or
12,000 keyed for 1,200).

**409 from `/post`:** the server computed a different split. Show **both**
sets of figures, the server's message and *"this screen made it x allocated
and y on account"*, and suggest F5. A disagreement the operator cannot see
turns into an argument with support.

**Print:** there is no print pipeline yet (`plan-react-printing.md`). The
button reads "Post & Print", posts, and says printing is not available. Do
not invent a print path.

---

## 11. Opening an existing receipt

### 11.1 Routes in

The register (F8, §14), Ctrl+PgUp/PgDn (§12.2), and **drill-through from
another screen**: Enter on a row of the cheque register (menu 51) opens the
receipt by its four keys. In React this is a route (`/receipts/:company/:branch/:year/:id`),
not a modal factory.

### 11.2 A POSTED (or CANCELLED) receipt: paint what it DID

`/get` → `allocations[]` is the voucher's **adjustment history**, not its
current state. `acc_bill_adjustment` never rewrites or soft-deletes a row. It
retracts by inserting a **negative counter-row** (`bill-adjustment-reverses`).
So after one amend, a receipt that settles two bills comes back holding the
original, its reversal and the replacement. Painted row by row, the same bill
appeared **three times**, with **−2,000 in an editable Receive cell**.

`domain/netAllocations.ts` (port it exactly, with its comments):

1. **Group by `billId`**, keeping the order the server first mentions each
   bill (the order the posting engine walked).
2. **Route by `adjType` into columns**: `ALLOCATION` / `ADVANCE_ADJUST` /
   `NOTE_ADJUST` / `TRANSFER` / empty → Receive; `DISCOUNT` → Disc;
   `WRITEOFF` → W/off; `ROUND_OFF` → R/off. Skip unknown types rather than
   folding them into Receive: a figure in the wrong column is worse than a
   missing one. (The payload has **no** `discount`/`writeoff` fields. Qt once
   read keys that never existed.)
3. **Sum, don't filter on `isReversed`.** A reversal is by constraint the
   exact negative of what it retracts, so summing is exact. Filtering on the
   flag would lose the leftover amount of a *partial* reversal.
4. **Add back only the MATURED part.** `pendingAmount` arrives as it stands
   *now*, already net of this receipt, and the row must show the bill as it
   *stood*. So add this receipt's settlement back, **but only where `matured`
   is not false**. A post-dated cheque did not move `abl_pending_amount`, so
   there is nothing to undo. (`rct00018`: 800 post-dated against a bill
   showing 5,380. A blind add-back paints **6,180**, a figure that bill has
   never carried.) The held amount goes to the PDC held column instead.
   `matured` absent means true.
5. **Drop bills that net to zero.** A correction emptied them, and the receipt
   no longer touches them.

**Reset `party` to not-loaded on every load** (§5.3), or a posted receipt
inherits the previous document's party facts.

The document's status decides whether it is editable. A POSTED or CANCELLED
receipt is read-only unless amending (§13).

### 11.3 A DRAFT: re-read, then clamp the memo onto it (`mergeDraft.ts`)

A draft touched no bill, so `/open-items` is the truth and nothing needs adding
back. Keep the memo (`allocations` and `creditsApplied` from `/get`) aside,
call `/open-items`, then write each remembered figure into its row:

- if the bill can still take it, restore it and mark the row typed;
- otherwise clamp it, **money first**: `receive = min(was, pending)`, then
  discount, then write-off, each limited to what is left. A discount is a
  decision about a balance that may no longer exist;
- if the bill has since **closed** and is no longer listed, drop it;
- clamp remembered credits the same way, matched by the credit's `billId`.
  Either half (bills or credits) may be empty by itself. In Qt, returning
  early on empty bills made an applied credit come back as zero.

**Say so** on the status chip whenever something was clamped or dropped
(*"The draft has moved on — INV/118 has been part-paid since…"*). Silently
entering a smaller number than the operator left is the worst of the options.

---

## 12. Lifecycle verbs

### 12.1 One "get rid of this" button

The document decides the verb:

- **DRAFT → DELETE** (`/delete`). It has no number, wrote no leg and moved no
  bill, so nothing is reversed, and a reversing voucher against it would be a
  lie in the register. Confirm naming the draft (*"this unnumbered draft"*)
  and the party, with *"This cannot be undone."* The server's message names
  what went with it, so show it as it is.
- **POSTED → CANCEL** (`/cancel`), with a **required reason**. The prompt
  names what else goes with it: *"Cancelling rct00049 also reverses 2
  post-dated cheque voucher(s), reopens every bill it settled and returns any
  credit it spent."* Afterwards: *"A reversal voucher was written. Nothing was
  deleted."*
- The server refuses a cancel once a cheque has been **deposited or cleared**,
  or once the on-account advance has been spent. Show its sentence as it is.

Delete and Cancel are never both enabled. After either one, clear the screen
for a new receipt.

**A just-saved draft must enable the delete button.** In Qt, the post-save
handler updated the status pill but not the button, so a freshly saved draft
could not be deleted until it was reopened.

### 12.2 Walking the register: Ctrl+PgUp / Ctrl+PgDn

`/adjacent` with the current four keys plus `direction`. Optionally pass the
same `status` / `fromDate` / `toDate` the register was filtered on, so the
walk follows the list the operator was looking at.

- The **server** decides the order. **The whole key comes back** (company,
  branch, year), so a receipt from another branch or year opens correctly.
  Fall back to the current scope for any key it omits.
- At either end, `data.voucher` is null. That is an **answer**, not a failure:
  *"This is the earliest receipt in the register."*
- **On an empty screen, Prev means "the latest receipt"** (the one keyed a
  minute ago). Next on an empty screen has no such meaning, so say so.
- If dirty, confirm before moving.

### 12.3 The status chip

A **short** notice on the title row, with the full sentence in a
tooltip/popover. It is not a full-width banner: every one of these is rare,
none of them stops the operator, and a strip across the screen pushed the
party panel down every time one fired. Raise notices **where the data
loads, never from a render path**.

---

## 13. Correcting a POSTED receipt (R20, amend)

### 13.1 When it is available

Only when the receipt is POSTED, the company setting
**`accounts.allow_posted_amend`** is on (default **false**; COMPANY scope),
and the user may write. Otherwise the button is **hidden, not greyed**: a
permanently dead button prompts the same question every time.

It **restates the receipt in place**. It keeps its number, because the
customer is holding a slip with that number. `avh_revision_no` goes up by one,
no reversal voucher is written, and every row's before/after is in
`audit.audit_log`. It is not a cancellation, and the confirm prompt says so.

### 13.2 Begin amend: the grid becomes an open list again

A posted receipt shows only the bills it settled. That is right for reading
it and useless for correcting it: the most common correction is "this went
against the wrong bill", and the right bill is not on screen. So:

1. keep the current bill rows aside as the **baseline**;
2. call `/open-items` and `/party-context` again;
3. **`mergeAmend.ts`**: for each baseline bill, add its settlement **back** to
   `pending` and write the figures into the cells (typed). Any baseline bill
   that `/open-items` did **not** list (because this receipt closed it) is
   **appended**, with `pending = what it settled`. Append it at the end rather
   than sorting it in: the server's order is the one the engine walks, and
   this row is one the server did not send;
4. **remove this receipt's own leftover ADVANCE from the credits**
   (`srcDocId === voucherId`). Offering it back is circular, and the server
   recomputes the remainder anyway.

Then the status reads **AMENDING**, the grids unlock, and Post becomes
**Save changes**.

The same add-back happens in §11.2 and §13.2, for the same reason (show the
bill as it stood before this receipt). **Keep both in `domain/`, side by
side**, so they cannot drift.

### 13.3 Save changes (`buildAmendPayload`)

```ts
{ ...buildDraftPayload(...) /* header + tenders + lines */,
  ...buildPostPayload(...)  /* allocations, creditsApplied, otherLineBills, onAccount */,
  avhVoucherId, baseRevision: header.avhRevisionNo, editRemark /* REQUIRED, asked first */ }
```

- **`editRemark` is asked for before sending**, and an empty one cancels. An
  amend that cannot say why it happened is worse evidence than the
  cancel-and-re-enter it replaces.
- **`baseRevision` is mandatory.** The amend carries the **whole** document,
  so last-writer-wins would silently undo somebody else's correction. A stale
  revision is refused and the refusal names it.
- **Amend refuses whatever cancel refuses** (a cleared cheque, a spent advance,
  a non-POSTED voucher), plus a **party change** (409: *"rct00259 was received
  from Deepan and an amend cannot move it to MADHAVAN…"*). Keep the party
  picker **read-only while amending**, so that refusal can never be reached
  from this screen.
- On success: *"Receipt x was restated. It keeps its number…"*, clear
  `amending`, put the Post caption back, and start a new document.
  **Resetting `amending` on every new or loaded document is required.** In
  Qt, missing it left the next receipt being entered on a screen that still
  thought it was restating.

---

## 14. The register (F8)

This is the shared `TxnMainView` descriptor `TxnDocTypes.receipt()`: grid
**108** "MAIN LIST - RECEIPTS", in **picker** mode from the entry screen (the
same way quotation and the sale bill work). Picking a row loads it. Qt used to
keep a second page with its own grid and paging, and that copy was deleted.
In React, reuse the shared transaction list (`txn-main-view-architecture`). Do
**not** build a receipt-specific list.

Grid 108 parameters: `iavh_company_id`, `iavh_branch_id`, `iavh_acc_year`,
**`iavh_status` (always sent, `""` for all)**, and the two dates (§4.4).

Row policy (the same function for the register's keys and the entry screen's
buttons):

| status | edit | F3 |
|---|---|---|
| DRAFT | open to finish | **delete** |
| POSTED | read (amend inside, if §13.1 allows) | **cancel**, reason required |
| CANCELLED | read only | — *"Cancelled — its reversal is already posted."* |

A cancelled receipt is shown struck through, with status badges, as the rest
of the register family does.

---

## 15. Async hygiene, permissions, keys

### 15.1 Replies for a party or document that is no longer current

Qt wrapped every callback in a `QPointer` because `NexDialog` deletes itself
on close. React has no crash here, but the rule still applies in two forms:

- **abort on unmount** (an `AbortController` per request);
- **drop a reply that is no longer current.** Tag `/open-items`,
  `/party-context` and the ledger contact fetch with the `partyId` they were
  requested for, and tag `/get` and `/adjacent` with a load sequence number.
  Ignore any answer whose tag does not match current state. Qt's
  `PartyContact.ledId` did this for one fetch. Do it for all of them.

### 15.2 Permissions

Menu 99's create/edit/delete rights, **resolved before they gate anything**.
An unloaded permission is not a denied one: Qt's `ctx.menuId == 0` escape
hatch existed because the dialog's menu id was assigned after construction,
and treating that as "denied" refused users who had every right. Render
read-only while permissions load, then apply them. The server is the
authority either way.

A screen opened by **drill-through** (from menu 51) still goes through menu
99's permissions in React. There is no reason to copy Qt's "opens readable
because menuId is 0".

### 15.3 Keys (all window-scoped: they work with focus anywhere, including inside a cell editor)

| key | action |
|---|---|
| **Ctrl+Enter** | Post (or Save changes). **Commit any open cell editor first** |
| Enter | belongs to the grid: commit and move on. **No button is a default button** |
| Alt+A | Auto-allocate |
| Alt+T | add an instrument row |
| F4 | cheque extras for the current row |
| F5 | re-read open items |
| F7 | paid detail of the current bill |
| F8 | register picker |
| F9 | ledger statement (**hidden until that screen exists**; not a dead key) |
| F1 | new receipt |
| Ctrl+PgUp / Ctrl+PgDn | previous / next receipt |

The user chose on 2026-09-17 to keep these rather than remap them to 3.0's
F2/F3/F5/F6/F7. F9 was once advertised in the footer and never bound. **The
footer hints and the key map must come from one table.**

---

## 16. Tests

**Pure (Vitest), first.** Take the fixtures from the Qt suites' worked
examples:

- `identity`
  - the flow worked example: 70,000 + 1,400 + 4,000 = 70,750 + 4,650 (the
    TDS inside ALLOCATED);
  - BANK_CHARGES 10 on a 15,000 card receipt changes **neither** side;
  - a discount on the bill column plus its mirrored line is counted **once**;
  - TCS 500 on a RECEIPT-basis party is an ADDITION;
  - paise: 0.1 + 0.2 balances exactly.
- `allocate`
  - credits first: credit 4,000, cash 10,000, bill 12,000 → credit applied
    4,000, cash 8,000, on-account 2,000;
  - TDS 100 on a single 1,000 bill → cash placed 900, **not** 1,000 − 100
    taken off the cash;
  - a typed row is untouched and its cash is consumed;
  - list order is respected, never re-sorted;
  - Clear restores `ppdSuggested`.
- `buildPostPayload`
  - `amount` excludes discount, write-off and **round-off**;
  - the mirrored roles are **absent** from `otherLines`;
  - a pin's `lineNo` counts against the filtered list (a DISCOUNT line before
    a pinned TDS line → `lineNo` 1, not 2);
  - an unpinned TDS is spread pro-rata.
- `buildDraftPayload`: the memo sends raw `receive`; `tdIsPdc` is absent; a
  role line never carries `ledgerId`.
- `netAllocations`
  - original + reversal + replacement → one row;
  - DISCOUNT goes into Disc, not a second row;
  - a partial reversal keeps its leftover amount;
  - an unmatured 800 is **not** added back to pending and goes to PDC held;
  - a bill netting to zero is dropped.
- `mergeDraft`: clamped money first; a closed bill dropped **and reported**;
  a credit-only memo restores its credit.
- `mergeAmend`: pending is added back; a bill this receipt closed is appended;
  the receipt's own advance is removed from credits.
- `validate`: a credit-only post is refused as a journal; every problem
  carries a target.
- `parse`: the `{data:{header}}` envelope for create, post and amend (a
  **shallow read must fail the test**).
- `seededLines`: TDS survives a rebuild with its typed amount and pin; a SALES
  basis party gets **no** TCS line; a seeded line refuses removal.
- `afterThis`: an applied credit does **not** move it; TDS does.

**Live (Playwright)**, porting the four Qt suites (`tests/receipt/run.sh`,
`run-post-new.sh`, `run-features.sh`, `run-full.sh`, about 150 checks). They
**assert against the database, not against the response that wrote it**, and
seed and clean up their own documents:

- cash receipt → posted → legs DR cash / CR party; the bill moves by exactly
  the amount;
- receipt with discount → the DR Discount Allowed leg written by the server;
  bill moves by money + discount;
- TDS pinned to one bill → the adjustment row lands on that bill only;
- post-dated cheque → a second voucher; bill pending **unchanged**; cancel
  reverses **both** ("2 voucher(s) reversed");
- on-account remainder → an ADVANCE CR bill that `/open-items` then lists as a
  credit;
- draft with a memo → reopen → figures restored; settle the bill elsewhere →
  reopen → clamped and reported;
- amend 500 → 1,200 → the bill moves by exactly 700; number kept; revision
  +1; a stale `baseRevision` refused;
- delete draft; cancel with a required reason; walk the register at both ends;
- duplicate check fires on the second receipt of the same total **after a
  new document**.

**Harness lessons that carry over:**

- a required-reason prompt must be **answered**, not just accepted;
- wait for the **result** text ("was restated"), not the **prompt** text
  ("restated in place");
- the tests must enable `allow_posted_amend` themselves, because no test run
  loads the login-time settings;
- `apd_voucher_id` points at the PDC **child** voucher;
- the register's newest row is often an unnumbered DRAFT, so don't assert
  "Prev loaded something" by looking at the refno box.

---

## 17. Backend gaps and open points this screen inherits

1. **The 9 orphaned PDC vouchers** (5,400 against Deepan) that predate the
   PDC-cancel fix. No route reaches them. This is Prathap's call, not the
   client's. Any query hunting orphans must exclude reversals, because
   `avh_against_voucher_id` means both "PDC child of" and "reversal of".
2. **AFTER THIS RECEIPT** (§5.4). The Qt formula is wrong for applied credits
   and for TDS. The React formula must be checked against the live
   `totalBalance` of one receipt of each shape before it ships.
3. **`writeoffApprovedBy`** is on the allocation DTO and Qt never sends it
   (§6.5). Confirm the service writes it to `abj_approved_by`, then use it
   instead of "put the name in the narration".
4. **`/update-header` is not wired in Qt.** The register's POSTED tooltip says
   *"edit only the narration from inside"*, and nothing inside does that. The
   route exists (`UpdateReceiptHeaderDto`: remarks, their/our refs, doc date,
   salesman, plus a **required** `editRemark`). Either wire it as "Edit
   header" on a POSTED receipt when amend is off, or change the tooltip.
   Carrying the false promise over would be a defect.
5. **A free ledger line (no role) cannot be offered in the merged grid.** The
   Type picker lists tenders and roles only. `OtherLine.ledgerId` and dropdown
   43 exist, but nothing in the grid picks a ledger. Either add a ledger
   picker column for a blank role, or keep it out of scope like 3.0 did.
   Decide explicitly.
6. **A cancel produces a reversal voucher in the receipt series.** Cancelling
   `rct00615` produced `rct00617`, POSTED, with no tenders, and it opens as an
   ordinary receipt. We have asked for a marker. Until it exists, the entry
   screen cannot open a reversal read-only by type.
7. **The per-row deposit-account override** (3.0's `payAccChanged`, gated by
   the tender master's `edit_ledger`) is a **blocking** gap in
   `gap-vs-3.0.md` for a company with more than one bank account. It is not
   in the Qt screen, so it is not in this port. Track it there.
8. **A manually applied credit.** Auto-allocate puts an applied credit inside
   a bill's `receive`. An Apply typed straight into a credit row raises
   CREDITS without placing it on any bill, so the identity shows it as a
   surplus (on-account). What the server does with that shape has not been
   tested live. Test it before deciding whether the React screen should place
   a typed Apply onto the bills automatically.
9. **Qt calls `mergeDraftAllocations()` inside `post()`.** It is a no-op once
   the memo has been merged at load, but if Post is pressed before
   `/open-items` answers, it merges onto an empty list and drops the memo. In
   React, disable Post until the reopened draft's open items have loaded.

---

## 18. Phases

This is a single-phase screen by the user's decision (R18). The phases below
are the **order of construction**, not staged releases:

1. `domain/` + `payload/` + `parse.ts`, fully unit-tested against the Qt
   fixtures. No UI.
2. Header, party band, bills grid, instruments grid, identity strip, and a
   cash/UPI/cheque receipt through draft → post. Auto-allocate.
3. Role lines (seeded and manual), pins, PDC chip and number,
   `onAccount` confirmation, 409 handling.
4. Open by key: posted (netting), draft (memo clamp), register picker,
   adjacent, delete, cancel.
5. Amend.
6. Duplicate guard, F7 paid detail, party-context panels.

---

## 19. Do not port

These are **Qt artefacts**, not design:

- **`QPointer` on every callback** → abort + current-tag checks (§15.1).
- **The `menuId == 0` permission escape hatch** → resolve permissions first;
  unloaded ≠ denied (§15.2).
- **Row-routing arithmetic** (`isBillRow`, `billIndex`, `creditIndex`,
  `isTenderRow`, `otherIndex`, `billRowForRef` returning a model index) →
  two lists in state, rows keyed by id.
- **Hidden columns as state** (`Kind`, `TenderId`, `ChequeJson`, `IsSeeded`,
  `DrCr`, `PpdSuggested`) → fields on the row objects.
- **`seedLayout()` and never re-running it on `layoutLoaded`** → render from
  `useGridColumns(32/33)`. The rule that carries over: the screen never
  hard-codes a width.
- **`m_loading` guards** against programmatic cell writes firing edit handlers.
- **Writing an empty `QTableWidgetItem`** to stop Currency painting 0.00 → a
  formatter.
- **`captureBillRow` / `cellNumber` parsing money back out of formatted
  cells** → the draft holds numbers.
- **The pinned two-row tender grid height** and `fitTenderGridToRows`.
- **The delegate-painted focus border, credit tint and chips** → CSS.

These are **defects or near-defects that must not come back**:

- Reading `data.avhVoucherId` / `data.avhVoucherRefno` instead of
  `data.header.*` (§4).
- Omitting `iarea_id` or `iavh_status` from a configured-grid call (§4.4).
- Sending DISCOUNT_ALLOWED / WRITE_OFF / ROUND_OFF as other-lines (§7.3).
- Folding round-off into `allocations[].amount` (§10.2).
- Numbering pins against the unfiltered line list (§10.2).
- Subtracting a deduction from the cash, or placing an addition on a bill
  (§9).
- Counting a deduction on one side of the identity only (§8).
- Painting `/get` allocations row by row, adding back unmatured PDC money, or
  filtering on `isReversed` (§11.2).
- Reading a default-false party flag before `/open-items` has answered
  (§5.3).
- Re-sorting the bills grid (§6.1).
- Holding a row reference across an awaited dialog (§6.4).
- A duplicate-guard memo that survives into the next document (§7.6).
- `amending` surviving into the next document (§13.3).
- A just-saved draft whose delete button stays dead (§12.1).
- Raising a status notice from a render path (§12.3).
- Offering ADJUST / CREDIT / LOYALTY / TEMP_CR as instruments (§5.1).
- Letting the party change during an amend (§13.3).
- The AFTER THIS RECEIPT formula `totalBalance − Σ settled()` (§5.4).
- A footer hint for a key that is not bound (§15.3).
