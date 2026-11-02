---
name: erp-sale-bill-lifecycle-port
description: "Sale Bill (menu 12) React port is being reworked in phases against docs/plan-react-sale-bill.md REV 2 (2026-09-25); phase 1 landed 2026-09-25; since 2026-10-13 Post = validate → ONE /bills/create post:true (notes 106)"
metadata:
  node_type: memory
  type: project
  originSessionId: c87a4aed-e336-4018-9420-5a8ee840bf40
  modified: 2026-10-06T18:16:00.000Z
---

The full plan is `/home/vk/Dev/erp/docs/plan-react-sale-bill.md` (32 sections, REV 2, 2026-09-25). It supersedes the older sale-bill and sale-tender plans; user chose "rework `features/sales/salebill` in place, phase by phase, commit after each phase" (2026-09-25).

**Phase 1 done (2026-09-25):** `BillKey` + lifecycle DTOs in `salebill.types.ts`; endpoints `/bills/{validate,post,amend,cancel,delete,...}` in constants; `salebill.dto-keys.ts` = server DTO whitelists (payload tests assert no key outside them); `salebill.notes.ts` (parse refusals/warnings from every holder incl. 422 `errors[]`); `salebill.verbs.ts` (`verbState` per §17.1, cancel/amend blocked reasons per §17.2); hook verbs `post/amend/beginAmend/cancelBill/deleteDraft/validateOnServer` + `confirmProceed` popup; `warning-strip.tsx`; toolbar is now a pure render of `verbState`.

**Post is ONE call since 2026-10-13 (Qt 6f15977, server notes 106):** validate on the screen payload → popup → `/bills/create` with `post:true` + `overrides` (`buildCreateAndPostBody`). `/bills/post`, `postBill`, `PostBillDto` and the adopt/discard auto-post draft machinery are GONE from salebill + testbill. A refusal writes nothing. Plain Save (not auto_post) needs `maySaveDraft`. See [[erp-qt-6f15977-port]].

**Wire facts learned (not in the plan text of the code):**
- `/bills/delete` is DRAFT-only delete; cancelling a bill is `/bills/cancel {keys, reason}` — the old client comment "delete cancels the source order" was wrong.
- Line trail: `srcDocId` = document, `srcItemId` = line (`soi_id`/`sqi_id`); `sbiSrcItemId` + `sbiSrcDocLineNo` must be sent. `cancelOrderLine` sends `srcItemId`.
- `sbDeviceId` must be the login's registered device (`actor.deviceMasterId`), else post refuses `SALES_DEVICE_UNREGISTERED`; hook refuses to post without it.
- `sbAdvanceAmt` = ADVANCE rows only, `sbNoteAdjAmt` = credit-note rows; `sbiBucket` never null (`SALEABLE`); `sbStatus`/`sbVersionNo`/`sbCounterId` never sent; `sbDiscAlterBase` never null.
- `tenderRoute` reads `sales.tender_type` (all_bills / cash_bills on a non-credit term) and holds WHILE AMENDING too (user 2026-10-06, = Qt): Tender (F5) re-settles inside the amend, "Save changes" saves without the dialog, F6 = Save changes & Print.
- Tender DTO: received = keyed + surcharge, total = received − change, `tdAmount` = total − surcharge; load restores `keyed = tdAmount + tdChangeAmt` (same as sale order).

**Tender route decision (user 2026-10-06):** F5/Save on the tender route opens the settle dialog WITHOUT any validation — no client checks, no `/validate` (plan §15.1 said validate first; that is superseded). Every check runs in the `save` that fires off the dialog's OK. `validateForTender` and the `openingTender` validation flag were removed from `features/sales/salebill`; `features/sales/testbill` still carries the old validate-before-tender flow.

**Qt vs React gap list (2026-10-06):** `/home/vk/.claude/plans/misty-napping-meadow.md` — §1a five bugs FIXED that day (tender change double-count, GST split rounding in shared `domain/pricing`, Ctrl/Shift F-key fall-through, `storedPricing` on load, party-context race); §1b items 6–22 re-checked and confirmed in code but NOT fixed; §4 lists the deliberate differences not to "fix".

**Remaining phases (plan §30):** phases 1–4 done; 5 imports/carry/shipping/GST and 6 re-tender/holds/lists not built in `salebill` (they exist in `testbill`). User has not yet answered §28 decisions Q1–Q12 (plan assumes the first option).

**How to apply:** read the plan section before touching a feature; keep `salebill.dto-keys.ts` in sync when the server adds DTO keys; never derive can-amend from status.
