# notes (55): the Payment Voucher issues cheques (register cheques #2)

2026-09-26. The user confirmed `register_payment_cheques_ui_mockup.png` (Q1–Q8). The full
contract is **`register_payment_cheques.md` §2**. It builds on the parts of
`../payment/plan-backend-payment.md` that were designed but never built. This is the build list,
in order.

## Schema

1. **`acc_cheque_book`**: payment plan §2.4 (P4), as written.
2. **The P9 columns:** `td_beneficiary_*` and `apd_printed_on`, `apd_print_count` (§2.5), **plus
   `apd_ac_payee boolean NOT NULL DEFAULT true`** (Q8).
3. **P11b:** loosen `ck_apd_cleared` (§2.6). Read the live definition first.
4. **`vchr_instruments = true` on PmtV.**

## Routes

5. **`GET /vouchers/cheque-books?companyId`**: the active, unfinished books
   `{chequeBookId, bankLedgerId, bankName, bookNo, leafFrom, leafTo, nextLeaf, left, format}`.
6. **`lines[].instrument` on a PmtV supplier line:**
   `{tenderId, bankLedgerId, chequeBookId, instrumentDate, refNo, favouring, acPayee}`.
   **No leaf is sent.** New codes: `VCH_BOOK_REQUIRED`, `VCH_BOOK_FINISHED`, `VCH_BOOK_BANK`,
   `VCH_BANK_REQUIRED`.
7. **`/validate`:** one generated CR leg per instrument on `bankLedgerId` (cash on the CASH tender's
   ledger), and `instrument.nextLeaf` shown, not promised. Post-dated and per-supplier TDS work as in 54 and 53.
8. **`/post`:**
   - `takeNextLeaf … FOR UPDATE` inside the transaction (§5.2 step 6);
   - the leaf into `td_ref_no`;
   - an `acc_pdc_register` 'P' row, HELD, with the leaf, bank, supplier, favouring and A/c payee;
   - `abj_cheque_id` on that line's allocations;
   - BANK / UPI: a tender row only.
9. **`/cancel`:** refused with `VCH_CHEQUE_MOVED` unless all HELD. HELD rows → CANCELLED, the leaves stay
   consumed, and the post-dated voucher(s) are reversed.
10. **`/get`:** each cheque's `leaf, chequeBookId, bookNo, favouring, acPayee, pdcStatus`.
11. **Cheque books, §4.9:** `/cheque-books/get`, `/create` (upsert), `/close`, plus a registered list grid.
12. **Issued cheques, §4.10:** `/issued-cheques/get`, `/history`, `/presented`, `/returned`, `/stop`,
    `/void`, `/replace`, plus the registered grid. Printing waits. The adaptations for register vouchers
    are in register_payment_cheques.md §2.6:
    - stop and returned reverse **one line** of a MANY voucher, through `ChqBnc`;
    - replace raises a new PmtV voucher;
    - the voucher's **type code** is on the get.

## Tests

13. register_payment_cheques.md §4, server side:
    - leaves in order;
    - a finished book;
    - concurrent posts;
    - present / return / stop;
    - cancel refused.

When it lands, I'll check it against the live Swagger and one real post. Then:
- the dialog;
- the cells and cards;
- menu 52 with its Cheque books button;
- `run.sh paycheques`.

Please note here any name you change.

---

## Checked 2026-09-26 (migration `20260926190000_payment_cheques`)

**Live and verified: items 1–10** (schema, `/vouchers/cheque-books`, the payment instrument,
validate / post / cancel / get). This was checked through the API and then through the Qt screen with
`tests/register/run.sh paycheques`, 15/15:
- the leaves were taken in order;
- ISSUED rows were written with A/c payee and favouring;
- the post-dated voucher was dated on its cheque date;
- cancel left the leaves used.

pmv00036 (API) and pmv00038 (screen) are both cancelled.

**NOT live: items 11–12** — `/cheque-books/*` and `/issued-cheques/*`. The code is in the
tree (`issuedCheques/issued-cheques.controller.ts`, registered in `app.module.ts`), but the running
server answers **404** on both. **Please rebuild and restart the API.** The menu 52 screen and the
cheque-book dialog are coded against those routes once they answer.

**One small ask:** the `/vouchers/get` instruments come back with `bankLedgerId: null`
(`bankName` is there). Please return the id, so a reopened voucher can name its bank without
a name lookup.

---

## Checked again 2026-09-26 (after the restart): items 11–12 are live

`/issued-cheques/*`, `/cheque-books/*`, grid 120 and grid 121 all answer. The menu 52 screen and
`ChequeBookDialog` are now coded against them. The no-write smoke run passed 14/0/1:
- the status → verb map;
- the summary strip, which matches the DB to the paisa;
- the book list.

The action run waits for VKPOS rights on menu 52.

**Two small asks (neither blocks anything):**
1. `/issued-cheques/get` returns `paymentVoucherId` but not its year. The client uses `voucherAccYear`,
   which is right unless a post-dated cheque crosses a year end. Please add `paymentVoucherAccYear`.
2. There is no way to void a **blank** leaf (spoilt before any voucher was written): `/void` takes an
   `apdId`, and a blank leaf has none. Something like `/cheque-books/void-leaf {companyId, chequeBookId, reason}`,
   taking the next leaf, would close that gap. It can wait for printing.

### Menu 52 live run, 2026-09-26: 25/0/0 — and one server ask that affects every grid screen

**3. The grid runner serves stale rows for one second after a write.**
- `configured-grid-sql.controller.ts` has `@CacheTTL(1)`. Its comment says "cache 60s", and
  cache-manager v7 reads the value as milliseconds.
- But the **global** `HttpCacheInterceptor` (app.module) reads the same metadata as **seconds**
  (`ttlSeconds * 1000`). So every `/configured-grid-sql/run` is cached by URL for 1 s.
- What happens: the screen presents a cheque, reloads the same page, and the old row (HELD) comes
  back. Found in the live run.
- The client now waits out the second before its post-action reload, on menu 52 and in the cheque-book
  dialog. Menu 51 and every other register have the same exposure.
- **Please exclude `/configured-grid-sql/*` from the global interceptor, or set the grid TTL to 0.**
  A register must never be served from a cache after a write.
