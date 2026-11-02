---
name: quick-rate-change-design
description: 2026-10-23 — Ctrl+R quick rate popup from sales lines / item list: APPROVED + BUILT + live-tested 22/22 (uncommitted); notes-146 A+B live
metadata:
  type: project
---

User 2026-10-23 (screenshot 2026-10-23 13-56-45 = 3.0 per-item Change Selling Price popup: unit price grid + purchase history, Save F5): wants rate changed instantly from sales entry / item main list.

**Plan:** `/home/vk/Shared/Prathap/inventory/plan-quick-rate-change.md` + `quick_rate_change_mockup.{py,png}`.

- One shared popup `QuickRateDialog`, opened with Ctrl+R (proposed; TxnKeys line action) on Sale Bill / Quotation / SO / DC lines and the Item list.
- Grid = menu 30's rows for ONE item (`GET /stock/price-bulk?itemId=`), with level bands A Whole Sale / B Retail / C Bulk WS / D Hotel (fixed.price_levels 1–4). Markup ↔ price on cost w/o tax; delta chips.
- Save via `POST /stock/price-bulk` (below-cost confirm, effectiveFrom, notes-144 price log).
- "Apply the new rate to this bill line" re-prices the line through the screen's own lookup.
- Rights: Edit on menu 30, else read-only.
- Panels: Recent inward (NEW route `/stock/item-inward-history` from stock_ledger inward rows — no purchase module yet) and Rate changes (price log item-history).
- Server asks: `originMenuId` on SaveSellingPriceBulkDto → "via Sale Bill" in the log reason.

**Questions:** Q1 Ctrl+R, Q2 apply-to-line default ticked, Q3 only the active levels. Related: [[change-selling-screen-built]], [[price-change-log-design]], [[txn-keys-standard]].

**APPROVED 2026-10-23 (user):** Ctrl+R; "apply to this bill line" ticked by default; ACTIVE levels only. Decisions in `inventory/notes-146-quick-rate-change.md` (plan file untouched).

**BUILT 2026-10-23 (uncommitted):** `src/modules/inventory/stock/sellingprice/quick_rate_dialog.{h,cpp}` — QDialog#QuickRateDialog (programmatic UI). Grid = GET /stock/price-bulk?itemId; save = POST /stock/price-bulk (menu-30 payload, BelowCostDialog on needsConfirm, scope BRANCH/CHAIN for HQ types, From date) + `originMenuId` (notes-146 B LIVE → log reason "via Sales Entry (menu 12)"). Levels from ComboHelper "priceLevels" (ids 1..4; on .106 only 1 WS + 2 Retail active). Editable only with a GRANTED Edit row on menu 30 (`permissionsFor(30).canEdit`, NOT AppSession::canEdit which defaults true). Panels: Recent inward = `/stock/item-inward-history` (notes-146 A LIVE; field names date/supplierName/costWot/cost — client reads both), Rate changes = item-history top 5. `openForLine(NexTable*, itemCol, unitCol, nameCol, unitNameCol, menuId, parent, reprice)` follows the line by QPersistentModelIndex. Hooks: TxnKeys Action::ChangeRate (Ctrl+R, line action) bound via bindKey on Sale Bill 12, Quotation 14, Sale Order 11, DC 182 (re-price = their fetchItemPrice); Item list btnChangeRate (Ctrl+R, origin 29). Test `tests/quickrate/run-live.sh` Q1–Q5 22/22, restores ZT-ITEM-A Retail 118. QSS rules in **v33, PUBLISHED 2026-10-23** (md5 a28d7b39).

**10-23 user feedback round:** levels now from the item card's source (`/price-level-masters/get`, 1..4 active → all four: Whole Sale, Retail, Bulk Whole Sale, Hotel); item name is the headline (#qrTitle, "CHANGE RATE" #qrCaption — no QSS yet, needs a v34 rule); grid columns stretch, level price columns ResizeToContents; inward/history tables fitToRows (no empty area, scrollbars off), dialog fitHeight(). Cost stays READ-ONLY: price-bulk save has no cost field → notes-147 B asks for optional `costPrice`. "Bottle cost ~13" = unit-factor direction bug ([[unit-factor-direction-bug]]), not the popup. Test 23/23.

**10-23 user decision (asked):** keep saving through menu 30's route — "today's rate only" — but LOOK like the item card's price grid; NOT the item card grid itself (it saves the whole item via /items/create, re-saving units = the notes-147 path). Built: per level **Profit % | W/o tax | Price** under NexHeaderBands (level colours, names uppercased); profit on cost WITHOUT tax (costWot from price-bulk; price ÷ (1+tax) when inclTax); edit any of the three (derived price takes round-off). From a bill line only the line's unit rows show; "Show all units (N more)" toggle (#chkShowAll); from the item list all units. Test Q1–Q6 all pass.

**notes-147 B LIVE 10-23:** save row `costPrice` (writes ipm_cost_price + cost_wot, logged). Popup: Cost / Cost w/o tax editable ONLY when the row's `costBasis == PRICE_ROW` (no stock cost); ITEM/MRP basis = stock average → read-only with tooltip (a typed cost there is saved to the price row but never shown — looked like "did not save"; found when ZT-ITEM-A restore failed). Sends costPrice only when changed; markups/live line use the current cost. Test Q7 on ZT-ITEM-B (PRICE_ROW, cost 40, restored) + ZT-ITEM-A read-only. All Q1–Q7 pass.
