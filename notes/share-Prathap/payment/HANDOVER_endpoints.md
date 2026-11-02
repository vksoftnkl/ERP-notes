# Payment + Issued Cheques — the endpoint list to build, FINAL (2026-09-15)

**Give Prathap this file.** Nothing of the payment module has been built yet,
so unlike the receipt there is nothing to retract on his side — this is simply
the corrected list. `plan-backend-payment.md` §4 is the spec; this is the
checklist.

**The four house rules, already folded into the plan**

1. **One `partyId`.** A supplier id, a customer id and a ledger id are the same
   value — `sup_id` IS `led_id`, `cus_id` IS `led_id`. **Do not add
   `sup_ledger_id`** (§2.3 withdrew it). `payeeKind` stays, because it decides
   which bill types and which panels the screen shows — not because the id
   differs.
2. **No `/list` route on anything.** Payments, the issued register and cheque
   books are all **registered grids** (`configured-grid-sql` / `grid-details`),
   already specified in §2.8 with their columns and params.
3. **`companyId · branchId · accYear · voucherId`** on every route that acts on
   an existing voucher; for a cheque register row the same shape with `apdId`
   in place of `voucherId`.
4. **No approval.** DRAFT → POSTED → CANCELLED. No `/approve`, no `/reject`,
   no `um_can_approve`, no `payment_needs_approval`. Post runs from a DRAFT.
   (The **write-off approver** — `abj_approved_by`,
   `accounts.writeoff_approval_above` — is a recorded name and stays.)

---

## Payment — SIX routes

| # | verb | route | takes | spec |
|---|---|---|---|---|
| 1 | GET | `/api/v1/payments/open-items` | `payeeKind`, `partyId`, `companyId`, `onDate` | §4.1 — bills we owe, credits we hold, summary, party |
| 2 | GET | `/api/v1/payments/party-context` | `partyId`, `companyId` | §4.2 — last payments, our cheques pending |
| 3 | POST | `/api/v1/payments/create` | the DRAFT payload (upsert on `avhVoucherId`) | §4.3 — header + tenders + `avh_draft_lines`; touches no bill, no book, no number |
| 4 | POST | `/api/v1/payments/post` | the four keys + `allocations[]`, `creditsApplied[]`, `otherLineBills[]`, `onAccount` | §4.5 + §5 — one transaction; takes the book's next leaf `FOR UPDATE` |
| 5 | GET | `/api/v1/payments/get` | the four keys | §4.6 — adds `chequesIssued[]` and `beneficiary` |
| 6 | POST | `/api/v1/payments/cancel` | the four keys + `reason` | §4.6 / `payment_flow.md` §6 |

Optional seventh, same as the receipt: `PUT /payments/update-header` — the four
keys + the editable fields + `editRemark`.

## Cheque books — THREE routes

`GET /cheque-books/get` · `POST /cheque-books/create` (upsert) ·
`POST /cheque-books/close` — §4.9. **No `/cheque-books/list`**: the books
screen is a registered grid (§2.8), and the book picker on the payment screen
is a dropdown.

## Issued cheques (menu 52) — EIGHT routes

`GET /issued-cheques/get` · `/history` · `POST /issued-cheques/print` ·
`/presented` · `/returned` · `/stop` · `/void` · `/replace` — §4.10, all on
`apd_tra_type = 'P'` (a request naming an R row is a 409). **No
`/issued-cheques/list`** — registered grid (§2.8).

## Do NOT build

    /payments/approve     /payments/reject     /payments/list
    /cheque-books/list    /issued-cheques/list

## Three registered grids instead (§2.8, ids to issue)

payment list · issued register · cheque books. Column sets and params are in
§2.8 as they were.

## New 2026-09-15 — `accounts.tds_rates` is a TABLE

**We deduct TDS when we pay**, so §2.9 replaces the JSON setting with a real
effective-dated master: `tdr_section`, `tdr_deductee_type`, `tdr_rate`,
`tdr_no_pan_rate` (206AA, 20), `tdr_threshold_single`, `tdr_threshold_annual`,
`tdr_effective_from/_to`, optional `tdr_company_id` (NULL = all companies), and
a GiST `EXCLUDE` so two rows cannot cover the same section × deductee × day.
Seed rows for 194C / 194J / 194H / 194I / 194Q are listed in §2.9 — check them
against the current Finance Act before loading.

Lookup at the payment date: the ledger's `led_tds_nature_of_payment` gives the
section, `led_tds_deductee_type` the deductee (falling back to `ANY`), empty
`led_pan_no` forces `tdr_no_pan_rate`. Thresholds are **checked, not stored** —
the annual one against the sum of payments to that party this financial year.
`/payments/open-items` returns `tdsSection`, `tdsRate`, the two thresholds,
`tdsPaidThisYear` and **`tdsRateSource`** (MASTER / NO_PAN / BELOW_THRESHOLD
today; CERTIFICATE later) so the screen can show why a line is zero and why the
rate is what it is. The source is stored on the line and in `acc_tds_register`
— notes (17).

Reported from `acc_tds_register` (26Q) — `../receipt/plan-backend-receipt.md`
§2.14, written at post, never re-summed. A rate-master screen is owed later;
the table and the seed are enough for Payment.

## The §2 DDL, minus the withdrawn item

`Pmt` voucher type row · four roles + map rows (`ADVANCE_PAID`,
`DISCOUNT_RECEIVED`, `INTEREST_PAID`, `BALANCES_WRITTEN_BACK`) ·
`accounts.acc_cheque_book` (NEW) · `td_beneficiary_*` · `apd_printed_on` /
`apd_print_count` · the P11b `ck_apd_cleared` loosening · **`accounts.tds_rates` (§2.9, NEW table)** · two settings (`cheque_print_after_post`, `epayment_default_mode` — the `tds_rates` JSON setting is withdrawn).
**Withdrawn: §2.3 `sup_ledger_id`, and `payment_needs_approval` from §2.7.**

**And the prerequisite that is not ours:** the receipt's §2.4 — derive
`avh_total_debit` / `avh_total_credit` by trigger. Both modules need it, and
it is still not run (note 4).

## Order I would build them in

1. §2 DDL (after the receipt's §2, which this builds on)
2. `open-items` — pure SQL, and the screen is useless without it
3. `create`, then `post` (§5 step by step — the leaf lock is the tricky part)
4. `get`, `cancel`
5. the three grid registrations
6. cheque books, then the issued-cheque actions
7. `party-context` and `update-header` last
