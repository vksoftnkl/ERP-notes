# Opening Balances (menu 55) — React port plan

Scope: **one screen**: the ledger opening set for one (company, scope, year),
the bill-wise breakup of the selected party, carry forward, and the check
against the server's trial balance. It is small by line count and full of
rules, and most of those rules sit on the server. The port has two jobs. It
must not restate any of the server's rules in TypeScript. And it must keep the
invariant that lets `replace:true` work safely (§6.2).

Qt source: `src/modules/accounts/opening/`, built and tested live on 2026-09-15:

    opening_balance_entry.{h,cpp,ui}   NexDialog: the whole page
    opening_balance_entity.h           both column enums, OpeningRow / OpeningBill,
                                       the Dr/Cr and date wire helpers
    opening_balance_delegate.{h,cpp}   ledger grid editors + painted chips
    opening_bill_delegate.{h,cpp}      bill grid editors + the frozen-cell rule
    src/core/api/endpoints/accounts_end_points.h   ApiEndpoints::OpeningBalances

Companion documents that this plan does **not** repeat:

- `../openingbal/plan-backend-opening-balance.md` has the six routes, the
  carry-forward transaction step by step, and DECISIONs 1–11.
- `../openingbal/plan-qt-opening-balance.md` §0 has the house rules for every
  accounts screen. The Qt-specific ones do not port (§15). What they were
  protecting does.
- `../openingbal/client_findings_2026-09-15.md` covers what the live routes
  actually do, and what was verified working so nobody tests it again.
- **`plan-react-dropdown.md`** covers the searchable popup-grid picker and the
  `grid_param` binding rules that apply to grid 107.

> **The whole screen in one line:** an opening is one row of
> `accounts.acc_opening_balance` per ledger per (company, branch-or-NULL,
> year). A bill-wise party's opening is not a figure at all: it is the **net**
> of its OPENING rows in `acc_bill_balance`, and a different route owns it.

---

## 1. What this screen is, and what it is not

It is a **review surface over a set**, not a data-entry form for one record.
It has no list page, no F8, no document number and no lifecycle. There is one
set per scope per year, and saving means "this is the whole set now".

Three facts shape every section below:

1. **Only balance-sheet ledgers have an opening.** Income and expense close to
   the P&L and open at zero. The server never lists them, and the picker's SQL
   never offers them (grid 107 restricts to `Assets` / `Liabilities`). An
   opening on "Sales Account" is a category error, not an empty field. The
   React code must not add its own nature filter. The grid SQL already has one.
2. **A bill-by-bill party opens with its invoices, not with a figure.**
   `POST /create` rejects a bill-wise row **even at amount 0**. `POST /bills`
   owns the party's `op_amount` and rewrites it in the same transaction.
   Typing one total over a debtor who owes across nine invoices would destroy
   ageing and set-off from day one.
3. **An opening does not post to `acc_vouchers`.** `ck_abl_voucher` forces an
   OPENING bill's voucher columns to NULL. That was decided and argued in the
   concept notes. Do not reopen it from the UI side.

---

## 2. Module layout

```
src/features/openingBalance/
  OpeningBalanceScreen.tsx     // the one page: layout + wiring, no arithmetic
  components/
    ScopeHeader.tsx            // title, company · scope · year, BALANCED pill,
                               // This branch / All branches, Carry forward, Reload
    Banners.tsx                // unclassified · stale · company-level bill-wise
    LedgerGrid.tsx             // ui_table 30, the trailing picker row (§6)
    LedgerPicker.tsx           // popup grid 107 in the Ledger cell
    BillPanel.tsx              // ui_table 31 + the tie band (§7)
    TotalsBand.tsx             // Σ Dr · Σ Cr · Difference · plug · server figure
    Chip.tsx                   // Source / Bill-wise / Stale / Status (§8.3)
  state/
    draft.ts                   // OpeningDraft + reducer (the ONLY mutable state)
    useOpeningDraft.ts         // reducer + derived totals + dirty tracking
  api/
    openingBalance.ts          // list / trialBalance / create / bills (get+post) /
                               // carryForward. NO delete, on purpose (§4)
  wire/
    side.ts                    // 'D'/'C' vs 'DR'/'CR' vs shown "Dr"/"Cr" (§4.3)
    dates.ts                   // ISO on the wire, dd-MM-yyyy on screen
    parse.ts                   // /list and /bills bodies → draft rows (pure)
  payload/
    buildLedgerPayload.ts      // §9.2, the four omission rules
    buildBillsPayload.ts       // §9.3
  derived.ts                   // totals, party net, agreement with the server (pure)
  validate.ts                  // §9.1, pure → Problem[]
  scope.ts                     // Scope type, previousAccYear()
```

Everything in `wire/`, `payload/`, `derived.ts`, `validate.ts` and `scope.ts`
is free of React imports and API calls. That is where the tests go (§12).

---

## 3. SCOPE: understand this before anything else

`ux_op_scope` is `(company, COALESCE(branch, 0-uuid), year, ledger)`. A NULL
branch is **a scope of its own, not a wildcard**:

    "This branch"   → branchId sent        → that branch's own set
    "All branches"  → branchId absent/null → the COMPANY-LEVEL set (op_branch_id IS NULL)

These are **two different sets of rows**. Neither trial balance can see the
other. This was verified live: three ledgers saved at company level and a
bill-wise party saved under a branch gave two separately balanced sets that
shared nothing. The combo's label, "All branches", is the Qt screen's wording.
It does **not** mean "every branch merged". Keep the label if the operators
know it, but the tooltip and the subtitle must say "company-level set".

`acc_bill_balance.abl_branch_id` is **NOT NULL**, so a bill always has a
branch. The consequence is that **a bill-wise party's opening can only live in
a branch set.** Under "All branches":

- the bill panel is closed and cannot be opened (a click on a bill-wise row is
  silent, and Alt+B says why);
- a banner states how many bill-wise parties are **not** in these totals.

Company, branch and year are **captured once** from the session when the
screen mounts. They are not pickers. `acc_opening_balance` is partitioned by
`op_acc_year`, so a wrong year writes into the wrong partition, not just a
wrong row. The only choice the screen offers is the scope combo. Changing it
while dirty asks for confirmation, **clears every cached breakup** (bills
from one scope mean nothing in the other), and reloads.

The company **name** is not in the login payload, which carries only
`company_id`. Fetch it once from `GET /company-masters/get?compId=` (field
`compName`, and `data` may be an array) and cache it on the session store for
the accounts screens that follow. If the fetch fails, show the short uuid and
no dialog.

---

## 4. The wire contract (read off the live Swagger 2026-09-24)

All routes are under `/api/v1/opening-balances`.

| call | when | notes |
|---|---|---|
| `GET /list?companyId&accYear[&branchId]&includeZero=false` | mount, Reload, after save, after carry forward, scope change | `includeZero` is **always sent** (§6.1) |
| `GET /trial-balance?…&includeZero=true` | "Check against server" only | the same totals `/list` already returns |
| `POST /create` | save, step 1 | `SaveOpeningBalanceDto` |
| `GET /bills?partyId&companyId&accYear&branchId` | a bill-wise row is selected | `branchId` **required**: a missing one returns a field error, not an empty list |
| `POST /bills` | save, step 2, once per dirty party | `SaveOpeningBillsDto` |
| `POST /carry-forward` | the button | `CarryForwardDto` |
| `DELETE /delete` | **never** | leave it out of `api/openingBalance.ts` so nobody calls it (§9.2) |

### 4.1 Shapes

```ts
// GET /list → data
type ListPayload = {
  opCompanyId: string; opBranchId: string | null; opAccYear: string;
  rows: OpeningRowDto[];
  unclassified: { ledId: string; ledName: string; groupName: string | null }[];
  trialBalance: TrialBalance;
};
type OpeningRowDto = {
  opId: string | null; ledId: string; ledName: string;
  groupName: string | null; groupNature: string | null;
  ledIsBillByBill: boolean;
  opAmount: number; opDrCr: 'D' | 'C' | null;
  opSource: 'CARRY_FORWARD' | 'MANUAL' | 'MIGRATION' | null;
  opIsStale: boolean; opStaleSince: string | null;
  opStaleReason: 'VOUCHER_POSTED' | 'VOUCHER_CANCELLED' | 'SOURCE_OPENING_EDITED'
               | 'YEAR_REOPENED' | 'MANUAL' | null;
  opRemarks: string | null;
  priorClosingAmount: number | null; priorClosingDrCr: 'D' | 'C' | null;
  billCount: number;
};
type TrialBalance = {
  totalDebit: number; totalCredit: number;
  difference: number;             // debit − credit, SIGNED: says which side is short
  isBalanced: boolean;
  unmappedCount: number;          // ledgers under a NULL-nature group, not in the totals
  differenceLedgerId: string | null;   // OPENING_DIFFERENCE; null = unmapped
  differenceLedgerName: string | null;
};

// POST /create
type SaveOpeningBalance = {
  opCompanyId: string; opAccYear: string;
  opBranchId: string | null;      // EXPLICIT null = company-level (§9.2)
  rows: { opId?: string; opLedgerId: string; opAmount: number;
          opDrCr: 'D' | 'C'; opSource?: string; opRemarks?: string | null }[];
  replace: true;
};
// → data: { created, updated, skippedZero, deleted,
//           retainedWithBills[], flippedToManual[], staledAccYears[], trialBalance }

// GET /bills → data;  POST /bills → the same + { created, updated, deleted,
//                                               frozenUnchanged, staledAccYears }
type BillsPayload = {
  companyId: string; branchId: string; accYear: string;
  partyId: string; partyName: string; opId: string | null;
  bills: OpeningBillDto[];
  billTotalAmount: number; billTotalDrCr: 'D' | 'C' | null;
  openingAmount: number;   openingDrCr: 'D' | 'C' | null;
  isTied: boolean;
};
type OpeningBillDto = {
  ablId: string; ablDocRefno: string; ablDocDate: string; ablDueDate: string | null;
  ablCreditDays: number; ablGraceDays: number; ablDrCr: 'DR' | 'CR';
  ablBillAmount: number; ablAllocAmount: number; ablDiscAmount: number;
  ablWriteoffAmount: number; ablPendingAmount: number;
  ablStatus: string | null; ablNarration: string | null;
  isFrozen: boolean;
};
// POST /bills body
type SaveOpeningBills = {
  companyId: string; branchId: string; accYear: string; partyId: string;
  opId?: string; bills: SaveOpeningBillRow[]; replace: true;
};
```

Errors come back as `{ success:false, message, errors:[{field, message}] }`.
The messages are specific and written for the operator ("…is a bill-by-bill
party — enter its opening as bills in the breakup panel…"). **Show them as they
arrive.** No message needed rewording when the Qt screen was built.

### 4.2 Amounts are always positive

`ck_op_amount >= 0` and `ck_abl_amount > 0`. The side goes in the flag, never
in a sign, which is the same rule as on the sale bill. A number input on these
screens has `min=0`. A typed minus is refused at the input, not converted.

### 4.3 Dr/Cr has two spellings, on purpose

    acc_opening_balance.op_dr_cr   char(1)  'D' / 'C'
    acc_bill_balance.abl_dr_cr     char(2)  'DR' / 'CR'

Do **not** unify them. They are two tables with two CHECKs, and a client that
"helpfully" sends the other spelling gets a 400. Both grids **display**
"Dr"/"Cr". `wire/side.ts` converts at the edge, and that is the only place
the conversion happens:

```ts
toLedgerWire(shown): 'D' | 'C'
toBillWire(shown): 'DR' | 'CR'
fromWire(wire | null): 'Dr' | 'Cr' | ''   // empty stays empty: no opening, no side
sign(wireOrShown): 1 | -1
```

`fromWire(null)` must return `''`, not "Dr". A ledger with no opening has no
side, and painting "Dr" there invents one.

### 4.4 Dates

ISO `YYYY-MM-DD` goes on the wire and dd-MM-yyyy on screen. Use a real date
input. The Qt helper existed because `NexTable` stored cell text verbatim, and
a typed "31-03-2026" would otherwise have reached `abl_doc_date` as that
literal string. A date the operator typed that cannot be parsed is reported
**by name**. It is never dropped silently.

---

## 5. State

```ts
type OpeningDraft = {
  scope: { companyId: string; branchId: string | null; accYear: string };
  rows: LedgerRow[];                        // the model of record, grid order
  unclassified: string[];
  server: TrialBalance;                     // the server's last word, kept beside ours
  billsByParty: Record<string, BillRow[]>;  // every breakup opened this session
  opIdByParty: Record<string, string>;
  dirtyParties: Set<string>;
  currentParty: string | null;
  dirty: boolean;
  saving: boolean;
};
```

Use one reducer and one `dispatch`, with the thunks in `useOpeningDraft`. That
is the same shape as `plan-react.md` §8. Three rules carry over from the Qt
screen:

- **`billsByParty` survives navigating between parties.** Editing party A,
  clicking party B, then saving must still save A. That is why the map exists.
- **Money is summed as numbers, never parsed back out of formatted text.**
  The Qt comment on this was about a comma-decimal locale, where `2400,50`
  parses as `240050`. In React the risk is the same with `Intl.NumberFormat`
  output. Sum in **paise** (`Math.round(x * 100)`) inside `derived.ts` and
  compare to the exact paisa. Qt used an epsilon of 0.005 because its values
  had passed through a grid as text. The React totals have not, so integer
  paise let the comparison be exact.
- **Do not keep a client copy of any column the server owns.** `opRemarks`
  was once stored and never returned. The Qt screen had deliberately not
  cached it, so the server fix needed **no client change**. Repaint from the
  server's answer after every save.

---

## 6. The ledger grid (ui_table 30, "OPENING BALANCE - LEDGERS")

### 6.1 The rows: the openings that EXIST, plus what you pick

The grid loads with **`includeZero=false`**. That returns every ledger that
already has an opening, plus every ledger carrying a prior-year closing (the
carry-forward candidates). Every other ledger is **picked**, the way a
quotation picks an item.

The first Qt build listed the whole chart, arguing that "a partial chart is a
wrong trial balance, not a slow one". **That argument was false, and the
server shows why.** Asked for the same year with `includeZero` true and then
false, it returned 31 rows and 5 rows with **the same trial balance**, because
it totals the whole chart server-side whatever it puts in `rows[]`. A medium
company keeps 1000+ debtors. Showing 970 rows of `0.00` buries the trial
balance instead of reviewing it.

What this loses is seeing at a glance that a ledger was forgotten. The totals
band covers that: it is permanent, it comes from the server over the **full**
chart, and it shows an omission as a difference.

`includeZero` is always sent explicitly. Until 2026-09-15 an **absent** flag
returned an empty `rows[]`, and because 0 = 0 that looks balanced. It is fixed
server-side, but the call does not rely on the default either way.

### 6.2 THE INVARIANT

`POST /create` goes out with `replace:true`, and the server **soft-deletes
every opening absent from the payload**. That is how a row is cleared: by
removing it. It is safe only while the grid holds **every** row that has an
opening.

`includeZero=false` guarantees that, and the picker only ever **adds**. So:

> **Never page, filter, virtualise-away or trim the loaded set.** A search box
> over these rows, added later with the best intentions, would make Save
> delete every opening scrolled out of view or filtered out.

Virtualised *rendering* is fine, because the rows stay in `draft.rows`. A
filter that drops rows from `draft.rows`, or a server-paged fetch, is not. Put
this in a comment at the top of `LedgerGrid.tsx` and in a test (§12).

### 6.3 Columns

| key | shown | editable | note |
|---|---|---|---|
| `ledgerName` | Ledger | trailing row only: the picker | a saved row cannot be repointed (§6.5) |
| `groupName` | Group | — | |
| `groupNature` | Nature | — | |
| `isBillWise` | Bill-wise | — | chip |
| `priorClosing` | Prior closing | — | blank when `priorClosingAmount` is null, not `0.00` |
| `priorDrCr` | Side | — | |
| `amount` | **Opening** | **yes, unless bill-wise** | `min=0` |
| `drCr` | **Dr/Cr** | **yes, unless bill-wise** | two-value select |
| `source` | Source | — | chip: Carry fwd / Migration / Manual |
| `isStale` | Stale | — | chip; the tooltip gives the date and reason (§8.2) |
| `billCount` | Bills | — | |
| `remarks` | Remarks | yes | round-trips (fixed server-side 2026-09-15) |
| `opId`, `ledId` | hidden | — | `opId` absent = no opening yet |

`ui_tables` **30** owns the order, headings, widths and visibility, through
`useGridColumns(30)`. The screen sets no widths of its own. That was a firm
Qt rule because a re-seed fought the operator's saved percentages, and the
same would happen in React (§15). Tests assert the grid **follows** the
layout. They never assert that a particular column is visible.

### 6.4 The trailing blank row IS the picker

There are no Add or Remove buttons. The grid always ends in **one** blank
row, and its Ledger cell opens popup grid **107** "POPUP - LEDGERS". Picking
into it grows the next blank row. The trailing row is furniture, not data:

- it does **not** mark the screen dirty;
- `buildLedgerPayload` drops it and `validate` ignores it;
- **nothing else on it opens until a ledger is picked.** Opening, Dr/Cr and
  Remarks are disabled and greyed on any row with no `ledId`. An amount with no
  account is not an opening balance.

Grid 107 parameters: **`iled_company_id` is always sent.** The configured-grid
runner substitutes **bare `i`-prefixed tokens** as text
(`NULLIF('iled_company_id','')::uuid`). It does **not** support `:name`
binding. A placeholder in the SQL with nothing in `grid_param` fails the whole
query. See `plan-react-dropdown.md` and notes.txt (14). The picked row carries
`led_id`, `led_name`, `acc_group_name`, `acc_group_nature` and `bill_wise`
(a non-empty string means true).

On a pick:

1. **Duplicate:** if the ledger is already on row *n*, refuse and name it:
   *"'X' is already row n. A ledger opens once per year."* (`ux_op_scope`).
   Clear the cell.
2. Otherwise fill the row: source `MANUAL`, side `D` if empty.
3. Grow the next blank row.
4. Move focus. A bill-wise party opens the bill panel, because that is where
   its work is. An ordinary ledger moves focus to Opening.

### 6.5 Editing rules

- **Amount on a row with no side:** the side defaults to the prior closing's
  side if there is one, otherwise `D` (what an asset opens on).
- **An edited CARRY_FORWARD row becomes MANUAL on screen immediately**, on
  Amount or Dr/Cr. The server flips it anyway and reports it under
  `flippedToManual`. The operator should see what they did to a derived figure
  before saving, not after.
- **`opSource` is echoed, never asserted.** It is server-owned. Echoing
  `CARRY_FORWARD` onto a `MIGRATION` row does nothing. Send back what was
  loaded and repaint from the answer.
- **A saved row cannot be repointed to another ledger.** `op_id` belongs to
  the ledger it was written for. The message is: remove the row and add the
  other ledger.
- **A bill-wise row's Opening and Dr/Cr are read-only**, greyed, with the
  tooltip *"Owned by the bill-wise breakup — the net of n opening bill(s)."*
  The reducer also refuses `SET_AMOUNT` on a bill-wise row. That is the
  equivalent of Qt's belt-and-braces reset in `onLedgerCellEdited`.

### 6.6 Removing a row (`-` key, or a row action)

Every removal **asks for confirmation**. It is one keystroke, and the row may
hold a figure somebody spent the morning agreeing.

- **A saved row** (`opId` present) says so more strongly: *"'X' has a SAVED
  opening of 4,17,000.00 Dr. Removing the row and saving DELETES it from the
  server."*
- **A bill-wise party with `billCount > 0` cannot be removed.** Its bills own
  its figure, and the server would keep the row anyway (`retainedWithBills`).
  The message is: open the breakup, remove the bills, save, then remove the row.
- The trailing blank row cannot be removed.
- If the removed row is the open party, close the panel.

### 6.7 Selection opens the breakup

A **single click** or an arrow-key move onto a row runs one function,
`syncBillPanelToRow(row)`, so mouse and keyboard behave the same:

- ordinary ledger: **close and clear** the panel, silently;
- bill-wise row under "All branches": close it, silently;
- bill-wise row under "This branch": open it and load it if it is not cached.

**Double-click is not wired to the panel.** It is how a cell is opened for
editing. In Qt, taking it over made an ordinary ledger answer "Not a bill-wise
party" instead of letting the operator type a figure. **Alt+B** is the
explicit request, and it is the one that explains a refusal ("not a bill-wise
party" / "bills live in a branch set").

---

## 7. The bill panel (ui_table 31, "OPENING BALANCE - BILLS")

It sits **under** the ledger grid and is **never a modal**. The tie has one
number on the ledger row and the other in the breakup, and a modal would hide
the trial balance just as the operator changes the figures that move it.

### 7.1 Columns

| key | shown | editable | frozen bill |
|---|---|---|---|
| `docRefno` | Invoice no | yes | **no** |
| `docDate` | Invoice date | yes | yes |
| `dueDate` | Due date | yes, `>= docDate` | yes |
| `creditDays` | Credit days | yes | yes |
| `graceDays` | Grace | yes | yes (see §13.5) |
| `drCr` | Dr/Cr | yes | **no** |
| `amount` | Amount | yes, `> 0` | **no** |
| `narration` | Narration | yes | yes |
| `allocated`, `pending`, `status` | | — | GENERATED; never sent |
| `ablId`, `isFrozen` | hidden | — | `ablId` is load-bearing (§9.3) |

A **frozen** bill (`isFrozen`: receipted against) has its money cells
(refno, amount, side) disabled and greyed, with the reason in the tooltip.
**Say why** instead of rendering an input that silently refuses. That is the
most common way these grids get reported as broken.

### 7.2 The tie: the party opens at the NET

5,000 DR plus 3,000 CR opens the party at **2,000 D**, not 8,000. This was
verified live. `derived.partyNet(bills)` returns `{ amount: |Σ sign·amt|,
side }`. The ledger row's Opening and Dr/Cr **mirror it live** as bills are
typed. The tie is true by construction once saved, because `POST /bills`
rewrites `op_amount` in the same transaction. Showing any other figure while
typing would be a lie that the save then silently corrects.

The band text is either *"n bill(s), net 2,000.00 Dr — this is the party's
opening."* or, with no bills, *"No opening bills yet — this party opens at
zero. Key its outstanding invoices into the blank row below."*

`GET /bills` also returns `isTied`, `openingAmount` and `billTotalAmount`. The
Qt screen ignores them. Use `isTied === false` on **load** as a loud warning,
because it means the data was written around the route. Do not use it as the
live figure.

### 7.3 Loading: no race by construction

Qt had to **clear the grid before the fetch**, because the label changed
straight away while the grid still showed the last party's invoices. It also
had to check "is this still the open party?" in the callback. In React, render
the panel from `billsByParty[currentParty]`, showing **Loading…** while that
entry is undefined. A late answer for party A then only fills A's cache entry
and can never paint under B's name. Keep the late write, which is harmless and
already cached, and add no "discard if stale" logic.

A bill-wise party picked into a fresh row has no bills yet. The panel opens
empty with its own trailing blank row.

### 7.4 Rows

The panel has the same trailing blank row as the ledger grid. A new bill
defaults to **DR**, dated **today**. `-` asks for confirmation. **A frozen
bill cannot be removed**: *"'X' has been receipted against. Reverse the
receipt, or write the balance off — either leaves a trail; deleting the
invoice does not."* The server would report it under `frozenUnchanged` rather
than delete it.

Any bill edit adds the party to `dirtyParties` and marks the screen dirty.

---

## 8. The totals band, banners and chips

### 8.1 The totals band (footer, above the buttons)

It shows Σ Dr, Σ Cr and Difference, as a small uppercase caption over a large
value. The **whole band** is green when balanced and red when not, because an
unbalanced opening is a fact about the set, not about one number in it. A
pill at top right repeats it in one word: **BALANCED** / **OUT BY 2,000.00**.

Below the figures:

- **Plug line:** if balanced, empty. Otherwise *"'Difference in Opening
  Balances' will take the difference."* when `differenceLedgerName` is set, or
  *"No OPENING_DIFFERENCE ledger is mapped for this company — nothing will
  absorb the difference."* when it is not. The server writes the plug. The
  client only names it.
- **Server figure:** *"Agrees with the server."*, or *"Unsaved — the server
  last said Dr x / Cr y."* The client's figure comes from `draft.rows` and
  `server` comes from the last `/list` or `/create`. **When nothing is dirty,
  a disagreement is a bug**, and the screen shows it loudly instead of
  averaging it away.

**Save is NOT refused when the set is unbalanced.** A half-keyed opening is a
normal state for an afternoon's work. Refusing to save would force the
operator to hold it in their head. Tell them, every time.

The totals band sits **beside the Save button**, so the number that decides
whether saving makes sense is next to the button that saves. It has its own
band. Figures squeezed onto the button line read as a caption, not a total.

### 8.2 Banners (one area, stacked)

1. **Unclassified:** *"n ledger(s) are under a group with no nature and are
   NOT listed or counted: A, B. Give their group a nature in the Ledger Group
   master (menu 54) — until then this trial balance is taken over an
   incomplete chart."* Add a link that opens menu 54.
2. **Stale:** *"n row(s) are stale…"*. The row's tooltip gives the date and
   `opStaleReason`. They are **flagged, never silently corrected**: the figure
   may already have been reported. Regenerate, do not overwrite.
3. **Company-level bill-wise:** under "All branches" with any bill-wise row:
   *"…n party(ies) here can only be opened under 'This branch' — their
   figures are not in these totals."*

### 8.3 Chips

| colour | meaning | values |
|---|---|---|
| amber | the operator did it, or must look | MANUAL · STALE · PARTIAL · FROZEN |
| blue | the system derived it | CARRY FWD · BILL-WISE · MIGRATION |
| green | settled | OPEN |
| grey | unrecognised value | anything else, shown verbatim |

The current cell is painted **dark with a 2px border**. That is the one piece
of the mockup Qt could only do in the delegate. In React it is plain CSS on
the focused cell.

---

## 9. Save (Ctrl+Enter, or the Save button)

### 9.1 Validate first (`validate.ts`, pure)

Problems are listed **by ledger name and bill number**, never by constraint
name:

- a row that carries an amount but no ledger;
- an amount > 0 with no side;
- for **every dirty party**, not just the one on screen, while skipping the
  trailing blank bill:
  - no invoice number (*"it is what the receipt will be matched against"*);
  - a duplicate invoice number within the party, compared **case-insensitive
    and trimmed**;
  - no invoice date (*"ageing measures from it"*);
  - a due date before the invoice date;
  - amount ≤ 0.

Imbalance is **not** a problem (§8.1). If a cell editor is still open, commit
it before validating. In Qt, Ctrl+Enter moved focus to the grid for this
reason. In React, commit on the shortcut before building anything.

### 9.2 `buildLedgerPayload`: four omission rules

A row is **omitted** when it is:

1. **bill-wise.** `/create` rejects it even at 0, and `replace:true` does not
   lose it: it comes back under `retainedWithBills`;
2. **blank** (no `ledId`);
3. **zero.** **Absence is the zero.** Sending 0 only earns a `skippedZero` and
   leaves the old row standing. Omitting it under `replace:true` is what clears
   it, which is why `DELETE /delete` is never needed.

and the fourth rule is on the envelope:

4. **`opBranchId` is an explicit `null` for the company-level set.** Omitting
   the key is not the same request. Make sure the serializer keeps `null`
   (it does with `JSON.stringify`, but a form library or an "omit empty"
   helper may not).

`replace: true` is always sent. `opRemarks` is sent trimmed, and omitted when
empty. `opDrCr` defaults to `D`.

### 9.3 `buildBillsPayload`: `ablId` is load-bearing

For every dirty party: `{ companyId, branchId, accYear, partyId, opId?, bills,
replace:true }`. `branchId` is **required** and is always the real session
branch, because the panel cannot be open under the company-level scope.

**Every loaded bill must carry its `ablId`.** A bill without one is an
insert. Re-sending a bill the screen loaded without its `ablId` is refused
by `ux_abl_doc_refno`: *"'ADV/2025/118' is already an opening bill of this
party for this year. Send its ablId to update it."* So `replace:true` does
**not** mean "here is the whole breakup, sort it out". Only genuinely new
bills leave out `ablId`.

Never send `ablPendingAmount`, `ablStatus` or `ablAllocAmount`. They are
GENERATED or server-derived. Drop the trailing blank bill.

### 9.4 The sequence

```
POST /create (whole ledger set)            → keep data.trialBalance
for party of dirtyParties, ONE AT A TIME:  → POST /bills
    (each rewrites its party's op_amount, so two in flight
     would race over the same trial balance: await, never Promise.all)
then: clear dirty → toast "Saved (+ n breakups)" → reload /list
```

**Partial failure must say exactly what happened.** If `/create` succeeded and
party X's `/bills` was refused: *"The ledger openings were saved, but the bills
of 'X' were refused. Fix them and save again — the rest is already on the
server."* Then show the server's field errors. Otherwise the operator reads a
generic failure and retypes work that is already safely saved. Keep X, and any
parties after it, in `dirtyParties`.

**Reload after every save.** The server owns the ids, the sources and the
generated bill figures. A screen that keeps its own copy drifts from the set
it has just written. A party whose panel is open stays open across the reload
and has its bills refetched.

Report the useful counts from the answers in the toast: `deleted`,
`flippedToManual.length`, `frozenUnchanged`, and **`staledAccYears`**
(*"Openings of 2027-2028 are now stale"*). The last one is the consequence
the operator is least likely to expect.

---

## 10. Carry forward

It is one button, `POST /carry-forward { companyId, fromAccYear, toAccYear,
branchId }`. `branchId: null` means the company-level set.

- `fromAccYear = previousAccYear(accYear)` is arithmetic on the first half
  (`"2026-2027"` → `"2025-2026"`), never a date calculation. `ck_op_acc_year`
  requires the second half to be the first plus one. For a year string that is
  not `YYYY-YYYY`, say so and send nothing.
- **Refused while dirty:** "Save or reload first." The reload that follows a
  carry forward would overwrite what is on screen.
- **Confirm with the rule in the dialog:** every balance-sheet closing becomes
  an opening; the year's P&L result goes to RETAINED_EARNINGS; every still-open
  bill of a bill-wise party is carried as an OPENING bill (*"without which every
  debtor's ageing would restart from scratch"*); **MANUAL and MIGRATION rows are
  left alone**; and the scope in words.
- **`overwriteManual` is never sent.** A button that can silently overwrite a
  figure somebody typed is not one this screen offers.
- Report `created`, `updated`, `skippedManual` (always reported, even at 0),
  `billsCarried`, and the P&L result. If `retainedEarningsLedgerId` is null
  and the result is non-zero: *"…but no RETAINED_EARNINGS ledger is mapped —
  it went nowhere."* If nothing was carried: *"2025-2026 has no closing
  balances on this server yet."* Then reload.

Permission: `canCreate` (the Qt screen enables it on `ctx.canCreate`).

---

## 11. Check against the server

This is an on-demand button. `GET /trial-balance?includeZero=true` updates
`server` and shows: *"The server has Dr x against Cr y, a difference of z."*
Then one of: *"This screen has unsaved changes, so its own strip is ahead of
that."*, or **"That does NOT match what this screen has computed and nothing
is unsaved — that is a bug, not a rounding difference."** Add
`unmappedCount` and the unmapped OPENING_DIFFERENCE line when they apply.

---

## 12. Tests

**Pure first (Vitest, no DOM).** These are the ones that catch real defects:

- `wire/side` — `toLedgerWire('Cr') === 'C'`, `toBillWire('Cr') === 'CR'`,
  `fromWire(null) === ''` (not "Dr"), `fromWire('DR') === 'Dr'`.
- `derived.partyNet` — 5,000 DR + 3,000 CR → `{2000, 'D'}`; 1,000 DR + 3,000 CR
  → `{2000, 'C'}`; no bills → `{0, null}`.
- `derived.totals` — computed in paise; 0.1 + 0.2 style inputs balance exactly;
  bill-wise rows count at their **mirrored net**.
- `buildLedgerPayload` —
  - a bill-wise row at any amount is absent, including 0;
  - a zero row is absent, not sent as 0;
  - the trailing blank row is absent;
  - company-level sends **`"opBranchId": null` in the JSON string**, with the
    key present;
  - `opSource` is echoed unchanged; `replace: true`.
- `buildBillsPayload` — every loaded bill keeps its `ablId`, a new one has
  none; the trailing blank bill is dropped; GENERATED fields are absent;
  `ablDrCr` is `DR`/`CR`, never `D`/`C`.
- `validate` — duplicate refno `"inv/1"` vs `" INV/1 "`; due date < doc date;
  a problem in a dirty party that is **not** on screen is still reported; an
  unbalanced set produces **no** problem.
- `previousAccYear` — `"2026-2027"` → `"2025-2026"`; `"2026"` → `null`.
- Reducer — editing Amount on a CARRY_FORWARD row flips it to MANUAL; the
  bill-wise row refuses `SET_AMOUNT`; picking a ledger already on the list is
  refused; the trailing row never sets `dirty`.

**Invariant test (component):** load N rows, type into a filter or search if
one is ever added, then save, and assert the payload still holds every loaded
non-bill-wise non-zero row. It fails the day someone makes §6.2 false.

**Live (Playwright, against the server).** Port `tests/openingbalance/run.sh`
as the fixture. It **seeds its own data through the same API** (four ledgers
plus one bill-wise party, Dr 350,000 = Cr 350,000) and leaves the set
balanced, so it can be re-run.

- save → reload → identical; the remark round-trips;
- bill-wise party: key 5,000 DR + 3,000 CR, and the ledger row shows 2,000 Dr
  **before** save and after;
- save a loaded bill again unchanged, which proves `ablId` goes through (no
  `ux_abl_doc_refno` 400);
- "All branches": panel closed, banner present, `branchId` absent on `/list`;
- close or navigate while dirty → confirmation;
- unmount while `/list` is in flight → no error, no state write.

**The fixture collides with a live session.** It uses `replace:true` on the
test company for 2026-27. If anyone has menu 55 open there at the same time,
each save deletes the other's rows. Three Qt runs failed this way before
anyone noticed. **Re-run before believing a failure.** Never assert an exact
row count or which column is last: the chart grows (seeding the ledger role
map took 21 rows to 31 overnight), and the layout is the operator's.

---

## 13. Backend gaps and open points this screen inherits

1. **DECISION 1 is still open:** `acc_ledger_master.led_ob_amount / led_ob_type
   / led_ob_as_on` (one figure, no year) vs `acc_opening_balance` (per year,
   with provenance). The screen is built on the second, and the argument is
   structural (a ledger's opening differs per year). But nothing stops the
   ledger master from writing the first. Until it is closed, two homes can
   disagree. A third home, `led_total_balance / _dr / _cr`, is stamped and read
   by nothing. **Never derive from it.**
2. **OPENING_DIFFERENCE and RETAINED_EARNINGS are not mapped on the test
   company**, so the plug line and the carry-forward P&L both say "unmapped".
   That is correct behaviour, not a defect.
3. **Stock-in-Hand has no FROM STOCK source.** `ck_op_source` allows only
   CARRY_FORWARD | MANUAL | MIGRATION, and voucher type 1 (Opening Stock) has
   `vchr_affects_accounts = false`. So the tie between the Stock-in-Hand
   opening and the menu 44 valuation cannot be derived. Do not invent a source
   value on the client.
4. **History (Ctrl+H)** needs an `audit_screen` id for this table. There is
   none yet, so hide the button until one exists.
5. **Grace days on a frozen bill.** The `/bills` route description says a
   receipted bill accepts "date, credit-day and narration changes". The Qt
   delegate also leaves **Grace** editable. It has not been tested live. Before
   shipping, either confirm the server accepts it or freeze it too. Do not copy
   the Qt behaviour on trust.
6. **Stale comment in the Qt source:** the header comment in
   `accounts_end_points.h` says the screen "always asks for true". The code
   sends `includeZero=false` on `/list` and `true` on `/trial-balance`. The
   **code** is right (§6.1).

---

## 14. Phases

1. **Read-only.** `/list`, the ledger grid painted from `useGridColumns(30)`,
   totals, the server-agreement line, banners, chips, and the scope combo.
   This proves scope, the wire parse and the paise arithmetic.
2. **Edit + save of ordinary ledgers.** The trailing picker row (grid 107),
   Amount / Dr/Cr / Remarks, the MANUAL flip, removal with confirmation,
   `buildLedgerPayload`, validate, save → reload. This ships on its own: a
   company with no bill-wise parties can be opened.
3. **The bill panel.** Load, the net tie mirrored live, frozen cells, the
   multi-party `billsByParty` cache, sequential `/bills` in the save, and the
   partial-failure message.
4. **Carry forward + Check against server.**
5. History, once §13.4 has an id.

---

## 15. Do not port

The following are **Qt artefacts**. None of them are design:

- **The `QPointer` guard on every callback.** `NexDialog` sets
  `WA_DeleteOnClose`, so an in-flight reply called into freed memory (a
  segfault in `QTableWidget::setRowCount`, reproduced under gdb). React will
  not crash here, but the underlying rule still holds: **a response for an
  unmounted screen or a previous scope must not write state.** Pass an
  `AbortController` signal to every fetch, abort on unmount **and on scope
  change**, and ignore a `/list` answer whose scope is not the current one.
- **The `ctx.menuId == 0` permission escape hatch.** Qt built the dialog
  before assigning its menu id, so permissions looked "all false" during
  construction and the screen refused a user who had every right. The lesson
  that ports is **unloaded ≠ denied**. Gate writes on the resolved permission,
  render read-only while permissions load, and let the server remain the
  authority.
- **`seedLedgerLayout()` / `seedBillLayout()`** and the "do not reconnect
  `layoutLoaded`" rule. `NexTable::loadLayout()` hid every column until the
  server named it back, and a re-seed overwrote the operator's saved widths.
  React renders from `useGridColumns`, with a static default only as a
  fallback. The rule that carries over: **the screen never hard-codes a
  width.**
- **`m_loading` flags** that stopped a programmatic `setCellValue` from firing
  the edit handler. React state writes do not fire user-intent handlers.
- **The delegate-painted focus border and chips.** A `QTableWidgetItem` has no
  border-radius and `::item:focus` does not follow the current index. In React
  this is CSS.
- **`cellNumber()` stripping commas** from formatted cell text. The draft
  holds numbers. Nothing is parsed back out of display text.
- **The hidden `IsBillWise` "1"/"" column** used so the delegate could read a
  flag. It is a boolean on the row.
- **The "Save *" label** as the dirty marker. Use the app's normal
  unsaved-changes indicator and route-leave guard (`closeEvent` in Qt).

These are **defects or near-defects that must not come back**:

- A search or filter box over the loaded rows (§6.2).
- Double-click opening the bill panel (§6.7).
- Leaving the last party's bills visible while the next party loads (§7.3).
- Sending a bill-wise row, or a zero row, to `/create` (§9.2).
- Rebuilding the bills array without `ablId` (§9.3).
- `Promise.all` over the `/bills` calls (§9.4).
- Refusing save on an unbalanced set (§8.1).
- A `DELETE /delete` call anywhere in the feature (§9.2).
- Sending `overwriteManual` (§10).
- A client-side cache of any server-owned column, `opRemarks` included (§5).
