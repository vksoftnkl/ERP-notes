# Notes (36) — what a LOADED receipt comes back as

**Viji, 2026-09-19.** Found from a screenshot of receipt `rct00702` after one
amend.

**UPDATE, same day — you shipped §3 and the extras, and the client is wired to
them.** `allocations[]` now carries `billAmount`, `pendingAmount`, `dueDate`,
`billType`, `status`, `reversalOfId` and `isReversed`, and the figures check
against `acc_bill_balance` exactly. Two notes on how the client reads them, at
the bottom. §4 is still an open question.

---

## The symptom

A posted receipt that settles TWO bills painted FOUR grid lines after a
correction — the same bill three times, one of them **−2,000 in an editable
Receive cell**, with the After column reading −2,000.

## 1. `allocations[]` is the voucher's HISTORY — FIXED IN THE CLIENT

`acc_bill_adjustment` never rewrites a row and never soft-deletes one; it
retracts by inserting a negative counter-row (`ck_abj_reversal_sign` makes that
the only legal shape). So `/receipts/get` on an amended receipt answers:

```
bill 125   ALLOCATION   +2000      the original
bill 125   ALLOCATION   +1000      what the amend replaced it with
bill 125   ALLOCATION   -2000      the reversal of the first
bill 128   ALLOCATION   +1000
```

The client now **nets per bill** before painting, which is exact rather than a
guess: a reversal is by constraint the precise negative of the row it retracts,
so nothing has to be paired up. **No change wanted from you here** — the raw
history is arguably the more honest payload, and netting it is the reader's
job.

It would still be worth adding **`reversalOfId`** (and/or `isReversed`) to the
row. Not for the netting — for a history view that wants to SHOW the
correction, which today is impossible to reconstruct from the payload.

## 2. adj_type drives which COLUMN — FIXED IN THE CLIENT

A settlement, its discount, its write-off and its round-off are four rows
against ONE bill, each with its own `abj_adj_type`. The client was reading
non-existent `discount` / `writeoff` fields, so both stayed 0.00 **and** the
discount appeared as its own grid line, looking like a second bill being paid
100. Now routed by `adjType`. No server change needed.

## 3. The bill's own figures — DONE, THANK YOU

`allocations[]` carries `billId`, `billAccYear`, `docRefno`, `docDate` — and
nothing else about the bill. So on a posted receipt the grid shows:

| Bill amount | Paid | Pending | Due | Days | After |
|---|---|---|---|---|---|
| 0.00 | 0.00 | 0.00 | — | — | **−2,000.00** |

`After = pending − settled`, and with pending 0 that is minus the settlement on
every row. The real bill (`125`) is 15,500 with 14,500 pending.

**Please add to each `allocations[]` row:**

```jsonc
"billAmount":    15500,
"pendingAmount": 14500,   // as it stands NOW, after this receipt
"dueDate":       "2026-10-18",
"billType":      "SALES",
"status":        "PARTIAL"
```

`pendingAmount` **as it stands now** is the useful one — the client can derive
what it was before by adding this receipt's own settlement back, and that is
what makes the After column true on a document being read.

The screen deliberately does NOT call `/open-items` for a posted receipt: what
a receipt shows is what it did, not what the party currently owes. So these
figures cannot come from anywhere else.

## 4. A question, not a defect: cancel creates a receipt-numbered voucher

Cancelling `rct00615` wrote its reversal legs and adjustment rows under a NEW
voucher **`rct00617`** — POSTED, in the receipt series, no tenders, legs the
exact mirror (CR 10, CR 1, CR 1, DR 12), three negative allocations.

It therefore appears in the receipt list as an ordinary receipt and opens as
one showing −12. Is a reversal voucher meant to take a receipt number and sit
in that list? If it is, the client needs a way to tell one apart —
`avh_src_doc_type`, a flag, anything — so the register can label it and the
entry screen can open it read-only instead of as a receipt somebody might try
to correct.

---

Nothing here blocks the client; 1 and 2 are already in. 3 is what stops a
posted receipt showing honest figures.


---

# How the client reads the new fields (2026-09-19, after your change)

## `pendingAmount` is served as it stands NOW, and the row shows it as it STOOD

Right call to serve it as-is — it is the unambiguous one. The screen adds this
receipt's own settlement back before painting, so a document being READ shows
the bill as it stood and what the receipt left it owing:

```
bill 125   billAmount 15,500   pendingAmount 14,500 (served)
           -> Pending 15,500   Receive 1,000   After 14,500
```

Painted straight through, After would read 13,500 — a figure that bill has
never carried. `mergeAmendAllocations()` already did the identical add-back on
the `/open-items` list when correcting a posted receipt, so both paths now
agree.

## `matured` turned out to matter more than expected

**Only the MATURED part is added back.** A post-dated cheque writes its row and
does not move `abl_pending_amount`, so there is nothing to undo. `rct00018`
holds 800 post-dated against a bill of 7,500 showing 5,380 — a blind add-back
paints Pending 6,180, a number that bill has never carried and never will. The
800 goes to the PDC held column instead.

So `matured` is load-bearing on this payload, not decoration. Please keep it
accurate for every settlement mode, not just cheques — if anything else ever
settles a bill later than the receipt date, it needs the same flag.

## `reversalOfId` / `isReversed` — taken, but NOT what the netting runs on

Useful to have, thank you. The netting still **sums per bill** rather than
filtering on `isReversed`, deliberately: nothing in the schema forces a
reversal to be the WHOLE of what it retracts — `ck_abj_reversal_sign` only
fixes the sign and `ux_abj_reversal` only caps it at one per original. Drop an
original because it is flagged reversed and a *partial* reversal would silently
lose its residual. Summing is right whatever shape the rows take.

If a partial reversal is in fact impossible by design, say so and I will
simplify; if it is possible, the flags are still the right way to SHOW a
correction, which is what they will get used for.
