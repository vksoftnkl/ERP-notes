# notes (48) — cheque details on the tender row → the cheque register

2026-09-25. The Qt tender dialog now captures, for a CHEQUE tender, the four
register fields the bill never sent: **drawer, bank branch, IFSC, MICR**
(`acc_pdc_register.apd_drawer_name / apd_bank_branch / apd_ifsc / apd_micr`).
Today the register row gets drawer = the customer's name and the other three
NULL, because neither `acc_tender_detail` nor `SaveTenderDetailDto` has them.

## Ask

Accept an optional nested object on a sale bill tender row (same pattern as
`tempCredit`), and copy it into the register row notes 46 writes:

```json
"cheque": { "drawerName": "…", "bankBranch": "…", "ifsc": "SBIN0001234", "micr": "600002003" }
```

* all four nullable; `drawerName` ≤ 150, `bankBranch` ≤ 100, `ifsc`
  `^[A-Z]{4}0[A-Z0-9]{6}$`, `micr` `^[0-9]{9}$` (reject a malformed one with
  a 400 naming the field);
* not stored on `acc_tender_detail` — the register is the cheque's home;
* `/bills/get` echoes it back on the tender row from the register row
  (`apd_tender_id = td_id`) so an amend re-sends what was keyed;
* same for the sale-order advance cheque (order-pdc-posting.helper.ts) and
  the receipt, if you want one shape everywhere.

The client sends it only after `kServerHasChequeDetails` (sale_bill_entry.cpp)
is flipped — until then nothing new reaches /bills/create, so no 400.

## VERIFIED 2026-09-25 (after deploy) — client flag flipped on
bil00721: post → register row ZT DRAWER | ZT BRANCH | SBIN0001234 | 600002003;
/bills/get echoes the block (reload shows all four); the client re-sends it on
amend (but see the gap below); cancel → both CANCELLED.

---- ACTION FROM HERE (2026-09-25, second pass) ----

**One gap — /bills/amend drops the block.** bil00721's amend payload carries
`tenders[].cheque` = {ZT DRAWER, ZT BRANCH, SBIN0001234, 600002003}
(audit.audit_log "Bill amended (revision 1 → 2)"), but the register row the
re-post wrote (12:36:28) has drawer = the customer's name and branch / IFSC /
MICR NULL. The create/post path copies it; the amend re-post path does not —
please pass the tender row's `cheque` through there too (and re-tender, if a
new cheque row can come in that way).

## VERIFIED 2026-09-25 (second deploy) — gap closed
bil00723: post → register row with all four; amend → the NEW register row
(12:46:48) also carries ZT DRAWER | ZT BRANCH | SBIN0001234 | 600002003;
cancel → both CANCELLED. Notes 48 is done.
