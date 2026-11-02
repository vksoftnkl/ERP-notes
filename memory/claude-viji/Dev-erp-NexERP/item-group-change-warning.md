---
name: item-group-change-warning
description: "Decided 2026-09-05: item master must warn on group change when the item's effective tracking signature would flip; needs a backend signature-preview query"
metadata: 
  node_type: memory
  type: project
  originSessionId: 28c2bb21-93af-4cd5-a729-6aea4637de01
  modified: 2026-09-05T05:33:58.740Z
---

User decision 2026-09-05: the **item master screen warns when the item's
group is changed**, because `fn_stp_effective` reads `item_master.
item_group_id` LIVE (16_stock.sql:2633) — moving groups silently re-tracks
the item from the next posting, while existing lots keep their identity
(nothing is re-keyed; old buckets sell through at their own smp prices, or
are folded in via the RELOT_OUT/RELOT_IN pair).

**Why:** group membership is not stamped anywhere in stock; even back-dated
documents resolve with TODAY'S group. An ITEM-scope policy row makes the
move a no-op (ITEM beats GROUP).

**How to apply:**
- Trigger = "effective signature would change" (resolve via old group vs new
  group), NOT "new group has a policy". Never warn when an ITEM-scope row
  exists; do warn when the new group has no row and the fallback flips the
  signature.
- Two severities: signature change + item holds stock in lots not matching
  the new signature → real warning (sell-through or relot; warn, don't
  block). Signature change with no stock → mild note or silent.
- Backend needed (Prathap): (1) a signature-preview — fn_stp_effective
  overload taking a group override, or a 16q query replicating its ORDER BY
  against a candidate group; (2) a "holds dimensioned stock" check (lots
  with any non-sentinel key, qty on hand).
- The group-master side of the same hazard is already designed: the
  item_group_ui_mockup's "locked-by-stock" combo state.

See [[stock-schema-design]]; related screens in
[[change-selling-screen-build-notes]].
