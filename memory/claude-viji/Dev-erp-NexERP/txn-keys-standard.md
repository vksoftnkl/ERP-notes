---
name: txn-keys-standard
description: "User-approved 2026-10-23 global key set for EVERY txn entry screen (TxnKeys) — Ctrl+N/S/Enter/O/H, F2, Alt+X, Esc, F1; rollout screen by screen, Opening Stock done"
metadata:
  node_type: memory
  type: project
  originSessionId: d91ee3ca-fe8a-4093-a454-fea514a4cc2b
  modified: 2026-10-23T06:19:25.643Z
---

User asked for keys "globally known, easy for users, no need to care about NexERP", approved my set 2026-10-23 and said "go ahead, start with opening stock".

| New/Clear Ctrl+N | Save Draft Ctrl+S | Save & Post Ctrl+Enter (asks) | Edit F2 | Cancel Alt+X (caption "Cancel", user 10-23; confirm + reason) | List/Open Ctrl+O | Close Esc | Print Ctrl+P | Find Ctrl+F | History Ctrl+H | Keys F1 | Previous/Next Ctrl+PgUp/Ctrl+PgDn — KEYS ONLY, no buttons (user 10-23) | Duplicate line Ctrl+D (Opening Stock: Split batch; line action → F1 "Lines") |

**Why:** Windows/Office/browser/mail habits (Ctrl+Enter = send; Alt+X from Tally). F5 means refresh to everyone, so it is NOT save; Ctrl+Delete rejected because it deletes a word in a text field.

**How to apply:** use `src/shared/ui/txn/txn_keys.{h,cpp}` — `TxnKeys::bind(window, Action, button, caption)` sets caption "Label · Key", clears the .ui shortcut, adds a WindowShortcut that clicks the button only when visible+enabled, committing an open grid cell editor first; `installHelp(window, extraLines)` = F1 sheet. Remove the screen's old F5/F8/Ctrl+Enter QShortcuts and .ui `shortcut` props when converting. Rules: one key = one meaning on every screen; Enter never saves/posts; irreversible = modifier + confirm.

**Rollout:** Opening Stock (menu 44) DONE + live-tested (test_entry_ux U12). Next screens not yet converted — many still use F5 (save/settle/refresh) and F8 (list); see grep in [[opening-stock-entry-ux-1023]]'s session. Test trap: NexPopupWindow and TxnMainView are MODAL, so a test's modal-closer swallows them — exempt the popup, read the closed list from modalsSeen.

**10-23 later:** `TxnKeys::bindKey(window, action, handler)` = key with no button (Prev/Next). F1 = `QDialog#TxnKeysDialog` sheet (Document + Lines sections, action left, equal-width key chips), `installHelp(window, QList<KeyRow>)`; its QSS is template **v32 — PUBLISHED 2026-10-23** (md5 57365cb6, read back = share seed-template-v32.qss). Prev/Next on Opening Stock walks grid 99 client-side until **notes-142** (`admin/`) — one server `GET /txn/adjacent?docType=…` for all 11 doc types (same header shape in all 8 tables) — lands.

**10-24:** Physical Stock (menu 45) converted too — same binds + Load Sheet keeps F9 (screen-only key); test_physical_ux V3.

**10-24:** Stock Adjustment (264) converted — Pick from stock moved F2→F12 so F2 is Edit everywhere.
