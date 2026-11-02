---
name: plans-separate-file
description: A new plan for a change goes in its OWN file; never revise an existing share plan in place unless asked
metadata:
  type: feedback
---

When writing up a change as a plan (React or otherwise), create a **separate file** next to the existing one, for example `plan-react-sale-bill-imports.md` beside `plan-react-sale-bill.md`. **Do not edit the original plan.** The new file states which sections of the original it overrides.

**Why:** on 2026-09-26 I began revising `share/sales/plan-react-sale-bill.md` to REV 3 in place. The user stopped me: "no write a separate plan file do not touch original". The original was restored byte-identical from a copy taken before editing.

**How to apply:** before editing any plan in the share, copy it to the scratchpad. Default to a new file with an "overrides §x of Y" table at the top. Link it from the related memory, as in [[react-sale-bill-plan]] and [[react-port-plans]].
