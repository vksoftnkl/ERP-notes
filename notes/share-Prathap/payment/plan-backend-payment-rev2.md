# Payment (menu 100) — backend plan, REVISION 2

**For Prathap. 2026-09-28.** This file supersedes `plan-backend-payment.md` (rev 1, 2026-09-14)
and `HANDOVER_endpoints.md` for the **Payment screen**. The Issued Cheques sections of rev 1
(§4.9, §4.10) are already built (notes 55, 58) and are not repeated.

**Why a revision.** Rev 1 mirrored the receipt's DESIGN (rev 3, 14 Sept). The receipt then
changed as it was built and tested:
- drafts remember allocations (notes 30);
- `/delete`, `/adjacent`, `/duplicate-check` and `/amend` (R20) were added;
- discount, write-off and round-off travel on the allocation, never as other-lines;
- any party ledger is accepted (R5), with no payee kind;
- the regularise sweep is company-scoped (R-B1).

And half of rev 1's §2 is already live. **Rule for this plan: `/payments/*` is
`src/modules/accountsModule/receipt/` mirrored**, route for route and DTO for DTO, with the money
going OUT. Where this file is silent, do what the receipt module does.

---

## 1. What is live today (checked 2026-09-28 against 192.168.0.106 and nexerp-api-ro)

| piece | state |
|---|---|
| `/payments/*` routes | **none**, and no module in the source |
| voucher type for this screen | **none**. `Rct` (12) is the receipt's; `PmtV` (31) is the Voucher Register's |
| `accounts.acc_cheque_book` (`acb_*`), `/cheque-books/*`, `/issued-cheques/*` | ✅ live (notes 55, 58) |
| `vouchers/cheque-book.helper.ts` `takeNextLeaf()` | ✅, row lock, used by PmtV |
| `accounts.tds_rates` + the lookup in `vouchers/voucher-facts.ts` | ✅ |
| `td_beneficiary_name / _account_no / _ifsc` | ✅ (no UPI column; see §6 D3) |
| `apd_printed_on / apd_print_count` | ✅ |
| roles `TDS_PAYABLE`, `INTEREST_PAID`, `BANK_CHARGES`, `ROUND_OFF`, `WRITE_OFF` | ✅ exist. TDS_PAYABLE and INTEREST_PAID are mapped |
| roles `ADVANCE_PAID`, `DISCOUNT_RECEIVED`, `BALANCES_WRITTEN_BACK` | ❌ missing |
| menus | 100 "Payment" hidden; 49 "Bill wise Payment" visible |

---

## 2. Data (one migration)

1. **Voucher type `Pmt`**, the mirror of `Rct` (12): nature PAYMENT, prefix `pmt`, AUTO numbering
   yearly, `vchr_in_register = false`, `vchr_menu_id = 100`. It is not a register type, so the
   Voucher Register and `/vouchers/*` never see it (and `PmtV` stays theirs).
2. **Roles + shared map rows:**
   - `ADVANCE_PAID` (Assets, "Advances to suppliers");
   - `DISCOUNT_RECEIVED` (Income);
   - `BALANCES_WRITTEN_BACK` (Income).

   Seed them like `26_ledger_map_seed.sql` did.
3. **Menus:** 100 visible; **49 hidden** (retired, because the bill-wise panel is always there, as on 99).
4. **Grid** "MAIN LIST - PAYMENTS": the receipt's grid 108 SQL with `Pmt`, same params and columns
   (payee in place of customer). Register it and tell me the id.
5. **Settings reused, not added:** `accounts.allow_posted_amend` (R20) and
   `accounts.receipt_bill_sort`. Add **`accounts.payment_salesman_mandatory`** only if you want
   the "paid by" rule; the client treats it like `receipt_salesman_mandatory`.
6. **Nothing else.** Rev 1's §2.4 (the book), §2.5 (beneficiary/print), §2.6 (P11b) and §2.9
   (tds_rates) are live already.

---

## 3. Module — `src/modules/accountsModule/payment/`, the receipt's files mirrored

| receipt file | payment file | what changes |
|---|---|---|
| `receipt.controller.ts` | `payment.controller.ts` | `@Controller('payments')`, same 11 routes (§4) |
| `receipt.service.ts` | `payment.service.ts` | draft upsert, get, update-header, delete: same rules, type `Pmt` |
| `receipt-posting.service.ts` | `payment-posting.service.ts` | the same ordered transaction, direction OUT (§5) |
| `receipt-cancel.service.ts`, `receipt-unwind.guards.ts` | `payment-cancel.service.ts` | + "a cheque past ISSUED (HELD)" and "a SETTLED NEFT" refusals |
| `receipt-amend.service.ts` | `payment-amend.service.ts` | R20 unchanged, behind the same setting |
| `open-items.service.ts` | `payment-open-items.service.ts` | CR bills we owe + DR items we hold (§4.1); adjacent / duplicate-check / party-context |
| `receipt-draft-lines.ts`, `receipt-lines.ts`, `receipt-ledger-roles.ts`, `receipt-cheque-links.ts`, `receipt.guards.ts`, `receipt.settings.ts`, `receipt-exception.filter.ts` | `payment-*` | same, with the payment's roles |
| `ppd-slab.ts` | **reused** | the same slab parser seeds DISCOUNT_RECEIVED from the supplier's terms |
| `allocation-engine.ts` | **reused, not copied** | add `direction: 'IN' \| 'OUT'` to `AllocationInput` (default IN, so the receipt is untouched). OUT flips `abj_dr_cr` to DR on CR bills and the party / holding legs. Its spec gets an OUT twin of every case |

**Shared helpers reused as they are:**
- `vouchers/cheque-book.helper.ts` (`listOpenChequeBooks`, `takeNextLeaf`, `formatLeaf`);
- `vouchers/voucher-facts.ts` (`loadLedgerFacts`, the tds_rates lookup, `isMoneyLedger`);
- `common/posting` (`VoucherPostingService`, `syncBillAdjustments`, `loadSetOffCredits`, `StatutoryService`).

**Do not build a second copy of any of these.**

---

## 4. Routes — `/api/v1/payments/*`, the receipt's list, **avh-prefixed keys like /receipts**

| # | route | receipt twin | payment difference |
|---|---|---|---|
| 1 | `GET /open-items?partyId&companyId&onDate` | same | bills = the party's **CR** bills pending (PURCHASE, OPENING CR, JOURNAL CR); credits = their **DR** items (ADVANCE paid, PURCHASE_RETURN / debit note, OPENING DR). `party` adds `tdsSection, tdsRate, tdsRateSource, tdsThresholdSingle, tdsThresholdAnnual, tdsPaidThisYear, panPresent` from voucher-facts, `bank {name, accountNo, ifsc}` (default `acc_ledger_bank_accounts` row), `favouringName`, `isMoneyLedger` |
| 2 | `GET /party-context?partyId&companyId` | same | lastPayments[10], **ourChequesOut[]** (`apd_tra_type 'P'`, HELD), summary totals |
| 2a | `GET /adjacent` | same | type `Pmt` |
| 2b | `GET /duplicate-check` | same | Σ tender amount paid to this party today |
| 3 | `POST /create` | `SaveDraftReceiptDto` | same shape. Tender row adds `cheque.chequeBookId`, `cheque.favouring`, `cheque.acPayee` and `beneficiary {name, accountNo, ifsc}`. **No `tdRefNo` on a cheque row** (the leaf is taken at post). Allocations and credits are REMEMBERED (notes 30) |
| 4 | `POST /post` | `PostReceiptDto` | same keys: `allocations[] {billId, billAccYear, amount, discount, writeoff, roundOff}`, `creditsApplied[]`, `otherLineBills[]`, `onAccount`. Answers `{data:{header,…}}` like /receipts/post, plus `cheques[] {tdRowNo, apdId, apdAccYear, leaf, bookNo}` |
| 5 | `GET /get` | same | + `chequesIssued[]` (leaf, book, printed, status), `beneficiary` per bank row |
| 6 | `PUT /update-header` | same | same whitelist + editRemark |
| 7 | `POST /cancel` | same | + refuse when a cheque is past HELD (presented / returned: unwind on menu 52) |
| 8 | `POST /delete` | same | DRAFT only |
| 9 | `POST /amend` | same | R20, same setting, same refusals |

There is no `/regularise-pdc` for payments: the one receipt sweep is company-wide over
`acc_bill_adjustment` and already regularises a post-dated payment row (check it handles
`abj_dr_cr = 'DR'`, and say so).

**Other-line roles accepted on a payment:** `TDS_PAYABLE` (CR, settles), `BANK_CHARGES` (DR,
seeded from `tdMdrAmt`), `INTEREST_PAID` (DR, an addition), `BALANCES_WRITTEN_BACK` (CR, settles;
over `accounts.writeoff_approval_above` needs `writeoffApprovedBy`), or a free `ledgerId`.
**Never** DISCOUNT_RECEIVED / WRITE_OFF / ROUND_OFF as other-lines: they ride on
`allocations[].discount / .writeoff / .roundOff`, exactly the receipt's
`linesThatTravel()` lesson. Sending both is the double-count the receipt already hit.

---

## 5. Post: the receipt's transaction, direction OUT

Same order as `receipt-posting.service.ts`. The payment-only steps:

1. **Party:** a money ledger (`isMoneyLedger`) is refused: "use Contra".
2. **TDS:** seeded server-side when the party is TDS-applicable, from voucher-facts. That's the
   same lookup PmtV uses, so the two can't disagree. It covers the on-account part too
   (rev 1 P5). A client figure that disagrees is a 409 naming both.
3. **Leaves:** `takeNextLeaf(bookId)` per cheque row, INSIDE the transaction. Write the leaf into
   `td_ref_no`, then one `acc_pdc_register` row with `apd_tra_type 'P'`, `apd_cheque_book_id`,
   `apd_favouring` and `apd_ac_payee`. That's the row Issued Cheques (52) already works on.
4. **Post-dated cheque:** its own `Pmt` voucher dated the cheque, `avh_against_voucher_id` = the
   payment. Same as the receipt's PDC voucher.
5. **On account:** one **ADVANCE (DR)** bill per voucher with a remainder, leg DR `ADVANCE_PAID`.
   A later purchase bill's adjust panel spends it (it's the receipt's pair, mirrored).
6. **Legs:**
   - DR party (the gross);
   - CR each instrument (the bank gross of its charge; BANK_CHARGES is the DR half);
   - CR TDS_PAYABLE;
   - CR DISCOUNT_RECEIVED = Σ allocation discounts;
   - DR/CR round-off and write-off from the allocation.

   The totals are derived by trigger.
7. `txn_status_log`: POSTED on the payment, ISSUED on each leaf.

---

## 6. Decisions (defaults given; tell me if you disagree)

| # | question | default |
|---|---|---|
| D1 | a payee KIND (supplier / customer refund / other ledger), as rev 1 had? | **No, as the receipt (R5).** Any party ledger. A customer refund is just a party whose credits you apply; an "other ledger" payment is a party with no bills (on account, or use the Voucher Register's PmtV) |
| D2 | new type `Pmt`, or reuse `PmtV`? | **`Pmt`**: the receipt has `Rct` apart from `RcpV`, and the register must not list screen-99/100 documents |
| D3 | UPI id on the beneficiary | not stored (no column); UPI goes in `tdPayerVpa` as on the receipt |
| D4 | print after post (payment advice, cheque) | **out of this phase**: there's no print runner yet (same as the receipt) |
| D5 | menu 49 | hidden, retired |

## 7. Tests: the receipt's suites, mirrored

The receipt module's spec list, OUT:
- one bill by NEFT;
- the worked example (NEFT + PDC cheque + TDS + discount + a debit note applied);
- TDS seeded, overridden, NO_PAN 20%, below threshold;
- discount by slab;
- credit applied;
- on account → ADVANCE bill, spent later;
- cash/bank payee refused;
- cheque:
  - finished book → 409;
  - two concurrent posts get different leaves;
  - a rolled-back post returns the leaf;
- cancel:
  - reverses the payment and the PDC vouchers;
  - refused once presented;
- delete draft;
- amend (setting on/off, baseRevision 409);
- adjacent;
- duplicate-check;
- update-header whitelist.
