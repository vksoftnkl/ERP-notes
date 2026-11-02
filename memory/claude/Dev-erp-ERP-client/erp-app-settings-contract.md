---
name: erp-app-settings-contract
description: /app-setting-values contract traps — value is raw TEXT (not typed), source is OVERRIDE|DEFAULT (not the layer), and the write rules for asvId/scope ids
metadata:
  type: project
---

`/app-setting-values/*` (server `src/modules/settings/appSettings`), verified live 2026-08-25.
The React screen is `features/settings/app-settings` at `/settings/app-settings` (DB menu 246
"app settings", under Settings > Configuration).

**Read `/effective` — two things the plan doc got wrong and the DTOs don't spell out:**
- `value` comes back as **raw text**, never typed (`"true"`, `"4"`). The client casts by `asdDataType`.
- `source` is **`OVERRIDE | DEFAULT`**, *not* the scope. Which layer set it lives in
  `override.asvScope` — and only that answers "may this screen edit/reset it".
- Data types include `UUID` and `DATE` beyond BOOL/INT/DECIMAL/TEXT/JSON. Any type may carry
  `asdAllowedValues`, so test the allowed list before the type when choosing a control.

**Write `/create` (`{data:[…]}`):**
- `asvValue` must be a **string** whatever the type (the trigger tests `asd_allowed_values @>
  to_jsonb(asv_value)`, a jsonb array of strings).
- Send `asvId` **only when `override.asvScope === the scope being edited`**. Without it the server
  upserts on the scope target (that is how you create a deeper override); a client-minted uuid
  400s with `No override found with id …`.
- Exactly **one** scope id, the one the scope names; the others `null`. Deployed
  `ck_asv_scope_ids` is one-id-per-scope — `10_app_settings.sql` in the share still carries an
  older cumulative version and is wrong.
- Extra trap not in the plan: on the **update** path (`asvId` sent) `ensureTargetIsUnchanged`
  compares every supplied id against the stored row, so a mismatched id 400s
  ("cannot be changed"). Sending the bar's own ids is safe because `/effective` only returns an
  override matched on the id that was sent.
- Reset is `DELETE ?asvId=` (soft delete), never a write of the default value.

Related: [[erp-menu-href-mapping]], [[erp-auth-endpoint-verification]], [[erp-full-height-page-shell]].
