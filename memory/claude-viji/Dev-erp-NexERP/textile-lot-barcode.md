---
name: textile-lot-barcode
description: "Textile \"scan the lot\" model — verdict 2026-09-27 that the stock model supports it additively; what exists live vs what must be added; PNGs in share/schema/stock"
metadata:
  node_type: memory
  type: project
  originSessionId: fb887529-7eff-40f5-9b97-2923bba4d2c2
  modified: 2026-09-27T01:29:43.112Z
---

User's textile need (2026-09-27): designs like "Deepa Saree" come once, so no
item master per design. Each purchase line (item, brand, size, colour, model,
MRP, selling price) = ONE lot with an auto batch no printed as a label; the
sale scans the label and gets every detail. User wants NO heavy customisation;
the research was to decide go / stop. **Verdict given: supported, proceed.**
User does NOT want size/colour on the item.

**Live facts (192.168.0.106 + nexerp-api-ro):** stock_lot has batch_no / mrp /
sale_price / supplier / costs; `ix_slt_batch_no (company, batch_no)` exists but
is NOT unique; sale_bill_item has sbi_lot_id and NestJS sales-stock.service
posts lotId; NestJS resolveLots blanks untracked dims, so the item policy must
track batch + MRP + selling price (custom 'BMS', no such preset) or prices go
NULL. Sale-bill scan = getItemByBarcode → item_ean_codes only (no lot).
**No purchase module exists** (DB: suppliers only; NestJS purchase: suppliers only).

**REV 2 — user decision: NO stock structure change.** User rejected adding
columns to stock_lot ("we have to change the stock structure"). Design is now a
TAG TABLE outside the stock schema: `inventory.item_lot_tag` (tag_barcode
UNIQUE per company, lot_id → slt_id, item, brand, size, colour, model). The auto
batch no goes in slt_batch_no as an ordinary batch; uniqueness lives on the tag
table; ix_slt_batch_no untouched. To add: tag table, counter, barcode → tag →
lot lookup route, lot branch in the sale-bill scan, purchase entry writes lot +
tag + label. Only stock-side item = BMS policy (configuration). No plan file
written yet.

**REV 3 — user decision: MRP + selling price ALSO on the tag row.** Textile
items use the seeded 'Batch only' preset (BATCH, signature B), so NestJS
resolveLots leaves slt_mrp / slt_sale_price NULL and the tag is the price's ONLY
home (never both). Cost (purchase/cost/landed rate) STAYS on the lot — the
engine values stock and COGS from it. Consequences: the sale bill takes a
tagged row's price from the tag (lookup route returns it), a markdown is an
UPDATE on the tag, and the Change Selling Price bulk screen does not reprice
tags. This supersedes the 'BMS custom policy' line above.

**PLAN WRITTEN 2026-09-27:** share/schema/stock/textile/plan-textile-lot-tag.md (new
file, edits no plan). inventory.item_lot_tag (ilt_) + item_lot_tag_series (ilts_,
because fixed.device_master has NO short code column); supplier also on the tag
(Batch-only blanks slt_supplier_id too); new preset TEXTILE_TAG = B + MANUAL
issue (BATCH preset is FIFO, and engine plan §1.4 D1 would pick a wrong design);
routes /api/v1/textile-tags/{reserve,scan,get,update,price,print,void}; opening
= go-live path with LAZY lot link (scan never writes); setting
stock.textile_tags_enabled in app_setting_def; build order T-1..T-5, decisions T1-T7.

**2026-09-27: user ACCEPTED ALL DEFAULTS T1-T7** (2-char series + 6 digits;
one label per line printed N times; size/colour free text; brand promos match
the TAG brand; tax from tag price by company slab; below-cost markdown with a
confirm; no series code = no tags). Recorded here and in the mockup header —
the plan file itself was NOT edited (plans-separate-file rule).
**Tag screen mockup DONE:** share/schema/stock/textile/textile_tag_ui_mockup.png (+ .py):
tabs Register / New tags (with label preview) / Markdown / Series, plus a
sale-bill scan inset; 14 numbered callouts. Next = user review, then T-1 backend.

PNGs: share/schema/stock/textile/textile_variant_fit.png (size/colour-as-item view)
and textile_lot_barcode_fit.png (this flow), each with its .py.
See [[stock-schema-design]], [[stock-engine-not-deployed]], [[plans-separate-file]].

**2026-10-17: files moved to share/schema/stock/textile/.** Gap addendum written there:
plan-textile-lot-tag-gaps.md (G1–G19 vs Ginesys/GoFrugal/Logic). Key proposals: tag stores NO
tax, slab decided at billing on net value (supersedes T5); cost cipher + print-only
ilt_label_cost; markup rules table; metre cut qty; size/colour masters (supersedes T3); hold
columns.
**2026-10-17: user ACCEPTED ALL DEFAULTS G1–G19** (recorded here + in the REV 2 mockup header;
neither plan file edited). **Mockup REV 2 DONE:** textile/textile_tag_ui_mockup_rev2.png (+ .py),
REV 1 kept. Five tabs Register / New tags / Markdown / Markup (new, G3) / Series + sale-bill inset
(net-value slab, metre qty prompt, hold warning). 18 callouts, teal G-tags. Next = user review of
REV 2, then T-1 backend. Nothing built yet.
**2026-10-17 later: addendum 2 = plan-textile-lot-tag-gaps-2.md, G20–G23 accepted** (user "yes"):
fabric list (ilt_fabric_id, item_fabric_master ifb_), season list (ilt_season_id,
item_season_master isn_, header field on New tags), ilt_supplier_article; gender is NOT a tag
field — it is the item's section tree (item_section_master exists live). Rule: a field goes on the
tag only if it changes between purchase lines of the same generic item. **Mockup REV 3** =
textile_tag_ui_mockup_rev3.png (+ .py), current; REV 1/2 kept.
**2026-10-17: BACKEND PLAN written** — share/schema/stock/textile/plan-nestjs-textile-tags.md.
Key moves: lot id is deterministic (lotIdentityUuid, posting.helper.ts:1383) → ilt_lot_id computed
at reserve, no FK, status stored only OPEN/VOID and RESERVED/ACTIVE/SOLD_OUT derived → no lazy link,
no nightly job (server has no cron). Cloud sync is PUSH-ONLY (v_sync_tables) → down-sync of tags is a
platform gap. Garment slab not in statutory_limits → new GST_VALUE_SLAB_APPAREL + public.tax_value_slab
(general, not textile-only). Decisions B1–B9 await user; build B-1..B-6.
**2026-10-17: user ACCEPTED B1–B9 defaults.** Ledger of all accepted sets: share/schema/stock/textile/DECISIONS.md. Next = B-1 + B-4 (server, Prathap's side; NestJS is read-only for me).
**2026-10-17: handover = notes-136** (share/schema/stock/textile/notes-136-textile-tags-backend.md), STATUS asked; one question to Prathap = down-sync of masters to tills (B7).
**2026-10-17: Qt tag screen STARTED** (user "start the Qt tag screen"; "use server side ui table").
Code: src/modules/inventory/stock/textile/ — TextileTagEntry (5 tabs; ONLY New tags built, others
show which server step they wait for), TextileTagDelegate, TextileTagLists (dropdown ids 0 = names
resolve via lists save, 409 = existing row), TextileLabelPreview (real Code 128B). ui_table **53**
"TEXTILE TAGS - NEW TAG LINES" registered by me (script share/schema/stock/textile/qt/
textile_new_tags_ui_table.py). NOT registered in form_inventory_menu.h — no menu id yet.
Test tests/textiletag/run.sh 37/37 (reserve + suggest-price answers FAKED via applyReserved /
applySuggestion). Client JSON choices for Prathap: share/schema/stock/textile/qt/client-contract.md.
Trap: never use "size" as a QObject dynamic property — QWidget already has it (use "tagSize").
**2026-10-23 USER CHANGE — tag page too complex; tags PARKED (kept for later, nothing deleted).** New ask: purchase
line = item + auto batch no (e.g. 5437-00001) + MRP + selling price; sale bill gets a Batch No column, scan
fills from stock. My verdict: current stock structure supports it with NO new table — item policy tracks
Batch+MRP+SP (signature BMS, CHECKs allow it, LOT_ACTUAL), lot carries slt_batch_no/slt_mrp/slt_sale_price;
sales posting already passes sbiBatchNo → svi_batch_no → lot resolve. Gaps: no purchase module; no
batch→lot scan route (ix_slt_batch_no non-unique → picker on dupes); NestJS has an UNUSED
common/utils/batch-number.util.ts generateNextBatchNo (MAX-scan, not offline/race safe → needs a locked
counter); new preset "Textile (batch+MRP+SP)" needed. Asked user what "5437" in the batch no stands for.
**2026-10-23 later:** user: "5437 was just an example" → I decided format `<br_bill_prefix|br_code><YY>-<NNNNN>`
(CBE26-00001) via allocateVoucherNumber + counter-only voucher type BATCH (scope is per branch per acc_year,
so branch+year in the text = unique offline). PLAN WRITTEN: share/schema/stock/textile/plan-textile-auto-batch.md
(D1–D8; setting stock.auto_batch_no; preset TEXTILE_BATCH = BMS; route GET /stock/lots/by-batch; opening stock
first since no purchase module; Qt sale bill already has SalesItemCols::BatchNo/LotId). Notes to Prathap not written yet.
**2026-10-23 final:** user chose size/colour/model on the PURCHASE LINE (not item master). Size + Colour =
masters (inventory.item_size_master isz_, item_colour_master icl_, quick-add); Model = free text + suggestions;
fabric/season/article NOT now. Detail table stock.stock_batch_detail (sbd_) keyed by (company, batch_no) —
re-lot needs no copy, drafts need no line column. Addendum: textile/plan-textile-auto-batch-2.md (E1–E4).
Handover: textile/notes-147-textile-auto-batch-backend.md (S1/S2, 2 questions). Next = Prathap S1; Qt after.
**2026-10-23 evening:** Prathap already built isz_/icl_/sbd_ tables, menus 291/292, grids 181/182 (tokens
iisz_is_deleted/iisz_company_id), dropdowns 70/71 (icompany_id), audit 153/156; ROUTES NOT LIVE (404).
Qt Size/Colour screens BUILT: src/modules/inventory/master/{sizes,colours}, endpoints ItemSizeMaster/
ItemColourMaster (/item-sizes|/item-colours save/get/delete, id keys isz_id/icl_id), fm.reg 291/292. Built OK;
no test yet (routes missing). Client contract appended to notes-147. NOTE: another session also wrote
inventory/notes-147-unit-factor… — number collision, left as is. Build dir = Desktop_Qt_6_10_2-Debug (6_10_3 is dead).
**Size/Colour LIVE 10-23:** server routes are /item-sizes|/item-colours {create,get,delete,restore} (not /save);
Qt switched + Show Only Deleted/Restore; Company locked on edit (server silently keeps scope); /get lacks
*_company_name → mapResponse fills from AppSession. Fixed MenuWindow::onCompanyChanged to set the session
company NAME (was never set except by opening balance). Tests: tests/itemmasters/api_size_colour.py 30/30,
SRC=test_size_colour_screen ALL PASS. Notes-147 "Client status 2" = 2 small asks + no grants for 291/292.
Next: S2 routes /stock/lots/by-batch + batch-detail/{models,update} are LIVE too → opening stock columns, sale bill scan.
**2026-10-24 Opening Stock textile columns BUILT (uncommitted):** OpeningStockCols 39–44 (SizeName/SizeId/ColourName/
ColourId/Model/BatchTyped), ui_table 27 rows added after Batch No via share textile/qt/opening_stock_textile_cols.py
(upsert, replaceColumns false); .ui columnCount 45 (a missing .ui column = NULL header item crash in retranslateUi).
AppSession::autoBatchNo() ← stock.auto_batch_no (off on every company; tests switch it on and remove it).
Size/Colour = editable combo from dd 70/71 + quick-add confirm; Model = completer from /stock/batch-detail/models; typed batch
→ BatchTyped=1 clears + locks them. Save message lists "Batch numbers given". Test SRC=test_textile_cols 36/37:
X10 fails = server: SaveOpeningStockVoucherItemDto lacks sizeId/colourId/model (put on transfer DTO by mistake);
draft cancel leaves sbd rows live → notes-149 (textile/). Next: sale bill scan + columns.
**2026-10-24: notes-149 LIVE + verified** — opening DTO takes sizeId/colourId/model (test_textile_cols ALL PASS 37), re-save with given batch OK, keys omitted = cleared, typed batch+size 422, draft cancel soft-deletes sbd (old rows cleaned). Open Q in the note: posted-opening cancel leaves sbd live (CBE26-00006).

**Opening Sale Price (10-24):** ui_table 27 col 13 "Sale Price" was visible but width 0 → never showed; set width 5 (share schema/stock/textile/qt/opening_stock_sale_price_width.py). Editable only when the item tracks S (preset TEXTILE_BATCH = BMS, or SP_ONLY). Selling price per batch lives on the lot (slt_sale_price), part of lot identity.
