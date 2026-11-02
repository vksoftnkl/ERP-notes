# NestJS — Stock Transfers (godown → godown, branch → branch)

Target repo: `/home/vk/Projects/ERP/ERP-Server` (NestJS + Prisma, PostgreSQL 18).
Schema it is written against: `schema/stock/16_stock.sql` + `19_stock_posting.sql` +
`20_stock_transfer.sql`, read queries `16q_stock_grid_queries.sql` **Q14**, worked example
`schema/stock/20_stock_transfer_flow.md`.

Read `plan-nestjs-opening-stock.md` first. Everything there about Prisma models, the shared
`stock-voucher` service, device numbering, the SQLSTATE trap and the envelope applies
unchanged; this plan says only what is **different** for transfers.

## Context

**There is no transfer table.** Both forms are `stock.stock_voucher` rows —
`svh_voucher_type = 'TRANSFER_OUT'` and `'TRANSFER_IN'` — with their lines, plus
`stock.stock_transit` for the inter-branch leg. The API writes **two tables and calls one
function**, twice in the inter-branch case.

**Forms 3 and 4 are one endpoint set, two screens.** The engine chooses the shape itself:

```
v_same := svh_to_branch_id IS NULL OR svh_to_branch_id = svh_branch_id
```

Do **not** build a `godown-transfer` module and a `branch-transfer` module. The document,
the lines, the validation and the despatch call are identical; only `toBranchId` and the
screen differ. Two modules means the second one drifts, and the drift lands in the ledger.

**What the engine already does, so the API must not:** stamp the cost (`fn_sml_cost_default`
on the OUT row, then it travels), create or reuse the lot (never re-resolved — the lot is
supplied), maintain `stock_balance` / `sbl_transit_in_qty` / `stock_item_cost` (triggers),
settle the transit rows, and decide when the OUT is closed.

---

## 0. Blockers to settle first

### 0.1 The schema is not on the live database

Unchanged from the opening plan §0.1, plus one line: `20_stock_transfer.sql` must be loaded
**after** `19_stock_posting.sql`. Verify with:

```sql
SELECT proname FROM pg_proc p JOIN pg_namespace n ON n.oid = p.pronamespace
 WHERE n.nspname = 'stock' AND proname IN
       ('fn_svh_post_transfer','fn_svh_receive_transfer','fn_svh_transfer_cancel_guard');
-- expect 3 rows
```

### 0.2 `stock_transit` needs a Prisma model, and it is not partitioned

Unlike `stock_voucher` / `stock_voucher_item`, `stock_transit` has a **single-column PK**
(`stt_id`) and no `PARTITION BY` — it is a live worklist, not a yearly ledger. `stt_short_qty`
is **GENERATED** (`sent − received − damage`) — map it read-only and keep it out of every
write payload, like `svi_value`. Model, no migration, same rule as everything else in `stock`.

### 0.3 The despatch screen has nowhere to put the lorry

`stock_transit` carries `stt_lr_no`, `stt_vehicle_no` and `stt_expected_on`.
`fn_svh_post_transfer` **never sets them** — nothing in the engine writes those three
columns. Decide before building the screen:

- **(a)** the API updates the transit rows immediately after `fn_svh_post_transfer` returns,
  inside the same transaction, keyed by `stt_out_voucher_id`; or
- **(b)** the function grows three parameters.

(a) needs no schema change and is the recommendation; whichever is chosen, say so in the
DTO, because a despatch note without a vehicle number is not a despatch note.

### 0.4 Six engine defects this module must code around

`REVIEW_2026-09-05.md` reproduced them; four are on this path. Until they are fixed in `20` /
`16`, the API is the only thing standing between them and the ledger:

| Defect | What the API must do |
|---|---|
| **Cost bypass** (MUST-FIX 5): a line with a nonzero `svi_cost_rate` skips `fn_sml_cost_default`, and 20's OUT insert omits the value columns — reproduced as an OUT ledger row at rate 555, **value 0** | **Zero `svi_cost_rate` and `svi_cost_rate_wot` on every TRANSFER_OUT line before insert.** Not "don't send one" — strip it. The policy cost is the only correct answer here |
| **`ux_stt_out_line`** (MUST-FIX 3): same item+lot from two source godowns on one voucher collides — raw `23505`, no message | Refuse at save with a worded error (§3.2) |
| **Post-lock holes** (MUST-FIX 2): IN_TRANSIT/RECEIVED are not frozen; a plain UPDATE regresses an OUT to DRAFT and a second despatch moves the stock twice; receive never checks the OUT's `svh_is_deleted` | **Never UPDATE `svh_status` from the API.** Only the two functions may write it. Do not offer a "reopen" action |
| **Receive bucket laundering** (SHOULD-FIX): a SALEABLE line against a DAMAGED transit row posts damaged goods in clean | Refuse `line.bucket <> stt_bucket` unless the line is DAMAGED (§7.2) |
| **Cancel guard over-reaches** (SHOULD-FIX, 20:598): it refuses cancelling *any* linked TRANSFER_IN, including a DRAFT one — and the link is mandatory on all of them | Drafts are **deleted**, never cancelled (§9). Do not expose Cancel on a draft receipt |
| **The short never settles** (top of the gap list): no `fn_stt_settle_short`, so a partial transit row stays PARTIAL for ever and keeps its remainder in the destination's `sbl_transit_in_qty` | Either add the settle endpoint here (UPDATE the transit row to RECEIVED, keeping `stt_short_qty`, which Q14 still reports) or accept that the inbound worklist accumulates ghosts. Decide with the DB owner |

---

## 1. Module shape

```
src/modules/stocks/stock-voucher/                <- already exists (opening plan §2)
src/modules/stocks/stock-transfer/
  stock-transfer.module.ts
  stock-transfer.controller.ts                   <- TRANSFER_OUT: save, list, one, despatch, delete
  stock-transfer-receive.controller.ts           <- TRANSFER_IN: inbound, prefill, save, receive
  dto/save-stock-transfer.dto.ts                 <- pins TRANSFER_OUT
  dto/save-stock-transfer-receive.dto.ts         <- pins TRANSFER_IN, requires the link
  dto/list-stock-transfer-query.dto.ts
```

`StockVoucherService` is imported, not copied. The two type-specific bits — which post
function to call, and which refusals to run first — go in the per-type rule record the
shared service already takes.

**The type is never in the payload**, same as opening: the route pins it, `@IsIn` enforces it.
A TRANSFER_OUT posted through `fn_svh_post` (the generic one) is refused by name — 19 will
not post a transfer, deliberately — so a mis-routed payload fails loudly rather than half-posting.

---

## 2. Numbering

Same helper, same advisory lock, same per-device serial. Two prefixes to agree with the
client (**open item** — nothing in the schema defines them):

```
TRF/{accYear}/{deviceCode}/{slno}     TRANSFER_OUT
TRI/{accYear}/{deviceCode}/{slno}     TRANSFER_IN
```

`svh_slno` is unique per `(company, branch, acc_year, voucher_type, device)`, so OUT and IN
number independently and the receiving branch's tablet numbers its own receipts offline.

---

## 3. `POST /api/v1/stock/transfer` — save a TRANSFER_OUT draft

### 3.1 Payload

```ts
{
  header: {
    svhId?, accYear, companyId, branchId, tenantId?,
    deviceId, sessionId?,
    slno?, refno?, usrRefno?,
    docDate,
    fromGodownId,                 // REQUIRED
    toGodownId,                   // REQUIRED
    toBranchId?,                  // absent / same as branchId = godown-to-godown (form 3)
    reasonId?, remarks?, userId,
    lrNo?, vehicleNo?, expectedOn?   // → stock_transit, see §0.3. Inter-branch only
  },
  lines: [{
    lineNo, splitNo?,
    itemId, uomId, baseUomId,     // iuc_id, not unit_id
    godownId,                     // the SOURCE godown of this line
    lotId,                        // REQUIRED — picked from stock_balance
    bucket?,                      // defaults SALEABLE
    qty, freeQty?, weightQty?,
    remarks?
  }]
}
```

**`lotId` is the difference from every other document.** Opening resolves a lot; a transfer
moves one. The grid is loaded from the balance (item × godown × lot × bucket × on-hand), so
the client already has it — reject a line without one at the DTO, not at the engine.

**Never send a cost — and strip it if it arrives.** `svi_cost_rate` left at 0 lets
`fn_sml_cost_default` stamp the policy cost, which is then copied to the IN row and the transit
row. A nonzero rate makes the trigger bail (it only fills gaps) *and* 20's OUT insert omits the
value columns, so the ledger row lands at that rate with **value 0** — reproduced in the
review. The DTO must zero it, not merely omit it from the docs.

### 3.2 Refuse before the engine does (422, per-line list)

- `fromGodownId == toGodownId` when it is a same-branch transfer.
- any `line.godownId == toGodownId` — moving stock from the destination to itself.
- `lotId` missing.
- `qty + freeQty == 0`.
- **quantity exceeds what is on hand** for that (godown, item, lot, bucket). The engine does
  *not* check this — `stp_allow_negative` decides, and under `ALLOW` a transfer will happily
  send stock the branch does not have. The screen picked from the balance, so it knows.
- duplicate `(lineNo, splitNo)`.
- **more than one bucket of the same (item, lot) to the same destination godown** on one
  inter-branch voucher. It saves and despatches fine, then the receipt cannot post
  (`0A000`, *"shipped in more than one bucket"*). Catch it at save, where it is one query.
- **the same (item, lot, bucket) from two source godowns** on one inter-branch voucher. The
  destination godown is header-level, so both lines produce the same `ux_stt_out_line` key and
  the despatch dies with a bare `23505` (MUST-FIX 3). Two source godowns = two vouchers, until
  the engine aggregates the transit row.

### 3.3 Computed server-side

Exactly as the opening plan §4.3 — `to_base_factor` from `item_unit_conversion`, base
quantities, header identity copied to the lines, status always `DRAFT`, totals never — with
one addition: **`svi_lot_id` comes from the payload here**, and must be checked to belong to
the same company and item (`slt_company_id`, `slt_item_id`).

---

## 4. `POST /api/v1/stock/transfer/despatch`

```ts
await tx.$queryRaw`SELECT stock.fn_svh_post_transfer(${svhId}::uuid, ${accYear}::bpchar, ${userId}::uuid) AS rows`;
// then, if inter-branch and §0.3(a) was chosen:
await tx.stock_transit.updateMany({ where: { stt_out_voucher_id: svhId, stt_out_acc_year: accYear },
                                    data: { stt_lr_no, stt_vehicle_no, stt_expected_on } });
```

One call, one transaction. The response must tell the screen **which shape happened**,
because the two forms end differently:

```json
{ "success": true, "message": "Transfer despatched",
  "data": { "svhId": "…", "refno": "TRF/2026-2027/TILL-01/2",
            "status": "IN_TRANSIT", "sameBranch": false, "ledgerRows": 1, "transitRows": 1 } }
```

`status` is `POSTED` for a same-branch pair and `IN_TRANSIT` for a despatch. Return it from
the row, never from the request — the engine decides.

---

## 5. Lists and loads

- **Outbound list** — `stock_voucher` where `svh_voucher_type = 'TRANSFER_OUT'`, scoped by
  company + branch + acc_year, ordered `svh_doc_date DESC, svh_slno DESC` (`ix_svh_list`).
  **The status filter must offer DRAFT, POSTED, IN_TRANSIT, RECEIVED and CANCELLED** — an
  inter-branch OUT is never POSTED, so a list hard-coded to POSTED loses every transfer in
  flight.
- **Inbound worklist** (form 4's landing screen) — `ix_svh_inbound`:
  `svh_to_branch_id = :me AND svh_status = 'IN_TRANSIT'` (a fully received transfer is
  `RECEIVED`; `PARTIAL` is a `stt_status`, never a voucher status) — plus **Q14** from
  `16q`, which is the same question asked of `stock_transit` and additionally reports
  days-in-flight, short quantity and loss value. Use Q14 for the screen; the index for the
  badge count.
- **One voucher** — header + lines, and for an inter-branch OUT its transit rows, so the
  sender can see what has been received against each line.

---

## 6. `GET /api/v1/stock/transfer/receive/prefill?outVoucherId=&accYear=`

The receipt screen is **not** a copy of the despatch. What is still owed lives in
`stock_transit`, not in the OUT's lines:

```sql
SELECT stt_id, stt_item_id, stt_lot_id, stt_to_godown_id, stt_bucket,
       stt_sent_qty, stt_received_qty, stt_damage_qty,
       stt_sent_qty - stt_received_qty - stt_damage_qty AS remaining_qty,
       stt_cost_rate, stt_base_uom_id
  FROM stock.stock_transit
 WHERE stt_out_voucher_id = :out_id AND stt_out_acc_year = :acc_year
   AND stt_is_deleted = false
   AND stt_sent_qty - stt_received_qty - stt_damage_qty > 0
 ORDER BY stt_item_id;
```

A second, partial receipt must open with **the remainder**, not the original quantities.
Prefilling from the OUT's lines is the bug that lets a clerk receive 30 twice.

---

## 7. `POST /api/v1/stock/transfer/receive` — save a TRANSFER_IN draft

### 7.1 Payload

Same shape as §3.1, with four differences the DTO must enforce:

| Field | Rule |
|---|---|
| `linkSrcDocId`, `linkSrcAccYear` | **required.** With `module='STOCK'`, `docType='STOCK_VOUCHER'` — the function checks all three and `ck_svh_transfer_in_link` checks the id |
| `branchId` | must equal the OUT's `svh_to_branch_id`; receiving another branch's transfer is refused |
| `line.godownId` | the **destination** godown — the opposite meaning to the OUT line. Take it from the prefill's `stt_to_godown_id`, never from a picker |
| `line.bucket` | `SALEABLE` for what arrived good, `DAMAGED` for what arrived broken. Damaged units still post IN — they exist, broken — and count as `stt_damage_qty` |

`fromGodownId` / `toGodownId` are still required on the header (`ck_svh_transfer_godowns`
applies to TRANSFER_IN too); copy them from the OUT.

**What never arrived gets no line.** A short is what remains unkeyed — do not invent a
zero-quantity line for it, and do not offer a "short" field.

### 7.2 Refuse before the engine

- a line whose `qty` exceeds `remaining_qty` for its transit row (engine: `23514`, and the
  message names the figures — but the screen should never let it be typed);
- a line whose (item, lot, godown) matches no open transit row of that OUT;
- a line whose `bucket` differs from the transit row's, unless the line is `DAMAGED` —
  otherwise a damaged consignment received as SALEABLE is laundered clean;
- the OUT not being `IN_TRANSIT`;
- an empty line set.

---

## 8. `POST /api/v1/stock/transfer/receive/post`

```ts
await tx.$queryRaw`SELECT stock.fn_svh_receive_transfer(${svhId}::uuid, ${accYear}::bpchar, ${userId}::uuid) AS rows`;
```

The response must carry **both** documents' new state, because the receipt closing does not
mean the transfer closed:

```json
{ "data": { "inVoucher": { "svhId": "…", "status": "POSTED", "ledgerRows": 2 },
            "outVoucher": { "svhId": "…", "status": "IN_TRANSIT" },
            "transit": [{ "itemId": "…", "sent": 30, "received": 25, "damaged": 3, "short": 2 }] } }
```

The OUT flips to `RECEIVED` only when no transit row of it has anything left. A short keeps
it open on purpose — that is the loss report, and the write-off (a DAMAGE or ADJUSTMENT
voucher at whichever branch the business blames) is a **separate decision, deliberately not
automated**. Do not add an "auto write-off short" option.

---

## 9. Cancel and delete

| Case | Route | What happens |
|---|---|---|
| DRAFT (either type) | `DELETE /stock/transfer?svhId=&accYear=` | soft delete; set `svi_is_deleted` on the lines in the same transaction. **Delete, never cancel** — the guard refuses cancelling any linked TRANSFER_IN, draft included |
| Same-branch POSTED | `POST /stock/transfer/cancel` → `fn_svh_cancel` | reverses both halves symmetrically |
| IN_TRANSIT / RECEIVED OUT | — | **refused** by `tr_svh_transfer_cancel_guard` (`23001`). Goods that left cannot be cancelled on paper |
| POSTED TRANSFER_IN | — | **refused**. Un-receiving is a reverse transfer, raised as a new document |

Surface the last two as **409 with the engine's own sentence** — they read as instructions
("receive them, or transfer them back"), which is exactly what the clerk needs.

---

## 10. Errors

The opening plan's table, plus the transfer-specific raises. Same `P2010` /
`error.meta.code` trap — switch on `meta.code`, pass the message through.

| SQLSTATE | Raised for | HTTP |
|---|---|---|
| `P0002` no_data_found | voucher not found; the IN links no TRANSFER_OUT; nothing of this lot in transit to this godown | **404** |
| `23001` restrict_violation | not DRAFT; OUT not IN_TRANSIT; cancel of an in-flight OUT; cancel of a posted IN | **409** |
| `0A000` feature_not_supported | wrong type for the function; more than one bucket in transit for one (item, lot, godown) | **409** |
| `23502` not_null_violation | a line names no lot | **422** |
| `23514` check_violation | godown transferred to itself; line has no quantity; receives more than is in transit; sent to another branch; no lines | **422** |
| `23505` unique_violation | refno / slno collision (offline device) | **409** |

---

## 11. Tests

The 2026-09-04 smoke cycle, as automated cases — and `20_stock_transfer_flow.md` is the
figure-for-figure expectation:

1. **Same-branch pair**: 2 ledger rows, voucher POSTED, balance moved between godown rows,
   `stock_item_cost` **unchanged** (an internal move is not a revaluation), `stock_transit`
   still empty.
2. **Same-branch cancel**: both halves reversed, balances back.
3. **Inter-branch despatch**: 1 ledger row, 1 transit row, voucher IN_TRANSIT, destination
   `sbl_transit_in_qty` = sent — including the case where the destination godown has **never
   held the item** (`fn_sbl_ensure_row` must have created the row).
4. **Partial receive with damage**: 2 ledger rows, transit PARTIAL, `stt_short_qty` = 2, OUT
   **still IN_TRANSIT**, DAMAGED bucket holding 3 at the despatch cost.
5. **Second receive closes it**: transit RECEIVED, OUT RECEIVED.
6. **Refusals**: receive more than remains; receive at the wrong branch; cancel in flight;
   un-receive; a line with no lot; a godown transferred to itself.
7. **Cost travels**: the IN row's `sml_cost_rate` equals the OUT row's, not the destination
   branch's average. Assert the number, not the code path.
8. **The wot asymmetry** (flow doc B6a): at receipt, `sml_cost_rate` comes from the transit
   row and `sml_cost_rate_wot` from the lot as it stands. Pin the current behaviour in a test
   so a later engine fix is a deliberate change and not a surprise.
9. `fn_sbl_rebuild` returns 0 differing holdings after each of the above.

---

## Open items

1. **Refno prefixes** `TRF` / `TRI` — agree with the client; nothing in the schema fixes them.
2. **LR / vehicle / expected date** (§0.3) — API-side update or three new function parameters.
3. **`cost_rate_wot` at receipt** — engine asymmetry, flow doc B6a. The review's fix is a
   `stt_cost_rate_wot` column on `stock_transit`; until it lands, the IN row's two rates are
   from different moments and `cost_value_wot` must not be reconciled across a transit.
4. **Over-issue on despatch.** Under `stp_allow_negative = 'ALLOW'` the engine will send
   stock a branch does not have. The API check in §3.2 is the only guard; decide whether a
   transfer should be `BLOCK` regardless of the item's policy.
5. **Who writes off a short**, and against which reason code — and, separately, **what closes
   the transit row**. A write-off voucher does not touch it; without `fn_stt_settle_short` the
   row stays PARTIAL and its remainder stays in the destination's `sbl_transit_in_qty` for
   ever. `stock_reason_master` exists; the reason list for transfer loss does not.
6. **Multi-destination despatch.** One OUT voucher carries one `svh_to_godown_id`. If the
   warehouse wants one lorry serving three shops, that is three vouchers — confirm the screen
   is built that way before somebody asks for a destination column in the grid.

