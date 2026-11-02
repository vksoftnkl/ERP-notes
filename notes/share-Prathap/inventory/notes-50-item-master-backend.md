# notes (50) — item master backend: audit findings (25-Sep)

Found by auditing `modules/Inventory/items-master/` (24-Sep source snapshot)
against a live round trip from the Qt item entry. The Qt side is fixed and
verified: an unchanged open + save now changes **0 of 151** stored values (it
used to rewrite prices on 13 of 25 live items). Everything below is
server-side. Paths are relative to `src/modules/Inventory/`, and
`svc` = `items-master/items-master.service.ts`.

**Root cause behind items 1–2:** the API compiles at target ES2022, so every
declared DTO field is an own property with value `undefined`, and
`hasOwnProperty(dto, k)` is always true. That is harmless where Prisma skips
`undefined`, but destructive wherever the code does `dto.x ?? null`, `?? []`
or `resolveRecordActor(undefined)`.

## 1 · Omitted keys are NULLED on update  (bug — data loss for any client)

| key | where | what happens when it is omitted |
|---|---|---|
| `item_base_unit_id` | svc:608 | set to null (read by bulk-load svc:387-389 and by the conversion base-unit fallback) |
| `item_company_id` | svc:589,605 | set to null |
| `item_packing_item_ids` | svc:811-812 | set to `[]` |
| `ipm_godown_id` | items-price-master.service.ts:332 | set to null |
| `ir_unit_id` | items-reorder-master.service.ts:318 | set to null |

Verified live: an unchanged Qt save wiped the base unit and the packing items.
Qt now always sends all five. Please make an omitted key mean "keep", as every
other item field already does, because the next client will not know.

## 2 · Actor columns on price / conversion rows  (bug)

`ipm_created_by` is null on create (items-price-master.service.ts:265), and
`ipm_updated_by` / `iuc_updated_by` are nulled on every update (:336-337;
item-unit-conversion.service.ts:282-283). Live: 66 of 66 price rows have no
creator. Item, EAN and reorder rows fall back to the request user (svc:530,610);
please do the same here. (Qt now sends the user id on both, so new Qt saves
are stamped.)

## 3 · Unique indexes that are NOT partial  (trap → 409 / 500 at the counter)

* `uq_item_name_en_global` is unique across all companies, **including deleted
  items**, and case-sensitive. A deleted item's name can never be reused, and
  "Rice" / "RICE" are both allowed.
* `uq_ean_code` includes deleted rows. Removing a barcode and adding it back,
  or reusing one from a deleted item, gives a 409.
* `uq_ir_item_unit_godown` includes deleted rows. Re-adding a removed reorder
  gives a 409.
* The price unique key is company + branch + item + iuc and **excludes
  godown**, while `syncPrices` matches rows on unit + godown. Two godown prices
  for one unit collide.
* `item_code` and `item_default_barcode` have no uniqueness at all.

Ask: make these `WHERE NOT is_deleted` partial indexes, decide the case rule
for names, and align the price key with the sync key (with godown or without,
the same on both sides).

## 4 · Error field names mislead  (trap)

Every item P2002 is reported against `item_name_en`, and every FK error
against `item_group_id` (svc:916-925), even when the cause is the barcode or
the tax. The client then points the operator at the wrong field. Please map
the constraint name to its field. Also unmapped: a stock-policy exclusion
clash (23P01) in `syncFromItem`, which would be a raw 500.

## 5 · `DELETE /items/delete` is a toggle  (bug)

* A second call **restores** the item (svc:496-514). The Qt list's "Delete"
  on a deleted row therefore restores it.
* The restore brings back **every** soft-deleted child, including rows your
  own sync removed long ago. That can trip the non-partial indexes in item 3
  after the item row is already restored, and it is not atomic.
* There is no check for transactions: an item with stock or bills can be
  deleted.

Ask: separate delete and restore routes (or an explicit `restore` flag),
restore only the children deleted together with the item, check for
transactions before deleting, and run it in one transaction.

## 6 · Conversion chain drift  (bug)

Only changed conversion rows are saved (item-master-update.service.ts:187-191),
but the chain recomputation (item-unit-conversion.service.ts:640-655) produces
new to-base factors for every row, and only the saved rows get them. Changing
a middle unit's factor leaves the rows above it stale. (Qt sends every row, so
from Qt a row whose computed factor changed counts as changed and is saved.
Another client could still hit this.)

## 7 · Orphan unit refs block every save  (trap)

GET rewrites `ipm_uc_unit_id` / `ean_unit_id` / `ir_unit_id` from the
conversion id to the unit id. A row whose conversion was soft-deleted keeps the
raw conversion id (svc:126-132). Echoing it back is a 400 (update svc:149-165),
and that item can never be saved again until the row is removed. Please
soft-delete the dependent rows when a conversion row goes, or return them
flagged so the client can drop them.

## 8 · Tracking policy: the ITEM always wins over the GROUP  (design question)

`syncFromItem` (svc:549,626) writes an ITEM-scope policy for every item, from
the item's own flags when it has no preset (stock-track-policy.service.ts:70),
and the resolver checks ITEM before GROUP (:121-123). A group's "Tracked as"
preset therefore never reaches an item saved from the item screen. Is that
intended? If the group should govern, skip writing an item policy when the
item has no preset of its own.

---

Nothing here needs a Qt change beyond what is already done. Tell me when 1 and
5 are in and I'll re-run the item round-trip test.
