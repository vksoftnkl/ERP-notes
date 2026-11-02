---
name: dont-test-every-change
description: "run the live offscreen suites once at the end of a piece of work, not after every edit — the user called it out as too slow"
metadata:
  type: feedback
---

**The user, 2026-09-18: "do not test every time it take too much time."**

Each `tests/*/run.sh` relinks the whole app's objects and then drives a live
screen against 192.168.0.106 — tens of seconds to minutes apiece. I was running
two or three of them after every individual edit, several times in a row, which
is most of the wall-clock in a session and almost none of the information.

**How to work instead:**
- `ninja -C build/Desktop_Qt_6_12_0_Debug` after an edit. That catches the
  typo, the wrong overload, the renamed member — which is what most edits risk.
- Run the live suite ONCE, when the piece of work is finished, before
  committing.
- Run a suite mid-way only when a specific behaviour is genuinely in doubt and
  the answer changes what I write next — and then run the ONE suite that covers
  it, not all of them.

**Why:** compilation and behaviour are different questions. Re-answering the
behaviour question after a change that could not have altered behaviour is
waste the user is paying for in time.

Related: the runners build first now ([[qt-offscreen-test-harness]]), so a
build step and a test run are not two separate waits — which makes it MORE
tempting to over-run them, not less.
