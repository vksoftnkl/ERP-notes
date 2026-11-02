# Party-wise Outstanding — backend plan (NestJS)

**For Prathap.** Written 2026-10-09 against the schema at 192.168.0.106/ERP and
the NestJS tree as it stands today (`nexerp-api-ro` snapshot). Where the schema
does not decide something it is marked **DECISION** and listed in §13.

That box is a TEST database: its catalog is authoritative, its rows are not.
Row counts quoted below only prove a *shape*.

Companion files in this folder (`Prathap/report/`):

    party_outstanding_ui_mockup.png   the screen: Party-wise / Bill-wise /
                                      Group-area summary / Due calendar tabs,
                                      filter strip, six tiles, party grid,
                                      bills of the selected party, party card
    party_outstanding_ui_mockup.py    regenerates it
    plan-react-party-outstanding.md   the client
    plan-backend-ledger-statement.md  the sister report; this module copies
                                      its shape (§11)

**The whole module in one line:** a read-only report over
`acc_bill_balance` + `acc_bill_adjustment` that answers *"what did each party
owe (or was owed) on date D, and how old is it"*. One SQL definition of
"pending as on D" (§4) drives every number. It is served from its own new
routes, and no existing URL is called.

---

## 1. What this is

Reports › Financial Statements › **Party-wise Outstanding**. It covers
receivables (customers: what they owe us) and payables (suppliers: what we owe
them) through one Side filter. It writes nothing: no table, no column, no DB
function, no trigger.

References:

- **3.0** `pos/Reports/customeroutstandings.*` (menu 97 "Customer
  Outstandings", Ctrl+Shift+O) and `supplieroutstandings.*` (menu 112). Three
  grids: customers (grid 578) → pending bills (579) → payments against the
  selected bill (580). Filters: company, city, area, type, salesman,
  customer, **collection day** (weekday), due days ≥ / ≤, created-before,
  only overdue / only balance / only advance / only nil balance. Buttons:
  Print list F6, Print bill balance F7 (a per-customer pending-bills
  statement with a message line), WhatsApp F8, SMS.
- **Tally**: Bills Receivable / Bills Payable, Ledger-wise Outstandings, the
  ageing analysis (Alt+F6 style buckets).

What 3.0 had that this keeps: the three-level drill (party → bills → what
settled a bill, §5.6), collection-day and salesman filters (§6), the
per-party pending-bills statement for WhatsApp (§5.9). What it drops: the
"customers created before N days" filter, and the SMS button (no SMS gateway
exists yet).

---

## 2. Routes are NEW and SELF-CONTAINED

The user's rule (2026-09-25): *"write into a separate urls dont use any old
urls because its gonna change bite later"*.

- Every route lives under **`/api/v1/reports/party-outstanding/…`** (§5).
- The report does **not** call or forward to `master-lookups/party-credit`,
  `/receipts/open-items`, `/payments/open-items`, `/account-ledger-masters/*`,
  the dropdown runner or the grid runner. It does not call
  `BillBalanceService.getCreditSummary()` either. That service has no as-on
  date, and it reads across years without excluding carried-forward parents
  (§3.2), so reusing it would import both problems.
- It may import Prisma models, `PgService` / `PrismaService`, and the shared
  decorators (`RequiredUuid`, `OptionalUuid`, `UpperMaxString`,
  `OptionalQueryBoolean`), the same building blocks ledger-statement uses.

---

## 3. Findings from reading the code — read BEFORE trusting a number

Three of these are defects or traps outside this module. The report has to
work around them, and they should be fixed where they live.

### 3.1 A sale bill's counter tender settles the bill with NO dated row

`bill-adjustment.helper.ts` `settleInvoiceAllocation()` (≈ line 614) **sets**
`abl_alloc_amount = tendered + adjusted`. The tendered part (cash/card/UPI
taken at the counter) has no `acc_bill_adjustment` row. Verified live:
**35 bills** have `alloc + disc + writeoff ≠ Σ live adjustment rows`, and all
of them are sale bills / sale returns settled at the counter (e.g. `bil00818`
1,180 alloc with zero rows; `bil00843` 680 alloc with zero rows; `srt00002`
472 refunded with zero rows).

Two consequences:

1. **As-on is not reconstructable from adjustment rows alone.** §4.3 derives
   the tendered part as `settled_now − Σ rows effective now` and dates it on
   the bill date. That is correct as long as (2) never happens.
2. **Suspect, please confirm:** `BillBalanceRecomputeService.recomputeBills()`
   rebuilds `abl_alloc_amount` from adjustment rows only (`ALLOCATING_TYPES`,
   ≈ line 100). It has no tender term. It is called from receipt/payment
   post, cancel and amend, cheque clear/bounce/return/reissue, voucher
   cancel and re-tender. If any of those touches a bill that also took a
   counter tender, recompute should **drop** the tendered part and re-open
   the bill. I did not find a guard. A test: post a part-tender credit bill,
   receive the rest with a receipt, cancel the receipt, then read
   `abl_alloc_amount`.

**Fix (recommended, DECISION O1):** the sale-bill / sale-return paths write
one `ALLOCATION` row per tender (with `abj_tender_id`, dated the bill date),
like receipts already do. Then the adjustment table is the whole truth,
recompute is safe, and §4.3's derived term disappears. Until then the report
uses the derived term and says so in the response (`tenderDerived: true` on
an affected bill).

### 3.2 Carry-forward makes a bill exist twice, and other readers double-count

`carry-forward.service.ts carryBills()` (≈ 346–616) creates, in the new year,
an `OPENING` bill for every still-open bill. It keeps the original refno, doc
date, due date and credit/grace days, and points back through
`abl_parent_bill_id / abl_parent_acc_year`. **The parent is not closed.** It
stays pending in the old year's partition.

`bill-balance.service.ts` (≈ 36–45), `get-party-credit-summary.dto.ts`
(≈ 44–46), `bill-balance-recompute.service.ts` (≈ 525) and
`receipts/open-items loadBills` (≈ 190–200) all read across every year with no
parent exclusion. **After the first carry-forward they count every carried
bill twice.** No carry-forward has run on the box yet (zero rows with a
parent), so nothing shows today. It will show on 1 April.

The report's rule is in §4.2 (the chain rule). The other readers need the same
rule. That is outside this module and should be a separate note.

### 3.3 Reversal rows carry the ORIGINAL date

Every reversal row has the same `abj_adj_date` as the row it reverses
(verified: 852 of 852). So cancelling a receipt dated 05-08 retracts its
settlement **on 05-08**, not on the cancel date. An as-on report for 31-08
printed before the cancel and one printed after it disagree.

That is the "books as they stand now" view (Tally works this way). It is the
right default. The report labels it with "as on D, as the books stand today".
A "what was reported on D" view (date reversals by `abj_created_on`) is
**DECISION O4**, and I recommend not building it.

### 3.4 Smaller facts that shape the queries

- **`abl_salesman_id` is never stamped** (0 of 1,422 rows). The salesman
  filter therefore uses `sales.customers.cus_default_salesman` (party level),
  not the bill. Note that 0 of 56 customers have a default salesman on the
  box either.
- **Due date can be NULL**: 19 SALES (credit days 0), 10 of 12 OPENING, every
  ADVANCE and SALES_RETURN. `receipt.utils daysOverdue()` treats NULL as
  "never overdue". The report must not do that, or cash-on-credit bills with
  zero credit days never age (§4.4).
- **The PDC rule is the adjustment date.** A post-dated cheque's rows carry
  `abj_is_post_dated = true` and `abj_adj_date = cheque date`. The recompute
  skips them while the date is in the future (recompute ≈ line 192). A plain
  `abj_adj_date <= D` therefore gives the PDC rule for free (§4.3). The
  `apd_status = 'HELD'` rule in older notes is superseded: every PDC on the box
  is `ON_RECEIPT`, and only the four future-dated rows (`jrl00321/343/365/387`,
  cheque date 15-10) are unsettled.
- **Credit limit exists** (`cus_credit_amt_limit`, `cus_credit_bill_limit`,
  `cus_credit_allowed`, `cus_debit_grace_days`). The mockup's "limit —"
  should show it. Suppliers have `sup_credit_days` only: no limit, no area.
- **Phone**: customers have `cus_phone1/2` (no "mobile" column); suppliers
  have `sup_phone`.
- **Party id = ledger id** for both customers and suppliers, so
  `abl_party_id` joins straight to either table and to
  `acc_ledger_master` (house rule, 2026-09-15).

---

## 4. The one definition of "pending as on D"

Every figure on the screen is built from this, and only this.

### 4.1 Inputs

`companyId` (required) · `asOn` D (required, `YYYY-MM-DD`) · `side`
(`RECEIVABLE | PAYABLE`) · `branchId` (optional, omitted = all branches
combined) · plus the party filters of §6.

`accYear` is **not** an input. D decides the year (`public.fiscal_years`
where `comp_id = companyId and D between fy_begin_date and fy_end_date`).
No row → `422 AS_ON_OUTSIDE_YEARS`. The bills read are every partition up to
and including that year (§4.2).

### 4.2 Which bills: the chain rule

A bill `b` is in scope when all of these hold:

    b.abl_company_id = :company
    b.abl_is_deleted = false
    b.abl_doc_date  <= :D
    b.abl_acc_year  <= year(:D)                  -- never a future year's copy
    (:branch IS NULL OR b.abl_branch_id = :branch)
    NOT EXISTS (                                  -- §3.2: superseded by its carry
      SELECT 1 FROM accounts.acc_bill_balance c
       WHERE c.abl_parent_bill_id   = b.abl_id
         AND c.abl_parent_acc_year  = b.abl_acc_year
         AND c.abl_bill_type        = 'OPENING'
         AND c.abl_is_deleted       = false
         AND c.abl_acc_year        <= year(:D))

So, for D in the new year after carry-forward, the child is read and its
parent is not. For D in the old year, the child (a later year) is out of scope
and the parent is read with its own adjustments. Before carry-forward has run,
old-year bills are simply read from their own partition. **INTEREST** bills
also have a parent, but they are not copies, so the rule only applies to
`OPENING` children.

### 4.3 How much of a bill is pending on D

    settled_rows(b, D) = Σ abj_amount
                           over acc_bill_adjustment a
                          where a.abj_bill_id = b.abl_id
                            and a.abj_bill_acc_year = b.abl_acc_year
                            and a.abj_is_deleted = false
                            and a.abj_adj_date <= :D
        -- every adj type counts: ALLOCATION, ADVANCE_ADJUST, NOTE_ADJUST,
        -- TRANSFER, DISCOUNT, ROUND_OFF, WRITEOFF.
        -- Reversal rows are negative and carry the original date (§3.3), so
        -- they cancel in the same sum. No "is reversed" join is needed.
        -- Post-dated rows carry the cheque date, so a future cheque drops out.

    tendered(b) = (abl_alloc_amount + abl_disc_amount + abl_writeoff_amount)
                  − settled_rows(b, today)                         -- §3.1
                  → counted as settled on b.abl_doc_date; 0 once O1 is fixed

    pending(b, D) = abl_bill_amount − settled_rows(b, D)
                    − (tendered(b) if b.abl_doc_date <= D else 0)

A bill with `pending(b, D) = 0` is not listed. A negative `tendered(b)`
(rows exceed the cached total) is a data fault: log it once per request,
clamp to 0, and set `dataWarning: 'ALLOC_BELOW_ROWS'` on the bill row.

**Candidate set (performance, not semantics):** a bill can be pending on D only
if it is pending now, or it has an adjustment row dated after D. So the scan
starts from

    (b.abl_pending_amount <> 0)                         -- ix_abl_open
    OR EXISTS (adjustment on b with abj_adj_date > :D)  -- ix_abj_bill

and then applies §4.3. When D = today, that reduces to the `ix_abl_open`
partial index plus the few future-dated PDC rows.

### 4.4 Owed vs on-account, and ageing

The **owed side** depends on Side: Receivable → `abl_dr_cr = 'DR'`; Payable →
`'CR'`. Bills on the other side are **on-account credit** (advances, sales
returns, credit notes, unallocated receipt or payment remainders, and a
dual-role party's purchase bills on the receivable view). They are listed and
netted. They are **never aged and never overdue**.

    netOutstanding(party) = Σ pending(owed) − Σ pending(on-account)
                            → shown with its side (Dr/Cr); never a minus sign

Due date for ageing and overdue (DECISION O2):

    dueEff(b) = COALESCE(b.abl_due_date, b.abl_doc_date + b.abl_credit_days)
    overdue on D  ⇔  D > dueEff(b) + b.abl_grace_days
    overdueDays   = D − dueEff(b) − abl_grace_days   (when > 0)

`ageBy = BILL_DATE` (default): `age = D − abl_doc_date`.
`ageBy = DUE_DATE`: `age = D − dueEff(b)`; `age < 0` goes to a **Not due**
bucket, which then becomes the first bucket.

Buckets: `buckets=30,60,90,180` (1 to 6 ascending positive integers) gives
`0–30 · 31–60 · 61–90 · 91–180 · >180`. Validate: strictly ascending, max
3650, otherwise `422 BAD_BUCKETS`. The server returns the bucket labels with
the data, so the client never builds them.

### 4.5 PDC — two different numbers

- **PDC in hand (not yet effective):** `acc_pdc_register` with
  `apd_tra_type = 'R'` (Receivable) or `'P'` (Payable),
  `apd_status IN ('HELD','DEPOSITED')`, `apd_instrument_date > :D`,
  `apd_received_on <= :D`, not deleted, same company/branch scope. These have
  **not** reduced pending (their rows are future-dated), so `deductPdc=true`
  may subtract them from net. This is the PDC column and tile.
- **Uncleared but already effective:** the same statuses with
  `apd_instrument_date <= :D`. They **already** reduced pending through their
  rows. Deducting them again would count them twice. They are shown on the
  party card only, as "uncleared, already counted".

---

## 5. The routes

All `GET`, all `@Version(API_VERSION)`, `@CacheTTL(0)`, `{success, message,
data}`, under `@Controller('reports/party-outstanding')`. Common query (DTO
`OutstandingScopeDto`): `companyId`\*, `asOn`\*, `side`\*, `branchId`,
`groupId`, `areaId`, `collectionDay` (`MON`..`SUN`), `salesmanId`,
`partyId`, `ageBy`, `buckets`, `onlyOverdue`, `includeOnAccount` (default
true), `deductPdc` (default false), `hideZero` (default true),
`minDueDays` / `maxDueDays` (3.0's "due days ≥ / ≤", on overdue days).

Amounts are **strings with two decimals**. A balance is `{amount, side}` where
`side` is `'DR' | 'CR' | null` (null only on 0.00). Dates are ISO
`YYYY-MM-DD`.

| # | Route | Purpose |
|---|---|---|
| 5.1 | `GET …/options` | filter sources: party groups, areas, salesmen, branches |
| 5.2 | `GET …/parties` | party grid page + totals + the six tiles |
| 5.3 | `GET …/party` | party card: facts, ageing, PDC, last settlement |
| 5.4 | `GET …/bills` | open items of one party (the bottom-left grid) |
| 5.5 | `GET …/bill-wise` | Bill-wise tab: open items across parties, paged |
| 5.6 | `GET …/bill-history` | what settled one bill (3.0's payments grid) |
| 5.7 | `GET …/summary` | Group / area summary tab |
| 5.8 | `GET …/due-calendar` | Due calendar tab |
| 5.9 | `GET …/export` | rows for Print / PDF / Excel / WhatsApp |

### 5.1 `GET …/options?companyId&side`

    { groups:   [{ groupId, name, depth, isDefault }],   // party groups under Sundry
                                                          // Debtors (R) / Creditors (P), tree order
      areas:    [{ areaId, name, collectionDays: ['MON','THU'] }],   // empty for PAYABLE
      salesmen: [{ salesmanId, name }],                               // empty for PAYABLE
      branches: [{ branchId, name }] }

`isDefault` marks Sundry Debtors / Sundry Creditors, the screen's default
group. "Group" means the **account group** (`acc_group_master`) and includes
sub-groups, because that is what decides receivable vs payable in the books.
It is not `cust_groups`. (Customer type, 3.0's "Type", is DECISION O6.)

### 5.2 `GET …/parties` + `sort`, `dir`, `page`, `pageSize` (≤ 500, default 200)

    { asOn, side, bucketLabels: ['0–30','31–60','61–90','91–180','> 180'],
      tiles: {
        net:        { amount, side }, parties, bills,
        overdue:    "403950.00", overduePctOfOwed: "57",
        aboveDays:  { days: 90, amount, parties },     // the 3rd tile; 90 = the
                                                        // 3rd bucket edge
        onAccount:  "21500.00",
        pdcInHand:  { amount, cheques },
        dueNext:    { days: 7, amount, from, to } },
      rows: [{
        partyId, name, area, phone, creditDays, creditLimit: "200000.00"|null,
        bills,                      // owed-side open bills
        owed:      "180300.00",
        onAccount: "4000.00",
        net:       { amount: "176300.00", side: "DR" },
        buckets:   ["28650.00","92650.00","0.00","59000.00","0.00"],
        overdue:   "170500.00",
        oldestDays: 178,
        pdcInHand: "20000.00",
        flags:     ['CHQ_BOUNCED','OVER_LIMIT','OVER_180','ADVANCE'] }],
      totals: { bills, owed, onAccount, net, buckets[], overdue, pdcInHand },
      page:   { page, pageSize, totalRows } }

- **Totals and tiles are over ALL rows that pass the filter, not the page.**
- `sort` ∈ `net | name | overdue | oldest | owed | bucket0..bucket5`, `dir`
  ∈ `asc | desc`, default `net desc`. A tie breaks on name, then partyId,
  so paging is stable.
- `flags`: `CHQ_BOUNCED` = the party has an `acc_pdc_register` row (same
  `apd_tra_type`) with `apd_status = 'BOUNCED'` and `apd_bounce_date <= D`.
  A bounce does **not** create a bill for the cheque amount. It re-opens the
  bills the cheque had settled (negative reversal rows, reason `"Cheque …
  bounced: …"`), and `cheque-bounce.service writeChargeBill()` adds a
  separate JOURNAL bill `BNC/<chq no>` (`abl_src_doc_type =
  'CHEQUE_BOUNCE_CHARGE'`, due at once) for the charges only;
  `OVER_LIMIT` = `creditLimit > 0 and net(DR) > creditLimit` (Receivable
  only); `OVER_180` = oldest age is past the last bucket edge; `ADVANCE` =
  net is on the on-account side.
- `hideZero=true` drops parties whose owed and on-account are both 0 on D.
  `onlyOverdue=true` keeps parties with overdue > 0 (and, in 5.4, overdue
  bills only).
- The `dueNext` tile = Σ owed pending with `dueEff` in `(D, D+7]`.

### 5.3 `GET …/party?…&partyId`

    { party: { partyId, name, ledgerGroup, area, phone, gstin, creditDays,
               creditLimit, creditBillLimit, isDualRole },
      ageing: { labels[], amounts[] },  owed, onAccount, net,
      onAccountItems: [{ billId, accYear, docRefno, date, type, amount }],
      pdcInHand:  [{ pdcId, accYear, chequeNo, bank, chequeDate, amount, status }],
      pdcEffectiveUncleared: [{ … same … }],                  // §4.5, info only
      lastSettlement: { date, voucherNo, voucherType, amount } | null }

`lastSettlement` = the latest non-reversed `ALLOCATION` row on an owed bill of
this party with `abj_adj_date <= D`, grouped by voucher. The mockup labels it
"Last receipt" for Receivable and "Last payment" for Payable.

### 5.4 `GET …/bills?…&partyId`

One row per pending bill of the party on D, both sides, ordered by
`abl_doc_date, abl_doc_refno`:

    { rows: [{ billId, accYear, branchName, docDate, docRefno, billType,
               srcDocType, srcDocId, srcAccYear, voucherId,
               side: 'OWED'|'ON_ACCOUNT', dueDate: string|null, dueEff,
               billAmount, adjusted, pending, ageDays, overdueDays: number|null,
               remarks, tenderDerived: boolean, dataWarning?: string }],
      totals: { billAmount, adjusted, net: { amount, side } },
      ledgerClosing: { amount, side } }

- `adjusted` = `billAmount − pending` on D (includes the derived tender).
- `remarks` is server-built: "part rct00002" from the latest allocation's
  voucher, "disc jv00007", "on account", "re-opened · chq bounced" when the
  latest reversal on the bill has a bounce reason, "bounce charge" for a
  `BNC/` bill. The client prints it as is.
- **`ledgerClosing`** is the party ledger's balance on D, computed with the
  ledger-statement balance definition (POSTED + CANCELLED legs + opening). The
  mockup prints "= ledger closing ✓" when it equals `totals.net`. A mismatch
  is a real finding: some posting moved the ledger without a bill, or the
  other way round (the notes-47 reconcile). The client shows a red "≠ ledger
  1,80,300.00 Dr" in its place. It must not hide it.

### 5.5 `GET …/bill-wise` + `sort`, `page`, `pageSize`

The same row shape as 5.4, plus `partyId, partyName, area`, across every
party in the filter. Owed-side bills only, unless `includeOnAccount`. Totals
over all rows. Extra parameter `dueOn` (ISO date): only bills whose `dueEff` equals it.
The Due calendar's day click uses it.

### 5.6 `GET …/bill-history?companyId&billId&accYear&asOn`

    { bill: { docRefno, docDate, billAmount, pending },
      rows: [{ date, adjType, voucherId, voucherAccYear, voucherNo, voucherType,
               againstDocRefno, amount, isReversal, reversalReason,
               isPostDated, chequeNo, effective: boolean }],
      tenderAtBill: "500.00" | null }

Every adjustment row on the bill, in `abj_adj_date, abj_row_no` order,
including reversals (signed). `effective = abj_adj_date <= asOn`.
`tenderAtBill` is §4.3's derived term, as its own line ("paid at counter").
This is 3.0's payments grid (580).

### 5.7 `GET …/summary?…&groupBy=AREA|GROUP|SALESMAN|BRANCH`

Rows of `{ key, name, parties, owed, onAccount, net, buckets[], overdue }` +
totals. `AREA` and `SALESMAN` are refused with `422 NOT_FOR_PAYABLE` on the
Payable side.

### 5.8 `GET …/due-calendar?…&from&to` (≤ 92 days)

`{ days: [{ date, amount, bills, parties }], overdueBefore: { amount, bills } }`.
Owed side, pending on `asOn`, grouped by `dueEff` in `[from, to]`.
`overdueBefore` is the lump already past due.

### 5.9 `GET …/export?…&shape=PARTIES|BILLS|PARTY_STATEMENT`

JSON rows only (the same contract as ledger-statement export). The client
prints and builds the workbook. Cap 20,000 rows, else
`422 RANGE_TOO_LARGE`. `PARTY_STATEMENT` needs `partyId`. It returns the
party header, its open bills (5.4 rows), ageing, net and `printedAs` (the
applied filters as one sentence, as in loyalty-status). That is the "Print
bill balance F7" / WhatsApp reminder document of 3.0.

---

## 6. Filters — where each one lands in SQL

| Filter | Receivable | Payable |
|---|---|---|
| `groupId` | party ledger's group ∈ subtree(groupId) | same |
| `areaId` | `customers.cus_area_id` | refused (`NOT_FOR_PAYABLE`) |
| `collectionDay` | `area_master.arm_collection_days @> {day}` (3.0) | refused |
| `salesmanId` | `customers.cus_default_salesman` (§3.4) | refused |
| `partyId` | `abl_party_id` | same |
| `branchId` | `abl_branch_id` | same |
| `minDueDays/maxDueDays` | on `overdueDays` of owed bills | same |

A party with no customer row (a bill-wise ledger that is not a customer, e.g.
a staff ledger) still appears under its group, with blank area. A supplier
with a customer twin (dual role) appears on both sides. On each side, the
other side's bills are on-account. That is DECISION O3.

---

## 7. The main query (for `/parties`)

Sketch only; one CTE chain, `$queryRaw` tagged template, no string
concatenation of values:

    WITH fy AS (...year(:D)...),
    cand AS (                       -- §4.3 candidate set + §4.2 chain rule
      SELECT b.* FROM accounts.acc_bill_balance b
       WHERE b.abl_company_id = $company AND NOT b.abl_is_deleted
         AND b.abl_doc_date <= $D AND b.abl_acc_year <= fy.name
         AND ($branch::uuid IS NULL OR b.abl_branch_id = $branch)
         AND (b.abl_pending_amount <> 0 OR EXISTS (SELECT 1 FROM accounts.acc_bill_adjustment a
               WHERE a.abj_bill_id = b.abl_id AND a.abj_bill_acc_year = b.abl_acc_year
                 AND NOT a.abj_is_deleted AND a.abj_adj_date > $D))
         AND NOT EXISTS (… OPENING child … §4.2)),
    rows AS (
      SELECT c.abl_id, c.abl_acc_year,
             COALESCE(SUM(a.abj_amount) FILTER (WHERE a.abj_adj_date <= $D), 0)     AS set_d,
             COALESCE(SUM(a.abj_amount) FILTER (WHERE a.abj_adj_date <= $today), 0) AS set_now
        FROM cand c LEFT JOIN accounts.acc_bill_adjustment a
          ON a.abj_bill_id = c.abl_id AND a.abj_bill_acc_year = c.abl_acc_year
         AND NOT a.abj_is_deleted
       GROUP BY 1,2),
    bill AS (… pending(b,D), owed/on-account by side, dueEff, age, bucket index, overdue …),
    party AS (… GROUP BY abl_party_id, join ledger/customer/area for name & filters …)
    SELECT … ORDER BY $sort LIMIT $pageSize OFFSET …;
    -- tiles/totals: same CTEs, aggregated without LIMIT (one round trip:
    -- a second SELECT in the same transaction, or window SUM() OVER ()).

`$today` = the server's business date (IST), not `CURRENT_DATE` in UTC.

### 7.1 Indexes

What exists is enough for phase 1:

- `ix_abl_open (company, party, doc_date) WHERE pending <> 0`: the D = today
  path.
- `ix_abj_bill (bill, bill_acc_year, adj_date)`: the per-bill sums and the
  "row after D" test.
- `ix_abl_parent`: the chain rule.

The gap: for D in the past, bills that are closed now but were open on D come
in only through the `EXISTS (row after D)` arm. That arm walks bills by
company. If a past-date run on a real store is slow, add
`ix_abj_company_date (abj_company_id, abj_adj_date) INCLUDE (abj_bill_id,
abj_bill_acc_year) WHERE NOT abj_is_deleted` and drive the arm from it. Do
not add it before measuring.

---

## 8. Rules only the service can enforce

- `assertMenuRight(menuId)` against `public.user_menus` (as ledger-statement
  does). Receivable and Payable are the same menu unless DECISION O5 splits
  them.
- `companyId` must be a company the user can see. `branchId` must belong to
  it (`public.branch_master`). `partyId` must have a bill or a ledger in the
  company.
- `asOn` must fall inside a fiscal year of the company. A future date is
  allowed: it is how "what will be due once these cheques mature" is read.
  It is labelled `isFuture: true`.
- Error codes: `AS_ON_OUTSIDE_YEARS`, `BAD_BUCKETS`, `NOT_FOR_PAYABLE`,
  `RANGE_TOO_LARGE`, `PARTY_NOT_IN_COMPANY`, `BRANCH_NOT_IN_COMPANY`,
  `NO_MENU_RIGHT`. All 422 except the last (403).

---

## 9. Numbers that must agree — the cross-checks the tests assert

1. Σ `/bills.totals.net` for a party = that party's row `net` in `/parties`.
2. Party `net` on D = the party ledger's closing on D (ledger statement), for
   any bill-wise party whose every posting carries a bill (§5.4
   `ledgerClosing`). It is the mockup's Sri Krishna Traders check:
   1,80,300 − 4,000 = 1,76,300 Dr.
3. For D = today and no PDC in the future: Σ `pending` per bill =
   `abl_pending_amount`, except the bills §3.1 lists (they agree too, through
   the derived tender).
4. `/parties` with `branchId` omitted = Σ over each branch's `/parties`.
5. After a receipt is cancelled, the as-on report for a date before the cancel
   changes (§3.3). The test asserts it, so the behaviour is deliberate.
6. After a carry-forward (test DB with two years): the as-on report for
   the new year shows each carried bill once (§3.2).

---

## 10. Menu and registration

- New `fixed.menu_master` row: **"Party-wise Outstanding"**, parent **137**
  (Financial Statements), position **0.60** (just after Ledger Statement
  258). Visible. The next free id is 279 on the box; Prathap assigns it.
- No `ui_table` / grid registration for data: the data comes from these
  routes (the loyalty-status pattern). If the client wants saved column
  widths, register LAYOUT-ONLY ui_tables later.
- Register `ReportsPartyOutstandingModule` in `app.module.ts` beside
  `LedgerStatementModule`.

---

## 11. Module layout (mirror ledger-statement)

    src/modules/reports/party-outstanding/
      party-outstanding.module.ts       // imports no other feature module
      party-outstanding.controller.ts   // 9 GETs (§5)
      party-outstanding.service.ts      // scope, menu right, orchestration
      party-outstanding.sql.ts          // the §4/§7 CTEs as tagged-template builders
      party-outstanding.ageing.ts       // buckets, dueEff, labels — pure, unit-tested
      dto/party-outstanding-query.dto.ts// Scope → Party → Paged → Export, class-validator
      types/party-outstanding.types.ts
      README.md                         // §4 copied in, so the rule lives beside the code

---

## 12. Test checklist

Unit (`party-outstanding.ageing.spec.ts`): bucket edges and labels; Not-due
bucket for DUE_DATE; dueEff with NULL due and 0 credit days; grace days;
`BAD_BUCKETS` cases.

Live, against ZT- data on the box (a script beside `tests/`, the same style as
`tests/loyaltystatus/run-api.py`):

1. A credit bill with no settlement → owed, aged, overdue after credit days.
2. A part receipt → pending drops on the receipt date, not before.
3. The receipt cancelled → the pending returns, **including for dates before
   the cancel** (§3.3).
4. A post-dated cheque dated D+5 → not settled on D, settled on D+5; PDC in
   hand on D, not on D+5.
5. A cheque bounce → the settled bill re-opens (pending returns on the
   cheque's date, §3.3), a `BNC/` charge bill appears, and the `CHQ_BOUNCED` flag is set.
6. A sales return adjusted against a bill → NOTE_ADJUST pair, and both sides
   net.
7. An unallocated receipt remainder → ADVANCE on-account; the party nets to
   Cr and gets `ADVANCE`.
8. A counter-tendered part payment (`bil00843` shape) → `tenderDerived: true`,
   pending = bill − tender, on and after the bill date.
9. A dual-role party → its purchase bill is on-account on Receivable, and its
   sale bill is on-account on Payable.
10. §9 cross-checks 1–4.
11. Paging: totals equal across pages, sort stable with ties.
12. Payable side refuses area/salesman/collectionDay with `NOT_FOR_PAYABLE`.

---

## 13. Decisions

| # | Question | Recommendation |
|---|---|---|
| O1 | Counter tender writes no adjustment row (§3.1) | Write one ALLOCATION row per tender at bill post (sale bill + sale return). Report uses the derived term until then. Also confirm the recompute suspicion. |
| O2 | Due date when `abl_due_date` is NULL | `doc_date + credit_days`, plus grace for overdue. Differs from `receipt.utils daysOverdue` (NULL = never). |
| O3 | Dual-role party | Shown on both sides; the other side's bills are on-account (Tally's view). |
| O4 | "As reported on D" (reversals dated by created_on) | Do not build. Books-as-they-stand only, labelled. |
| O5 | One menu, or Receivables + Payables as two menus (3.0 had 97 + 112) | One menu with the Side combo. Split only if a role must see one side and not the other. |
| O6 | 3.0's customer "Type" filter (cust_groups) | Add later as `custGroupId`, Receivable only; not in phase 1. |
| O7 | The carry-forward double count in the OTHER readers (§3.2) | Separate notes file; not this module's job, but it lands on 1 April. |

---

## 14. Do not build

- No DB function, view or trigger (house rule: no SQL logic in the DB).
- No write route. No "send reminder" route: WhatsApp is client-side from
  `/export?shape=PARTY_STATEMENT`. SMS waits for a gateway.
- No reuse of `getCreditSummary`, open-items or the grid runner (§2).
- No client-supplied bucket labels, sides or totals: the server sends them.
- No `accYear` input (D decides it, §4.1).
