---
name: customer-creates-ledger
description: /customers/create writes the acc_ledger_master row in the same call — what is copied, what is not, and that the accounting group comes from the AREA
metadata:
  type: project
---

Live-probed 2026-09-19 on 192.168.0.106: `sales.customers.cus_id` is an FK to
`accounts.acc_ledger_master.led_id`, and `POST /customers/create` writes BOTH
rows in one call. Create, edit (same route with `cusId`) and
`DELETE /customers/delete` (soft: is_deleted=t, is_active=f) all keep the two in
step. A name clash is refused 409 *before* either row is written, but the error
names the field `ledName`, not `cusName`.

The accounting group is the **AREA**, not the customer group: every
`sales.area_master` row has a mirror `acc_group_master` row with the SAME uuid
and name, parented to "Customers" (BALANCESHEET/Assets), and `led_group_id` =
`cusAreaId`. `cusGroupId` (sales.cust_groups) never reaches the ledger — by
design, it is a sales grouping. Stamped constants: led_ledger_type=PARTY,
led_is_bill_by_bill=true, led_category=GENERAL.

Copied: name, short, addr1-3, city, district, state name/code, pin, country,
all four region address fields + region name, tel, phone1/2, whatsapp, email,
contact person, aadhar, GSTIN, PAN, ecommerce GSTIN, notes→led_remarks,
company/branch.

NOT copied at first (raised as notes (38)). **Fixed same day, verified
2026-09-19 on create AND edit:** `cusGstType` → led_gst_party_reg_type,
`cusEnableSms` → led_allow_sms, `cusSortOrder` → led_sort_order.
**Still dropped (notes (39)):** `cusTcsApplicable` → led_is_tcs_applicable
stays false — the live one, because acc_tcs_register and the bill-wise view
read the LEDGER flag, so a customer ticked for TCS collects nothing
([[tcs-billwise-design]]); and `cusItcollExempted`/`cusItcollType` →
led_itc_eligibility stays NULL.
led_tally_name, led_mailing_name and led_alias are left NULL — the Tally export
keys on led_tally_name, see [[ledger-master-gap-analysis]].

Customer-only fields with no ledger column at all (fine): title, code, landmark,
credit terms, price level, birth/marriage date, transport, geo location.

**Raised as notes (38)** in the share, with acceptance. Two further facts found
while writing it: `cus_gst_type` / `sup_gst_type` are unconstrained varchar(30)
(the box holds 'test', 'test21', and 'Regular' next to 'REGULAR') while
`led_gst_party_reg_type` is CHECKed to REGULAR|COMPOSITION|UNREGISTERED — so
the enum has to land before the copy can. And the LEDGER FORM's Registration
Type combo offers six values of which **Consumer, SEZ and Overseas are a hard
400**; only Regular/Composition/Unregistered save (case is normalised server
side). Left in place deliberately pending Prathap's answer on whether the enum
widens — do not "fix" the combo by cutting them without that answer.
