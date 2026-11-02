# Received Cheques (menu 51) — React port plan

Scope: **one screen**, the received-cheque register. It covers the filter bar,
the summary tiles, the register itself (grid 109), the detail strip, the
action bar, the six action panels (Deposit, Clear, Bounce, Re-present,
Replace, Return/Cancel), history, printing the deposit slip, and opening the
receipt a cheque came in on.

There is no document here. The screen has no draft, no save and no number.
Every action is **one server transition** on one row of
`accounts.acc_pdc_register` (or a batch of rows, for Deposit). The port's work
is the **state machine**, the **parameter contract of grid 109**, and making
sure every action sends exactly what its DTO wants.

Qt source: `src/modules/accounts/cheques/`, about 4,000 lines, built
2026-09-17, with the panel moved into a dialog on 2026-09-19:

    received_cheques.{h,cpp,ui}      NexDialog: filters, tiles, register, detail, action bar
    cheque_register_entity.h         ChequeRow, allowedActions(), allowedForAll(),
                                     whyOnlyLooking(), the due bucket
                                     (SHARED with Issued Cheques, menu 52)
    cheque_action_panels.{h,cpp}     one base + six panels: endpoint / payload / validate / confirmation
    cheque_history_dialog.{h,cpp}    /cheques/history
    cheque_row_delegate.{h,cpp}      status pills with operator wording
    src/core/api/endpoints/accounts_end_points.h   ApiEndpoints::Cheques

Companion documents that this plan does **not** repeat:

- `../cheques/received_cheques_flow.md` and
  `../cheques/plan-backend-received-cheques.md`: what every transition writes
  (vouchers, legs, adjustment rows), decisions C1–C10, and the ChqClr and
  ChqBnc voucher types.
- `../cheques/notes-37/38/39-*.md`: the re-present fixes and their regression
  list.
- **`plan-react-receipt.md`**: the receipt that creates every row here, the
  drill-through target (§10), and the bills band the ON_CLEARING Clear will
  eventually reuse (§6.2).
- **`plan-react-dropdown.md`**: dropdowns 25 and 43, and the bare-parameter
  binding rules (§3.2 below restates only what is specific to grid 109).

> **The whole screen in one line:** a cheque is one `acc_pdc_register` row
> that moves HELD → DEPOSITED → CLEARED or BOUNCED (and then re-presented or
> replaced), or HELD → RETURNED / CANCELLED. Each move is **one server call**,
> and money moves only where the server says it does. The client greys
> whatever the state machine forbids, and shows the server's sentence for
> everything else.

---

## 1. What this screen is, and what it is not

1. **A register, not an entry screen.** Cheques are *created* by the receipt
   (menu 99), which writes one register row per cheque tender. This screen
   never creates a cheque, apart from the replacement row that `/replace`
   writes for it.
2. **Deposit moves no money.** Handing paper over a counter posts **no
   voucher**. The bank has taken the cheque into custody, not paid us.
   Crediting the bank at deposit would put every later bank reconciliation
   out. Clearing is the voucher (`ChqClr`: DR bank / CR Cheques in Hand).
3. **Bounce is one voucher with five legs** (`ChqBnc`): DR party (amount +
   party charge), CR Cheques in Hand, CR BOUNCE_CHARGES_RECOVERED, DR
   BANK_CHARGES, CR bank. It also writes negative adjustment rows for
   everything the cheque settled, reverses any advance the cheque funded (the
   C4 cascade), and raises a JOURNAL bill `BNC/<no>` for the party charge.
   **The client sends four numbers and a reason. It computes none of this.**
4. **Re-present is not "undo the bounce"** (C5). Between the bounce and now,
   the party genuinely owed the money. So re-present takes the cheque in
   again: a re-issue receipt voucher (DR Cheques in Hand / CR party), with the
   **originating bills' split restored by the server** (§6.4).
5. **Nothing is edited or deleted.** Every move appends to `txn_status_log`.
   A mistake is corrected by the next move, never by rewriting one.
6. **Shared with Issued Cheques (menu 52).** It is the same table read with
   `apd_tra_type = 'P'`. Only the words on the buttons change: we *deposit* a
   received cheque and *present* an issued one. Build the state machine and
   the row type **free of anything that knows which screen is asking**, and
   let the screen pass in a vocabulary object.

---

## 2. Module layout

```
src/features/cheques/                 // shared by menu 51 and menu 52
  domain/                             // no React import, no API import
    chequeRow.ts                      // ChequeRow + fromGridRow (§4.1)
    stateMachine.ts                   // allowedActions / allowedForAll / whyOnlyLooking (§5)
    bucket.ts                         // FUTURE / DUE_TODAY / OVERDUE / STALE → label + tone
    vocabulary.ts                     // received vs issued wording
  api/
    cheques.ts                        // summary, get, history, deposit, clear, bounce,
                                      // represent, replace, return, depositSlip
  actions/                            // one file per verb — the Qt panel contract, as data
    types.ts                          // ActionSpec<TForm> { verb, endpoint, initial, build, validate, confirm }
    deposit.ts  clear.ts  bounce.ts  represent.ts  replace.ts  returnCancel.ts
  received/
    ReceivedChequesScreen.tsx         // menu 51
    components/
      FilterBar.tsx                   // status ticks, dates, bank, party, search
      SummaryTiles.tsx                // in hand / with bank / bounced / cleared / Cheques In Hand
      Register.tsx                    // grid 109, with row checkboxes
      DetailStrip.tsx                 // /cheques/get, for the current row
      ActionBar.tsx                   // the verbs + the "why not" line
      ActionDialog.tsx                // hosts whichever ActionSpec is open
      HistoryDialog.tsx
      DepositSlipDialog.tsx
```

The Qt design already had the right idea. Each panel answers the same three
questions (where the request goes, what it sends, why it cannot go yet), so
the screen never needs to know which panel is showing. In React that becomes
a plain **`ActionSpec` object per verb**, with no class hierarchy:

```ts
type ActionSpec<F> = {
  verb: (rows: ChequeRow[], form: F) => string;          // "Deposit 3 cheques", "Cancel cheque"
  endpoint: string;
  initial: (rows: ChequeRow[], today: Date) => F;        // fresh EVERY time it opens (§6)
  build: (rows: ChequeRow[], form: F) => object;         // the POST body
  validate: (rows: ChequeRow[], form: F, today: Date) => string | null;
  confirm: (rows: ChequeRow[], form: F) => string;       // names the money and the effect
  Fields: React.FC<{ form: F; set: (p: Partial<F>) => void; rows: ChequeRow[]; detail?: ChequeDetail }>;
};
```

A seventh verb is then one new file, with nothing in the screen to change.
`build` and `validate` are pure and tested without a DOM (§12).

---

## 3. The register: grid 109, and its parameter contract

### 3.1 Why a configured grid and not `/cheques/list`

The rows come from grid **109** "MAIN LIST - RECEIVED CHEQUES", **not** from
`GET /cheques/list`, even though that route returns the same rows. Column
widths, visibility and the operator's own saved layout live in
`fixed.grid_columns`, and a screen that paints rows from a REST list throws
all of that away.

`/cheques/list` **is** called once per refresh, **for its `summary` only**
(§7). Send `limit=1` and discard the row.

### 3.2 Nine bare parameters, ALL sent, EVERY time

Grid 109's SQL names nine bare identifiers, and PostgreSQL resolves every one
of them when the query is prepared. **Leave one out and the whole call fails
with 400 `column "istatus" does not exist`**. It is not a filter quietly
ignored. The SQL guards each parameter with `NULLIF(…,'') IS NULL`, so an
empty string means "no filter".

| param | from | empty means |
|---|---|---|
| `iapd_company_id` | session scope | — (never empty) |
| `iapd_branch_id` | session scope | — |
| `iapd_acc_year` | session scope | — |
| `istatus` | status ticks, **comma-joined** (`HELD,DEPOSITED`) | every status |
| `ifrom`, `ito` | date range (ISO) | open-ended |
| `ibank_ledger_id` | bank filter (dropdown **25** "BANK LEDGERS") | any bank |
| `iparty_id` | party filter (dropdown **43** "LEDGERS - ALL") | any party |
| `isearch` | the search box | no search |

Build the whole object in one pure function (`gridParams(filters, scope)`) and
unit-test that **all nine keys are always present** (§12).

**The search box sets `isearch` INSIDE `grid_param`.** It must not use the
generic `search` query key that the shared table component lifts out on its
own. Verified live on 2026-09-17: `search=55492` with no `isearch` returned
**zero rows and no meta**. That is the worst kind of failure, because it looks
like an empty register. If the shared React grid component has a built-in
search feature, **turn it off for this grid** and wire the box to `isearch`
yourself.

Debounce the search at about 300 ms, and search immediately on Enter. A
register of four thousand cheques answers in its own time, and eight searches
in flight race each other to paint the grid.

### 3.3 Status ticks

The ticks are **Held · Deposited · Bounced · Cleared · Returned · Replaced**.
"Returned" also sends **CANCELLED**, because the two are one idea to an
operator ("the paper left the register"). The default is **Held + Deposited**
(what is still in play). **Nothing ticked means every status, not no rows.**
An operator who unticks the last box has cleared the filter, and an empty
grid would look broken rather than filtered.

**Dates default to the whole year.** An accountant hunting "that cheque from
Mr Rao" has no idea which month it was in. Qt uses **1 January of the app
date's year up to one year ahead**, which is a calendar year. **Use the
accounting year instead** (the start of `accYear` to its end, plus a margin
for post-dated cheques). A cheque received in March 2027 belongs to
2026-2027, and Qt's default hides it in January 2027. "Clear filters"
restores the defaults.

### 3.4 Columns and pills

The layout (`useGridColumns(109)`) decides what shows. The row keys the
screen **reads** come from the row object, **never from a visible cell**:
`apd_id`, `apd_acc_year`, `apd_company_id` and `apd_branch_id` are hidden
columns, and a screen that reads a key off a cell breaks the moment an
operator hides the column.

**Money arrives as a STRING** (`"1000.00"`) and dates as ISO strings.
`fromGridRow` parses both. Qt learnt that reading a string-typed number as a
number gives 0, and a register that silently totals to nothing looks fine.

Status pills use **operator words** while the colour still keys off the
**raw** value:

| raw | pill | tone |
|---|---|---|
| HELD | In hand | amber |
| DEPOSITED | With bank | blue |
| BOUNCED | Bounced | **red, NOT dimmed or struck through** |
| CLEARED | Cleared | green |
| RETURNED / CANCELLED | Returned / Cancelled | grey, dimmed |
| REPLACED | Replaced | grey |

Two defaults of the shared badge palette were wrong here and had to be
overridden. BOUNCED fell into the palette's "danger" bucket, which dims **and
strikes through** the row, and that is wrong for the row that most needs
action on the screen (red yes, dead no). DEPOSITED and REPLACED were in no
bucket and got arbitrary hashed colours. Use the palette, **with these
overrides**.

The due bucket (`due_bucket`, computed by the grid and NULL once a cheque has
left play) shows as **Post-dated / Due today / Matured / Stale**. STALE means
more than three months old, which a bank will refuse.

### 3.5 Paging

Pages come from the grid runner, with numbered page buttons and "showing X–Y
of Z". See §4.2 for what happens to ticked rows across pages.

---

## 4. Rows and selection

### 4.1 `ChequeRow`

```ts
type ChequeRow = {
  apdId: string; accYear: string; companyId: string; branchId: string;   // the KEY
  instrumentNo: string; instrumentType: string; instrumentDate: string | null;
  amount: number; bankName: string; drawerName: string;
  partyName: string; status: Status; bucket: Bucket | null; postingMode: 'ON_RECEIPT' | 'ON_CLEARING';
  depositDate: string | null; depositSlipNo: string; depositBankName: string;
  presentCount: number; clearDate: string | null; bounceDate: string | null;
  bounceReason: string; bounceCharges: number;
  receiptRefno: string; clearRefno: string; bounceRefno: string; replacedByNo: string;
  remarks: string;
};
```

**`accYear` is the year the cheque was RECEIVED**, the partition key, not the
year it clears in. **Every route takes the row's own four keys** (`apdId`,
`apdAccYear`, `apdCompanyId`, `apdBranchId`), taken from the **row, never the
session**. `/get` answers a correct id under the wrong branch with a 404,
which is scoping working, not a missing row. A register showing another
year's rows while the screen posts this year's scope is the bug this
prevents.

`describe(row)` → `"55492 — Deepan — 1,000.00"`. Every confirmation names
rows this way.

### 4.2 What is being acted on: ticked rows beat the current row

Deposit is the only **bulk** verb, because a deposit slip is a piece of paper
listing several cheques. Every other verb is single-row. So there is ONE
notion of the target set:

> **the ticked rows if any are ticked, otherwise the current row.**

**Keep the ticked set keyed by `apdId`, not by row index**, and make it
survive paging, so a slip can span two pages. That is an improvement on Qt,
where ticks lived on the page. Two rules keep it safe:

- **Clear the ticks whenever the filters change.** Otherwise a ticked row
  that a filter has hidden would be deposited without being seen.
- The action dialog **lists every targeted row by name** (§6), so nothing is
  acted on unseen. The Qt confirmation stops listing names above six rows. In
  React, show them all in a scrollable list.

---

## 5. The state machine (`domain/stateMachine.ts`)

This is the reason the entity file exists. The **server owns the rule**:
every route re-checks it and answers 409. The client greys the buttons,
because offering a button that is guaranteed to fail is worse than not
offering it.

| status | meaning | allowed |
|---|---|---|
| HELD | the paper is in our drawer | Deposit · Replace · Return/Cancel · History |
| DEPOSITED | with the bank, outcome unknown | Clear · Bounce · Print slip · History |
| BOUNCED | it came back | Re-present · Replace · Print slip · History |
| CLEARED | the money is ours | Print slip · History |
| RETURNED / CANCELLED / REPLACED | finished | History |
| anything else | a status a later migration adds | **History only** (safe, not permissive) |

- **History is always allowed**, even on terminal rows, but only for **one**
  row, because a history of several cheques at once means nothing.
- **Print slip** is offered wherever a slip number can exist, which is every
  status after a deposit. Reprinting a cleared cheque's slip is routine.
- **`allowedForAll(statuses)` is the intersection.** Two deposited cheques and
  one held share nothing but History, and that is correct: a bulk deposit
  that silently skipped the deposited ones would leave the operator believing
  all three went to the bank. An empty target set allows **nothing**, not
  even History.

### 5.1 Say WHY the bar is dead

Seven greyed buttons are not a message. The action bar carries one line (the
selection description, or the reason), in amber when it is a reason. In order
of precedence:

1. **Permission:** *"read-only — your login may not act on cheques"* (or *"…may
   not print"* when only Print slip is blocked by rights).
2. **A finished cheque** (only History, or History + Print slip, is allowed):
   `whyOnlyLooking(status)`:
   - CANCELLED → *"cancelled with its receipt — nothing further happens to
     it"*
   - RETURNED → *"returned to the party…"*
   - REPLACED → *"replaced by another cheque, which carries it now"*
   - CLEARED → *"cleared and settled — only its slip can be reprinted"*
3. **A mixed tick** that shares no verb: *"these are in different states and
   share no action — tick rows that are all at the same stage."*

It is worth being loud about permissions. `/user-administration/create`
**fully replaces** a user's menu array, so a save made to grant one menu can
silently revoke this one (`user-menu-full-replace`). The next person to open
the screen then finds it inert with nothing saying why. Without the line, the
only way to find out was to read `user_menus` on the server.

### 5.2 A known hole the client must NOT predict

A replacement row is created with `apd_tender_id` NULL (plan §4.6), but
`/return` and `/replace` resolve Cheques in Hand **from the tender row** and
refuse a NULL one. So a replacement cheque is HELD, the map says it can be
returned or replaced, and the server says it cannot (§14.2). **Do not add
logic that guesses which HELD rows are replacements.** Let the call go and
show the server's sentence. Guessing would build a bug into the client that
outlives the server fix.

---

## 6. The action dialog and the six verbs

It opens as a **modal dialog** over the register, not inline under it. The
inline version took the height of the tallest panel from the register just
when the operator wanted to re-read the rows they were about to act on. The
dialog:

- is **re-initialised from `initial(rows, today)` every time it opens.** A
  panel that kept last week's deposit date because it had been opened before
  is how a slip gets back-dated by accident;
- shows the verb as its title and on the OK button (*"Deposit 3 cheques"*,
  never "OK");
- runs `validate` live and shows the reason **under the fields**, with OK
  disabled until it is empty;
- on OK: a confirmation naming the money and the effect (`confirm`) plus
  **every targeted row by name**, then the POST;
- closes on Esc (Esc closes the dialog first, and the screen only when no
  dialog is open). A half-typed bounce must not be one keystroke away from
  taking the whole register with it;
- on error: stays open with the **server's own sentence**. The messages are
  specific (*"55491 is CLEARED — it is settled and nothing further happens to
  it"*), and nothing composed on the client would be better.

**Dates are the session's app date, never `new Date()`.** A branch keying
yesterday's collections at 1 a.m. is routine. The server compares against
*its* today, so the two can disagree by a day near midnight. Client
validation catches the obvious cases. It is not the authority.

**Client validation covers only what can be checked without asking the
server:** dates against today and against the cheque, required fields, and
amounts. Anything that needs the ledger or the bill balances comes back as the
server's sentence.

### 6.1 Deposit (bulk) → `POST /cheques/deposit`

```ts
{ cheques: rows.map(r => ({ apdId: r.apdId, apdAccYear: r.accYear })),
  apdCompanyId, apdBranchId,          // HOISTED: a slip belongs to one branch
  bankLedgerId, depositDate, slipNo, remarks? }
```

Fields: bank (dropdown 25), date (default today), slip number, remarks.

Validate:

- at least one row;
- a bank is chosen;
- a date is set, and it is not in the future;
- **a slip number** (*"the bank's slip number is what the printed slip is
  keyed on"*);
- **not earlier than any cheque's own date** (*"Cheque 55492 is dated …"*).
  Banking a cheque early is how one comes back marked "post-dated presented
  early".

It is **all or nothing** server-side: every row must be HELD. The company and
branch are taken from the first row. **Also check on the client that every
target row shares that company and branch**, and refuse a mixed set by name.
The DTO has room for one scope only, and a mixed set would otherwise surface
as a vaguer server error.

After success, **keep `(bankLedgerId, depositDate, slipNo)`** for the slip,
then ask *"Print the deposit slip now?"* (§9).

### 6.2 Clear (single) → `POST /cheques/clear`

```ts
{ ...key, clearDate, bankDate?, remarks? }       // allocations ABSENT (see below)
```

- `clearDate` is the day the money became ours. `bankDate` is the date **the
  bank** says it happened (it goes to `av_recon_date` on the bank leg) and
  defaults to `clearDate`, following it until the operator edits it.
- Validate: one row; a date that is not in the future and **not before the
  deposit date** (`ck_apd_seq`).
- **Posting mode note** in the dialog:
  - ON_RECEIPT: *"the bills were settled when the cheque was taken. Clearing
    moves the money from Cheques In Hand to the bank."*
  - ON_CLEARING: *"clearing it settles the party's bills now — oldest
    first."*

**`allocations` is deliberately left out, not sent as `[]`.** Under
ON_RECEIPT the server refuses a non-empty one outright. Under ON_CLEARING,
leaving it out asks for auto-FIFO. A **bills band for ON_CLEARING**, the
receipt's bills grid reused (`plan-react-receipt.md` §6), is owed. Qt could
not extract it. In React, if the receipt's `BillsGrid` is built as a
component, mounting it here is the natural way to close that gap
(`ChequeAllocationDto` = billId, billAccYear, amount **excluding**
discount/write-off, discount, writeoff, `writeoffApprovedBy`).
`accounts.pdc_posting_mode` defaults to ON_RECEIPT, and ON_CLEARING is not
in the receipt plan, so this gap is inactive until that mode is switched on.

### 6.3 Bounce (single) → `POST /cheques/bounce`

```ts
{ ...key, bounceDate, reason, reasonText?, bankCharge, partyCharge }
```

**The field names are not what the column names suggest.** It is `reason`,
`bankCharge` and `partyCharge`, **not** `bounceReason` / `bounceCharges`. A
wrong name is a 400, and if the caller does not check the response, the
cheque silently stays DEPOSITED, which reads exactly like a product bug.
`bankCharge` and `partyCharge` are **required** (send 0, not absent).

- **The two charges are different numbers.** `bankCharge` is what the bank
  took off **us**: an expense (DR BANK_CHARGES / CR bank). `partyCharge` is
  what **we** charge the party: income (DR party / CR
  BOUNCE_CHARGES_RECOVERED), and it raises a JOURNAL bill they now owe. Either
  one at 0 skips its legs. Label them *"Bank charged us"* and *"We charge the
  party"*.
- **Reason:** an editable combo. The list is a **pre-fill, not a whitelist**,
  because banks return cheques for reasons no list anticipates. Qt
  **hard-codes** seven reasons. The setting `accounts.bounce_reasons` exists
  for exactly this. **Read the list from the setting** and fall back to the
  hard-coded seven. `reasonText` holds *"what the bank actually wrote"*.
- Validate: one row; a date that is not in the future and not before the
  deposit date; a non-blank reason (`ck_apd_bounced`); charges ≥ 0.
- **Preview:** *"These reopen when it bounces: INV/118 — 1,000.00 of 4,500.00
  still open"*, from the detail (§8), or *"No bill was settled by this cheque
  — nothing reopens."* It is information only. The server recomputes.
- Confirm: *"55492 bounces: the bills it settled reopen, the bank's 150.00
  goes to bank charges, the party is billed 300.00."*

### 6.4 Re-present (single) → `POST /cheques/re-present`

```ts
{ ...key, bankLedgerId, depositDate, slipNo, remarks? }     // allocations ABSENT
```

The fields are the same as Deposit, and the bank **need not be the one it
bounced from**. Validate as for Deposit, for one row.

**Send no `allocations`, and that is correct, not lazy.** Since the notes-37
fix, the server **restores the originating bills' split itself**, including a
cheque split 1,500/1,000 over two bills. That split exists only in the
reversed adjustment rows, and **the client cannot see it**: after a bounce,
`/cheques/get` reports `settledByThisCheque: 0` on every bill, correctly,
because those rows are reversed. Anything the client sent would be a guess
overriding a server that knows. The DTO's `allocations` is an OVERRIDE "for
the rare case where the operator really is re-pointing". Leave it unbuilt
until someone asks for it.

If the originating bill has been paid off since the bounce, the server answers
**409 naming the bill and the figures**, and the cheque stays BOUNCED with
zero rows written. Show the sentence as it is.

### 6.5 Replace (single) → `POST /cheques/replace`

```ts
{ ...key, reason?,                                   // see below
  newCheque: { instrumentNo, instrumentDate, amount,
               bankName?, bankBranch?, ifsc?, micr?, drawerName? } }
```

- **The new cheque is a NESTED `newCheque` object**, not flat fields.
- **Optional strings are sent only when filled.** An empty string on a
  nullable column is not NULL, and a blank IFSC stored as `""` looks like a
  value was entered.
- **Seed from the old cheque:** amount, bank name, drawer (a party replacing a
  cheque usually draws on the same account). Clear the number, the MICR,
  branch and IFSC, and set the date to today. **The amount need not match**:
  replacing a bounced cheque with a smaller one and paying the rest another
  way is ordinary.
- Validate: one row; a number; a date **within three months back and one year
  ahead** (`ck_apd_dates`, because the new row is received today; older than
  three months is stale and no bank will take it); amount > 0. Max lengths:
  number 30, bank 100, branch 100, IFSC 11, MICR 9, drawer 150.
- **From HELD, send a `reason`.** The DTO says it is *"required in practice
  when replacing from HELD, because that path RETURNS the old cheque first"*,
  and the RETURNED status refuses a row with no reason. **The Qt panel has no
  reason field and never sends one.** Show a reason input when the source row
  is HELD, and require it. Test the Qt behaviour live (§14.3). It is probably
  refused.
- The note in the dialog: *"Replaces 55492 — 1,000.00. The old row becomes
  REPLACED and the two stay linked both ways."*
- `allocations` (ON_RECEIPT, auto-FIFO when empty) is not sent, the same as Qt.

### 6.6 Return / Cancel (single, HELD only) → `POST /cheques/return`

```ts
{ ...key, action: 'RETURNED' | 'CANCELLED', reason, remarks? }
```

- There is **no `returnDate`**. Do not add one.
- Two choices: **Returned** (*"the party has the paper back"*) and
  **Cancelled** (*"it is void and nobody has it"*). The verb changes with the
  choice: "Return" or "Cancel cheque".
- A **reason is required** (`ck_apd_cancelled`), max 250 characters. Remarks
  max 500.
- Hint: *"Cancelling releases the cheque number — the same one can be entered
  again."* The unique index on the instrument number excludes CANCELLED rows,
  which is the reason to prefer Cancel over Return for a mis-keyed cheque.
- Confirm: *"…goes back to the party. The receipt's settlement is reversed."*
  or *"…is voided. The receipt's settlement is reversed and the cheque number
  is released."*
- It is refused once DEPOSITED, because the bank's record and ours have to
  agree.

### 6.7 After any successful action

- Show the server's `message` in the status line.
- Close the dialog.
- **Clear the ticks**.
- **Reload the register from scratch** (not a refresh that restores the
  selection by index). An action can move a row out of the current filter
  entirely (a deposited cheque leaves a Held-only register), and restoring the
  selection by index would land on a different cheque.
- Refresh the summary tiles.

---

## 7. The summary tiles

`GET /cheques/list?apdCompanyId&apdBranchId&apdAccYear&limit=1` → `data.summary`:
`inHandCount/Amount`, `withBankCount/Amount`, `bouncedCount/Amount`,
`clearedCount/Amount`.

The summary covers the **whole register, whatever the filters say**, which is
what the tiles want: "what do we hold", not "what is on screen". Say so in the
tooltip, so nobody reports it as a filter bug.

There are four coloured tiles (amber **in hand**, blue **with the bank**, red
**bounced**, green **cleared**), each with the figure large and the count
small underneath. A fifth, slate tile shows **CHEQUES IN HAND = in hand + with
the bank**. That is the figure the Cheques In Hand ledger should stand at. If
the ledger differs, a posting went astray.

**Qt only shows the sum and never compares it.** It does not fetch the
ledger's balance, so the tie is displayed, not checked. The ledger id is on
`/cheques/get` (`chequesInHandLedgerId`), and a ledger-balance route does not
exist yet (the ledger statement is unbuilt). Show the sum as Qt does, and
record the real tie as owed (§14.5). **Do not fake it** by summing
`acc_vouchers` on the client.

If the summary fails, the tiles show a dash and *"summary unavailable"*, never
stale figures next to a freshly loaded register.

---

## 8. The detail strip (`GET /cheques/get`)

Load it for the **current** row. One call feeds both the strip and the bounce
preview. The strip shows:

- *"came in on rct00049"*, *"cleared by chqclr00012"*, *"bounced on
  chqbnc00007"*, *"bounce charge billed as BNC/7"*;
- *"settled INV/118 — 1,000.00 of 4,500.00 still open; …"* (only bills with
  `settledByThisCheque > 0`), or *"settled no bill"*;
- the **replacement chain**: *"replaces cheque 55401"* / *"replaced by cheque
  55610"*. That link is what makes a REPLACED row something other than a dead
  end.

**Drop a reply for a row the operator has already moved off** (tag the request
with `apdId` and compare it against the current row). Otherwise one cheque's
detail gets painted under another's. On failure the strip shows *"could not
load this cheque's detail"*.

**Re-derive the detail after every reload.** Qt had a bug where a reload left
the current index where it was, emitted no change, and left the strip blank.
The strip is the only place the bills a cheque settled are named, so an
operator about to bounce one was looking at an empty box. In React, derive
"current row" from the key rather than from an index-change event, and fetch
whenever the **key** of the current row, or the data under it, changes.

`/cheques/get` also returns `cheque.apdBankLedgerId` (the bank it was actually
deposited into). §9 uses it.

---

## 9. Deposit slip (F6)

`GET /cheques/deposit-slip?apdCompanyId&apdBranchId&bankLedgerId&depositDate&slipNo`,
keyed by **(company, bank ledger, date, slip number)**, not by an id.

- **Which slip:** the deposit just made (the three values kept in §6.1), or
  otherwise the current row's own slip.
- **The bank id problem, solved better than Qt.** The grid row carries the
  bank's **name**, not its ledger id. Qt therefore asks the operator to pick
  the bank in the filter bar first. In React, take `cheque.apdBankLedgerId`
  from the `/cheques/get` detail already loaded for the row. That is the bank
  it actually went into, since deposit overwrites that column. Fall back to
  the filter bar only if the detail is missing.
- **A slip that matches nothing returns 200 with no cheques, not 404**
  (§14.4). **Check the count before printing.** Otherwise F6 prints a blank
  slip with a real bank's name across the top.
- **Printing waits for the print pipeline** (`plan-react-printing.md`; print
  purpose `CHEQUE_DEPOSIT_SLIP` is seeded). Until then, show the slip's own
  dataset in a dialog (bank, the numbered cheques, count and total) and say
  that the printed form arrives with the printing module. Do not build a
  one-off print path.

---

## 10. Drill-through: Enter opens the receipt

Enter on a row opens the **receipt the cheque came in on**
(`plan-react-receipt.md` §11). The grid row has the receipt's **number** but
not its voucher id, so the id comes from `/cheques/get` → `receiptVoucher`.

**Open it with the receipt's own scope.** Qt builds the key from the
**session's** company and branch plus the voucher's year. That is wrong for a
cheque written under another branch, and it contradicts Qt's own comment ("in
that receipt's OWN scope"). Use `receiptVoucher.companyId/branchId` if `/get`
returns them. Otherwise use **the cheque row's** `companyId/branchId` (a
cheque's receipt is written in the same scope as the cheque). **Never use the
session's.**

If there is no `receiptVoucher`: *"This cheque is not linked to a receipt. A
row written by a replacement carries its own re-issue voucher instead."*

In React this is navigation to the receipt route, not a modal factory.

---

## 11. History (Ctrl+H)

`GET /cheques/history` → `entries[]` of `{ seqNo, event, fromStatus,
toStatus, changedOn, changedBy, remarks }`, **newest first**, shown as a table.

**An empty list is correct for a cheque that has only been received.** The
log starts at the first status change, so empty means "nothing has happened
yet", not "the log is missing". Say that in the empty state.

The log is append-only, so a cheque that was bounced and then re-presented
shows both events.

---

## 12. Permissions, async, keys

**Permissions.** Menu 51. `canEdit` gates the six writing verbs, `canPrint`
gates Print slip, and **History is never gated** (it only reads). An unloaded
permission is **not** a denied one. Qt's `ctx.menuId == 0` escape hatch
existed because the dialog's menu id was assigned after construction. In
React, resolve permissions before gating, render read-only while they load,
and remember that the server is the authority.

**Async.** Abort requests on unmount, and tag every request (detail by
`apdId`, register by filter generation, summary by scope) so a late reply for
something no longer current is dropped. The Qt `QPointer` guards existed
because of `WA_DeleteOnClose`. The rule behind them carries over.

**Scope** (company, branch, year) is captured once, when the screen opens. A
branch switched in another window mid-session must not make this screen's
next call disagree with the rows it is showing. The **actions**, however, use
each row's own keys (§4.1).

**Keys** (window-wide; no button is a default button, and Enter on a row
opens the receipt):

| key | action |
|---|---|
| F5 | Deposit |
| F7 | Clear |
| F8 | Bounce |
| F3 | Return / Cancel |
| F6 | Print slip |
| Ctrl+H | History |
| Enter | open the receipt |
| Esc | close the dialog; with none open, close the screen |

Re-present and Replace have no key in Qt, and the port keeps it that way. A
key fires only if its button is enabled. **Generate the footer hints and the
key map from one table.**

---

## 13. Tests

**Pure (Vitest):**

- `gridParams`:
  - **all nine keys present in every combination** (no ticks, no dates, no
    bank, no party, empty search);
  - `istatus` comma-joined;
  - "Returned" adds CANCELLED;
  - no ticks → `""`;
  - search lands in `isearch` and **no `search` key** is present.
- `fromGridRow`: string money `"1000.00"` → 1000; a timestamp date truncated
  to ten characters; keys from hidden fields.
- `stateMachine`:
  - the table in §5, row by row;
  - an unknown status → History only;
  - `allowedForAll` of HELD+DEPOSITED → History; of `[]` → nothing.
- `whyOnlyLooking` for each terminal status; a mixed-tick sentence.
- Every `ActionSpec.build`:
  - deposit hoists company/branch, and `cheques[]` holds `{apdId, apdAccYear}`
    only;
  - bounce uses `reason/bankCharge/partyCharge` with the charges present at 0;
  - replace nests `newCheque` and **omits** empty optional strings;
  - return sends `action` and **no `returnDate`**;
  - clear and re-present have **no `allocations` key**;
  - clear's `bankDate` follows `clearDate` until edited.
- Every `ActionSpec.validate`:
  - deposit before a cheque's own date;
  - clear or bounce before the deposit date;
  - future dates;
  - replace date bounds (3 months back / 1 year ahead);
  - replace from HELD needs a reason;
  - deposit with mixed company/branch is refused.
- The ticked set: survives a page change; is cleared by a filter change.

**Live (Playwright)**, porting `tests/cheques/run.sh` and `run-ops.sh` (68
assertions). They seed their **own** posted receipt with four 1,000.00
cheques, because the register may otherwise be empty. **Assert against the
server's row, not the screen's labels.**

- deposit two → both DEPOSITED on one slip; slip dataset lists both;
- clear one → CLEARED; `av_recon_date` on the bank leg only;
- bounce one with charges → BOUNCED; ONE ChqBnc voucher; the bill reopens;
  a `BNC/…` JOURNAL bill exists;
- re-present → the originating bill settled again, **the same bill**, not a
  FIFO pick;
- return a HELD one; cancel one and re-key its number;
- replace from BOUNCED; replace from HELD **with** a reason;
- a forced 409 (e.g. clear a HELD row via the API) shows the server's
  sentence;
- a permission-denied user sees the read-only line and live History.

**Fixture rule:** seed receipts against an `abl_dr_cr = 'DR'` bill. A
fixture that grabs "the largest pending bill" will eventually grab a CR one,
which is correctly refused.

---

## 14. Backend gaps and open points this screen inherits

1. **A second bounce was a 500** (deposit → bounce → re-present → bounce),
   found on 2026-09-17. The likely cause is that step 5 re-reverses the
   original allocation and hits `ux_abj_reversal`. The transaction rolls back
   cleanly. **Its status since the notes-37/38/39 fixes has not been
   re-checked.** Retest before shipping. Until then, a second bounce shows the
   server's error and nothing else.
2. **A replacement cheque is a dead end.** It has `apd_tender_id` NULL, so
   `/return` and `/replace` refuse it (§5.2). The ledger could be found from
   `apd_voucher_id`, which the replacement row does carry. This is Prathap's
   fix. The client must not predict it.
3. **Replace from HELD without a reason.** The Qt panel never sends `reason`
   (§6.5). Confirm whether the live server refuses it. If it does, the Qt
   screen has the same defect and needs the same field.
4. **`/deposit-slip` returns 200 with an empty slip** for a key that matches
   nothing. `/list` also accepts an unknown `status` and returns 0 rows
   instead of a 400. The client checks the count (§9).
5. **The Cheques In Hand tie is displayed, not verified** (§7). It needs a
   ledger-balance route, which will come with the ledger statement.
6. **Four pre-fix DRAFT bounce vouchers** (`chqbnc00017`, `00061`, `00062`,
   `00075`) were not backfilled. `fix-draft-bounce-vouchers.sql` is in the
   share. This is test-server data, not a client work item
   (`test-box-data-not-a-bug-list`), but those vouchers will read as a party
   credited twice the day the ledger statement ships.
7. **ON_CLEARING allocation band** in Clear (§6.2). It is owed, and inactive
   while `accounts.pdc_posting_mode` is ON_RECEIPT.
8. **`accounts.bounce_reasons`** exists as a setting. Qt ignores it (§6.3).

---

## 15. Phases

1. `domain/` + `gridParams` + every `ActionSpec.build/validate`, fully unit
   tested. No UI.
2. The register read-only: filters, grid 109 with all nine parameters, pills,
   buckets, paging, the summary tiles, the detail strip, history, Enter to
   the receipt.
3. The actions: dialog, the action bar with its "why" line, the ticked set,
   Deposit/Clear/Bounce/Return, then Re-present/Replace.
4. Deposit slip dataset view (printing arrives with the print pipeline).
5. Menu 52 (Issued Cheques): the same `domain/`, `actions/` and components
   with the issued vocabulary and `apd_tra_type = 'P'`. Separate plan.

---

## 16. Do not port

These are **Qt artefacts**, not design:

- **The `QStackedWidget` of panels**, `sizeStackToCurrentPage()`, `Ignored`
  size policies on hidden pages, and the double `fitPanelDialog()` with its
  zero-delay re-fit. They are all Qt layout workarounds (a stack is as tall as
  its tallest page, a word-wrapped label's height hint, and so on). A React
  dialog sizes to its content.
- **Lifting `framePanel` out of `vboxRoot`**, and the test harness skipping
  `"panelDialog"` by name.
- **Setting layout stretch in code** because `uic` silently dropped
  `<property name="stretch">`.
- **Setting up the grid on first show** because `NexApiTableView` measured a
  hidden widget.
- **`QPointer` guards** → abort + request tags (§12).
- **The `menuId == 0` escape hatch** → resolve permissions first.
- **`restoreSelection()`** to re-read under an unchanged index → derive from
  the current row's key (§8).
- **The `ChequeRowDelegate::displayText()` override** → a pill component with
  a label map.
- **Class inheritance for panels** → `ActionSpec` objects.

These are **defects or near-defects that must not come back**:

- Omitting any of the nine grid parameters, or sending the search as `search`
  instead of `isearch` (§3.2).
- Reading row keys off visible cells, or money as a number when it arrives as
  a string (§3.4).
- Posting actions with the **session's** scope instead of the row's (§4.1),
  or opening the receipt with the session's company and branch (§10).
- A bulk action over rows in different states, or over ticked rows hidden by a
  filter (§4.2, §5).
- Wrong bounce field names (`bounceReason`, `bounceCharges`), or omitting a
  zero charge (§6.3).
- Flat replacement fields, or `""` for an empty optional (§6.5).
- A `returnDate` on Return (§6.6).
- Sending `allocations` on Re-present, or `[]` on an ON_RECEIPT Clear (§6.2,
  §6.4).
- A dialog that keeps last time's values (§6).
- Restoring selection by index after an action (§6.7).
- Printing a slip without checking it has cheques (§9).
- BOUNCED styled as dimmed and struck through (§3.4).
- Seven dead buttons with no sentence (§5.1).
- A calendar-year default date range (§3.3).
- Predicting the replacement-row dead end on the client (§5.2).
