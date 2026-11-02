# notes (52): `GET /vouchers/adjacent` for the Voucher Register's Prev / Next

2026-09-26. The Qt Voucher Register (menus 101-104, 163, 259-262) now has Prev
/ Next buttons (Ctrl+PgUp / Ctrl+PgDn), the same walk the Receipt screen does
over `/receipts/adjacent`. `/vouchers/*` has no such route, so the screen
**currently walks grid 117** ("TXN MAIN LIST - VOUCHER REGISTER"). It reads up to
100 rows a page and searches them for the voucher on screen, paging on until
it finds it. That works, but it is up to 20 page reads to take one step in a
long register, and it depends on the grid's ORDER BY never changing. One small
route replaces all of it.

## The ask

`GET /api/v1/vouchers/adjacent`, the same shape as `/receipts/adjacent`
(`open-items.service.ts` `adjacent()`), with two differences called out below.

**Query** (bare camelCase, like every `/vouchers/*` route):

| key | required | meaning |
|---|---|---|
| `companyId`, `branchId`, `accYear`, `voucherId` | yes | the voucher on screen; a miss on any of the four is a **404** (as on the receipt: a caller scoped elsewhere does not learn it exists) |
| `direction` | yes | `prev` = older, `next` = newer |
| `typeCode` | no | **difference 1:** a type menu (103 Journal, 163 PurA, …) walks its own type only; omitted = the register (menu 262), every type below |
| `status`, `fromDate`, `toDate` | no | the list's filters, when it has them, exactly as on the receipt |

**Which vouchers are walked:**

- `vt.vchr_in_register = true`, so the `Rev` reversals and the goods documents are
  never stepped onto (the receipt uses `avh_against_voucher_id IS NULL` for the
  same purpose; either is fine as long as a Rev voucher is not a stop);
- **difference 2: only types the caller may VIEW.** That means `user_menus.um_can_view` on
  the TYPE's `vchr_menu_id`, the same rule `/vouchers/types` and grid 117 apply
  (grid 117 joins `public.user_menus` on `iuser_id`). A user with rights on 103
  only must never be stepped onto a Purchase (Accounting) voucher.
- `avh_is_deleted = false`, same company / branch / year.

**Order:** exactly the receipt's tuple, so Prev / Next agree with the F8 list:
`(avh_voucher_date, COALESCE(avh_voucher_slno, <draft sentinel>), avh_created_on, avh_voucher_id)`,
with `<` + `DESC` for prev and `>` + `ASC` for next, and `LIMIT 1`.

**Answer:**

```jsonc
{ "success": true, "message": "pua00014 is the prev voucher",
  "data": { "direction": "prev", "fromVoucherId": "…",
            "voucher": { "voucherId": "…", "companyId": "…", "branchId": "…", "accYear": "2026-2027",
                         "typeCode": "PurA", "voucherRefno": "pua00014", "date": "2026-09-25",
                         "status": "POSTED" } } }
```

`voucher` is `null` at either end, with the message "No prev voucher — this is
the end of the register". The client then loads it with `/vouchers/get` as usual.

A DRAFT is a stop, with no refno (`voucherRefno: null`), exactly as on the receipt.

## Also wanted: Prev on an EMPTY screen

The screen's Prev with nothing loaded opens the **newest** voucher. Either
allow `voucherId` to be omitted with `direction=prev` (answer = the newest
under the same filters), or leave it and the client keeps reading row 1 of
grid 117 for that one case. The first is tidier; say which you prefer.

## When it lands

The client swaps `VoucherRegisterEntry::stepVoucher()` (the grid-117 walk) for
one call; nothing else on the screen changes. Checked by
`tests/register/run.sh features` (Prev from empty → newest, Prev → older,
Next → back, Next past the newest → "newest" message), which passes today on
the grid-117 walk (pua00015 → pua00014 → pua00015).

---

## Checked 2026-09-26 after the 14:00 re-sync

- **`/vouchers/adjacent`: DONE**, and the client uses it. prev/next, the optional
  `voucherId` (prev with none = the newest) and `typeCode`; a Rev is never a
  stop; `voucher: null` at the ends. The screen's grid-117 walk is gone.
- **41 `accounts.drcr_display`: DONE** (seed `App_Setting_Drcr_Display.sql`);
  `/app-setting-values/effective` returns it at DR_CR.
