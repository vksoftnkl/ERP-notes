# notes (68): an item follows its group unless it has its own preset (decision)

2026-09-30. This settles notes 50 #8 / notes 67 #8.

**The user's rule:** an item's stock tracking comes from its GROUP, unless the item names its own tracking preset. Only a preset (NONE included) is an item-level choice.

## What the server does today

`StockTrackPolicyService.syncFromItem` (stock-track-policy.service.ts) writes an ITEM-scope policy when the item has a preset. It **also** writes one when the item has no preset but its own flags "say something" (`saysSomething(derived)`, from `deriveFromItem`). Those flags are:
- `item_is_batch_based`, `item_is_expiry_item`, the MRP / sale-price flags;
- `item_allow_neg_stock = false`, which becomes BLOCK.

Because an ITEM row outranks the GROUP row in the resolver, any of those flags hides the group's "Tracked as".

Found live: a new item with no preset and negative stock off got an ITEM row `N` (a tracking signature of "nothing") that shadows its group.

## Ask

1. **No preset → no ITEM row.** In `syncFromItem`, write or refresh the derived ITEM row only when `itemTrackPresetId` is set. With no preset, retire any derived ITEM row the item has (`retireDerived`), exactly as the `!saysSomething` branch does now. Hand-authored rows (remark not `DERIVED_FROM_ITEM_REMARK`) stay untouched, as today.
2. **Existing data:** retire the derived ITEM rows of items that have no preset. There is 1 live today (signature `M`):

   ```sql
   select p.stp_id, i.item_name_en, p.stp_track_signature
     from stock.stock_track_policy p
     join inventory.item_master i on i.item_id = p.stp_item_id
    where p.stp_scope = 'ITEM' and not p.stp_is_deleted
      and i.item_track_preset_id is null
      and p.stp_remarks like 'Auto-derived from item%';
   ```

   Run the same retire the service would, not a raw delete.
3. **The item flags then stop affecting tracking** when no preset is set. Tracking, expiry and the negative-stock rule come from the group, then the company.

   If a flag is meant to matter for the item alone (e.g. "allow negative stock" on one item), say so. The item screen would then need to show it as an override, the same way it shows the preset.

## Qt side

No change needed. The item screen already treats "Tracked as" as optional when the group has a preset (blank = inherit) and required when it has none.

Note: 0 item groups have a preset on the box today, so nothing inherits from a group yet. Test with a group given "Tracked as" (e.g. BE) and an item in it with no preset: the resolver should answer BE.

## Verified live 2026-09-30 (after Prathap's run)

- **Data:** 0 flag-derived ITEM rows remain on items without a preset (18 → 17 ITEM rows, all preset-backed).
- **Item side:** correct, tested on a throwaway group `ZT-N68-GROUP` (preset BE) and an item in it with negative stock OFF. Both were deleted afterwards.

  | step | ITEM policy row |
  |---|---|
  | no preset | none |
  | preset NONE | `N` |
  | preset cleared | none again |

- **But the item still resolves to `N`, not the group's BE.** Cause: the GROUP row is filed under the LOGIN user's company, not the item's. That is notes 69.
