---
name: erp-nested-dialog-escape-race
description: "A bespoke dialog's Escape handler must decide per EVENT (defaultPrevented + a DOM probe), never from a ref/state — React commits a nested surface's close between two document listeners"
metadata:
  node_type: memory
  type: feedback
  originSessionId: 34eaabef-7963-4c26-9602-0c2eeb4a718f
  modified: 2026-09-26T14:35:19.765Z
---

Hit 2026-09-26 on the User Administration dialog: Escape from the nested
"Copy rights from…" picker closed the picker AND the dialog under it.

**Why:** both listen for `keydown` on `document` (capture). The picker's
handler ran first and called `setPickerOpen(false)`; React flushes a discrete
update in the microtask that runs *between the two listeners*, passive
effects included, so by the time the dialog's handler ran, its
`nestedOpenRef.current` already said "nothing nested" and it closed too.
`stopPropagation()` does not stop other listeners on the same node, and
registration order is whatever the effects' dependency churn made it.

**How to apply:** a nested surface (picker, context menu, confirm) must
`event.preventDefault()` on the Escape it takes, and the owning dialog checks
`event.defaultPrevented` AND probes the DOM for a nested-surface marker
(`[data-erp-nested-surface]`, `features/masters/settings/user-administration/nested-surface.ts`)
before closing — that covers both listener orders. `DeleteConfirmModal`
already marks its Escape (preventDefault + stopImmediatePropagation on
window). Verified headlessly with a keydown-capture / keyup trace; see
[[erp-user-administration-port]], [[erp-client-ui-browser-verification]].
