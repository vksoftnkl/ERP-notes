# The six sales transactions — common controls, money safety, and the till session

**2026-09-22.** Raised by the user while A3 was being built. Every claim about
the current state below was **checked against the live box this morning**, not
assumed — where something already exists it says so, so nothing is built twice.

The six: **quotation (14) · sale order (11) · delivery challan (182) ·
DC return (251) · sale bill (12) · sale return (13).**

The premise, in the user's words: *"its completely money oriented — once a
wrong entry is done we can't get our money back."* Everything here follows from
that, and the last section is the one that matters most.

---

## 0 · What ALREADY exists — do not rebuild these

| thing | state | where |
|---|---|---|
| **Decimal everywhere in the DB** | ✅ **zero** `float` / `real` / `double precision` columns in `sales`, `stock`, `accounts`, `inventory` | money `numeric(15,2)`, qty + rate `numeric(18,6)`, stock value `numeric(18,2)` |
| **`session_id` on all six headers** | ✅ present on every one | `sb_session_id`, `sq_session_id`, `so_session_id`, `sdc_session_id`, `sdr_session_id`, `sr_session_id` |
| **device on all six** | ✅ | `*_device_id` (+ `sb_device_type`) |
| **print count on all six** | ✅ | `*_print_count` |
| **a full print audit trail** | ✅ **already built** | `public.print_log` — see §7, it is the single most important existing control |
| **tender void** | ✅ built by `35` | `td_is_voided`, `td_void_reason`, `td_voided_on`, `td_voided_by`, `td_replaces_id` |
| **line-wise godown, sale return** | ✅ designed | `sri_godown_id` — header godown is only a DEFAULT that seeds lines |
| **line-wise godown, DC return** | ✅ designed | line `godown` column, same rule |

**Two gaps that matter and are NOT built:**

* **There is no till session table.** `session_id` is threaded through all six
  headers, `acc_tender_detail`, `acc_voucher_header`, `acc_bill_adjustment` and
  `acc_temp_credit` — and it is a **free-floating uuid with no foreign key to
  anything**. `audit.user_login_sessions` is a LOGIN session (token, IP, user
  agent); it is not a till session and has no float, no count, no close.
* **No void columns on any of the six.** `void=0` on all six headers.

---

## 1 · Line-wise godown — already right, stated so it stays right

The user asked for it on the sale return and the DC return. It is already the
design on both, and the rule is the same on each:

> The **header** godown is a convenience that seeds new lines.
> The **line** godown is authoritative.

Because one return can split across godowns: five good ones back to Main, two
damaged into the Damaged bay. If the header were authoritative that return
would have to be keyed as two documents, and the customer's single credit note
would become two.

**This extends to all six.** The bill and the challan already issue per line;
the quotation and the order carry an intended godown per line that the bill
may override. Nothing here needs a schema change — only that no screen is
allowed to grey the line column "because the header has one".

---

## 2 · Barcode on all six

One scanner behaviour, one widget, six screens. The scan target is the **item
grid**, and the field never has to be focused first — the screen listens.

| scan resolves to | then |
|---|---|
| item barcode | add a line, qty 1, or **increment** the existing line for the same item + lot + godown + bucket |
| **weight barcode** (Q48) | already specified for the bill — embedded qty/price; the SAME parser serves all six |
| lot / batch barcode | set the line's lot — on the returns this is how the goods go back to the lot they left on, without typing |
| unknown code | a single beep and a red hint. **Never** silently create an item |

**Increment, not append**, is the rule that matters at a counter: scanning the
same tin five times must give one line of 5, not five lines of 1 — otherwise
the customer's printed bill is unreadable and a voided line is invisible in the
noise. On the two RETURN screens a scan may only touch a line that already
exists (the ceiling is the source document); an unknown scan there is an error,
not a new line.

---

## 3 · Scan a ref no and load the transaction — all six

A second, separate scan mode: **scan the printed document's ref no and the
screen loads it.** The operator points the gun at the customer's slip instead
of typing `bil00042`.

* It is a **load**, never a create. It opens the document read-only exactly as
  F8 → pick would.
* Every one of the six prints its ref no as a barcode (Code-128) — the printing
  layer (`17_printing.sql`) is data-driven, so this is a template change, not
  code.
* **Cross-document scanning is the real win**, and each screen declares what it
  accepts:

| screen | scanning a bill ref | scanning a challan ref |
|---|---|---|
| sale return (13) | loads it as the **source bill** | rejected — billed goods only |
| DC return (251) | rejected — that is a sale return | loads it as the **source challan** |
| sale bill (12) | loads the bill | imports its **open lines** |

* A scan that resolves to a document in **another branch or another year**
  must say so and offer to switch, never silently load nothing.

---

## 4 · The settlement dialog opens itself — sale return

Confirmed as asked, and it is how `sale_return_settle_ui_mockup.png` is drawn:

* **ADJUST** → the dialog opens on the **open CR bills** grid, seeded
  **original bill first**, then the party's other open bills by due date. The
  operator re-spreads; they may never exceed a bill's pending.
* **CASH** → the dialog opens on the **tender rows** (the bill's tender widget
  in CR mode) with the full amount on the default cash tender.
* **ADVANCE** → nothing to fill; a confirmation line only.

Choosing the mode **is** the gesture that opens it. There is no second click,
because a mode chosen and never settled is the shape that leaves money
unaccounted for.

---

## 5 · One lifecycle, all six, no exceptions

The user: *"draft, cancel draft, post, amend posted, cancel posted — without
any miscue."* One state machine, and every screen shows the same verbs in the
same places:

```
            ┌──────────── cancel draft ────────────┐
            ↓                                      │
   (new) ─ DRAFT ─ post ─→ POSTED ─ amend ─→ POSTED(rev+1)
                              │
                              └─ cancel posted ─→ CANCELLED
```

**The rules that make it "no miscue":**

1. **A number is issued at POST and is never reused.** A cancelled document
   keeps its number and stays visible. A gap in the series is itself a signal.
2. **Cancel a DRAFT ≠ cancel a POSTED.** A draft has moved nothing, so it just
   stops existing in play (and it must still be listable with its reason —
   that is not the same as delete). A posted cancel writes **reversal rows**:
   stock, legs, register, caches. Never an UPDATE, never a DELETE.
3. **Amend is the whole payload again**, with `baseRevision` + an `editRemark`,
   revision + 1 — the receipt's R20 pattern, already settled.
4. **Every transition writes `txn_status_log`** — who, when, why. The header
   carries only the CURRENT state; that is why the posted-on/cancelled-on
   columns were dropped.
5. **Two documents have NO amend at all, by design:** the **DC return** (cancel
   + re-enter) and a **declared credit note** on the sale return. Those are not
   missing features and no right unlocks them.

### 5a · GST first, then the document — the hard gate

> **If an IRN or an e-way bill has been generated, the GST document must be
> cancelled BEFORE the transaction can be cancelled.** The screen refuses, and
> says which one is blocking.

This is not a preference; it is the law's order of operations. An IRN cancelled
after its invoice is gone cannot be matched to anything, and the 24-hour IRN
cancel window closes whether or not we got to it.

| blocking | the screen must |
|---|---|
| **IRN live** | grey Cancel, show "Cancel IRN first", offer the button. Past the statutory window the IRN **cannot** be cancelled at all — then the correction is a credit note, not a cancel, and the screen must say that instead of offering a dead button |
| **EWB live** | grey Cancel until the EWB is cancelled, or it has expired past `EWAY_CANCEL_HOURS` |
| **both** | IRN first, then EWB, then the document |

Both windows come from `fn_statutory_limit` per company — never a constant.

---

## 6 · Decimal — the schema is safe, the code is not

**The database is already correct**: no float columns anywhere in the four
schemas, money `numeric(15,2)`, quantity and rate `numeric(18,6)`.

**The risk is the two layers above it, and one of them is already leaking.**

### 6a · The leak, found 2026-09-22

`src/modules/sales/sale-order/sale-order.service.ts` converts every Decimal to
a JavaScript `number` (float64) and does the arithmetic there:

```ts
const AMOUNT_EPSILON = 0.005;                      // line 186
function asNumber(v: unknown): number { … Number(v) … }
function roundAmount(v: number) { return Math.round(v * 100) / 100 }
const unitShare = netQty > 0 ? asNumber(line.soiNetAmt) / netQty : 0;
Math.abs(rollup.billedAmt - asNumber(order.soBilledAmt)) > AMOUNT_EPSILON
```

**`AMOUNT_EPSILON` is the tell.** A system that compares money with a tolerance
has already accepted that its arithmetic drifts. And `Math.round(v*100)/100` is
not safe rounding: `1.005` is stored as `1.00499999…` in float64, so it rounds
**down** to `1.00`. On one line nobody notices. Across a supermarket's daily
line count, distributed charges and per-line tax, it is a real rupee.

**79 files already use `Prisma.Decimal` correctly.** The order service is the
exception — and it is the exception **A3 will be copied from**, because the bill
does the same work: distributing charges across lines, rolling up, comparing
against a stored total.

### 6b · The rule to hold

1. **Money and quantity never become a JS `number`.** Keep `Prisma.Decimal`
   (or `decimal.js`) end to end: add, subtract, multiply, divide, compare.
2. **No epsilon comparisons on money.** With decimals, equality is equality. An
   epsilon anywhere is a bug report waiting to be filed.
3. **Round once, at the column**, with the decimal library's own rounding and a
   stated mode (`ROUND_HALF_UP` for money in India), to the column's scale.
4. **Distribute by largest-remainder, never by rounding each share.** Split a
   charge across lines and the parts must sum to the whole **exactly** — compute
   each share, round down, then hand the remaining paise to the largest
   remainders. Rounding each line independently loses or invents money.
5. **The client obeys the same rules.** Qt must not compute a total in `double`
   and send it; the server recomputes and the two must agree to the paise.
6. **The server is authoritative.** A figure the client sends is a proposal —
   §10 already lists what the server ignores. Totals belong on that list.

---

## 7 · Void, the session, and the reason all of this exists — **DEFERRED 2026-09-22**

> **Parked by the user, not cancelled:** *"no need void bills problem and session
> issue we can handle later."* Kept in full — the reasoning is the expensive
> part. `print_log` (§7b) already gives partial cover today.

The user described the failure honestly, and it is not hypothetical:

> *"user can bill but won't take print — he will say printer not worked, so the
> customer unknowingly gives the amount to the user, he clears the bill and
> takes the money."*

Plus the simpler one: *"the system error if any mis-calculation."*

### 7a · Why "cancel" does not cover this

Cancel is an **accounting** correction: it reverses stock and legs and leaves an
audit trail. It answers "this document was wrong."

It does not answer **"was there money on the counter that is now not in the
drawer?"** — and that is the question a supermarket has to be able to ask about
any shift, on any day, without trusting the operator's account of it.

### 7b · The control that ALREADY exists, and is the strongest one

**`public.print_log`.** It is built, it is data-driven, and it records **every
print attempt**:

```
plg_src_doc_type · plg_src_doc_id · plg_printer_id · plg_output_mode
plg_status · plg_error · plg_page_count · plg_duration_ms
plg_printed_on · plg_printed_by · plg_copy_no
```

So *"the printer wasn't working"* is a **checkable claim**:

| what `print_log` shows | what it means |
|---|---|
| a row with `plg_status` failed + `plg_error` | the printer really did fail. The claim is true |
| **no row at all** | **no print was ever attempted.** The operator did not try |
| a successful row, then the bill cancelled | a bill was printed, handed over, and then cancelled |

That third line is the theft pattern, and it is one query. **`print_log` is
currently missing from `ensure_acc_year_partitions`** (a known gap, noted as
"not an issue" because it only bites on the 2028-29 roll) — if it becomes an
anti-theft control, that gap needs closing before it matters.

### 7c · VOID BILL — what to build

A void is **not** a delete and **not** a cancel. It is a recorded statement that
a document raised at the counter did not complete.

* **The number is kept.** Voided, visible, numbered, in the list with a VOID
  badge. A missing number is the loudest possible alarm, so numbers must never
  go missing.
* **A reason is mandatory**, from a short controlled list (not free text alone):
  customer left · wrong item · price dispute · **printer failure** ·
  operator error · system error. Free text in addition.
* **Stamped**: who, when, which **session**, which device, how long after the
  post, and **whether a successful `print_log` row existed at the moment of the
  void.** That last field is the whole control — computed at void time, stored,
  never recomputed later.
* **Manager approval** required when any of these is true, and the thresholds
  are settings, per company:
  * the bill was **printed** before the void
  * the value is over `VOID_APPROVAL_AMOUNT`
  * more than `VOID_SESSION_LIMIT` voids already in this session
  * the void is more than `VOID_MINUTES` after the post
* It reverses exactly what a cancel reverses — stock, legs, register, caches,
  loyalty, promotion. The accounting is a cancel; the **evidence** is the
  difference.

### 7d · VOID ITEM — the quieter one

The same vector, one line at a time: scan ten items, let the customer see the
total, take the cash, void three lines and re-total. It never shows up as a
cancelled bill.

* Every line removed or reduced **after it was added** is logged — item, qty,
  value, who, when, session, and **whether the bill had already been printed**.
* A void-line log is kept for DRAFT lines too. "It was only a draft" is exactly
  the cover this needs to not have.
* The per-session **count and value** of voided lines is what gets watched. One
  is a mis-scan. Forty in a shift, always by the same operator, always in the
  last minute of a sale, is not.

### 7e · THE SESSION — the missing table everything hangs on

`session_id` is already on all six headers, on every tender row, on every
voucher and every bill adjustment. **It points at nothing.** Build the table it
should have pointed at:

```
till_session
  open:   operator, device, branch, opening float (counted, typed),
          opened_on, opened_by
  during: every txn written in it carries its id (already true)
  close:  closing count by tender (cash counted, card settled, UPI settled),
          expected vs counted, the DIFFERENCE, closed_on, closed_by,
          and a manager sign-off when the difference exceeds a threshold
```

What it gives that nothing else can:

1. **A drawer that reconciles.** Expected cash = opening float + cash tenders −
   cash refunds. Counted cash is typed in. The difference is a number somebody
   signs for, per shift, not per month.
2. **Voids get an owner.** "Six voids, 4,200.00, all by one operator, all
   within two minutes of posting, five of them after a successful print" is a
   sentence the system can produce on its own. Without a session it is a
   spreadsheet job nobody does.
3. **A day close that means something.** Day close is on the **ledger**, not the
   tenders — that was already settled — but the *session* is what ties a
   counted drawer to the ledger it should agree with.
4. **Re-tender, temp credit and adjustments already carry the id**, so they come
   into the same reconciliation for free.

**Order of build:** the session table first, then void, because a void without a
session is an event with no owner — and an owner is the entire point.

---

## 8 · What this changes in the three mockups

| mockup | change |
|---|---|
| `sale_return_ui_mockup.png` | add the barcode scan hint on the grid; add "scan ref to load"; state the line-godown rule on the grid legend |
| `dc_return_ui_mockup.png` | same three |
| `sale_return_settle_ui_mockup.png` | no change — it already opens per mode |
| **new** | a **void + session** mockup, per the user's PNG-first rhythm. That is the one worth drawing next, because §7 is the part with no existing screen at all |

---

## 9 · Open questions — CLOSED 2026-09-22, all three belonged to the deferred §7

1. **Does a void exist on all six, or only the bill?** A quotation void is
   meaningless; a **challan** void is not (goods left the building). My reading:
   bill and challan void; the other four cancel only.
2. **Does the session gate login?** i.e. can an operator bill at all with no
   open session, or does the first bill open one implicitly? Implicit is kinder
   at a counter; explicit is what makes the opening float real.
3. **One session per device, or per operator?** Two operators sharing a till on
   one shift is common in a supermarket, and it decides whether the drawer
   difference has one name on it or two.
