Date: 2026-10-29
From: Viji / Claude
To: Prathap

# Notes 163: what changed on the server today (2026-10-29)

Everything below is in `/home/vk/Dev/erp/ERP server`, branch DEV-v1, on .106. API :3011 (your `nest --watch`) already serves it. Nothing here was pushed to prod.

## 1. The staged tree (please read first)

1. The index of `ERP server` holds about 126 staged files (8,260 insertions) plus about 53 unstaged edits on top. Neither of us has committed them. The staged files are the work of several agents at once. The "MM" files (staged AND changed since) are mostly purchase and print-template files.
2. I did not touch the index. The user decides how to stage and commit. Nothing is committed from this batch except what is listed in item 3.
3. Already committed today by the user (13:04 and 13:05, three commits, same subject "feat(party): party master rules P1 + customer save fixes; ledger names reusable after delete"): the customer / customer-group / master-lookup fixes, migrations 110000 and 120000, State_Codes.sql, `module-shared.utils.ts`. Also f2b509ed (11:43, grid 195). So the customer fixes are no longer pending.
4. `dist/` is tracked and rewritten by the watch compile. Many dist files are in the staged list; please do not hand-merge them. `dist/tsconfig.tsbuildinfo` shows as added.
5. Asked of you: if you are committing in that folder today, tell the user first, so the staged set is not committed under the wrong subject.

## 2. Migrations APPLIED today (all four are in `_prisma_migrations`, finished 17:32 IST)

1. `20261029100000_grid_195_supplier_item_scheme`: grid 195 (Supplier-Item Map list, menu 295) now returns and shows Scheme Buy / Scheme Free. Committed (f2b509ed).
2. `20261029110000_ledger_name_unique_skip_deleted`: drops the two old plain unique indexes on `accounts.acc_ledger_master` (name, Tally name). A deleted ledger no longer holds its name. The Tally name gets partial unique indexes `uq_led_tally_name_company` and `uq_led_tally_name_shared` (live rows only). The name keys `uq_led_name_company` / `uq_led_name_shared` already existed. The Prisma model dropped the matching `@@unique` lines. Committed.
3. `20261029120000_party_master_rules_p1`: ledger GST type + SEZ flag derived from the party; `cus_gst_type` / `sup_gst_type` normalised to five codes; state 96 "Other Country"; index `ix_sb_cust_posted`; one-time reset of `cus_billed_date`, `cus_billed_count`, `sup_billed_date` (nothing writes them any more); plain wording rows. The CHECKs `ck_cus_gst_type` / `ck_sup_gst_type` are phase 2, not here. Committed.
4. `20261029130000_purchase_fidelity_db_needs` (staged, not committed):
   - menu 297 "Supplier Dues" under Purchase (verbs VIEW CREATE EDIT PRINT EXPORT POST CANCEL); also in `Menu_Master.sql`;
   - seven shared purchase-return reasons `PRET_*` (also in `Stock_Reason_Master.sql`);
   - `pur_shortage_receipt.psr_supervisor_id` FK;
   - `pur_return.pr_memo_ref`, `pur_invoice.pi_vehicle_no`, `pi_driver_name`;
   - new table `purchase.pur_terms` (PO terms master) + `pur_order.po_terms_id`; dropdown 73 "PO TERMS"; registry row 803 in `Db_Table_Master.sql`;
   - menu 196 (GRN) gets the OVERRIDE verb;
   - `pur_claim_settlement` now settles EITHER a return claim line OR a shortage close of kind CLAIM (`pcs_shortage_close_id`, `ck_pcs_source`).
   Rights: VKPOS was granted 279, 297 and Override on 196 (by the user, in User Administration). Please remember Override on 196 and 297 when you check rights on your side.

## 3. New or changed routes and keys (purchase)

Verified in the diff and Swagger on :3011; field-level detail is in Swagger.
1. NEW `GET /purchase/supplier-dues/open`, `POST /purchase/supplier-dues/settle`, `POST /purchase/supplier-dues/settle-undo`. Open rows carry age, reason, writtenOff, status, lastAction, kindText, claimRef; totals carry openCount. Settle takes several rows at once, has `dryRun` and `writeOffRest`.
2. `POST /purchase/shortages/close`: kind now SUPPLIER_CN, CLAIM (transporter / insurer: claim receivable) or WRITE_OFF. `close-undo` unchanged.
3. NEW `/purchase/terms/{create,get,delete}` (rights on menu 23); PO header takes `termsId` and `agentId`; PO takes `charges[]` (CHG-07, an ESTIMATE only: nothing posts); PO totals gain estimate fields.
4. GRN: `POST /purchase/grns/post` takes `overrideWarnings` + `overrideReason` (Override on 196, logged as WARN_OVERRIDE; needed for PUR_SHELF_LIFE in REFUSE mode); a validate route now returns `warnings[]` / `refusals[]`; header takes a shared `transport` band (mode, transporter, LR no/date, distance) and e-way date; `/get` returns supplierName, godownName, supervisorName, receivedBy as {id,name}, billStatus, per-line check, billed / billRefno, charges, totals.
5. Purchase invoice: header `agentId`, `billFrom`, `transport` band, vehicle / driver; post takes `overrideWarnings` + `overrideReason` (Override on 24).
6. `POST /purchase/returns/validate` added.
7. `PurchaseModule` now also provides `TransportBandService` (from sales/posting) and the two new controllers.

## 4. The DTO "own-undefined" trap and its fix (please apply the same pattern in your modules)

1. Cause: tsconfig target is ES2022, so class fields are emitted. With `transform: true` every declared DTO field is an OWN property, holding `undefined` when the client left it out. So `hasOwnProperty(dto, 'x')` is true for a key nobody sent, and `dto.x ?? null` wipes a stored value.
2. Seen live: every Qt customer save reset the ledger TDS flag; collection days were wiped; a group rename checked the wrong company; publishing a print template with a status-only version row nulled `ptv_params` and `ptv_note`, so every render was refused PRINT_RENDER_PARAM_UNKNOWN.
3. Fix: `isPresentField(obj, key)` in `src/common/utils/module-shared.utils.ts` (own AND `!== undefined`). `applyPresentFields` uses it. Rule: never `hasOwnProperty` / `in` on a DTO; use `isPresentField`.
4. Swept (staged, not committed): print-template (plus new `print-template-frozen-compare`), print-template-assignment, charge master, items master, suppliers, sale agent, tax rate master, widget master, area, batch prefix, tender master (+ DTO), ledger shipping address, ledger bank account, device list master, items-cust-rates master, transport-band, txn-status-log helper, staff-advance-ledger. Customer, customer-group and the ledger company check were fixed earlier and are committed.
5. Specs must run the body through `createHouseValidationPipe`, otherwise they cannot see the bug.
6. A PUBLISHED print revision is frozen. Re-publish as a NEW revision with a FULL row, never a status-only row. PARTY_OUTSTANDING_BILLWISE_A4 was broken exactly this way and repaired as rev 2.
7. Customer bug batch (committed): null numbers become 0; price level / branch existence checks; a blank name on update is refused; the actor always comes from the request; the create name-clash code; DTO min/max, GSTIN (all GSTN types; state prefix checked for NORMAL only), PAN, email; lookup TCS and unknown company. `cusDebitBalance` may be negative.

## 5. Error-message seed: NOT yet run

1. `prisma/seed/Error_Messages.json` in the working tree is version `2026-10-29.4` (HEAD holds an earlier one).
2. The DB catalogue (`fixed.error_message_catalogue`) is still `2026-10-29.1`, 388 rows (set by migration 120000).
3. So the purchase / dues / GRN / return wording added in .2 to .4 is NOT in the DB. Until the seed runs, those refusals show the fallback wording.
4. Asked of you: nothing to change in code. The user runs the seed (it does not overwrite rows whose wording was edited; please confirm that is still how your seed loader behaves).

## 6. Print template

PARTY_OUTSTANDING_BILLWISE_A4 was re-published as rev 2 (script `/home/vk/Shared/Prathap/print/party_outstanding_billwise_template.py`, ids in `party_outstanding_billwise_ids.json`; proofs `PartyOutstandingBillwiseA4*.pdf/png`). Rev 1 was the casualty of the status-only wipe in item 4.

## 7. Open questions for you

1. Party plan (`/home/vk/Shared/Prathap/party/plan-party-master-rules.md`) steps after P1 are being built now; the status file is `party/status-party-master-rules-2026-10-29.md`. Please do not change the customer / supplier save path until that build reports.
2. Supplier "last billed" stops being written by supplier saves (decision Q10); phase 2 computes it from the last posted purchase invoice.
3. Server menu-rights checks on masters are an open gap (not decided).
