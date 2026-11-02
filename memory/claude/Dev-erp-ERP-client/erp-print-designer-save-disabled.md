---
name: erp-print-designer-save-disabled
description: "Print-designer canvas Save stays grey for 3 reasons — published rev (no tooltip), a client lint ERROR, or not dirty; how to find which"
metadata:
  node_type: memory
  type: project
  originSessionId: 7bd3601a-2ffe-4893-9572-ebe5f66b9fde
  modified: 2026-10-30T04:56:30.989Z
---

Canvas Save = `dirty && !isSystemTemplate && errors === 0`, AND `!host.readOnly`
(`useTemplateSave`). Only the error case gets a tooltip; a PUBLISHED revision
greys it silently (only DRAFT is `ptvIsEditable`).

2026-10-30, TillRemitA5 (a DRAFT): Save never enabled because the client
expression linter (`lib/expression.ts`) read jexl's logical `||` as a transform
pipe → `visible: "{{ header.bag_no || header.seal_no }}"` linted as "unknown
transform 'header'" (error). Fixed with `ENDS_IN_PIPE = /(?<!\|)\|\s*$/`. The
server (jexl AST in `expression.validator.ts`) always accepted it.

**How to diagnose fast:** GET `/print-templates/get?ptlId=` with a minted token
([[erp-auth-endpoint-verification]]), then a throwaway vitest that runs
`toDesignerDraft` → `toTemplateDefinition` + `toProviderDescriptors` →
`validateDefinition` and prints `severity:"error"` problems — exactly what the
canvas computes. Known leftover: TillMovement80 rev 1 DRAFT has a real error
("'t73' has no value"). Related: [[erp-print-designer-ui]].
