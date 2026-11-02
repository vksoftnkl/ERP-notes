---
name: erp-item-dropdown-search-broken
description: "FIXED 2026-07-23 — item dropdown (dropdown_id=6) search was returning empty for any query; root cause was a data config gap, not code"
metadata: 
  node_type: memory
  type: project
  originSessionId: 59166b0b-1e68-4bcc-8b0c-b907db8e574a
  modified: 2026-07-23T06:06:23.543Z
---

**Status: fixed 2026-07-23** via ERP server migration `20260723060000_make_item_dropdown_columns_searchable` (applied with `npx prisma migrate deploy`, in the ERP server repo). Root-cause trail kept below in case it regresses, or another dropdown needs the same fix.

The configured items dropdown (`dropdown_id=6`, backs `useLazyGetItemOptionsQuery` in `store/api/lookupsApi.ts` and `ItemSearchCell` in [[erp-item-qty-price-page]], endpoint `/dropdown-details/run`) returned `{"items":[],"meta":{"total":0}}` for ANY `search` value, even an exact match — omitting `search` returned the normal unfiltered page fine.

**Root cause:** not a SQL/param bug — `fixed.dropdown_columns` had all 3 rows for `dropdown_columns_dropdown_id=6` seeded with `dropdown_columns_filter=false` and no `dropdown_columns_sql_name`. `ConfiguredGridSqlService.deriveSearchableFieldNames()` returns `[]` when no column is filterable; `buildSearchSql()` then forces `WHERE 1=0` onto the wrapped query whenever `searchableFieldNames.length === 0` (a deliberate no-match default, not a crash) — so any non-empty `search` param always yielded zero rows. Working dropdowns (state=21, company=22, acct group=23, branch=24 per [[erp-configured-dropdowns]]) all have `filter=true` + `sql_name` set on their name-ish columns; item's (id=6) just never got that seeding.

**Fix applied:** `UPDATE fixed.dropdown_columns SET dropdown_columns_filter=true, dropdown_columns_sql_name='item_name_en' WHERE dropdown_columns_id='019ebb95-1a0f-7a66-a704-381915eb4972'` (Item name) and same pattern for Item alias → `item_alias` (id `019ebb95-1a0f-7a6b-8974-16c23ef4fe37`). `sql_name` must match the exact SELECT alias in `fixed.dropdown_details.dropdown_sql` for that dropdown_id. "Item id" (the UUID) deliberately left non-filterable.

**How to apply elsewhere:** if a *different* configured dropdown's search silently returns nothing for every term (not just for one bad term), check `fixed.dropdown_columns` for that `dropdown_id` first — `dropdown_columns_filter=false` on every row is the same latent trap, one `UPDATE` (packaged as a migration) fixes it. Don't assume it needs a code change.
