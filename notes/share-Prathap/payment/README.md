# payment — Payment voucher (menu 100) + Issued Cheques (menu 52), revision 1

The mirror of `../receipt/` (rev 3) and `../cheques/`: money OUT to one
payee — a supplier bill-wise, a customer being refunded, or any other ledger
— and what happens to a cheque WE write after that. Assumes both earlier §2s
have been run; adds a cheque-book table, the beneficiary snapshot, TDS
deducted by us, and four roles.

**Building the endpoints? Start with `HANDOVER_endpoints.md`** — the corrected
route list (one `partyId`, no `/list` routes, the four keys, no approve/reject)
with the withdrawn items named, so `notes.txt` (9), (11) and (12) do not have
to be reconciled by hand.

**Read in this order:**

| file | what it is |
|---|---|
| `payment_ui_mockup.png` | **start here** — the screen: payee kind three-way, favouring name, last-payments / our-cheques-pending panels, BILLS WE OWE (pay / disc recd / r-off), CREDITS WE HOLD, INSTRUMENTS (NEFT with beneficiary snapshot, cheque taking the book's next leaf), OTHER LEDGERS (TDS payable, discount received, bank charges, interest, round-off), the identity strip, Approve / Post & Print (`.py` regenerates it) |
| `issued_cheques_ui_mockup.png` | menu 52 — filters, summary strip with the book's free leaves, the register (eight worked leaves in every state), Print / Presented / Returned unpaid / Stop-Void / Replace / Cheque books / History |
| `payment_tables.png` | which table, which column, in what order — eight cards, four lanes (pay a supplier · pay any ledger · refund a customer · our cheque's life), decisions P1–P15 + P11b |
| `payment_flow.md` | the spec — every column a payment touches with the worked example (Balu Traders: 1,25,000 settled by NEFT 74,775 + PDC cheque 45,000, TDS 125, discount 100, debit note 5,000) written through the tables; §5 leaves / post-dated / printing / the register's six actions; §10 decisions |
| `plan-backend-payment.md` | what Prathap builds — §2 DDL (`Pmt`, four roles, ~~`sup_ledger_id`~~ **withdrawn — `sup_id` IS `led_id`**, `acc_cheque_book`, `td_beneficiary_*`, `apd_printed_*`, the P11b CHECK, four settings, grids, datasets), endpoints for payments / cheque books / issued cheques, the posting transaction in order, 18 tests; one phase |
| `plan-qt-payment.md` | the Qt client plan for menu 100 — file set, `.ui` bands, delegates, endpoint calls, the identity on the client, keyboard, tests |
| `plan-qt-issued-cheques.md` | the Qt client plan for menu 52 |

**References read:** 3.0 `supplierpayment.cpp` (Incl Cust, auto-adjust, TDS,
discount received, last payments / pending cheques, ledger statement),
`issuedcheques.cpp`, `financialvoucherentry.ui` (NEFT / RTGS / IMPS with
beneficiary); TallyPrime Payment (cheque range + favouring, post-dated, cheque
printing per bank, payment advice, e-payments, auto TDS); GoFrugal payables.

**Corrected while writing this:** `../receipt/receipt_flow.md` §4.2 had a
third credit-pair leg that did not balance — fixed to one holding-ledger leg;
the payment mirrors that.

**Out, by decision (P15):** multi-ledger vouchers, batch / multi-party,
bank reconciliation, day-close, 163 / 104 / 215. One phase.
