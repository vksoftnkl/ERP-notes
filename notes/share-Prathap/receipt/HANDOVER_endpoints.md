# Receipt — the endpoint list to build (2026-09-15, + two added 2026-09-17)

**Give Prathap this file.** It is the whole `/receipts/*` surface, so nothing
has to be reconciled across three notes. Where it says "see", the detail is in
`plan-backend-receipt.md` §4 — that file is the spec, this one is the
checklist.

**The three files that go with it**, and nothing else is needed to build:

| file | when to open it |
|---|---|
| `plan-backend-receipt.md` | the spec behind routes 1–7 — §2 schema prerequisites, §4 payloads, §5 the transactions |
| `plan-backend-delete-draft.md` | route 8, in full |
| `plan-backend-amend-posted.md` | route 9, in full |

(`receipt_flow.md` is the table-by-table reference underneath all of it;
`gap-vs-3.0.md` and `plan-qt-receipt.md` are client-side and not his.)

**What changed since notes (7), and why this sheet exists**

| # | change | note |
|---|---|---|
| 1 | **one `partyId`** — a customer id, a supplier id and a ledger id are the same value (`cus_id` IS `led_id`, `sup_id` IS `led_id`). No `cus_ledger_id`, no `sup_ledger_id`, no resolution step, no 404 for a missing bridge | (11) |
| 2 | **no `/list` route** — the list is a registered grid (`configured-grid-sql` / `grid-details`) | (11) |
| 3 | **`companyId · branchId · accYear · voucherId`** on every route that acts on an existing voucher | (11) |
| 4 | **no `/approve`, no `/reject`, no `um_can_approve`, no `receipt_needs_approval`** — DRAFT → POSTED → CANCELLED; Post runs from a DRAFT | (12) |

---

## Build these SIX routes (the original sheet, 2026-09-15)

| # | verb | route | takes | spec |
|---|---|---|---|---|
| 1 | GET | `/api/v1/receipts/open-items` | `partyId`, `companyId`, `onDate` | §4.1 — bills[] + credits[] + summary + party. No branch, no year, no paging cap |
| 2 | GET | `/api/v1/receipts/party-context` | `partyId`, `companyId` | §4.2 — `lastReceipts[]` (10) + `pendingCheques[]` |
| 3 | POST | `/api/v1/receipts/create` | the DRAFT payload (upsert on `avhVoucherId`) | §4.3 — header + tenders + `avh_draft_lines`; **touches no bill**, takes no number |
| 4 | POST | `/api/v1/receipts/post` | **the four keys** + `allocations[]` + `creditsApplied[]` + `otherLineBills[]` + `onAccount` | §4.4 + §5.2 — the one-way door, one transaction |
| 5 | GET | `/api/v1/receipts/get` | **the four keys** | §4.5 — header + tenders + legs + allocations + credits + cheques + PDC vouchers + the ADVANCE bill |
| 6 | POST | `/api/v1/receipts/cancel` | **the four keys** + `reason` | §4.8 + §5.3 |

Plus one optional seventh, only if header editing on a POSTED receipt is
wanted now rather than later:

| 7 | PUT | `/api/v1/receipts/update-header` | **the four keys** + the four editable fields + `editRemark` | §4.7 — nothing else accepted |

## And TWO more, added 2026-09-17

Both found by calling the live server, not by reading the plan. Each has its
own spec file in this folder — the row below is the checklist entry, the file
is the detail.

| # | verb | route | takes | spec |
|---|---|---|---|---|
| 8 | POST | `/api/v1/receipts/delete` | **the four keys**, no reason | `plan-backend-delete-draft.md` — DRAFT only; soft-deletes header + tenders + draft lines in one transaction |
| 9 | POST | `/api/v1/receipts/amend` | **the four keys** + the WHOLE create+post payload + `editRemark` + `baseRevision` | `plan-backend-amend-posted.md` — restate a POSTED receipt in place, behind `accounts.allow_posted_amend` (default OFF) |

**8 is not optional.** `/cancel` refuses a DRAFT — correctly, there is nothing
to reverse — and nothing was built to take its place, so an abandoned draft is
permanent. Seven of them on Acme Foods already, against 28 posted. Graded A in
`gap-vs-3.0.md`: a daily operation with no workaround at all.

**9 is a client request.** A one-person shop corrects a wrong cheque number by
cancelling and re-keying the whole receipt. Amend unwinds (§5.3's money work,
no reversal voucher, status stays POSTED) and re-applies (§5.2) against the
same header, same number, `avh_revision_no + 1`. **It refuses exactly what
`/cancel` refuses** — nothing may be amended past an instrument that has left
the drawer — plus `baseRevision`, the optimistic lock that stops two people
with the same receipt open from overwriting each other. Default OFF, so no
existing client changes. Decision R20.

**It is its own route on purpose, not a mode on `/create`** — §1a of the spec
says why, so the question does not get re-opened: `/create` must keep
answering 409 for a POSTED voucher, because it is the route every draft save
and every retry goes through.

**Order to build them in:** 8 first. It is an afternoon, it unblocks a daily
operation, and it needs no decision from anybody. 9 needs the three additions
in its §3 (one column, three tables into `audit.audit_log`, one setting) and
is worth reading in full before starting.

## Do NOT build

    POST /receipts/approve        POST /receipts/reject        GET /receipts/list

## Not an endpoint, but still owed

**A registered grid for the receipt list** — hidden keys (`avh_voucher_id`,
`avh_company_id`, `avh_branch_id`, `avh_acc_year`), then refno, date, party,
`avh_doc_amount`, `avh_adjust_amount`, on-account, instruments summary, PDC
count, status (DRAFT / POSTED / CANCELLED, filterable), created-by. PDC
vouchers do not list separately — they show under their receipt
(`avh_against_voucher_id`). Grid id to be issued.

## New since the sheet was first written (2026-09-15, same day)

**§2.13 — `acc_bill_balance.abl_tcs_amount`** (one column, `numeric(14,2)`,
default 0). Stamped by the **sale bill** when it raises the `abl_` row, from
the same figure it already puts in `gdr_tcs_value`. It is what makes
**bill-wise TCS outstanding** answerable for clients on
`accounts.tcs_basis = 'SALES'`. Plus the view `accounts.v_bill_tcs`
(collected / pending, pro-rata of the allocation) — no second column, no new
table for this.

`/receipts/open-items` therefore returns two more fields per bill:
**`tcsAmount`, `tcsPending`** (0 unless the basis is SALES).

On the SALES basis the receipt raises **no** TCS leg — the TCS is already in
the bill. The `TCS_PAYABLE` line stays exactly as specified for the RECEIPT
basis and the two never both apply.

**§2.14 — `acc_tcs_register` / `acc_tds_register`**, the 27EQ / 26Q projection:
section, rate, **`*_rate_source` (MASTER / CERTIFICATE / NO_PAN /
BELOW_THRESHOLD — add it now even though only three can occur yet, notes 17)**,
PAN, challan no/date/BSR, quarter, certificate no. Written at
post, never edited, never re-summed. On the SALES basis the register row is
raised **at the receipt** (206C(1H) liability arises on receipt), for the TCS
slice of that allocation.

## The §2 DDL still has to run first

`plan-backend-receipt.md` §2, minus the two withdrawn items. In particular
**§2.4 — `avh_total_debit` / `avh_total_credit` must become DERIVED by a
trigger on `acc_vouchers`**. They are stamped today, so `ck_avh_balanced`
cannot catch an unbalanced voucher; every accounts screen after this one
depends on that fix (it is note (4), still not run).

Withdrawn from §2: **2.10** `um_can_approve` · **2.11** `cus_ledger_id`.

## Order I would build them in

1. §2 DDL (the balance trigger first, then `Rct`, `av_role`, the roles, the settings)
2. `open-items` — the screen is useless without it and it is pure SQL
3. `create` — the draft, so the screen can be wired end to end
4. `post` — the transaction, §5.2 step by step
5. `get`, then `cancel`
6. the list grid registration
7. `party-context` and `update-header` last — both are conveniences
