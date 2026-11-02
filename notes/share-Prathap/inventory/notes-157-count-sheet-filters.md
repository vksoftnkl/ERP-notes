# notes-157 — Physical Stock count sheet: filter by brand, category, section

Date: 2026-10-26. From: Viji / Claude. To: Prathap.

The user wants Physical Stock's **Load Sheet** to narrow the sheet the way a shop counts:
- by item group, which you already have;
- by **brand**, **category** and **section**;
- by **only negative stock**.

**Asked:** `GET /stock/physical/count-sheet` should take three optional query params:

| param | filters on |
|---|---|
| `itemBrandId` | `item_master.item_brand_id` |
| `itemCategoryId` | `item_master.item_category_id` |
| `itemSectionId` | `item_master.item_section_id` |

Each param narrows only when present, ANDed with `itemGroupId` and `bucket`.

Today the DTO (`GenerateCountSheetQueryDto`) rejects them as unknown fields, so the screen shows the three dropdowns **disabled** with a tooltip. When this is live, I turn them on (`PhysicalSheetFilterDialog::kServerFiltersLive`).

**Optional, no rush:** `negativeOnly=true` (book quantity < 0). The screen already filters negative-only on the rows the route returns, so it works today. A server filter would only save paging through big godowns: the route caps at 1000 rows per page.
