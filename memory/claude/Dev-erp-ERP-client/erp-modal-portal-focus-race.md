---
name: erp-modal-portal-focus-race
description: Dialogs on ModalShell/ModalPortal that focus via requestAnimationFrame can leave focus BEHIND the dialog — use autoFocus
metadata:
  node_type: memory
  type: project
  originSessionId: 61d68e87-a4a8-4159-aa12-3b99eb20915e
  modified: 2026-10-07T05:32:57.060Z
---

`components/ui/modal-portal.tsx` renders `null` on its first commit and portals the content one commit later (its `useEffect` sets `mounted`). A dialog that does `useEffect(() => { if (open) requestAnimationFrame(() => ref.current?.focus()) })` races that second commit; in headless Chromium the frame won, so the ref was null. Focus stayed in the grid cell behind the dialog, and Enter there opened the item picker under a "Discard entry?" question.

Found and fixed 2026-10-07 for stock-adjustment's confirm, stock-pick and cancel-reason dialogs by switching them to `autoFocus`, which React runs when the element really mounts. The item picker in the same folder already used `autoFocus`.

**Why:** "Yes has the focus, so Enter answers it" is a promise these dialogs make, and the keyboard flows (Esc → confirm → Enter) depend on it.

**How to apply:** for any ModalShell dialog that should own focus on open, use `autoFocus` (no jsx-a11y rule in this repo), not a rAF `focus()`. The same rAF pattern probably exists in other features' dialogs (quotation, sale bill, opening stock), so suspect it whenever focus "doesn't move into" a dialog. Related: [[erp-modal-portal-compositing-bleed]], [[erp-client-ui-browser-verification]].
