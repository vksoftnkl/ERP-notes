# Receipt voucher — Qt client plan (menu 99)

**For Viji. Written 2026-09-14; code after `plan-backend-receipt.md` §2 is
run and the nine `/receipts/*` routes answer.** House rules — its own `.ui`
on `NexDialog`, no `NexCrud*`, delegate `cellEdited`, Ctrl+Enter, scope
captured once, doubles not labels — are in
`../openingbal/plan-qt-opening-balance.md` §0 and apply here unchanged. The
screen is `receipt_ui_mockup.png` (rev 3) band for band.

## 1. File set

```
src/modules/accounts/receipt/
  receipt_entry.h / .cpp / .ui           // NexDialog; QStackedWidget pages: Entry, List (F8)
  receipt_bill_delegate.h / .cpp         // BILLS grid: Receive, Disc, W/off, R/off cells; PDC HELD column read-only
  receipt_credit_delegate.h / .cpp       // CREDITS grid: Apply cell only
  receipt_tender_delegate.h / .cpp       // INSTRUMENTS grid: tender combo, amount, ref, instrument date, bank name / drawer
  receipt_other_delegate.h / .cpp        // OTHER LEDGERS grid: role combo | ledger picker, Dr/Cr, amount, settles-bill, narration
  receipt_entity.h                       // ReceiptHeader, OpenBill, CreditItem, TenderRow, OtherLine, column enums, toJson/fromJson
  receipt_allocation_engine.h / .cpp     // PURE mirror of the server's allocation-engine.ts — preview only
  receipt_identity.h                     // the identity as a struct: received, deductions, credits, allocated, onAccount, additions, difference
src/core/api/endpoints/accounts_end_points.h   // ApiEndpoints::Receipts::{OpenItems, PartyContext, Create, Post, Get, UpdateHeader, Cancel}
                                         // NO List (registered grid) and NO Approve/Reject (withdrawn 2026-09-15);
                                         // Get/Post/Cancel all take companyId+branchId+accYear+voucherId
```

`receipt_allocation_engine` is a pure function over `QList<OpenBill>`,
`QList<TenderRow>`, `QList<OtherLine>`, `QList<CreditItem>` → the per-bill
split (money / TDS / claims / credits / discount) and the on-account
remainder. It exists so Auto-allocate and the identity strip show EXACTLY
what the server will post; the server's answer is still the truth and a
disagreement is shown as a 409 with both figures.

## 2. The `.ui` — page "Entry", top to bottom

1. **Header band:** `txtReceiptNo` (read-only, "assigned at Post"),
   `dateReceipt` (`NexDateEdit`), `cmbBeat` (`NexDropdownSingle` id 13 —
   optional; choosing it re-filters the customer list), `cmbCustomer`
   (`NexDropdownSingle` on the "CUSTOMERS BY AREA" dropdown — id to issue —
   with `setFilterParam("iarea_id", …)` when a beat is chosen; **all other
   defaults unchanged** — the user's rule), `cmbLedger` (dropdown 43,
   shown only when "Other party" is ticked), `cmbSalesman` (employee
   dropdown; mandatory per `accounts.receipt_salesman_mandatory`),
   `txtTheirRef` / `dateTheirRef`, `txtNarration`, `txtOurRef`; the status
   pill `lblStatus` (DRAFT / POSTED / CANCELLED — there is no APPROVED).
2. **Party summary line** `lblPartySummary` from `/open-items`'s `summary`
   + `party` (group, bill-by-bill, TDS applicable / deductee type, TCS,
   total pending, overdue count, credits held, PDC held).
3. **Two context panels** (`QTableWidget`, read-only): Last receipts ·
   Pending cheques — from `/party-context`.
4. **BILLS band:** `txtAmount` (`NexDoubleEdit`) · `btnAutoAllocate` (Alt+A)
   · `btnClearAlloc` · `tblBills` (`NexTable`, layout "RECEIPT - BILLS", id
   to issue): # · Bill date · Bill ref · Type chip · Due · Days · Pending ·
   PDC HELD · **Receive** · **Disc** (PPD-seeded, editable) · **W/off** ·
   **R/off** · After · note; hidden `billId`, `billAccYear`, `ppdSuggested`.
   **Two more read-only columns — `TCS` and `TCS pending` — shown only when
   `accounts.tcs_basis = 'SALES'`** (from `tcsAmount` / `tcsPending` on
   `/open-items`, plan §2.13). That is the bill-wise TCS outstanding the
   client asked for; the operator never edits them, and on the RECEIPT basis
   the columns are hidden and the TCS line appears in OTHER LEDGERS instead.
   Rows come from the server in `accounts.receipt_bill_sort` order; the
   engine walks them in list order.
5. **CREDITS band:** `tblCredits` (layout "RECEIPT - CREDITS"): date · ref ·
   type chip · Held · **Apply** · Against bill (combo of the bills above,
   optional) · note; hidden keys.
6. **INSTRUMENTS band:** `tblTenders` (layout "RECEIPT - TENDERS"): # ·
   Tender (combo from `/tender-masters/list`, `ttm_allow_in_return` not
   required here — this is money IN; default tender pre-selected, display
   order) · Amount · Received / Change (CASH only) · Ref (label per
   `refLabel`) · Instrument date (CHEQUE) · Bank / drawer (CHEQUE: bank,
   branch, IFSC, drawer via a small popup `ReceiptChequeDialog` opened from
   the cell — the 3.0 `CustomerReceiptTender` shape) · MDR (BANK/CARD/UPI,
   editable per `tnd_edit_surcharge`) · PDC chip (computed: instrument date
   > receipt date) · Voucher (after post). Alt+T adds a row. The
   `TenderMasterRow` struct from `sale_bill_tender.h` is REUSED (move it to
   `src/shared/txn/tender_master_row.h` first — one small refactor).
7. **OTHER LEDGERS band:** `tblOther` (layout "RECEIPT - OTHER LINES"): # ·
   Role (combo: TDS_RECEIVABLE, BANK_CHARGES, DISCOUNT_ALLOWED, TCS_PAYABLE,
   CLAIMS_ALLOWED, SURCHARGE_RECOVERED, INTEREST_INCOME, ROUND_OFF, WRITE_OFF,
   or "(any ledger)") · Ledger (picker via dropdown 43 when role is blank;
   read-only name when the role resolves) · Dr/Cr · Amount · Settles bill ·
   Against (pin to a bill, optional) · note. Seeded rows (TDS when the party
   is TDS-applicable; TCS when applicable AND `accounts.tcs_basis = RECEIPT`;
   BANK_CHARGES from every MDR; DISCOUNT_ALLOWED = Σ Disc column, read-only)
   are painted with a "seeded" chip and cannot be deleted, only re-amounted
   (the server refuses a disagreement anyway). Alt+L adds a line.
8. **Identity strip:** RECEIVED + DEDUCTIONS + CREDITS APPLIED = ALLOCATED +
   ON ACCOUNT + ADDITIONS · DIFFERENCE (red when ≠ 0) — recomputed on every
   `cellEdited` from `ReceiptIdentity`; **Post is disabled while DIFFERENCE
   ≠ 0**. BANK_CHARGES is excluded on both sides (receipt flow §4.5).
9. **Actions:** `btnSaveDraft` · `btnPost` ("Post &
   Print") — **no Approve, no Reject** (withdrawn 2026-09-15: a DRAFT is
   already the unapproved state, so a gate in front of Post is the same
   decision twice) · `btnEditHeader` (POSTED only: opens `ReceiptHeaderEditDialog`
   — the four fields + edit remark) · `btnCancel` (POSTED only, reason
   required) · `btnLedgerStatement` (F9 — opens the ledger statement view
   when that screen exists; hidden until then) · `btnList` (F8 → page
   "List") · `btnNew` · `btnClose`.
10. **Footer hint bar** exactly the mockup's: Ctrl+Enter posts, plain Enter belongs to the grids, F8 list, Alt+A / Alt+T /
    Alt+L, F9.

**Page "List":** `NexApiTableView` on the **registered receipt grid** (id to issue — there is no `/receipts/list` route) with
From / To (`NexDateEdit` ×2, defaults = the year), status combo (ALL / DRAFT /
APPROVED / POSTED / CANCELLED — APPROVED is the approver's queue), search;
Enter / double-click → `loadReceipt(TxnDocKey)`; F1 → New. PDC vouchers do
not list — they show under their receipt.

## 3. Behaviour — the order things happen

- **New:** scope from `AppSession`; date = app date; the default tender
  row seeded; identity all zero; focus `cmbCustomer` (or `cmbBeat` when the
  branch works by beat — a per-user preference in `QSettings`, not a
  server setting).
- **Party chosen** (`NexDropdownSingle::selected`): `GET /open-items?
  party_id&company_id&on_date` + `GET /party-context` in parallel;
  paint bills / credits / panels / summary; seed TDS / TCS lines from
  `party`; clear allocations. Changing the party after allocations exist
  asks first.
- **Amount typed** → `btnAutoAllocate` (or Alt+A): the engine fills Receive
  due-date-first, `min(amount, pending)` per row, applying credits first
  and PPD discount per `ppdSuggested`; a typed Receive cell is an override
  and the engine skips that row on the next auto run.
- **Every cellEdited** → validate the cell (Receive + Disc + W/off ≤
  pending; Apply ≤ held; amount > 0; a CHEQUE row needs ref + instrument
  date + bank) → recompute `ReceiptIdentity` → repaint the strip and the
  After column.
- **Save draft** (`POST /receipts/create`, `replace: true`): header +
  tenders + other lines; NO allocations (R10 — a draft never touches a
  bill). The answer's `avhVoucherId` is kept; the screen stays on the
  document (unlike opening stock — a receipt is usually posted in the same
  sitting).
- **Post** (Ctrl+Enter / `btnPost`): identity must be 0; save draft first
  if dirty (one gesture, chained callbacks the way `OpeningStockEntry::
  saveDraft(postAfterwards)` chains); then `POST /receipts/post` with
  `allocations[]` (amount = the bill's total settlement EXCLUDING
  discount/write-off — money + TDS + claims + credits, as the engine
  split it), `creditsApplied[]`, `otherLineBills[]` (pins), `onAccount`.
  On 409 show the server's figure beside ours. On success: paint the
  numbers (receipt + each PDC voucher), status POSTED, lock every band
  (`setEntryEnabled(false)`, delegates `setReadOnly(true)`), then print
  (below), then `startNextDocument()` keeping party? No — a receipt's next
  document is another party; keep only the date and salesman.
- **Post from a draft** is the only path: `/create` saves the DRAFT (header +
  tenders + other lines, no allocations), `/post` turns it into a voucher in
  one transaction. Ctrl+Enter on a DRAFT always means Post.
  ~~Approve / Reject~~ — withdrawn 2026-09-15.
- **Load** (`GET /receipts/get?companyId&branchId&accYear&voucherId` — the four keys on all five voucher routes): paints header,
  tenders, legs (a read-only "Legs" tab is worth having — the accountant
  wants to see Dr/Cr), allocations with `isPostDated` / `matured` chips,
  credits applied, cheques, PDC vouchers, the ADVANCE bill; DRAFT →
  editable, APPROVED → editable only for the approver's Reject, POSTED /
  CANCELLED → read-only with Edit header / Cancel.
- **Cancel** (`POST /receipts/cancel`, reason): confirm naming every PDC
  voucher that will be reversed; the server refuses when a cheque is
  DEPOSITED or later or the on-account was used — show its message as is.
- **Print:** the receipt voucher via the printing pipeline — resolve with
  `ApiEndpoints::PrintAssignments::Resolve` (purpose = receipt) and render.
  **There is no print runner in the client yet** (`quotation_entry.cpp`
  TODO); a shared `PrintRunner` (resolve → dataset → render → QPrinter /
  PDF) is a prerequisite item for the whole accounts set and is planned
  once, in `../payment/plan-qt-payment.md` §7, not per screen.

## 4. Settings the client reads (`AppSession::applySettings` gains them)

`accounts.pdc_posting_mode` (display only — the server refuses
ON_CLEARING), `accounts.receipt_bill_sort`, `accounts.receipt_salesman_
mandatory`, `accounts.writeoff_approval_above` (asks for the approver's
name above it — the write-off approver is a recorded name, not a workflow),
`accounts.tcs_basis`,
`accounts.ppd_slabs` (informational — `ppdSuggested` comes from the server).

## 5. Endpoint table

| call | when |
|---|---|
| `GET /receipts/open-items` | party chosen; F5 refresh |
| `GET /receipts/party-context` | party chosen |
| `POST /receipts/create` | Save draft; the first half of Post |
| `POST /receipts/post` | Post |
| `GET /receipts/get` | load from the list |
| ~~`GET /receipts/list`~~ | **withdrawn** — page "List" is a registered grid through `NexApiTableView`, no endpoint |
| `PUT /receipts/update-header` | Edit header dialog |
| `POST /receipts/cancel` | Cancel |
| `GET /tender-masters/list` | once per open (the sale bill's call, reused) |

## 6. Tests

1. Party → bills painted in server order, summary line right; 2. Amount +
Alt+A allocates FIFO with credits first and PPD seeded; 3. typed Receive is
an override; 4. Receive > pending refused in the cell; 5. identity strip
tracks every edit, Post disabled at ≠ 0; 6. CHEQUE row dated later → PDC chip,
and the strip still counts it as RECEIVED; 7. TDS line seeded and undeletable;
8. Save draft payload has no allocations; 9. Post payload amounts = engine
split; 10. 409 shows both figures; 11. POSTED locks every delegate; 12.
Ctrl+Enter on a DRAFT posts (there is no approval path); 13. beat filter re-fetches the
customer dropdown with `iarea_id`; 14. List page loads the grid and Enter
loads the receipt with its own acc year.
