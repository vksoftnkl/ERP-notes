---
name: erp-modal-onvaluechange-double-run
description: FIXED 2026-08-04 — dynamic-modal-form no longer runs field onValueChange inside setFormData updaters; commit values through commitFormData/replaceFormData so the ref mirror stays accurate
metadata: 
  node_type: memory
  type: project
  originSessionId: 68dafc24-4959-48fd-b98d-ae66451864b8
  modified: 2026-08-04T05:42:21.398Z
---

`ERPDynamicModalForm` (components/design-system/ui/dynamic-modal-form.tsx) used to call `revalidateFieldNames` and `runFieldValueChangeHandler` **inside `setFormData((current) => …)` updaters**. Updaters run during render, so a caller's `onValueChange` that touched parent state threw React's "Cannot update a component (`ItemMasterPageContent`) while rendering a different component (`ERPDynamicModalForm`)", and StrictMode double-ran every handler in dev.

**Fixed 2026-08-04:** a `formDataRef` mirrors `formData`, and all writes go through two helpers — `replaceFormData(values)` (open/reset) and `commitFormData(updater)`, which returns `{previousValues, nextValues}`. Updaters are now pure; validation and `onValueChange` run after the commit, in the event-handler tick.

**Why:** state updaters must be pure. The ref is what makes "latest committed values" readable outside an updater — needed because `applyFieldValueChangeResult` lands in a promise `.then()`, where the render-scoped `formData` can be stale.

**How to apply:**
- Never add a new `setFormData` call — route every write through `commitFormData`/`replaceFormData`, or `formDataRef` silently goes stale.
- Field `onValueChange` handlers now run **once** per edit, dev included. The `itemServiceConfirmMemo` / `itemReorderDuplicateAlertAt` gates in item-master-page ([[erp-item-entry-spec-port]]) are leftover band-aids, no longer required.
- The ref-mirror trips `react-hooks/refs` twice at the `renderField(...)` JSX call sites (false positive — the closures only run from events). `--max-warnings` in package.json was bumped 342 → 344 to match.
