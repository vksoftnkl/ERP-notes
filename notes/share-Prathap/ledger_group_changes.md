# acc_group_master / acc_ledger_master — what still has to change

**2026-09-17.** A three-way diff of the two account masters: table columns
against the save/GET DTOs against what the two client screens actually expose.

**Every ask here is a SCHEMA or CONTRACT ask.** 192.168.0.106 is a test box:
its schema is authoritative, its data is not. Row counts and ids appear below
only as evidence that a hole is reachable — they are not a cleanup list, and
nothing here asks you to repair a row. Where a count is the whole argument I
say so.

The measuring stick is twofold, because the two ends disagree about what
"complete" means:

- **a complete GST ERP** — can we produce GSTR-1 and GSTR-3B honestly, and
- **the Tally export**, which is where this all ends up. A Tally *company* is
  its own file, so exporting company X emits the shared (`company_id IS NULL`)
  masters **plus** X's own. That one sentence is behind half the items below.

Items are ordered by what blocks what, not by size. Each one says what it is,
the evidence, the change, and how to tell it worked.

---

## Status, 2026-09-17 evening

**Closed by Prathap the same day:** 1.1 (FK + all four orphan parties
backfilled), 1.2 (all four partial unique indexes + `tr_led_name_scope` /
`tr_acc_group_name_scope`), 1.3, 1.4 (`acc_group_child_ids` dropped from the GET
AND the table), 2.1, 2.2, 3.1.

**Closed client-side:** the group behaviour flags and the two new GST fields are
on their forms; 2.6's ship-to grid is built on ui_table 34 and ships read-only
pending notes (23).

**Still open:** 1.5 (cost centres), 1.6 (currency), 2.3 (dated GST details),
2.4 (statutory pack), 2.5 (credit limit), 3.2–3.5 — plus the VAPI row itself,
notes (24).

---

## 0 · Already done — please don't redo

**`22_retire_ledger_gst_columns.sql` is APPLIED.** `led_gst_rate`,
`led_taxability` and `led_tax_rate` are gone from the table, `ck_chg_tax_id` is
validated, and `/account-ledger-masters/create` now 400s on any of the three
("property ledGstRate should not exist"). The helper view
`public.vw_charge_tax_setup` has since been dropped, which is fine — it was
scaffolding for that file.

**The client side of it shipped today.** `ledger_entry.cpp` now has one Tax Rate
picker bound to `led_tax_id`, sourced from dropdown 36 (TAXES) exactly the way
the item master picks its default tax, and the three old fields are gone.
"HSN / SAC", "Duty Head", "Party Type" and "Registration Type" all stayed, per
your §1. The ledger GET's `ledTaxName` / `ledTaxRatePerc` / `ledTaxTaxability`
are what the form reads back — those are good, keep them.

Also fixed client-side while in there: the ledger TYPE is no longer derived from
`acc_ledger_profile` alone. It could not be — `SalesPurchase` is both Sales
Accounts (Income) and Purchase Accounts (Expenses), and `General` is every
income *and* expense group. It now reads `accGroupNature` as well. (There is
still one ledger under Sales Accounts stamped `EXPENSE` from the old rule; it is
harmless, but it is the fingerprint of the bug if you see it.)

**`23_ledger_party_fields.sql` is NOT applied** — no CHECK exists on
`led_gst_party_reg_type` and 6 live rows are still `REGULAR` / `UNREGISTERED`.
See §4: one premise in it needs correcting before you run it, the rest stands.

---

## 1 · Blocks the Tally export

### 1.1 — `cus_id` / `sup_id` are supposed to BE `led_id`, with no FK saying so

**Evidence.** `sales.customers` and `purchase.suppliers` have no foreign key to
`accounts.acc_ledger_master`. The convention holds for everything created since
June 2026 (when auto-creation went in), and for nothing before it — these four
are test rows, listed to show the gap was reachable, not to be repaired:

| party | id | created |
|---|---|---|
| `test2` (customer) | `019cae41-960e-789b-abf4-df9436dc7814` | 2026-03-02 |
| `SUN ELECTRONICS G` (customer) | `019d85b1-e237-7618-9bac-25f6666a1b9e` | 2026-04-13 |
| `GOOGLE INDIA PRIVATE LIMITED` (supplier) | `019cc2cb-71f6-7f91-9290-f9f56f6d6d44` | 2026-03-06 |
| `NSD ELECTRICAL` (supplier) | `019e6e0a-3c0e-7991-8d62-78852faca9ef` | 2026-05-28 |

Those four have **no ledger row at all**. They cannot be posted to,
cannot appear in a trial balance, and cannot be exported — `acc_bill_balance`,
`acc_vouchers` and `acc_voucher_header` all FK their party to `led_id`.

**The change.** Backfill the four, then add the FK so it cannot happen again:

```sql
ALTER TABLE sales.customers
    ADD CONSTRAINT fk_cus_ledger FOREIGN KEY (cus_id)
        REFERENCES accounts.acc_ledger_master(led_id)
        ON UPDATE CASCADE ON DELETE RESTRICT;

ALTER TABLE purchase.suppliers
    ADD CONSTRAINT fk_sup_ledger FOREIGN KEY (sup_id)
        REFERENCES accounts.acc_ledger_master(led_id)
        ON UPDATE CASCADE ON DELETE RESTRICT;
```

Add them `NOT VALID` first if the backfill is going to take a while, then
`VALIDATE` — but do not leave them off. The convention is load-bearing for the
whole accounts layer and nothing states it.

**Acceptance.**
```sql
SELECT count(*) FROM sales.customers c
  LEFT JOIN accounts.acc_ledger_master l ON l.led_id = c.cus_id
 WHERE l.led_id IS NULL AND NOT c.cus_is_deleted;   -- must be 0
```
and the same for `purchase.suppliers`.

### 1.2 — Ledger and group names are not unique in any way that matters

**This is not a request to scope `company_id`.** `NULL` means *shared by every
company* and that is deliberate — one customer billed in all four companies is
one ledger. 50 of 61 ledgers are shared, 11 are scoped (9 Acme Foods, 1 CHANDRA
SAW MILL, 1 HAP Solutions); all 48 groups are shared. Both modes are in use.

**Evidence.** `UNIQUE (led_company_id, led_name)` is **inert for shared rows**,
because Postgres treats NULLs as distinct — two shared ledgers named the same
thing both fit. And it says nothing at all about a scoped row colliding with a
shared one, which is what these two test rows demonstrate:

| led_id | name | group | company |
|---|---|---|---|
| `01a0a4c8-5f63-7e4b-a825-4ad402e2c84f` | VAPI BATTERY & AUTO ELECTRICALS | Suppliers | **shared (NULL)** |
| `01a0a4c7-77de-7d0b-91bc-510a244c796e` | VAPI BATTERY & AUTO ELECTRICALS | MUSIRI | **Acme Foods** |

The same trade partner is both a customer and a supplier, so it got two ledgers
with the same name in two different scopes. Working in Acme Foods you see both.
The API's 409 is scoped "for this company", so it passed them, and it will keep
passing this whole class.

`acc_group_master` has **no name uniqueness index at all** — only the pkey,
`parent_id`, `company_id` and `tally_guid`.

**Why it bites the export specifically.** Exporting Acme emits the 50 shared
ledgers plus Acme's 9 into one file. Tally merges two `<LEDGER>` entries with
the same `NAME` into one and combines their balances, silently.

**The change.** "Unique within what one company can see" is three rules, and an
index expresses only two of them:

```sql
-- 1 · no two SHARED ledgers share a name
CREATE UNIQUE INDEX uq_led_name_shared
    ON accounts.acc_ledger_master (lower(led_name))
 WHERE led_company_id IS NULL AND NOT led_is_deleted;

-- 2 · no two ledgers within the SAME company share a name
CREATE UNIQUE INDEX uq_led_name_company
    ON accounts.acc_ledger_master (led_company_id, lower(led_name))
 WHERE led_company_id IS NOT NULL AND NOT led_is_deleted;

-- 3 · a company-scoped ledger must not collide with a SHARED one
--     -> BEFORE INSERT OR UPDATE trigger. No index can say "must not match a
--        row in the other scope".
```

`lower()` on purpose: Tally matches ledger names case-insensitively, so
`VAPI Battery` and `VAPI BATTERY` are one ledger there and must be one here.

Same three for `acc_group_master`. And the API's duplicate check needs all
three, not just rule 2 — otherwise the DB starts rejecting things the API
happily accepted and the error surfaces as a 500.

**ANSWERED 2026-09-17 — see notes (24).** One party, one ledger: keep
`01a0a4c7-77de-7d0b-91bc-510a244c796e` (the customer's, under MUSIRI / Acme),
point `purchase.suppliers.sup_id` at it, soft-delete
`01a0a4c8-5f63-7e4b-a825-4ad402e2c84f`. Every FK count on both ids is zero, so
it is free today. The note also asks which of two shapes you want for the rule
that stops the next one — reuse-the-led_id, or a 409 the client turns into
"already exists as a customer, add supplier details to it?"

**Acceptance.** Both indexes create without error after the VAPI pair is
resolved; inserting a scoped ledger whose name matches a shared one is refused.

### 1.3 — Four Tally group flags exist as columns and nothing can set them

**Evidence.** These are on `acc_group_master`, and are in **neither DTO and on
no form**:

| column | Tally field |
|---|---|
| `acc_group_behave_as_subledger` | `ISSUBLEDGER` |
| `acc_group_net_debit_credit` | `NETDEBITCREDITFORREPORTING` |
| `acc_group_used_for_calculation` | `USEDFORCALCULATION` |
| `acc_group_affects_gross_profit` | `AFFECTSGROSSPROFIT` |

`SaveAccGroupMasterDto` is seven keys total: `accGroupId`, `accGroupName`,
`accGroupAlias`, `accGroupShort`, `accGroupDescription`, `accGroupParentId`,
`accGroupSort`. Every export will write Tally's default for all four.

`acc_group_is_active` is in neither DTO either, and the checkbox is commented
out in `account_group_entry.cpp` — **a group cannot be deactivated from the UI
at all.**

**The change.** Add the five to `SaveAccGroupMasterDto` and to
`AccGroupMasterPayloadDto`. No schema change; the columns are already there,
`NOT NULL DEFAULT false`. I'll put them on the form once they're in the DTO.

**Acceptance.** `POST /account-groups/create` with
`accGroupAffectsGrossProfit: true` round-trips through `/account-groups/get`.

### 1.4 — `acc_group_child_ids` is stale, and the GET marks it REQUIRED

**Evidence.** Nothing maintains the array. 21 of 48 groups have it NULL, 14 have
it `{}`, and where it is populated it disagrees with reality:

| group | `child_ids` says | actual children |
|---|---|---|
| Customers | 1 | **11** |
| Current Assets | 3 | **6** |
| Office accounts | 6 | **1** |
| Capital Account | NULL | 1 |

`AccGroupMasterPayloadDto` lists `accGroupChildIds` as **required**, so any
consumer that trusts it builds the wrong tree — including an export, which has
to emit parents before children.

**The change.** Either drop the column and the DTO key, or maintain it in the
same trigger that already does the nature inheritance
(`tr_acc_group_inherit_nature`). My preference is drop it: `acc_group_parent_id`
is indexed and a recursive CTE is the correct way to walk this.

**Acceptance.** No consumer references `accGroupChildIds`; the group list orders
by hierarchy from `parent_id`.

### 1.5 — `led_is_cost_center_req` promises a feature that does not exist

**Evidence.** Three-way dead end:

- `acc_ledger_master.led_is_cost_center_req` — a checkbox on the ledger form
  ("Cost Centres Applicable"),
- `accounts.acc_vouchers.av_cost_centre_id` — a column with **no foreign key**
  and **0 of 104 rows populated**,
- and **no cost centre master table anywhere in the database.**

**The change.** Decide, then do one of two things. Either build
`accounts.acc_cost_category` + `acc_cost_centre` with per-leg allocation (Tally
needs `COSTCATEGORY`, `COSTCENTRE` and `CATEGORYALLOCATIONS` for the voucher
export), or drop `av_cost_centre_id` and `led_is_cost_center_req` so the
checkbox stops promising it. Leaving it as-is is the only bad option.

### 1.6 — No currency master

No multi-currency and no forex gain/loss anywhere. Fine for domestic-only; a
blocker the day there is an export sale or an import purchase. Flagging it so
the decision is explicit rather than discovered.

---

## 2 · Blocks GST regardless of Tally

### 2.1 — No ITC eligibility flag anywhere in the database

**Evidence.** I grepped every column in every schema for
`itc|input_credit|eligib` — nothing. (`sales.customers.cus_itcoll_exempted` /
`cus_itcoll_type` are income-tax TCS, not input tax credit.)

**Why it matters.** Without it you cannot mark a purchase or expense ledger as
blocked credit under s.17(5) — motor vehicles, food and beverage, works
contract, personal consumption. **GSTR-3B table 4(D) "Ineligible ITC" cannot be
produced**, and 4(A) is overstated by exactly those amounts.

**The change.** On `acc_ledger_master`:

```sql
ALTER TABLE accounts.acc_ledger_master
    ADD COLUMN led_itc_eligibility varchar(20)
        CHECK (led_itc_eligibility IS NULL OR led_itc_eligibility IN
               ('ELIGIBLE','INELIGIBLE_17_5','INELIGIBLE_OTHER','CAPITAL_GOODS','INPUT_SERVICES'));
```

Tally's equivalent is the ledger's "Eligibility for input credit" under GST
details. Same list, same place.

### 2.2 — No reverse-charge flag on a party or an expense ledger

**Evidence.** `inventory.tax_rate_master.tax_is_reverse_charge` exists, and
`acc_voucher_doc_register.gdr_is_reverse_charge` exists on the document. There
is nothing that says *this supplier* or *this expense* is RCM.

That is the wrong level. RCM is decided by who you are buying from and what
(unregistered purchases, GTA, legal services, director's fees, import of
services) — not by the rate row, which is shared with ordinary forward-charge
sales at the same percentage.

**The change.** `led_is_reverse_charge boolean NOT NULL DEFAULT false` on
`acc_ledger_master`, and let the document flag default from it.

### 2.3 — GST details have no "applicable from" date

Tally stores a ledger's GST details as a **dated list** (`GSTDETAILS.LIST` with
`APPLICABLEFROM`), because a party's registration status changes. Ours is one
flat set of columns, so a party moving Unregistered → Regular in October
silently rewrites how every April invoice reads.

This is the largest of the GST items and the one I would think hardest about,
because it is a shape change (a child table), not a column. Worth deciding
before go-live rather than after.

### 2.4 — TDS and TCS are typed, not computed

**Evidence.** `led_tds_deductee_type` and `led_tds_nature_of_payment` are plain
`varchar` with no master behind them, and **no `tds_rates`, `tcs_rates` or
`statutory_*` table exists on the box** — those designs are in the share but
were never deployed. TCS is a bare boolean with no nature-of-goods.

So today TDS can only be typed in by hand, per voucher, from memory. Deploy the
statutory pack and point both columns at it.

### 2.5 — Credit limit lives where the accounting layer cannot see it

`cus_credit_amt_limit`, `cus_credit_bill_limit`, `cus_credit_days` are on
`sales.customers`; `sup_credit_days` on `purchase.suppliers`. Tally puts
`CREDITLIMIT` and `BILLCREDITPERIOD` on the **ledger**.

More to the point: `acc_bill_balance`, which does the ageing, FKs its party to
`led_id` and therefore cannot see a limit at all. Either move them to the ledger
or expose a view the accounts layer can join.

### 2.6 — `acc_ship_addrs` has a table and three live routes and no screen

`/ledger-shipping-addresses/create|get|delete` are live and
`SaveLedgerShippingAddressDto` exists. Nothing in the client touches it.

The table carries `saa_gstin`, `saa_state_code` and `saa_distance_km` — that is
**place of supply for a bill-to / ship-to split**, and the **e-way bill
distance**. Both are compliance, not convenience. I'll build the screen; no
backend change needed unless the GET needs a by-ledger list route.

---

## 3 · DTO tidy-ups (small, but do them before something depends on them)

### 3.1 — Six balance columns must come out of the ledger save DTO

`ledObAmount`, `ledObType`, `ledObAsOn`, `ledTotalDr`, `ledTotalCr`,
`ledTotalBalance` are all in `SaveAccountLedgerMasterDto`.

**A shared ledger spans companies, so a single balance on its row cannot be
right.** `accounts.acc_opening_balance` already keys on
`op_company_id + op_branch_id + op_acc_year` — that is the correct home and it
is already built and working (menu 55).

All six are `0` on every row today, so nothing depends on them yet. Take them
out of the save DTO now; leave them in the GET if anything reads them, but a
client must not be able to write them. `led_is_deleted` is already correctly
GET-only — same treatment.

### 3.2 — `ledCategory` is free text against a NOT NULL column

`led_category` is `NOT NULL DEFAULT 'GENERAL'` and REQUIRED in the GET payload,
but the form has a plain text box, so `''` gets written — 7 rows already hold
it. Either give it a CHECK and an enum the client can render as a combo, or drop
it from the DTO and let the default stand. Tell me which and I'll match.

### 3.3 — `ledAllowEdit` and `ledIsEntry` are undefined

Both are in the save DTO and appear **nowhere** in the client. If they mean
something, say what, and I'll put them on the form. If they don't, drop them.

### 3.4 — An invalid `ledLedgerType` returns 500, not 400

```
POST /account-ledger-masters/create  {"ledLedgerType": "NONSENSE", ...}
  -> 500 Internal server error, errors: []
```

`chk_led_ledger_type` is escaping raw. Every other bad value on this endpoint
gives a clean 400 with a field (`ledTaxId`, `ledGroupId`, `ledName` all do).
The client never sends a bad one, so this is cosmetic — but it is the one
endpoint response that tells you nothing.

### 3.5 — A root group cannot be created

`accGroupParentId` is required, so the 15 Tally primary groups can only ever
arrive from a seed. That is probably right — but if a deployment ever needs its
own top-level group, there is no way to make one. Worth a conscious "no".

---

## 4 · One correction to `23_ledger_party_fields.sql` before you run it

Your §1 premise — *"None of the 11 match, so the field renders BLANK on every
one of those ledgers — and saving then writes the blank back"* — **is not
true**, and has not been since 1 June 2026:

```cpp
// src/shared/widgets/nex_combobox.cpp
void NexComboBox::selectByText(const QString &text)
{
    const QString needle = text.trimmed().toLower();
    for (int i = 0; i < count(); ++i)
        if (itemText(i).trimmed().toLower() == needle) { setCurrentIndex(i); return; }
}
```

It is case-insensitive, so `'REGULAR'` selects `Regular` and `'UNREGISTERED'`
selects `Unregistered`. Nothing renders blank and nothing is being blanked on
save. Please don't spend time chasing that symptom.

**The rest of the file still stands and is worth running**: one vocabulary, a
CHECK behind it like every other combo-backed column on this table already has,
and the SEZ three-way contradiction tied together `NOT VALID`. Just re-check the
row counts first — it says 8 `REGULAR` / 3 `UNREGISTERED`, the box now has
**3 and 3** among live rows.

---

## Suggested order

1. **1.1** FK + backfill the four orphan parties.
2. **1.2** The three uniqueness rules, both tables, and the API check to match.
3. **3.1** Six balance columns out of the ledger save DTO.
4. **1.3** Five group keys into the group DTOs.
5. **2.1 / 2.2** ITC eligibility + reverse-charge columns.
6. **2.4** Deploy the statutory pack.
7. **1.5** Decide cost centres — build or delete.

1–4 are what make a Tally export possible at all. 5–6 are what make GSTR-3B
honest. 7 is the one that is currently lying to the operator.

Anything here that reads as a decision rather than a fix — the dual
customer/supplier naming in 1.2, cost centres in 1.5, dated GST details in 2.3,
`ledCategory` in 3.2 — send it back rather than guessing. Those four are ours to
answer.

