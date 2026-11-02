---
name: charge-auto-apply
description: "How chgAutoApply seeds the charge grid on a new/cleared document, and the guards that keep it off loaded documents"
metadata: 
  node_type: memory
  type: project
  originSessionId: a00e4b3d-7fec-4c75-a81a-54925d7dd66f
  modified: 2026-09-11T06:24:48.878Z
---

`ChargeGridController::applyAutoCharges()` (added 2026-09-11) seeds tblCharges from
`charges/get?chgModule=<module>` — the module's charges **plus** the `B` (both)
ones — keeping the `chgAutoApply` rows, ordered by `chgDispOrder` then name.

Called from two places per host: the deferred constructor tail (first blank
document) and `resetScreenState()` (Clear, and the reset after a save). Wired on
**quotation** and **sale bill**; the **sale order has no call yet**.

**Why it is safe even though a reset is also the prelude to a load/import:**
- no-op once the grid holds any charge row (`hasAnyChargeRow()`);
- every load/restore/import path calls `populate()`, which does `setRowCount(0)`
  first, so a seed never survives onto a loaded document — *including one with no
  charges at all*;
- the module list is cached per session, so from the second document on the fill
  is synchronous and there is no race;
- lookup failure is deliberately silent (no modal on screen open).

`autoApplyCharges()` is a **public static pure function** purely so the filter and
ordering can be tested with no server — the endpoint needs a bearer token, and
`ApiClient::handleError` raises a modal, so screen-level tests hang in a headless
harness without credentials.

On the live master a sales document seeds 3 rows: `CD - AFTER TAX`,
`LOADING CHARGE`, `New Reg` — all default rate 0, so they contribute nothing until
a rate is typed. A seeded LOADING/FREIGHT row also satisfies the save-time role
gate that fires when chkHasLoad/chkHasFreight is on.

Related: [[charge-distribution-model]], [[place-of-supply-split]].
