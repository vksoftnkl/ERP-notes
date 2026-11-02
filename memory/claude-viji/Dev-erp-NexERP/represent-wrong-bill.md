---
name: represent-wrong-bill
description: "/cheques/re-present without allocations settles the WRONG bill (FIFO auto-pick) — the client cannot fix it, notes (37) asks Prathap"
metadata:
  type: project
---

**FIXED by Prathap 2026-09-19, same day — verified end to end. Kept because
the verification is the regression list, and because the near-miss is worth
remembering.**

The defect was: `POST /cheques/re-present` with no `allocations` allocated the
cheque to an auto-picked bill (the party's first by date/ref), NOT the bill it
came off. Written up as
`share/cheques/notes-37-represent-allocation.md`.

Traced with one 2,000 cheque on bill MKL785, watching `abl_alloc_amount`
against the sum of that bill's live adjustment rows:

```
after POST       17,000   -> MKL785
after BOUNCE     15,000   -> reversed off MKL785, correct
after RE-PRESENT 15,000   -> the +2000 landed on A7856      WRONG BILL
```

Sending `allocations` explicitly was correct even then, so it was the FALLBACK
that was broken, not the route.

**What the fix now guarantees — all four verified against the live server:**

| case | result |
|---|---|
| single bill, no `allocations` | restored to the originating bill (15,000 → 17,000) |
| **cheque split 1500/1000 over two bills** | both restored exactly, per-bill split intact |
| originating bill paid off since the bounce | **409 naming the bill and the figures**, cheque stays BOUNCED, **zero rows written** |
| explicit `allocations` | still overrides, money goes where told |

The multi-bill case is the one that mattered: that split lives only in the
reversed adjustment rows, so no caller could ever have reconstructed it.

**`ChequeRePresentPanel::payload()` sends no `allocations`, and that is now
CORRECT** — the server restores the split itself. Do not "helpfully" add one:
the client cannot see the per-bill split after a bounce (`/cheques/get` reports
`settledByThisCheque: 0` for every bill, correctly, because the rows are
reversed), so anything it sent would be a guess overriding a server that knows.

**Everything around it is correct — verified, do not "fix":**
- amend with cheque HELD → allowed, old allocation reversed, new tender's
  written, cheque soft-deleted and gone from grid 109 and `/cheques/list`
- amend with cheque DEPOSITED / BOUNCED / CLEARED → **409** naming the status
- bounce reverses cleanly off the right bill
- 23 of 24 bills on the box reconcile to the paisa; the 24th is the known
  post-dated cheque ([[bill-adjustment-reverses]])

**`/cheques/bounce` field names are NOT what the columns suggest:** `reason`,
`bankCharge`, `partyCharge` — not `bounceReason` / `bounceCharges`. A wrong
payload is a 400 that is easy to miss if the caller does not check, and then
the cheque silently stays DEPOSITED.
