# Notes 135: Two till printouts: the movement slip and the remittance memo (2026-10-17)

**STATUS:** asked.

TILL_DESIGN §9 lists five till printouts. Three are live (notes 117 §7, notes 121): `TILL_SESSION_Z`, `TILL_SESSION_X` and `TILL_DAY_Z`. These two are left:

| Purpose | Prints | Today |
|---|---|---|
| `TILL_MOVEMENT` | the slip for one cash movement: a pickup, top-up, drop, paid in, change, float or hand-over | `movements/get` answers "the slip", but there is no purpose and no print data |
| `TILL_REMIT` | the memo for one remittance, which goes with the cash to the bank or the cash collection agency | nothing |

Please do it the way you did notes 117 §7:
- a purpose row each (module ACCOUNTS, like the other three);
- print **providers** built on the payload the screen already reads, so a print shows what the screen shows;
- the fields listed in `src/modules/till/README.md` ("Printing"), since a provider dataset has no field list in the designer.

Times are `DD-MM-YYYY hh:mm AM` (IST) and dates `DD-MM-YYYY`, as in notes 121.

**Not needed from you:**
- **Amount in words.** The designer's `numToWords` transform does it.
- **Company / branch heading.** We use `company.profile` and `branch.profile`.

## 1 · `TILL_MOVEMENT`: the movement slip

**Document:**
- `docId` = `tcm_id`, `accYear` = `tcm_acc_year`.
- Every kind except `REMIT`, which has its own purpose (§2).
- **Rights:** the same as `movements/get`. Your own session needs Open Till (272) VIEW; another session needs Till Sessions (273) VIEW. `SAFE_TRANSFER` has no session, so it needs Cash Office (287) VIEW.

### `till.movement.header` (one)

| Field | What |
|---|---|
| `doc_no`, `doc_date` | `tcm_doc_no`, `tcm_doc_date` |
| `kind` | `tcm_kind` as stored |
| `kind_text` | the kind in plain words, the slip's title: Float issue · Top-up · Pickup · Cash drop · Hand-over · Paid in · Change · Safe transfer |
| `status`, `voided_on`, `voided_by`, `void_reason` | POSTED / VOIDED. A voided slip prints "VOID" across it |
| `amount` | `tcm_amount` |
| `session_no`, `business_date`, `counter_code`, `counter_name`, `cashier` | the session's; null for `SAFE_TRANSFER` |
| `safe_code`, `safe_name`, `to_safe_code`, `to_safe_name` | the safe; the second only for `SAFE_TRANSFER` |
| `reason` | the reason's text |
| `ledger_name` | the ledger (PAID_IN) |
| `party_name`, `ref_no`, `ref_date`, `notes` | as entered |
| `bag_no`, `seal_no` | DROP |
| `gives`, `receives` | **the two signature lines, by name.** See the table below |
| `witness_verified_on` | when the second person's PIN was checked (`tcm_witness_verified_on`); null = not verified |
| `receipt_status`, `received_by`, `received_on`, `received_amount` | DROP / CLOSE_HANDOVER once a supervisor received the bag (notes 113 / 114), so a reprint after receipt shows it |
| `voucher_no` | the voucher's number (TFlt / TDrp / TPIn), null for EXCHANGE |
| `created_on` | when it was entered |
| `print_count`, `printed_on` | as the Z: please bump `tcm_print_count` once per render in `print-log.service.ts`, as you do `tss_print_count`, so a second print says "REPRINT" |

**Who gives and who receives** (notes 110 roles: `tcm_done_by` = the cashier, `tcm_witness_by` = the supervisor):

| Kind | `gives` | `receives` |
|---|---|---|
| PICKUP | the cashier | the supervisor |
| TOP_UP, FLOAT_ISSUE | the supervisor (or whoever issued the float) | the cashier |
| DROP, CLOSE_HANDOVER | the cashier | `received_by` once received, else null (the line prints blank, for a hand signature) |
| PAID_IN | `party_name` | the cashier |
| EXCHANGE | the cashier | the cashier (`receives` null is fine) |
| SAFE_TRANSFER | the caller, from the first safe | the second safe's holder, or null |

**Old rows:** pickups and top-ups posted **before** notes 110 have the roles the other way round. You noted 4 test rows on 2026-10-13. If you work `gives` / `receives` out on the server, swap them for rows older than the notes 110 migration. Otherwise tell us it doesn't matter, since they are test data.

### `till.movement.lines` (many)

The denomination count of the movement (`tcm_count_id` → `till_count_line`), in face-value order, high to low:

| Field | What |
|---|---|
| `face_value`, `denomination` | the face value, and its name ("₹500", "₹10 coin") |
| `qty`, `amount` | `tcl_qty`, `tcl_amount` |
| `side` | **EXCHANGE only:** `IN` (notes taken into the drawer) or `OUT` (given out). Null for every other kind |

Empty when the movement was entered as an amount without a count.

**Question:** how does an EXCHANGE store its two sides? `tcm_count_id` names one count. If both sides are in that one count, how do we tell IN from OUT?

## 2 · `TILL_REMIT`: the remittance memo

**Document:**
- `docId` = the REMIT's `tcm_id`, `accYear` = `tcm_acc_year`.
- **Rights:** Cash Office (287) VIEW.
- Printed when the remittance is raised (two copies: the bank's or agency's, and the store's), and again after `remit-confirm` (the store's copy, with the bank's figures).

### `till.remit.header` (one)

| Field | What |
|---|---|
| `doc_no`, `doc_date` | `tcm_doc_no`, `tcm_doc_date` |
| `voucher_no` | the CRem number |
| `mode`, `mode_text` | BANK_DEPOSIT / CIT, and "Deposited at the bank" / "Handed to the cash collection agency" |
| `status` | DEPOSITED · IN_TRANSIT · CONFIRMED (and VOIDED if a remit can be voided) |
| `safe_code`, `safe_name` | the safe the cash left |
| `bank_name` | the bank ledger's name (`tcm_ledger_id`) |
| `bank_account_no`, `bank_ifsc`, `bank_branch` | **see the question below** |
| `amount` | `tcm_amount` |
| `bag_no`, `seal_no`, `cit_agency` | as entered |
| `slip_no`, `slip_date` | `tcm_ref_no`, `tcm_ref_date` |
| `done_by` | who raised it |
| `approved_by` | the approver, when REMITTANCE asked for one; null otherwise |
| `handed_to` | free text on the memo is fine: we leave a blank line for the agency's signature |
| `bank_date`, `bank_ref`, `bank_amount`, `difference` | after `remit-confirm`; `difference` = `bank_amount − amount`. Null before |
| `confirmed_by`, `confirmed_on` | after `remit-confirm` |
| `notes`, `created_on` | as entered |
| `print_count`, `printed_on` | bump `tcm_print_count` per render, as §1 |

### `till.remit.lines` (many)

The REMIT count's denominations (`lines` on `movements/remit`), high to low: `face_value`, `denomination`, `qty`, `amount`. A bank's cash deposit slip asks for this breakdown. Empty when no count was sent.

**Question: the bank account number.** A deposit memo needs the account number, and usually the IFSC. No ledger column holds them today (`acc_ledger_master` has no bank-detail fields), and `CHEQUE_DEPOSIT_SLIP` doesn't print one either. Where should they come from?
- **(a)** New ledger fields for a Bank Accounts ledger: account number, IFSC, branch. These would also be useful for the cheque deposit slip.
- **(b)** Leave the three fields null for now. The memo prints a blank line, written by hand.

We are fine with (b) first and (a) later.

## 3 · What we do once it is live

- **Layouts:**
  - `TillMovement80`: 80 mm, kind as title, amount in figures and words, count table, the two named signature lines, REPRINT / VOID marks.
  - `TillRemitA5`: A5, two copies labelled "Bank / agency copy" and "Store copy", count table, bag and seal, a signature line for the agency.

  Both are made by a script beside `till_z_x_templates.py` and published.
- **Qt buttons:**
  - Cash Movement (S3): **Post & print**, as the design says.
  - Session Cockpit Money tab: **Print slip** on the picked movement.
  - Cash Office: **Print memo** after a remit, and on the Remittances tab for a reprint or a confirmed copy.
- **Test:** each kind printed live from the box's own movements (FLOAT_ISSUE, TOP_UP, PICKUP and CLOSE_HANDOVER exist today), and one CIT remit before and after confirm.
