---
name: erp-print-designer-ui
description: "Print Designer UI feature map — routes, store, canvas unit adapter, verification recipe"
metadata:
  node_type: memory
  type: project
  originSessionId: 07a21bdb-e8a2-4137-892b-d89a14378d40
  modified: 2026-10-01T13:53:59.480Z
---

Built 2026-08-24 in `features/print-designer/` (client). **Styled as a desktop
report designer (QtRPT / Qt Designer), not as the rest of the web app** — menu
bar over icon toolbars, docked panels with title bars, band captions in a gutter
beside the page, a two-column property grid, and a status bar. 11px UI text,
20px rows, square corners: that density is deliberate, and `--erp-radius` is
deliberately unused inside the designer. Users arrive from the Qt desktop app and
should not have to relearn where anything is.

Routes:
`/settings/print-templates` (list), `/print-designer/new`, `/print-designer/[templateId]`.
The LIST is not part of that desktop-designer skin: since 2026-08-25 it rides the
shared `CrudMasterPage` shell like every master page (Add/Edit/Delete/History
toolbar, filter card, configured table, delete confirm), with Add → paper-setup
dialog and Edit/double-click → the designer. Two shell props were added for it:
`clientSideList` (the endpoint returns the whole collection and 400s on
`page`/`limit`/`search`, so the shell filters+slices and, via
`listRefetchIgnoresPaging` in `useMasterModule`, stops refetching on paging or
typing) and `isRowDeleteDisabled` (mirror of `isRowEditDisabled`; shipped
templates refuse both). `printTemplateApi` mutations must also invalidate
`{type:"MasterList", id:"/reports/templates"}` — the shell reads through
`mastersApi`, not `listPrintTemplates`.
It lives inside the normal app shell — Next's root layout applies to every route
group, so there is no "fullscreen" escape; the page uses `height: 100%` under
`.erp-app-content` ([[erp-full-height-page-shell]]).

- **Canvas grid**: rulers, band-caption gutter and page are three columns of ONE
  css grid sharing a row per band, so they cannot drift out of alignment. The
  page is a single white rect spanning all band rows behind the bodies. A band
  caption's content is absolutely positioned so a tall caption cannot stretch a
  5mm band.
- **Property grid**: `panels/controls.tsx` primitives (`Section`, `FieldRow`,
  `FieldGrid`) render the two-column grid; the seven section components feed it
  and know nothing about the layout. Change the primitives, not the sections.
- **Canvas unit adapter** (`lib/grid.ts`): components work in CANVAS UNITS and
  multiply by one `CanvasScale` at the pixel boundary — mm in GRAPHIC mode,
  character cells in GRID. One set of components draws both. See
  [[erp-print-template-contract]] for why GRID units are cells.
- **History** (`store/history.ts`): immer `produceWithPatches`, capped at 50,
  400ms coalescing. Only `definition` edits enter it.
- `data-uppercase="off"` on the designer root is load-bearing —
  [[erp-global-uppercase-inputs]] would otherwise uppercase expressions.
- `lib/vocabulary.generated.json` is written by `npm run gen:print-vocab` from
  the sibling server checkout, and a vitest asserts parity. Not in postinstall:
  a deploy box has no server checkout.

- **Popups off a `.toolbar` are clipped** — the bar is `overflow-x: auto`, so an
  absolutely-positioned dropdown inside it opens invisibly. The top bar's ⋮ menu
  (fixed 2026-10-01) portals to <body> with `.overlayTokens`, pinned via
  `layoutRect` + `Z_POPUP`. Do the same for any new toolbar dropdown.

**Browser verification (2026-10-01):** `/print-designer/new` now redirects to
`/settings/printing/templates/new`; the hosted canvas renders at
`/settings/printing/templates/new/layout` with EVERY `/api/` call faked by CDP
`Fetch.requestPaused` → `{success:true,data:[]}` (no JWT needed). The app header
also has `[role=menu]` nodes — select designer popups as `body > [role=menu]`.

**Older browser verification recipe:** the local dev server runs on **:3001** (HTTPS).
`playwright-core` + `/snap/bin/chromium`, but snap confinement blocks a profile
under `~/.cache` — put `userDataDir` under `~/snap/chromium/common/`. Seed auth by
`addInitScript` writing `sessionStorage.erp_client_redux_state` with a minted JWT
([[erp-auth-endpoint-verification]]).

- **Sheet layout fix (2026-10-13):** `.sheet` was `grid-row: 2 / -1` in a grid
  with no declared rows, so it collapsed into the RULER row and bands sat on the
  grey desktop (page-border guide squeezed too). It is now absolutely
  positioned in the grid (`grid-row: 2 / auto`, `.viewportInner` relative) —
  an explicit span instead would push every auto-placed band cell below it.
  QtRPT-parity additions to this feature: [[erp-print-qtrpt-parity]].
- **Server-defaulted keys may be absent in stored bodies.** The server's zod
  schema `.default()`s many element/pageBorder keys, so a saved `ptv_body` can
  omit them, yet the client types mark them required. RECT/pageBorder
  `radiusMm` missing gave a NaN `borderRadius` (fixed 2026-10-14 in
  `canvasBridge` `withElementDefaults` + pageBorder fill). Any new key the
  canvas does arithmetic on needs a load-time default there or a `??` fallback.
