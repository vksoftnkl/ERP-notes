---
name: erp-message-popup
description: react-toastify is gone — every success/error/warning opens one central popup; raise messages through @/lib/notify
metadata:
  type: project
---

As of 2026-09-21 the ERP client shows application messages as a **centred modal
popup**, not toasts. `react-toastify` is no longer imported anywhere (the package
is still in package.json, unused), `components/feedback/global-toaster*.tsx` are
deleted, and `app/layout.tsx` mounts `components/feedback/message-popup.tsx`.

**Why:** the operator asked for it — a top-right toast covers the header pickers
on the counter screens and can be missed entirely by someone watching the grid.

**How to apply:** raise messages with `import { toast } from "@/lib/notify"` —
the API is toast-shaped (`success` / `error` / `warn` / `warning` / `info` /
`dismiss`, `{ toastId, autoClose }`), so call sites did not change. Never
re-introduce a `react-toastify` import. Behaviour worth knowing:
successes self-close after 2s and everything else waits for Enter/Esc/OK;
messages queue one at a time (oldest first, "N more"); identical text collapses
into one popup unless a `toastId` says otherwise; the popup swallows ALL keydown
in the capture phase except Tab, so screens' F-key and Enter bindings cannot fire
behind it (see [[erp-grid-enter-focus-chain]]).

**Plain-words refusals (2026-10-16, Qt ec9d92d / server notes 120):** a thrown
request error goes through `toast.apiError(err, { kept?, fallback? })` — short
title + line + "Do: …" (server `errors[0].user` block wins, `lib/api/error-text.ts`
table is the fallback), the server's status · code · field and sentence small
and grey (`MessageOptions.title` / `.detail`). Unworded codes keep the server
sentence + every `errors[].message` under a status title ("Cannot save", never
"409"). `kept` = the screen's "Bill not lost." line (4xx only). Inline in a
dialog: `components/feedback/refusal-text.tsx`. See [[erp-qt-bec3f98-port]].
