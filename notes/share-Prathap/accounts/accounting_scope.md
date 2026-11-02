# accounting_scope — every form a complete accounting system needs, and which of them we have

**Written 2026-09-15. Revision 3** (rev 1 and 2 the same day; **rev 3** adds
§2.5 — tenders are instrument metadata, **day close is ledger-based** — which
settles S18/S19, upgrades the day-close row in §6 into a described form, and
takes the tender invariant back out of the register. Rev 2 folds in the
**voucher register** model after the observation that GoFrugal carries Sales,
Purchase, Receipt, Payment, Debit Note, Credit Note, Contra and Journal as ONE
form with the type as the only variation, and books GST expenses as a Purchase
voucher — see §2, which changes §3, §9 and §10). Reference document, not a build order for
one screen: it answers one question only — *what is still missing before
NexERP can be called an accounting system* — by listing the complete form set
of the two products we are measured against, and marking each row with what we
actually have.

This is a **scope** document. It contains no DDL, no endpoints and no screen
layouts. Nothing here is a task for Prathap yet; the per-form plans
(`../openingbal/`, `../receipt/`, `../cheques/`, `../payment/`) are where the
work is specified, and `../notes.txt` is where it is asked for.

**How the comparison was made:** TallyPrime / Tally ERP 9 voucher, master,
utility and statutory menus; GoFrugal (RPOS / HQ / ServeEasy) accounts,
payables and day-close modules; and NextLvl 3.0 read from source —
`pos/Accounts/` (44 screens), `pos/Purchase/`, `pos/Reports/`, `pos/Till/`,
`pos/GST/`, `pos/Utilities/`, `pos/Settings/`. **3.0 is a reference only** —
it is in the table because it tells us how these forms were actually used by
these customers, and its file names are a shortcut to the real behaviour. It is
not the target and not a specification: where 3.0 did something badly or not at
all, that is not an argument either way. Tally and GoFrugal set the scope; 3.0
explains the habits.

## Legend

| mark | meaning |
|---|---|
| **✔** | built in the Qt6 client and working against the API |
| **▣** | fully designed in this share (spec + plan + mockup), no code |
| **◐** | schema exists, no design and no screen |
| **✖** | nothing — not in schema, not designed, not built |
| **—** | not applicable / deliberately not part of this product |

Verdicts: **MUST** = there is no honest way to call the product an accounting
system without it · **SHOULD** = expected by the customers we sell to, can
follow the musts · **PARK** = real feature, consciously deferred ·
**DECIDE** = blocked on a decision in §9, not on effort.

---

## 1. Where we stand today

**Built (Qt6 client):** Ledger Group master · Ledger master · Tax Rate master ·
Sale Bill (with the accounts posting spec) · Quotation · Sale Order · Opening
Stock · Physical Stock · item / customer / supplier / charge / tender masters.

**Designed, not coded (this share):** Opening Balance · bill-wise Receipt ·
Received Cheques · bill-wise Payment · Issued Cheques.

**So the accounting we have is:** a party can be opened with a balance, a sale
can be billed and posted, and money can be received or paid against those
bills including the full life of a cheque. That is the collection cycle of one
side of the business, and nothing else.

**One thing that is already true and easy to misread:** the designed Receipt
and Payment do write **multi-leg** vouchers — one party leg, one leg per
instrument, and one `av_role`-tagged leg per other-ledger line (TDS, discount,
bank charges, interest, round-off). What they refuse is a *free-form* voucher
and a *second party*. The distinction matters for §2.

**What that leaves unable to happen at all today:** buying anything, returning
anything, booking an expense, moving money between our own accounts, correcting
a mistake, closing a year, reconciling a bank, or reading a single financial
statement.

---

## 2. The voucher register — how Tally and GoFrugal actually do this

In GoFrugal's accounts module, **Sales, Purchase, Receipt, Payment, Debit Note,
Credit Note, Contra and Journal are one form**; the type is the only variation.
The decisive fact — and the one that is easy to get wrong — is that **the
register's "Sales" and "Purchase" are not goods documents.** They are the
accounting sale and purchase vouchers: party on one side, revenue or expense
ledgers on the other, GST as its own legs, a supplier/customer invoice number
and date, an HSN/SAC and rate per line, and **no item, no batch, no godown, no
stock movement**. That is why they have no GST-expense screen: rent, freight,
professional fees, AMC, a fixed asset with ITC, RCM, import of services are all
*Purchase* vouchers; scrap sale, rent received, commission income are *Sales*
vouchers.

### 2.1 The model

**One structure.** Header (type, number, date, narration, ref) + **n legs**
(ledger, Dr/Cr, amount) + **per-leg** bill-wise allocation. Nothing about
"receipt" or "journal" is in the structure.

**The type row carries the behaviour, not the code:**

| the type decides | example |
|---|---|
| numbering series, prefix, width, reset | `Jrl/0001`, reset yearly |
| which side the main ledger sits on | Receipt: money DR, party CR; Payment the reverse |
| which ledger *groups* are legal on each side | Contra: Cash/Bank both sides · Purchase(acct): DR side Purchase/Expense · Sales(acct): CR side Income |
| whether party legs **demand** bill-wise allocation | 3.0 read exactly this flag off the voucher type (`auto_adjust`, `financialvoucherentry.cpp:128`) |
| GST relevance and which return it feeds | Sales → GSTR-1 · Purchase → GSTR-2B / ITC |
| classification in day book, cash book, statements | "Cash voucher", "Bank voucher" |

So the screen is thin: a ledger grid with Dr/Cr, a running total that must
match, a bill-wise popup on party lines, narration. **One screen + one type
master = eight forms.**

**Two producers, one set of tables.** The goods documents (inventory sale bill,
inventory purchase) post into the *same* voucher tables when saved; register
vouchers are typed directly. The register lists both, which is why it looks
like everything lives in one form.

**Receipt/Payment are multi-ledger there.** A register receipt is not
party + bank; it is n legs — CR party (bill-wise), DR bank, DR discount
allowed, DR bank charges, CR interest received, DR TDS receivable — and it can
be **multi-party** (two customers' cheques in one deposit voucher), because
nothing in the structure says one party. The whole validation is: **ΣDr = ΣCr,
and every party leg's allocations sum to that leg.**

### 2.2 What we already have of it

Our tables are that model almost exactly, which was not planned but is lucky:

- `accounts.acc_voucher_header` — type, number, date, party, ref, doc ref/date
- `accounts.acc_vouchers` — `av_row_no`, `av_dr_cr`, `av_ledger_id`,
  `av_amount`, `av_signed_amount`, `av_remarks`, plus `av_role` (new, in the
  receipt plan) as our equivalent of Tally's duty-head classification
- `accounts.acc_bill_adjustment` — keyed on **`abj_voucher_line_id`**, so
  allocation already hangs off the **line**, not the voucher
- `accounts.acc_voucher_types` — already carries `vchr_category`,
  `vchr_nature`, `vchr_affects_accounts`, `vchr_affects_inventory`,
  `vchr_is_cash_voucher`, `vchr_is_bank_voucher`, numbering and Tally mapping

And our designed Receipt/Payment **already write multi-leg vouchers**: the
OTHER LEDGERS band is one `av_role`-tagged leg per line, and
`../receipt/receipt_flow.md` §4.5 explicitly permits "any other ledger", typed,
with a NULL role. What P15 in `../payment/` forbids is a **free-form** voucher
*on that screen* and **multi-party** — two different restrictions, and only the
second one conflicts with the register.

### 2.3 The five gaps if we adopt it

| # | gap | why it bites | decision |
|---|---|---|---|
| 1 | `avh_party_id` is singular on the header, and the receipt/payment specs say "one CR party per voucher" | a two-party register voucher becomes unrepresentable if endpoints and reports key off the header party. The column is nullable, so the fix is semantic, not structural | **S13** |
| 2 | `avh_total_debit` / `avh_total_credit` are **stamped, not derived**, so `ck_avh_balanced` cannot catch an unbalanced voucher | in a free-form register the balance check is the *only* guardrail. It has to be a trigger off the legs before the screen exists — already specified as `../receipt/plan-backend-receipt.md` §2.4 and asked as note (4) | — (known fix) |
| 3 | the type master has only coarse flags — nothing says "Contra = Cash/Bank both sides", "party legs demand bill-wise", "this type feeds GSTR-2" | without rule columns the one screen becomes a switch statement, and a switch statement is how you end up with four screens again | **S14** |
| 4 | there is **no GST band on a voucher** — a leg has nowhere to hold taxable value, rate, HSN/SAC, place of supply or ITC eligibility, and no `acc_voucher_doc_register` row is written | without it the accounting Purchase voucher cannot feed GSTR-2B matching or an ITC claim, which is the entire point of booking expenses there | **S15** |
| 5 | the register bypasses every operational guard — tender detail, `acc_pdc_register`, charges, promotions, the sales doc register | an accountant's tool needs a role gate, a refusal on inventory-affecting types, and an exception report of register vouchers that touch party bills. **Tender detail is the exception: §2.5 takes it off the list deliberately** — but instrument-controlled ledgers (Cheques in Hand, PDC, clearing) must still be refused, because those rows are owned by menus 51/52 | **S16** |

### 2.4 What this changes

**Make the generic payload canonical** — `{header, legs[], allocations[]}` —
and make every screen a preset over it:

- **ONE register screen**, type-driven, covering Journal, Contra, Debit Note,
  Credit Note, **accounting Sales**, **accounting Purchase** (so the separate
  GST-expense form dissolves), and a free-leg voucher mode for the accountant.
  Six MUST forms become one screen plus one type master.
- **The bill-wise Receipt and Payment stay as they are designed** — they carry
  tenders, the cheque register, credits-applied, printing and the
  identity strip, none of which belongs in a register. But their POST must
  serialize to the **same leg payload**, so the posting engine and the balance
  validator are written once and the register is the same endpoint behind a
  different UI (**S17**).
- **Keep P15 on the payment screen** (no free legs, no multi-party — it wrecks
  the identity strip and belongs to a batch run); allow multi-party **only** in
  the register.

### 2.5 Tenders on a voucher, and why day close does not use them

The register raises a question the operator screens never had to answer: a
free-leg Journal or Contra can move the cash ledger, and it writes **no tender
row**. `ck_td_src_doc_type` permits only `SALES_ORDER, SALE_BILL, SALE_RETURN,
RECEIPT, PAYMENT, OTHER`, so a register voucher cannot legally carry one today
even if we wanted it to. If a tender-wise day close were the control, that hole
would have to be closed with an invariant, a CHECK change and a two-way
validator on every screen.

**It is not the control. Day close is ledger-based (S19, decided).** Two
reasons, and neither is about effort:

1. **The ledger is complete by construction.** Every voucher leg hits a ledger;
   nothing moves money without one. A tender row can be absent, so a
   tender-based close has to be *defended*; a ledger-based close does not.
2. **A tender is a property of an instrument, not of an account.** "Cash",
   "Visa", "UPI" say how money arrived. Day close asks how much is *in*
   something, and only the second question is countable.

It also disposes of the opening float: the float is that ledger's opening
balance for the day, not a new concept.

**What ledger-only demands in exchange — master data, not schema.** One money
ledger per countable thing. If every counter posts to a single Cash-in-Hand,
the ledger can never say which cashier is short, so per-cashier accountability
means **a cash ledger per counter/drawer** ("Cash — Counter 1", "Main Cash /
Safe"), and cash to the safe becomes a Contra between two cash ledgers — which
is correct. Same on the other side: a clearing ledger per card network or
terminal if the terminal settlement slip must be matched, or one Card Clearing
if closing at that level is enough and settlement is left to bank
reconciliation. **This is the decision that stands in for the till session**
(the session id is being deferred deliberately); `acc_vouchers` also carries
`av_user_id` and `av_session_id`, so a per-user drill-down can be added later
without redesign. Open as **S20**.

**Read the figures off the vouchers, not the master.** Day close needs movement
*and* closing balance per money ledger for the branch and date — a date-bounded
sum of `acc_vouchers` (`av_ledger_id`, `av_signed_amount`, `av_voucher_date`)
inside the year's partition. **Not** `led_total_balance` / `led_total_dr` /
`led_total_cr`: those are running totals with no date and cannot answer "as at
close of 15-09".

**So tender rows keep exactly the job they already have** — instrument and
settlement facts: card auth code and last four, UPI VPA, MDR, `td_mdr_amt`,
`td_expected_settle_on` / `td_settled_on`, cheque number and date feeding
`acc_pdc_register` — plus a tender-wise collection *report*, which is analysis,
not a control. Receipt, Payment and the sale bill go on writing them because
they know the instrument; a register journal will not, and that now costs
nothing. **S18 is therefore decided as: tenders are optional instrument
metadata on money legs, not an invariant** — no `ck_td_src_doc_type` change is
needed for the register, and no two-way validator. If we ever do want a
register voucher to carry tenders, the additive shape is known:
`td_voucher_acc_year` + `td_voucher_line_id` (today `td_voucher_id` has no FK
and no year, and names the voucher rather than the leg, which is useless when a
voucher has three bank legs — `acc_bill_adjustment` already does this correctly
and should be mirrored), plus `'VOUCHER'` in the CHECK.

**The caveat to write into the day-close spec:** with a ledger-based close, the
cash-vs-card mix for the day is only as granular as the money ledgers are. When
someone asks why UPI is not shown separately, the answer is a ledger, not a
feature.

**And the teeth of day close are the lock, not the counting.** A close that can
be back-dated into is a report; a close that refuses posting is a control.

---

## 3. A — Accounting vouchers

| Form | Tally | GoFrugal | 3.0 (ref) | us | verdict |
|---|---|---|---|---|---|
| Receipt (bill-wise + on account) | ✔ | ✔ | `customerreceipt`, `financialvoucherentry` | **▣** | done — `../receipt/` |
| Payment (bill-wise + on account) | ✔ | ✔ | `supplierpayment`, `financialvoucherentry` | **▣** | done — `../payment/` |
| **Journal** — multi-leg Dr/Cr, bill-wise adjust per leg | ✔ | ✔ | `financialvoucherentry` (type 7) | **✖** | **MUST — register type (§2)** |
| **Contra** — cash↔bank, bank↔bank, deposit slip | ✔ | ✔ | `financialvoucherentry` (type 10) | **✖** | **MUST — register type** |
| **Debit Note** (financial, no items) | ✔ | ✔ | `financialvoucherentry` (type 8) | **✖** | **MUST — register type** |
| **Credit Note** (financial, no items) | ✔ | ✔ | `financialvoucherentry` (type 9) | **✖** | **MUST — register type** |
| **Purchase (accounting — no items)** — expense/asset + GST + ITC against a supplier bill | ✔ | ✔ | `gstexpenseentry`, `ledgernotesmainview` | **✖** | **MUST — register type; this IS the GST-expense form** |
| **Sales (accounting — no items)** — scrap, rent received, service/commission income + GST | ✔ | ✔ | `finsalesentry` | **✖** | **MUST — register type** |
| Free-leg voucher mode for Receipt / Payment (accountant) | ✔ | ✔ | `financialvoucherentry` (36 / 35) | ✖ | SHOULD — register type, same engine as the operator screens |
| Advance receipt against order | ✔ | ✔ | — | **✔** (`ARc`) | done |
| Post-dated voucher / PDC | ✔ | ✔ | `receivedcheques`, `issuedcheques` | **▣** | done |
| Optional voucher (entered, not counted until approved) | ✔ | ✔ | `approvaldashboard` | ◐ | **PARK (2026-09-15)** — a DRAFT already is this: it affects nothing until Post. The user removed the approval step as "double work" |
| Memorandum voucher (park with no effect) | ✔ | — | — | ◐ (`txn_hold`) | PARK |
| Reversing journal (provisional, self-reversing) | ✔ | — | — | ✖ | PARK |
| Payroll voucher / salary journal | ✔ | ✔ | — | ✖ | PARK |

**Six of the rows above are one screen, not six** — see §2. GoFrugal carries
them as one type-driven register, and 3.0 independently arrived at the same
shape (`Init/mainwindow.cpp:1017-1050` builds one `FinancialVoucherEntry` and
only sets `voucherType`). The row-per-form count in this table is the *form*
count, not the screen count: Journal, Contra, DN, CN, accounting Purchase and
accounting Sales close together or not at all.

**Why DN/CN is not optional:** the Receipt spec's CREDITS band reads
`abl_bill_type IN ('ADVANCE','SALES_RETURN','OPENING')` on the CR side and
`('SALES','OPENING','INTEREST','JOURNAL')` on the DR side
(`../receipt/receipt_flow.md` §4.4). Of those six producers we have three.
`JOURNAL` and `INTEREST` bills have no source at all, and `SALES_RETURN` has
only a table. The credits band is specified against rows nothing can create.

---

## 4. B — Trading documents the accounting depends on

We have **no purchase side**: the `purchase` schema holds `suppliers` and
`supplier_groups` and nothing else, and `src/modules/purchase` holds only those
two masters. Payables therefore cannot exist, which means the Payment screen
we just designed has nothing but opening balances to pay.

| Form | Tally | GoFrugal | 3.0 (ref) | us | verdict |
|---|---|---|---|---|---|
| Sale Bill / tax invoice | ✔ | ✔ | ✔ | **✔** | done |
| **Purchase Entry** | ✔ | ✔ | `purchaseentry`, `batchpurchaseentry` | **✖** | **MUST — blocks the whole payable side** |
| **Purchase Return / supplier debit note with items** | ✔ | ✔ | `purchasereturnentry` | **✖** | **MUST** |
| **Sale Return / credit note with items** | ✔ | ✔ | ✔ | **◐** (`02_sale_return.sql`) | **MUST** |
| Purchase Order | ✔ | ✔ | `purchaseorderentry`, `pomainview` | ✖ | SHOULD |
| GRN / Receipt Note (goods in before the bill) | ✔ | ✔ | — (bills directly) | ✖ | SHOULD |
| Delivery Note / challan (goods out before the bill) | ✔ | ✔ | `SalesDeliveryChallan` | ✖ | SHOULD |
| Supplier claim / damage / expiry return | — | ✔ | `expiryreturn`, `claimsummary` | ✖ | SHOULD |
| Supplier rate-difference / scheme credit | — | ✔ | `ledgernotesmainview` | ✖ | SHOULD |
| Stock transfer (godown / branch) | ✔ | ✔ | ✔ | **▣** | done (stock engine) |
| Physical stock | ✔ | ✔ | ✔ | **✔** | done |
| Rejections in / out | ✔ | partial | — | ✖ | PARK |
| Manufacturing journal / BOM | ✔ | ✔ | `BOMMainview` | ✖ | PARK |

---

## 5. C — Masters

| Master | Tally | GoFrugal | 3.0 (ref) | us | verdict |
|---|---|---|---|---|---|
| Account groups | ✔ | ✔ | `accgroupentry` | **✔** | done |
| Ledgers (with GST / party / TDS fields) | ✔ | ✔ | `accledgerentry` | **✔** | done |
| GST tax rate master | ✔ | ✔ | — | **✔** | done |
| **Voucher Type master** — series, prefix, width, reset, manual/auto, **plus the rule columns the register needs** (legal ledger groups per side, party required, bill-wise demanded, GST register fed) | ✔ | ✔ | — (hard-coded ids) | **◐** | **MUST** — `acc_voucher_types` is a table with no screen, only six types are registered, and its flags are too coarse to drive one register (§2.3 gap 3, **S14**) |
| **Bank master + cheque book ranges** | ✔ | ✔ | `bankmastermainview`, `bankentry` | ◐ / **▣** | **MUST** — bank accounts exist, the cheque book table is designed inside `../payment/`, neither has a screen |
| **TDS masters** — deductee type, nature of payment, rate slabs, no-PAN rate | ✔ | ✔ | — | **✖** | **IN (S2 decided 2026-09-15)** — `accounts.tds_rates` table, payment plan §2.9; a rate-master screen owed later |
| TCS masters + collection on sale | ✔ | ✔ | TCS reports only | ✖ | **IN (S2 decided)** — `tcs_basis = SALES` for the clients who need it; `abl_tcs_amount` on the bill |
| Cost categories / cost centres | ✔ | light | — | **✖** | DECIDE (S1 — now the only one of the two still open) — we already carry `acc_vouchers.av_cost_centre_id` and `led_is_cost_center_req` for a master that does not exist |
| Interest parameters (rate, from date, on overdue) | ✔ | ✔ | — | **◐** | SHOULD — `led_is_interest_applicable` / `led_interest_rate` exist, no engine, no screen |
| Salesman / agent commission scheme → commission payable | — | ✔ | `purchaseagententry`, `SalesAgentsMainview` | ✖ | SHOULD |
| Credit limit / overdue block rules | light | ✔ | in customer master | partial | SHOULD |
| Budgets & scenarios | ✔ | — | — | ✖ | PARK |
| Multi-currency + exchange rates, forex gain/loss | ✔ | — | — | ✖ | PARK (S5) |
| Invoice / voucher category for numbering by counter | — | ✔ | `invoicecategoryentry` | ✖ | PARK |

---

## 6. D — Period-end and utility forms

| Form | Tally | GoFrugal | 3.0 (ref) | us | verdict |
|---|---|---|---|---|---|
| Opening balances | ✔ | ✔ | `openingbalentry` | **▣** | done — `../openingbal/` |
| **Bank Reconciliation** | ✔ | ✔ | **— none** | **◐** | **MUST** — `av_recon_date` / `av_recon_refno` / `av_is_reconciled` are already on `acc_vouchers`; this is a screen-only gap |
| **Financial year: create, close books, carry forward balances** | ✔ | ✔ | `fiscalyeargeneration`, `fiscalyearselection` | **◐** | **MUST** — `public.fiscal_years` already has `fy_status`, `fy_lock_date`, `fy_is_carried_forward`, `fy_prev_fy_id`; carry-forward is what should *write* opening balances instead of a human typing them |
| **Period lock / freeze up to a date** | ✔ | ✔ | `lockfiscaldata` | ◐ | **MUST** |
| **Day close — per branch, per date, per money ledger** (+ cash denomination) | light | ✔ | `tillopeningsession`, `tillcashdenomination`, `tillsessionstatement`, `cashiermanagement` | ✖ | **MUST** for retail — see §2.5 and the description below |
| Cheque printing + cheque register | ✔ | ✔ | `tmpchequeprint`, `issuedcheques` | **▣** | done — `../payment/` |
| Cheque deposit / bounce | ✔ | ✔ | `chequedeposit`, `chequebounce` | **▣** | done — `../cheques/` |
| **Export to Tally / Tally sync** | — | ✔ | `TallyExport` (menu 175) | ✖ | SHOULD (S4) — every master already carries `tally_guid` / `tally_master_id` / `tally_alter_id`, so the intent is in the schema |
| Copy / duplicate a voucher | ✔ | ✔ | `copyvoucher`, `autocopysales` | ✖ | SHOULD |
| Recurring entry / standing instruction (rent, EMI, AMC) | ~ reversing | ✔ | — | ✖ | SHOULD |
| Bulk / multi-party receipt in one screen | — | ✔ | `linereceiptentry` | ✖ | SHOULD |
| **Collection entry by salesman + approval** | — | ✔ | `linecollectionentry`, `approvaldashboard` | ✖ | SHOULD — the hook is already in `../receipt/receipt_flow.md` §1 (`avh_src_doc_type='COLLECTION'` creates receipt DRAFTs) |
| Write-off / discount / bad-debt approval | via journal | ✔ | `approvaldashboard` | partial | SHOULD — the receipt carries the write-off column, nothing approves it |
| Party statement send (print / SMS / WhatsApp / email) | ✔ | ✔ | `customerstatement`, `sendwhatsapp`, `sendsms` | ✖ | SHOULD |
| Ledger merge / rename / delete with checks | ✔ | partial | — | ✖ | PARK |
| Audit log of voucher changes | ✔ | ✔ | `auditlog`, `auditkeylog` | **✔** | done (`18_audit_projections.sql`) |

### 6.1 Day close, as decided in §2.5

**Basis: the ledger, not the tender (S19).** Per branch and date, one row per
**money ledger** — every ledger whose group nature is Cash-in-Hand, Bank
Accounts or Bank OD A/c.

Two tables plus denomination:

| table | rows | columns that matter |
|---|---|---|
| `acc_day_close` | one per branch + date (later per session) | status, closed_by / closed_at, remarks, **the lock** that refuses posting into a closed day |
| `acc_day_close_ledger` | one per money ledger on that close | opening, movement, system closing, **counted**, variance, remark |
| denomination rows | under the cash ledgers only | note/coin denomination × count |

Figures come from a date-bounded sum of `acc_vouchers` inside the year's
partition, never from `led_total_*` (§2.5). Bank ledgers are not counted — they
are **reconciled**, which is the separate Bank Reconciliation form in this same
section; card and UPI clearing ledgers close at collection and settle later
through `td_settled_on` and bank recon.

Not in it: tender-wise variance (§2.5), the till session (deferred — S20 covers
what stands in for it), and cashier handover mid-day (follows the session).

---

## 7. E — Statutory forms

| Form | Tally | GoFrugal | 3.0 (ref) | us | verdict |
|---|---|---|---|---|---|
| **GSTR-1 (+ JSON)** | ✔ | ✔ | `gstr1` | ✖ | **MUST** |
| **GSTR-3B** | ✔ | ✔ | `gstr3b` | ✖ | **MUST** |
| **GSTR-2A/2B download + purchase reconciliation, ITC mismatch** | ✔ | ✔ | `gstr2b` | ✖ | **MUST** once purchase exists |
| **e-Invoice generate / cancel** | ✔ | ✔ | `geneinvoice`, `canceleinvoice`, `gsteinvoiceapi` | **◐** | **MUST** — the provider layer and `gde_` columns are done (`11_gst_provider.sql`), the client screens are not |
| **e-Way bill generate / cancel / extend** | ✔ | ✔ | `genewaybill`, `cancelewaybill` | **◐** | **MUST** — same |
| GST on advance receipt (11A) + adjustment | ✔ | ✔ | — | partial | in `../receipt/` as a deferred row |
| RCM (reverse charge) purchase handling | ✔ | ✔ | — | ✖ | SHOULD |
| HSN summary | ✔ | ✔ | `hsnwisesaledetail` | ✖ | SHOULD |
| TDS challan / 26Q return | ✔ | ✔ | — | ✖ | **SHOULD** — the projection table exists (receipt plan §2.14); the return screen is a later phase |
| TCS 27EQ return | ✔ | ✔ | TCS reports only | ✖ | **SHOULD** — same: `acc_tcs_register` exists, the screen is later |

**Where the purchase side of GST comes from:** every expense, asset and service
we buy without an inventory document reaches GSTR-2B and the ITC claim through
the **accounting Purchase voucher** (§2), not through a separate screen. That
makes the GST band on a voucher (**S15**) a statutory prerequisite, not a
convenience: taxable value, rate from `tax_rate_master`, HSN/SAC, place of
supply, ITC eligibility (input / capital / ineligible), supplier invoice number
and date, and an `acc_voucher_doc_register` row.

---

## 8. F — Reports

Not forms, but the reason forms are trusted: **today there is no screen in the
product that can show whether a single posting is correct.** Ledger statement
and Day Book are therefore prerequisites for accepting any posting engine, not
follow-up work.

**MUST:** Trial Balance · Balance Sheet · Profit & Loss · P&L without stock ·
**Ledger statement / ledger vouchers** · **Day Book** (summary + detail) ·
Cash Book · Bank Book · Outstanding receivables · Outstanding payables ·
**ageing** on both · Bills receivable / payable.
*For reference, 3.0 had all of these (`trialbalance`, `balancesheet`,
`profitlossacc`, `profitlossaccwostock`, `ledgerreport`,
`typewiseledgerreport`, `daybook`, `daybookdetail`, `cashbook`,
`customeroutstandings`, `supplieroutstandings`) — useful for what the users
expect to see on each, not as a layout to copy. We have none of them, and this
is the largest block of missing surface in the list.*

**SHOULD:** Account group summary · Ledger monthly summary · Customer /
supplier statement · Cash flow · exception reports (negative cash, negative
stock, vouchers with no legs, unallocated bills) · tender-wise collection ·
counter-wise receivables.

**PARK:** Funds flow · ratio analysis · cost-centre reports · budget variance ·
consolidated multi-company.

---

## 9. Decisions that must be taken before these are designed

These are schema-shaped: taking them late means changing tables that will
already have rows.

| # | decision | why it cannot wait |
|---|---|---|
| **S1** | **Cost centres — in or out?** | `acc_vouchers.av_cost_centre_id` and `acc_ledger_master.led_is_cost_center_req` exist with no master table, no allocation panel and no report. Either a master + allocation UI is scoped, or both columns come out before vouchers are written en masse |
| **S2** | ~~TDS and TCS — in or out?~~ → **TCS DECIDED IN, 2026-09-15** (a client needs bill-wise TCS outstanding and collection): `accounts.tcs_basis = 'SALES'` for those clients, so the **Sale Bill** charges TCS and stamps `acc_bill_balance.abl_tcs_amount` (`../receipt/plan-backend-receipt.md` §2.13); collection and outstanding come from the view `accounts.v_bill_tcs`, and the receipt raises no TCS leg on that basis. The RECEIPT basis stays the default elsewhere. **27EQ / 26Q** get a projection table each (§2.14). **TDS DECIDED IN too, 2026-09-15** — we deduct TDS when we pay, so `accounts.tds_rates` is a real effective-dated TABLE (`../payment/plan-backend-payment.md` §2.9), not a JSON setting: section × deductee × date → rate, `no_pan_rate` 20 for 206AA, single and annual thresholds checked (not stored), GiST EXCLUDE against overlapping periods. Payment seeds the `TDS_PAYABLE` line from it; 26Q comes out of `acc_tds_register`. **S2 is fully closed** |
| **S3** | **Financial statements — Qt screens or React/BI datasets?** | decides whether the next design days produce screens or stored-SQL datasets. `17_printing.sql` already gives us datasets, which would make Trial Balance / P&L / Balance Sheet nearly free but not interactive |
| **S4** | **Tally export — obligation or nice-to-have?** | if it is an obligation, group/ledger Tally mapping must be enforced in the masters *now* (the `tally_*` columns are already there and empty) |
| **S5** | Multi-currency — confirm OUT | everything (bill balance, tenders, adjustments) is single-currency today; saying it out loud stops it being assumed in |
| **S6** | **Who owns purchase?** | purchase entry is the biggest missing block and it is not an accounts screen. It needs its own schema + design track; until it starts, Payment is a screen with nothing to pay |
| **S7** | **GRN — separate document or bill-only?** | Tally and GoFrugal both separate goods-in from the bill. Deciding after purchase entry is built means rewriting it |
| **S8** | Day close — accounts or POS/Till? | it is a till/shift function that produces an accounting figure; wherever it lives, cash-in-hand reconciliation has to reach accounts. Narrowed by S19/S20: the *basis* is settled, the *home* is not |
| **S9** | Voucher Type master — screen now, or keep inserting by hand? | every voucher we design adds a row Prathap has to insert; a small screen ends that |
| **S10** | ~~Approval — one dashboard or per screen?~~ → **DECIDED 2026-09-15: neither, on vouchers** — DRAFT → POSTED → CANCELLED, no approve/reject, no `um_can_approve`. Field-collection approval (menu 188) is a different screen and unaffected; so is the write-off approver, which is a recorded name. Old text: | our designs each carry their own approve button and `canApprove`; GoFrugal centralises it, and collection, write-off and discount all need it |
| **S11** | Interest on overdue — engine or manual debit note? | `led_interest_rate` exists; `../receipt/receipt_flow.md` §12 defers it to "an interest debit-note screen". That screen is the Debit Note type on the register (§2), so the only question left is whether the interest is computed for the user or typed |
| **S12** | Opening balance vs carry-forward ownership | already open as DECISION 1 in `../openingbal/`; the financial-year form is the other half of it |
| **S13** | **Party cardinality on a voucher — one or many?** | GoFrugal's register allows two parties in one voucher; our `avh_party_id` is singular and the receipt/payment specs say "one CR party per voucher". Proposed: the header party is the *primary* party and NULL when there are several, `acc_bill_adjustment.abj_party_id` is the truth, and P15 keeps the payment screen single-payee — but every report and endpoint keyed on `avh_party_id` has to agree before rows exist |
| **S14** | **Rule columns on the voucher type** | the register is one screen only if the type says which ledger groups are legal on each side, whether a party is required, whether bill-wise is demanded, and which GST register it feeds. Added after the fact, they cannot be enforced on vouchers already posted (§2.3 gap 3) |
| **S15** | **Where the GST band on a voucher lives** | taxable value, rate, HSN/SAC, place of supply and ITC eligibility are per-line facts a leg has no columns for. Either `acc_vouchers` grows them, or a voucher-detail table carries them beside the legs. Decides whether accounting Purchase can feed GSTR-2B at all (§2.3 gap 4) |
| **S16** | **Register permissions and guards** | a free-leg register bypasses tenders, the cheque register, charges and promotions. Needs a role gate, a refusal on inventory-affecting types, and an exception report of register vouchers touching party bills — decided with the screen, not after it (§2.3 gap 5) |
| **S18** | ~~money leg ⟺ tender row~~ → **DECIDED (§2.5): tenders are optional instrument metadata on money legs, not an invariant** | the register needs no tender rows to be correct, so no `ck_td_src_doc_type` change and no two-way validator. The additive shape is recorded in §2.5 if we ever want register vouchers to carry them |
| **S19** | ~~day close: tender basis or ledger basis?~~ → **DECIDED (§2.5): ledger basis** | per branch + date, one row per money ledger, with a posting lock; tender-wise stays a report |
| **S20** | **OPEN — one cash ledger per branch, or per counter/drawer?** | this is what stands in for the deferred till session. Per-counter gives per-cashier variance and makes cash-to-safe a clean Contra between two cash ledgers; per-branch is simpler and closes only at branch level. Same question for card/UPI clearing ledgers: one each per network/terminal (matchable against the settlement slip) or one shared. Cheap now, expensive after rows exist |
| **S17** | **Is `{header, legs[], allocations[]}` the canonical posting payload for every screen?** | if yes, Receipt, Payment, the register and later the goods documents all POST the same shape and the posting engine plus the balance validator are written once. If no, we maintain a bespoke payload and validator per screen, and the register's rules will drift from the operator screens' |

---

## 10. Build order

Dependency-driven, not preference. Each wave unblocks the next.

**Wave 1 — make the books postable and verifiable**
1. **The voucher register** (§2) — **DESIGNED 2026-09-15, see this folder's
   `README.md`** — one type-driven screen: Journal, Contra, Debit Note, Credit
   Note, accounting Purchase, accounting Sales, plus a free-leg mode. Closes six forms, dissolves the separate GST-expense screen,
   gives the Receipt and Payment credit bands their missing producers, and
   answers "that is a journal" in three existing specs. Prerequisites inside
   it: the derived-totals trigger (note 4), S13–S17 (S18/S19 already decided
   in §2.5, which is what keeps tenders out of the register).
2. **Voucher Type master** with the rule columns (S9, S14) — the register is
   one screen only because this master exists; everything after it needs
   numbering anyway.
3. **Ledger statement + Day Book** — the first screens that can *check* a posting.
4. **Trial Balance → Balance Sheet → P&L** (S3) — the first screens that can
   check the whole ledger.

**Wave 2 — make the other half of the business exist**
5. **Purchase Entry**, then **Purchase Return**, then **Sale Return** (S6, S7)
   — the goods documents; expenses and services are already covered by the
   register's accounting Purchase type from wave 1.
6. **Outstanding + ageing**, both sides.

**Wave 3 — make the books closable and provable**
7. **Bank Reconciliation**.
8. **Financial year: close, carry forward, period lock** (S12).
9. **Day close / cash denomination** — ledger basis, with the posting lock
   (§6.1; S8 home, S20 ledger granularity).

**Wave 4 — statutory**
10. **e-Invoice / e-Way bill client screens** (the backend is already built).
11. **GSTR-1 / 3B**, then **2A/2B reconciliation** once purchase exists.

**Then:** collection entry + approval, recurring entries, bulk receipt,
commission, Tally export, statement send.

**Count:** **15 MUST forms** plus **12 MUST reports** remain — 19 forms before
the register collapse of rev 2, which turns six of them into one screen and one
master. The five designed screens are roughly a fifth of the accounting
surface, and the part of it that was easiest to specify because sales already
existed.

---

## 11. Out by decision

Reversing journals · memorandum vouchers · payroll · budgets & scenarios ·
multi-currency · cost-centre reporting (pending S1) · funds flow · ratio
analysis · ledger merge · rejections in/out · manufacturing journal ·
consolidated multi-company reporting.

Nothing in this file is a commitment to a date, a table or a layout.

**Wave 1 item 1 is now designed** (2026-09-15, same day): see `README.md` in
this folder — `voucher_register.md` (the whole design, one file) with
`voucher_register_ui_mockup_rev2.png`, and
notes (10). S13, S14, S15, S16, S17 and S21 were approved on the PNGs before
any of it was written; **S20 is the only decision still open**, and it does not
block the register.
