---
name: erp-app-theme-state
description: Company theme (app_theme_master tokens) — backend committed 0626a3c7; React phase 5 committed 0689692; Qt client has ThemeManager (see nexerp-qt-client-source)
metadata:
  node_type: memory
  type: project
  originSessionId: 84c44cd4-9bd8-4f0e-b23a-37a6a86c2f0c
  modified: 2026-10-01T13:55:17.555Z
---

Company theme plan (share: Prathap/theme/plan-app-theme.md). Status re-checked 2026-10-01:

- **Phase 1 backend** — committed in ERP server 0626a3c7 (pushed): src/modules/settings/appTheme
  (get/effective with ETag+304/save/delete/restore), migrations 20260930190000_app_theme_tokens +
  20260930190100_app_theme_id_seq_resync, company master refuses a dead theme. Dev ids: menu 266
  "App Themes", grid 125 "MAIN LIST - APP THEMES" — both SEQUENCE-allocated by the migration, so the
  live box may have different ids (look them up there before a client hardcodes them).
  Tests: app-theme.service.spec 5/5, test/app-theme.e2e-spec.ts 5/5 (covers plan §8 live list).
- **Phase 5 React** — committed 2026-10-01 as 0689692 on `dev` in `/home/vk/Dev/erp/ERP client` (not pushed):
  lib/app-theme.ts (+test, 12/12), components/layout/app-theme-controller.tsx, store/api/appThemeApi.ts,
  layout.tsx bootstrap script, erp-header.module.css vars. tsc clean.
- **Qt phases 2–4** (§5–§8) — Qt source appeared 2026-10-02 at /home/vk/VKNexERP/NexERP
  ([[nexerp-qt-client-source]]). Its HEAD 09f8896 already has src/core/theme/theme_manager.{h,cpp} +
  theme_tokens.h (fill(), applyEffective(), QSettings cache, /app-themes/bootstrap at login) and
  api_endpoints.h AppTheme routes incl. Template/TemplateSave — the client team built it; check before redoing.
  Theme sweep 2 plan: share `Prathap/theme/plan-app-theme-sweep-2.md` (steps 2–6 Qt-side).
- **Part 2 (template in the DB), server half** — built 2026-10-01, uncommitted: migration
  20261001140000_app_theme_template (table app_theme_template, seeded from share theme/seed-template-v1.qss),
  5 v2 token keys (38 total), GET /app-themes/template, POST /app-themes/template/save (409 on stale
  tplModifiedOn), template on /effective (+ETag), PUBLIC GET /app-themes/bootstrap. Repo status:
  plan/app-theme-template-server.md. MAROON corrections / v2 row values are the client team's (via /save).
  React lib/app-theme.ts mirrors 33 keys (its test pins 33) — not updated.

**How to apply:** if asked to continue, don't redo phase 1/5; Qt work goes in [[nexerp-qt-client-source]]; read its theme_manager first.
Related: [[erp-live-vps-deploy]].
