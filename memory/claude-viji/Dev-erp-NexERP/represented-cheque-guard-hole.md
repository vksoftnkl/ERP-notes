---
name: represented-cheque-guard-hole
description: "FIXED 2026-09-19 — a re-presented cheque used to escape the amend/cancel guards and leave its bounce voucher DRAFT; this is the regression list"
metadata:
  type: project
---

**OPEN as of 2026-09-19** — `share/cheques/notes-38-represented-cheque-amend-hole.md`.
Found by the user on live `rct00819`, reproduced twice. **It loses money.**

`/receipts/amend` and `/receipts/cancel` both refuse a cheque DEPOSITED or
later, and both resolve "this receipt's cheques" by
`acc_pdc_register.apd_voucher_id`. **`/cheques/re-present` repoints
`apd_voucher_id` to the re-issue voucher** ([[represent-wrong-bill]] — the
notes 37 design). From then on the original receipt looks cheque-free and both
guards wave it through.

```
post cheque 500 -> bill X      apd_voucher_id = rct00821
bounce / re-present            apd_voucher_id = rct00822  ← moves
clear                          CLEARED
amend rct00821 cheque->cash    HTTP 201   ← should be 409
  bill X alloc 2500 -> 3000
  cheque still settling 500 AND the new cash settled 500
```

**500 handed over, 1,000 of debt cleared.** The cheque's settlement is never
reversed because the amend does not believe a cheque exists.

`/cancel` is the same: it cancelled a receipt whose cheque was CLEARED,
leaving a CANCELLED receipt whose cleared cheque still settles a bill through
the re-issue voucher.

**The guards must resolve by what the cheque was TAKEN IN on, not who owns it
now** — an immutable originating column, or walk the re-issue chain. A
re-presentation must never make a receipt newly amendable. Anything else that
asks "which cheques belong to this receipt?" needs the same treatment.

**Never a client bug** — the screen only offers Correct / Cancel; the server
decides. `rct00819` was cancelled at the user's request, releasing bill 135.

**The four pre-fix DRAFT bounce vouchers were NOT backfilled** (`chqbnc00017`,
`00061`, `00062`, `00075`). Three carry live bill rows with unstamped totals,
so they will read as a party credited twice the day the ledger statement
ships. Flagged in notes (39) for a one-off correction.

## Its sibling: re-present also leaves the BOUNCE voucher in DRAFT

`share/cheques/notes-39-represent-leaves-bounce-draft.md`, also OPEN. Three
re-presented cheques on the box, three DRAFT bounce vouchers, no exceptions —
every other bounce is POSTED or CANCELLED. Their `avh_total_debit/credit` are
0.00 (never stamped) though the legs are there and balance, and the bounce's
bill adjustment row stays LIVE.

So the BILL is right and the LEDGER is not: counting POSTED only, cheque NQ123
leaves **Cheques in Hand at +2,000 for a cheque that has cleared** and the
party **credited twice**. The bounce is the entry that re-debits the party and
relieves Cheques in Hand, and it is the one sitting in DRAFT.

**Latent today** — no grid reads `acc_vouchers`, the ledger statement and
trial balance are not built — so it has never shown up. It surfaces the day
they are.

**Two things there that are NOT defects**, checked so nobody re-chases them:
- a POSTED bounce with party CR / Cheques-in-Hand DR is the REVERSAL voucher a
  cancellation writes, 0.3s after the original goes CANCELLED
- `chqbnc00037-40` (POSTED, 18,500, zero legs, no cheque) are one 13:46:20
  test-harness batch, not a product path
