# notes (62): `/payments` review against plan rev 2

2026-09-29. I read the `payment/` module and migration `20260929090000` against
`share/payment/plan-backend-payment-rev2.md`. This was a read of the code: the API was down, so
nothing here is a live test.

**Overall it matches the plan.** All routes are there, and the data is live (`Pmt` = 33, grid
123, menus 100 / 49, the three roles mapped). The legs balance in every path I traced. Your
README's declared deviations are all sound:
- ADVANCE_PAID stays unused;
- the bank charge sits inside `tdAmount`;
- post uses `assertVoucherBooksReconcile`;
- the credit pair has no holding leg.

The receipt defects your work exposed are in **notes (61)**.

What needs changing, most serious first:

## A · Must fix before the client is built

**A1 · TDS on an on-account payment (an advance) always fails.** Plan §5.2 says TDS covers the
on-account part, and PmtV already does it (`voucher-derive.ts:1379`).
- `seedTds` marks every TDS line `settlesBill = true` (`payment-lines.ts:711, 734`).
- `reserveDeductions` spreads that line only over bill capacity (`allocation-engine.ts:717`).
- With no bills, nothing is placed and the identity can never match, so it is always a 400.
- With a bill smaller than the tax, `takeFromBill` refuses with "room for X".
- **Scenario:** a 50,000 advance to a 194C supplier cannot be posted.
- **Fix:**
  - TDS settles the bills up to their capacity, and the rest is a non-settling deduction on the
    on-account part.
  - Add an OUT engine case and an e2e test for an advance with TDS.

**A2 · The routes never check that the document is a `Pmt`.** `loadHeaderOrThrow`
(`payment.service.ts:860-884`) checks only "not deleted, has a party", and the scope check
covers only company, branch and year.
- `/get`, `/post`, `/cancel`, `/delete`, `/amend` and `/adjacent` will act on an `Rct` or a
  register voucher.
- `/create` with an existing `avhVoucherId` (`:164-197`) is worse: `upsertDraftHeader`
  rewrites `avhVoucherTypeId` and the company (`:1009-1013`), turning a receipt draft into a
  payment.
- `/receipts` has the same hole the other way round.
- **Fix:** check `vchr_type_code` in both modules' loaders, and answer 404 on the wrong type.

**A3 · A reopened draft loses its cheque book.**
- `chequeBookId`, `favouring` and `acPayee` are stored only in `avh_draft_lines.cheques`
  (`payment-draft-lines.ts:58`).
- `PaymentTender` (`payment-api.types.ts:139`) has no `cheque`, and `chequesIssued` is empty on
  a draft. So neither `/get` nor the `/create` response returns them.
- `cheque.chequeBookId` is required (`save-payment.dto.ts:43`), so the draft cannot be re-saved
  without picking the book again.
- **Fix:** return `cheque {chequeBookId, bookNo, favouring, acPayee}` on each cheque tender row
  of a draft.

**A4 · The prompt-payment discount comes from `accounts.ppd_slabs`**, which is the discount WE
give customers (`payment.settings.ts:53`). It now seeds `ppdSuggested` on every supplier bill
(`payment-open-items.service.ts:175`). Plan §3 and rev 1 say the supplier's own terms:
`sup_cash_disc_perc` within `sup_credit_days`. Nothing in `payment/` reads them.

## B · TDS correctness (26Q)

**B1 · The TDS base includes other DR extras.** `payment-lines.ts:689-691` subtracts only the
MDR, not INTEREST_PAID or a free DR line.
- **Scenario:** NEFT 10,000 including 200 of interest gives tax 204.08 on a base of 10,204.08.
  It should be 200 on 10,000.
- Interest is 194A, not the party's section.
- The register row carries the inflated base.

**B2 · A keyed TDS line on a party whose master is not TDS-flagged writes no
`acc_tds_register` row.** `seedTds` returns null (`:675-677`), and the register is written only
`if (tds)` (posting `:291`). The CR TDS_PAYABLE posts, but the deduction never reaches 26Q. The
comment at `:418` says the register row records it. Either write the row, or refuse the line.

## C · Cheque leaves

**C1 · Amend soft-deletes the old leaves** (`payment-amend.service.ts:333-339`, `apdIsDeleted:
true`) instead of marking them CANCELLED as `/cancel` does. The leaf is never re-issued, because
the counter isn't rolled back. But:
- the book's leaf list filters `apd_is_deleted = false` (`cheque-books.service.ts:247`), so on
  Issued Cheques (menu 52) the leaf becomes an unexplained gap in the book;
- `ux_apd_issued_leaf` is `WHERE apd_is_deleted = false`, so the database no longer guards that
  leaf number.

A leaf is our own stationery and must stay visible. Please mark it CANCELLED, with reason
"amended into revision N". The receipt does the same, but a customer's cheque isn't our
stationery, so it can stay as it is.

## D · Contract, as it affects the Qt client (copied from the Receipt screen)

**D1 · Two renames the plan didn't ask for:** `summary.creditsHeld` → `debitsHeld` on
`/open-items`, and `summary.totalCredits` → `totalDebits` on `/party-context`. Please keep the
receipt's names, as rev 2 says ("DTO for DTO"). Or, if you'd rather keep yours, tell me and I'll
map them in the client.

**D2 · No rights check for menu 100.** Neither the controller nor the services call `loadRights`
(`common/posting/rights.ts`). The receipt has none for 99 either, but the Voucher Register and
the sales documents do. Please add it to both.

**D3 · `/open-items` accepts `mobile` and ignores it** (the DTO is shared with the receipt).
Either refuse it or drop it from the payment's query.

**Known and fine:**
- `roundoff` is lower-case, as the receipt's; the plan was wrong.
- The cheque object has `chequeBookId` and no `bankLedgerId`.
- There are no card-surcharge fields.
- `/post` `cheques[]` is the short form.

The client will follow all of these.

## E · Small

- **E1 · The write-off approver is required but not kept.** It is required on a
  BALANCES_WRITTEN_BACK line (`payment-lines.ts:518`), but `deductionRow` sets
  `approvedBy: null`, and the draft lines are nulled at post (`:1187`).
- **E2 · Round-off is CR only.** Rounding up (paying 5000 on 4999.60) becomes a 0.40 ADVANCE
  bill. Plan §5.6 said DR/CR. The receipt has the same limit, so this is your call.
- **E3 · The post re-reads money through `Number(...)`** (posting `:504-507`). It is harmless at
  numeric(·,2), but please pass the Decimal through; this is the float64 pattern from
  sale-order.service.
- **E4 · Stale comments.** `payment-lines.ts:77, 596` say the MDR is "on top of" the amount,
  but it is inside it. `payment.guards.ts:127` names migration 20260928200000; it should be
  20260929090000.

## F · Tests §7 still wants

1. The worked example in one post: NEFT + post-dated cheque + TDS + discount + debit note.
2. Cancel of a payment carrying an un-matured post-dated cheque. `payment-cancel.service.ts:198`
   still uses the plain `assertBooksReconcile`; see notes (61) §1.
3. Amend with a cheque: the old leaf stays visible and a new leaf is taken.
4. Amend with TDS: `tdsReversed` > 0.
5. Amend refused when a cheque has been presented, an advance has been spent, or a transfer is
   SETTLED.
6. Cancel refused on a RETURNED (bounced) cheque. Only CLEARED is tested.
7. An advance with TDS (A1).
8. The amend "setting ON" test returns early with a warning when the setting is off
   (`payment-amend.e2e-spec.ts:91-97`), so it can pass without testing anything. Set the setting
   in `beforeAll` instead.
