---
name: loaded-receipt-allocations
description: "/receipts/get allocations[] is the voucher's adjustment HISTORY — net it per bill and route by adjType, or a corrected receipt paints the same bill three times"
metadata:
  type: project
---

**`/receipts/get` → `allocations[]` is the voucher's HISTORY, not its current
state.** Because `acc_bill_adjustment` retracts by negative counter-row
([[bill-adjustment-reverses]]), an amended receipt comes back holding the
superseded allocation, its reversal AND the replacement. Painted a row at a
time that is the same bill three times with a **−2,000 in an editable Receive
cell** and After reading −2,000. Found from a screenshot 2026-09-19, fixed in
`549a6ed` + `b7915f6`.

**`netAllocations()` in `receipt_entity.h`** is the fix — pure, testable with
no server and no widgets, and `ReceiptEntry::applyLoadedAllocations()` is now
one line. Summing per bill is **exact, not a heuristic**: a reversal is by
`ck_abj_reversal_sign` the precise negative of what it retracts, so nothing has
to be paired up and two legitimate equal allocations cannot be misread as one.
A bill a correction emptied nets to zero and is dropped.

**`adjType` decides the COLUMN, and the payload has no discount/writeoff
fields.** A settlement, its discount, its write-off and its round-off are FOUR
ROWS against ONE bill (`ALLOCATION` / `DISCOUNT` / `WRITEOFF` / `ROUND_OFF`;
`ADVANCE_ADJUST`, `NOTE_ADJUST`, `TRANSFER` are settlements too). The old code
read `"discount"` and `"writeoff"` keys that have never existed — so they were
permanently 0.00 *and* the discount became its own grid line.

**A flag nobody fetched is not a fact.** `ReceiptParty` default-constructs
`isBillByBill = false`, and a POSTED receipt being read never calls
`/open-items` — it paints from its own allocations. So the grid hint announced
"this party is NOT bill-by-bill" over a screen full of that party's bills.
`ReceiptParty::loaded` (set only by `fromJson`) now separates the two, and
`m_party` is reset per document so a loaded receipt cannot inherit the previous
one's party. **Watch for this shape elsewhere** — any default-false flag read
as a fact before its fetch.

**The bill figures LANDED the same day** (`b7915f6`). `allocations[]` now also
carries `billAmount`, `pendingAmount`, `dueDate`, `billType`, `status`,
`reversalOfId`, `isReversed` — verified against `acc_bill_balance`.

**`pendingAmount` is served as it stands NOW and the row must show it as it
STOOD.** Painted straight through, After reads 13,500 for a bill sitting at
14,500. So this receipt's settlement is added back before painting — the same
thing `mergeAmendAllocations()` already did to the `/open-items` list.

**ONLY THE MATURED PART IS ADDED BACK — this is the subtle one.** A post-dated
cheque writes its row and does NOT move `abl_pending_amount`
([[bill-adjustment-reverses]]), so there is nothing to undo. `rct00018` holds
800 post-dated against a bill of 7,500 showing 5,380; a blind add-back paints
Pending **6,180**, a figure that bill has never carried. The held amount goes
to the PDC held column instead. **`matured` on the payload is load-bearing, not
decoration.**

**`isReversed` is deliberately NOT what the netting runs on.** Nothing in the
schema forces a reversal to be the WHOLE of what it retracts —
`ck_abj_reversal_sign` only fixes the sign, `ux_abj_reversal` only caps it at
one per original — so filtering on the flag would lose a *partial* reversal's
residual. Summing survives any shape. Asked Prathap whether a partial reversal
is possible at all; if it is not, this can simplify.

**Still open:** a cancel creates a **reversal voucher in the receipt series** —
cancelling `rct00615` produced `rct00617`, POSTED, no tenders, legs the exact
mirror, three negative allocations. It lists and opens as an ordinary receipt.
Asked whether it should carry a marker so the register can label it and the
entry screen open it read-only.
