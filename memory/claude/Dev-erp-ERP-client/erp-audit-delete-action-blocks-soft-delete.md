---
name: erp-audit-delete-action-blocks-soft-delete
description: "Soft deletes are audit-logged as 'cancel', never 'delete' — audit.audit_log_action has no 'delete' member and normalizeAction() 400s on it. (Was a live bug in the quotation module until ~Aug 2026; fixed.)"
metadata: 
  node_type: memory
  type: project
  originSessionId: a756e224-1e96-4bb6-95f6-738f98b7336b
  modified: 2026-09-16T13:43:31.947Z
---

`ERP server/src/modules/audit-log/audit-log.service.ts` → `normalizeAction()` accepts only
`new | insert | update | approve | cancel` (`new` maps to `insert`) and throws
`BadRequestException("Unsupported audit action: <x>")` for anything else, because
`audit.audit_log_action` has no `'delete'` member.

**So every module logs a soft delete as `action: 'cancel'`.** Follow that in any new soft-delete
path; passing `'delete'` is a 400 at runtime, not a compile error.

**Status: FIXED — do not re-report this as a bug.** It was live until roughly Aug 2026: the
quotation service passed `action: 'delete'` in three places, so `DELETE /quotations/delete` and
any update that dropped a line or charge returned 400. All three now pass `'cancel'`, each with
a comment saying why (`quotation.service.ts` ~L599 header, ~L1251 item, ~L1518 charge).

Re-verified live 2026-09-16 against quotation `Q0043`: removing a line from a saved quotation
through the Size Entry dialog and re-saving returns **201 "Quotation updated successfully"**;
the dropped row lands `sqi_is_deleted = true` with an `audit.audit_log` row
`log_action = 'cancel'`, `log_notes = 'Quotation item soft deleted'`.

If a soft delete 400s somewhere else, check that module's own `action:` argument first — the
audit service is not the thing that is wrong.

Related: [[erp-quotation-entry-port]], [[erp-quotation-multi-size-entry]].
