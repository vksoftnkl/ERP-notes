# Notes (38) — a RE-PRESENTED cheque escapes the amend and cancel guards

**Viji, 2026-09-19. Found by the user on live receipt `rct00819`.**

**FIXED AND VERIFIED the same day — retested end to end, all green.** Kept
because the checks below are the regression list.

| retested | result |
|---|---|
| amend, re-presented cheque CLEARED | **409**, bill did not move |
| cancel, re-presented cheque CLEARED | **409** |
| amend, re-presented cheque DEPOSITED | **409** |
| cancel, re-presented cheque DEPOSITED | **409** |
| amend while HELD (never re-presented) | still **allowed**, old reversed, new written, cheque soft-deleted |
| amend while DEPOSITED (never re-presented) | **409** |

The original defect, for the record: it lost money — a 500 payment settled
1,000 of debt.

It is a consequence of notes (37)'s re-issue voucher, so it arrived with that
change. The (37) fix itself is correct and verified — this is a second place
that needed to know about it.

---

## 1. The hole

`/receipts/amend` and `/receipts/cancel` both refuse when the receipt's cheque
is DEPOSITED or later. They resolve "the receipt's cheques" by
**`acc_pdc_register.apd_voucher_id`**.

**`/cheques/re-present` repoints `apd_voucher_id` to the re-issue voucher.**
From that moment the original receipt looks cheque-free, and both guards wave
it through — while the cheque is alive, cleared, and settling a bill.

```
rct00821 posted, cheque 107960 (500) -> bill A78561    apd_voucher_id = rct00821
bounce                                                  -500 off A78561
re-present            -> re-issue voucher rct00822      apd_voucher_id = rct00822   ← moves
clear                                                   cheque CLEARED
amend rct00821 cheque -> cash                           HTTP 201  ← SHOULD BE 409
```

Result, measured:

```
bill A78561 alloc   2,500.00  ->  3,000.00
cheque still settling 500.00, and the new cash settled another 500.00
```

**The customer handed over 500. One thousand of debt was cleared.** The
cheque's own settlement was never reversed, because the amend did not believe
there was a cheque.

## 2. `/cancel` has it too

Same setup, cancel instead of amend:

```
CANCEL rct00824   HTTP 201  "cancelled — 2 voucher(s) reversed"   ← SHOULD BE 409
```

leaving:

| receipt | status | cheque | cheque status | cheque now belongs to |
|---|---|---|---|---|
| rct00824 | **CANCELLED** | 105617 | **CLEARED** | rct00825 |

A cancelled receipt whose cleared cheque is still settling `A78561` through the
re-issue voucher. Money in the bank against a document that says it never
happened.

## 3. The user's live case, for reference

`rct00819` (500, party SATHEESH), cheque `NMK001`:

```
14:06  rct00819 posted        +500 -> bill 137
14:07  bounced                -500 off 137
14:08  re-presented           +500 -> 137, under re-issue rct00820
       cleared
14:11  AMENDED cheque -> cash +500 -> bill 135      ← allowed, nothing reversed
```

`rct00819` now carries two live allocations totalling 1,000 on a 500 receipt,
against two different bills. Bill 135 is over-settled by 500.

## 4. What I think the fix is

The guards must find a cheque by **what it was taken in on**, not by who owns
it now. Either:

- **keep an immutable originating-receipt column** (`apd_origin_voucher_id`, or
  whatever the re-issue chain already records) and guard on that; or
- **walk the chain**: a cheque whose `apd_voucher_id` is a re-issue voucher
  still belongs, for guard purposes, to the receipt that voucher was issued
  against.

Whichever way, the rule stands unchanged: *a receipt with a cheque DEPOSITED or
later cannot be amended or cancelled.* Re-presenting must not make a receipt
newly amendable — if anything it is stronger evidence that the money moved.

Worth checking anything else that asks "which cheques belong to this receipt?"
— printing and the receipt's own detail strip resolve it the same way.

## 5. Clean-up state

Both of my reproductions are cancelled and their bills are back where they
started. **`rct00819` and `rct00820` are the user's own and I have left them
alone** — bill 135 is over-settled by 500 until somebody decides how to unwind
it.
