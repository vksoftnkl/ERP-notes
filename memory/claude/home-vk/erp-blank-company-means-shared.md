---
name: erp-blank-company-means-shared
description: "ERP rule set by the user: a blank company (or branch) on a master row means SHARED by every company / branch — items, ledgers, groups, customers, suppliers"
metadata:
  node_type: memory
  type: feedback
  originSessionId: 84c44cd4-9bd8-4f0e-b23a-37a6a86c2f0c
  modified: 2026-10-01T06:53:52.533Z
---

A NULL company on a master row means "every company may use it", not "nobody owns it". The same goes
for a NULL branch, which means "the whole company". This is the user's rule (notes 73, 2026-10-01). It
already held for ledgers, groups, customers and suppliers. Items were the exception in the opening-stock
lookup, `/items/bulk-load` and grid 71, and all three were fixed on 2026-10-01.

**Why:** the item card leaves Company blank on purpose so an item is shared. A strict
`company = $1` filter refused all 6 shared items on opening stock. An earlier comment in the code
called treating NULL as shared "inventing a rule"; the user overruled that.

**How to apply:** any item/master filter scoped to a company or branch is
`(param IS NULL OR col IS NULL OR col = param)`. Never use strict `=`. A branch always belongs to one
company, so blank company + set branch is invalid (refused on item save since notes 73 C).
Optional grid tokens: `NULLIF(NULLIF('itoken',''),'itok' || 'en')` stays inert when the token isn't
sent, because the grid runner binds quoted tokens but never touches the split literal. See
[[erp-notes-70-item-masters]].
