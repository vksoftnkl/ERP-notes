---
name: qt-offscreen-test-harness
description: "How to exercise NexERP widget code headlessly — link ninja's object list minus main.cpp.o, run under QT_QPA_PLATFORM=offscreen"
metadata: 
  node_type: memory
  type: reference
  originSessionId: d53e5ac5-c399-4c45-b65a-d6d098966e50
  modified: 2026-08-24T14:16:13.047Z
---

**Screen capture does not work on this machine.** The desktop is a Wayland
session (`XDG_SESSION_TYPE=wayland`), so an X grab through XWayland returns a
solid black frame — verified, 1 distinct colour over 1920×1080. There is no
Xvfb, no scrot/import/ImageMagick, and no `xvfb-run`. `xdotool` exists and can
drive XWayland windows, but blind, on the user's real desktop. So a GUI change
cannot be verified by looking at it; it has to be verified another way, or by
the user.

**The way that works: link the app's own objects and supply your own `main()`.**

```bash
cd build/Desktop_Qt_6_12_0_Debug
ninja -t commands NexERP | tail -1 | tr ' ' '\n' | grep '\.o$' \
  | grep -v 'main\.cpp\.o$' > objs.txt        # ninja's list, not `find`
sed -i "s|^CMakeFiles|$PWD/CMakeFiles|" objs.txt   # it emits relative paths
g++ -std=c++17 -fPIC -o harness harness.cpp $(cat objs.txt) \
    -I<each module dir> -I$QT/include/QtCore -I$QT/include/QtGui \
    -I$QT/include/QtWidgets -L$QT/lib -lQt6Core -lQt6Gui -lQt6Widgets \
    -lQt6Network -lQt6PrintSupport
LD_LIBRARY_PATH=$QT/lib QT_QPA_PLATFORM=offscreen ./harness
```

Two traps, both hit on the first attempt:
- **Use ninja's object list, never `find -name '*.o'`.** The build directory
  keeps stale objects for sources no longer in `CMakeLists.txt`
  (`mainwindow.cpp.o`, `nex_dynamic_modal.cpp.o`, `nex_master_list.cpp.o`);
  their mocs are gone, so linking them fails on missing vtables.
- Ninja prints **relative** paths; make them absolute or link from the build dir.

The code under test is then byte-identical to what ships, which an
independently compiled test could not promise.

**What this reaches.** A real `NexTable` works offscreen if you skip
`loadLayout()` (it needs the server) and call `setDisplayFormats()` +
`setColumnCount(SalesItemCols::ColumnCount)` yourself. That is enough to drive
grid controllers end to end — populate rows, run the controller, read the cells
back, call it repeatedly to prove idempotency, and count callbacks to prove a
network lookup is not being re-fired. What it does not reach is the host screen's
own wiring (its context builder, its call sites, its widget connections).

**Design consequence worth keeping:** the promotion rules and engine were written
widget-free precisely so they could be linked into a harness without Qt Widgets
at all. Keeping decision logic free of `NexTable` is what makes it testable here.

**Why:** verifying a GUI change on this box means either a headless harness or
asking the user to look — guessing is not verification.
**How to apply:** used to verify the promotion engine and
`SalesPromotionBinder` on 2026-08-24; see [[promotion-engine-client]].


**2026-09-23 — SNAPSHOTS, not just asserts.** `tests/salebill/snap.sh` links
the same way, logs in (or not — the layout renders without the server), sets
`setFixedSize(1680, 1000)` (the MDI subwindow's real size; a free top-level
grows to sizeHint and hides the squeeze), `layout()->activate()` then
`QWidget::grab().save()`. Reading the PNG with the Read tool is the closest
thing to seeing the screen this box allows — it caught a fixed-height grid
painting over a sibling that no assert would have.

**`tests/salebill/snap.sh` now BUILDS and refuses a stale render** — it never
did, which is why two renders in a row showed the previous layout and sent me
chasing phantom bugs (a "disabled" button that was enabled in the .ui). It
loops `ninja` up to 3 times (Qt AUTOGEN is two-phase: pass 1 regenerates
`ui_*.h`, pass 2 compiles what includes it) and then requires
`ninja -n` to say "no work to do". `SNAP_SCREEN=quotation` renders
QuotationEntry instead — useful for comparing against the house style.

**Stale-object trap (2026-09-23).** After a `.ui` edit, one `ninja` run
regenerated `ui_*.h` but the `.cpp.o` was still OLDER than the header and the
snapshot linked the stale object — the render showed the previous layout while
the header had the new one. Before trusting a snapshot: `ninja -n` must say
"no work to do". The harness now prints nothing about this; check it yourself.
**2026-10-09: a .ui edit can need TWO ninja runs.** After editing a .ui, the first `ninja` (incl. the one inside tests/till/snap.sh) regenerated ui_*.h but did NOT recompile the .cpp that includes it — the render showed the old button labels. A second `ninja` compiled + relinked. After any .ui edit run ninja twice (or check the .o timestamp) before trusting a snapshot.
