# Receipt voucher — backend plan (NestJS)

**For Prathap. Revision 3, 2026-09-14** — revision 2 folded in the 3.0 screen
and TallyPrime's Receipt; revision 3 folded in GoFrugal's receivables features
and, on the user's decision, is ONE phase: everything below is built together
(§10 lists what is deliberately NOT in it). `receipt_flow.md` §0a lists what
changed and why. Written against the schema at 192.168.0.106/ERP and the
conventions the existing routes follow (`/bills/create`,
`/tender-details/create`, `/transactions/party-balance`). Nothing is assumed;
the decisions are §11 and every one of them is TAKEN — this plan has no open
questions, only things you may disagree with.

That box is a TEST database: its schema is authoritative, its rows are not.

**The whole module in one line:** money received from a party, split across
instruments, allocated due-date-first against the party's open bills and any
credit the party holds, every non-money line a role-tagged leg, the remainder
held as an ADVANCE bill — one voucher for today's money, one more per
post-dated cheque dated the cheque, bills settling on maturity.

---

## 1. What exists, what does not

| | state |
|---|---|
| tables | all deployed |
| voucher legs written by anything | **never** — `acc_vouchers` has 0 rows; the receipt is the first writer |
| bill-balance trigger | **not deployed** (`receipt_flow.md` §1.3) |
| receipt voucher type | none |
| `acc_ledger_map` | **empty** — every role resolves to nothing |
| routes | none under `/receipts`; `/tender-details/*` and `/transactions/party-balance` reused |
| client | none; menus 99 / 48 hidden |

Because this is the first writer of legs, the header-total derivation
(`notes (4)`) and the `av_role` column are built here, and the ledger
statement (`../openingbal/opening_balance_flow.md` §6b) becomes testable the
day the first receipt posts.

---

## 2. Schema prerequisites — run before the module

### 2.1 Bill-balance trigger, partition-aware, post-dated-aware, plus the daily regularise

```sql
CREATE OR REPLACE FUNCTION accounts.fn_abl_recompute(p_bill_id uuid, p_bill_year char(9))
RETURNS void LANGUAGE plpgsql AS $$
BEGIN
  IF p_bill_id IS NULL THEN RETURN; END IF;
  UPDATE accounts.acc_bill_balance b
     SET abl_alloc_amount    = s.alloc,
         abl_disc_amount     = s.disc,
         abl_writeoff_amount = s.wrf,
         abl_settled_on      = CASE WHEN b.abl_bill_amount - (s.alloc + s.disc + s.wrf) <= 0
                                    THEN s.last_on ELSE NULL END,
         abl_modified_on     = now()
    FROM (SELECT COALESCE(SUM(abj_amount) FILTER (WHERE abj_adj_type IN
                    ('ALLOCATION','ADVANCE_ADJUST','NOTE_ADJUST','TRANSFER')),0) AS alloc,
                 COALESCE(SUM(abj_amount) FILTER (WHERE abj_adj_type = 'DISCOUNT'),0) AS disc,
                 COALESCE(SUM(abj_amount) FILTER (WHERE abj_adj_type = 'WRITEOFF'),0) AS wrf,
                 MAX(abj_adj_date) AS last_on
            FROM accounts.acc_bill_adjustment
           WHERE abj_bill_id = p_bill_id AND abj_bill_acc_year = p_bill_year
             AND NOT abj_is_deleted
             -- receipt_flow.md §5.3: a post-dated cheque settles on maturity, not at post
             AND NOT (abj_is_post_dated AND abj_adj_date > CURRENT_DATE)) s
   WHERE b.abl_id = p_bill_id AND b.abl_acc_year = p_bill_year;
END $$;

CREATE OR REPLACE FUNCTION accounts.fn_abj_refresh_balance()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  IF TG_OP = 'DELETE' THEN
    PERFORM accounts.fn_abl_recompute(OLD.abj_bill_id, OLD.abj_bill_acc_year); RETURN NULL;
  END IF;
  PERFORM accounts.fn_abl_recompute(NEW.abj_bill_id, NEW.abj_bill_acc_year);
  IF TG_OP = 'UPDATE' AND (NEW.abj_bill_id, NEW.abj_bill_acc_year)
                          IS DISTINCT FROM (OLD.abj_bill_id, OLD.abj_bill_acc_year) THEN
    PERFORM accounts.fn_abl_recompute(OLD.abj_bill_id, OLD.abj_bill_acc_year);
  END IF;
  RETURN NULL;
END $$;

CREATE TRIGGER tr_abj_refresh_balance
  AFTER INSERT OR UPDATE OR DELETE ON accounts.acc_bill_adjustment
  FOR EACH ROW EXECUTE FUNCTION accounts.fn_abj_refresh_balance();

-- Tally's "Regularized": once a day, every bill with a cheque maturing today.
CREATE INDEX ix_abj_pdc_maturing ON accounts.acc_bill_adjustment (abj_adj_date)
  INCLUDE (abj_bill_id, abj_bill_acc_year)
  WHERE abj_is_post_dated AND NOT abj_is_deleted;

CREATE OR REPLACE FUNCTION accounts.fn_abl_regularise_pdc(p_on date DEFAULT CURRENT_DATE)
RETURNS integer LANGUAGE plpgsql AS $$
DECLARE r record; n integer := 0;
BEGIN
  FOR r IN SELECT DISTINCT abj_bill_id, abj_bill_acc_year
             FROM accounts.acc_bill_adjustment
            WHERE abj_is_post_dated AND NOT abj_is_deleted AND abj_adj_date = p_on
  LOOP
    PERFORM accounts.fn_abl_recompute(r.abj_bill_id, r.abj_bill_acc_year); n := n + 1;
  END LOOP;
  RETURN n;
END $$;
```

Run `fn_abl_regularise_pdc()` from cron at 00:05 (and from `/receipts/post`
for any cheque dated today or earlier, so nothing waits for midnight).
`fn_abl_regularise_pdc(<date>)` also back-fills after downtime.

### 2.2 The voucher type row

`receipt_flow.md` §2 — one INSERT, `Rct`, nature RECEIPT, affects accounts.

### 2.3 R1 — one cheque table

```sql
ALTER TABLE accounts.acc_bill_adjustment DROP CONSTRAINT fk_abj_cheque;
ALTER TABLE accounts.acc_bill_adjustment
  ADD CONSTRAINT fk_abj_cheque FOREIGN KEY (abj_cheque_id, abj_cheque_acc_year)
      REFERENCES accounts.acc_pdc_register (apd_id, apd_acc_year)
      ON UPDATE CASCADE ON DELETE RESTRICT;

ALTER TABLE accounts.acc_pdc_register DROP CONSTRAINT ck_apd_dates;
ALTER TABLE accounts.acc_pdc_register
  ADD CONSTRAINT ck_apd_dates
  CHECK (apd_instrument_date >= apd_received_on - INTERVAL '3 months'
     AND apd_instrument_date <= apd_received_on + INTERVAL '12 months');
```

`acc_voucher_cheques` gets no rows from here on; dropping it is a separate
decision.

### 2.4 Header totals derived from legs (notes (4))

```sql
CREATE OR REPLACE FUNCTION accounts.fn_avh_refresh_totals()
RETURNS trigger LANGUAGE plpgsql AS $$
DECLARE v_id uuid; v_year char(9);
BEGIN
  IF TG_OP = 'DELETE' THEN v_id := OLD.av_voucher_id; v_year := OLD.av_acc_year;
  ELSE                     v_id := NEW.av_voucher_id; v_year := NEW.av_acc_year; END IF;
  UPDATE accounts.acc_voucher_header h
     SET avh_total_debit = s.dr, avh_total_credit = s.cr
    FROM (SELECT COALESCE(SUM(av_amount) FILTER (WHERE av_dr_cr = 'DR'),0) AS dr,
                 COALESCE(SUM(av_amount) FILTER (WHERE av_dr_cr = 'CR'),0) AS cr
            FROM accounts.acc_vouchers
           WHERE av_voucher_id = v_id AND av_acc_year = v_year AND NOT av_is_deleted) s
   WHERE h.avh_voucher_id = v_id AND h.avh_acc_year = v_year;
  RETURN NULL;
END $$;

CREATE TRIGGER tr_av_refresh_totals
  AFTER INSERT OR UPDATE OR DELETE ON accounts.acc_vouchers
  FOR EACH ROW EXECUTE FUNCTION accounts.fn_avh_refresh_totals();
```

The service never writes `avh_total_*`. **Order inside the posting
transaction:** header DRAFT → legs (trigger fills totals) → header POSTED
last, so `ck_avh_balanced` compares derived totals. Set POSTED before the legs
and it compares 0 with 0 — the vacuous pass `notes (4)` reported.

### 2.5 Settlement modes

```sql
ALTER TABLE accounts.acc_bill_adjustment DROP CONSTRAINT ck_abj_settlement_mode;
ALTER TABLE accounts.acc_bill_adjustment ADD CONSTRAINT ck_abj_settlement_mode
  CHECK (abj_settlement_mode IS NULL OR abj_settlement_mode IN
    ('CASH','CARD','UPI','WALLET','CHEQUE','BANK','CREDIT_NOTE','ADVANCE','LOYALTY',
     'VOUCHER','JOURNAL','DISCOUNT','WRITEOFF','MIXED','TDS','CLAIM'));
```

`abj_adj_type` is unchanged: TDS and a claim are full-value settlement
(`ALLOCATION`, folded into `abl_alloc_amount`); the mode says how.

### 2.6 `av_role` — the column that makes every future report a WHERE clause

```sql
ALTER TABLE accounts.acc_vouchers ADD COLUMN av_role varchar(30);
ALTER TABLE accounts.acc_vouchers
  ADD CONSTRAINT fk_av_role FOREIGN KEY (av_role)
      REFERENCES accounts.acc_ledger_role (alr_role) ON UPDATE CASCADE ON DELETE RESTRICT;
CREATE INDEX ix_av_role ON accounts.acc_vouchers (av_company_id, av_role, av_voucher_date)
  WHERE av_role IS NOT NULL AND NOT av_is_deleted;
```

NULL on party and instrument legs; the role on every other-ledger leg
(`receipt_flow.md` §4.5). TDS deducted this quarter =
`SUM(av_amount) WHERE av_role = 'TDS_RECEIVABLE'`.

And the one column a DRAFT needs — its other-ledger lines have no home until
they become legs at post (R10 keeps a draft off the bill tables):

```sql
ALTER TABLE accounts.acc_voucher_header ADD COLUMN avh_draft_lines jsonb;
```

Cleared at post. Nothing reads it but `/receipts/get` on a DRAFT
receipt.

### 2.7 Roles and the map

```sql
INSERT INTO accounts.acc_ledger_role (alr_role, alr_label, alr_want_nature, alr_want_type) VALUES
  ('TDS_RECEIVABLE',      'TDS deducted by customers',  'Assets',      NULL),
  ('BANK_CHARGES',        'Bank charges / MDR',         'Expenses',    'EXPENSE'),
  ('SURCHARGE_RECOVERED', 'Card surcharge recovered',   'Income',      'INCOME'),
  ('CLAIMS_ALLOWED',      'Customer claims allowed',    'Expenses',    'EXPENSE'),
  ('INTEREST_INCOME',     'Interest on overdue',        'Income',      'INCOME');
```

Then `acc_ledger_map` rows for the company: these five plus
`DISCOUNT_ALLOWED`, `WRITE_OFF`, `ADVANCE_RECEIVED`, `TCS_PAYABLE`, `ROUND_OFF`.
The map is empty today (`receipt_flow.md` §1.5); the map screen is deferred,
so by hand for now.

### 2.8 Settings — `public.app_setting_def` rows, `asd_max_scope = 'BRANCH'`, module `ACCOUNTS`

Keys are lowercase dotted like the live `system.regional`; values through
`app_setting_value` the way every other key resolves.

| key | `asd_data_type` | values | default |
|---|---|---|---|
| `accounts.pdc_posting_mode` | TEXT | ON_RECEIPT / ON_CLEARING | ON_RECEIPT (this plan refuses ON_CLEARING) |
| `accounts.receipt_bill_sort` | TEXT | DUE_DATE / BILL_DATE | DUE_DATE |
| `accounts.receipt_salesman_mandatory` | BOOL | | false |
| `accounts.writeoff_approval_above` | NUMBER | | 0 (every write-off needs an approver) |
| `accounts.tcs_basis` | TEXT | RECEIPT / SALES | RECEIPT — R15 |
| `accounts.ppd_slabs` | TEXT (JSON) | `[{"days":7,"perc":2},{"days":15,"perc":1}]` | `[]` — R16 |

### 2.13 Bill-wise TCS — `abl_tcs_amount` (NEW, 2026-09-15)

Some clients bill TCS on the **invoice** (`accounts.tcs_basis = 'SALES'`) and
need **bill-wise TCS outstanding and collection** — the user's requirement of
2026-09-15. Today `acc_bill_balance` stores only
`bill / alloc / disc / writeoff / pending` and **no tax breakup at all**, so
the question cannot be answered: the TCS sits inside `abl_bill_amount` and
nothing records how much.

**One column, no new table:**

```sql
ALTER TABLE accounts.acc_bill_balance
  ADD COLUMN abl_tcs_amount numeric(14,2) NOT NULL DEFAULT 0;
COMMENT ON COLUMN accounts.acc_bill_balance.abl_tcs_amount IS
  'TCS charged ON THIS BILL (tcs_basis = SALES). 0 on the RECEIPT basis, where
   TCS is raised by the receipt as a TCS_PAYABLE leg instead.';
```

**Who stamps it:** the **sale bill's** posting, when it raises the `abl_` row —
the same figure it already writes to `acc_voucher_doc_register.gdr_tcs_value`.
Nothing else writes it. (`sale_bill_posting_flow.*` needs this one line added.)

**The basis decides which mechanism applies, and they never both apply:**

| `accounts.tcs_basis` | where TCS lives | collection is | outstanding is |
|---|---|---|---|
| `RECEIPT` (default, 206C(1H) literal) | a `TCS_PAYABLE` leg on the receipt, money **on top of** the bill | the `av_role = 'TCS_PAYABLE'` adjustment rows | **there is none** — TCS is not charged until money arrives |
| `SALES` (**these clients**) | inside the invoice, inside `abl_bill_amount` | a **slice of the allocation** — the receipt raises no TCS line at all | `abl_tcs_amount` less what has been collected |

So on the SALES basis the receipt screen must **not** seed a TCS line
(`receipt_flow.md` §4.5 already conditions it on `tcs_basis = 'RECEIPT'` — that
stays exactly as it is). The customer simply pays a bill that happens to
include TCS.

**Collection is derived, not stored.** On the SALES basis TCS is a proportional
slice of the bill, so a second column would only be a second thing to keep in
step:

```sql
CREATE OR REPLACE VIEW accounts.v_bill_tcs AS
SELECT abl_id, abl_acc_year, abl_party_id, abl_company_id, abl_branch_id,
       abl_doc_refno, abl_doc_date, abl_bill_amount, abl_pending_amount,
       abl_tcs_amount,
       round(abl_tcs_amount * abl_alloc_amount   / nullif(abl_bill_amount,0), 2) AS tcs_collected,
       round(abl_tcs_amount * abl_pending_amount / nullif(abl_bill_amount,0), 2) AS tcs_pending
FROM   accounts.acc_bill_balance
WHERE  abl_tcs_amount > 0 AND abl_is_deleted = false;
```

Add an explicit `abl_tcs_collected` **only** if a client demands that TCS be
collected in full before anything else, rather than pro-rata — that is a real
but different requirement, and nobody has asked for it.

**The RECEIPT basis keeps its own view**, over the adjustment rows, since there
the TCS is its own leg:

```sql
SELECT abj.abj_bill_id, abj.abj_bill_acc_year, sum(abj.abj_amount)
FROM   accounts.acc_bill_adjustment abj
JOIN   accounts.acc_vouchers av ON av.av_id       = abj.abj_voucher_line_id
                               AND av.av_acc_year = abj.abj_voucher_acc_year
WHERE  av.av_role = 'TCS_PAYABLE' GROUP BY 1,2;
```

That works because the receipt writes **one adjustment row per settling
deduction per bill** — pinned or spread pro-rata (`receipt_flow.md` §4.4) — so
the per-bill split is stored, never inferred at report time.

### 2.14 `accounts.acc_tcs_register` / `acc_tds_register` — the filing projection (NEW)

A register IS worth a table, because 27EQ and 26Q need facts that live nowhere
on a voucher leg: section (`206C(1H)`, `194Q`, `194C`…), the rate applied,
**why that rate applied**, the collectee/deductee PAN, challan number + date +
BSR code, the quarter, and the certificate number issued.

**`*_rate_source varchar(16)` is not optional and goes in from day one:**

```
  MASTER            the rates master gave it
  CERTIFICATE       a §197 / 206C(9) certificate the party holds gave it
  NO_PAN            206AA (20 %) / 206CC (1 %) forced it
  BELOW_THRESHOLD   nothing was deducted — the threshold was not crossed
```

Only `MASTER`, `NO_PAN` and `BELOW_THRESHOLD` can occur today, because the
certificate table does not exist yet (`../accounts/statutory_masters.md` §0 —
deliberately not built). **The column is added now anyway**, because a posted
document can never be re-explained later: when certificates do arrive, every
deduction made before them would otherwise be indistinguishable from a
master-rate one, and 26Q reports a certificate deduction *with its certificate
number*. One `varchar(16)` now keeps that whole layer optional. Notes (17). Keyed `(av_id, av_acc_year)` where a leg exists, or
`(abl_id, abl_acc_year, voucher)` on the SALES basis where the collection event
is the **receipt of money against a TCS-bearing bill**.

Rules: **written at post, never edited, never re-summed.** The leg (or the
allocation) stays the source of truth; the register only adds the filing facts.
Challan and certificate columns are filled later by the return screen, which is
not in this phase.

**Legal note worth writing down:** under 206C(1H) the liability arises on
**receipt**, even for a client on the SALES basis who shows TCS on the invoice.
So on the SALES basis the register row is raised **at the receipt**, for the
TCS slice of that allocation — not when the invoice was raised.

**The 50-lakh threshold is computed, not stored.** 206C(1H) applies only above
₹50 lakh per buyer per financial year, and the trigger is cumulative
**receipts** from that buyer, not cumulative TCS. SUM over the year at retail
volumes; add an accumulator table only if it measurably hurts.

### 2.10 The approve right — **WITHDRAWN 2026-09-15**

**Do not add `um_can_approve`.** There is no approval step on a voucher any
more: *"remove approve option because we have posted its a double work"*
(user, 2026-09-15). A voucher that has not been POSTED is a DRAFT, and a DRAFT
affects nothing — that IS the unapproved state. Adding a second gate before it
is the same decision taken twice.

**The lifecycle is now DRAFT → POSTED → CANCELLED.** `APPROVED` stays legal in
`ck_avh_status` (no DDL change, and it costs nothing to leave), but **nothing
writes it**.

This does not touch the **write-off approver**, which is a different thing: a
recorded name on the line (`abj_approved_by`, `ck_abj_writeoff_approval`,
`accounts.writeoff_approval_above`). That stays — it is a fact captured about a
write-off, not a workflow state on the document. It used to lean on
`um_can_approve` for "who may"; with the right gone it is simply the name the
operator records, and the threshold still decides when it is demanded.

### 2.11 The customer ↔ ledger bridge — **WITHDRAWN 2026-09-15, there is no bridge to build**

**Do not add `cus_ledger_id`.** A customer and its ledger are the SAME ROW
IDENTITY: `sales.customers.cus_id` **is** `accounts.acc_ledger_master.led_id`,
because the id is generated by the ledger master when the customer is created
under Sales. `purchase.suppliers.sup_id` is the same id the same way (user,
2026-09-15). Verified on the live DB: the ids match one for one and the names
agree; neither `cus_ledger_id` nor `sup_ledger_id` exists, and neither should.

So there is nothing to resolve anywhere — **a customer id, a supplier id and a
party ledger id are one value.** Every endpoint below takes ONE `partyId`.

The only thing actually missing is the **declaration**: no FK says
`customers.cus_id → acc_ledger_master.led_id`, so nothing stops a customer row
being written with an id that is not a ledger (the test box already has old
rows like that). Worth adding, and it is a one-line change rather than a new
column:

```sql
-- after cleaning the pre-convention rows
ALTER TABLE sales.customers   ADD CONSTRAINT fk_cus_is_ledger
  FOREIGN KEY (cus_id) REFERENCES accounts.acc_ledger_master (led_id);
ALTER TABLE purchase.suppliers ADD CONSTRAINT fk_sup_is_ledger
  FOREIGN KEY (sup_id) REFERENCES accounts.acc_ledger_master (led_id);
```

### 2.12 Dropdown "CUSTOMERS BY AREA"

Dropdown 39's SELECT with one optional parameter:

```sql
SELECT cus_id, cus_name FROM sales.customers
 WHERE cus_is_active AND NOT cus_is_deleted
   AND (iarea_id IS NULL OR cus_area_id = iarea_id::uuid)
 ORDER BY cus_name
```

Registered through the dropdown API (id to be issued); the Beat picker is
dropdown 13 unchanged. Bare i-prefixed parameter, the way every dropdown
takes them.

### 2.9 Menu rows

Show 99 "Receipt"; retire 48 "Bill wise Receipt" — one screen, the bill-wise
panel is always there.

---

## 3. Module layout

```
src/modules/accounts/receipt/
  receipt.module.ts
  receipt.controller.ts
  receipt.service.ts               // §5.1 draft, get, header edit
  receipt-posting.service.ts       // §5.2 — THE transaction
  receipt-cancel.service.ts        // §5.3
  open-items.service.ts            // §4.1 — bills AND credits, due-date first
  allocation-engine.ts             // §5.2 step 5 — pure function: inputs → adjustment rows; unit-tested alone
  ppd-slab.ts                      // R16 — pure function: bill age + slabs → suggested discount
  dto/
    save-receipt.dto.ts            // header + tenders[] + otherLines[]
    post-receipt.dto.ts            // allocations[] + creditsApplied[] + onAccount
    receipt-payload.dto.ts
    open-item.dto.ts
```

`allocation-engine.ts` is deliberately a pure function — the screen previews
with the same rules the server posts with, so the two cannot disagree.

Reuse: the voucher-number allocator `/bills/create` uses; `BillAdjustmentService`
from `plan-nestjs-adjustments.md` for the credit pairs (row lock +
`ck_abl_settled` discipline); the `tender-details` DTO for `tenders[]`.

---

## 4. The endpoints

### 4.0 The identity of a voucher on every route — house convention (2026-09-15)

Three routes act on one existing voucher: **`/get`, `/post`, `/cancel`**
(`/approve` and `/reject` were withdrawn 2026-09-15 — see §2.10). All three
take the **same four keys, always**:

    companyId  ·  branchId  ·  accYear  ·  voucherId

Not a bare id, not an id plus year. The tables are partitioned by `acc_year`
and scoped by company and branch, so those four ARE the key — the same
`TxnDocKey` the client already carries for every other document (user,
2026-09-15). `party_id` is the other identity that matters, and there is only
ever one of those too (§2.11).

**And there is no `/list` route on any document.** Lists come from a registered
grid (`configured-grid-sql` / `grid-details`) that `NexApiTableView` drives —
every list in the product already works that way. What a new document needs is
a grid registration, not an endpoint.

### 4.1 `GET /api/v1/receipts/open-items`

Query: **`party_id`** (the one id — a customer, a supplier and a ledger are the
same value, §2.11), `company_id`, `on_date` (the receipt date — the PPD slab is
aged against it). No branch, no year. Nothing is resolved and nothing 404s for
a missing bridge, because there is no bridge. Returns

```jsonc
{ "bills":   [ { "billId", "billAccYear", "billType", "docRefno", "docDate", "dueDate",
                 "billAmount", "pendingAmount", "status", "daysOverdue", "pdcHeld",
                 "ppdSuggested",                  // R16: the slab's discount for this bill at on_date, else 0
                 "tcsAmount", "tcsPending" } ],   // §2.13 — 0 unless tcs_basis = SALES; the screen shows the
                                                  // bill-wise TCS the client asked for, from accounts.v_bill_tcs
  "credits": [ { "billId", "billAccYear", "billType", "docRefno", "docDate",
                 "pendingAmount", "srcDocType", "srcDocId" } ],
  "summary": { "totalPending", "billCount", "overdueCount", "creditsHeld", "pdcHeld" },
  "party":   { "ledId", "ledName", "groupName", "isBillByBill", "isTdsApplicable", "tdsDeducteeType",
               "isTcsApplicable", "tcsBasis", "tanNo" } }     // tcsBasis echoes accounts.tcs_basis
```

`receipt_flow.md` §4.3 is the SQL; sort by the `RECEIPT_BILL_SORT` setting.
**No paging cap.** `/transactions/party-balance` must call this rather than
keep its own SELECT — one answer to "what does this party owe and hold".

### 4.2 `GET /api/v1/receipts/party-context`

Query: `party_id`, `company_id`. The two 3.0 panels: `lastReceipts[]`
(`ix_avh_party`, last 10 POSTED Rct headers with date, refno, amount,
instruments summary) and `pendingCheques[]` (`acc_pdc_register` HELD /
DEPOSITED for the party, with instrument date, amount, bank, status). Read-only
context; no cap needed at 10 + open cheques.

### 4.3 `POST /api/v1/receipts/create` — the DRAFT

Upsert on `avhVoucherId`. Writes the header (DRAFT, no number), the tender
rows (`td_voucher_id` NULL) and the other-ledger lines into
`avh_draft_lines` (§2.6) — they become `acc_vouchers` rows only at post.
Nothing touches a bill (R10).

```jsonc
{ "avhVoucherId": "…",
  "avhCompanyId", "avhBranchId", "avhAccYear", "avhTenantId",
  "avhVoucherDate": "2026-09-14",
  "avhPartyId": "…",                 // THE id — customer / supplier / ledger are one value (§2.11)
  "areaId": "…",                     // the Beat picked, informational (the filter already happened in the dropdown)
  "avhEmployeeId": ["…"],
  "avhUsrRefno", "avhDocRefno", "avhDocDate", "avhRemarks",
  "avhDeviceType": "DESKTOP", "avhDeviceId", "avhSessionId", "avhUserId",
  "tenders": [ { "tdRowNo": 1, "tdTenderId", "tdTenderTypeId": 1, "tdTenderLedgerId",
                 "tdAmount": 5000, "tdReceivedAmt": 5000, "tdChangeAmt": 0 },
               { "tdRowNo": 2, "tdTenderId", "tdTenderTypeId": 3, "tdTenderLedgerId",
                 "tdAmount": 15000, "tdMdrAmt": 10, "tdRefNo": "UTR 6291…", "tdPayerVpa": null },
               { "tdRowNo": 3, "tdTenderId", "tdTenderTypeId": 5, "tdTenderLedgerId",
                 "tdAmount": 50000, "tdRefNo": "445123", "tdBankName": "KVB",
                 "tdInstrumentDate": "2026-09-20",
                 "cheque": { "bankBranch": "Trichy", "ifsc": "KVBL0001234",
                             "drawerName": "Sri Krishna Traders", "bankLedgerId": null } } ],
  "otherLines": [ { "role": "TDS_RECEIVABLE", "drCr": "DR", "amount": 700, "settlesBill": true },
                  { "role": "BANK_CHARGES",   "drCr": "DR", "amount": 10,  "settlesBill": true },
                  { "ledgerId": "…", "drCr": "DR", "amount": 350, "settlesBill": true,
                    "narration": "damage claim CN-88" } ],
  "replace": true }
```

`tdIsPdc` is never sent — computed. `BANK_CHARGES` from `tdMdrAmt` is seeded
by the server if the client omits it (the two must agree).

### 4.3a ~~`POST /receipts/approve` and `/reject`~~ — **WITHDRAWN 2026-09-15**

Do not build them. A DRAFT posts directly (§4.4); see §2.10 for why.

### 4.4 `POST /api/v1/receipts/post` — the one-way door

```jsonc
{ "avhCompanyId", "avhBranchId", "avhAccYear", "avhVoucherId",   // the four keys, §4.0
  "allocations":    [ { "billId", "billAccYear", "amount": 48600, "discount": 0,   "writeoff": 0 },
                      { "billId", "billAccYear", "amount": 11650, "discount": 500, "writeoff": 0,
                        "writeoffApprovedBy": null } ],
  "creditsApplied": [ { "billId", "billAccYear", "amount": 4000 } ],
                        // allocations[].discount carries the PPD-seeded figure AS THE OPERATOR LEFT IT — the server never re-seeds
  "otherLineBills": [ { "lineNo": 1, "billId", "billAccYear", "amount": 700 } ],   // optional per-bill pin for TDS/claims
  "onAccount": 14450 }
```

`amount` on an allocation is the bill's TOTAL settlement by this receipt
excluding discount/write-off (money + TDS + claims + credits — the engine
splits it into rows). The server recomputes everything and refuses a mismatch;
the client's figures are a preview, not an instruction. Returns both numbered
headers (receipt + PDC vouchers), the rows written, the ADVANCE bill id, the
register row ids.

### 4.5 `GET /api/v1/receipts/get`

`?companyId&branchId&accYear&voucherId` (§4.0). Header + `tenders[]` + `legs[]` (with `avRole`)
+ `allocations[]` (joined to bill refno/date, with `isPostDated` and
`matured`) + `creditsApplied[]` + `cheques[]` + `pdcVouchers[]` + the ADVANCE
bill. A POSTED receipt paints read-only from this.

### 4.6 ~~`GET /api/v1/receipts/list`~~ — **WITHDRAWN 2026-09-15: no list endpoint**

**Do not build a list route.** Document lists come from a **registered grid**
(`configured-grid-sql` / `grid-details`), which the client already drives with
`NexApiTableView` — the same way every other list in the product works (user,
2026-09-15). What is still needed is therefore a **grid registration**, not an
endpoint: key columns hidden (`avh_voucher_id, avh_company_id, avh_branch_id,
avh_acc_year`), then refno, date, party, `avh_doc_amount`,
`avh_adjust_amount`, on-account, instruments summary, PDC count, status
(DRAFT / POSTED / CANCELLED — filterable), created-by. PDC vouchers do NOT list separately — they show under
their receipt (`avh_against_voucher_id`). Grid id to be issued.

### 4.7 `PUT /api/v1/receipts/update-header`

`{ avhCompanyId, avhBranchId, avhAccYear, avhVoucherId, avhRemarks?, avhUsrRefno?,
   avhDocRefno?, avhDocDate?, avhEmployeeId?, editRemark }` on a POSTED receipt — nothing else accepted;
`avh_modified_*` stamped; `editRemark` to the audit trail.

### 4.8 `POST /api/v1/receipts/cancel`

`{ avhCompanyId, avhBranchId, avhAccYear, avhVoucherId, reason }` (§4.0) — §5.3. Refused when any cheque is
DEPOSITED or later, or the on-account credit has been used.

---

## 5. Service logic

### 5.1 Draft save

1. Party: live ledger visible to the company; DECISION R5.
2. Date in an OPEN, unlocked year.
3. Tender rows: master live for company/branch; ledger snapshot server-side
   unless `tnd_edit_ledger`; `tdAmount > 0`; a CHEQUE row carries `tdRefNo`,
   `tdInstrumentDate`, `cheque.bankName`; `tdIsPdc` computed; surcharge from
   the master unless `tnd_edit_surcharge`.
4. Other-lines: a role must exist and be mapped; a free ledger must be live
   and not the party; `amount > 0`. Seed rules the client mirrors: a
   `TDS_RECEIVABLE` line when the party is TDS-applicable; a `TCS_PAYABLE`
   line only when the party is TCS-applicable AND `accounts.tcs_basis =
   'RECEIPT'`; a `BANK_CHARGES` line from every `tdMdrAmt > 0`. The client may
   omit them; the server adds what is missing and refuses what disagrees.
5. `avh_doc_amount = Σ tdAmount` — derived. Header DRAFT, no number,
   `avh_draft_lines` = the other-lines. Replace semantics. One transaction.

### 5.2 Post — in this order, in one transaction

1. Load the receipt `FOR UPDATE`; **refuse unless DRAFT.** (A new receipt that
   was never saved is created as a DRAFT by `/create` first — Post is always
   post-from-draft.)
2. Re-validate 5.1. Salesman mandatory per setting. Every date (receipt AND
   each PDC's instrument date) in an OPEN, unlocked year.
3. **Lock the open items:** `SELECT … FOR UPDATE` on every bill and credit
   named. Per bill: `amount + discount + writeoff ≤ abl_pending_amount` NOW;
   per credit: `amount ≤ pending` NOW; else 409 naming the item and its
   current pending.
4. **The identity** (`receipt_flow.md` §4.5), to the paisa:
   `Σ tdAmount + Σ DR-other + Σ creditsApplied = Σ allocations + onAccount + Σ CR-other`.
   Write-off needs `approvedBy` (and above the threshold, always). Refuse on
   any mismatch.
5. **Allocation engine** (pure): with the order credits → settling
   deductions → instant money → cheques current-dated → post-dated by date,
   walking the bills in list order, produce the `acc_bill_adjustment` row set
   of `receipt_flow.md` §4.4 — one MIXED row per bill for instant money, one
   row per cheque per bill, one row per settling deduction per bill (pinned
   or pro-rata), one PAIR per credit applied. Also produce: which rows belong
   to which voucher (receipt vs each PDC voucher), and the on-account split
   (instant money's remainder vs each cheque's remainder).
6. **Register rows:** one `acc_pdc_register` per CHEQUE tender row (§4.6 of
   the flow); `apd_posting_mode` from the setting (refuse ON_CLEARING).
7. **Numbers:** allocate for the receipt and for each PDC voucher (dated the
   cheque) through the shared allocator. A DRAFT never holds a number
   (`ck_avh_no`), so the number is always taken here.
8. **Headers:** the receipt (DRAFT already), one DRAFT header per post-dated
   cheque with `avh_voucher_date = instrument date`,
   `avh_against_voucher_id` = the receipt, `avh_party_id`,
   `avh_doc_amount` = the cheque.
9. **Legs** (`receipt_flow.md` §4.2): per voucher, `av_row_no` 1..n; DR per
   instrument (cash net of change; bank net of MDR — the `BANK_CHARGES` leg
   is the other half of that split and stays outside the identity in step 4;
   clearing ledger when the master has one); DR/CR per other-line with
   `av_role`; the credit-applied
   pair (DR party / CR the credit's liability ledger via `ADVANCE_RECEIVED` or
   the credit note's ledger, `av_role` set); ONE CR party for the rest.
   Trigger 2.4 derives the totals.
10. **Adjustments:** write the engine's rows; `abj_voucher_id` = the voucher
    the row belongs to; post-dated rows `abj_is_post_dated = true`,
    `abj_adj_date` = cheque date. Trigger 2.1 moves the caches; matured rows
    (cheque dated today or earlier) count immediately; the rest wait for
    `fn_abl_regularise_pdc`.
11. **On account:** one ADVANCE bill per voucher that has a remainder
    (receipt-dated for instant money, cheque-dated for a PDC's remainder).
12. **Tender rows:** `td_voucher_id` = the voucher carrying that instrument.
13. `apd_voucher_id` = the voucher carrying the cheque's legs.
14. **Headers to POSTED, last:** `avh_adjust_amount` = Σ of that voucher's
    settling rows, `avh_opposite_ledger_id`, `avh_round_off`, status,
    `avh_posted_on / _status_on / _status_by`, `avh_draft_lines = NULL`.
    `ck_avh_balanced` and `ck_avh_no` fire here, on derived totals.
15. Call `fn_abl_regularise_pdc(CURRENT_DATE)` for cheques dated today.

Return both numbered headers. A failure after step 7 rolls the numbers back
with the transaction — keep the allocator inside it.

### 5.3 Cancel

`receipt_flow.md` §6: for the receipt AND each PDC voucher, a reversal header
(`Rct`, `avh_against_voucher_id` back, mirrored legs incl. `av_role`), one
negative row per adjustment row, ADVANCE bills soft-deleted (refused if
touched), register rows → CANCELLED (refused if past HELD), tender rows
soft-deleted, originals → CANCELLED with reason and `avh_reversal_voucher_id`.
One transaction.

### 5.4 Header-only edit

§4.7. Whitelist the four fields; anything else in the body is a 400, not
ignored.

### 5.5 What is NOT posted

No `acc_voucher_doc_register` (until GST on advance); no `acc_opening_balance`
change; no `acc_voucher_cheques` row (R1); no unreferenced on-account (R11).

---

## 6. Rules the DATABASE enforces

`receipt_flow.md` §7. Remember while writing: `ck_abj_against` (who may name
an opposite bill), `ck_td_pdc` (a PDC has a date), `ck_apd_posting`
(ON_RECEIPT names its voucher), and the new `fk_av_role`.

## 7. Rules ONLY the service can enforce

`receipt_flow.md` §8, all ten. Plus for the list: a CANCELLED receipt still
lists, with its reversal beside it.

---

## 8. Conventions to follow

1. camelCase keys; the `td*` set identical to `/bills/create`.
2. Upsert on id presence; `replace: true` on child arrays.
3. Nulls writable.
4. `GET /get` by id + partition year.
5. Numeric as number, dates `yyyy-MM-dd`.
6. No cache on open-items / party-context.

---

## 9. Test checklist

- Draft: header + tenders + `avh_draft_lines`; no legs, no adjustments, no register rows, no number.
- Post on the worked example: receipt with 8 legs (totals 25,200 / 25,200 derived), PDC voucher rct00018 dated 20-09 with 2 legs (50,000 / 50,000), 7 adjustment rows, 1 register row HELD, 1 ADVANCE bill 14,450 dated 20-09 on the PDC voucher; `avh_adjust_amount` 24,700 / 35,550.
- Before 20-09: bill 1 pending 23,900, bill 2 pending 11,650, `pdcHeld` 23,900 / 11,650; the credit note pending 0. On 20-09 after `fn_abl_regularise_pdc`: both CLOSED. `ck_avh_balanced` fails if any leg is dropped.
- A cheque dated today posts as part of the receipt (no PDC voucher) and settles immediately.
- Two concurrent posts against one bill: one 409, never a 500.
- The identity off by one paisa is refused; a client `onAccount` that disagrees with the server's is refused.
- TDS line with no `TDS_RECEIVABLE` mapping refused by role name; write-off without approver refused before the constraint; a free-ledger line naming the party refused.
- TDS pinned to one bill lands on that bill; unpinned spreads pro-rata and sums exactly.
- `tdMdrAmt` 10 produces the BANK_CHARGES leg and a bank DR of 14,990 while the party CR carries 15,000.
- Same cheque number, same party, same year: refused (`ux_apd_instrument`). A cheque dated 4 months before receipt: refused (`ck_apd_dates`).
- ON_CLEARING in the setting: post refused with the message, nothing written.
- Cancel: reversal for BOTH vouchers, bills reopen to 48,600 / 12,150, credit note back to 4,000, ADVANCE bill gone, register row CANCELLED, ledger nets to zero. Refused once DEPOSITED; refused once the ADVANCE bill was adjusted.
- Header edit changes narration and nothing else; a body with `tenders` is a 400.
- `open-items` lists an OPENING bill beside a SALES bill, credits in their own array, due-date order; a CLOSED bill absent; `pdcHeld` present.
- A receipt with no instruments and only a credit applied is refused ("journal").
- Salesman mandatory refuses without one when the setting is on.
- Post runs straight off a DRAFT; there is no approval gate to test (§2.10).
- PPD: bil00031 aged 6 days at on_date returns `ppdSuggested` 200 with slabs `[{7,2},{15,1}]`; aged 10 days → 100; aged 20 → 0. The operator's cleared discount posts as 0 — the server never re-seeds.
- `accounts.tcs_basis = 'SALES'`: no TCS line is seeded on a TCS-applicable party; `'RECEIPT'`: one is.
- open-items takes one `party_id`; there is no customer→ledger resolution step to test (§2.11).
- Dropdown "CUSTOMERS BY AREA" with `iarea_id` returns only that area's customers; without it, all.
- Bank charges 10 never enters the identity: RECEIVED 70,000 + DEDUCTIONS 1,400 + CREDITS 4,000 = ALLOCATED 70,750 + ON ACCOUNT 4,650.

---

## 10. One phase — and what is deliberately NOT in it

**In (all together):** §2 in full (schema, roles, the ten map rows, the
settings, the dropdown); every
endpoint in §4 — open-items, party-context, create, post,
get, list, update-header, cancel; the formal receipt print (`avh_print_count`,
a dataset in the printing schema 17); ON_RECEIPT only.

The first receipt against an opening bill is the first end-to-end proof of
`../openingbal/`.

**Not in this plan, by decision (2026-09-14):**

| what | why not | where it lives |
|---|---|---|
| Received Cheques screen (menu 51): deposit / clear / bounce / replace | its own screen on the register rows this leaves | next plan |
| ON_CLEARING posting mode | moves allocation into the cheque screens | with menu 51 |
| Payment (menu 100) | the mirror — `td_dr_cr = 'CR'`, `apd_tra_type = 'P'`, PURCHASE bills, TDS payable; same module, one direction flag | next plan |
| Collection Entry / Approval (187/188) | its own screen; creates receipt DRAFTs with `avh_src_doc_type = 'COLLECTION'` | later |
| batch posting of many receipts | dropped — N receipts are N posts | — |
| bank reconciliation screen | skipped; `av_recon_date` is the hook | — |
| day-close lock | skipped; `fy_lock_date` is the only lock | — |

---

## 11. Decisions — all taken

| # | decision | taken |
|---|---|---|
| R1 | cheque table | `acc_pdc_register` only |
| R2 | PDC | own voucher dated the cheque; bills settle on maturity; setting default ON_RECEIPT; ON_CLEARING with menu 51 |
| R3 | POSTED receipt | header-only edit; money = cancel + re-enter |
| R4 | credits inside a receipt | yes |
| R5 | party picker | PARTY ledgers; any ledger with an empty panel |
| R6 | collected-by | one salesman; mandatory by setting |
| R7 | remainder | always an ADVANCE bill |
| R8 | TDS/TCS/charges/surcharge/claims | other-ledgers band, roles, `av_role` |
| R9 | customer ↔ ledger bridge | **RE-TAKEN 2026-09-15: there is no bridge** — `cus_id` IS `led_id` (§2.11). One `partyId` everywhere |
| R10 | draft allocations | nowhere; other-lines in `avh_draft_lines` |
| R11 | unreferenced on-account | not offered |
| R12 | bill sort | due date; setting |
| R13 | collection import | own screen, later |
| R14 | approval | **RE-TAKEN 2026-09-15: there is none.** DRAFT → POSTED → CANCELLED; a DRAFT is the unapproved state, so a second gate is the same decision twice (§2.10). The write-off approver field is unaffected |
| R15 | TCS basis | `accounts.tcs_basis`, default RECEIPT |
| R16 | prompt-payment discount | `accounts.ppd_slabs` seeds; operator decides |
| R17 | party picker | customer via dropdown 39 + `iarea_id`, Beat narrows; the picked id IS the ledger id — nothing to resolve (§2.11) |
| R18 | phasing | one phase; batch / bank recon / day-close out |

---

## 12. Do not build

- **No in-place edit of money** on a POSTED receipt.
- **No stamped totals** — `avh_total_*` by trigger, `avh_doc_amount` from tenders, `avh_adjust_amount` from adjustments.
- **No adjustment rows for a DRAFT**; no writes to `abl_alloc/_disc/_writeoff` — the trigger owns them.
- **No allocation logic in two places** — `allocation-engine.ts` is the only one; the client previews with its port.
- **No ledger-name lookups** — roles through the map, `av_role` on the leg.
- **No `acc_voucher_cheques` rows.**
- **No second SELECT for "what does this party owe"** — `open-items` is it; `party-balance` calls it.
- **No paging on open-items.**
- **No multi-company receipt** (3.0's regional mode) — one company per receipt.
- **No fixed TDS field on the header** — it is a line with a role.
- **No batch endpoint** — dropped; a collection screen posts N receipts.
- **No re-seeding of the PPD discount at post** — the slab suggests once, on open-items; what the operator left is what posts.
- **No approval screen** — approval is a right on menu 99 and a filter on its list; menu 188 is for field collections.
