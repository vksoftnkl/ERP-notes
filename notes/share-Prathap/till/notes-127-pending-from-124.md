# Notes 127: what is still pending from notes 124 (2026-10-16)

**STATUS:** asked. Migration `20261016100000_till_approval_gate_notes_124` closed §1, §5, §6 and §7; we checked each live (see the recheck table at the end of notes 124). This file is **only what is left**, in the order to do it. The detail for each item stays in the notes it names.

## 1 · SECURITY, first: a client can create a SUPER ADMIN (notes 125 §3)

Today any ADMIN can save a user with `usrType = 'SUPER ADMIN'` through `/user-administration/create`, and so get VK Softwares' own powers: all settings, every company, and the implicit HO_FINANCE till grant. The Qt screen hides the role now, but a direct API call still gets through.

**Ask:**
- Refuse `usrType = 'SUPER ADMIN'` on create/update unless the caller is a SUPER ADMIN (403 with a `code`, e.g. `USER_ROLE_RESERVED`).
- The same for editing, deactivating, deleting or resetting the password of an existing SUPER ADMIN user.
- Your call: hide SUPER ADMIN users from a non-super-admin's user list.
- The implicit HO_FINANCE grant for SUPER ADMIN **stays** (user re-confirmed).

## 2 · User Role names = till levels (notes 125 §1, §2, §4)

- Add `STORE MANAGER`, `AREA MANAGER`, `HO FINANCE`, `CASH OFFICE` to `UserType`.
- Rename `MANAGER` → `STORE MANAGER` (VKSOFT, zt-role-4804-manager) and in the seed, then drop `MANAGER`.
- `isHqUserType`: should HO FINANCE count as HQ for chain price saves? Your call; if unsure, leave it out.

Qt switches the User Admin role list once this is live.

## 3 · Supervisor approves their own cash into the safe (notes 124 §11)

`sessions/approve-cash` (`till-movement.service.ts` ~1301) still allows the session's own cashier only under `till.single_operator`, which also skips the PIN.

**Ask:** also allow them when the company's CASH_VARIANCE rule has `tar_allow_self` (or add an APPROVE_CASH rule, as you prefer). **Always with their PIN.** Keep it journalled, so the Cockpit and the day Z can show "approved by own cashier". (`decide-variance` and `approvals/decide` already follow `tar_allow_self`. Nothing to do there.)

## 4 · Receipt surcharge rounding (notes 107 / 124 §10)

Work out a percentage surcharge inside the tender: `surchargeAmt = round2(tdAmount × perc / (100 + perc))`. Then 200 + 1% UPI is **202.00**, not 202.02, the same as on the sale bill. The client works with either formula.

## 5 · Error codes still missing (notes 119 / 124 §9)

- §1: the voucher header refusals and the 2 voucher throws without a `code`.
- §3: `code` required in the shared helpers, so a new refusal can't go out without one.
- §2: the office masters.

## 6 · Force-close wording (notes 117 / 124 §8)

When the session's own cashier tries a force close, the refusal used to say it "does not decide its own variance". It should say the cashier cannot force-close their own session, with its own `code`. Please confirm it's done; we couldn't test it without a live session of the caller.

## 7 · Three answers (no code)

| From | Question |
|---|---|
| 124 §2 / notes 115 §3 | The till events the e2e cleanup deleted (ZT1-261012-01, VIJI PC-261012-01, the 12-10 part of VIJI PC-261012-02, ZT1 / VIJI PC-261014-01 … -03): can they be restored from a backup, or are they lost for good? |
| 124 §3 / notes 122 §6 | Negative safe opening on the day Z: 13-10 froze **−1,426.00** and now 15-10 **−1,346.00**. Print it as it is, flag it ("hand-overs not yet approved"), or count approved hand-overs back to the day they were handed over? |
| 124 §4 / notes 122 §2 | The 500s on "Plain stock" bills, 10-15 between 12:30 and 13:10 IST: what does the server log say was the cause? |

## Not for you (for information)

- **124 §1:** the two closed sessions are now decided from the Qt Cockpit by an approver (VIJI PC-261012-02 +1,900 cash / +50 cheque; ZT1-261012-01 −177). The user picks the treatment.
- **124 §5:** Qt builds the approvers' inbox and request screens on your new routes.
- Plain wording for TILL_APPROVAL_NOT_PENDING / _EXPIRED / _NOT_YOURS / _INVALID and EXPENSE_GST_BILL_DUPLICATE: we draft it, the user approves, then the rows come to you for the catalogue.

Tell Viji when 1–6 are deployed. Claude will recheck each live, as for notes 124.
