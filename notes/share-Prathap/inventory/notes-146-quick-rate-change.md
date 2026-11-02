# Notes 146 — Quick rate change popup: APPROVED (2026-10-23)

The design is `plan-quick-rate-change.md` (with `quick_rate_change_mockup.png`).
That file is unchanged; this note records the user's decisions and asks for
two small server additions. Where this note and the plan differ, this note wins.

## User's decisions

| Question | Answer |
|---|---|
| Key to open it | **Ctrl+R** on a Sale Bill / Quotation / Sale Order / DC line and on the Item list |
| "Apply the new rate to this bill line" | **ticked by default** |
| Which price levels | **only the active ones** (the `priceLevels` list: `item_price_levels.ipl_status`), levels 1–4 = price A–D |

The client is being built now against the routes that already exist
(`GET/POST /stock/price-bulk` and `/item-price-log/item-history`). Both asks
below are **additive**: the popup works without them and starts using them when
they appear.

## A. `GET /stock/item-inward-history` (new, read-only)

The "Recent inward — what this item cost" panel. There's no purchase module
yet, so it reads the stock ledger, where every inward already lands (opening
today, purchase / GRN later).

**Params:** `companyId`, `branchId` (optional = whole company), `itemId`, `limit` (default 10, ≤ 50).

**Rows** come from `stock.stock_ledger`:
- `sml_direction = 1`
- `NOT sml_is_reversal`
- no reversal exists for the row (`NOT EXISTS (… sml_reverses_id = s.sml_id)`), so a cancelled document drops out
- `sml_txn_type` in the inward types (`OPENING`, and the purchase types when they exist)
- newest first: `sml_doc_date DESC, sml_created_on DESC`

**Each row:**
```json
{ "docDate": "2026-10-21", "txnType": "OPENING", "refno": "OPN0006",
  "partyId": "…", "partyName": "Ravi Distributors",          // sml_party_id → name, null if none
  "qty": 24, "unitName": "PCS", "baseQty": 24,
  "costRateWot": 79.29, "costRate": 93.56, "taxPerc": 18,     // tax % = round((rate/wot − 1)·100, 2)
  "mrp": 150, "batchNo": null, "godownName": "Coimbatore" }
```

**Rights:** VIEW on menu 30 (Change Selling Price) or menu 29 (Item master), whichever you prefer. Cost is sensitive, so not every billing user should see it.

## B. `originMenuId` on `POST /stock/price-bulk`

Optional integer on `SaveSellingPriceBulkDto`: the menu the popup was opened
from (Sale Bill 31, Quotation, Sale Order, DC, Item list 29).

When it's present and isn't 30:
- `PriceChangeLogService.record()` appends **"via <menu name> (menu N)"** to `ipcl_reason`. For example, "via Sale Bill (menu 31)".
- The source stays `CHANGE_SELLING_PRICE` and `ipcl_menu_id` stays 30. Menu 30's own rights and rules are what allow the save; the reason just says where the user was.

Then the Price Change Log can tell a rate changed while billing from one
changed on Change Selling Price. Until this lands, the client simply doesn't
send the field. The DTO whitelists its fields, so sending it now would be a 400.

## Rights (already true, no change)

The popup saves only if the user has **Edit on menu 30**. Without it, the popup
opens read-only. The server's existing check on `POST /stock/price-bulk` stays
the real rule.
