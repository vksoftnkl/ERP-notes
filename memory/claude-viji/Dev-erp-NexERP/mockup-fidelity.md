---
name: mockup-fidelity
description: An approved PNG mockup is the visual spec — build the screen to look like it, and name every deviation out loud before handing over
metadata:
  type: feedback
---

When a screen has an approved mockup, the built screen must LOOK like the mockup. That covers:
- the title and status treatment;
- the header rows;
- the grid's columns and their order;
- in-cell details (chips, "auto · FEFO", greyed engine values, the row-note column, the blank-row prompt);
- card headings and layout;
- button set and order;
- inline panels.

It is not enough to "have the same features". House habits (blue title, big red status, group boxes, always-visible Edit/Cancel) do NOT override the mockup.

**Why:** 2026-09-30 the user asked "why stock adjustment entry not same as like in the mockup ui". I had built menu 264 function-first, with house styling and the server's registered columns, and never listed the differences.

**How to apply:**
- Before coding, crop the mockup's panels at full resolution and build to them.
- Before handing over, render snapshots (`ADJ_SNAP` / snap.sh, see [[qt-offscreen-test-harness]]) and compare them side by side with the mockup.
- Any deviation forced by the server or by behaviour must be stated in the reply. Examples: Remarks on re-lot because its reasons require it; Edit/Cancel shown only for a saved document.
- Keep that "why" list short, and give it without being asked.
