# Notes 102: a "Till Masters" group under Till (2026-10-09)

**STATUS:** open. Menu data only. The client needs no change: it finds each screen by its id, and it builds submenus to any depth (`MenuWindow::buildMenu`, recursive). Your `menu-master.service.ts` tree is recursive too.

**Order:** after notes 101.

## Asked (the user's call)

Put the four till master screens from notes 101 into their own group under Till:

```
Sales (1)
└── Till (271)
    ├── Open Till            272   1.00
    ├── Till Sessions        273   2.00
    ├── Business Day         274   3.00
    ├── Till Masters  (NEW GROUP 283)   4.00
    │   ├── Till Counters    275   1.00
    │   ├── Till Safes       280   2.00
    │   ├── Till Reasons     281   3.00
    │   └── Denominations    282   4.00
    └── Till Approval Setup  276   5.00
```

1. **New group row:** id 283 (or the next free id), parent 271, name **Till Masters**, position 4.00, verbs `{VIEW}`, `menu_separator` as on 271, visibility as on 275.
2. **Move the four screens:** set `menu_parent` = 283 on 275, 280, 281 and 282, at positions 1.00–4.00.
3. **Grant the group:** a group shows only to a user who holds VIEW on it (VKPOS has rows on 1 and 271). Please give VIEW on 283 to every user who holds any of 275 / 280 / 281 / 282.
4. **Seed:** `prisma/seed/Menu_Master.sql`, the same as above.

The ids of the four screens and their rights do not change. `TILL_MENU` needs no change, apart from a GROUP-style constant for 283 if you want one.

Not moved: Till Approval Setup (276) stays directly under Till. If the user wants it in the group too, it would be one more `menu_parent` = 283 at position 5.00.
