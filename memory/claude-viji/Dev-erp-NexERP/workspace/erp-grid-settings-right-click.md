---
name: erp-grid-settings-right-click
description: "2026-10-03 every layout-driven table in the React client has the admin right-click menu; shared hook for non-CrudMasterPage configured grids; uncommitted"
metadata:
  type: project
---

**Built 2026-10-03, UNCOMMITTED,** in `/home/vk/Dev/erp/ERP client`. The user asked for the "admin setting right click popup" on every page's table.

**Two kinds of menu**

- **ui_tables entry grids** use the existing `useGridSettings` + `useColumnResize` from `features/sales/quotation/components/`. The menu offers "save column width" and "Admin settings" (a dialog).
  - Now on: voucher legs (ui 39), bill-wise popup and card (ui 40, through `useBillGrid`), Opening Stock lines (its resolver now carries columnId / necessity / columnNumber), Selling Price.
  - Stock Adjustment works in width shares and picks its columns per kind. It gets the dialog only, fed by the new `settingsColumnsFromLayout(rows)`. `onSaveWidths` is now optional and omits the menu item.
- **Configured grids outside CrudMasterPage** use the new `components/master/use-configured-grid-settings.tsx`. It has the master menu's items: grid filter setting, column visibility setting, Admin settings.
  - Now on: the Issued and Received cheque registers, the Cheque Books list, voucher Exceptions (grid 118), and the Opening and Physical Stock picker dialogs.

**Decisions not obvious from the code**

- From the new hook, Admin settings opens the Grid Designer in a NEW TAB (`window.open`). These grids sit on working screens, and CrudMasterPage's same-tab `router.push` would drop unsaved work without asking.
- While either settings dialog is open, F1–F12 and Ctrl/Meta+Enter are swallowed on the window capture phase, so a screen's own F5 save / F7 new / Ctrl+Enter post can't fire underneath it.
  - Esc is left to ModalShell, which already stops it at document capture.
  - Both hooks expose `active` so a screen's key guard can stand aside.

**Verified:** tsc, 3112 tests, lint with the same 377 pre-existing warnings, SSR render of the resize handles, all routes 200. **Not done:** never clicked through logged in (no credentials, no jsdom).
