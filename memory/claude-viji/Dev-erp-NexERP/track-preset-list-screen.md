---
name: track-preset-list-screen
description: "Tracking preset list — grid 98 + menu 249 (TrackPresetList), read-only because presets have only a GET route; and the fact that grid/dropdown registration goes through the API, not the DB"
metadata:
  type: project
---

Built 2026-09-07: **grid 98** `MAIN LIST - TRACKING PRESETS` over
`stock.stock_track_preset` (reference copy: `grids/stock_track_preset_grid.sql`
in the share) and the client screen `TrackPresetList`
(`src/modules/inventory/master/track_preset/`), registered on **menu 249** in
`form_inventory_menu.h`.

**Registration goes through the API, not psql.** The `claude` DB role is
SELECT-only on `fixed.*`, but `POST /api/v1/grid-details/create` (bearer token
from `POST /api/v1/auth/login`, **VKPOS / VKPOS**, user id
`019e441b-…b857ffb35db1` — the same id that created grids 95/97) writes the
grid and its columns happily. `replace_columns: true` makes the POST
idempotent; re-POST WITH `grid_id` to update. This supersedes the "blocked on
DB write access" note in [[track-preset-dropdown]] — the same route exists for
dropdowns. **Stored `grid_sql` carries NO comments** (95/96/97 hold none
either): the runner appends LIMIT/OFFSET to the last line, so a trailing `--`
would swallow it. Keep the annotated copy in the share file only.

**The screen is READ-ONLY and that is deliberate** — the backend has only
`/stock-track-presets/get`; no create/update/delete exists, so `showAdd` /
`showEdit` / `showDelete` are false. A NexCrudForm entry dialog was written and
then deleted on the user's instruction ("we don't have api right now"); rebuild
it when the write routes land. `auditScreenId = 0` too: audit screen 92 is
Stock Track **Policy**, a different table.

**Two mismatches to settle:** menu 249 is named *"Stock Track Policy"* in
`fixed.menu_master` but now opens the *preset* list — rename the menu row, or
move the presets to their own menu when the policy screen is built. There is no
menu-master create/update endpoint (only get/usermenu/visibility), so the
rename is Prathap's or the admin screen's.

See [[stock-schema-design]] for the preset/policy doctrine and
[[track-preset-dropdown]] for dropdown 49 over the same table.
