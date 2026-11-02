---
name: nexdialog-stack-crash
description: NexDialog sets WA_DeleteOnClose — a NexDialog subclass on the STACK + exec() crashes the app when it closes (munmap_chunk invalid pointer)
metadata:
  type: project
---
NexDialog's ctor sets `Qt::WA_DeleteOnClose` (nex_dialog.cpp ~48). `Foo dlg(...); dlg.exec();` on a NexDialog subclass → close() → deleteLater on a stack object → `munmap_chunk(): invalid pointer`, the whole app exits.

Found 2026-10-12 from the user's report "Open Till Session exits after ~20 s or on pressing buttons": S1 → Continue count / End shift / Cash movement opened TillEndShiftDialog / TillCashMovementDialog on the stack; closing them (Esc, Close, or their own close after a successful post) killed the app. 7 sites, all in till (S1, cockpit, txn_doc_types_till, business day → cockpit) — fixed with `dlg.setAttribute(Qt::WA_DeleteOnClose, false);` after construction. App-wide scan (120 NexDialog descendants) found no others.

**How to apply:** a modal NexDialog popup either gets that line, or is `new`-ed (QPointer + exec). Repro harness: tests/till/idle_open.cpp (`DEVICE_UID=<this PC uid> SNAP_SRC=idle_open tests/till/snap.sh`) — idles, keys, clicks every button, backtraces any close. Plain-QDialog popups (AppMessage, TillSlipCheckDialog) are not affected.
