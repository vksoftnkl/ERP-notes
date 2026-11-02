# notes-152 — Previous / Next walks documents in the wrong order (/txn/adjacent)

Date: 2026-10-24. From: Viji / Claude. To: Prathap.

The user reported it on Opening Stock (menu 44):
- Their last entry is OPN0023, a textile opening.
- Ctrl+PgUp/PgDn opens a cancelled voucher instead of the one next to it.

## Cause

The walk key is `(date, slno, created_on, id)` (`txn-adjacent.service.ts` `walkKey`). But `svh_slno` is counted **per device**, while the refno (`OPN00nn`) is counted per branch. So documents from two devices on the same date interleave by each device's own counter.

Coimbatore branch, 2026-10-24, in walk order (date, slno):

| refno | slno | device |
|---|---|---|
| OPN0022 | 4 | 019e4e4c… (the user's PC) |
| OPN0023 | 5 | 019e4e4c… |
| OPN0015 … OPN0021 | 15 … 21 | 01a14a14… (dbg / tests) |
| OPN0024, OPN0025 | 22, 23 | 01a14a14… |

LIVE (`docType=OPENING_STOCK`, Amazon Retail / Coimbatore / 2026-2027):
- `docId=OPN0023, direction=next` → **OPN0015**. Expected OPN0024.
- `docId=OPN0022, direction=prev` → **OPN0014** (dated 10-23). Expected OPN0021.
- No docId, `prev` → OPN0025. That one is right: it is the newest, a cancelled test document.

## Ask

Walk by something that is branch-wide and follows entry order:
- `(date, created_on, id)`, dropping slno. A per-device serial can't order documents across devices, so adding the device to the key wouldn't help.

Every docType in the registry probably has the same problem wherever the serial is per device: stock vouchers, and sales documents if their slno is per device too. Please check each.

## Not a defect

Cancelled documents are part of the walk, which is right: the user can open and inspect one. Our live tests create and cancel documents on the same branch, so a cancelled `ZT-` document can legitimately be the newest.

## STATUS (Prathap fills in)

-
