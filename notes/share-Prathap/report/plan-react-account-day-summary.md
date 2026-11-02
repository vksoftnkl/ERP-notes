# Account Day Summary — React plan

Scope: **one routed report screen**, Reports › Financial Statements › Account
Day Summary (menu 284; menu 181 Day Book Detail opens the same screen on the
Vouchers tab). It covers one company, one branch or all, one day or a range
inside one fiscal year, with three tabs (Summary / Vouchers / Day-wise),
drill-downs into a voucher, a group's ledgers and the Ledger Statement, and
Excel export. It writes nothing.

**This is not a port.** There is no Qt screen. 3.0's `pos/Reports/daybook.*`
is only a reference for the gestures: the ◀ ▶ one-day step, the type filter,
Enter opens the voucher, and Print / Export / WhatsApp.

Companion documents that this plan does **not** repeat:

- **`plan-backend-account-day-summary.md`** (this folder): the seven routes
  (§5), the counted-legs and pair rules (§4.1), how money ledgers and report
  groups are classified (§4.2–4.3), checks (§8), error codes (§9), and
  decisions D1–D9.
- **`account_day_summary_ui_mockup.png`** (this folder): the screen. Section
  numbers below refer to its bands.
- **`plan-react-ledger-statement.md`** (this folder): this screen copies its
  money formatting (§4.2), date handling (§4.3), loading generations (§5) and
  URL rules (§3.3). Where this plan says "as the ledger statement", it means
  that file, and the shared code should be lifted into `src/features/reports/common/`
  rather than duplicated.

> **The whole screen in one line:** the client asks, formats and navigates.
> **It does no money arithmetic at all.** Totals, tiles, opening and closing,
> the Day-wise running closings and the Dr = Cr verdict all come from the
> server. The client never adds a column.

---

## 1. What this screen is, and what it is not

It is a **read-only report**, so it has no draft, no dirty state and no save.
It is the screen an accountant opens at the end of a day: "did everything
post, where did the cash go, does the day balance". So the figures must be the
server's own, and the panels must agree with each other.

Four facts shape everything below:

1. **Its routes are its own.** Every call goes to
   `/api/v1/reports/day-summary/*`. The Type combo does not use the dropdown
   runner; the business-day status does not come from `/till/*`. The one
   cross-screen move, Enter on a ledger, is a **navigation** to the Ledger
   Statement screen with its URL, not a call to its routes.
2. **The Summary tab is ONE call** (`/summary`). The tiles, the type grid, the
   money grid, the group grid and the checks all come from one server snapshot,
   so they tie to each other. **Never** split the panels into separate fetches
   "for speed" — two fetches can straddle a posting and disagree by one bill.
3. **Cancelled pairs inside the range are left out of the summary figures**
   (backend §4.1, D2), and listed on the Vouchers tab. The client does not
   filter, net or hide anything itself; it shows the `cancelledPairs` count
   the server sends.
4. **The contract is a plan, not yet a live API.** `wire/parse.ts` is written
   against the backend plan's examples. **When the routes land, read the live
   Swagger and one real response per route before trusting a single field
   name.**

---

## 2. Module layout

```
src/features/daySummary/
  DaySummaryScreen.tsx           // layout + wiring; reads/writes the URL (§3.3)
  components/
    FilterStrip.tsx              // ◀ date ▶, To, Company, Branch, Type, 4 checkboxes, Show F5
    BusinessDayPill.tsx          // OPEN / CLOSING / CLOSED / NONE / MIXED + provisional note
    KpiTiles.tsx                 // tiles[] in server order (§7.1)
    VoucherTypeGrid.tsx          // types[] + typeTotals (§7.2)
    MoneyLedgerGrid.tsx          // moneyLedgers[] + moneyTotals + tie-out note (§7.3)
    GroupGrid.tsx                // groups[] + groupTotals (§7.4)
    GroupLedgersDialog.tsx       // Enter on a group → /group-ledgers (§9.2)
    ChecksPanel.tsx              // checks[] → cards; "show" → /checks/detail (§7.5)
    VoucherList.tsx              // Vouchers tab, paged, ▾ legs (§8)
    DailyTable.tsx               // Day-wise tab (§8.3)
    ReportFooter.tsx             // hints + Close / Excel / PDF / WhatsApp / Print
  api/
    daySummary.ts                // the seven GETs, AbortSignal-aware
  wire/
    types.ts                     // response shapes from backend §5
    parse.ts                     // fails loudly on a missing required field
    checks.ts                    // check code + facts → sentence (§7.5)
    errors.ts                    // server error code → operator sentence (§11)
  query/
    params.ts                    // Filters ⇄ URLSearchParams ⇄ request query
    useSummary.ts                // /summary, one generation
    useVoucherPages.ts           // page cache keyed by (filters, page)
    useDaily.ts
  nav/
    drillTarget.ts               // voucher src → route; ledger → Ledger Statement URL
  export/
    toXlsx.ts                    // /export → workbook: Summary sheet + Vouchers sheet
```

From `src/features/reports/common/` (lifted out of the ledger statement):
`money.ts`, `dates.ts`, `TypeChip.tsx`, `useGeneration.ts`. `wire/`,
`query/params.ts`, `nav/` and `export/` import no React; the tests go there.

---

## 3. Scope, inputs and the URL

### 3.1 Session vs filter

| Input | Source |
|---|---|
| `companyId`, `accYear` | the session, captured on mount; the Company combo is shown, as in the mockup, but it switches the session company (the same action as the header switcher), so it is never a URL key |
| `branchId` | the Branch combo; `All branches` = omit the key |
| `fromDate` | the Date edit (◀ ▶ step it) |
| `toDate` | the To edit; equal to Date = one day, the default |
| `voucherTypeId` | the Type combo, over `/voucher-types` |

The date edits clamp to the fiscal year (from the session). The server still
refuses (`RANGE_OUTSIDE_YEAR`). **Defaults on first open:** Date = To = today
(or the year end if past it), branch = the session's branch, type = all.

### 3.2 ◀ ▶ and Alt+← / Alt+→

Step **both** dates by one day and show at once, keeping the range length at
one day (3.0 did exactly this: `dEFrom ± 1`, `dETo = dEFrom`). When a range is
open, ◀ ▶ collapse it to a single day first. They stop at the fiscal year's
edges (disabled, not wrapping into the next year).

### 3.3 The filters live in the URL

`/reports/day-summary?date=2026-10-09&to=2026-10-09&branch=<uuid>&type=3&tab=summary&clr=1&zero=0&group=REPORT`

As the ledger statement: Show (F5) writes the URL, the URL change fetches;
◀ ▶ write the URL at once. Back returns to the previous day. Menu 181 opens
`…?tab=vouchers` with the defaults.

---

## 4. The wire contract

### 4.1 Routes (backend §5)

| Function | Route | Used by |
|---|---|---|
| `getVoucherTypes` | `GET …/voucher-types` | Type combo, once per session (cache) |
| `getSummary` | `GET …/summary` | Summary tab: tiles, 3 grids, checks, business-day pill |
| `getVouchers` | `GET …/vouchers` | Vouchers tab, one call per page; `draftsOnly` view |
| `getDaily` | `GET …/daily` | Day-wise tab |
| `getGroupLedgers` | `GET …/group-ledgers` | Enter on a group row |
| `getCheckDetail` | `GET …/checks/detail` | "show" on a check card |
| `getExport` | `GET …/export` | Excel (Print / PDF / WhatsApp later) |

No route outside that list. A missing field is a server ask (§12), not a call
to an old route.

### 4.2 Amounts and dates

As the ledger statement §4.2–4.3: amounts are strings, formatted from the
string (Indian grouping, two decimals as sent), sides from `side`, never a
minus sign; `0.00` with no side. **No `Number()` anywhere** — this screen has no
chart, so it has no exception. Dates stay ISO strings; `dd-MM-yyyy` on screen.
The subtitle's weekday ("Friday") comes from `wire/dates.ts` formatting the ISO
string, not from `new Date(iso)`.

### 4.3 Types (`wire/types.ts`)

```ts
type Side = 'DR' | 'CR';
type Bal = { amount: string; side: Side | null };
type MoneyKind = 'CASH' | 'BANK' | 'CLEARING';
type RowKind = 'NORMAL' | 'CANCELLED' | 'REVERSAL';
type CheckLevel = 'OK' | 'WARN' | 'INFO' | 'FAIL';

type TypeRow = {
  voucherTypeId: number; short: string; name: string; nature: string; category: string;
  vouchers: number; drafts: number; cancelledPairs: number; loneHalves: number;
  amount: string; cashIn: string; cashOut: string; bankIn: string; bankOut: string;
};
type MoneyRow = { ledgerId: string; ledgerName: string; kind: MoneyKind;
  opening: Bal; in: string; out: string; closing: Bal; openingNote: string | null };
type GroupRow = { groupId: string; groupName: string; nature: string | null;
  debit: string; credit: string; net: Bal; hasChildren: boolean };
type Check = { code: string; level: CheckLevel; facts: Record<string, unknown> };
type Tile = { code: string; amount?: string; vouchers?: number; drafts?: number;
  cancelledPairs?: number; opening?: Bal; net?: Bal; closing?: Bal };

type Summary = {
  scope: { companyName: string; branchName: string; fromDate: string; toDate: string;
           days: number; asOf: string };
  businessDay: { status: 'OPEN'|'CLOSING'|'CLOSED'|'NONE'|'MIXED'; zNo: string | null;
                 sessions: { total: number; closed: number } } | null;
  tiles: Tile[]; types: TypeRow[]; typeTotals: Omit<TypeRow, 'voucherTypeId'|'short'|'name'|'nature'|'category'|'loneHalves'>;
  moneyLedgers: MoneyRow[]; moneyTotals: Record<MoneyKind | 'ALL', { opening: Bal; in: string; out: string; closing: Bal }>;
  groups: GroupRow[]; groupTotals: { debit: string; credit: string };
  checks: Check[];
};
```

Unknown tile codes, check codes and voucher natures are **shown, not dropped**:
an unknown tile renders with its code as the heading; an unknown check renders
"<code>" with its level colour. The server adds codes; the client must not
hide them.

---

## 5. Loading

As the ledger statement §5: one **generation** per URL; the previous
generation is aborted and its late responses ignored; the old figures stay on
screen dimmed under a thin progress bar while the new day loads. That matters
most here: an accountant pressing ◀ five times to walk back a week must never
see Tuesday's tiles over Monday's grids. Summary tab = 1 request; Vouchers tab
= 1 per page; Day-wise = 1.

The tab not on screen is not fetched. Switching tabs fetches that tab for the
current URL (cached per generation).

---

## 6. The filter strip and header

- **Title row:** "Account Day Summary"; subtitle `company · branch · year ·
  <weekday> <date>` (a range: `dd-MM-yyyy → dd-MM-yyyy · n days`).
- **BusinessDayPill** (top right, from `summary.businessDay`): OPEN amber with
  "figures are provisional until Day Close"; CLOSING amber; CLOSED green with
  the Z no; NONE grey "no till day"; MIXED amber "branches differ"; `null` (a
  range) → no pill.
- **Fields:** ◀ Date ▶ (required), To, Company, Branch, Type, Show F5.
- **Checkboxes** — all four are **server parameters**, so changing one only
  arms Show (no client-side filtering):

| box | key | default |
|---|---|---|
| Count drafts | — shows the Draft column and the DRAFTS check; **display only** (the server always counts) | on |
| Include cancelled pairs | `includeCancelledPairs` on `/vouchers` only; the summary always leaves in-range pairs out (D2) | on |
| Clearing ledgers as bank | `clearingAsBank` | on |
| Zero rows | `zeroRows` | off |

"Count drafts" is the one display switch; label it so in the tooltip.

---

## 7. Summary tab

### 7.1 Tiles

Render `tiles[]` in the server's order; colour by code (SALES blue, SALE_RETURNS
teal, RECEIPTS green, PAYMENTS red, EXPENSES amber, CASH_IN_HAND accent). Sub
line from the counts: "41 bills · 1 cancelled pair", "4 posted · 1 draft",
"opening 48,250.00 · +21,600.00" (the `+`/`−` comes from `net.side` for a
cash ledger: DR = `+`, CR = `−` — the only place a sign appears, and it is the
server's side, not a subtraction). Click a tile → Vouchers tab with
`natureFilter` = that tile's code.

### 7.2 Voucher-type grid

Columns exactly as the mockup: Type (chip), Vouchers, Draft, Cancelled,
Amount, Cash in, Cash out, Bank in, Bank out; blank for zero; Total row from
`typeTotals` (never summed on the client). Chip = `TypeChip` by `nature`
(SALES blue, RECEIPT green, PAYMENT red, CONTRA blue, JOURNAL violet,
CREDIT_NOTE teal, DEBIT_NOTE teal, PURCHASE amber, STOCK_JOURNAL grey), text =
`name` upper-cased short form. Cancelled cell: "1 pair" in red; `loneHalves`
→ "+1 half" with a tooltip "a cancel whose other half is outside this range".
"Bank in / Bank out" header reads "Bank + clearing" when `clearingAsBank` is
on. Enter / double-click on a row → Vouchers tab, `type` = that id.

Footer hint (static): "Drafts are counted, never summed. Cancelled = the
original + its mirror, both in Vouchers, net 0 here."

### 7.3 Cash & bank position

`moneyLedgers[]`, grouped CASH → BANK → CLEARING in server order. Columns
Ledger, Opening, In, Out, Closing (bold); Total from `moneyTotals.ALL`.
`openingNote = COMPANY_LEVEL_ONLY` → an ⓘ on the opening cell: "this ledger's
opening is held at company level; pick All branches to include it". Enter on a
row → **Ledger Statement** (§9.3).

**Tie-out note** under the grid: "Cash in X · out Y" from `moneyTotals.CASH`,
"Bank in X · out Y" from BANK (+ CLEARING when the box is on). It is green with
✓ when the `AMOUNT_TIES` check is OK and red with the check's sentence when it
is not. The client does **not** compare the two grids itself.

### 7.4 Group summary

Columns Ledger group, Nature, Debit, Credit, Net (Dr blue, Cr green — the
mockup); Total from `groupTotals`; Net total = "0.00" when the BALANCED check
is OK. A **group level** selector in the section caption (Report groups /
Primary / Ledger groups → `groupLevel`) — a server parameter, refetches at once.
Enter on a row with `hasChildren` → `GroupLedgersDialog` (§9.2).

### 7.5 Checks

One card per `checks[]` entry: level → colour and mark (OK ✓ green, WARN !
amber, INFO i blue, FAIL ✕ red), heading + one line from `wire/checks.ts`:

| code | heading | line (from facts) |
|---|---|---|
| BALANCED | Dr = Cr | "4,95,270.00 = 4,95,270.00 — every posted voucher balanced" / FAIL: "n vouchers do not balance" |
| AMOUNT_TIES | Voucher amount = Σ Dr | "types grid total 4,95,270.00" / FAIL: "the report disagrees with itself — tell support" |
| LEGLESS | Vouchers with no entries | "n vouchers carry no ledger lines" |
| DRAFTS | n drafts not in figures | "pmt00088 · 15,000.00 · exv00041 · 640.00" (first 3) |
| CANCELLED_PAIRS | n cancelled pair(s) | "bil01431 / bil01432 (mirror) · net 0.00" |
| CLEARING_PENDING | Card clearing 38,400.00 | "settles T+1 (Settlement, menu 278)" |
| BUSINESS_DAY | Business day OPEN | "Day Close (menu 274) not run — tills 2 of 3 closed" |
| FUTURE_DATED | n vouchers dated after today | — |
| NUMBER_GAPS | Numbering | OK: "no gaps: bil01412–bil01453"; INFO: "Bil 3 missing (2 deleted, 1 unused)" |

Cards with a list behind them get a "show" link → `/checks/detail?code=` in a
small dialog (EnterNavigation installed, Esc closes). Order: FAIL, WARN, INFO,
OK — a broken day leads with what is broken.

---

## 8. Vouchers and Day-wise tabs

### 8.1 Vouchers (day book detail)

Paged, virtualised list (page 200), columns: ▸, Date, Type (chip), Voucher no,
Party / particulars ("(as per details)" when `asPerDetails`), Narration,
Amount, Branch (hidden for one branch), By. `rowKind` CANCELLED → struck
through grey + "· CANCELLED" on the chip; REVERSAL → red chip; `pairInRange`
pairs sit together (server order already does that). ▸ / Alt+F1 expands the
legs (`withLegs=true` for the page — one round trip, as ledger statement L4).
Footer: total amount from the response's `totals` + "n vouchers · m cancelled
pairs". A "Drafts" toggle switches the list to `draftsOnly=true` (no legs, no
number, doc amount shown greyed with "draft").

Filters arriving from drills (`type`, `nature`, `money`) show as removable
chips above the list.

### 8.2 Enter opens the voucher

`nav/drillTarget.ts`: `src.module` + `src.docType` → the source screen's route
with `src.docId` (sale bill, receipt, payment, expense voucher, voucher
register, opening stock, …); no `src` → the Voucher Register in view mode by
`voucherId` + `accYear`. Back returns to the same page and row. An unmapped
`src` → a toast "no screen opens this voucher type yet" and nothing else.

### 8.3 Day-wise

`/daily` rows: Date, Vouchers, Drafts, Cancelled, Amount, Cash in, Cash out,
Cash closing, Bank in, Bank out, Bank closing, Day (business-day status chip).
Enter on a day → Summary tab for that one day (URL write). The tab is enabled
only when To > Date; on a single day it shows "pick a To date to see a range".

---

## 9. Drill-downs

### 9.1 Map

| from | gesture | to |
|---|---|---|
| tile | click | Vouchers tab, nature filter |
| type row | Enter | Vouchers tab, type filter |
| money ledger row | Enter | Ledger Statement, that ledger, same range + branch |
| group row | Enter | GroupLedgersDialog |
| ledger in the dialog | Enter | Ledger Statement |
| voucher row | Enter | the source document (§8.2) |
| day row | Enter | Summary tab for that day |
| check card | show | check detail dialog |

### 9.2 GroupLedgersDialog

Modal, `/group-ledgers?groupId=` with the current scope; columns Ledger,
Group, Debit, Credit, Net; total row from the response. Enter → Ledger
Statement (closes the dialog, navigates). Esc closes.

### 9.3 Into the Ledger Statement

A URL, not a call: `/reports/ledger-statement?ledger=<id>&from=<date>&to=<to>&branch=<id>&tab=vouchers`.
Back returns here, on the same day and tab.

---

## 10. Export, print, WhatsApp

Excel: `/export` → `toXlsx.ts` builds two sheets: **Summary** (tiles as a
two-column block, then the three grids with their totals, then the checks as
sentences) and **Vouchers** (one row per leg when legs are present, voucher
columns repeated). Amounts written as numbers **parsed from the strings at the
last moment** for the cell value, with the display format `#,##,##0.00`; that
is the one sanctioned parse, and it never feeds a calculation. 422
`RANGE_TOO_LARGE` → "Too many vouchers to export (n). Narrow the range or pick
a type." Print / PDF / WhatsApp wait for `plan-react-printing.md` — show the
buttons disabled with that tooltip.

---

## 11. Errors → sentences (`wire/errors.ts`)

| code | sentence |
|---|---|
| `RANGE_OUTSIDE_YEAR` | "The dates must be inside the year <accYear>." |
| `RANGE_REVERSED` | "The To date is before the Date." |
| `BRANCH_NOT_IN_COMPANY` | "That branch is not in this company." |
| `YEAR_UNKNOWN` | "No financial year <accYear> for this company." |
| `TYPE_UNKNOWN` | "That voucher type no longer exists — reload the screen." |
| `NO_MENU_RIGHT` | "You do not have the Account Day Summary right." |
| other / network | "Could not load the day summary (<route>). Try Show again." |

Errors replace the figures with the sentence (never a blank screen or stale
figures without a warning).

---

## 12. Backend gaps this screen inherits

- Card / wallet tenders post straight to the bank ledger (backend §3.5), so on
  the test box the CLEARING rows will be near empty and `CLEARING_PENDING`
  rarely fires. Not a client bug.
- 12 legless receipts on 2026-09-29 → the LEGLESS check WARNs on that day.
- 24 future-dated vouchers → FUTURE_DATED INFO when the range includes today.
- Number gaps everywhere historically → NUMBER_GAPS is in-range only (D7).
- If any of D2, D3, D5 is decided differently, §7 changes with it; the
  client still sums nothing.

---

## 13. Tests

Unit (no React):
- `params.ts`: URL ⇄ filters round trip; ◀ on a range collapses to one day;
  ◀ at year begin is a no-op.
- `checks.ts`: every code × level from fixture facts; an unknown code renders.
- `parse.ts`: missing required amount → loud failure, never `"0.00"`.
- `drillTarget.ts`: each known `src` → route; unknown → null.
- `toXlsx.ts`: amounts as numbers with the Indian format; strings never
  `parseFloat`-ed before that point (grep test on `src/features/daySummary`).

Component (fixture = the mockup's example, which reconciles):
- the five grids render the fixture with **no** client-computed figure (spy on
  `money.format` inputs: every input is a string from the fixture);
- a stale generation's response is dropped (ledger statement's test, reused);
- tile click / row Enter write the expected URL.

Live (once the routes exist), against `ZT-DS-` data from backend §11: the
Summary money closing = the Ledger Statement closing for the same ledger and
day, read through both screens.

---

## 14. Phases

1. Filter strip, URL, `/summary` with the three grids + tiles; Excel of the
   Summary sheet.
2. Checks panel + check detail; business-day pill.
3. Vouchers tab with legs, drafts toggle, drill to source; Day-wise tab.
4. Group ledgers dialog + Ledger Statement drill; full Excel.
5. Print / PDF / WhatsApp after the printing plan lands.

---

## 15. Do not build

- No client arithmetic on money — no totals, no opening + in − out, no Dr = Cr
  comparison, no tie-out comparison between grids.
- No separate fetches per Summary panel (§1 fact 2).
- No call to `/reports/ledger-statement/*`, `/till/*`, `/vouchers/*`, the grid
  runner or the dropdown runner — §1 fact 1.
- No client-side hiding of cancelled vouchers.
- No sentences from the server rendered raw for checks — codes + facts → `checks.ts`.
