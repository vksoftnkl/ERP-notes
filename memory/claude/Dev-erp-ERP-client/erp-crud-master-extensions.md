---
name: erp-crud-master-extensions
description: Opt-in CrudMasterPage props (onDeleteAction, enableListActionKeys, listHintMessage, toolbarActions(row); deleteActionLabel + modalShowValidationErrorPopup 2026-10-02) and the columnRenderOverrides fix
metadata:
  type: project
---

`components/master/crud-master-page.tsx` gained three OPT-IN props, and one
latent bug in it was fixed. All four came out of building the Receipts register
([[erp-receipt-entry-port]]).

- **`onDeleteAction(row) => boolean | Promise<boolean>`** — the page performs
  the delete instead of the shell's `DELETE {delete}?id=`. Needed because
  `POST /receipts/delete` takes its four-field key in a JSON BODY, which no
  url/query override reaches. The confirmation dialog stays the shell's;
  returning false leaves the list untouched and the page owns the message.
- **`enableListActionKeys`** — binds F1 add, F2 / Enter edit, F3 delete, F5
  refresh through the same handlers as the buttons, and prints the legend in a
  hint bar under the grid. OFF by default because these are window-level keys.
  Plain Enter is bound only while the grid itself has focus, which is why it
  does not fight `use-master-list-keyboard`'s deliberate "leave Enter alone".
- **`listHintMessage`** — a ReactNode, or a **function of the selected row**
  (the shell already holds the selection). Renders beside the key legend.
  `ReusableTable` gained a matching `hintBar` slot, drawn between the rows and
  the pager.
- **`toolbarActions`** (2026-10-07) — also accepts a **function of the selected
  row**, so a page's own row verbs can set `disabled`/`title` per row without a
  second selection copy (first user: Temp Credits Receive/Follow-up).

**The fix:** `columnRenderOverrides` only ever reached columns that resolved to
NO shell accessor. So a register that handed over a status-chip renderer got
plain text instead, because `lookupKeys.active` had already claimed that
column — silently, with nothing to debug. The override now wins in both
branches (sorting and searching still read the raw value). This made the chips
in Sale Order, Sale Bill and Quotation start rendering, which is what those
pages always intended.

**Added 2026-10-02** (company/branch Qt parity, [[erp-company-branch-qt-parity]]):

- **`deleteActionLabel`** (default "Delete") — the toolbar button, confirm
  title, confirm button and busy label ("Restoring...") follow it. Used for a
  "Show deleted records" view where the same button restores.
- **`modalShowValidationErrorPopup`** — with `modalHideFieldErrorText` the
  error TEXT is not in the DOM at all (only `aria-invalid` / a red outline), so
  a rule like "GSTIN belongs to state 33 but State is 24" was unexplainable.
  This raises the first failing field's message (field order) as a warning
  popup. Off by default; the other masters are unchanged.
- `features/masters/shared/use-soft-delete-actions.ts` bundles delete/restore
  + Edit/View refusal for soft-deleted lists, and shows a 409's `errors[].message`
  (useApi's popup reads only the generic top-level `message`).

**Added 2026-10-07** (Temp Credits Add/Delete, [[erp-temp-credits-register]]):

- **`hideEditAction`** — only Edit leaves (button, Alt+A/F2, canEdit=false); Add
  and Delete stay under menu rights. `hideWriteActions` still hides all three.
- **`deleteReasonPrompt {label,placeholder,maxLength}`** — a required one-line
  reason box INSIDE the shell's DeleteConfirmModal (`reasonPrompt` prop; confirm
  dead while empty, Enter in the box confirms, box gets focus). Reaches the page
  as `onDeleteAction(row, { reason })`; needs `onDeleteAction`.
- Do NOT add a shell busy state around `onDeleteAction`: Opening Stock resolves
  its promise only after its OWN reason dialog closes, so a busy flag would show
  "Deleting..." behind that dialog. Guard double-submit in the page (ref).

**Do not `git stash` in this checkout** — several sessions share it. I lost the
shell edits that way once and had to pop them straight back.
