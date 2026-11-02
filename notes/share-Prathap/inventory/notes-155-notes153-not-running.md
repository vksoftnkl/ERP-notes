# notes-155 — notes-153 AND notes-154 code is in the source but not running on .106

Date: 2026-10-26, rechecked twice. From: Viji / Claude. To: Prathap.

**The DB part of notes-153 is live:**
- `svh_load_*`, `svh_unload_*`, crew and vehicle columns;
- `svi_load_rate` / `svi_unload_rate`;
- roles LOADING_CHARGES / LOADING_PAYABLE;
- reasons REPLENISH / REARRANGE / RETURN_STORE;
- voucher type 48 TRF.

**The code is written too.** The source synced at 10:35 today has:
- `save-stock-voucher.dto.ts:431` — the loading / unloading block;
- `opening-stock-lookup.service.ts:138` — `loadRate` / `unloadRate`;
- `stock-adjustment.service.ts:622` — rates on pick-stock;
- `stock-reason.dto.ts:19` — `TRANSFER_REASON_KINDS`;
- `stock-accounts-posting.service.ts:379` — the LOADING_CHARGES / LOADING_PAYABLE voucher.

**But the API answering on https://192.168.0.106:3011 is still the old build.** Each check below was run twice:

| check | expected with notes-153 | what .106 answers |
|---|---|---|
| `POST /stock/transfer` with `header.loadEnabled`, `loadDatetime`, `loadmanIds`, `driverId`, `supervisorId`, `vehicleNo`, `unloadEnabled`, … | saves | **400 VALIDATION_WHITELIST** — "property loadEnabled should not exist" (same for every new header field) |
| `GET /stock/adjustment/pick-stock?…&itemId=<Texttile Item>` | each row has `loadRate`, `unloadRate` | keys absent |
| `GET /stock/reasons?companyId=…&voucherType=TRANSFER_OUT` | REPLENISH, REARRANGE, RETURN_STORE | **400 VALIDATION_IS_IN** — "voucherType must be one of: ADJUSTMENT, ISSUE, DAMAGE, EXPIRY_WRITEOFF, BUCKET_MOVE" |

**Please build and restart the API on .106** (`prisma generate` too, since the schema changed).

After that, I will:
- switch the three Qt screens to send the fields (`kHandlingLive`);
- re-run the live suites, including the posted LOADING voucher and its reversal on cancel;
- report anything left.

## Update, later on 10-26: notes-154 is the same story

The source synced at 11:11–11:14 has notes-154:
- `stock-transfer-receive.controller.ts:132` — `@Get()`, receipt GET;
- `stock-transfer-response.dto.ts:661` — `outVoucherId`;
- `fromBranchName`, `toBranchName`, `toGodownName` in the response DTOs.

None of it answers on .106. Checked LIVE after a fresh despatch, receive and send-back cycle; stock is back where it started:

| notes-154 | expected | what .106 answers |
|---|---|---|
| #1 `GET /stock/transfer/receive?svhId=<posted TRI>&…` | header + lines | **404 "Cannot GET /api/v1/stock/transfer/receive"** — the route is not registered (same for a draft receipt) |
| #2 `GET …/receive/inbound` | `outVoucherId`, `outAccYear`, `fromBranchName`, `fromGodownName` | absent (only `outRefno`) |
| #3 `GET …/receive/prefill` `outVoucher` | `fromBranchName`, `fromGodownName`, `toGodownName` | absent |
| #4 `GET /stock/transfer?svhId=<branch despatch>` header | `toBranchName` | absent |
| notes-153 save with `loadEnabled` (rechecked) | saves | still **400 VALIDATION_WHITELIST** |

**One build + restart of the API on .106 should bring both 153 and 154 live.**
