---
name: theme-shipping-workflow
description: How a new app theme template version reaches every server — publish on dev through the API, then ship the same text as a guarded migration (decided 2026-10-30, first used for v48)
metadata:
  type: project
---

**Decided 2026-10-30 (user asked "migration or API, which is right").** Both, for different jobs.

1. **Draft** `share/theme/block-<screen>-vNN.qss` + `seed-template-vNN.qss` (= the live text + the block, nothing removed). Preview offscreen with tests/apptheme `APPTHEME_QSS=`.
2. **Publish on dev through the API**, after the user's OK:
   - `POST /app-themes/template/save` as VKPOS, with `tplId`, `tplQss`, `tplModifiedOn` taken from `GET /app-themes/template` (an optimistic stamp) and `tplRemarks`.
   - The API validates placeholders, braces and size, refuses on a stale stamp, and writes an audit row.
   - Read the template back and compare the md5 with the file.
3. **Ship it as a migration**, `ERP server/prisma/migrations/<ts>_app_theme_template_vNN`. Model: `20261030190000_app_theme_template_v48`.
   - A `DO $mig$` block holding the exact text in `$qss_vNN$`.
   - It replaces the active template ONLY when md5(current) is a version we shipped: the original seed in 20261001140000, plus every `share/theme/seed-template-v*.qss`.
   - A company-customised template is left alone with a NOTICE.
   - It never RAISEs (P3009, [[live-vps-sync-migration-p3009]]) and sets `tpl_modified_on = now()`, so client caches and ETags refresh.
   - On dev it is a no-op ("already holds vNN").

**Why:** until v48, v2..v47 were published only through the API on the dev box. Only migration 20261001140000 (the v1 seed) existed, so nothing carried later versions to the live VPS. That is the dev-only config drift in [[erp-live-config-drift]], under client/.

**How to apply:**
- Build the fingerprint list from the share files with Python md5, and put each comma BEFORE its `-- name` comment. On 10-30 the commas sat inside the comments; SQL then glued the strings into one and nothing matched.
- Test all paths on a `pg_temp.app_theme_template` copy inside BEGIN/ROLLBACK as `claude`: an older version upgrades, vNN is a no-op, a customised text is left alone.
- Related: [[theme-template-decision]], [[theme-master-screen]].
