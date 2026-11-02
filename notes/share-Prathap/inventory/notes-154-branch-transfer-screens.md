# notes-154 — Branch Outward (192) / Branch Inward (193): what the screens need

Date: 2026-10-26. From: Viji / Claude. To: Prathap.

The Qt screens are built:
- **Branch Outward** is the Godown Transfer screen in branch mode, the same document with `toBranchId`.
- **Branch Inward** is a new screen, driven by a worklist (grid 186).

**LIVE**, run three times on 192.168.0.106:
- despatch 2 pcs Coimbatore → Counter2/karur from the screen;
- receive at Counter2 from the screen, including a damaged split and a draft that is saved and then deleted;
- send the stock back by API.

Every holding ends where it started. The routes work. These are the gaps.

## Fix first

1. **A receipt cannot be read back.**
   - `/stock/transfer/receive` has POST and DELETE but no GET.
   - `GET /stock/transfer?svhId=<a TRANSFER_IN>` → 404 "Stock transfer not found", because it uses the TRANSFER_OUT rules.
   - Result: a saved draft receipt cannot be reopened, the Branch Inward list cannot open a row, and Prev/Next (`STOCK_TRANSFER_RECEIPT`) has nothing to load.
   - Please add `GET /stock/transfer/receive?svhId=&accYear=&companyId=&branchId=`, with the same payload shape as the transfer GET (header + lines). The header should also carry the link: `linkSrcDocId`, `linkSrcAccYear`, and the despatch's refno.
   - The screen already calls this URL. Until it exists, the screen says "cannot open yet".

## Should fix

2. **The inbound worklist rows carry no despatch id.** `GET …/receive/inbound` returns `outRefno` and `fromBranchId`, but not `outVoucherId` / `outAccYear`, and `prefill` needs both. A client cannot go from the worklist to the prefill. Please add `outVoucherId`, `outAccYear`, `fromBranchName`, `fromGodownName`. The Qt screen uses its own grid 186 instead (one row per despatch), so this matters for React and any other client.
3. **Prefill `outVoucher` names nothing.** Please add `fromBranchName`, `fromGodownName`, `toGodownName`.
4. **The transfer GET header has `toBranchId` but no `toBranchName`.** The screen reads the branch master for the name.

## For information only

5. **Cost on the way back.** karur already held 10 pcs of lot 5437-0001 at about ₹212. 2 pcs arrived from Coimbatore at ₹500, and karur's average went to ₹260. Sending 2 back took them out at ₹260, so Coimbatore received them at ₹260, not the ₹500 they left with.
   - This is AVG_COST working as designed: cost per branch, not per lot. It is the case notes-92 (lot cost valuation, server pending) is about.
   - No separate ask. When notes-92 lands, a transfer of a tracked lot should leave at the lot's cost.
6. While my tests ran, device `44c01ccc…` was posting transfers on the same box (TRF/…/44c01ccc…/1–8 from 10:19). Is that your e2e? There's no problem; the tests measure stock relative to their own start.
7. Not exercised live: **settle-short**. The Outward screen has the button. It shows only on an IN_TRANSIT despatch whose rows have all been at least partly received, and it sends `reasonId` = TRANSIT_LOSS, found through `/stock/reasons/list`.

Unloading at the receiving branch (the crew and charge on the TRANSFER_IN) is notes-153 D. The Inward screen has the box and sends nothing until that lands.
