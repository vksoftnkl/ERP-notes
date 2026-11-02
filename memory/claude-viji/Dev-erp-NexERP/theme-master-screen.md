---
name: theme-master-screen
description: menu 266 App Themes screen BUILT 2026-10-01 (src/modules/admin/theme) — contract facts, template v7 live, tests/apptheme 40+/0
metadata:
  type: project
---
Built from the approved mockup (share/theme/theme_master_mockup.png) on 2026-10-01.
Files: src/modules/admin/theme/app_theme_entry.{h,cpp,ui} + app_theme_widgets.{h,cpp} (QssEditor w/ gutter + {{key}} highlighter, ThemePreviewCard painted from the edited tokens). Registered fm.reg<AppThemeEntry>(266) in form_setting_menu.h. Token catalogue (key/group/meaning) = ThemeTokens::catalogue() in theme_tokens.h. ThemeManager gained preview(tokens, tpl) / endPreview(companyId) / isPreviewing().

Contract facts found while building:
- grid 125 via /configured-grid-sql/run — `limit` max 100 (500 = 400).
- grid runs are cached ~1 s: after delete/restore/save the screen patches m_list locally and reloads 1.5 s later (reloadListSoon).
- /app-themes/delete and /restore are POST with ?thmId= query; save sends the WHOLE tokens object (it replaces).
- Template v7 = v6 + a `QDialog#AppThemeEntry` block (before Part C); published live 2026-10-01 13:33Z, md5 matches share/theme/seed-template-v7.qss.

Tests: tests/apptheme/run.sh (APPTHEME_SNAP=dir, APPTHEME_QSS=seed to preview an unpublished template). Creates + deletes ZT-THEME-QT each run; never saves the template.
Build note: plain `ninja` is NOT on the shell PATH — use /home/vk/Qt/Tools/Ninja (and Tools/CMake/bin for a reconfigure after adding files); the runners set it.
Related: [[theme-template-decision]], [[company-stylesheet-3-0]], [[current-line-marker]].
