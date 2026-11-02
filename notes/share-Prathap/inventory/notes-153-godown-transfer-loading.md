# notes-153 — Godown Transfer (menu 41): loading / unloading, and posting the charges

Date: 2026-10-26. From: Viji / Claude. To: Prathap.

The Qt Godown Transfer screen is being built now. It follows `godown_transfer_entry_ui_mockup.png`, the September mockup in the share root. It uses the routes you already built (`/stock/transfer`, same-branch shape).

**LIVE check 10-26.** One full cycle on 192.168.0.106 as VKPOS, device debug-void-dev:
- Save: TRF/2026-2027/debug-void-dev/1, DRAFT.
- Validate: clean.
- Despatch: 2 ledger rows, POSTED. 2 pcs moved Coimbatore → namakkal.
- Cancel: 2 reversal rows, stock back.

The routes are fine. **Nothing in this note is a defect.** It asks for one new feature.

## What is missing

3.0's Stock Transfer had a **Loading** box and an **Unloading** box (`public.stock_transfer`: `enable_loading`, `load_loadmanid[]`, `load_driverid[]`, `load_vehicleid[]`, `load_supervisorid[]`, `loading_charge`, `enable_unloading`, `unload_date`, `unload_loadmanid[]`, `unload_supervisorid[]`, `unloading_charge`). Each line carried a per-unit rate (`cooly` = the unit's `unit_loading`, only when the item had `add_cooly`) and its amount (`cooly_amt` = qty × rate). The header charge was the sum of those amounts.

`stock.stock_voucher` has none of this.

**The user's decision (10-26): the charges POST TO ACCOUNTS.** In 3.0 they were only recorded.

## A. Columns

Crew and vehicle go **on the document**, the same rule as the sales documents (`sb_driver_id`, `sb_loadman_id uuid[]`, …; see `33_txn_transport_detail.sql`). No new table, because nothing existing can hold these facts.

`stock.stock_voucher`:

| column | type | note |
|---|---|---|
| `svh_load_enabled` | boolean not null default false | the Loading tick |
| `svh_load_datetime` | timestamptz | |
| `svh_loadman_id` | uuid[] | employee_master |
| `svh_driver_id` | uuid | employee_master |
| `svh_supervisor_id` | uuid | employee_master |
| `svh_vehicle_id` | uuid | vehicle_master, optional |
| `svh_vehicle_no` | varchar(30) | typed, as on the bill |
| `svh_loading_charge` | numeric(18,2) not null default 0 | **computed, never from the body** |
| `svh_unload_enabled` | boolean not null default false | the Unloading tick |
| `svh_unload_datetime` | timestamptz | |
| `svh_unloadman_id` | uuid[] | employee_master |
| `svh_unload_supervisor_id` | uuid | employee_master |
| `svh_unloading_charge` | numeric(18,2) not null default 0 | **computed** |

`stock.stock_voucher_item`:

| column | type | note |
|---|---|---|
| `svi_load_rate` | numeric(18,4) not null default 0 | per unit of the line's unit; editable on the screen |
| `svi_unload_rate` | numeric(18,4) not null default 0 | same |

**Why the rate is stored on the line:** a later change to the unit's rate must not change an old transfer's charge.

The amount is not stored. It is qty × rate.

## B. The save

- DTO header: `loadEnabled, loadDatetime, loadmanIds[], driverId, supervisorId, vehicleId, vehicleNo, unloadEnabled, unloadDatetime, unloadmanIds[], unloadSupervisorId`.
- DTO line: `loadRate`, `unloadRate`.
- The GET returns all of these, plus `loadmanNames[]`, `driverName`, `supervisorName`, `unloadmanNames[]`, `unloadSupervisorName`, `loadingCharge`, `unloadingCharge`.
- **Server computes:**
  - `svh_loading_charge = round2(Σ qty × svi_load_rate)` when `load_enabled`, else 0;
  - `svh_unloading_charge` the same way from `svi_unload_rate`.
  - Never take a total from the body (same ask as notes-143 #3 / 151 #3).
- Refuse (422 per field):
  - a negative rate;
  - an employee id from another company;
  - a load or unload datetime earlier than the doc date.
- **Only on TRANSFER_OUT / TRANSFER_IN.** Other stock types must refuse these fields, or ignore them and store false/0.

## C. Default rates for the screen

`inventory.item_unit_master.unit_loading` / `unit_unloading` already exist, and so does `item_master.item_allow_loading` (3.0's `add_cooly`).

Please return, in `/stock/opening/item-lookup` (each unit row) and in `/stock/adjustment/pick-stock` (each holding, for its unit):

- `loadRate`: `unit_loading` when `item_allow_loading`, else 0;
- `unloadRate`: `unit_unloading` when `item_allow_loading`, else 0.

The screen pre-fills the line with these values, and the user can change them.

## D. Posting the charges

On **despatch of a same-branch transfer** (it ends POSTED), when `loading + unloading > 0`, write ONE accounts voucher:

```
DR  LOADING_CHARGES   loading + unloading
CR  LOADING_PAYABLE   loading + unloading
```

- **Two new roles** in `accounts.acc_ledger_role`. Both are SHARED, `alr_by_supply=false`, `alr_by_rate=false`.
  - `LOADING_CHARGES`: label "Loading & unloading charges", type EXPENSE, nature Expenses (direct expense).
  - `LOADING_PAYABLE`: label "Loading charges payable", nature Liabilities (current liability).
- Map them per company in `acc_ledger_map`. The Ledger Map screen (menu 250) shows new roles by itself.
  - If a role is unmapped and the charge is > 0: 422 with a worded error (role name + "map it in Ledger Map").
  - Do not skip the entry silently.
- **The labour is paid later** with a normal Payment (menu 100) against the payable ledger. That is why the credit side is a payable and not cash.
- **Independent of `cogs_mode`.** This is an expense, not a stock value. It posts under PERIODIC too, unlike `postForVoucher`'s inventory legs.
- **Voucher type:** there is no accounts voucher type for transfers (`acc_voucher_types` has OPENING 1, Phy 6, StkAdj 32). Please add `TRF` "Stock Transfer", category BOTH, `affects_accounts=true`, `vchr_menu_id=41`. Or use the Journal type, as settle-short does. Your choice; tell me which, so the Voucher Register shows it right.
- Link: `avh_src_module='STOCK'`, `avh_src_doc_type='TRANSFER_OUT'`, `avh_src_doc_id=svh_id`, the same as the other stock vouchers.
- **Cancel** of a same-branch transfer reverses it (`reverseForVoucher`, dated the original).
- **Inter-branch, for later** (Branch Outward 192 / Inward 193):
  - loading posts at despatch, at the sending branch;
  - unloading posts when the TRANSFER_IN posts, at the receiving branch, from that receipt's own unload fields.
  - The OUT's unload fields stay empty then.

## E. Small things seen on the way

1. The list (`GET /stock/transfer`) shows a CANCELLED transfer with `totalQty 0`, `totalValue 0`. The other stock lists keep the original figures on a cancelled document. Please keep them here too, because the list's status chip already says CANCELLED.
2. Menu 41 "Stock Transfer" is `menu_visiblity=false` with CTRL+T. I will make it visible for the screen and rename it "Godown Transfer", unless you object. Branch Outward 192 / Inward 193 already exist for the other two screens.
3. `/stock/transfer` has no menu-41 rights check. It is the same as notes-151 #1. Please add it when you add the fields.
4. **There are no transfer reasons.** The mockup's header has a Reason ("Replenish cold room"). But `stock_reason_master` has no row whose `srm_allowed_txn_types` holds TRANSFER_OUT, except TRANSIT_LOSS, which is settle-short's. Also, `GET /stock/reasons?voucherType=` accepts only the adjustment kinds. Please:
   - seed three shared reasons with direction BOTH and `{TRANSFER_OUT,TRANSFER_IN}`:
     - REPLENISH "Replenish godown";
     - REARRANGE "Re-arrange stock";
     - RETURN_STORE "Back to main store";
   - let `/stock/reasons` take `voucherType=TRANSFER_OUT`, without listing TRANSIT_LOSS there.
   - Until then, the screen shows the Reason box disabled.
5. **`GET /stock/transfer?svhId=` lines carry no lot identity (LIVE).**
   - TRF/2026-2027/debug-void-dev/1 line 1 moved lot `8568e4ab…`, whose batch is 5437-0001 and MRP 550. The GET line answers `batchNo: null, expiryDate: null, mrp: null`, because those columns are for keyed identity and a transfer line names a lot instead.
   - Please fill `batchNo` / `expiryDate` / `mrp` / `serialNo` on a transfer line from its lot (`stock_lot`), as pick-stock does, so a reopened transfer shows which batch moved.

Also, the labour charge counts what was carried: `(qty + freeQty) × rate`, not qty alone. Please use that in B.

## Order

A + B + C first; the screen can save crew and rates once they exist. D can follow. Until D is live, the screen shows the charges but posts nothing.
