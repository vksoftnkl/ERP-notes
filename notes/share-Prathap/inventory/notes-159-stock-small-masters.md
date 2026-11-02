# notes-159 — stock small masters: Stock Reasons (built), Ageing Slabs and Tracking Presets (need routes), menu rows

Date: 2026-10-26. From: Viji / Claude. To: Prathap.

## 1. Menu rows (please insert)

| menu_id | name | parent | position | verbs |
|---|---|---|---|---|
| **293** | Stock Reasons | 34 (Item Masters) | 110 | VIEW, CREATE, EDIT, DELETE, PRINT, EXPORT |
| **294** | Ageing Slabs | 34 (Item Masters) | 120 | VIEW, EDIT |

Put them after Tracking Presets (249, position 100), with `menu_separator = true` on Units' group as it already is. Then grant VKPOS, or tell me and I will do the grant.

## 2. Stock Reasons — BUILT, live-tested

- **Qt:** `StockReasonList`, registered as `fm.reg(293)`, over **grid 188** "MAIN LIST - STOCK REASONS" (shared reasons plus the company's own; a shared row the company has replaced is not listed). `StockReasonEntry` is the popup.
- **Uses your existing routes:** `POST /stock/reasons` (create / update), `POST /stock/reasons/deactivate` (with `reactivate`).
- **Edit on a shared reason** saves a company copy with the same code, which replaces it for that company. Off on a shared reason saves an inactive copy.
- **Live test** (`tests/stockreason`): all pass. It created, renamed and deleted a ZT reason, copied GIFT and removed the copy. No rows are left behind.

Nothing is needed from you here except the menu row.

## 3. Ageing Slabs — please add routes

Table `stock.stock_ageing_slab` exists. The screen is the mockup `schema/stock/ageing_slab_ui_mockup.png`: **one ladder, saved whole, never edited row by row.** `ex_sag_overlap` refuses a one-row edit that moves a boundary, and the fallback to the default ladder is all-or-nothing.

```
GET  /stock/ageing-slabs?companyId=
     → { usingOwn: bool,                     // the company has its own rows
         slabs: [{ slabNo, label, fromDays, toDays|null }] }   // its own, else the shared default (company NULL)

POST /stock/ageing-slabs/save   { companyId, slabs: [{ slabNo, label, fromDays, toDays|null }] }
     One transaction, SET CONSTRAINTS ex_sag_overlap DEFERRED; replace every company row.
     422 per field (slabs.<i>.fromDays …) unless:
       - the first slab starts at 0;
       - each slab starts the day after the previous one ends (no gap, no overlap);
       - exactly one slab is open-ended (toDays null), and it is the last;
       - labels are non-empty and unique; 1–20 slabs; slabNo = 1..n in order.
     "Customise" on the screen = this save, sent with a copy of the default ladder.

POST /stock/ageing-slabs/revert { companyId }   → deletes the company's rows; the default applies again.
```

The rights are menu 294: VIEW for GET, EDIT for save and revert.

## 4. Tracking Presets — please add write routes

Today there is only `GET /stock-track-presets/get`, and the screen (menu 249, grid 98) is read-only. A preset decides **how an item's stock is split into lots**, so:

```
POST /stock-track-presets/save      create (no spt_id) or update
POST /stock-track-presets/deactivate { companyId, sptId, reactivate? }
```

The rules:
- **A shared preset (company NULL) is read-only.** A company row with the same `spt_code` overrides it, the same pattern as stock reasons.
- **Once any item or group uses the preset:**
  - **the six `spt_track_*` flags and `spt_valuation_method` cannot change** (409, worded). Changing them would re-split existing stock into different lots.
  - Editable are: name, description, issue strategy, allow negative, shelf life, near-expiry days, block expired sale, ageing basis, sort, remarks.
- **The CHECKs already on the table** should come back as worded 422s, not raw errors:
  - expiry needs batch;
  - FEFO needs expiry;
  - valuation is LOT_ACTUAL when tracked, else WAVG.
- **Deactivate is refused** while any active item or group uses the preset.
- **The GET should add `usedByItems` / `usedByGroups`**, so the screen can show and lock correctly.

When §3 and §4 are live, tell me. I will build the Ageing Slabs ladder editor (to the mockup) and add Add / Edit to the Tracking Presets list.
