---
name: six-txn-controls
description: The cross-cutting controls for all six sales transactions — barcode, scan-to-load, one lifecycle, GST-cancel-first, decimal safety, and the void + till session anti-theft design
metadata:
  node_type: memory
  type: project
---

Raised by the user 2026-09-22 while Prathap was building A3. Spec written to
`share/sales/SIX_TXN_CONTROLS.md` (363 lines). The six: quotation 14, order 11,
DC 182, DC return 251, bill 12, sale return 13.

**The premise, the user's words:** *"its completely money oriented — once a
wrong entry is done we can't get our money back."*

## Already true — checked on the live box, do not rebuild

* **Decimal is already safe in the DB: ZERO float/real/double columns** in
  sales, stock, accounts, inventory. Money `numeric(15,2)`, qty + rate
  `numeric(18,6)`, stock value `numeric(18,2)`.
* **`session_id` is on all six headers** + `acc_tender_detail`,
  `acc_voucher_header`, `acc_bill_adjustment`, `acc_temp_credit` — and it is a
  **free-floating uuid with NO FK**. `audit.user_login_sessions` is a LOGIN
  session (token/IP/agent), not a till session.
* **`*_print_count` and `*_device_id` on all six.**
* **`public.print_log` is BUILT** — plg_status, plg_error, plg_printed_on/_by,
  plg_copy_no, per print attempt.
* Tender void exists (35): `td_is_voided/_reason/_voided_on/_by/_replaces_id`.
* Line-wise godown already designed on both returns (header seeds, LINE wins).
* **No void columns on any of the six** (`void=0` on all six headers).

## THE DECIMAL LEAK — the urgent one

`src/modules/sales/sale-order/sale-order.service.ts` drops Decimal to float64:
`asNumber()` → `Number(v)`, `roundAmount()` = `Math.round(v*100)/100`, and
**line 186 `const AMOUNT_EPSILON = 0.005`**. An epsilon on money means the
arithmetic already drifts; `Math.round(1.005*100)/100` = 1.00 not 1.01.
**79 files use `Prisma.Decimal` correctly — the order service is the exception,
and it is the one A3's bill will be copied from.** Put in the handover §3a.
Rule: Decimal end to end, no epsilon, round once at the column with
ROUND_HALF_UP, **distribute by LARGEST-REMAINDER** (round down, hand the
leftover paise to the largest remainders) so parts sum to the whole exactly.

## The anti-theft model — why void ≠ cancel

User's scenario: *"user can bill but won't take print — he will say printer not
worked, so the customer unknowingly gives the amount to the user, he clears the
bill and takes the money."*

Cancel is an ACCOUNTING correction. It cannot answer "was there money on the
counter that is now not in the drawer?"

**`print_log` is the strongest control and it already exists**: no row at all =
no print was ever attempted (the claim is false); a failed row = the printer
really did fail; a SUCCESSFUL row then a cancel = the theft pattern. One query.
**But `print_log` is missing from `ensure_acc_year_partitions`** — tolerated as
"not an issue" (bites on the 2028-29 roll); if it becomes an anti-theft control
that gap must close first.

**Void design:** number KEPT and visible (a missing number is the loudest
alarm), mandatory reason from a controlled list, stamped with session + device +
minutes-after-post + **whether a successful print_log row existed at void time**
(computed then, stored, never recomputed). Manager approval when printed
before void / over VOID_APPROVAL_AMOUNT / over VOID_SESSION_LIMIT / later than
VOID_MINUTES. **Void ITEM** is the quieter vector (void 3 lines after the
customer paid) — log every line removed or reduced after it was added, drafts
included.

**`till_session` is the missing table everything hangs on:** open (operator,
device, counted opening float), close (counted by tender, expected vs counted,
the DIFFERENCE, manager sign-off over a threshold). Build the SESSION FIRST,
then void — a void without a session is an event with no owner.

## Other asks in the same message

* **Barcode on all six** — scan targets the item grid, **increment not append**
  for the same item+lot+godown+bucket; weight barcode (Q48) parser shared; on
  the two RETURN screens a scan may only touch an existing line.
* **Scan ref no → LOAD the txn** on all six (Code-128 on every print; the
  printing layer is data-driven so it is a template change). Cross-document:
  a bill ref loads as the sale return's source, a challan ref as the DC
  return's; each screen declares what it accepts.
* **One lifecycle, all six:** draft → post → amend(rev+1) → cancelled, cancel-
  draft off DRAFT. Number issued at POST, **never reused**; cancelled documents
  stay visible. DC return and a declared credit note have **no amend at all**.
* **GST FIRST, then the document (§5a):** an IRN or live EWB must be cancelled
  BEFORE the txn can be. Past the IRN window it cannot be cancelled at all —
  the correction is then a credit note, and the screen must say so rather than
  offer a dead button. Windows from `fn_statutory_limit`, per company.
* **Sale return settle opens itself**: ADJUST → open CR bills seeded original-
  bill-first; CASH → tender rows. Choosing the mode IS the gesture.

## VOID + SESSION ARE DEFERRED (user, 2026-09-22 evening)

*"no need void bills problem and session issue we can handle later."*
**Parked, NOT cancelled** — kept in full in `SIX_TXN_CONTROLS.md` §7 and
`HANDOVER_endpoints.md` §12 because the reasoning is the expensive part. The
three open questions all belonged to §12, so they are closed and **nothing is
blocked**. When it returns: **session table first, then void.**
`print_log` gives partial cover meanwhile (no row = no print was attempted).
**Do not draw the void/session mockup yet.**

**LIVE WORK is §11**, which needs no answers: `/sales/resolve-scan`,
`/sales/resolve-item-scan`, and the **GST-cancel-first gate** — do the gate
first, it is a correctness gate on a shipped route and the only one that can
lose money.
