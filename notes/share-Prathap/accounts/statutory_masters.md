# statutory_masters — where Indian law lives in the schema

**Revision 1, 2026-09-15. Reference, not a work order — read §0 first.** Written after the TDS/TCS decisions (notes 14, 15),
then reshaped the same day to follow how Tally and GoFrugal actually operate
(§0) — which added a whole layer we were missing: **per-party certificates**.

Scope: the parameters the **law** sets — rates, thresholds, limits, due dates —
as opposed to the parameters the **client** sets. No screens here; this is
where the numbers live, who may change them, and in what order they are
consulted.

---

## 0. WHEN to build this — mostly NOT now

**This document is a map, not a work order.** The user's position, 2026-09-15:
*"may be today we dont need this option but in future client must need this
thats why im asking"*. That is the right reading — no client has asked for a
§197 certificate or a 269ST refusal yet, and building all of it now would be
inventing requirements.

So the only question that matters today is: **what gets expensive if we wait?**

### Do now — two things, and both are nearly free  →  **BOTH ARE NOW SPECIFIED (2026-09-15)**

Item 1 is written into `../receipt/plan-backend-receipt.md` §2.14 and
`../payment/plan-backend-payment.md` §2.9 + §4.1. Item 2 is a runnable
migration: **`../27_tax_rate_effective_dates.sql`**. Notes (18).

| # | do it now | why it cannot wait | cost today |
|---|---|---|---|
| 1 | **Stamp WHY a rate applied**, not just the rate: `tdsRateSource` = `MASTER` / `CERTIFICATE` / `NO_PAN` / `BELOW_THRESHOLD` on the TDS line and in `acc_tds_register` / `acc_tcs_register` | A posted document can never be re-explained later. If a certificate layer arrives in 2028, every deduction made before it is indistinguishable from a master-rate one, and 26Q cannot report it. **Unrecoverable** | one `varchar(16)` column in tables we are creating anyway |
| 2 | ~~`tax_effective_from` / `_to` on `inventory.tax_rate_master`~~ → **DOWNGRADED 2026-09-15, see §6.2** | The premise was wrong: the government does not restate a slab from 5 % to 7 %. The slabs are stable; what moves is **which slab an item sits in**, and that edge is already dated by `inventory.item_tax_history` | `27_tax_rate_effective_dates.sql` still exists and is harmless, but it is optional |
| **2′** | **Make `inventory.item_tax_history` actually get written** | It is the right table with the right shape — `ith_item_id`, `ith_tax_id`, `ith_effective_from`, `ith_effective_to`, `ith_reason`, indexed on (item, from, to) — and it has **0 rows**. Nothing writes it when an item's tax changes, so the one dated edge that matters is empty | a write on item-tax change + closing the previous row |

### Add whenever a client actually asks — no retrofit cost

`statutory_limits` · `tcs_rates` · `acc_party_tax_certificate` ·
`statutory_pack` · `statutory_due_dates`.

All five are consulted **at entry time only**, so adding one later changes
nothing already posted — *provided* item 1 above is in place. That is the whole
reason item 1 is worth doing now: it is what keeps the door open for the rest.

`statutory_pack` is five columns and no dependencies, so add it with the first
statutory migration whenever that happens; there is no gain in adding it to an
empty set.

### What this buys us

When a client does ask — "my supplier has a lower-deduction certificate", "we
need the 269ST cash limit enforced", "the GST rate changed mid-year and my old
invoices reprinted wrong" — the answer is a known table and a known resolution
order, not a redesign. That is what this file is for.

---

## 1. How Tally and GoFrugal actually operate this

Worth stating plainly, because it is the model this document copies.

**Tally** separates the product from the law. A release carries a **Statutory
Masters version** ("stat file") that is updated *independently of the product
version* — when a Finance Act changes a rate, Tally publishes a new stat
version and installations pick it up without a product upgrade. On top of that:

- **F11 › Statutory & Taxation** is the *company's* switchboard — GST
  applicable, TDS applicable, TCS applicable, e-invoicing applicable with the
  turnover. That is a **choice**, not a law.
- **TDS Nature of Payment** is a real master, not a setting: section, payment
  code, rate for Individual/HUF vs others, exemption limit, effective from.
  Shipped with defaults, alterable by the user.
- **Per-party overrides exist and matter**: a §197 lower-deduction certificate
  on a ledger (number, rate, valid from/to, limit) beats the master rate.
- **Stat Adjustment** journals handle GST/TDS adjustment entries.

**GoFrugal** is config-driven and centrally pushed: numbered configuration IDs
for choices (the TCS basis is one of them — `receipt_flow.md` §1 already cites
config 76), tax masters with effective dates for rates, and **HQ pushes master
updates down to outlets** so every branch is on the same numbers.

**What both agree on, and what we adopt:**

1. the law ships with the product and is **versioned separately**;
2. what the *client* switches on is a different layer from what the *law* says;
3. a **party** can lawfully override a rate (a certificate), and that is a
   third layer again;
4. the document keeps what was applied, so history never moves.

That gives four layers, and everything in this file belongs to exactly one:

| layer | what it holds | who sets it | example |
|---|---|---|---|
| **L0 applicability** | is this law switched on for this company | the client | `comp_tds_applicable`, `comp_tcs_applicable`, e-invoice turnover crossed |
| **L1 the statutory pack** | rates, limits, due dates | **shipped by us**, effective-dated, versioned | `tds_rates`, `tcs_rates`, `statutory_limits` |
| **L2 party overrides** | a lawful exception for one party | the client, from a document the party gives them | §197 certificate, MSME status, PAN absent |
| **L3 the stamp** | what was actually applied | the posting engine | `tdsSection` / `tdsRate` on the line, the registers |

---

## 2. The test: who decides the number?

| the number is decided by | layer | home | temporal? | scope |
|---|---|---|---|---|
| **the client** — a preference or an applicability switch | L0 | `public.app_setting_def` + its values, company flags | **no** | company / branch |
| **the law, with dimensions** — a rate that varies by section, party type, goods class | L1 | a **typed master**: `accounts.tds_rates`, `accounts.tcs_rates`, `inventory.tax_rate_master` | **yes — effective dated** | national, optional company row |
| **the law, as a scalar** — a limit, a percentage, a number of days | L1 | `public.statutory_limits` (§5, NEW) | **yes** | national, optional state |
| **the law, as a calendar** — filing due dates | L1 | `public.statutory_due_dates` (§7, parked) | yes | national |
| **a lawful exception for ONE party** — a certificate they hand you | L2 | `accounts.acc_party_tax_certificate` (§6.3, NEW) | **yes** | one ledger |

**Why law must never live in `app_setting_def`:** it has no effective dates —
the columns are `asd_key`, `asd_data_type`, `asd_default_value`,
`asd_min_value`, `asd_max_value`, `asd_max_scope`, and nothing temporal. Edit a
rate there when the Finance Act changes and you **silently rewrite the past**:
last year's vouchers reprice, and a reprint disagrees with the document the
customer holds. Settings are for choices (`tcs_basis`, `receipt_bill_sort`,
`cheque_print_after_post`), never for rates.

**Why law must never be hard-coded:** the Finance Act moves annually, sometimes
mid-year by notification. A client cannot wait for a release to bill correctly —
but see §7: they should not be typing rates either.

---

## 3. The rule that makes all of it safe

> **Stamp what was applied, on the transaction.**

Every document records the section, rate and limit **as used at entry time** —
the TDS line already carries `tdsSection` / `tdsRate`, `acc_tds_register` and
`acc_tcs_register` carry section + rate + PAN, the voucher leg carries
`av_role`. A master is therefore consulted **only when the document is
created**, and amending a law never reaches a posted document.

Effective dates alone are **not** enough for this: without the stamp, a reprint
recomputes and can disagree with the paper the customer already has.

---

## 4. What exists, what is missing

| master | holds | state |
|---|---|---|
| `accounts.tds_rates` | section × deductee × date → rate, no-PAN 20 % (206AA), single + annual thresholds | **done** — `../payment/plan-backend-payment.md` §2.9 |
| `accounts.tcs_rates` | section × collectee × date → rate, no-PAN **1 %** (206CC), threshold | **MISSING — §6.1** |
| `accounts.acc_party_tax_certificate` | §197 / 206C(9) lower-or-nil certificates per party | **MISSING — §6.3.** Tally has it; without it the rate master is simply wrong for any party holding one |
| `public.statutory_pack` | which law version this installation carries | **MISSING — §8.** Tally's stat file number |
| `inventory.tax_rate_master` | GST slabs (11 rows, 21 inbound FKs) | exists, but **has no effective dates** — §5.2 |
| `public.statutory_limits` | 269ST, 40A(3), MSME days, e-invoice turnover, e-way value, interest rates | **MISSING — §4** |
| `public.statutory_due_dates` | GSTR-1 / 3B / challan / 26Q / 27EQ | missing, **parked** — §6 |

---

## 5. `public.statutory_limits` — the scalar law constants (NEW)

Most of Indian law that touches this product is a number with a section
reference. One table, not fifteen.

```sql
CREATE TABLE public.statutory_limits (
  stl_id             uuid PRIMARY KEY DEFAULT uuidv7(),
  stl_code           varchar(48)  NOT NULL,           -- see the seed list
  stl_label          varchar(160) NOT NULL,
  stl_value          numeric(16,4) NOT NULL,
  stl_unit           varchar(8)   NOT NULL,           -- AMOUNT | PERCENT | DAYS | COUNT
  stl_section_ref    varchar(24),                     -- '269ST', '40A(3)', '43B(h)', '206CC'
  stl_applies_to     varchar(24) NOT NULL DEFAULT 'ANY', -- CASH | GOODS | SERVICES | a state code
  stl_effective_from date NOT NULL,
  stl_effective_to   date NULL,
  stl_enforcement    varchar(8) NOT NULL DEFAULT 'WARN',  -- WARN | REFUSE | INFO
  stl_remarks        varchar(250),
  stl_is_active      boolean NOT NULL DEFAULT true,
  stl_is_deleted     boolean NOT NULL DEFAULT false,
  stl_created_on timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP, stl_created_by uuid,
  stl_modified_on timestamptz, stl_modified_by uuid,
  CONSTRAINT ck_stl_unit  CHECK (stl_unit IN ('AMOUNT','PERCENT','DAYS','COUNT')),
  CONSTRAINT ck_stl_enf   CHECK (stl_enforcement IN ('WARN','REFUSE','INFO')),
  CONSTRAINT ck_stl_dates CHECK (stl_effective_to IS NULL OR stl_effective_to >= stl_effective_from)
);

ALTER TABLE public.statutory_limits ADD CONSTRAINT ex_stl_overlap
  EXCLUDE USING gist (
    stl_code WITH =, stl_applies_to WITH =,
    daterange(stl_effective_from, coalesce(stl_effective_to,'infinity'::date), '[]') WITH &&
  ) WHERE (stl_is_deleted = false AND stl_is_active = true);
```

`stl_enforcement` is deliberately data: whether breaching a limit **refuses**
the document or only **warns** is a policy call that differs by client and by
limit, and it should not be a hard-coded `if`.

### 5.1 The seed list — and where each one is actually consumed

| `stl_code` | value | section | consumed by |
|---|---|---|---|
| `269ST_CASH_RECEIPT` | 2,00,000 | 269ST | **Receipt (99)** — cash received from one person in a day / against one document. **We have no such check anywhere in the receipt spec.** REFUSE |
| `40A3_CASH_PAYMENT` | 10,000 | 40A(3) | **Payment (100)** — cash paid to one person in a day, disallowed as expense. WARN |
| `40A3_CASH_PAYMENT_TRANSPORT` | 35,000 | 40A(3) proviso | Payment, when the payee is a transporter. WARN |
| `MSME_PAYMENT_DAYS` | 45 | 43B(h) / MSMED | **Payment + payables ageing** — and `led_msme_type` / `led_udyam_no` already exist on the ledger with nothing reading them. Sets the supplier bill's due date. INFO |
| `MSME_PAYMENT_DAYS_NO_AGREEMENT` | 15 | MSMED §15 | same, when there is no written agreement |
| `EINVOICE_TURNOVER` | 5,00,00,000 | Rule 48(4) | decides whether the e-invoice screens apply to this company at all |
| `EWAYBILL_VALUE` | 50,000 | Rule 138 | the e-way bill screens |
| `TCS_206C1H_THRESHOLD` | 50,00,000 | 206C(1H) | the sale bill / receipt TCS trigger — cumulative **receipts** per buyer per FY |
| `GST_INTEREST_PERC` | 18 | §50(1) | interest on late GST payment |
| `GST_INTEREST_UNDUE_ITC_PERC` | 24 | §50(3) | interest on ITC wrongly availed |
| `GST_LATE_FEE_PER_DAY` | 50 | §47 | return late fee (20 for nil) |
| `TDS_LATE_DEDUCT_INT` | 1 | 201(1A) | interest, per month, failure to deduct |
| `TDS_LATE_PAY_INT` | 1.5 | 201(1A) | interest, per month, deducted but not paid |
| `TDS_LATE_FEE_PER_DAY` | 200 | 234E | late filing of 26Q / 27EQ |
| `ROUNDING_288B` | 1 | 288B | tax amounts rounded to the nearest rupee |

**Every value above is the shape, not the authority.** Verify against the
current Finance Act and the GST notifications before loading — this document is
written from memory and the law moves.

---

## 6. The typed masters still needed

### 6.1 `accounts.tcs_rates` (NEW)

TCS is **not** TDS with the sign flipped, and the difference that matters here
is the no-PAN rule: **206CC is 1 %** (or twice the rate, whichever is higher),
where 206AA for TDS is 20 %. Riding TCS on `tds_rates` would bury that.

```sql
CREATE TABLE accounts.tcs_rates (
  tcr_id             uuid PRIMARY KEY DEFAULT uuidv7(),
  tcr_company_id     uuid NULL REFERENCES companys(comp_id),  -- NULL = every company
  tcr_section        varchar(16)  NOT NULL,      -- '206C(1H)', '206C(1)', '206C(1F)'
  tcr_section_name   varchar(120) NOT NULL,      -- 'Sale of goods over 50 lakh'
  tcr_collectee_type varchar(16)  NOT NULL DEFAULT 'ANY',
  tcr_rate           numeric(6,3) NOT NULL,
  tcr_no_pan_rate    numeric(6,3) NOT NULL DEFAULT 1,   -- 206CC — NOT 20
  tcr_threshold_annual numeric(14,2) NOT NULL DEFAULT 0,
  tcr_basis          varchar(8) NOT NULL DEFAULT 'RECEIPT', -- RECEIPT | SALES, the legal basis for the section
  tcr_effective_from date NOT NULL, tcr_effective_to date NULL,
  tcr_is_active boolean NOT NULL DEFAULT true, tcr_is_deleted boolean NOT NULL DEFAULT false,
  tcr_remarks varchar(250),
  tcr_created_on timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP, tcr_created_by uuid,
  tcr_modified_on timestamptz, tcr_modified_by uuid,
  CONSTRAINT ck_tcr_rate  CHECK (tcr_rate >= 0 AND tcr_no_pan_rate >= 0),
  CONSTRAINT ck_tcr_dates CHECK (tcr_effective_to IS NULL OR tcr_effective_to >= tcr_effective_from)
);
ALTER TABLE accounts.tcs_rates ADD CONSTRAINT ex_tcr_overlap
  EXCLUDE USING gist (
    coalesce(tcr_company_id,'00000000-0000-0000-0000-000000000000'::uuid) WITH =,
    tcr_section WITH =, tcr_collectee_type WITH =,
    daterange(tcr_effective_from, coalesce(tcr_effective_to,'infinity'::date),'[]') WITH &&
  ) WHERE (tcr_is_deleted = false AND tcr_is_active = true);
```

Seed: `206C(1H)` 0.1 %, no-PAN 1 %, threshold 50,00,000, basis RECEIPT.

**Note the distinction from the setting:** `tcr_basis` is what the *law* says
for that section; `accounts.tcs_basis` is what the *client* chose for how they
bill it (SALES for the clients of note 14). Both exist, and they are not the
same fact.

### 6.2 `inventory.tax_rate_master` effective dates — **downgraded to optional**

**The original argument here was wrong, and the user corrected it on
2026-09-15:** *"govt wont change tax completed from 5 to 7, it only happens by
items — for that we have item tax history table"*.

That is right. The GST **slabs** (0, 0.25, 3, 5, 12, 18, 28, plus zero-rated /
exempt / nil / non-GST) are structural and effectively permanent. What changes
by notification is **which slab an item sits in** — "these HSN codes move from
18 % to 12 %". That is a different edge, and it is already dated:

```
inventory.item_tax_history
  ith_item_id · ith_tax_id · ith_effective_from · ith_effective_to · ith_reason
  index (ith_item_id, ith_effective_from, ith_effective_to)
  FK → inventory.tax_rate_master        -- correct since 25_item_tax_to_tax_rate ran
```

So "what rate did this item carry on 12-08-2026" is answered **there**, not by
dating the slab row. And on the rare occasion a slab genuinely changes — a new
40 % demerit slab, or the 12 % band being collapsed — the answer is a **new
row** with `tax_supersedes_id` set and the old one deactivated, which the table
already supports.

`27_tax_rate_effective_dates.sql` therefore stays available but is **optional**:
two nullable columns that cost nothing and make an unusual case tidier. It is
not the thing that answers the history question.

**§6.2′ — the real gap: `item_tax_history` has 0 rows.**

The table is right and nothing writes it. To make it the answer it is meant to
be:

1. **write a row whenever `item_master.item_default_tax_id` changes** — the new
   `ith_tax_id`, `ith_effective_from` = the date the notification applies (not
   `now()` — a notification is usually dated), `ith_reason` = the notification
   reference;
2. **close the previous row** — `ith_effective_to` = the day before;
3. **decide whether billing resolves through it** (§6.2″).

A trigger on `item_master` is the obvious home for 1 and 2, since the item
screen, a bulk update and an import must all produce the history.

**§6.2″ — the decision worth taking with it.** A sale bill already **stamps**
its own tax (each line's rate and amounts, and `vtx_*` on the doc register), so
reprints and GSTR are safe regardless. Resolving through `item_tax_history` at
billing time matters for exactly two cases:

- a **back-dated document** — a bill dated before a rate change, entered after
  it. Without the lookup it takes today's slab, which is wrong;
- an **audit answer** — "prove why you charged 12 % on the 3rd of August".

Whether the sale bill does that lookup, or simply uses the item's current slab
and accepts that back-dated bills must be corrected by hand, is a real
decision — not a schema one. It has never been asked, and nothing forces it
now.

**The old §6.2 text follows, for the rare slab-change case:**

11 rows, 21 inbound FKs, so this is expand → migrate → contract, not a rebuild:

```sql
ALTER TABLE inventory.tax_rate_master
  ADD COLUMN tax_effective_from date,
  ADD COLUMN tax_effective_to   date;
-- backfill: the live rows take the company's books-begin date; superseded rows
-- take their successor's effective_from as their _to.
```

Documents already stamp the rate they used (`vtx_cgst_rate` etc. on
`acc_voucher_doc_detail`), so history is safe — this is about *new* documents
picking the right row, and about reporting being able to explain a change.

---

### 6.3 `accounts.acc_party_tax_certificate` — L2, the lawful exception (NEW)

**Tally has this and we do not**, and without it the rate master is wrong for
any party who hands you a certificate. A payee can lawfully be deducted at a
**lower or nil rate** under **§197** (TDS) or **206C(9)** (TCS) — they give you
a certificate with a number, a rate, a validity window and usually a **ceiling
amount**, and above that ceiling the normal rate resumes.

```sql
CREATE TABLE accounts.acc_party_tax_certificate (
  ptc_id            uuid PRIMARY KEY DEFAULT uuidv7(),
  ptc_company_id    uuid NOT NULL REFERENCES companys(comp_id),
  ptc_ledger_id     uuid NOT NULL REFERENCES accounts.acc_ledger_master(led_id),
  ptc_tax_kind      varchar(4)  NOT NULL,          -- TDS | TCS
  ptc_section       varchar(16) NOT NULL,          -- '194C', '206C(1H)' — the section it covers
  ptc_cert_no       varchar(32) NOT NULL,
  ptc_cert_date     date NOT NULL,
  ptc_rate          numeric(6,3) NOT NULL,         -- 0 = nil deduction
  ptc_ceiling_amount numeric(14,2) NOT NULL DEFAULT 0,   -- 0 = no ceiling
  ptc_valid_from    date NOT NULL,
  ptc_valid_to      date NOT NULL,
  ptc_is_active     boolean NOT NULL DEFAULT true,
  ptc_is_deleted    boolean NOT NULL DEFAULT false,
  ptc_remarks       varchar(250),
  ptc_created_on timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP, ptc_created_by uuid,
  ptc_modified_on timestamptz, ptc_modified_by uuid,
  CONSTRAINT ck_ptc_kind  CHECK (ptc_tax_kind IN ('TDS','TCS')),
  CONSTRAINT ck_ptc_rate  CHECK (ptc_rate >= 0),
  CONSTRAINT ck_ptc_dates CHECK (ptc_valid_to >= ptc_valid_from)
);
ALTER TABLE accounts.acc_party_tax_certificate ADD CONSTRAINT ex_ptc_overlap
  EXCLUDE USING gist (
    ptc_company_id WITH =, ptc_ledger_id WITH =, ptc_tax_kind WITH =, ptc_section WITH =,
    daterange(ptc_valid_from, ptc_valid_to, '[]') WITH &&
  ) WHERE (ptc_is_deleted = false AND ptc_is_active = true);
```

**The resolution order, and it is the whole point of the three layers:**

    1. L0  is the tax switched on for this company at all?      -> if no, stop
    2. L2  a live certificate for this party + section + date?  -> use its rate,
                                                                   until the ceiling is used up
    3. L1  the rates master for section x deductee x date       -> the normal rate
    4. L2' PAN absent on the ledger?                            -> no-PAN rate (206AA 20 % / 206CC 1 %)
                                                                   and a certificate cannot rescue it
    5. L3  stamp section, rate, and WHICH of the above decided it, on the line

Step 4 outranks step 2 deliberately: 206AA overrides a §197 certificate when
the PAN is missing. Getting that order wrong is the classic TDS defect.

The stamp in step 5 needs one more field than we planned — **why** this rate:
`tdsRateSource` = `MASTER` | `CERTIFICATE` | `NO_PAN` | `BELOW_THRESHOLD`. The
operator sees it, and 26Q needs it (a certificate deduction is reported with
the certificate number).

### 6.4 L0 — applicability already exists, mostly

`companys.comp_tds_applicable` and `comp_tcs_applicable` are Tally's F11 in
our schema, and they are there. What is missing at L0:

- **e-invoice / e-way applicability** — a company-level "turnover crossed"
  flag, or the turnover itself compared against `EINVOICE_TURNOVER` (§5.1);
- **the effective date of applicability** — a company that crosses the
  threshold mid-year is liable from a date, not for the whole year.

Both are small and belong with the GST provider config (`11_gst_provider.sql`),
not here. Named so they are not forgotten.

## 7. `public.statutory_due_dates` — parked, named

`(form, period_type, due_day, grace_days, effective_from/_to)` for GSTR-1 11th,
GSTR-3B 20th, TDS challan 7th, 26Q / 27EQ quarterly. Feeds a compliance
calendar later. Not needed by any screen in wave 1.

---

## 8. Who maintains these

**Ship statutory packs as migrations**, with `effective_from` set, not as "the
client types the new rates". A rate change is then a release, which is the
right amount of ceremony for something that changes what customers are charged,
and it keeps every installation on the same numbers.

**Version the pack separately from the product — this is Tally's stat file, and
it is why they can support thousands of installations.** One tiny table:

```sql
CREATE TABLE public.statutory_pack (
  spk_id           integer PRIMARY KEY,        -- the pack number, monotonic: 1, 2, 3…
  spk_label        varchar(80)  NOT NULL,      -- 'Finance Act 2026 — TDS 194Q revision'
  spk_applied_on   timestamptz  NOT NULL DEFAULT CURRENT_TIMESTAMP,
  spk_effective_from date       NOT NULL,
  spk_notes        varchar(500)
);
```

Every statutory migration inserts its row. Then support can ask **"which pack
are you on?"** and get an answer, the client can see it on the About screen,
and a wrong rate becomes a diagnosable fact rather than an argument. Without
this, the same product version can carry different law on two installations and
nobody can tell.

The client edits only where the law is genuinely a choice: composition scheme,
whether their turnover crosses the e-invoice threshold, `stl_enforcement` on a
WARN-level limit, and `accounts.tcs_basis`.

A **rate-master screen** (the `tax_rate_master` shape: grid + entry, read-only
for most users) is owed eventually for `tds_rates` / `tcs_rates` /
`statutory_limits`, so a client on an old build can be corrected without a
patch. Not in this phase.

---

## 9. Decisions

| # | decision | state |
|---|---|---|
| L1 | law never lives in `app_setting_def` — no effective dates there | **taken** |
| L2 | every document stamps the section / rate / limit it used | **taken** — already true of TDS, TCS and the registers |
| L3 | scalar limits share one `statutory_limits` table rather than a column each | **taken** |
| L4 | `stl_enforcement` (WARN / REFUSE / INFO) is data, not code | **taken** |
| L5 | `tcs_rates` is its own table, because 206CC ≠ 206AA | **taken** |
| L6 | `tax_rate_master` gains effective dates, expand → migrate → contract | **taken** |
| L7 | 269ST is a **refusal** on the receipt screen, not a warning | **proposed** — confirm |
| L8 | statutory packs ship as migrations, not client data entry | **proposed** — confirm |
| L9 | the due-dates calendar is parked | **taken** |
| L10 | four layers — applicability (L0) / pack (L1) / party certificate (L2) / stamp (L3) — and every parameter belongs to exactly one | **taken**, §0 |
| L11 | resolution order is **applicability → certificate → master → no-PAN**, with no-PAN outranking the certificate (206AA) | **taken**, §5.3 — getting this order wrong is the classic TDS defect |
| L12 | the line stamps **why** the rate applied (`MASTER` / `CERTIFICATE` / `NO_PAN` / `BELOW_THRESHOLD`), not just the rate | **taken** — 26Q needs it |
| L13 | the statutory pack is versioned separately from the product (`statutory_pack`) | **taken** — Tally's stat file; without it two installations can carry different law indistinguishably |
| L14 | e-invoice / e-way applicability is an L0 company fact with its own effective date, and lives with the GST provider config | **proposed** — confirm where it goes |
