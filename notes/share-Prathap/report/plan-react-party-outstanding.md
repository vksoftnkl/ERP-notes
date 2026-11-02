# Party-wise Outstanding — React plan

Scope: **one routed report screen**, Reports › Financial Statements ›
Party-wise Outstanding. It covers receivables and payables (one Side filter)
as on one date, over four tabs: Party-wise / Bill-wise / Group-area summary /
Due calendar. It drills party → bills → what settled a bill, opens the source
document, and exports. It writes nothing.

**This is not a port.** There is no Qt screen for it. 3.0's
`pos/Reports/customeroutstandings.*` and `supplieroutstandings.*` are a
reference for gestures and filters only (backend §1).

Companion documents that this plan does **not** repeat:

- **`plan-backend-party-outstanding.md`** (this folder): the nine routes, the
  one definition of "pending as on D" (§4), the chain rule for carried bills
  (§4.2), the derived counter tender (§3.1), the PDC split (§4.5), error codes
  (§8), and decisions O1–O7.
- **`party_outstanding_ui_mockup.png`** (this folder): the screen. Once
  approved it is the visual spec. Section numbers below refer to its bands.
- **`plan-react-ledger-statement.md`** (this folder): the sister report. This
  plan reuses its `wire/money.ts`, `wire/dates.ts`, its generation-based
  loading (§5 there) and its drill-down rules (§10 there), and says so where
  it does.
- `../react-plans/plan-react-dropdown.md`: the searchable popup picker used by
  the Group / Area / Party combos.
- `../react-plans/plan-react-printing.md`: Print / PDF / WhatsApp wait for it.

> **The whole screen in one line:** the server decides what is pending, how old
> it is and what it adds up to; the client asks with filters held in the URL,
> prints the strings it gets back, and navigates. **No money arithmetic, no
> date arithmetic, no ageing on the client.**

---

## 1. What this screen is, and what it is not

A **read-only report**: no draft, no dirty state, no save. Three facts shape
everything below.

1. **Its routes are its own.** Every call goes to
   `/api/v1/reports/party-outstanding/*`. The party credit popup
   (`master-lookups/party-credit`), `/receipts/open-items` and the grid runner
   would all return something similar today. None of them is called (the
   user's 2026-09-25 rule, backend §2).
2. **"As on" is the only date, and it decides everything.** It decides the
   year, which bills exist, what is pending, the ages, what is overdue and
   which cheques are "in hand". So the client never computes an age, a due
   date, an overdue flag or a bucket. If a column needs a date fact, it comes
   in the row. The worst bug this screen could have is a client that ages
   from `new Date()` while the server aged from `asOn`.
3. **The contract is a plan, not yet a live API.** The routes are not built.
   `wire/parse.ts` is written against the backend plan's shapes. **When the
   routes land, read the live Swagger and one real response per route before
   trusting a field name** (the 2026-09-22 lesson).

---

## 2. Module layout

```
src/features/partyOutstanding/
  PartyOutstandingScreen.tsx       // layout + tabs; reads/writes the URL query (§3.3)
  components/
    FilterStrip.tsx                // side, group, area, branch, as on, age by, buckets,
                                   //   4 checkboxes, Show F5 (§6)
    StatTiles.tsx                  // the six tiles from /parties.tiles (§7)
    PartyGrid.tsx                  // virtualised, server-sorted, paged (§8)
    PartyFlags.tsx                 // flag → chip (CHQ BOUNCED, > 180 DAYS, ADVANCE, OVER LIMIT)
    BillsGrid.tsx                  // one party's open items + Total + ledger check (§9)
    BillHistoryPopover.tsx         // Alt+F1 on a bill: /bill-history (§9.3)
    PartyCard.tsx                  // facts, ageing bars, less on-account, PDC (§10)
    AgeingBars.tsx                 // labels + amounts from the server; bar width only
    BillWiseTab.tsx                // §11.1
    SummaryTab.tsx                 // §11.2  group by AREA | GROUP | SALESMAN | BRANCH
    DueCalendarTab.tsx             // §11.3
    BillTypeChip.tsx               // billType / srcDocType → chip
    ReportFooter.tsx               // hints + Close / Excel / PDF / WhatsApp reminder / Print
  api/
    partyOutstanding.ts            // the nine GETs, one function each, AbortSignal-aware
  wire/                            // no React import, no fetch
    types.ts                       // response shapes from backend §5
    parse.ts                       // raw JSON → typed; loud on a missing required field
    errors.ts                      // server code → operator sentence (§14)
    (money.ts, dates.ts)           // IMPORTED from the ledger-statement feature (§4.2)
  query/
    params.ts                      // Filters ⇄ URLSearchParams ⇄ request query (§3.3)
    useOutstanding.ts              // /parties page 1 + options, one generation (§5)
    usePartyPages.ts               // page cache keyed by (filters, sort, page)
    useSelectedParty.ts            // /party + /bills for the focused party (§5.2)
    useBillHistory.ts              // per-bill cache for Alt+F1
  nav/
    drillTarget.ts                 // bill.src → route + key (§12)
  export/
    toXlsx.ts                      // /export shapes → workbook (§13)
```

`wire/`, `query/params.ts`, `nav/` and `export/` import no React. The tests go
there (§15).

**Shared code:** `money.ts` and `dates.ts` are the ledger-statement feature's.
If they still live under `src/features/ledgerStatement/wire/`, promote them
once to `src/shared/reports/wire/` (a move with no behaviour change) and
import from there in both features. Do not copy them. Two Indian-grouping
formatters drift.

---

## 3. Scope, inputs and the URL

### 3.1 What comes from the session, and what is a filter

| Input | Source | Why |
|---|---|---|
| `companyId` | session, captured on mount | the report is always one company |
| `asOn` | the As on date edit | required; **decides the year** (backend §4.1). There is no year picker |
| `side` | Side combo | `RECEIVABLE` / `PAYABLE` |
| `branchId` | Branch combo | "All branches (combined)" = omit the key |
| `groupId`, `areaId`, `salesmanId`, `collectionDay`, `partyId` | combos | from `/options` (§6.2) |
| `ageBy`, `buckets` | Age by combo, Buckets field | server parameters |
| `onlyOverdue`, `includeOnAccount`, `deductPdc`, `hideZero` | checkboxes | server parameters (§6.3) |

There is **no `accYear`** anywhere in this feature, not even from the
session. An As on date in last year is legal and reads last year's partition
(the chain rule, backend §4.2). The session's year is only the **default**
for the As on date (today, or the year end if today is past it).

**Defaults on first open:** side = Receivable, group = the option with
`isDefault` (Sundry Debtors), branch = the session's branch, as on = today,
age by = Bill date, buckets = `30,60,90,180`, include on-account = on, hide
zero = on, the rest off. The screen runs at once on open. That differs from
the ledger statement, which needs a ledger first. Here every input has a
default, and the first thing a user wants is the list.

### 3.2 Side changes the vocabulary, not the code path

Receivable and Payable are the same screen with the same components. The
server sends the words that differ:

| | Receivable | Payable |
|---|---|---|
| default group | Sundry Debtors | Sundry Creditors |
| Area / Salesman / Collection day | shown | **hidden** (the server refuses them, `NOT_FOR_PAYABLE`) |
| party card "Last …" | Last receipt | Last payment |
| Ctrl+R | new Receipt for the party | new Payment for the party |
| credit limit / OVER LIMIT | shown | not shown (suppliers have none) |

Changing the Side **clears** group / area / salesman / collection day /
party, because their ids belong to one side. Then it applies the new side's
default group. It does not fetch until Show (§3.3).

### 3.3 The filters live in the URL

`/reports/party-outstanding?side=R&asOn=2026-09-25&branch=<uuid>&group=<uuid>&age=BILL&b=30,60,90,180&oa=1&hz=1&tab=parties&sort=net&dir=desc&party=<uuid>`

The same rule as the ledger statement (its §3.3). Show (F5) writes the URL,
and the URL change fetches. Back returns to the previous filter set. A link
pasted to a colleague opens the same report. `companyId` is never in the URL.

`party=` (the selected party) and `sort` / `dir` / `tab` are in the URL too.
So Back from a drilled-into bill lands on the same party with the same bills
grid open.

---

## 4. The wire contract

### 4.1 Routes (backend §5)

| Function in `api/partyOutstanding.ts` | Route | Used by |
|---|---|---|
| `getOptions` | `GET …/options` | filter combos, on mount and on Side change |
| `getParties` | `GET …/parties` | tiles + party grid + totals |
| `getParty` | `GET …/party` | party card |
| `getBills` | `GET …/bills` | bills grid |
| `getBillWise` | `GET …/bill-wise` | Bill-wise tab |
| `getBillHistory` | `GET …/bill-history` | Alt+F1 popover |
| `getSummary` | `GET …/summary` | Group / area summary tab |
| `getDueCalendar` | `GET …/due-calendar` | Due calendar tab |
| `getExport` | `GET …/export` | Excel, and Print / WhatsApp later |

No route outside that list. A missing field is a server ask (§16), not a call
to an old route.

### 4.2 Amounts and dates

Exactly the ledger-statement rules (its §4.2 and §4.3), through the same
`money.ts` / `dates.ts`:

- Amounts are **strings with two decimals**. A balance is
  `{ amount, side }`. Format from the string, append ` Dr` / ` Cr` from
  `side`, never print a minus sign.
- Never `Number()`, `parseFloat` or `Intl.NumberFormat` on an amount, except:
  (a) `AgeingBars` / the summary bars, for **bar width only**, and (b) inside
  `toXlsx`.
- Dates are ISO on the wire and `dd-MM-yyyy` on screen. Never `new Date(iso)`.
  **Ages and overdue days arrive as integers in the row.** The client
  never subtracts dates.

### 4.3 Types (`wire/types.ts`) — the ones the screen leans on

```ts
type Side = 'DR' | 'CR';
type Bal  = { amount: string; side: Side | null };
type Flag = 'CHQ_BOUNCED' | 'OVER_LIMIT' | 'OVER_180' | 'ADVANCE';

type PartyRow = {
  partyId: string; name: string; area: string | null; phone: string | null;
  creditDays: number; creditLimit: string | null;
  bills: number; owed: string; onAccount: string; net: Bal;
  buckets: string[];                 // same length as bucketLabels
  overdue: string; oldestDays: number | null; pdcInHand: string; flags: Flag[];
};

type PartiesResponse = {
  asOn: string; side: 'RECEIVABLE' | 'PAYABLE'; isFuture: boolean;
  bucketLabels: string[];
  tiles: { net: Bal; parties: number; bills: number;
           overdue: string; overduePctOfOwed: string;
           aboveDays: { days: number; amount: string; parties: number };
           onAccount: string; pdcInHand: { amount: string; cheques: number };
           dueNext: { days: number; amount: string; from: string; to: string } };
  rows: PartyRow[];
  totals: { bills: number; owed: string; onAccount: string; net: Bal;
            buckets: string[]; overdue: string; pdcInHand: string };
  page: { page: number; pageSize: number; totalRows: number };
};

type BillRow = {
  billId: string; accYear: string; branchName: string;
  docDate: string; docRefno: string; billType: string;
  srcDocType: string | null; srcDocId: string | null; srcAccYear: string | null;
  voucherId: string | null;
  side: 'OWED' | 'ON_ACCOUNT'; dueDate: string | null; dueEff: string;
  billAmount: string; adjusted: string; pending: string;
  ageDays: number; overdueDays: number | null; remarks: string | null;
  tenderDerived: boolean; dataWarning?: string;
};

type BillsResponse = { rows: BillRow[];
  totals: { billAmount: string; adjusted: string; net: Bal };
  ledgerClosing: Bal };
```

`parse.ts` checks `buckets.length === bucketLabels.length` on every row and
fails loudly if not. A short bucket array silently shifts every amount one
column left, the most dangerous mis-render this screen can have.

---

## 5. Loading

### 5.1 One generation per Show

Show fires `/parties?page=1` (and `/options` only if the side changed). The
generation rule is the ledger-statement one (its §5): bump on every URL
change, abort the previous generation, and **apply nothing whose filters are
not the current URL's**. The previous result stays on screen dimmed while
the new one loads.

### 5.2 The selected party is a second, smaller generation

Moving the focus in the party grid (arrow keys, click) selects a party. It
fetches `/party` and `/bills` for it, **debounced 150 ms**, so holding ↓
through 40 parties does not fire 80 requests. It has its own generation
counter, nested in the outer one: a new Show invalidates it, and a new
selection invalidates only it. The bills grid and party card show the
previous party dimmed until the new one arrives. They never show party B's
card over party A's bills: card and bills are applied together, or not at all
(`Promise.all` inside one generation check).

The first row is selected automatically after each Show. That matches the
mockup and 3.0 (the balance grid always follows the customer grid's current
row).

---

## 6. The filter strip

### 6.1 Fields

Side\* · Group · Area · Branch · As on · **Show F5** (row 1) · Age by ·
Buckets · Only overdue · Show on-account / advances · Deduct PDC · Hide zero
balance (row 2).

The mockup's row 2 has no room for Salesman, Collection day, Party, or
Due days ≥ / ≤ (3.0's filters). They go in a **More filters (F8)** popup,
the loyalty-status pattern (2026-10-05, user: "move filters into popup").
The popup has Reset / Cancel / Apply, Cancel restores a snapshot, and a
one-line summary is shown under the strip when any of them is set
("Salesman: Ravi · Collection: Thu"). This is a deviation from the mockup. It
is named here so it is not a surprise.

### 6.2 Combos

The house dropdown component with **this feature's** data:
`/options` gives groups (tree order, `depth` for indentation), areas (with
their collection days), salesmen and branches. No grid runner or dropdown id.
Group lists sub-groups indented. Picking a parent includes its subtree (the
server does that). The label says "(+ sub-groups)".

Buckets is a text field that accepts `30 60 90 180` / `30,60,90,180` /
`30·60·90·180`. The client only normalises separators. It does **not**
validate order or range: the server refuses with `BAD_BUCKETS` and the
sentence in §14. One rule in one place.

### 6.3 All checkboxes are server parameters

Unlike the ledger statement, there are **no display-only switches** here.
Each of the four changes which rows exist or what the totals are. So each one
is in the URL and takes effect on Show. `deductPdc` changes the Net column and
the tile caption ("PDC deducted"). The client never subtracts PDC itself.

---

## 7. The six tiles

Straight from `/parties.tiles`, so the tiles cover **all filtered rows, not
the page**:

| Tile | Value | Caption |
|---|---|---|
| Net outstanding | `net` (accent; green if `side` is the on-account side) | `{parties} parties · {bills} bills` |
| Overdue | `overdue` (red) | `{overduePctOfOwed}% of pending bills` |
| Above {aboveDays.days} days | `aboveDays.amount` | `{aboveDays.parties} parties` |
| On-account credit | `onAccount` | "advances + unadjusted notes" |
| PDC in hand | `pdcInHand.amount` | `{cheques} cheques · not deducted` / `· deducted` |
| Due next {dueNext.days} days | `dueNext.amount` | `{from} → {to}` |

The "Above N days" heading takes N from the server (the third bucket edge).
It is not hard-coded at 90, because the buckets are the user's.

Clicking a tile is a filter shortcut: Overdue sets `onlyOverdue`, and Above N
sorts by the bucket from N upward. Each writes the URL like Show. Phase 3.

When `isFuture` is true, a one-line banner shows above the tiles: *"As on is
in the future: post-dated cheques up to that date are counted as settled."*

---

## 8. The party grid

### 8.1 Columns (mockup order)

`▸` · Party (+ flag chips) · Area · Cr days · Bills · Pending bills ·
On-acct Cr · **Net outstanding** · one column per `bucketLabels[i]` · Overdue
· Oldest · PDC.

- The bucket columns are **built from `bucketLabels`**, not declared
  statically. Four edges give five columns, and six edges give seven. Column
  widths for the bucket columns are shared (one saved width for all), so a
  bucket-count change does not scramble saved widths.
- Colours are presentation of server facts: Net in accent (green on the Cr /
  on-account side); the 3rd bucket onward amber → orange → red by column
  **position**; Overdue red; Oldest red above the last-but-one edge, amber
  above the middle edge; PDC blue; On-acct green.
- Area / Cr days are hidden on Payable when the server sends `null` for every
  row (§3.2).

### 8.2 Sorting and paging are the server's

A header click sets `sort` / `dir` in the URL. The server sorts and pages
(`pageSize` 200), and the totals row is `/parties.totals`, **always**, never
a sum of loaded pages. The grid is virtualised. Pages load on scroll through
`usePartyPages`, cached by `(filters, sort, page)`. The totals row is pinned
at the bottom.

Client-side sorting of loaded rows is **not** allowed. With 3 of 12 pages
loaded it would sort a sample and look right.

### 8.3 Keyboard

↑/↓ moves the selection (§5.2). Enter goes to the bills grid's first row.
Ctrl+L opens the Ledger Statement for the party (`ledger=<partyId>`,
from = year begin of `asOn`'s year, to = `asOn`). Ctrl+R opens a new
Receipt (Payment on Payable) with the party pre-filled. F8 opens More
filters. Ctrl+F focuses a find-in-grid box. Find **asks the server**
(`partyId` filter through the picker). It does not grep loaded pages.

---

## 9. The bills grid (bottom left)

### 9.1 Columns

Date · Type (chip) · Bill no · Branch · Due · Bill amt · Adjusted · **Pending**
(`… Dr` / `… Cr`) · Age · Overdue · Remarks.

- `Due` shows `dueDate`. When that is `null`, it shows `dueEff` in muted
  italics with the tooltip *"No due date on the bill. Due = bill date +
  credit days."* (backend O2). Red when `overdueDays > 0`.
- `Pending` is green with ` Cr` on `ON_ACCOUNT` rows (credit notes,
  advances), as in the mockup's `crn00014` row.
- `tenderDerived` puts a small "paid at counter" hint in Remarks, appended to
  the server's remark. `dataWarning` puts a ⚠ with the server's code in the
  tooltip. It is never hidden: it is a data fault worth a bug report.
- Chips map `billType` + `srcDocType`: OPENING · SALES · CR NOTE
  (SALES_RETURN) · ADVANCE · JOURNAL · BNC CHARGE
  (`CHEQUE_BOUNCE_CHARGE`) · PURCHASE · DR NOTE. Presentation only.

### 9.2 The total row and the ledger check

Total = `totals.billAmount` · `totals.adjusted` · `totals.net`. In the Overdue
+ Remarks span:

- `ledgerClosing` equals `totals.net` (same amount **and** side, compared as
  strings) → green **"= ledger closing ✓"**;
- otherwise → red **"≠ ledger 1,80,300.00 Dr"**, with the tooltip *"The
  ledger and the bills disagree. A posting moved one without the other."*
  Clicking it opens the Ledger Statement (Ctrl+L).

String comparison is deliberate: both come from the server formatted the same
way, so no parse is needed to compare them.

### 9.3 Alt+F1: what settled this bill

Alt+F1 (or the ▾ on the row) opens `BillHistoryPopover` from `/bill-history`:
date · type · voucher · against · amount. Reversal rows are signed, muted and
show their reason. Post-dated rows not yet effective are labelled
"PDC · from 30-09". A first line "paid at counter" shows `tenderAtBill` when
present. This is 3.0's third grid (payments against a bill). Enter on a
history row opens that voucher (§12).

---

## 10. The party card (bottom right)

From `/party`: Group · Phone · GSTIN · Credit (`creditDays` days · limit
`creditLimit`) · Last receipt / payment.

- **Ageing bars:** `AgeingBars` gets `labels` + `amounts`. Bar width =
  `Number(amount) / max`. That is the one allowed `Number()` (§4.2). An
  amount of 0 shows "—" and an empty track.
- "less on-account `<docRefno>…`" lists `onAccountItems` (first two, "+N
  more"), then **Net outstanding** = the server's `net`. The card does not
  compute "ageing total − on-account". It prints three server figures.
- **PDC in hand:** `pdcInHand` rows. Under them, if
  `pdcEffectiveUncleared` is non-empty, a muted line: *"N cheque(s) dated on
  or before {asOn}, not yet cleared: already counted as received."* That
  sentence is the whole reason backend §4.5 splits the two lists. Without it,
  users deduct those cheques twice in their heads.
- `isDualRole` adds a small "also a supplier" / "also a customer" chip.

---

## 11. The other tabs

### 11.1 Bill-wise

`/bill-wise`: the §9.1 columns plus Party and Area, server-sorted and paged,
with the totals row from the response. Enter opens the bill. Ctrl+P on
this tab prints the bill-wise list.

### 11.2 Group / area summary

`/summary?groupBy=`: a "Group by" segmented control (Area · Group · Salesman ·
Branch). Area and Salesman are disabled on Payable. Rows show parties, owed,
on-account, net, buckets and overdue, with a horizontal bar per row on net.
Enter on a row returns to the Party-wise tab with that area / group /
salesman / branch filter set (a URL write).

### 11.3 Due calendar

`/due-calendar?from&to`: a month grid (weeks × days). Each day shows the
amount due and the bill count, darker for more. One lump "Overdue before
{from}" tile sits at the left. Clicking a day opens the Bill-wise tab
filtered to bills with `dueEff` = that day (`dueOn`, backend §5.5). Default window: `asOn` → `asOn` + 30, max 92
days (the server's limit).

---

## 12. Drill-down: Enter opens the document

The ledger-statement rules (its §10) apply unchanged:

- `nav/drillTarget.ts` maps `srcDocType` (`SALE_BILL`, `SALE_RETURN`,
  `SALES_ORDER`, `RECEIPT_ADVANCE`, `PAYMENT_ADVANCE`, `VOUCHER`,
  `OPENING_BALANCE`, `CHEQUE_BOUNCE_CHARGE`) to a route + key. It is the only
  type-keyed map in the feature.
- **The key is the row's** (`srcDocId`, `srcAccYear`, the bill's branch), never
  the session's. A bill from last year opens in last year's partition.
- `OPENING_BALANCE` opens the Opening Balances screen on that party.
  `CHEQUE_BOUNCE_CHARGE` opens the Received Cheques screen on that cheque.
  An unmapped type → toast *"This bill has no screen to open yet."* Never a
  dead Enter.
- Back restores the URL (§3.3), the selected party, the bills grid's focused
  row and the party grid's scroll position, from history state.

---

## 13. Export, print, WhatsApp

| Button | Phase | How |
|---|---|---|
| **Excel** | 1 | `getExport?shape=PARTIES` (or `BILLS` on the Bill-wise tab) → `toXlsx`. Sheet 1 starts with `printedAs` (the applied filters as one sentence), then the grid with the bucket columns from the response labels and a totals row. Amounts are numeric cells with `#,##,##0.00` and a separate Dr/Cr column. File name `Outstanding_<R/P>_<asOn>.xlsx` |
| **Print / PDF** | later | the printing pipeline. Two print purposes: `OUTSTANDING_LIST` (dataset `/export?shape=PARTIES`) and `PARTY_STATEMENT` (`shape=PARTY_STATEMENT`, 3.0's "Print bill balance F7"). Disabled with the tooltip *"Arrives with the printing module"* until then |
| **WhatsApp reminder** | later | sends the `PARTY_STATEMENT` PDF for the selected party to its phone. It needs Print. It is per party, never a bulk blast from this button |

`RANGE_TOO_LARGE` on export → *"This list has 24,310 rows, and the export limit
is 20,000. Narrow the filters."*

No `window.print()` of the grid (the same reason the ledger plan gives).

---

## 14. Errors → sentences (`wire/errors.ts`)

| Server | Shown |
|---|---|
| `AS_ON_OUTSIDE_YEARS` | "{date} is outside every financial year set up for this company." (the As on field gets the accent outline) |
| `BAD_BUCKETS` | "Buckets must be rising whole numbers of days, e.g. 30 60 90 180." |
| `NOT_FOR_PAYABLE` | should never be seen (the fields are hidden); if seen: "Area / salesman filters apply to customers only." |
| `PARTY_NOT_IN_COMPANY` | "This party belongs to another company." |
| `BRANCH_NOT_IN_COMPANY` | "That branch is not part of this company." |
| `RANGE_TOO_LARGE` | §13 |
| 403 `NO_MENU_RIGHT` | "You do not have access to Party-wise Outstanding." (whole screen) |
| network / 5xx | "The report could not be loaded." + Retry; the previous result stays dimmed |

One line under the filter strip. No modal dialogs for report errors.

---

## 15. Tests

- **Vitest, `query/params.ts`:** round trip `parse(build(f)) ≡ f` for every
  filter, including buckets normalisation. `companyId` and any `accYear` are
  never in the URL. A Side change clears the side-specific ids.
- **Vitest, `wire/parse.ts`:** a row with `buckets.length ≠
  bucketLabels.length` throws. A missing `net` throws. An amount is never
  defaulted to `"0.00"`.
- **Vitest, `nav/drillTarget.ts`:** the key is the row's year and branch. An
  unknown type returns `null`.
- **Vitest, `export/toXlsx.ts`:** bucket columns come from the labels, the
  totals row is present, and amounts are numeric with Dr/Cr in its own column.
- **MSW fixtures from the mockup** (as on 25-09-2026): 9 parties, the six tile
  values, Sri Krishna Traders' 9 open items with `ledgerClosing` =
  1,76,300.00 Dr. The screen test asserts every tile and the totals row to the
  character, and the "= ledger closing ✓" state. A second fixture with
  ledger 1,80,300.00 Dr asserts the red "≠ ledger" state.
- **Race tests:** (a) Show ×3 with responses delayed in reverse; (b) hold ↓
  through 10 parties with `/bills` delayed in reverse. The final card and bills
  belong to the focused party, always the same party.
- **Paging test:** 1,200 fixture parties. Scroll to the end. The totals row
  never changes while pages load. A header click refetches page 1 sorted (spy:
  no client sort).
- **No-date-arithmetic test:** freeze the system clock at a date far from the
  fixture's `asOn`. Every age, overdue and due-date cell still matches the
  fixture exactly.
- **Live, once the routes land:** backend §12 tests 2, 3, 4, 7, 8 and 9 driven
  through the screen.

---

## 16. Backend gaps this screen inherits

1. **None of the nine routes exist.** Phases 1–3 run on MSW fixtures.
2. **Backend §3.1 / O1:** counter tenders have no adjustment row, so the
   pending is derived. Bills show "paid at counter". If the recompute suspicion
   is confirmed, a bill can re-open after a receipt cancel. The screen will
   show it re-opened, and that is the server being wrong, not the screen.
3. **Backend §3.2 / O7:** after the first carry-forward, the party-credit
   popup and receipts' open items will double-count carried bills, and this
   report will not. Testers will see the two disagree. This one is right.
4. **Backend §3.3:** an as-on figure for a past date changes after a later
   cancel. Put that in the as-on field's tooltip: "as the books stand today".
5. **`abl_salesman_id` is never stamped:** the salesman filter is by the
   customer's default salesman, which is empty on the box. The filter will
   return nothing until customers have one.
6. **`dueOn` on `/bill-wise`** for the calendar click (§11.3) is in backend
   §5.5. Check that it survives into the built route.
7. **Print purposes** `OUTSTANDING_LIST` / `PARTY_STATEMENT` are not seeded.
8. **Menu row** under 137 (backend §10), and decision O5 (one menu or two).

---

## 17. Phases

| Phase | Builds | Done when |
|---|---|---|
| 1 | `wire/` (types, parse, errors; money/dates promoted), `query/params.ts`, `nav/`, MSW fixtures from the mockup | the Vitest suites pass |
| 2 | Filter strip + More filters popup + tiles + party grid (server sort/paging) + bills grid + party card + Alt+F1 history | the mockup reproduces from fixtures; race + paging + clock tests pass |
| 3 | Bill-wise, Summary and Due calendar tabs; tile shortcuts; drill-down + Back restore; Ctrl+L / Ctrl+R; Excel | Enter on each mapped type opens the right screen in the row's scope |
| 4 | Fixtures → live routes | Swagger + one real response per route read; `parse.ts` corrected; live tests pass |
| later | Print / PDF / WhatsApp reminder | printing module exists and both purposes are seeded |

---

## 18. Do not build

- **No money arithmetic.** No net = owed − on-account, no totals from loaded
  rows, no PDC deduction, no "ageing total". Every figure comes from a
  response.
- **No date arithmetic.** No age, no due date, no overdue flag, no bucket
  placement, no "days since". `new Date()` appears only to default the As on
  field.
- **No client-side sort or filter of loaded rows.** The server sorts and pages.
  Find goes through the server.
- **No old routes**: not `master-lookups/party-credit`, not
  `/receipts/open-items`, not `/payments/open-items`, not the grid runner,
  not a dropdown id.
- **No `accYear`** input, URL key or session read (§3.1).
- **No bucket labels built on the client.** They come with the data.
- **No session scope in the drill key** (§12).
- **No report data in `localStorage`.** Saved column widths only.
- **No bulk WhatsApp / SMS** from this screen. A reminder is per party and
  waits for printing.
- **No `window.print()` print path.**
