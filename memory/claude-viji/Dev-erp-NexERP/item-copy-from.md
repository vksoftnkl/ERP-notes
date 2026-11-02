---
name: item-copy-from
description: Item card "Copy from Item" (new items only) + no Save-as-Template in edit mode; what a copy keeps and drops
metadata:
  type: project
---

Built 2026-10-22 (user: "stop save as template in edit mode"; "copy from another item, load all details, user changes the name").

- **Button `btnCopyFromItem`** in the bottom bar (everyone; NoFocus, autoDefault off), hidden in edit mode. Picker = small QDialog with NexDropdownSingle dropdown **42** (the Qty-wise Price screen's item list) + Copy/Cancel, EnterNavigation. Asks before replacing a form that was already typed into (collectJson != m_baseline).
- **ItemEntry::loadItem(id, asCopy)** = the old edit() body; edit() = set m_itemId + hide btnSaveTemplate / btnCopyFromItem + loadItem(id,false). Copy error → message only (the form is NOT closed).
- **detachAsCopy():** no item_id, no row ids (iuc/ipm/ir cleared with signals blocked; `ItemPriceGridController::detachFromStored()` clears ipm ids + the ↑/↓ baseline), base unit unlocked; NOT copied: item code, barcodes (field + grid — EAN unique across items), photo, packing items. Name kept, focused + selected.
- **Template:** button hidden in edit mode AND `saveAsDefaultTemplate()` returns when m_itemId is set.
- Test: tests/itemcard SRC=test_item_copy (read-only; `LIVE_SAVE=1` saves one ZT-COPY item, verifies, deletes). Box data: biscuit's reorder row has a godown of another branch → server 400 ITEM_REORDER_GODOWN_OUT_OF_SCOPE on ITS OWN save too; the live test drops reorder rows. A saved card closes + deletes itself — tests must not touch it after `saved`.
