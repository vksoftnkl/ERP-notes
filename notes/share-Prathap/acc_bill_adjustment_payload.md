# `acc_bill_adjustment` — what the client sends, and what the server derives

Companion to `grids/adjust_open_credits.sql`, which holds the two SQL halves
(the open-credits SELECT that fills the panel, and the INSERT that saving the
bill produces). This file is the column-by-column contract behind them.

Subject: the sale bill's **Adjustment** — one merged instrument covering an
order advance and a sale-return credit note, because both are the same event: a
credit the customer already holds, set off against this invoice.

> **Status (2026-08-15).** The client half is built and building clean. The
> adjustment is picked in a shared grid reachable two ways — as the ADJUST
> tender row's panel inside the settle dialog, and on **F4** from the bill
> screen, which is the only route a credit-term bill or a
> `salesTenderType = kNone/kCashBills` counter has. The server half is planned
> in `plan-nestjs-adjustments.md`.
>
> **An adjustment is never a tender.** The ADJUST row (type 7) is a read-only
> mirror of the panel: it is excluded from the tender total and produces **no**
> `acc_tender_detail` row. It reaches the server only through the `adjustments`
> array below. Anything that posted it as a tender as well would write the same
> money twice.

---

## 1. The short version

Two fields per adjusted credit carry the contract; the rest is an audit echo the
server may ignore.

```json
"adjustments": [
  { "againstBillId": "<abl_id>", "againstBillAccYear": "2025-2026",
    "amount": 5000.00, "remarks": "",
    "billType": "ADVANCE", "adjType": "ADVANCE_ADJUST", "settlementMode": "ADVANCE" }
]
```

`againstBillId` is an `accounts.acc_bill_balance.abl_id` taken straight from
the open-credits list. `amount` is what this bill takes off it. `remarks` is
optional.

`againstBillAccYear` is sent for the live DB's `abj_against_bill_acc_year`
(§7) — it is the credit's *own* year, and carries no meaning for the settlement.

`billType` / `adjType` / `settlementMode` are **echoed back from the list, and
must not be used to post.** They exist so a stored payload can be reconciled
against `acc_bill_adjustment` without a join. The server derives all three from
the credit's own row (§4), which is what stops a client that mislabels an
advance as a credit note from posting one.

Header context — company, branch, accYear, partyId, billDate, userId,
sessionId — rides on the bill payload this array is nested in. **Do not repeat
it per adjustment.**

Everything else on the table is HEADER, DERIVED, POSTED or AUTO. That is
deliberate: every column that could cause a mis-post is computed server-side
from the credit's own row, so a client that gets it wrong cannot write a bad
adjustment.

---

## 2. Column classification

| Column | Source | Note |
|---|---|---|
| `abj_id` | AUTO | `uuidv7()` |
| `abj_company_id` | HEADER | |
| `abj_branch_id` | HEADER | the branch that **SETTLED** it — the bill's branch, which may differ from the branch that raised the credit |
| `abj_tenant_id` | HEADER | |
| `abj_acc_year` | HEADER | FY of the **ADJUSTMENT** — the bill's year, never the credit's |
| `abj_bill_id` | **DERIVED** | §3 — the client cannot know the invoice's `abl_id` |
| `abj_against_bill_id` | **DERIVED** | §3 — same source, swapped |
| `abj_party_id` | HEADER | the customer's **ledger** id, not `cus_id` — see §6 |
| `abj_row_no` | LOOP | ordinal per bill; no unique index on it |
| `abj_adj_type` | **DERIVED** | §4, from the credit's `abl_bill_type` — echoed in the payload, still derived |
| `abj_adj_date` | HEADER | the bill date — the same for every row on the document |
| `abj_dr_cr` | **DERIVED** | §3, falls out of which side the row is on |
| `abj_amount` | **PAYLOAD** | the one number the client owns; `> 0`, `ck_abj_amount` refuses zero |
| `abj_settlement_mode` | **DERIVED** | §4 — echoed in the payload, still derived |
| `abj_settlement_ledger_id` | POSTED | §6 — open item for credit notes |
| `abj_tender_id` | POSTED | the ORIGINAL advance receipt on the order; server looks it up |
| `abj_tender_acc_year` | POSTED | partition key of `acc_tender_detail`, travels with the id |
| `abj_cheque_id` | POSTED | from the cheque row created at posting |
| `abj_cheque_acc_year` | POSTED | as above |
| `abj_is_post_dated` | POSTED | |
| `abj_voucher_id` | POSTED | |
| `abj_voucher_acc_year` | POSTED | |
| `abj_voucher_line_id` | POSTED | |
| `abj_reversal_of_id` | REVERSAL | never present on a create |
| `abj_reversal_reason` | REVERSAL | never present on a create |
| `abj_remarks` | PAYLOAD | optional |
| `abj_approved_by` | PAYLOAD | `WRITEOFF` only (`ck_abj_writeoff_approval`); not used by the settle screen |
| `abj_user_id` | HEADER | |
| `abj_session_id` | HEADER | |
| `abj_created_by` | HEADER | |
| `abj_is_deleted` | AUTO | |
| `abj_created_on` | AUTO | `now()` |
| `abj_sync_date` | POSTED | |
| `abj_modified_on` | — | update only |
| `abj_modified_by` | — | update only |

---

## 3. Why `abj_bill_id` is derived, not sent

The invoice's `acc_bill_balance` row **does not exist when the client saves** —
it is created server-side after sync. So the client can only name the credit,
and the server expands each payload entry into the pair:

```
row 1   bill_id = <the invoice, just created>   against = againstBillId   dr_cr = CR
row 2   bill_id = againstBillId                 against = <the invoice>   dr_cr = DR
```

Two rows because **each row moves exactly one balance** — the one named in
`abj_bill_id`. `fn_abj_refresh_balance()` recomputes `NEW.abj_bill_id` and
nothing else; it never touches the against-bill. `abj_against_bill_id` is a
cross-reference, not a second target.

`abj_dr_cr`: the invoice is a DR bill, so settling it is a CR movement; the
credit is a CR bill, so relieving it is a DR movement.

> **Confirm this direction against the ledger.** The schema constrains
> `abj_dr_cr` to DR/CR but never states which way a settlement runs, so the
> above is read off the balance-sheet direction rather than off a rule in 09.

`ck_abj_against` then holds automatically: both rows are `ADVANCE_ADJUST` /
`NOTE_ADJUST`, and both name an opposite bill.

---

## 4. Derivation table

Read from the credit's own `abl_bill_type`, never from the client:

| `abl_bill_type` | `abj_adj_type` | `abj_settlement_mode` | ledger relieved |
|---|---|---|---|
| `ADVANCE` | `ADVANCE_ADJUST` | `ADVANCE` | customer-advance liability |
| `SALES_RETURN` | `NOTE_ADJUST` | `CREDIT_NOTE` | credit-note liability |

Deriving rather than accepting these means a client that mislabels an advance
as a credit note simply cannot post one.

`OPENING` and `JOURNAL` credits are **not offered yet**. `ck_abj_against` only
lets `ADVANCE_ADJUST` / `NOTE_ADJUST` / `TRANSFER` name an opposite bill, and
which of those an opening credit should post as is an accounting decision, not
one to guess. Adding them later is one line in the SELECT's type filter.

---

## 5. What the server must validate

Re-read the credit rather than trusting the payload — the panel was fetched
seconds or minutes ago and another counter may have spent it since. Re-apply
the same predicates the open-credits SELECT used, plus the balance check:

```sql
AND cr.abl_is_deleted     = false
AND cr.abl_is_active      = true
AND cr.abl_dr_cr          = 'CR'
AND cr.abl_party_id       = :iparty_id
AND cr.abl_bill_type     IN ('ADVANCE', 'SALES_RETURN')
AND cr.abl_pending_amount >= a.amount
```

Take the row lock while doing it — `SELECT … FOR UPDATE` on the credits inside
the same transaction. `ck_abl_settled` is the final backstop, but reaching it
means the loser of a race dies on a constraint violation (a 500) instead of
being told cleanly that the credit is spent (a 409).

`ck_abl_settled` is the final backstop: `fn_abl_recompute()` writes **through**
it, so an over-allocation aborts the transaction that caused it. Two counters
racing for the same credit cannot both win.

Also required for an `ADVANCE_ADJUST`: the per-tender
`sales.sale_order_advance_alloc` ADJUSTED rows. The order side stays per-tender
because a refund or a bounce applies to one instrument, not to the booking as a
whole — which is the same reason `abj_tender_id` holds one tender per row. The
client does not model this split; an order's advance may have arrived as four
instruments and only the server can see them.

Do **not** update `abl_alloc_amount` afterwards. It is trigger-maintained and
recomputed (not incremented) from the adjustment rows.

---

## 5a. Re-saving an edited bill — absent is not empty

The client **omits the `adjustments` key entirely** when it could not read what
a bill was settled with: a bill loaded for edit whose GET returned no array but
whose header carries `sbAdvanceAmt > 0`. It will not rewrite a settlement it was
never shown.

So the two states mean different things, and the server must honour the
difference:

| payload | meaning |
|---|---|
| key absent | leave the existing adjustments alone |
| `"adjustments": []` | the operator cleared them — reverse everything |

Reversals are rows, never edits: same `abj_adj_type`, **negative** `abj_amount`,
`abj_reversal_of_id` set (`ck_abj_reversal_sign` enforces the sign both ways).
Nothing in this table is ever UPDATEd or DELETEd.

---

## 5b. What is NOT an adjustment row

Only credits the customer already holds come through the `adjustments` array.
The bill's tenders also write `acc_bill_adjustment`, but as `ALLOCATION` rows —
no against-bill, `abj_tender_id` set, one per tender. Two rules for whoever
writes bill posting:

- **A `CREDIT` tender must not produce an `ALLOCATION` row.** It is not money
  received; it is the declaration that the remainder stays receivable — which
  the `acc_bill_balance` row already represents. Allocating it would settle the
  debt with the debt and zero out a live outstanding.
- The **ADJUST** tender (type 7) never appears in `tenders[]` at all; the client
  drops it. Its money is this array.

Worked example — a 5,000 bill settled with 500 cash, 1,000 UPI, 1,500 credit and
a 2,000 advance adjusted:

| # | `abj_bill_id` | `abj_against_bill_id` | `abj_adj_type` | amount | from |
|---|---|---|---|---|---|
| 1 | invoice | — | `ALLOCATION` | 500 | cash tender |
| 2 | invoice | — | `ALLOCATION` | 1000 | UPI tender |
| 3 | invoice | advance | `ADVANCE_ADJUST` | 2000 | `adjustments[0]` |
| 4 | advance | invoice | `ADVANCE_ADJUST` | 2000 | `adjustments[0]` |

Invoice: alloc 3,500 → `abl_pending_amount` 1,500, PARTIAL — which is exactly
the header's `sbBalanceAmt`. Advance: alloc 2,000 → 0, CLOSED. The credit tender
produces nothing here, which is why the 1,500 survives as outstanding.

---

## 6. Open items

1. **BLOCKER — there is no customer → ledger bridge at all.**
   `abl_party_id` FKs to `accounts.acc_ledger_master.led_id`; the client sends
   `sales.customers.cus_id` as `partyId`. Checked against ERP-Server's Prisma
   models on 2026-08-15: `customer.prisma` has no ledger column and
   `accountLedgerMasters.prisma` has no party column — **nothing joins the two
   in either direction**, so "resolve it in the endpoint" currently has nothing
   to resolve with.
   Recommended: add `cus_ledger_id uuid` to `sales.customers`, FK to
   `acc_ledger_master(led_id)`, set when the control ledger is created. Same
   resolution then fills `tdPartyLedgerId`, which
   `SaleBillEntry::buildTenderPayload()` sends as null with a TODO for exactly
   this reason. Until it exists the endpoint cannot return a row.

2. **`abj_settlement_ledger_id` for a credit note.** An order advance names its
   ledger directly (`sale_order.so_advance_ledger_id`). There is no equivalent
   column for a credit note, so it must come from company settings. Until that
   exists it goes NULL rather than borrowing the advance ledger and quietly
   mixing two liabilities.

3. **The read endpoint is `GET /transactions/party-balance`**, not the
   `/bill-balance/open-list` this file first guessed. Confirmed with the client
   2026-08-15 and already wired:

   ```
   GET /api/v1/transactions/party-balance?partyId=&companyId=&type=CR|DR
   ```

   `type` is the ledger's own vocabulary (`abl_dr_cr`): **CR** = the party's open
   credits, advances and credit notes merged — the sale bill's list; **DR** =
   their open invoices, which a future sale return sets its credit against. No
   branch and no accYear param, deliberately (§ the SELECT's own comments).
   Row shape adds `adjType` / `settlementMode`, derived in the SELECT by the same
   CASE the INSERT uses, so the client never computes them. It is **not** a
   configured grid — no grid id — and it must **not** be response-cached:
   these are live balances another counter can spend mid-sale.

   Not implemented yet. Until it lands the panel degrades to an empty grid with a
   hint line and never a modal, so the client ships ahead of it.

4. **`sbAdvanceAmt` vs `adjustments` — exactly one must post.** Both describe
   the ADVANCE settlement. If bill posting turns the header field *and* the
   array into `acc_bill_adjustment` rows, the advance posts twice.
   Recommended contract: **when `adjustments` is present it is authoritative and
   `sbAdvanceAmt` is a header cache only.** The client keeps
   `sbAdvanceAmt == Σ(adjustments where billType = 'ADVANCE')` whenever the
   panel is live, so the two cannot disagree — but the server must still choose
   one. Note the header field stays *advance-only* on purpose: a credit note
   relieves a different liability and lands in `sbPaidAmt` instead.

5. **`ix_abl_open` does not cover this query.** Its INCLUDE list is
   `(abl_doc_refno, abl_bill_amount, abl_pending_amount, abl_dr_cr)`; the
   panel also needs `abl_bill_type`, `abl_status`, `abl_acc_year` and the four
   `src_doc` columns, so it is a heap fetch per row. A handful of rows per
   customer, so this is a note and not a problem — but the index comment's
   "the heap is never touched" no longer holds for this caller.

---

## 7. Schema drift — needs a decision

The live database (pgAdmin script output) carries two columns that are **not**
in `schema/09_acc_bill_balance.sql`:

```sql
abj_bill_acc_year          character(9) NOT NULL,
abj_against_bill_acc_year  character(9),
```

They are redundant by design. `acc_bill_balance` is `PRIMARY KEY (abl_id)` —
**not partitioned** — and the file says why at `09:315-317`:

> *No acc_year on the bill FK: acc_bill_balance is not partitioned, which is
> exactly what makes cross-year settlement a plain foreign key.*

That is the whole trick that lets a March invoice be settled by an April
receipt without either row moving. Note the asymmetry it produces, which is
correct and worth preserving: `abj_tender_id` and `abj_cheque_id` **do** carry
an `_acc_year` partner, because `acc_tender_detail` and `acc_voucher_cheques`
are partitioned and their composite PKs force it. `abj_bill_id` does not,
because its target is not.

`abj_bill_acc_year` is also `NOT NULL` with no default, so every insert must
now supply a value that carries no information.

**Either drop both columns or update the schema file — do not leave the two
disagreeing.** If they were added deliberately as a reporting shortcut, say so
in the file, because the next person to read it will otherwise take the comment
above as current.

---

## 8. Cross-references

- `grids/adjust_open_credits.sql` — the SELECT and the INSERT
- `schema/09_acc_bill_balance.sql` — both tables, `fn_abl_recompute`,
  `fn_abj_refresh_balance`, and the worked example at `:633-647`
- `schema/06_sale_order.sql` — `sale_order_advance_alloc`, the per-tender side
- `schema/04_tenders.sql` — tender type 7, to be renamed `RRN` → `ADJUST`. The
  client answers to **both** names (`isAdjustTender()`), so the rename can land
  on either side first. Its `needsRef` must be **false**: the row is a read-only
  summary nobody can type into, and `true` made the settle screen refuse every
  bill that adjusted anything.
- `plan-nestjs-adjustments.md` — the server-side plan: Prisma models for
  `acc_bill_balance` / `acc_bill_adjustment` (they exist in the DB but not in
  Prisma), the `party-balance` endpoint, and `BillAdjustmentService`.
- Client, all of it new under `src/shared/adjustments/`:
  `party_adjust_entity.h` (columns + the `BillAdjustment` payload type),
  `party_adjust_grid_controller.cpp` (the fetch, validation and `collectJson()`),
  `party_adjust_panel.cpp` (the shared widget). Hosts:
  `sale_bill_entry.cpp` (`buildAdjustmentPayload()`, the F4 route, the header
  roll-ups) and `sale_bill_tender.cpp` (the ADJUST row).
