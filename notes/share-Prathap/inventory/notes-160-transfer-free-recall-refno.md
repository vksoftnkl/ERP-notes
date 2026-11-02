# notes-160 — transfers: free stock, recall a despatch, short voucher numbers

Date: 2026-10-26. From: Viji / Claude. To: Prathap.

The user tested Godown Transfer and Branch Outward and raised three things the server must do. The Qt side is built and waiting on each one.

## A. Free qty must come out of FREE stock

**The problem.** A transfer line takes `freeQty` and the save accepts it, even when the holding has no free stock left. The user keyed Free 1 on a lot that never received any free stock, and it saved.

In the live data, Coimbatore × Texttile Item lot `8568e4ab…` now shows `sbl_free_in_qty 0`, `sbl_free_out_qty 2`. Free has gone out that never came in.

**Asked:**
1. **`GET /stock/adjustment/pick-stock`:** add `freeAvailableQty` to each row: `sbl_free_in_qty − sbl_free_out_qty`, base unit, never below 0.
2. **The engine, on every OUT line** (TRANSFER_OUT and any other type that takes free qty):
   - refuse when `freeBaseQty` is more than the holding's free balance: **422** per line, `lines.<i>.freeQty`, worded.
   - `qty + free ≤ available` stays as it is.

**Qt today:** the Free cell is locked on a holding without free stock. A free qty above the free balance is flagged and blocks the save. The screen reads `freeAvailableQty`; until the server sends it, every holding counts as having no free stock, so Free stays locked. Moving that stock as Qty still works.

## B. Recall a branch despatch that nobody received

The user asked why Branch Outward has no Cancel. Today `/stock/transfer/cancel` refuses IN_TRANSIT, which is right once the other branch has keyed anything. But a lorry that never left, or came straight back, has no way back.

```
POST /stock/transfer/recall   { svhId, accYear, companyId, branchId, reason }
```

- **Allowed only** when the despatch is IN_TRANSIT and **every** transit row is untouched: received 0, damaged 0, no receipt at all against it, a draft one included.
  - Otherwise **409** `STOCK_TRANSFER_RECALL_RECEIVED`, worded: "Counter2 has already keyed a receipt against it."
- **What it does:**
  - reverses the OUT rows and closes the transit rows (the stock is back in the From godown at the same lot and cost, and the receiving branch's inbound list no longer shows it);
  - reverses the loading voucher (notes 156);
  - sets status CANCELLED with the reason and the audit.
- **Rights:** menu 192, CANCEL.
- `/stock/transfer/inbound` must stop listing it at once.

**Qt today:** on an untouched despatch Branch Outward shows **Cancel · Alt+X**, disabled, with the tooltip "Recall is not on the server yet (notes 160)". When this is live I set `GodownTransferEntry::kRecallLive = true`.

## C. Voucher numbers must not carry the machine id

The user asked: "why voucher no saving with uuid".

`resolveDeviceCode()` (stock-voucher-numbering.helper.ts) returns `dev_device_uid`, then the name, then the device uuid. The office PC "NEXERP MACHINE" has `dev_device_uid = 44c01ccc-a162-8314-34b0-1dbf2f5c7bab`, so every stock voucher from it reads:

`TRF/2026-2027/44c01ccc-a162-8314-34b0-1dbf2f5c7bab/16`

Sale bills on the same PC read `bil00017`.

**Asked (every stock voucher type: OPN, ADJ, PHY, TRF, TRI):**
1. A short device code: **`fixed.device_master.dev_code varchar(10)`**, unique per branch.
   - Assign it when a device registers if left blank: `D1`, `D2`, … per branch.
   - Backfill the existing devices the same way.
   - Make it editable in the device master.
2. `resolveDeviceCode()` returns `dev_code`, **never** `dev_device_uid` or the uuid.
3. **The number is zero-padded to 5 digits:** `TRF/2026-2027/D1/00016`. It is still one series per device, so offline tills never clash.
4. **Old documents keep their refno.** Only new ones change. A renumber would break printed slips and links.

Nothing to change on the Qt side. The screens show whatever refno the server gives.

## Done on our side today (for your information)

- **Dropdown 72 "EMPLOYEES BY DESIGNATION".** It takes the params `iemp_branch_id` and `iemp_role`, and matches the designation code or name.
  - While nobody holds a designation it lists every employee.
  - The transfer screens' Loadman, Driver and Supervisor pickers use it. Before, they used dd 38, which lists every employee.
- **New designations:** Loadman (LOADMAN), Driver (DRIVER), Supervisor (SUPERVISOR).
- **Test employees:** ZT Loadman Ravi, ZT Loadman Kumar, ZT Driver Mani and ZT Supervisor Bala, all at the Coimbatore branch.
