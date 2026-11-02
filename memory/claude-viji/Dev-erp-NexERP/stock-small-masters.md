---
name: stock-small-masters
description: Stock small masters 10-26 — all three BUILT + live: Stock Reasons (293, grid 188), Ageing Slabs editor (294), Tracking Presets Add/Edit/Active (249); notes-159 live
metadata:
  type: project
---

User 10-26 "write small masters"; notes-159 server routes live ("ran 159").
- **Stock Reasons** (97a8e59): `src/modules/inventory/master/stock_reason/`, grid 188, fm.reg(293). Shared row → Edit = company copy (same code).
- **Ageing Slabs** (d27045b): `master/ageing_slab/` AgeingSlabEntry (NexDialog in code, theme v40 rules), fm.reg(294). GET /stock/ageing-slabs → {usingOwn, slabs}; Customise = edit a copy of the default; save posts the whole ladder; revert deletes the company rows. From day = previous To + 1 (derived).
- **Tracking Presets** (d27045b): TrackPresetEntry (New/Edit/Copy) + list Add/Edit/Active. POST /stock-track-presets/save + /deactivate. A shared row opens a Copy (code locked). Track ticks lock when usedByItems+usedByGroups > 0. valuation derived (LOT_ACTUAL if tracked, else WAVG).
- Test: `SRC=test_small_masters EXTRA_LIBS=-lQt6Test tests/stockreason/run.sh` 21/21. Leaves inactive ZT_PRE_* presets (no delete route); ageing reverts to 0 company rows.
- Trap: a test's modal closer must skip the NexCrudMasterPage itself (it is a QDialog) or it accepts → deletes the page.
