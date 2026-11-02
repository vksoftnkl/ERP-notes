---
name: popup-enter-navigation-rule
description: "EVERY plain-QDialog popup calls EnterNavigation::install(this) — Enter moves to the next field, no default/auto-default button; the three exceptions and the picker-list trap"
metadata:
  node_type: memory
  type: feedback
  originSessionId: 5cd4e0af-1d3c-497e-9bf6-ad46ff0af939
  modified: 2026-09-28T06:43:26.221Z
---

Every popup that is a plain `QDialog` calls `EnterNavigation::install(this)` at the END of its constructor (or `install(&dlg)` before `exec()` for an inline dialog). The helper is `src/shared/widgets/enter_navigation.{h,cpp}`. NexDialog screens already have their own Enter filter (NexDialog::setup), so they don't need it.

What it does:
- **Keypad Enter carries `Qt::KeypadModifier`.** Every "no modifier" test must mask it: `(key->modifiers() & ~Qt::KeypadModifier) == Qt::NoModifier`, or number-pad Enter is silently ignored. This was THE real-app bug of 2026-09-28: offscreen traces sent the main Return and passed. Trace both keys (`TRAIL_KEYPAD=1 run.sh chequetrail` uses QTest::keyClick through the window system).
- Enter/Return moves to the next field, like Tab. It sends a Tab key, because `focusNextChild()` is protected.
- An open NexDropdown popup or QComboBox list takes Enter first, to commit its row.
- Item views (grids, lists, trees) and multi-line text keep their own Enter.
- Enter on a push/tool button clicks it.
- No push button is default or auto-default.
- **First focus:** on Show, if the popup chose no field (`dialog->focusWidget()` empty), focus goes to the first Tab-focusable field; a button only if there is nothing else. An Enter that lands on the dialog itself also goes to the first field. Without this, a popup opened with nothing focused ignored Enter entirely (the real-app bug of 2026-09-28; offscreen traces had hidden it by setting focus themselves).
- On every Show it clears defaults again and adopts children built later. This is needed because a QDialogButtonBox re-defaults its first accept button when shown, and the validation popup builds its buttons late.

**Why:** the user, 2026-09-28: "use it for all popup and save it into memory". The triggers were the cheque popup, where Enter accepted a half-filled form, and default buttons on several popups.

**How to apply:**
- A new popup calls `install()`. Prefer plain QPushButtons over a QDialogButtonBox in new code.
- **The picker-list trap:** in a popup whose list/table is a PICKER, Enter used to fall through to the default button ("Restore", "Import"). With no default button that stops working. So wire the view's `itemActivated`/`activated` to `accept`, as the sale bill's Held Bills picker does.
- **Deliberately NOT installed:**
  - NexPopupWindow: Enter selects the grid row.
  - AppMessage: OK is the default on purpose, so Enter answers the box.
  - LoginWindow: Enter logs in, and it is not a popup.
- VoucherBillwiseDialog has it on top of its own rule (Enter = OK outside the grid).
- Installed in 2026-09-28 in 16 popup classes plus 4 inline dialogs (Received Cheques panelDialog, the NexWarningStrip validation popup, the Held Bills picker, the image preview).

Related: [[grid-line-keys-plus-minus]], [[voucher-register-screen]], [[cheque-panel-dialog]].

**2026-10-15 (till screens):** a NexDialog that never calls `setup()` has NO Enter filter — the till screens (Open Till, End Shift, Approval Rules) now call `EnterNavigation::install(this)` like Cash Movement. NOT on dashboard screens of action buttons (Business Day): Enter walked onto "Close with exception" and the next Enter would click it. Esc "not working" = keyboard focus left on the shell menu when a screen focuses nothing of its own; FormManager::createSubWindow now activates the sub and, if focus isn't inside, focuses the screen's chosen/first field, else the screen itself. Probe: `EXTRA_LIBS=-lQt6Test SNAP_SRC=esc_probe tests/till/snap.sh` (opens via the real FormManager with a "menu" field holding focus first).
