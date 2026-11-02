---
name: counter-screen-deferred
description: User decision 2026-10-17 — Counter screen (Till mode) is SKIPPED for now; built later, after sales return exists
metadata:
  type: project
---
User 2026-10-17: "skip counter screen because we haven't had any dc / sales return in our code — later we can run this module". The one-screen Counter (Bill/Return/DC/Order/Quote) waits until the pieces it combines exist. Note: the Qt DC screen (menu 182) IS built; the Qt sale-return screen is not.

**Why:** the counter merges all five sales txns; building it before return exists means building it twice.

**How to apply:** don't propose the Counter screen, till strip or F11 menu as next work until the user brings it back. Till work continues on office-side items (reports, Day Summary tabs, HO dashboard, untested live paths). Related: [[counter-screen-analysis]], [[till-lives-in-counter-screen]].
