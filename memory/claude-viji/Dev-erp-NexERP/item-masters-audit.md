---
name: item-masters-audit
description: 2026-09-30 audit of every master under menu 34 + the item list — three grids dead/wrong (45, 50, 67), hierarchy half-built, DELETE toggles, unit_uqc unreachable; client fixes applied same day; notes 70 written; ui_table 19 re-registered
metadata:
  type: project
---

Full write-up for Prathap: `share/inventory/notes-70-item-masters-audit.md` (A blockers, B hierarchy, C delete guards, D contract gaps, E verified clean).

**Client fixed 2026-09-30 (built clean):** restore-aware confirm on the six toggle-delete lists (group/brand/section/category/unit/godown); item list `nameColumn` → `item_name_en` (grid 67 has no `item_name`); unit Decimal Count spin min 0; godown: Branch required, `gdl_code` field, `blankAs(0)` on sort/volume, dead beforeSubmit removed, self-parent guard; self-parent `extraValidate` on group/section/category (dropdowns 17/19/20 offer self); qty-wise price: null price level + null To Qty ("& above"), mode Q→R on the wire, From Date defaults to today, paged load (`getByItem(itemId,page,limit)`, limit 100, walks `meta.total_pages`).

**Still owed:**
- ~~ui_table 19~~ DONE 2026-09-30 with the user's permission: re-registered to 22 columns via `/ui-table-masters/create` + `replaceColumns:true` (col 0 `Id` hidden, 17-21 hidden, old col 22 soft-deleted); verified by GET.
- Grids 45/50/67 are Prathap's (A1-A3). Menu 38 and 37 lists are EMPTY / 400 until fixed; the item list's deleted filter is inert.
- nexerp-api-ro snapshot is behind the box (stock-track-policy.service.ts) — re-sync.

**Section status (menu parent 3):** 30 backend-only + plan (see [[pricing-one-table-analysis]]); 164 Reorder backend-partial (per-item rows only); 31/115/32/238/33/176/178 nothing. See [[item-master-audit]] for the item ENTRY (done), [[user-menu-full-replace]], [[test-box-data-not-a-bug-list]].

**Notes 70 + 71 applied by Prathap, re-verified live 2026-09-30 evening:** grids 45/50/67 fixed; restore routes on all masters (POST ?<idKey>=); DELETE refuses a second delete (409), live children (409), unit in use (409); group cycle 400; level computed; godown keeps an omitted parent; group delete retires its policy; sec_name unique; bulk price save 201 (update + insert, noStock listed); lookup with no mrp on an all-bucket item now answers the dearest bucket. STILL OPEN: unit_uqc not in DTO; category tax/HSN/UOM keys still accepted with no columns; lookup with an unknown mrp → 404; a bulk-inserted bucket does NOT copy addl_cess/min from the headline (0.00).
**Client restore DONE 2026-10-01:** the six master lists (group/brand/section/category/unit/godown) now POST `<ns>::Restore?<idKey>=` in the Show-Only-Deleted view, and DELETE otherwise, like item_list.cpp. They build clean but haven't been clicked live yet.
