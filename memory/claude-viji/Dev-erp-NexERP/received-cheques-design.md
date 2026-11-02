---
name: received-cheques-design
description: "Received Cheques (menu 51) design rev 1, 2026-09-14: share/cheques/ — acc_pdc_register lifecycle, what deposit/clear/bounce/re-present/replace/return write, decisions C1-C10 all taken, notes (8)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 152e1bb6-b02d-4445-8ee2-a03b779d2771
  modified: 2026-09-14T06:58:27.442Z
---

**Folder:** `/home/vk/Shared/Prathap/cheques/` — README, `received_cheques_ui_mockup.py/.png`
(screen), `received_cheques_tables.py/.png` (table flow, house style of
`openingbal/opening_balance_tables.py`), `received_cheques_flow.md`,
`plan-backend-received-cheques.md`. In `notes.txt` as **(8)**. Builds on
[[receipt-design]] rev 3 and assumes its plan §2. User confirmed C1–C10 on the
two PNGs ("ok") before the docs were written — the PNG-first pattern again.

**Model:** `acc_pdc_register` (R rows) through its own CHECK-constrained
lifecycle HELD → DEPOSITED → CLEARED | BOUNCED → re-present | REPLACED;
HELD → RETURNED | CANCELLED. One register update + one `txn_status_log` row
per transition; one voucher wherever money moves; never edit/delete.
- DEPOSIT: bulk, one slip (bank, date, slip no), NO voucher.
- CLEAR: new type `ChqClr` (CONTRA): DR own bank / CR Cheques In Hand,
  `av_recon_date` = bank date, `avh_src ('ACCOUNTS','PDC',apd_id)` → one
  clearing per cheque (C10). ON_CLEARING rows: the clearing IS the receipt
  (Rct, DR bank / CR party + allocations via the receipt's engine).
- BOUNCE: new type `ChqBnc` (JOURNAL), ONE voucher, five legs (DR party
  amount+charge, CR Cheques In Hand, DR BANK_CHARGES / CR bank, CR
  BOUNCE_CHARGES_RECOVERED — new role); negative abj rows for every
  `abj_cheque_id` row; C4 cascade (reverse adjustments that consumed the
  cheque's ADVANCE, then soft-delete it); party charge = JOURNAL DR bill src
  PDC; `apd_bounce_voucher_id = apd_charge_voucher_id`.
- RE-PRESENT / REPLACE: re-issue voucher Rct DR Cheques In Hand / CR party +
  fresh FIFO allocations (C5) so Clear stays one shape; replace = new HELD
  row + old REPLACED (`apd_replaced_by_id`).
- RETURN/CANCEL: bounce without charges; refused once DEPOSITED.
- Cheques In Hand ledger from the cheque's own tender row
  (`apd_tender_id → td_tender_ledger_id`), never today's master.
- Due buckets computed (FUTURE / DUE_TODAY / OVERDUE / STALE = +3 months);
  strip tie: Cheques In Hand = IN HAND + WITH THE BANK.

**Prerequisites beyond receipt §2:** ChqClr + ChqBnc type rows,
BOUNCE_CHARGES_RECOVERED role + map, settings `accounts.bounce_charge_to_party`
/ `accounts.bounce_reasons`, a grid id, a deposit-slip dataset, menu 51.

**Out by decision:** bank reconciliation (hook `av_recon_date`), Issued
Cheques (52) = mirror, own plan. **Next candidates:** Payment (100) / Issued
Cheques (52), Journal (103), ledger statement view.
