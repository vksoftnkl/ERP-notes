---
name: transfers-notes-160
description: "Notes 160 built 2026-10-27 — free qty only from free stock (save + engine), POST /stock/transfer/recall (menu 192 CANCEL), dev_code in stock voucher numbers (ONE series across all devices, never reused)"
metadata:
  node_type: memory
  type: project
  originSessionId: 9e5de5d7-7b44-4f6c-8cfe-9ba61cec4ba9
  modified: 2026-10-27T05:02:22.636Z
---

Migration `20261027110000_transfers_notes_160` (applied on .106; seeds Menu_Master / Grid_Details / Grid_Columns match).

- **A:** `pick-stock` → `freeAvailableQty`. The transfer save and the engine (`assertFreeStock`, after `applyLedgerRows`) refuse 422 `STOCK_FREE_QTY_EXCEEDS_FREE_STOCK` at `lines.<i>.freeQty`. Sales shadows (`ledgerSource`) are exempt: scheme free goods come off the shelf. Adjustments already refuse free outright.
  - No holding at the test branch has free stock, so e2e suites must not send freeQty on transfers. The notes 153 and 156 suites were changed to 3 qty with no free (same labour totals).
  - A branch despatch's free travels as plain qty (`stt_sent_qty` = base + free).
- **B:** `recallTransferOut` → `StockPostingService.recall` → `StockVoucherService.recall`. Judged on menu 192 CANCEL; the migration added the verb but granted it to nobody. A receipt of any status except CANCELLED blocks it, drafts included.
- **C:** the device code is unique across ALL devices, not per branch as notes 160 asked.
  - **Why:** `ux_svh_refno` is per document branch, and the office PC (dev_branch_id NULL) posts in 3 branches.
  - Codes are never reused (`ux_dev_code` covers deleted rows).
  - `resolveDeviceCode` lazily assigns a code to a device that has none.
  - Serials are padded to 5 digits. OPN and PHY use the accounts series and are unchanged.
  - .106: NEXERP MACHINE = D1 … dbg (debug-void-dev) = D14.

**How to apply:** don't narrow dev_code uniqueness to per branch. Recall rights must be granted on 192 in User Administration. Proof: `test/transfers-notes-160.e2e-spec.ts` (live, value-neutral). See [[notes-153-godown-transfer-loading]].
