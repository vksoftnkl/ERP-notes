# notes-156 — labour charges on a branch transfer (the rest of notes-153 D)

Date: 2026-10-26. From: Viji / Claude. To: Prathap.

**notes-153 and notes-154 are LIVE and verified** after the restart:
- **Godown transfer** with Loading 2 × 1.50:
  - TRF voucher 3.00, DR Loading & unloading charges / CR Loading charges payable;
  - the charge and the vehicle are stored, and come back on the GET;
  - cancel reverses the voucher.
- An earlier API check: (2 + 1 free) × 1.50 loading + × 0.50 unloading = 4.50 + 1.50 → voucher 6.00, reversed on cancel (rev00222).
- Receipt GET, inbound `outVoucherId`, prefill names, `toBranchName` and line batch/MRP all work.
- Both Qt suites pass.

**One thing still open.** A branch transfer refuses a labour rate:

> `422 STOCK_LOADING_CHARGE_INTER_BRANCH` — "a loading charge on a branch-to-branch transfer does not post to accounts yet — only a godown transfer's does."

The Qt screens follow that for now:
- Branch Outward records the loading crew and vehicle, with no rate column and no charge.
- Branch Inward records the unloading crew, with no rate.

**The user's decision (10-26) is that the charges post to accounts**, so please finish the branch half as notes-153 D described:
- **Loading** posts at **despatch**, at the **sending** branch: DR LOADING_CHARGES / CR LOADING_PAYABLE, mapped for that branch. Charge = Σ (qty + free) × `loadRate` on the TRANSFER_OUT lines.
- **Unloading** posts when the **TRANSFER_IN posts**, at the **receiving** branch, from the receipt's own `unloadRate` lines. Charge = Σ (qty + free) × `unloadRate` of what was received (a DAMAGED line counts; the goods were still carried).
- A despatch is never cancelled, so its loading voucher has no reversal path. A short-settle does not touch it: the labour was done.
- The `LOADING_WRONG_HALF` rule stays: loading on the OUT, unloading on the IN.

When this is live, tell me. The screens show the rate columns again in branch mode, and the suite checks both vouchers.
