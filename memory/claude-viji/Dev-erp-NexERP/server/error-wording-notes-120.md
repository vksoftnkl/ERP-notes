---
name: error-wording-notes-120
description: "Notes 120 built 2026-10-15 — plain error wording lives on the server: fixed.error_message + catalogue version, prisma/seed/Error_Messages.json loaded by TS seed error-messages, `user` block added by every exception filter, GET /api/v1/error-messages"
metadata:
  node_type: memory
  type: project
  originSessionId: 17534719-2eec-463b-abb4-2356cb12087b
  modified: 2026-10-15T14:33:29.852Z
---

Migration `20261015130000_error_messages_notes_120` (fixed.error_message: code × variant × lang; fixed.error_message_catalogue: one version row). `src/common/error-wording/error-wording.ts` is a DI-free in-memory registry; `ErrorMessageService` (src/modules/error-messages) loads it at boot and reloads within 60 s of a new catalogue version. Every filter writes through `wordedResponse()` (ModuleExceptionFilter — so Till and all module filters — AllExceptionsFilter, and the 5 hand-written filters); `ErrorWordingInterceptor` (APP_INTERCEPTOR) words `data.warnings[]` / `data.refusals[]` of success bodies.

Resolution: exact code before a prefix row (`HOLD_*`); within one, `field=` first, then any `<key>=<value>` on the error, then `<key>=*`, then the plain row; a language with no row → en. No row → no `user` block, never the developer sentence.

**Why:** the user's call (10-15): one copy of the wording for Qt, React and mobile; Qt's `api_error_text.cpp` table becomes a fallback.

**How to apply:** the seed file is the WHOLE approved catalogue — rows missing from it are deactivated. To load a new wording file: replace `prisma/seed/Error_Messages.json` (bump `version`), then `npm run seed:run -- --only=error-messages` (boot runs it only under DB_AUTO_SEED = production). The 2026-10-15.1 file was built from the Qt cpp the user pasted (110 rows; the share's own JSON, 109 rows, was not on this box). .3 added sales (notes 123); .4 (notes 124, receipt/payment/cheque REV 1) added 61 VCH_ + 26 ACC_ + 10 CHQ_ rows. The REV 1 doc left 2 questions open (VCH_BACKDATED todo, VCH_TDS_DEPOSITED "TDS return correction"), loaded as drafted. See [[error-codes-notes-119]].
