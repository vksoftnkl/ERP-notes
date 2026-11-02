# Voucher Register: the whole design in one file

**2026-09-25. Design only: nothing here has been run.** No SQL, no routes, no
Qt code. This single file replaces an earlier rev 1 set (spec, backend plan, Qt plan,
two PNGs), which was never executed and has been deleted. It was read against the live schema on
192.168.0.106 and the NestJS tree (`nexerp-api-ro`) the same day. Where it
names an existing column, helper or function, that thing exists today.

**The screen:** `voucher_register_ui_mockup_rev2.png` (its `.py` beside it),
approved by the user on 2026-09-25 with decisions A–E.

| Read | For |
|---|---|
| §1–§4 | everyone: the model, the decisions, the types, worked examples |
| §5–§8, §10, §11.1 | **Prathap**: migration, routes, posting routine, lifecycle, grids, server tests |
| §9, §11.2 | **the Qt client** |
| §12 | the order of work |

> **The whole module in one line:** one register screen for every accountant
> voucher, and **one posting routine** behind it that Receipt, Payment and
> (later) the goods documents also call. The TYPE row carries the rules, the
> client sends only the lines the operator typed, and the server works out
> every tax, TDS and party leg itself.

---

## 1. The model

### 1.1 One register, not a form per voucher

Journal, Contra, Debit Note, Credit Note, the accountant's Receipt / Payment,
and Purchase / Sales **with no items** are the same form:

- a header;
- n Dr/Cr legs;
- bill-wise allocation on the party leg;
- an optional GST band.

What differs is **rules**: which ledgers each side may use, whether a party
is required, whether bill-wise is demanded or raises a bill, which GST
register it feeds, whether TDS is deducted. Rules are data on the voucher
type, so they are one screen. That is how TallyPrime (one Accounting Voucher
screen, F4–F10 switch the type) and GoFrugal (one form, the type the only
variation) both do it. Separate forms would be six screens, six route
families, six posting codes and six test suites that drift apart.

**"Purchase" and "Sales" here are not goods documents.** They have no item,
batch, godown or stock. That makes them the home for every expense, service,
fixed asset, reverse-charge and import-of-services entry, which is why NexERP
needs no separate GST-expense screen.

### 1.2 Which screens stay separate, and the rule that decides

A form gets its own screen **only if its workflow has objects the legs model
does not**: tenders, instruments, items, stock.

| Form | Where |
|---|---|
| Journal, Contra, Debit Note, Credit Note | **register** |
| Receipt / Payment in voucher mode (accountant, free legs) | **register** |
| Purchase / Sales (accounting: expense, service, asset, RCM) | **register** |
| Bill-wise Receipt (menu 99), Payment (menu 100) | own screens (tenders, cheque register, credits, TDS slabs, counter print), **same routine** |
| Sale bill, sale return, purchase with items | own screens (items, stock); legs already go through the shared leg writer |

### 1.3 One posting routine

Register, Receipt and Payment each map their own request into one internal
shape, **`{header, legs[], allocations[], gst?, tds?, bill?}`**, and hand it
to one server routine. That routine owns:

- numbering, the Dr = Cr check, the instrument-ledger guard;
- raising and settling bills;
- the GST document and the TDS register;
- cancel by reversal, and the day/period locks.

It is written once, and the balance check exists once.

---

## 2. Decisions

| # | Decision | State |
|---|---|---|
| **A** | A Contra carries **no** party; a Journal may carry **several**. `avh_party_id` becomes nullable (it is `NOT NULL` today); each party leg keeps its own bill-wise | approved |
| **B** | **The GST band generates the tax legs.** The operator types only the lines, with a rate and HSN/SAC on each; input/output tax legs are generated and locked. Reverse charge flips them to input + RCM payable | approved |
| **C** | **Cancel = a reversal under the `Rev` type**, which Prathap shipped on 2026-09-25 for the sale bill (its own series `rev00001…`, narration "Reversal of <refno>", linked both ways). The original's series never skips a number; the cancelled voucher stays listed | approved |
| **C2** | Reversal numbering and date | **decided 2026-09-25: keep the LIVE rule.** A reversal is a `Rev` voucher in its own series (`rev000NN`), dated **the original's date**, narration "Reversal of <refno>". The rev 2 PNG's `<refno>/R` / cancel-day drawing is superseded |
| **D** | **One screen, many menus.** Debit Note (101), Credit Note (102), Journal (103), Contra (104), Purchase (Accounting) (163, renamed) and three new menus open it with the type fixed; a new *Voucher Register* menu opens it with every permitted type on the band | approved |
| **E** | **Rights per voucher TYPE, through its own menu.** Each type points at a menu; the user's `user_menus` rights on that menu decide view / create / edit / delete / post / cancel / print; the band shows only permitted types; the server checks every call | approved |
| **TDS** | Deducted by us is in scope (S2 closed): the party's section adds a `TDS_PAYABLE` leg on Purchase (acct) and Payment (voucher), and the party is owed the net | approved |
| S20 | Cash ledger per branch, or per counter | **open**: master data only, does not hold the build |
| S1 | Cost centres | open: `av_cost_centre_id` stays NULL |
| R1 | Amend a posted voucher | **not in v1**: cancel and re-enter |
| R2 | The sale bill's D3 (a cancel burned the next bill number) | **done by Prathap 2026-09-25**: migration `20260925150000_reversal_voucher_type`; existing mirrors re-numbered into the `rev` series |

---

## 3. The types, their rules, their menus

| Code | Type | Party | Party side | Bill-wise | Raises | GST | TDS | Menu |
|---|---|---|---|---|---|---|---|---|
| `Jrl` | Journal | MANY | any | optional | JOURNAL bill if the party is bill-by-bill | — | — | 103 |
| `Con` | Contra | NONE | — | off | — | — | — | 104 |
| `DrN` | Debit Note | ONE | DR | raise | JOURNAL | output, GSTR-1 9B | — | 101 |
| `CrN` | Credit Note | ONE | CR | raise | JOURNAL | output, GSTR-1 9B | — | 102 |
| `PurA` | Purchase (Accounting) | ONE | CR | raise | PURCHASE | input, GSTR-2B / ITC | deduct | 163 |
| `SalA` | Sales (Accounting) | ONE | DR | raise | SALES | output, GSTR-1 | — | new |
| `RcpV` | Receipt Voucher | ONE | CR | demand | — | — | — | new |
| `PmtV` | Payment Voucher | ONE | DR | demand | — | — | deduct | new |
| `Rev` | Reversal (hidden) | as the original | — | off | — | — | — | the original's |

**Legal ledgers per side** (group, or any sub-group under it). The names are
the live groups; there is no "Expenses" / "Income" group on the box, only the
Tally set.

| Type | Debit side | Credit side |
|---|---|---|
| Contra | Cash-in-Hand, Bank Accounts, Bank OD A/c | same |
| Debit Note | any | Direct / Indirect Incomes, Sales Accounts, Duties & Taxes |
| Credit Note | Direct / Indirect Incomes, Sales Accounts, Direct / Indirect Expenses, Duties & Taxes | any |
| Purchase (acct) | Direct / Indirect Expenses, Purchase Accounts, Fixed Assets, Duties & Taxes | Sundry Creditors, Duties & Taxes |
| Sales (acct) | Sundry Debtors | Direct / Indirect Incomes, Sales Accounts, Duties & Taxes |
| Receipt Voucher | Cash-in-Hand, Bank Accounts | any |
| Payment Voucher | any | Cash-in-Hand, Bank Accounts |
| Journal | any | any |

**On every type:**

- **No leg on an instrument-controlled ledger** (Cheques in Hand, PDC holding,
  card/UPI clearing): those belong to Received / Issued Cheques (51 / 52).
- **No inventory**: the register never moves stock.

**Rights example (E):**

| User | Granted | Sees |
|---|---|---|
| Cashier | Contra: create + post | the register opens with Contra only |
| Accounts assistant | Journal, DN, CN: create | saves drafts, cannot post |
| Accountant | everything except Purchase (acct), Sales (acct) | those two tabs are absent, and the server refuses them anyway |
| Auditor | view on all | reads everything, no Save / Post |

---

## 4. Worked examples, through the tables

### 4.1 Purchase (accounting): the PNG example

**PurA, 15-09-2026.** Sundaram Facility Services, bill SFS/2026/451 dated
12-09-2026: housekeeping 25,000 @ 18% (SAC 998533) + stationery 4,000 @ 12%
(HSN 4820). The supplier's TDS is 194C, 2%, on the service only.

| Row | Dr/Cr | Ledger | Amount | Typed? | Role | From |
|---|---|---|---|---|---|---|
| 1 | DR | Housekeeping Charges | 25,000.00 | typed | — | |
| 2 | DR | Printing & Stationery | 4,000.00 | typed | — | |
| 3 | DR | Input CGST | 2,490.00 | generated | `INPUT_CGST` | 2,250 @ 9 + 240 @ 6 |
| 4 | DR | Input SGST | 2,490.00 | generated | `INPUT_SGST` | same |
| 5 | CR | TDS Payable — 194C | 500.00 | generated | `TDS_PAYABLE` | 2% × 25,000 (row 1 only) |
| 6 | CR | Sundaram Facility Services | 33,480.00 | generated | — | the balancing party leg |
| | | **Σ** | **33,980.00 = 33,980.00** | | | |

What else posting writes:

- **Supplier bill:** one `abl_` PURCHASE bill, 33,480.00 Cr, due +30 days
  (the supplier's credit days). It is the first thing Payment (menu 100) can
  ever pay.
- **GST document:** one `acc_voucher_doc_register` row with `gdr_doc_type
  PURCHASE`, and two `acc_voucher_doc_detail` rows (one per typed line;
  `vtx_row_no` = the line's row; ITC: input services / inputs).
- **TDS register:** one `acc_tds_register` row (194C, base 25,000, tax 500).

**Reverse charge on:** rows 3–4 are DR Input / CR RCM payable, and the party is
owed 29,000 − 500.

### 4.2 Credit Note with tax

**CrN, 15-09-2026.** A rate difference of 4,000 + 18% is allowed to Sri Krishna
Traders against bill bil/00214.

| Row | Dr/Cr | Ledger | Amount | Typed? |
|---|---|---|---|---|
| 1 | DR | Rate Difference Allowed | 4,000.00 | typed, 18% |
| 2 | DR | Output CGST | 360.00 | generated. **Output tax on DEBIT**, because a credit note reduces the tax the customer was charged |
| 3 | DR | Output SGST | 360.00 | generated |
| 4 | CR | Sri Krishna Traders | 4,720.00 | generated party leg |

What else posting writes:

- **The note's own bill:** a JOURNAL `abl_` bill, CR.
- **The allocation to bil/00214:** the pair of `abj_` NOTE_ADJUST rows on the
  party leg (`abj_voucher_line_id` = row 4). Unallocated, the note stays an
  OPEN credit that the Receipt screen's credits band reads.
- **GST document:** `gdr_doc_type CREDIT_NOTE` (GSTR-1 9B). To a registered
  party it is e-invoice applicable, but the e-invoice screen calls the
  portal, not the register.

**A credit note with no GST lines writes no GST document.** It is a financial
note and must never reach GSTR-1 (S21).

### 4.3 Contra

Counter drawer to the bank: CR Cash — Counter 1, 85,000 / DR HDFC Bank —
004312, 85,000.

- No party (`avh_party_id` NULL).
- No bill, no GST, **no tender rows**: day close reads the ledger.
- A Sundry Debtors ledger is refused on either side.

### 4.4 Journal with two parties

DR Ravi Traders 10,000 / CR Sri Krishna Traders 10,000: a balance transferred
between two customers.

- `avh_party_id` is NULL, and the screen shows "— several —".
- Each party leg carries its own bill-wise allocation (`abj_party_id` is the
  truth).
- If a party ledger is bill-by-bill, any unallocated remainder raises a
  JOURNAL bill.

### 4.5 Cancel

`pua00019` cancelled on 20-09-2026 writes a reversal in the **`Rev`** series (e.g. `rev00045`, narration "Reversal of pua00019"):

- under type `Rev`, dated the original's date, 15-09-2026 (C2);
- every leg mirrored;
- the raised bill closed;
- allocations reversed by counter-rows (never deleted);
- the GST document CANCELLED;
- a TDS reversal row.

The next Purchase voucher is still `pua00020`. The number series never loses a
number.

---

## 5. DDL: `39_voucher_register.sql`

> **The runnable script is `share/39_voucher_register.sql`** (dry-run clean on
> `erp_dry` 2026-09-25, twice; its group-name guard proven by a negative
> test). **Where this section and the script differ, the script wins.** Three
> deliberate differences:
> - `Rev` is **not inserted**: it exists (Prathap's `20260925150000`); the
>   script only gives it its rule columns, and stops if it is missing.
> - The raise-bill check allows a bill type on OPTIONAL (Journal = JOURNAL).
> - The RCM roles use the live CHECK values (`alr_group 'PURCHASE'`, type
>   `TAX`, a duty, nature `Liabilities`).

### 5.1 The party column allows NULL (decision A)

```sql
ALTER TABLE accounts.acc_voucher_header ALTER COLUMN avh_party_id DROP NOT NULL;
```

- `ix_avh_party` stays; a NULL party simply is not indexed.
- In `ux_avh_doc_refno (company, party, type, year, doc_refno)`, NULLs are
  distinct, so a no-party voucher's supplier reference is not
  uniqueness-checked. That is correct: a Contra has no supplier.
- **Check before running:** grep the service for `avhPartyId` reads that assume
  non-null. Receipt, cheques and sales always set it; nothing reads it as
  required.
- The party of a multi-party Journal lives on each party leg and in
  `abj_party_id` / `abl_party_id` (S13).

### 5.2 Rule columns on the voucher type

```sql
ALTER TABLE accounts.acc_voucher_types
  ADD COLUMN vchr_party_mode     varchar(4)  NOT NULL DEFAULT 'NONE',  -- NONE | ONE | MANY
  ADD COLUMN vchr_party_side     varchar(3)  NOT NULL DEFAULT 'ANY',   -- DR | CR | ANY
  ADD COLUMN vchr_billwise_mode  varchar(8)  NOT NULL DEFAULT 'OFF',   -- OFF | DEMAND | RAISE | OPTIONAL
  ADD COLUMN vchr_raise_bill_type varchar(20),                         -- abl_bill_type a RAISE writes
  ADD COLUMN vchr_dr_groups      uuid[]      NOT NULL DEFAULT '{}',    -- empty = any; a CHILD of a listed group counts
  ADD COLUMN vchr_cr_groups      uuid[]      NOT NULL DEFAULT '{}',
  ADD COLUMN vchr_gst_register   varchar(10),                          -- NULL | GSTR1 | GSTR1_9B | GSTR2 | GSTR2_NOTE
  ADD COLUMN vchr_gst_side       varchar(6),                           -- INPUT | OUTPUT: which tax roles the band generates
  ADD COLUMN vchr_tds_mode       varchar(8)  NOT NULL DEFAULT 'OFF',   -- OFF | DEDUCT (from the party's section)
  ADD COLUMN vchr_menu_id        integer,                              -- decision E
  ADD COLUMN vchr_in_register    boolean     NOT NULL DEFAULT false;   -- offered on the register's band

ALTER TABLE accounts.acc_voucher_types
  ADD CONSTRAINT ck_vchr_party_mode    CHECK (vchr_party_mode IN ('NONE','ONE','MANY')),
  ADD CONSTRAINT ck_vchr_party_side    CHECK (vchr_party_side IN ('DR','CR','ANY')),
  ADD CONSTRAINT ck_vchr_billwise_mode CHECK (vchr_billwise_mode IN ('OFF','DEMAND','RAISE','OPTIONAL')),
  ADD CONSTRAINT ck_vchr_raise_type    CHECK ((vchr_billwise_mode = 'RAISE') = (vchr_raise_bill_type IS NOT NULL)),
  ADD CONSTRAINT ck_vchr_gst_register  CHECK (vchr_gst_register IS NULL
        OR vchr_gst_register IN ('GSTR1','GSTR1_9B','GSTR2','GSTR2_NOTE')),
  ADD CONSTRAINT ck_vchr_gst_side      CHECK ((vchr_gst_register IS NULL) = (vchr_gst_side IS NULL)
        AND (vchr_gst_side IS NULL OR vchr_gst_side IN ('INPUT','OUTPUT'))),
  ADD CONSTRAINT ck_vchr_tds_mode      CHECK (vchr_tds_mode IN ('OFF','DEDUCT')),
  ADD CONSTRAINT ck_vchr_party_billwise CHECK (vchr_party_mode <> 'NONE' OR vchr_billwise_mode = 'OFF'),
  ADD CONSTRAINT fk_vchr_menu FOREIGN KEY (vchr_menu_id) REFERENCES fixed.menu_master(menu_id);
```

- **Group ids, not names**: names are editable. They are all global today
  (`acc_group_company_id` NULL on every row), so one seed serves every company.
- **"A child counts"**: a ledger is legal on a side when its group **or any
  ancestor** (`acc_group_parent_id`, walked in the service with a recursive
  CTE, cached per request) is in the list. "Indirect Expenses" then admits a
  sub-group such as "Office Expenses" and every ledger under it, without
  listing each leaf.

### 5.3 Menus: one per type, plus the register (decisions D + E)

| Type | Menu | Action |
|---|---|---|
| Debit Note | 101 | exists: set `menu_visiblity = true` |
| Credit Note | 102 | exists: visible |
| Journal | 103 | exists: visible |
| Contra | 104 | exists: visible |
| Purchase (acct) | **163** "GST Expenses" | **rename to "Purchase (Accounting)"**, visible, and add the verbs `POST, CANCEL` |
| Sales (acct) | new | insert under parent 5 |
| Receipt (voucher) | new | insert under parent 5 ("Receipt Voucher", distinct from menu 99 "Receipt") |
| Payment (voucher) | new | insert under parent 5 ("Payment Voucher") |
| **Voucher Register** | new | insert under parent 5. It grants nothing itself (§7.1) |
| Reversal (`Rev`) | none | never offered on the band. It inherits the ORIGINAL type's menu for rights |

- Every register menu gets `menu_verbs = '{VIEW,CREATE,EDIT,DELETE,PRINT,EXPORT,POST,CANCEL}'`,
  like 101. There is no AMEND verb: a posted voucher is corrected by cancel
  and re-enter (§8.3).
- New `menu_id`s come from `fixed.menu_master_menu_id_seq`. The type rows
  point at them by a lookup inside the same migration, never by a literal id.
- **Grant nothing in the migration.** Rights are granted per user through the
  normal screen. Remember that `user_menus` save is a **full replace**: a
  partial list revokes a user's other menus.

### 5.4 Seed the types

| code | name | nature | party | side | bill-wise | raise type | DR groups | CR groups | GST | TDS | menu |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `Jrl` | Journal | JOURNAL | MANY | ANY | OPTIONAL | — | any | any | — | OFF | 103 |
| `Con` | Contra | CONTRA | NONE | ANY | OFF | — | Cash-in-Hand, Bank Accounts, Bank OD A/c | same | — | OFF | 104 |
| `DrN` | Debit Note | DEBIT_NOTE | ONE | DR | RAISE | `JOURNAL` | any | Direct Incomes, Indirect Incomes, Sales Accounts, Duties & Taxes | GSTR1_9B / OUTPUT | OFF | 101 |
| `CrN` | Credit Note | CREDIT_NOTE | ONE | CR | RAISE | `JOURNAL` | Direct / Indirect Incomes, Sales Accounts, Direct / Indirect Expenses, Duties & Taxes | any | GSTR1_9B / OUTPUT | OFF | 102 |
| `PurA` | Purchase (Accounting) | PURCHASE | ONE | CR | RAISE | `PURCHASE` | Direct Expenses, Indirect Expenses, Purchase Accounts, Fixed Assets, Duties & Taxes | Sundry Creditors, Duties & Taxes | GSTR2 / INPUT | DEDUCT | 163 |
| `SalA` | Sales (Accounting) | SALES | ONE | DR | RAISE | `SALES` | Sundry Debtors | Direct Incomes, Indirect Incomes, Sales Accounts, Duties & Taxes | GSTR1 / OUTPUT | OFF | new |
| `RcpV` | Receipt Voucher | RECEIPT | ONE | CR | DEMAND | — | Cash-in-Hand, Bank Accounts | any | — | OFF | new |
| `PmtV` | Payment Voucher | PAYMENT | ONE | DR | DEMAND | — | any | Cash-in-Hand, Bank Accounts | — | DEDUCT | new |
| `Rev` | Reversal (**exists**: Prathap's 20260925150000) | JOURNAL | MANY | ANY | OFF | — | — | — | — | OFF | **NULL** |

Every row has:

- `vchr_category 'ACCOUNTING'`, `vchr_affects_accounts true`,
  `vchr_affects_inventory false`;
- `vchr_numbering_mode 'AUTO'`, `vchr_reset_freq 'YEARLY'`, `vchr_no_width 5`
  (matching `bil`, `rct`, `tnc`);
- a lower-case prefix: `jrl, con, drn, crn, pua, saa, rcv, pmv, rev`;
- `vchr_in_register = true`, except on `Rev`.

`vchr_is_cash_voucher` / `_is_bank_voucher` stay false. The day book
classifies by the legs' ledgers, not the type.

- **The group names above are the live ones** (checked 2026-09-25): there is **no** "Expenses" or "Income" group on the box. It is Tally's set: Direct / Indirect Expenses, Direct / Indirect Incomes, Purchase Accounts, Sales Accounts, Fixed Assets, Duties & Taxes, Sundry Debtors / Creditors, Cash-in-Hand, Bank Accounts, Bank OD A/c. The PNG's rule strip says "Expenses" as shorthand for the first two.
- **Group names in the table resolve to ids at migration time** (a
  `SELECT … WHERE acc_group_name IN (…)`). **Verify each name exists on the box
  before running.** A missing name must fail the migration, not seed an empty
  array that means "any".
- **No `acc_voucher_seq` rows are seeded.** `allocateVoucherNumber()` creates
  the row on first use, from the type's format.
- `vchr_type_id` comes from the sequence. The service resolves types by
  **`vchr_type_code`**, never by a literal id.
- The DrN/CrN "GST or not" fork (S21) is **per voucher**: a note with no GST
  lines writes no `gdr_` row even though the type allows one (§7.3 step 9).

### 5.5 Roles, and one GST column

```sql
INSERT INTO accounts.acc_ledger_role (alr_role, alr_label, alr_group, alr_want_type, alr_want_nature, alr_sort_order)
VALUES ('INTEREST_PAID',    'Interest paid',              'SHARED','EXPENSE','Expenses',   211),
       ('RATE_DIFFERENCE',  'Rate difference',            'SHARED','EXPENSE','Expenses',   212),
       ('RCM_CGST_PAYABLE', 'RCM CGST payable',           'SHARED','DUTY',   'Liabilities',230),
       ('RCM_SGST_PAYABLE', 'RCM SGST payable',           'SHARED','DUTY',   'Liabilities',231),
       ('RCM_IGST_PAYABLE', 'RCM IGST payable',           'SHARED','DUTY',   'Liabilities',232)
ON CONFLICT DO NOTHING;

ALTER TABLE accounts.acc_voucher_doc_detail
  ADD COLUMN vtx_itc_eligibility varchar(16),                 -- INPUTS | INPUT_SERVICES | CAPITAL_GOODS | INELIGIBLE
  ADD CONSTRAINT ck_vtx_itc CHECK (vtx_itc_eligibility IS NULL
        OR vtx_itc_eligibility IN ('INPUTS','INPUT_SERVICES','CAPITAL_GOODS','INELIGIBLE'));
```

- Check `alr_group` / `alr_want_type` values against the live CHECKs, and the
  role column widths, before running. The shape above copies the existing
  rows.
- `INTEREST_INCOME`, `BANK_CHARGES`, `DISCOUNT_ALLOWED`, `WRITE_OFF`,
  `ROUND_OFF`, `TDS_PAYABLE`, and `INPUT_*` / `OUTPUT_*` already exist.
- The three RCM roles need `acc_ledger_map` rows **per company**. They appear
  in the ledger-map screen (menu 250) automatically, because it lists roles
  from the table.
- ITC eligibility defaults from the expense ledger's **`led_itc_eligibility`**
  (live column) and can be changed per line. It feeds GSTR-3B table 4. There
  is no other home for it: `vtx_taxability` is taxable / exempt / nil.

### 5.6 TDS rates: shared with Payment

TDS rates come from **`accounts.tds_rates`** (section × deductee type × date →
rate, thresholds, 206AA no-PAN 20%). That table is specified in
`../payment/plan-backend-payment.md` §2.9 and is **not on the box yet**. Ship
it in whichever migration runs first (Payment's or this one) and do not copy
the DDL here. The filing row goes to the **live** `accounts.acc_tds_register`
(26Q projection).

---

## 6. Routes: `/api/v1/vouchers/*`

All routes are bearer-authenticated, use URI versioning, and have **no path
params**. Every route that acts on an existing voucher takes the house key:
**`companyId · branchId · accYear · voucherId`**. There is **no `/list`**: the
F8 list and the exceptions report are registered grids (§10). **No route
reuses another module's URL.** The picker, the balance and the party facts are
this module's own, so a change elsewhere cannot move them.

| # | Method + path | In | Out |
|---|---|---|---|
| 6.1 | GET `/vouchers/types` | `companyId`, `menuId?` | the register types **the caller may view**, each with its rule columns and **the caller's rights on its menu** `{view, create, edit, delete, post, cancel, print}`. With `menuId` (opened from 101/102/…), only that menu's type |
| 6.2 | GET `/vouchers/ledger-pick` | `companyId, branchId, typeCode, side (DR\|CR), q?, limit?` | ledgers legal on that side for that type (group or ancestor in the list), **minus instrument-controlled ledgers**: `{ledId, name, groupName, isParty, isBillByBill, gstApplicable, itcEligibility (led_itc_eligibility), defaultTaxId (led_tax_id), isTdsApplicable (led_is_tds_applicable — the TDS tick's default, §9.8), tdsSection?}` |
| 6.3 | GET `/vouchers/ledger-balance` | `companyId, branchId?, accYear, ledgerId, asOn` | `{amount, side}`: opening (`acc_opening_balance`, `'D'`→+) + Σ `av_signed_amount` of POSTED **and CANCELLED** headers ≤ asOn. **Never `led_total_*`**. The same figure `fn_ledger_book_balance` gives (its Dr-opening sign was fixed 2026-09-25, migration 20260925130000) |
| 6.4 | GET `/vouchers/party-facts` | `companyId, partyId, asOn` | `{gstin, gstType, stateCode, stateName, creditDays, isBillByBill, tds: {applicable, section, deducteeType, rate, rateSource} \| null, outstanding: {amount, side}}` |
| 6.5 | GET `/vouchers/open-bills` | `companyId, partyId, side (DR\|CR)` | `abl_` rows with pending > 0, oldest first: `{ablId, ablAccYear, refno, docRefno, date, dueDate, billType, billAmount, pending}` |
| 6.6 | GET `/vouchers/tax-rates` | `companyId` | active `tax_rate_master` rows `{taxId, name, ratePerc, cgst, sgst, igst, cess, isReverseCharge, taxability}`. The client needs them to preview the generated legs; the server recomputes regardless |
| 6.7 | POST `/vouchers/create` | payload (§6.13) | saves or updates a **DRAFT**: the header row with no number, and the payload in **`avh_draft_lines` jsonb** (live column). **No legs, no allocations, no bills** until post |
| 6.8 | POST `/vouchers/validate` | payload + `overrides[]?` | **writes nothing.** Returns the **derived voucher** (every leg, generated legs flagged, the GST summary, TDS, the party amount, the bill to be raised) + `refusals[]` + `warnings[]`. The client paints the grey rows from it |
| 6.9 | POST `/vouchers/post` | keys + payload + `overrides[]?` | runs the routine (§7). A new voucher may post without a prior create (it is created and posted in one transaction). Returns `/get` |
| 6.10 | POST `/vouchers/cancel` | keys + `reason` (required) | the reversal (§8.3). Returns `{voucherRefno, reversalRefno, cancelledOn}` |
| 6.11 | POST `/vouchers/delete` | keys | DRAFT only (hard or soft per house rule). POSTED → 409 "cancel it" |
| 6.12 | GET `/vouchers/get` | keys | header + legs (with `generated`, `role`) + allocations + bills raised + GST doc + TDS row + `rights` + `locks {editable, dayClosed, periodLocked, allocatedElsewhere}` |

**Envelope and codes**, the same shape as the bill module:
`{success, message, data}`, or `{success:false, message, errors:[{field, message,
code, line?}]}`. The status map is 400 malformed, 403 right missing, 404,
409 state, 422 rule refusals (all of them in one body). Codes are prefixed
`VCH_`: `VCH_RIGHT_CREATE|POST|CANCEL|VIEW`, `VCH_UNBALANCED`,
`VCH_LEDGER_SIDE`, `VCH_INSTRUMENT_LEDGER`, `VCH_PARTY_MODE`,
`VCH_BILLWISE_SHORT`, `VCH_BILL_OVERSPENT`, `VCH_PERIOD_LOCKED`,
`VCH_YEAR_CLOSED`, `VCH_TYPE_NOT_REGISTER`, `VCH_GST_RATE_MISSING`,
`VCH_GST_LEDGER_UNMAPPED`, `VCH_TDS_UNMAPPED`, `VCH_ALLOCATED_ELSEWHERE`
(cancel), `VCH_NOT_DRAFT`, `VCH_POSTED`.

### 6.13 The payload: only what the operator typed

```jsonc
{
  "header": {
    "voucherId": "…",              // absent on a new voucher (the server mints uuidv7)
    "companyId": "…", "branchId": "…", "accYear": "2026-2027",
    "typeCode": "PurA",
    "date": "2026-09-15",
    "partyId": "…",                // ONE-mode types; omitted for NONE; omitted for MANY (it lives on the lines)
    "docRefno": "SFS/2026/451", "docDate": "2026-09-12",   // the supplier's / customer's document
    "usrRefno": null,
    "posStcd": "33",               // place of supply; default = the party's state (SalA) / the company's (PurA, RCM)
    "reverseCharge": false,
    "remarks": "Housekeeping Sept + office stationery"
  },
  "lines": [                       // TYPED lines only — never a generated leg
    { "rowNo": 1, "drCr": "DR", "ledgerId": "…", "amount": 25000.00, "remarks": "Sept housekeeping",
      "gst": { "taxId": "…", "hsn": "998533", "itcEligibility": "INPUT_SERVICES" },
      "tdsBase": true },           // this line's amount counts toward the TDS base
    { "rowNo": 2, "drCr": "DR", "ledgerId": "…", "amount": 4000.00, "remarks": "A4 reams",
      "gst": { "taxId": "…", "hsn": "4820", "itcEligibility": "INPUTS" }, "tdsBase": false }
  ],
  "allocations": [                 // against the PARTY leg (ONE) or a typed party line (MANY)
    { "lineRowNo": 0,              // 0 = the generated party leg; n = typed line n
      "billId": "…", "billAccYear": "2026-2027", "amount": 5000.00 }
  ],
  "newBill": { "dueDays": 30 }     // RAISE types: the remainder raises a bill; dueDays defaults to the party's credit days
}
```

- **Generated legs are never accepted from the client.** A line on a tax
  ledger that the band would generate (a ledger mapped to an `INPUT_*` /
  `OUTPUT_*` / `RCM_*` / `TDS_PAYABLE` role) on a type with
  `vchr_gst_register` set is refused with `VCH_GST_LEDGER_TYPED`. A Journal
  (no GST register) **may** carry tax ledgers as ordinary typed lines; that is
  how a GST set-off or reclassification journal is keyed.
- Amounts arrive as numbers or numeric strings. The server rounds each **once**
  at 2 dp HALF_UP (Decimal end to end, no float).
- **Absent ≠ empty** holds for `allocations`, as it does on the bill.

---

## 7. The posting routine

### 7.0 Where it lives: lift, don't fork

The pieces exist today inside `modules/sales/posting` and
`modules/sales/bill`. Move them to **`src/common/posting/`** (or an
`accounts-posting` module) and keep the sales imports working through
re-exports:

| Piece | Today | Used by the routine for |
|---|---|---|
| `allocateVoucherNumber`, `allocateVoucherSlno`, `buildRefno` | `common/Sequence/voucher-sequence.helper.ts` (already common) | numbering (§7.3 step 6) |
| `deriveVoucherTotals` | `accountsModule/accountVoucherHeader/voucher-totals.helper.ts` | a readable Dr≠Cr failure before POSTED |
| `loadRights` / `assertRight` | `sales/posting/sales.guards.ts` | rights. **Extend** to read `um_can_view, um_can_create, um_can_edit, um_can_delete, um_can_print` too; today it reads the five posting flags only |
| `postLegs`, `reverseLegs` | `SalesPostingService` | writing legs; the reversal (as-is: it already uses the `Rev` type) |
| `syncBillAdjustments`, `loadSetOffCredits` | `sales/bill/bill-adjustment.helper.ts` | allocations + `abl_` alloc / pending / status by hand (there is no `fn_abl_recompute`, and installing one would double count) |
| GST register writer | sale bill `postCore` step 4 | the `gdr_` + `vtx_` rows, generalised to `gdr_source_module = 'ACCOUNTS'` |
| `assertAccYearWritable`, `assertVoucherPartitionExists` | `accountsModule/receipt/receipt.guards.ts` | year and partition |
| `assertBooksReconcile` | `accountsModule/reconcile/books-reconcile.guard.ts` | trial-mode check at the end (the setting `accounts.reconcile_on_post`) |

### 7.1 Rights (decision E)

- The rights menu is **the voucher type's `vchr_menu_id`**, whichever menu the
  screen was opened from. The Voucher Register menu itself grants nothing.
  Opening it needs only its `um_can_view`; every action is judged on the
  type's menu.
- Per route: `/get`, `/validate` and the pickers need **view**; `/create` needs
  **create** (new) or **edit** (existing draft); `/post` needs **post** (plus
  create on a new voucher); `/cancel` needs **cancel**; `/delete` needs
  **delete**. `Rev` never has its own rights: cancel is judged on the original
  type's menu.
- **No row means no right.** That is `loadRights`' existing rule: "defaulting
  a missing permission to TRUE is how a permission system stops being one".
- `/types` returns only types with view, so a type the user lacks never
  reaches the band.

### 7.2 The canonical input

`{ header, legs[], allocations[], gst?, tds?, bill? }` is the routine's
**internal** input, after derivation. Register, Receipt and Payment each map
their own DTO into it. The register derives it from §6.13. The receipt maps its
live `/receipts/*` DTO without changing its routes.

### 7.3 Post, in order, in ONE transaction

1. **Load and lock the type.** It must have `vchr_in_register` (or be called by
   Receipt/Payment with their own type). `vchr_affects_inventory` → refuse.
   Rights → 403.
2. **Calendar.** `assertAccYearWritable` + `assertVoucherPartitionExists`. The
   date must be inside the year. `fiscal_years.fy_lock_date` ≥ date →
   `VCH_PERIOD_LOCKED`. (A day-close lock joins here when that table exists;
   today it does not.) The check is at POST, not at save: a draft typed before
   a lock can still post into an open day.
3. **Typed lines.** ≥ 1 typed line, and every amount > 0 (`ck_av_amount`).
   Every ledger is legal on its side (group or ancestor, §5.2) and active.
   **No instrument-controlled ledger**: `accounts.fn_is_cheques_in_hand_ledger()`
   plus any ledger that is a tender master's `tnd_ledger_id` or
   `tnd_settlement_ledger_id` for a non-cash tender. Message: "use Received /
   Issued Cheques (menu 51 / 52)".
4. **Generate the GST legs** (when `vchr_gst_register` is set and a line
   carries `gst`):
   - rates from `inventory.tax_rate_master` by `taxId`;
   - **intra vs inter** = `posStcd` vs the company state;
   - per line: `taxable = amount`, then CGST/SGST **split from one rounded
     figure** (the sale bill's paise rule), or IGST, plus cess;
   - **one leg per tax ledger** (summed across lines), with the ledger from
     **`inventory.tax_rate_ledger`** (`trl_tax_id`, `trl_role` =
     `vchr_gst_side`'s `INPUT_*` / `OUTPUT_*`, `trl_supply_nature`), falling
     back to `acc_ledger_map` by role. An unmapped ledger →
     `VCH_GST_LEDGER_UNMAPPED` naming the tax and role;
   - **side**: PurA and DrN put input/output tax on the DR/CR side that
     increases the party amount. A **CrN** puts **output tax on DR** (it
     reduces the tax the customer was charged). The side comes from the
     type's party side, not a switch;
   - **reverse charge** (`reverseCharge`, PurA only): DR `INPUT_*` and CR the
     matching `RCM_*_PAYABLE`. The party is owed the taxable only.
     `gdr_is_reverse_charge = true`.
5. **Generate the TDS leg** (`vchr_tds_mode = 'DEDUCT'`, and the party is TDS
   applicable):
   - `base = Σ lines with tdsBase` (the taxable, never incl. GST). A line that
     omits `tdsBase` defaults to its ledger's `led_is_tds_applicable`;
   - rate from `tds_rates` by section × deductee type × date, and 206AA (no
     PAN) → 20%;
   - threshold (single / annual) → below it, no deduction, and a WARN that
     says so;
   - leg: CR TDS Payable (role `TDS_PAYABLE`), amount = round(base × rate).
   - `VCH_TDS_UNMAPPED` when the role has no ledger.
6. **Generate the party leg** (party mode ONE): side = `vchr_party_side`,
   amount = the balancing figure (Σ the other side − Σ this side). It must be
   > 0, else `VCH_UNBALANCED` "the lines leave nothing for the party". For
   party mode MANY the party lines are typed, and the voucher must balance as
   typed. For NONE there is no party leg. `avh_party_id` = the ONE party, else
   NULL.
7. **Balance.** Σ DR = Σ CR exactly (Decimal), using `deriveVoucherTotals()` for
   the readable failure. The trigger fills the header totals, and
   `ck_avh_balanced` is the final guard.
8. **Number.** `allocateVoucherNumber(tx, {vchrTypeId, companyId, branchId,
   accYear, documentDate})` and `allocateVoucherSlno`. Write `avh_voucher_no /
   _slno / _refno`.
9. **Write** the header (clear `avh_draft_lines`), then the legs through the
   lifted `postLegs` (`av_row_no`, `av_role`, `av_opp_ledger_id` = the party
   on non-party legs when there is exactly one party, else NULL). Generated
   legs are written like any other; `/get` marks them from their role +
   type.
10. **Bills and allocations** (per party leg):
    - `DEMAND` (RcpV / PmtV): allocations must sum **exactly** to the leg,
      else `VCH_BILLWISE_SHORT`;
    - `RAISE`: allocations first (against open bills, FOR UPDATE; an overspend
      → 409 `VCH_BILL_OVERSPENT`), then **the remainder raises one `abl_`
      bill**: `abl_bill_type = vchr_raise_bill_type`, `abl_dr_cr` = the leg's
      side, `abl_src_module 'ACCOUNTS'`, `abl_src_doc_type 'VOUCHER'`,
      `abl_voucher_id / _line_id / _type_id / _no / _date / _refno`,
      `abl_doc_refno / _doc_date` = the party's document (else the voucher's
      own refno/date; both are NOT NULL), `abl_due_date` = date + dueDays,
      `abl_credit_days`, `abl_branch_id` = the voucher's branch (NOT NULL);
    - `OPTIONAL` (Journal): allocate what is sent; the remainder raises a
      `JOURNAL` bill **only if the party ledger is bill-by-bill**;
    - `OFF`: nothing;
    - every allocation writes the pair through `syncBillAdjustments` with
      `abj_voucher_line_id` = the party leg, and `abj_party_id`.
11. **GST document** (only if at least one line carried `gst`):
    - one `acc_voucher_doc_register` row: `gdr_source_module 'ACCOUNTS'`,
      `gdr_doc_type` per type (PurA `PURCHASE`, SalA `SALES`, CrN
      `CREDIT_NOTE`, DrN `DEBIT_NOTE`), party snapshot (name, GSTIN, state),
      POS, `gdr_is_reverse_charge`, `gdr_is_einvoice_applicable` (SalA / DrN /
      CrN to a registered party, per the company's e-invoice applicability),
      values;
    - one `acc_voucher_doc_detail` row per typed GST line: `vtx_row_no` = **the
      line's `av_row_no`**, `vtx_tax_id`, rates and amounts, `vtx_hsn_code`,
      `vtx_is_service` (from the HSN/SAC's leading 99), `vtx_itc_eligibility`,
      the taxable and tax ledger ids, `vtx_item_*` NULL.
12. **TDS register**: one `acc_tds_register` row (`atd_direction` = deducted by
    us, quarter, party, PAN, deductee type, section, rate, `atd_rate_source`,
    base, tax, the voucher, and the bill raised).
13. **Status** POSTED, `avh_posted_on`, `_status_on`, `_status_by`. Then
    `assertBooksReconcile` (trial mode). Any failure rolls back everything,
    **the number included**.

### 7.4 Validate = the same code, dry

`/vouchers/validate` runs steps 1–7 and 10–12 in a **read-only** pass (no
number, no writes). It returns the derived voucher and every refusal and
warning together, on the same model as `/bills/validate`: an overridable WARN
stays a WARN on validate and becomes a refusal on post unless it is in
`overrides[]` and the user holds override. The v1 WARNs:

- `VCH_TDS_BELOW_THRESHOLD`;
- `VCH_BACKDATED` (per an `accounts.backdate_mode` setting, if the company wants one);
- `VCH_DUP_DOC_REFNO` (the same party document number on another live
  voucher, beyond what `ux_avh_doc_refno` enforces).

### 7.5 Draft

`/create` stores the header plus `avh_draft_lines` (the §6.13 payload,
verbatim). **No legs, allocations, bills, GST or TDS rows exist for a DRAFT**
(receipt rule R10). This keeps a draft from touching any balance, report or
reconcile. Reopening a draft returns the payload from `avh_draft_lines`.

### 7.6 Receipt, Payment and the goods documents

- **Receipt (live):** move its post onto the routine **without changing
  `/receipts/*`**. Its DTO maps to `{header, legs, allocations}`, and its
  tender rows and cheque register stay its own extras, written in the same
  transaction. **The acceptance test** (§11 #20) posts the same receipt through
  the old and the new path on `erp_dry` and compares the `av_` / `abj_` /
  `abl_` rows.
- **Payment (not built):** build it on the routine from day one; its
  `tds_rates` is shared.
- **Sale bill / return:** not in this plan. Their legs already go through
  `postLegs`, and moving them fully is a later refactor.

---

## 8. Lifecycle

### 8.1 States

`DRAFT → POSTED → CANCELLED`. There is no approve step (`APPROVED` stays legal
in `ck_avh_status`, and nothing writes it).

### 8.2 Editing a posted voucher

It cannot be edited. The correction is cancel + re-enter, or a Journal. **No
amend verb in v1**, and no `AMEND` in the menus' verbs. If the user later
wants Receipt's R20-style amend for the register, it is an additive change.

### 8.3 Cancel = a reversal under the `Rev` type (decision C)

**Already solved on the server.** `SalesPostingService.reverseLegs()` used to
draw the reversal's number from the **original's own type**. A cancelled sale
bill burned the next bill number and left a gap in the GST series (D3). Since
Prathap's migration `20260925150000` it numbers every reversal in the
**`Rev`** type instead (`rev00001…`), and the old mirrors were moved into that
series. The register reuses it unchanged.

**The register's cancel:**

1. Rights: cancel on the type's menu. The voucher is POSTED (a DRAFT → 409
   "delete it"). The year is writable and the period is not locked **as of
   the cancel date**.
2. **Refuse if another voucher has allocated against a bill this voucher
   raised** (`VCH_ALLOCATED_ELSEWHERE`, naming the receipt/payment). Release it
   there first, the same rule as the sale bill's `SALES_ALLOCATION_LOCKS_BILL`.
3. Reverse this voucher's own allocations (negative counter-rows,
   `abj_reversal_of_id`, `abj_reversal_reason`; **never delete**). Recompute
   every touched `abl_`. Soft-delete / close the bill it raised.
4. **The reversal voucher:** the lifted `reverseLegs()`, **as it is today**:
   - type `Rev`, with its own series, so the original's series never loses a
     number;
   - legs mirrored;
   - narration "Reversal of <original refno>";
   - linked both ways: `avh_reversal_voucher_id / _acc_year` on the original,
     `avh_against_voucher_id / _acc_year` on the reversal;
   - dated **the original's date** (C2, decided: the live rule);
   - the original is marked CANCELLED **before** the mirror is written, and the
     mirror carries no `avh_src_*` / `avh_doc_refno`. Both are the live
     ordering rules (`ux_avh_src`, `ux_avh_doc_refno`).
5. The GST document: `gdr_doc_status 'CANCELLED'`, `gdr_doc_cancel_reason`,
   `gdr_doc_canceled_on`. If an IRN exists, the cancel is refused with the
   e-invoice screen's message (the register never calls the portal).
6. The TDS register row gets a reversal row (`atd_reversal_of_id`), never a
   delete. **A deduction already paid by challan** (`atd_challan_no` set) →
   refuse (`VCH_TDS_DEPOSITED`); the correction is a 26Q revision, not a cancel.
7. The original → CANCELLED with `avh_cancel_reason` (`ck_avh_cancel`). **Both
   stay on every list and in every balance.** They net to zero, and the ledger
   statement counts CANCELLED headers.

### 8.4 Delete

A DRAFT only. It has no number, legs or bills, so nothing to unwind.


---

## 9. The Qt client

**For Viji. Written 2026-09-25, and coded once §6's routes answer on the box.**
Migration 39 is live (Prathap's `20260925160000`); the `/vouchers/*` module
is not built yet. It follows the Receipt screen (menu 99) wherever that screen
has already answered a question, because the Receipt is the one accounts
document screen that is live and tested.

### 9.1 House rules

These are copied from the live accounts screens, not invented here:

- Its own `.ui` on **`NexDialog`**, never `NexCrud*`.
- `NexTable` + a delegate's **`cellEdited`**, never `itemChanged`.
- `loadLayout(<ui_table id>)`, with a local `seedLayout()` fallback, the
  Receipt's pattern, so the grid works even when the layout call fails.
- `NexDateEdit` → listen on **`dateEdited`** (user intent), never
  `dateChanged`.
- `NexDoubleEdit`, `AppMessage` (and `AppMessage::askText` for every prompt),
  `ApiClient`.
- **Scope captured once**: company / branch / year from `AppSession` on New,
  and from the loaded voucher afterwards. Never re-read mid-life.
- **Ctrl+Enter is the primary verb.** No button is a default button.
- Every async callback is guarded against a closed screen **and** against a
  stale answer (a generation counter, §9.6).
- **The list is `TxnMainView`**, exactly as the Receipt does it. The menu opens
  `TxnMainView::openForMenu(<descriptor>)`; `AppSession::txnEntryFirst()`
  decides whether the menu lands on the register or the voucher. F8 inside
  the voucher is the **picker** (`TxnMainView::openPicker`). (This supersedes
  the older "F8 is a page in the same .ui" note; the Receipt moved to
  TxnMainView and this screen follows it.)

**The one rule specific to this screen: the client holds no rules of its
own.** Which side the party sits on, which ledgers a side accepts, whether
bill-wise is demanded, whether the GST band shows, whether TDS applies, and
what the user may do all come from `GET /vouchers/types`, and the derived legs
come from `POST /vouchers/validate`. **If a rule ends up in a `switch` on the
type code, or a tax figure is multiplied in C++, the design has failed.**

### 9.2 Files

```
src/modules/accounts/register/
  voucher_register_entry.h / .cpp / .ui   // NexDialog — the voucher (entry) screen
  voucher_line_delegate.h / .cpp          // legs grid editors: Dr/Cr combo, Ledger (opens the picker),
                                          //   GST rate combo, HSN/SAC, Debit / Credit (NexDoubleEdit), narration
  voucher_alloc_delegate.h / .cpp         // bill-wise card: "This voucher" amount per open bill
  voucher_ledger_picker.h / .cpp          // popup over GET /vouchers/ledger-pick (a ROUTE, not a grid — §6)
  voucher_register_entity.h               // VoucherTypeRules, VoucherLine, VoucherAlloc, DerivedLeg,
                                          //   GstSummaryRow, PartyFacts; toJson / fromJson; column enums
  voucher_derive_client.h / .cpp          // the /validate round trip: debounce + generation counter (§9.6)
src/modules/accounts/common/txn_doc_types_accounts.{h,cpp}
                                          // + TxnDocTypes::voucher(typeCode) and voucherRegister()
src/core/api/endpoints/accounts_end_points.h
                                          // + namespace Vouchers (§9.3)
src/core/navigation/form_accounts_menu.h  // + nine registrations (§9.10)
tests/register/                           // run.sh + test_register.cpp (§11.2)
```

`CMakeLists.txt` is edited by hand, as for every module.

### 9.3 `ApiEndpoints::Vouchers`

```cpp
namespace Vouchers {
inline constexpr char Types[]         = "/vouchers/types";          // GET  companyId, menuId?
inline constexpr char LedgerPick[]    = "/vouchers/ledger-pick";    // GET  companyId, branchId, typeCode, side, q, limit
inline constexpr char LedgerBalance[] = "/vouchers/ledger-balance"; // GET  (not on the rev 2 screen; kept for the ledger tooltip)
inline constexpr char PartyFacts[]    = "/vouchers/party-facts";    // GET  companyId, partyId, asOn
inline constexpr char OpenBills[]     = "/vouchers/open-bills";     // GET  companyId, partyId, side
inline constexpr char TaxRates[]      = "/vouchers/tax-rates";      // GET  companyId
inline constexpr char Create[]        = "/vouchers/create";         // POST payload (§6.13)
inline constexpr char Validate[]      = "/vouchers/validate";       // POST payload + overrides[]
inline constexpr char Post[]          = "/vouchers/post";           // POST key + payload + overrides[]
inline constexpr char Cancel[]        = "/vouchers/cancel";         // POST key + reason
inline constexpr char Delete[]        = "/vouchers/delete";         // POST key (DRAFT only)
inline constexpr char Get[]           = "/vouchers/get";            // GET  key
struct VoucherKey { QString companyId, branchId, accYear, voucherId; };   // the house key, always all four
QString get(const VoucherKey &k);   // builds the query; one place spells the four names
}
```

Its own namespace in `accounts_end_points.h`, beside `Receipts` and `Cheques`.
No reuse of `/receipts/*` or `/account-ledger-masters/*` (the own-URLs rule).
**Verify every key name against the live Swagger and one real response
before coding a consumer**: the module is written from §6, not from a running
route.

### 9.4 Opening the screen: which types, which rights

| Opened from | Band shows | Type fixed? |
|---|---|---|
| 103 Journal / 104 Contra / 101 Debit Note / 102 Credit Note / 163 Purchase (Accounting) / 259 Sales (Accounting) / 260 Receipt Voucher / 261 Payment Voucher | that one type only | **yes**: the band is one pressed button, and Alt+n does nothing |
| 262 Voucher Register | every type `/vouchers/types` returns (**only those the user may view**) | no: the band is live |
| F8 / drill-in with a key | the loaded voucher's type | yes, until New |

- `GET /vouchers/types?companyId&menuId=<ctx.menuId>` returns the types, each
  with its rules and **the caller's rights on that type's menu**.
- The screen enables verbs **from those per-type rights**, not from
  `NexDialog`'s ctx flags. The ctx knows the menu the operator clicked (262
  for the register), which is not the menu the rights live on. It is the same
  lesson as the sale bill's `rights` block.
- With **no type returned** (a user with the register menu but no type
  rights), show one line, *"You have no voucher types on this menu. Ask for
  Journal, Contra … rights."*, and nothing else.

### 9.5 The `.ui`, top to bottom (`voucher_register_ui_mockup_rev2.png`)

1. **Header row:**
   - `lblTitle` "Purchase (accounting) — new" / "— pua00019";
   - the `AppSession` context line;
   - `lblStatus` pill: DRAFT (amber, "numbered at Post") / POSTED (green) /
     CANCELLED (grey, with "reversed by rev000NN").
2. **Type band `frameTypes`:**
   - a `QButtonGroup` of flat checkable buttons **built at runtime** from
     `/types`. A new type needs no client change; nothing about the band is
     in Designer except its frame;
   - at the right, "Opened from: …" (§9.4);
   - below, the **rule strip** `frameRules`: seven read-only caption-over-value
     labels (numbering · party · bill-wise · debit side · credit side · GST ·
     TDS), filled from `VoucherTypeRules`. The group lists come back as names
     from `/types`.
3. **Voucher band `grpVoucher`** (grid layout, three columns):
   - **Party** `txtParty` (FetchLineEdit → the party popup: `/vouchers/ledger-pick`
     with the party side). **Hidden for NONE; for MANY it reads "— several —"
     and is disabled**, since the parties are on the lines;
   - **Date** `dtDate` (NexDateEdit, required);
   - **Narration** `txtNarration`;
   - **Their bill no** `txtDocRefno` + **Bill date** `dtDocDate` (required when
     the type raises a PURCHASE / SALES bill; the captions come from the rules:
     "Supplier bill no" on the credit side, "Customer ref" on the debit side);
   - **GSTIN** `txtGstin` (read-only, from `/party-facts`);
   - **Place of supply** `ddPos` (dropdown 21; default = the party's state on
     SalA / DrN / CrN, the company's on PurA and when reverse charge is on).
     It is shown only when the type has a GST register;
   - the **facts line** `lblFacts` under it: "Credit N days · TDS <section>
     <rate> (from the supplier master) · outstanding X Cr".
4. **Legs grid `tblLegs`** (NexTable, ui_table §10), columns:
   `# · Dr/Cr · Ledger · Group · GST · HSN/SAC · TDS · Debit · Credit · Role · Leg narration`
   - **Typed rows** (`VoucherLine`) are editable.
   - **Derived rows** (`DerivedLeg` from the last `/validate`) are painted
     **grey with a lock glyph** and are never editable. Their narration's
     right edge says where they came from: "from the GST band" / "from the
     party's TDS section" / "balances the voucher".
   - GST and HSN/SAC are editable only on a typed line of a type with a GST
     register.
   - Role is display only (`av_role` chips on the derived rows).
   - A trailing "+ add line (Ins)" hint row, and a **totals row** with the
     **DIFFERENCE pill** (green at 0.00).
5. **Lower band `frameLower`, three cards side by side:**
   - **`grpGst`**: shown only when the rules have a GST register. It is a
     per-rate summary table (Rate · Taxable · CGST · SGST, or IGST · ITC
     combo), plus the **Reverse charge** switch (`chkRcm`, PurA only) and two
     hint lines. The ITC combo on a rate row sets `itcEligibility` on every
     typed line at that rate. Each line keeps its own value, defaulted from
     its ledger's `itcEligibility`.
   - **`grpBillwise`**: hidden when the rules say OFF.
     - RAISE: "Raises a new supplier / customer bill" + Bill · Amount · Due
       (days editable, default the party's credit days), and under it the open
       bills (`/vouchers/open-bills`) with a "This voucher" column to set part
       of it against an advance.
     - DEMAND: the open bills only, with a chip "allocated must equal the leg".
     - OPTIONAL (Journal): per selected party line.
   - **`grpRule`**: "THE ONE RULE — Σ DEBIT = Σ CREDIT", the two totals, and
     "Post stays disabled until the difference is 0.00".
6. **Footer row:**
   - the **`NexWarningStrip`** one-liner (the same widget as the six posting
     screens) for `/validate` refusals and warnings;
   - verbs: New · Save draft · **Post Ctrl+Enter** · Post & Print · Copy ·
     List F8, and **Cancel voucher** right-aligned in red (posted only).

### 9.6 The derive loop: the heart of the screen

The screen never computes a tax, TDS or party figure. **`/vouchers/validate`
is its calculator.**

```
any TYPED change (line added/removed, ledger, side, amount, GST rate, HSN, ITC,
party, date, place of supply, reverse charge, an allocation)
   → m_deriveTimer.start(400 ms)                 // debounce; restart on every change
   → timeout: ++m_generation; POST /vouchers/validate { payload, overrides }
   → reply: if (reply.generation != m_generation) drop it        // a newer edit is in flight
            paint derived legs, GST summary, TDS, party amount, bill card, totals,
            DIFFERENCE pill; strip.setFromResponse(reply, popup=false)
   → a 422 is the ANSWER (refusals painted in the strip), not an error box
   → no answer (server away): keep the last derived rows, dim them, say
     "Figures not refreshed — the server did not answer" and DISABLE Post
```

- **Post always re-validates**, because the server does. The strip's answer
  is shown to the operator before the post is sent (`confirmProceed`, the
  sale bill's rule: an operator overriding a WARN must have seen it).
- A Journal / Contra typed out of balance shows DIFFERENCE ≠ 0 from the
  derived totals. Post is disabled. A ONE-party type always balances through
  its party leg, so its DIFFERENCE is 0 whenever the server answered.
- **The only arithmetic in C++** is the sum of the typed Debit/Credit for the
  totals row **while a validate is in flight**, shown dimmed and replaced by
  the server's figures when they land. It is never posted.

### 9.7 Behaviour table

| Trigger | What happens |
|---|---|
| type button clicked (register menu only) | if any line is typed → confirm "Switch to <type>? The lines typed so far are cleared." Then load the rules; show/hide Party, GST card, bill-wise card; recaption the doc fields; re-scope the ledger picker; clear the derived rows |
| party picked | `/party-facts` → GSTIN, state (the POS default), credit days, TDS, outstanding → the facts line; `/open-bills` → the bill-wise card; derive |
| a line's Ledger cell | opens `VoucherLedgerPicker` over `/vouchers/ledger-pick?typeCode&side=<the line's Dr/Cr>&q=` (type-ahead 250 ms). It shows only legal ledgers; instrument-controlled ledgers are already excluded by the server. A picked ledger fills Group, the default GST rate (`defaultTaxId`) + ITC, and a TDS-base default (§9.8) |
| Debit typed on a line | clears Credit on that line, and the reverse: one side per line (in the delegate). The Dr/Cr combo follows the column typed |
| a typed amount / rate / HSN edited | derive (§9.6) |
| Ins / Ctrl+Del | add a typed line / remove the current typed line (derived rows cannot be removed; the cursor skips them) |
| a derived row double-clicked | a tooltip-style note: "Worked out from lines 1–2 — change those lines to change this" |
| DIFFERENCE ≠ 0, a refusal in the strip, no server answer, or no post right | **Post and Post & Print disabled**, tooltip naming which |
| Save draft | `/vouchers/create` (the typed payload only). Title → the draft's state; no number yet |
| Ctrl+Enter / Post | validate → `confirmProceed(verb)` if there are notes → `/vouchers/post` (key if it was a draft, + the typed payload + ticked overrides) → "Posted pua00019." → print if Post & Print → a fresh voucher **of the same type** (the party and date kept; the lines cleared) |
| Cancel voucher (posted, cancel right) | `AppMessage::askText` "Why is pua00019 being cancelled?" (required; presets "Keyed wrong", "Wrong party", "Duplicate", "Amount wrong") → `/vouchers/cancel` → reload → pill "CANCELLED · reversed by rev000NN" |
| Copy | a new DRAFT of the same type with the typed lines, party and narration; no number, no allocations, the date = today |
| F8 | `TxnMainView::openPicker(TxnDocTypes::voucher(<current type>))`: pick → `loadVoucher(key)` |
| Esc | backs out one level (picker popup → bill-wise card → grid → screen). A dirty screen asks before closing |

### 9.8 TDS on the screen

The TDS leg itself is a derived row. To let the operator say which lines the
TDS is on (`tdsBase` in the payload), the grid has **one narrow `TDS` tick
column** (on the PNG, between HSN/SAC and Debit). It is visible only when the rules say `DEDUCT` **and**
`/party-facts` says the party is TDS applicable. Its default per line = the
picked ledger's TDS flag (`led_is_tds_applicable`, returned by
`/ledger-pick`). It was added to the PNG on 2026-09-25 at the user's request (line 1 ticked, line 2 not, in the worked example).

### 9.9 Loading, reading, printing

- **`loadVoucher(key)`** → `/vouchers/get`:
  - a **DRAFT** repaints the typed lines from its stored payload, then
    derives;
  - a **POSTED / CANCELLED** voucher paints every leg exactly as written, with
    generated legs grey. It is read-only, with no derive and no edit;
  - `locks.editable` false → read-only;
  - the Cancel button follows `rights.cancel` and the posted state.
- **A cancelled voucher** shows its reversal's refno from
  `avh_reversal_voucher_id`. **Enter on it opens the reversal**, read-only,
  with its own "Reversal of pua00019" narration. **Reversal convention: the
  live one** (decision C2): type `Rev`, series `rev000NN`, dated **the
  original's date**.
- **Print:** no voucher print purpose or template exists yet. The print verbs
  are **visible and disabled** with "Printing arrives with the voucher print
  template". They never pretend (the sale bill's rule).

### 9.10 Registration (`form_accounts_menu.h`)

```cpp
// One screen, nine menus. The type menus open their own register (or the voucher,
// per AppSession::txnEntryFirst()) with the type FIXED; 262 opens with the band live.
fm.reg(103, [](const DlgContext &c){ return TxnMainView::openForMenu(TxnDocTypes::voucher("Jrl"),  c.parent); }); // Journal
fm.reg(104, … "Con" …);   // Contra
fm.reg(101, … "DrN" …);   // Debit Note
fm.reg(102, … "CrN" …);   // Credit Note
fm.reg(163, … "PurA" …);  // Purchase (Accounting)   — was "GST Expenses"
fm.reg(259, … "SalA" …);  // Sales (Accounting)
fm.reg(260, … "RcpV" …);  // Receipt Voucher          — menu 99 stays the bill-wise Receipt
fm.reg(261, … "PmtV" …);  // Payment Voucher          — menu 100 stays the bill-wise Payment
fm.reg(262, [](const DlgContext &c){ return TxnMainView::openForMenu(TxnDocTypes::voucherRegister(), c.parent); });
```

- `TxnDocTypes::voucher(typeCode)` is one descriptor factory.
  - Row identity: `avh_voucher_id / avh_company_id / avh_branch_id /
    avh_acc_year / avh_voucher_refno / avh_voucher_status` (the Receipt's
    fields).
  - The list grid: §10 #1, with its `itype_code` param set.
  - Its `createEntry` builds `VoucherRegisterEntry` with the type preset.
  - `voucherRegister()` is the same descriptor with `itype_code` empty (every
    permitted type) and a type filter combo.
- **Send every optional grid token, even empty** (grid-runner rule: an unsent
  token is a 400). The row policy: CANCELLED → no edit or delete; POSTED → no
  delete (Cancel instead).
- **Grant rights through the rights screen, never a partial API list**:
  `user_menus` save is a full replace. For testing, grant the test user every
  one of the nine menus in one save.

### 9.11 What the client must NOT do

- compute tax, TDS or the party amount (the derive loop, §9.6);
- send a derived leg;
- decide numbering, legal groups or rights;
- write tender rows;
- touch stock;
- post with a refusal standing or with no fresh server answer;
- keep any rule in a switch on the type code;
- offer a type `/types` did not return;
- print through anything but the print pipeline.

---

## 10. Grids, ui_tables, registration

| # | What | Notes |
|---|---|---|
| 1 | **Voucher list grid** (F8) | hidden `avh_voucher_id, avh_company_id, avh_branch_id, avh_acc_year`; then type, refno, date, party (or "— several —" / blank), narration, debit, credit, status badge, created by. **Filtered to the types the caller may view.** Params: `icompany_id, ibranch_id, iacc_year, ifrom_date, ito_date, itype_code, iuser_id`. Send **every** token; never cast a token (grid-runner rules) |
| 2 | **Exceptions grid** (S16 guard 5) | register vouchers (Jrl / DrN / CrN) whose allocations touch a SALES or PURCHASE bill in the period, so a hand-journalled settlement is visible |
| 3 | ui_table: **legs grid** | #, Dr/Cr, Ledger, Group, GST, HSN/SAC, TDS, Debit, Credit, Role, Leg narration, plus a hidden `generated` flag |
| 4 | ui_table: bill-wise panel | bill/ref, date, type, pending, this voucher, due |
| 5 | menus | §5.3; `menu_verbs` for each |
| 6 | Qt registration | `form_accounts_menu.h`: 101/102/103/104/163 + the three new menus + Voucher Register, all opening the same screen with a preset `typeCode`; the register menu opens with none |

The ledger picker, bills, balance and party facts are **routes** (§6), not
grids, per the user's own-URLs rule. The list and the exceptions report stay
grids, because they are lists.

---

## 11. Tests

### 11.1 Server (on `erp_dry`, then the box with `ZT-` data)

1. A 2-line Journal posts; a Journal unbalanced by 0.01 is refused by the
   service **and**, when forced past it, by `ck_avh_balanced` (trigger + check
   together).
2. An amount of 0 or negative is refused (`ck_av_amount`).
3. A Contra with a Sundry Debtors ledger → `VCH_LEDGER_SIDE`. Its party stays
   NULL, and the header saves (§5.1).
4. A multi-party Journal: `avh_party_id` NULL, two party legs, allocations on
   both, `abj_party_id` right on each.
5. **PurA, the PNG example:** 25,000 @ 18% + 4,000 @ 12%, TDS 194C 2% → legs
   exactly Input CGST 2,490 / Input SGST 2,490 / TDS 500 / party 33,480; a
   `PURCHASE` bill of 33,480 due +30 days; one `gdr_` + **two** `vtx_` rows
   (one per line, `vtx_row_no` = the line's row); one `acc_tds_register` row.
6. The same with `posStcd` ≠ the company state → IGST legs only.
7. RCM on: DR Input, CR `RCM_*_PAYABLE`; the party is owed the taxable only;
   `gdr_is_reverse_charge`.
8. A typed Input CGST line on PurA → `VCH_GST_LEDGER_TYPED`; the same ledger on
   a Journal → accepted.
9. A CrN with tax: output tax on **DR**; the party leg CR; `gdr_doc_type
   CREDIT_NOTE`. A CrN **without** GST lines writes **no** `gdr_` (S21).
10. RcpV with allocations short of the leg → `VCH_BILLWISE_SHORT`.
11. A Cheques-in-Hand ledger on any type → `VCH_INSTRUMENT_LEDGER`, with the
    menu 51/52 message.
12. **Rights (E):**
    - a user with create but not post on menu 163 can save a PurA draft,
      and gets 403 `VCH_RIGHT_POST` on post;
    - with no row for menu 163, `/types` omits PurA and `/vouchers/create`
      with `PurA` → 403;
    - a user with rights on 103 only, opening the register, sees Journal only.
13. A locked period (`fy_lock_date`) refuses post; a draft typed before the
    lock posts into an open date.
14. Two concurrent PurA posts get different numbers.
15. **Cancel (C):**
    - the reversal is a `Rev` voucher in its own series, linked both ways;
    - **the next PurA takes the next number** (no gap);
    - allocations reversed by counter-rows; the raised bill closed;
    - `gdr_` CANCELLED; `atd_` reversal row.
16. Cancel of a PurA whose bill a payment has already allocated →
    `VCH_ALLOCATED_ELSEWHERE`.
17. Cancel after a TDS challan → `VCH_TDS_DEPOSITED`.
18. A DRAFT writes **no** `av_` / `abj_` / `abl_` / `gdr_` / `atd_` row, and
    `avh_draft_lines` round-trips the payload byte-equal.
19. `/validate` returns the same derived legs `/post` then writes (compare row
    by row).
20. **The Receipt on the routine** posts identical `av_` / `abj_` / `abl_` rows
    to today's receipt path on the same input.
21. `/ledger-balance` equals the ledger statement's closing for the same date
    and `fn_ledger_book_balance` (fixed 2026-09-25).
22. With `accounts.reconcile_on_post` on, a posted PurA leaves
    `fn_books_reconcile` at zero difference for the supplier.

### 11.2 Client tests (`tests/register/`, offscreen, live box, `ZT-` data)

Run under `QT_QPA_PLATFORM=offscreen`, linking the build's objects minus
`main.cpp.o`, with the same harness as `tests/receipt`. **Run `ninja -n`
first**: it must say "no work to do", or the test links stale objects.

1. **Rights (E):**
   - opened from 103, only Journal shows and Alt+n does nothing;
   - opened from 262 as a user with rights on 103 + 104 only, the band has
     exactly Journal and Contra;
   - a user with create but not post on 163 can Save draft, Post is disabled,
     and the server's 403 is shown if forced.
2. **Rules drive the bands:** Contra hides Party, GST and bill-wise; Journal
   shows "— several —"; PurA shows everything.
3. **The PNG example:**
   - typing lines 1–2 (25,000 @ 18%, 4,000 @ 12%) with Sundaram Facility
     Services produces, from `/validate`, rows 3–6 exactly: Input CGST 2,490 ·
     Input SGST 2,490 · TDS 500 · party 33,480;
   - none of those rows is editable;
   - DIFFERENCE is 0.00.
4. Line 1 to 12% repaints rows 3–4 and the party amount. Reverse charge on
   repaints them as Input + RCM payable, and the party amount drops to the
   taxable less TDS.
5. **Stale answers:** five rapid edits send five validates, and only the last
   is painted (the generation counter).
6. **Server away:** the derived rows dim, the "not refreshed" line shows, and
   Post is disabled.
7. A line cannot hold both Debit and Credit.
8. The ledger picker offers no Sundry Debtors ledger on a Contra, and no
   Cheques-in-Hand ledger on any type.
9. A validate refusal (e.g. an unmapped GST ledger) disables Post and names
   the fix; clearing it re-enables Post.
10. Draft → close → F8 → reopen restores every typed line and allocation, and
    re-derives.
11. Post → the number appears → a fresh voucher of the same type with the
    party and date kept.
12. Cancel asks for a reason, refuses an empty one, and after cancel the pill
    reads "CANCELLED · reversed by rev000NN"; Enter opens the reversal
    read-only.
13. A posted voucher is fully read-only; no derive call is made.
14. The ui_table layout applies; with the layout call failing, the local seed
    still shows every column.
15. Print verbs are visible, disabled, and say why.

---

## 12. Order of work

1. **Prathap:**
   - dry-run `39_voucher_register.sql` on `erp_dry`, verifying the group names
     and the role CHECK values, then run it on the box;
   - lift the shared posting pieces into `common/posting` (§7.0);
   - build the routine and the routes;
   - register the grids and ui_tables;
   - pass the server tests (§11.1).
2. **Client:** code §9 against the live routes (check every key name against Swagger first); pass §11.2.
3. **Then:**
   - move Receipt onto the routine (§7.6, test §11.1 #20);
   - build Payment (menu 100) on it;

Better still, have the **Ledger Statement** live before step 2
(`../report/plan-backend-ledger-statement.md`), so every register posting can
be checked on screen.

---

## 13. Not in scope

- tenders on a register voucher (day close reads the ledger);
- items and stock;
- multi-currency;
- cost centres (S1);
- recurring or reversing-journal templates;
- amend of a posted voucher (R1);
- e-invoice / e-way portal calls (the `gdr_` row is written; the e-invoice
  screen calls the portal);
- print templates beyond a plain voucher print;
- the day-close table and lock;
- moving the sale bill fully onto the routine;
- the Tally export.
