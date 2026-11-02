---
name: user-menu-groups-follow
description: User Administration rights tree — a ticked screen now ticks its groups (fixed 10-12); before, a screen saved without its group was invisible on the menu bar
metadata:
  type: project
---
2026-10-12 user: "in user admin when i click open till its group header checkbox not checked". UserMenuWidget (src/modules/admin/master/user/user_menu_widget.cpp) only cascaded DOWN (group → children). fieldValue() saves only rows whose COL_MENU is ticked, so ticking Open Till (272) saved it without its group Till (271) → no Till group on the menu bar.

Fixed: onItemChanged → syncAncestors (ticked row ticks every ancestor; a group with no ticked child unticks); ticking any right (View…) ticks the row; applyPermissions ticks ancestors of granted rows so the next save repairs an old user. tests/till/menu_groups.cpp 5/5 live (272 → 271 → 1 = "&1 Sales").
Menu tree table: fixed.menu_master (menu_id, menu_parent, menu_name). Orphan check SQL: user_menus row whose menu_parent has no visible row for the same user — on 10-12: VKSOFT 6 screens (groups "&6 Reports", "Financial Statements"), zt-approver (Till, my test user). Related: [[user-menu-full-replace]].
