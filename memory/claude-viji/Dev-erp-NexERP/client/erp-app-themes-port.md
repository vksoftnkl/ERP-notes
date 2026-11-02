---
name: erp-app-themes-port
description: App Themes (menu 266) web port of Qt AppThemeEntry at /settings/app-themes — what differs from Qt and why
metadata:
  node_type: memory
  type: project
  originSessionId: 1134ba40-4fa2-4d1d-a7d9-d767812d7421
  modified: 2026-10-07T07:59:29.530Z
---

Built 2026-10-07 from `/home/vk/VKNexERP/NexERP/src/modules/admin/theme` into
`features/settings/app-themes` (route `/settings/app-themes`, bound under Settings →
Configuration as "App Themes"; menu 266 + grid 125 both ship in the server migration
`20260930190000_app_theme_tokens`, so no live drift).

Deliberate differences from Qt:
- **Template tab cannot Preview** — it is QSS for the desktop client; the web paints only
  from tokens (`lib/app-theme.ts`). The Preview button is disabled there with a tooltip.
- Preview (F9) = `applyTheme(tokens)` on `<html>` without `rememberTheme`; end/unmount
  restores the `/effective` answer. Only tokens mapped in `lib/app-theme.ts` DIRECT show.
- Swatch = native `<input type="color">` (no alpha; an `#rrggbbaa` keeps its alpha suffix).
  ERPColorPicker was rejected: its absolute popover clips inside the scrolling token grid.
- Token catalogue (38 keys, Qt built-ins) lives in `features/settings/app-themes/lib/token-catalogue.ts`;
  `lib/app-theme.ts` THEME_TOKEN_KEYS still lists only v1's 33 (it is the painter, not the editor).
- The QSS editor needs `data-uppercase="off"` or [[erp-global-uppercase-inputs]] rewrites it.

Seeding traps hit while building: see [[erp-rtk-structural-sharing-reseed]] and
[[erp-data-freshness-bus]] (focus refetch must not reseed over unsaved edits).
