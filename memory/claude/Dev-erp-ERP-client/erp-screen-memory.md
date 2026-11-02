---
name: erp-screen-memory
description: "Unsaved entries survive menu navigation (Qt MDI stand-in) via lib/screen-memory; per-screen opt-in pattern, slice-screen arming, master popup reopen, and the traps hit building it"
metadata:
  node_type: memory
  type: project
  originSessionId: 66437ea9-1316-4a2e-85fb-4e9ded529103
  modified: 2026-10-16T10:58:04.999Z
---

Built 2026-10-16 because a half-entered bill vanished when the cashier visited another
menu screen (App Router unmounts the page; Qt kept the MDI window). User chose
"remember the draft" (no tab strip), every screen, a small auto-closing "Your unsaved
X was restored." notice, and lifetime = until logout (in-memory, no TTL).

- **Store:** `lib/screen-memory.ts` — module Map keyed user×screen×company×branch×year,
  values by reference, LRU 40, cleared on `AUTH_SESSION_EVENT` for null/other user.
  Hooks in `lib/use-screen-memory.ts` (`useRecalledOnce`, `useRememberWrite`,
  `useForgetIfWritingOnUnmount`). Page passes `memory={{key, resume}}`; resume only
  when the page reopened on a remembered entry, never on Create/Open/deep links.
- **Line documents** share `features/sales/document-memory.ts` (predicate: entry mode,
  not deleted, and holdId || isDirty || unsaved with item lines) and
  `features/sales/hold-lease.ts` (re-verify a resumed hold's lease on return; detach
  on 403/404/409/converted).
- **Redux-slice screens (Sale Bill, Quotation, Sale Order)** need a `memoryArmed`
  ref: the slice still holds the previous visit's (maybe discarded / other-tenant)
  draft until clear/load/resume, and writing it would resurrect it.
  `sliceResumeSource` prefers the live slice draft, else the remembered one.
- **Confirmed Back/Esc = `forgetRemembered()`**; everything else forgets via the
  predicate. A save in flight at unmount forgets (else a late save resurrects → double bill).
- `finishResume` waits one `setTimeout(0)` first: lazy RTK triggers fired from a mount
  effect are unsubscribed by StrictMode's simulated unmount (see
  [[erp-rtk-lazy-trigger-in-mount-effect]]); hold re-read uses `initiate`.
- Default info popups wait for dismissal — the restore notice passes
  `RESTORED_MESSAGE_OPTIONS` (autoClose 2500).
- **Masters:** `CrudMasterPage` remembers which popup was open (`master-modal:<modalDraftKey>`)
  and reopens it on return if `lib/form-drafts` still has its draft; waits for menu
  permissions (`status !== "loading"`), skipped for `hideListPage` embeds. Opt-in per
  page with `modalDraftKey`; pages with lazy-dropdown derived state need
  `onModalDraftApplied` (see [[erp-modal-form-drafts]]).
- Browser check: fake-API CDP driver; global capitalization rewrites typed text
  (REF-777 → Ref-777) — compare against the read-back value. A governed route needs a
  faked `/menu-masters/usermenu` granting it, or the reopen waits on "loading" forever.
- **Popup drafts never hold `password` or `file` fields** (2026-10-16, `omitFormDraftFields`
  in lib/form-drafts, applied on write AND restore) — GSP Provider had leaked a typed
  password into localStorage. Tier B masters opted in: city, item brand/category/group,
  ledger groups, device list; NOT employee (user declined: PAN/Aadhaar would sit in
  localStorage), tax rate (ledger rows live outside the form,
  supersedes name never written) or GSP company service (was blocked by the password leak).
  Custom screens wired: account ledger, item qty price (its router.push "leave?" patch was
  removed — memory makes it moot), module administration, app settings. Skipped: user
  administration dialog (password/PIN + parallel mount loads), promotion loyalty points.

