# Notes 73: a blank company on an item means SHARED, and the item picker must filter by company

2026-10-01. Found when the user could not put **MRP ITEM** on an opening stock line. Two server-side items (A, B).
The item card's half of it (C) is already fixed on the client and is listed only so you know what to expect.

---

## A. Opening stock refuses an item whose company is blank

**What happens**

- Pick MRP ITEM (`01a0f5ff-1e28-75e5-9dab-35fc9b65ff89`, `item_company_id` NULL) on Opening Stock (menu 44) in Acme Foods / Coimbatore.
- `GET /stock/opening/item-lookup?companyId=019c8ea6-…&branchId=019c8ea7-…&itemId=01a0f5ff-…&onDate=2026-10-01` answers **404**:

  > "This item cannot be put on an opening line" — "This item has no company set, so no company can open it; set one on the item master."

**Where**

`modules/stocks/opening-stock-voucher/opening-stock-lookup.service.ts`:

- the WHERE, around line 198:

  ```sql
  AND (NULLIF(${companyId}::text, '') IS NULL
       OR i.item_company_id = ${companyId}::uuid)
  ```

- the explanation, around line 299:

  ```ts
  if (companyId !== null && cause.companyId === null)
      return notFound('itemId', 'This item has no company set, …');
  ```

The comment above the WHERE says that a blank company on an item is "a row nobody owns, not a shared one", and that treating it as shared "would invent a rule".

**Why this is wrong**

1. **The user's rule.** The user decides what a blank company means, and has done so: **a blank company = every company**. It already works this way for ledgers, groups, customers and suppliers. The item card's Company dropdown can be left blank exactly so that an item is shared.
2. **The server already uses that rule elsewhere for items. Only this lookup differs:**
   - `stock-voucher/stock-voucher.service.ts:2557`: `(itm.item_company_id IS NULL OR itm.item_company_id = ${companyId}::uuid)`
   - `selling-price-bulk/price-bucket.gateway.ts:797`: the same rule, for both company and branch.
   
   So today a shared item can be adjusted and priced, but it cannot be opened.
3. **Live data.** 6 live items have a blank company: SUGAR, MRP ITEM, newlama, BILL ITEM, OFFER ITEM, BILL ITEM 1. Every one of them is refused by opening stock.

**Ask**

- **Change the company predicate** to the same shape as the branch predicate below it:

  ```sql
  AND (NULLIF(${companyId}::text, '') IS NULL
       OR i.item_company_id IS NULL
       OR i.item_company_id = ${companyId}::uuid)
  ```

- **Remove the `cause.companyId === null` refusal.** Keep "belongs to another company" for a company that is set and differs.
- **Correct the comment** so it states the rule as "blank company = shared".
- **Check the other stock lookups** that might copy the narrow rule and make them agree with the two services above: physical stock, the adjustment's pick-stock, the sales item lookups.

**Test**

The same call as above should return 200, with MRP ITEM's unit and `trackSignature`.

---

## B. Grid 71 (POPUP - ITEMS) ignores company and branch

**What happens**

Every Qt screen that offers an item sends `iitem_company_id` and `iitem_branch_id` to grid 71. That is 8 callers: sale bill, quotation, sale order, delivery challan, opening stock, physical stock, stock adjustment, change selling price.

Grid 71's SQL uses neither:

```sql
SELECT IM.item_id, CM.iuc_id AS item_uom_id, IM.item_name_en, UM.unit_name
  FROM inventory.item_master IM
  JOIN inventory.item_unit_conversion CM ON CM.iuc_item_id = IM.item_id
  JOIN inventory.item_unit_master UM ON UM.unit_id = CM.iuc_unit_id
 WHERE IM.item_is_active = true AND IM.item_is_deleted = false
 ORDER BY IM.item_name_en
```

So an Acme Foods user's item picker lists **every company's items**. Live counts:

| Company | Items |
|---|---|
| CHANDRA SAW MILL | 10,010 |
| Acme Foods | 214 |
| THE SCM SILK | 13 |
| bana | 1 |
| shared | 6 |

The picker therefore offers another company's item, and the screen's own lookup then refuses it ("belongs to another company").

There are two smaller issues in the same SQL:

- a deleted unit conversion still shows, because there is no `CM.iuc_is_deleted = false`;
- a deleted unit still shows.

**Ask**

Replace the SQL with this version. The tokens are compared as TEXT and never cast, so that '' cannot break the plan (see the grid runner binding rules):

```sql
SELECT IM.item_id, CM.iuc_id AS item_uom_id, IM.item_name_en, UM.unit_name
  FROM inventory.item_master IM
  JOIN inventory.item_unit_conversion CM ON CM.iuc_item_id = IM.item_id
                                        AND CM.iuc_is_deleted = false
  JOIN inventory.item_unit_master UM ON UM.unit_id = CM.iuc_unit_id
 WHERE IM.item_is_active = true AND IM.item_is_deleted = false
   AND ('iitem_company_id' = '' OR IM.item_company_id IS NULL
        OR IM.item_company_id::text = 'iitem_company_id')
   AND ('iitem_branch_id' = '' OR IM.item_branch_id IS NULL
        OR IM.item_branch_id::text = 'iitem_branch_id')
 ORDER BY IM.item_name_en
```

**Before you change it**

- Check the web client. Grid 71 is registered as Desktop, but if a web screen also runs it **without** the two tokens, the unsent tokens stay as literal text and the filter would hide everything. Either send them there too, or confirm nothing else uses 71.
- Check the column `grid_column_filter` flags, so that search still covers `item_name_en`.

**Test**

`/configured-grid-sql/run?grid_id=71&grid_param={"iitem_company_id":"019c8ea6-…","iitem_branch_id":"019c8ea7-…"}&search=MRP` should list Acme's items and the shared ones, and no CHANDRA SAW MILL item.

---

## C. (Client: done) The item card allowed a blank company with a branch set

MRP ITEM was saved with company blank but branch = *Acme Foods – Coimbatore Branch*. A branch always belongs to one company, so "every company, but only this one company's branch" means nothing.

The Qt item card now:

- clears Branch when Company is cleared;
- refuses to save a branch without a company.

**Server ask (optional, belt and braces)**

`/items/create` could refuse `item_branch_id` set with `item_company_id` NULL, and refuse a branch that is not one of the company's.

**Live rows already in that state**

SUGAR, MRP ITEM and newlama each have a blank company and a branch set. They need the user's decision: set the company, or clear the branch. **Not** a data fix for you to apply.
