---
name: erp-ui-table-layout-settings
description: PUT /ui-table-masters/layout-settings writes only the fields the body names; grid order comes from ui_tbl_clm_column_position
metadata:
  type: reference
---

`PUT /ui-table-masters/layout-settings` is the partial-update route for `fixed.ui_table_columns`. Per column it accepts `uiTblClmId` (required) plus any of `uiTblClmColumnWidth`, `uiTblClmColumnVisibility`, `uiTblClmColumnFocus`, `uiTblClmColumnPosition`, `uiTblClmColumnNecessity`, `uiTblClmNextColumn`, `uiTblClmPreviousColumn`.

**The service writes a field only when it is `!== undefined`** (`updateVisibilitySettings` loops `UI_TABLE_VISIBILITY_SETTING_FIELDS`), so an omitted key leaves the stored value alone — send only what changed; never echo a width back just to be safe. Response message is a stale "Visibility settings updated successfully".

**Ordering:** the sales entry grids sort by `(uiTblClmColumnPosition, uiTblClmNo)`. Live layouts ship duplicate positions (grid 24 has two rows at 92), hence the `uiTblClmNo` tie-break. As of 2026-08-25 grids 21/23/24 all have `position == uiTblClmNo`, so switching from no-ordering to position-ordering was a visual no-op.

**Renumbering trap:** `resolveColumns` drops configured rows the client has no meaning for, so the settings dialog only holds a subset of the layout. Renumbering densely from 1 would collide with the dropped rows' positions — write the drafted order back over the *same position numbers the dialog's own rows already hold* (sorted), with a `previous + 1` floor to repair duplicates.

`focus` and `necessity` are persisted but no entry screen reads them yet. Related: [[erp-quotation-entry-port]], [[erp-ui-table-width-is-not-a-percent]], [[erp-sale-order-entry-port]].
