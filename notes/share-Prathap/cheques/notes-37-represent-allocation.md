# Notes (37) — a re-presented cheque settles the WRONG BILL

**Viji, 2026-09-19. Reproduced end to end on 192.168.0.106.** This one moves a
customer's money to a debt they did not pay, silently. Nothing on the client
can work around it correctly — see §4.

---

## 1. What happens

`POST /cheques/re-present` **with no `allocations`** re-allocates the cheque to
an auto-picked bill (the party's first by date/ref) instead of the bill the
cheque came off.

Traced with one 2,000 cheque against bill **MKL785**, printing the bill's
`abl_alloc_amount` and the sum of its live `acc_bill_adjustment` rows at every
step:

```
                        bill        rowsum
start                  15,000.00   15,000.00
after POST (2000)      17,000.00   17,000.00   -> MKL785
after DEPOSIT          17,000.00   17,000.00
after BOUNCE           15,000.00   15,000.00   -> reversed off MKL785, correct
after RE-PRESENT       15,000.00   15,000.00   -> +2000 landed on A7856  ✗
```

The +2,000 row is real, live and `abj_bill_id = A7856` — a bill this cheque was
never against. `A7856`'s own `abl_alloc_amount` moved instead. The customer's
re-presented cheque paid somebody else's invoice.

`GET /cheques/get` then reports it as fact:

```jsonc
"bills": [
  { "docRefno": "A7856",  "settledByThisCheque": 2000 },   // never touched it
  { "docRefno": "MKL785", "settledByThisCheque": 0    } ]  // the real one
```

## 2. It is the FALLBACK that is wrong, not the route

The same trace with `allocations` sent explicitly is **correct**:

```
after RE-PRESENT       17,000.00   17,000.00   -> back on MKL785  ✓
```

So the shape works. What is wrong is what the route does when `allocations` is
omitted: it auto-allocates FIFO rather than restoring what the bounce reversed.

## 3. Why FIFO is never the right default here

A re-presentation is **the same money for the same debt**. The cheque did not
become a fresh payment when the bank returned it — it is the original
settlement being attempted again. The bills are not a choice the caller is
making; they are a fact the server already holds, in the very rows the bounce
reversed (`abj_reversal_of_id` → the original row's `abj_bill_id` and amount).

**Asked for:** when `allocations` is omitted, restore the allocation to the
bills the bounce reversed, with the amounts it reversed. Keep `allocations` as
the explicit override for the rare case where the operator really is
re-pointing it. If restoring is impossible — a bill closed since, say — refuse
and say which, rather than choosing a different bill.

## 4. The client CANNOT fix this on its own

I looked. After a bounce, `/cheques/get` reports `settledByThisCheque: 0` for
every bill — correctly, since the rows are reversed. The bill is still listed
in `bills[]`, so the client can see WHICH bills for a single-bill cheque, but
**not how much went to each** once a cheque was split across several. Any
client-side reconstruction would be a guess on exactly the documents where a
guess is most expensive.

The per-bill split only exists in the reversed adjustment rows, which are
yours. Hence the ask above rather than a workaround here.

## 5. Everything around it is CORRECT — do not change these

Checked while I was in there, all four behaving exactly as documented:

| state at amend | result |
|---|---|
| cheque HELD | amend allowed; old allocation reversed, new tender's written, cheque soft-deleted and gone from the register and `/cheques/list` |
| cheque DEPOSITED | **409** "Once an instrument has left the drawer…" |
| cheque BOUNCED | **409**, naming BOUNCED |
| cheque CLEARED | **409**, naming CLEARED |

Bounce itself reverses cleanly (17,000 → 15,000 on the right bill). And across
the whole box, 23 of 24 bills reconcile to the paisa between
`abl_alloc_amount` and the sum of their live adjustment rows; the 24th is the
known post-dated cheque, which is correct — a PDC writes its row and does not
settle until it clears.

## 6. Reproduction

```
POST /receipts/create + /post   one cheque 2000, allocated to bill X
POST /cheques/deposit
POST /cheques/bounce            { reason, bankCharge, partyCharge }
POST /cheques/re-present        { apdId, apdAccYear, apdCompanyId, apdBranchId,
                                  bankLedgerId, depositDate, slipNo }   <- no allocations
then: SELECT abj_bill_id FROM accounts.acc_bill_adjustment
      WHERE abj_cheque_id = <apdId> AND NOT abj_is_deleted;
```
The newest row names a bill that is not X.

**Small thing while you are in the file:** `/cheques/bounce` takes `reason`,
`bankCharge`, `partyCharge` — not the `bounceReason` / `bounceCharges` the
column names suggest. Cost me a silent no-op before I checked the DTO. Worth a
line in the swagger description.
