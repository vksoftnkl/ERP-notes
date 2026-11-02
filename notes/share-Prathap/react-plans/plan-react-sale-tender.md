# Sale Tender (settlement / advance) — React port plan

Scope: **one component family**, `SaleBillTender`
(`src/modules/sales/salebill/sale_bill_tender.{h,cpp,ui}`), ported to
React/TypeScript against the unchanged backend.

This is a **delta plan**. `plan-react-sale-order.md` §5 sketched this dialog in one
page; **this document is that section, in full**, and the order plan should be read
as pointing here. Nothing is forked: the bill screen and the order screen mount the
**same** `<TenderDialog>` with a different `purpose`.

Read `plan-react.md` (draft/reducer shape, money handling, API client) first.

> **Correction to `plan-react-sale-order.md` §5.1.** That section warns that the Qt
> per-row ceiling compares the *gross* against the bill. It no longer does —
> `checkInputValues()` computes `base = amount − surchargeAmt` and compares that.
> Port what is in the file today; the warning is stale.

---

## 1. What this screen actually is

One row per configured tender, ordered by the master's display position, with
**exactly one editable column: Amount**. Everything else a row knows — ledgers,
limits, surcharge, reference rules, colour, hotkey — lives in the tender master
beside the row, never in hidden grid columns. That is what keeps it a 1-column
table instead of a 30-column one.

The single most important rule in the whole screen:

> **Behaviour comes from the tender MASTER, not from the tender type's name.**

A shop can add a second card tender with its own surcharge, its own ledger and its
own reference label, and nothing in the component knows or cares. The type name is
used for exactly four things — which types an advance may not offer (§6), whether
CASH always gives change (§5), which row owns the redemption panels (§9, §10), and
which instrument fields to show (§7). Everything else reads the master row.

---

## 2. Module layout

Lives under `sale/tender/` and is imported by both entry screens. **Do not fork it
per screen.**

```
src/features/sale/tender/
  TenderDialog.tsx          // shell: rows + totals + the right-hand panel column
  components/
    TenderRows.tsx          // the 1-column table, hotkeys, F1
    TotalsChips.tsx         // Bill Total / Tender / Surcharge / Balance
    ReferencePanel.tsx      // switches on instrument (§7)
    RedemptionPanel.tsx     // credit notes (§9)
    LoyaltyPanel.tsx        // points (§10)
  domain/
    arithmetic.ts           // PURE — surcharge, totals, balance, change routing
    rows.ts                 // master JSON -> TenderRow, type-rule defaults
    instruments.ts          // which fields per type, and their labels
    validate.ts             // PURE — the ordered gate list (§12)
    payload.ts              // TenderRow[] -> the `tenders` array (§13)
  state/
    useTenderDraft.ts       // reducer over rows + panels
  api/
    tenderMasters.ts        // GET /tender-masters/list
    openReturns.ts          // GET sale-returns open list
  types.ts
```

`domain/` has no React import in it. That is the point: §5, §9, §10, §12 and §13 are
all testable without mounting anything.

---

## 3. Data in — the tender master

`GET /tender-masters/list`. No params; the screen filters.

**Offerable** = active **and** not deleted **and** within `tndEffectiveFrom` /
`tndEffectiveTo` **measured against the DOCUMENT's date, not the wall clock**. A
bill dated back into an earlier period must offer the tenders that were live then.
Take the date from the draft, not from `new Date()`.

Then sort by `displayPosition`, `name` as the tie-break. Display position is the
counter's muscle memory — cash first, then whatever the shop uses most.

### 3.1 The three nullable flags

`needsRef`, `allowChange`, `allowInReturn` are **NULLABLE overrides** of the tender
TYPE's defaults (`schema/04_tenders.sql`). The API sends `{}` for NULL.

```ts
// WRONG — {} is falsy-ish and quietly stops cash giving change
allowChange: Boolean(t.tndAllowChange)
// RIGHT
allowChange: isNullish(t.tndAllowChange) ? typeRule.allowChange : t.tndAllowChange
```

Mirror the type table exactly (`rows.ts`) until `/tender-masters/list` returns the
type's own behaviour columns — then read those and **delete the table**:

| type | needsRef | ref label | allowChange | allowInReturn |
|---|---|---|---|---|
| CASH | no | — | **yes** | yes |
| CARD | yes | Card Last 4 | no | yes |
| UPI | no | UTR No | no | yes |
| WALLET | yes | Txn Ref | no | yes |
| CHEQUE | yes | Cheque No | no | yes |
| BANK | yes | UTR / Ref No | no | yes |
| RRN | yes | RRN | no | **no** |
| TEMP_CR | yes | Approved By | no | no |
| CREDIT | no | — | no | no |
| LOYALTY | no | — | no | no |
| VOUCHER | yes | Voucher No | no | no |

### 3.2 Fallback seeding — never leave the counter with nothing

Three failure shapes, three different sentences, all of them seeding CASH (plus
CREDIT in settlement mode) by hand with no ledger — the backend resolves that:

| situation | hint bar says |
|---|---|
| call failed | *Tender master unavailable (…) — only cash can be taken.* |
| list empty | *No tender is configured — add one in the Tender Master.* |
| nothing offerable | *No configured tender can be used here — check the Tender Master's active dates.* |

The reason **replaces the keyboard hint for the rest of the dialog**. A screen
offering one tender when the shop has six is a fault to be fixed in the master, not
a layout the operator should quietly get used to.

---

## 4. Row state, and the two amounts

Per row the cashier keys a **base** — what the customer is settling with. The row's
**amount** is base + surcharge, which is what the customer is actually charged. For
the overwhelmingly common surcharge-free tender the two are identical, which is
exactly why the distinction gets lost if it isn't named in the types:

```ts
interface TenderRowState {
  base: number;          // keyed
  surchargePerc: number; // from master, editable only when master.editSurcharge
  surchargeAmt: number;  // derived
  amount: number;        // base + surchargeAmt — what is charged, what is stored
  bankId?: string; bank?: string;      // bank master id + its NAME (payload carries the name)
  cardNo?: string; cardExpiry?: string;
  refNo?: string;
}
```

### 4.1 Typing vs committing — port this deliberately

Two distinct events, and collapsing them breaks the keyboard:

- **per keystroke** — recompute that row's surcharge/amount and the totals, but
  **leave the cell text exactly as keyed**. Folding the surcharge into the text mid-number
  moves the digits out from under the cursor.
- **on commit** (blur / Enter) — validate the row, then rewrite the cell to the
  charged amount.

In React: keep the raw string in row-local state, derive numbers from it on every
change, and only normalise the displayed string on commit.

### 4.2 Rows the operator may not type into

RRN and LOYALTY amounts are **owned by their panels** (§9, §10). Typing over the
summary would only desync the two, so a commit on those rows re-renders the value
from state and opens the panel instead.

---

## 5. Arithmetic — `domain/arithmetic.ts`, pure, written and tested first

```
surcharge(base, master, perc) = base <= 0 ? 0 : (base * perc / 100) + master.surchargeAmount
rowAmount                      = base + surcharge
tendered                       = Σ rowAmount
surchargeTotal                 = Σ surchargeAmt
balance                        = tendered − surchargeTotal − documentAmount
```

Note the flat fee applies **only once the row is used** (`base > 0`), so an untouched
card row doesn't quietly add ₹5 to the bill.

> **The surcharge is the bank's cut, not the shop's takings.** It settles no part of
> the document, so it comes back out of the balance, and **every comparison against
> the document amount uses the base**, never the gross. This is the one rule that,
> got wrong, produces a screen that refuses to settle a full bill on a card that
> carries a fee.

**Balance presentation** — one box, three states, and the caption changes with it:

| balance | colour | caption (settlement) | caption (advance) |
|---|---|---|---|
| `= 0` | grey `#A9A9A9` | Refund | Refund |
| `< 0` | red `#FF4500` | Balance | **amber `#FB8C00`** · Balance to Collect |
| `> 0` | green `#006400` | Refund | Refund |

On an advance a shortfall is the **normal** case — it is what will be billed later —
so it is not painted as a problem. Amber says "still to collect", not "you are short".

---

## 6. `purpose` — settlement vs advance

```ts
type TenderPurpose = 'settlement' | 'advance';
```

Not a preference; two different transactions.

| | `settlement` (bill) | `advance` (order) |
|---|---|---|
| window title | Settle Bill | Receive Advance |
| total captions | Bill Total / Tender Amount | Order Amount / Advance Received |
| must the total be covered? | **yes** | no |
| types offered | all | **minus RRN, LOYALTY, CREDIT, TEMP_CR** |
| open credit notes fetched? | yes | **never** — no row to redeem them on |

Why those four are dropped: a credit note closes out against an invoice and loyalty
is earned on the bill that moves the goods — neither exists yet; CREDIT and TEMP_CR
collect nothing, and an under-collected order simply carries a balance until billed.

---

## 7. The reference panel — `instruments.ts`

One right-hand column that swaps content with the selected row (and collapses when
the row needs nothing). Per type:

- **CARD** — bank (dropdown 46, the bank master), card no (**last 4 only**), expiry
  `MMM/yyyy`, slip no.
- **CHEQUE** — bank, **cheque no is the reference**, cheque date `dd-MM-yyyy`,
  `isPdc = chequeDate > documentDate`.
- **UPI / WALLET / BANK / VOUCHER / TEMP_CR** — one reference field, labelled from
  the type (`UTR No`, `Txn Ref`, `Voucher No`, `Approved By`).
- **CASH / CREDIT** — nothing.

Two rules that look like details and are not:

1. **The cheque panel must not be gated on `needsRef`.** A master row with the flag
   off would otherwise leave no way to key a cheque at all.
2. **`cardLast4` is CHECK-constrained to `^[0-9]{4}$`.** The reference block is free
   text across types, so a cheque number or an approval name must go as **null**,
   not as its last four characters — the insert would be rejected. One helper,
   used by every screen that serialises a tender line:

```ts
export const cardLast4 = (s?: string) =>
  /^\d{4}$/.test((s ?? '').trim().slice(-4)) ? (s as string).trim().slice(-4) : null;
```

---

## 8. The credit gate

A cash-only customer's CREDIT row stays **keyable**. Using it asks once —
*"This customer is not allowed to buy on credit. Put this bill on credit anyway?"* —
and the answer is remembered **for that settle**. Re-asking per keystroke trains the
answer out of anyone.

The cursor still does not *land* on that row when the dialog opens (`rowIsUsable`);
suggesting it would be strange. Keyable, not suggested — those are different things.

F1 onto the CREDIT row (§11) goes through the same confirmation.

In React the "asked already" flag is draft state, not a module variable.

---

## 9. Credit-note redemption (the RRN row)

Fetched **once per settlement and only when an RRN row is actually offered**.
Endpoint: the sale-returns open list, keyed by customer + company + branch + accYear.

```
balance = srReturnAmt − srRefundAmt − srAdjustedAmt      // computed here
```

Computed client-side rather than trusted from a balance column, so a backend that
sends only the raw three figures still works. Notes with `balance <= 0` are dropped.

Panel columns: **Ref No · Date · Balance · Adjust**. The note's face value is
deliberately not shown — in a narrow panel the cashier needs *which note, when, what
is left, what to take*.

The RRN row's amount is **the sum of the adjustments**, and on save that one row
**expands into one settled line per note adjusted** (§13) — each naming its own
document, because a lump sum cannot be closed out against anything.

Failure is never blocking: the panel stays empty, the RRN row simply can't be used,
and every other tender still works. The operator is mid-settlement.

---

## 10. Loyalty redemption (the LOYALTY row)

- **Points** come from the customer (`cust_points`, the same figure the bill screen's
  credit panel shows).
- **What they are worth** comes from the LOYALTY tender's own `conversionRate`
  (fallback 1) — `worth = points × rate`.
- **The floor** is that master's `minAmount`.

Usable = `loyaltyAllowed && worth > 0 && worth >= minAmount`. When it isn't, the row
**stays visible, read-only**, and the hint bar says which of the two reasons applies
(*"This customer is not on the loyalty scheme."* / *"Needs at least X to redeem."*).
A row that vanishes teaches nobody anything.

Panel shows: total points, rate, total worth, minimum, redeem amount (the only input),
points used = `redeem / rate`, balance points. Over-redeeming warns and resets that
row to zero rather than clamping silently.

---

## 11. Change, and who gives it

```ts
givesChange(master) = master.allowChange || master.typeName === 'CASH'
```

CASH always can, regardless of the master flag — the drawer is exactly what change
comes out of, and taking a ₹500 for a ₹470 bill is how a counter works. Everything
else goes by the master: a card or UPI collection can't hand anything back.

On save the change goes back out of the **first** change-capable row, all of it, as
that line's `refundAmt`. If change remains and no row can give it:

> *No tender on this bill can give change back. Reduce the over-tendered amount, or
> add a cash tender.*

…and the save is refused. (Nothing here is a silent clamp.)

**F1** — pay the outstanding balance with the current row: `newBase = currentBase −
balance`, then the surcharge is recomputed from the master. Does nothing when
nothing is outstanding.

**Hotkeys** — from the master, falling back to `A…L` by position; each jumps the
cursor to that row's amount.

---

## 12. Validation — `validate.ts`, pure, in this order

Return the first failure with `{ message, focusRow, focusField }`; the dialog does
the focusing.

1. **Settlement only** — `balance < 0` → *The bill is not fully tendered.*
   (An advance is partial by definition and skips this.)

Then per row with `amount > 0`, on the **base**:

2. `base < master.minAmount` → *%1 needs at least %2.*
3. `!givesChange(master) && base > documentAmount` → *%1 cannot exceed the bill
   amount — it can't give change back.* — **unless** the excess-tender setting is on
   (`sales.allow_excess_tender`, see `schema/10_app_settings.sql`).
4. `(needsRef || type === CHEQUE) && !refNo` → *%1 needs its %2.* → open the panel,
   focus the reference.
5. `type === CHEQUE && !bank` → *%1 needs the bank the cheque is drawn on.* A cheque
   nobody can present is not a settlement.
6. `type === RRN && |amount − Σ adjusted| > 0.005` → *The adjusted credit notes come
   to %1, not %2.* They can only drift if a note was edited after the summary was
   written — but a mismatch posts money against no document.
7. `type === LOYALTY && amount > worth + 0.005` → *Only %1 of points is available.*

**`tndDailyLimit` is deliberately NOT checked here.** It bounds the day's takings on
that tender across every counter, which this screen cannot see. The backend enforces
it on save. Do not reimplement it client-side and do not "helpfully" warn — a wrong
warning about a limit is worse than none.

---

## 13. What it hands back

```ts
onSettled({
  tendered,        // Σ amount — includes surcharge, it is what crossed the counter
  credit,          // Σ amount of CREDIT rows
  refund,          // the balance box (change handed back)
  surcharge,       // Σ surchargeAmt
  lines,           // TenderLine[] — the `tenders` array of the save payload
  print,           // Save & Print was used
})
```

Per line the master travels with the money, because a tender posts to one ledger
now, may clear through another T+n days later (card/UPI), and books its surcharge to
a third: `tenderId, tenderTypeId, type, tenderName, shortName, ledgerId, ledgerName,
settlementLedgerId, settlementDays, surchargeLedgerId, bankAccountId, conversionRate,
openCashDrawer`.

Emit rules:

- a line is emitted when `amount > 0 || refundAmt > 0`;
- **RRN expands** — one line per adjusted note, each with `amount = note.adjusted`,
  `surchargeAmt = 0`, `refundAmt = 0`, `refNo = note.refNo`, `srcReturnId = note.id`;
- `cardNo` goes through `cardLast4()` (§7);
- the document's own totals take **net** figures: what settles the document is
  `tendered − surcharge − refund`, and pay-status is decided on that net. There is no
  `paid_amt` column on the order, so on load it is re-derived the same way.

---

## 14. Keyboard

| key | does |
|---|---|
| `F1` | pay the outstanding balance with the current row |
| master hotkey / `A…L` | jump to that row's amount |
| `Enter` | commit the cell (§4.1) |
| `Esc` | close without settling |

The hint bar carries the keyboard summary at rest, and is taken over by the
fallback-seeding reason (§3.2) when that happens.

---

## 15. Tests

Pure modules first, and these are the cases that actually break:

- **arithmetic** — surcharge only when base > 0; flat + % together; balance nets the
  surcharge out; balance colour/caption per state × purpose.
- **rows** — `{}` on the three flags inherits the type default (the cash-change bug);
  effective dates judged against the document date; sort stability with equal
  positions; each of the three fallback-seeding shapes.
- **validate** — every gate in §12, plus the card-that-carries-a-fee settling a full
  bill (must pass), and the excess-tender setting flipping gate 3.
- **payload** — RRN expansion into per-note lines; `cardLast4` dropping a cheque
  number to null; change routed to the first change-capable row; the refusal when
  none can give it.

---

## 16. Backend gaps this screen inherits

1. `/tender-masters/list` does not return the tender TYPE's behaviour columns, so
   §3.1's table is duplicated on the client. Fix the endpoint, delete the table.
2. Open credit notes: the endpoint may send only the raw three amounts — the balance
   is computed here (§9). If a `balance` column arrives later, prefer it.
3. Daily limit is server-only (§12) — the client cannot show *why* a save was
   refused, so the error message the API returns has to be human-readable.
4. Loyalty: there is no loyalty ledger yet. Points are read from the customer and
   spent as an amount; nothing writes the redemption anywhere but the tender line.

---

## 17. Phases

1. `arithmetic.ts` + `rows.ts` + `validate.ts` with tests. No UI.
2. `<TenderDialog>` with rows, totals chips, F1, hotkeys — CASH and CREDIT only.
3. Reference panel + instruments + `cardLast4`.
4. `purpose='advance'` (captions, offered types, no fetch of notes) — the order
   screen can now settle.
5. Redemption panel.
6. Loyalty panel.

Phases 1–4 are a working settle screen for both callers. 5 and 6 are the legacy
panels and can land later without touching anything above them.

---

## 18. Do not port

- The commented-out "Last Tender" label — it was never wired.
- The hidden-cell approach to row data. The master is the row's state; the table has
  one column and keeps it.
- Any client-side daily-limit check (§12).
