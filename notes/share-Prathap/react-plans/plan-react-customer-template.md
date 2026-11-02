# Customer Master — "Save as Default Template" — React port plan

Scope: **one button and two pure functions**. The Customer Entry form itself
(fields, GSTIN/state validation, the credit-section rule, the collection-days
widget, the list grid) is NOT in scope here — that is its own plan. This
document covers only the path that turns the form the operator is looking at
into the *default template* for new customers at this company or branch, and
the path that seeds a blank form back from it.

Qt source: `src/modules/sales/master/customer/customer_entry.cpp`
(`installTemplateAction`, `saveAsDefaultTemplate`) and
`src/shared/ui/nex_crud_form.cpp` (`applyDefaultsJson`, `buildDefaultsJson`,
`fillFromJson`).

Companion documents: **`plan-react-app-settings.md`** — this feature is a
*writer* of one app-setting override row, and every trap in its §3.2 applies
here verbatim. Do not re-derive them; §4.4 below cross-references them.

> **The whole rule in one line:** the setting is written **only** when the
> button is clicked. Typing in the form writes nothing, autosave writes
> nothing, unmount writes nothing — and the button writes the *template*, never
> the *customer*. Two writes live on one form and neither may trigger the
> other. §4.6 is the test for it.

---

## 1. What the template is, and what it is not

**The form is the template editor.** A superuser opens Customer Entry, fills it
in the way a typical new customer should start, and presses one button. That is
the whole authoring UI, and it is the only one that is usable by the people who
need it — the alternative is hand-typing a 700-character JSON object into a
text input on the settings screen. 3.0 did it this way
(`customerentry.cpp:615 saveTemplate`) and it was right.

What the stored document is:

- **A seed for a FORM, not a save payload.** It carries the display halves of
  searchable dropdowns (`cusAreaName`, `cusGroupName`, `cusStateName`,
  `cusPriceLevelName`) *because the form needs them to render a label*. The save
  payload deliberately strips those (`beforeSubmit`), and a template built from
  the payload shape sets Area's id and leaves the box looking empty. That is
  what "the template did nothing" looks like from the outside. §3.2.
- **Only what describes a KIND of customer.** Everything that identifies one
  customer is excluded — name, short name, code, GSTIN, PAN, Aadhaar,
  e-commerce GSTIN, contact person, all four phone fields, email, notes, birth
  and marriage dates. §3.1.
- **Add-mode only.** On an existing record the stored values are the answer and
  a default would overwrite them. The template is applied to a blank draft and
  nowhere else.
- **Not retroactive.** Existing customers are never touched. The confirm text
  must say so, because "default template" reads like a bulk update to anyone
  who has not written one.

It is also what makes a **hidden mandatory field** work. A deployment that hides
Area with `applyFieldConfig` has removed the operator as the source of a value;
it has not removed the NOT NULL column. The template puts a source back.

Here is a real stored document, read out of the deployed DB — 29 keys, 730
bytes, the whole feature:

```json
{"cusAllowDiscount":true,"cusAreaId":"019f1710-…","cusAreaName":"MUSIRI",
 "cusCollectionDays":[0],"cusCountry":"India","cusCreditAllowed":false,
 "cusCreditAmtLimit":0,"cusGroupId":"019f213b-…","cusGroupName":"VIP Parties",
 "cusGstType":"Unregistered","cusIsActive":true,"cusPriceLevelId":"1",
 "cusPriceLevelName":"WS Price","cusStateCode":"33","cusStateName":"Tamil Nadu", …}
```

Note `cusPriceLevelId: "1"` — a string, not a uuid. Ids in this system are not
all one type; the template must round-trip whatever the field holds and never
"normalise" it.

---

## 2. Where the defaults live

### 2.1 The setting

| | |
|---|---|
| key | `masters.customer_form_defaults` |
| `asd_data_type` | **`TEXT`** — not `JSON`. The value is a *string that happens to contain JSON* |
| `asd_max_scope` | `BRANCH` — the DEEPEST level allowed, not a requirement |
| storage | `app_setting_value.asv_value` (text) |

Verified against the deployed database, not the schema file. Two override rows
exist today for this key, one `COMPANY` and one `BRANCH`, 730 bytes each.

Data type `TEXT` has one consequence worth stating: **the server does not
validate the JSON.** A malformed document is accepted by the write and only
discovered by the client that tries to apply it. §3.4 is why that is survivable.

### 2.2 The read path

Unchanged from every other setting, and the client holds **no merge logic**:

```
GET /app-setting-values/effective?companyId=…&branchId=…&deviceId=…&userId=…
  → row { asdKey: "masters.customer_form_defaults", value: "<json text>", source, override }
  → session store:  customerFormDefaults: string      ("" = no template, form opens blank)
```

Resolution — `USER ▸ DEVICE ▸ BRANCH ▸ COMPANY ▸ GLOBAL ▸ catalog default` —
is the server's job (`fn_app_settings`). React reads one string out of the same
settings store `plan-react-app-settings.md` §7 already builds. Do not give this
feature its own fetch.

### 2.3 The write-back

After a successful POST, push the JSON into the **running** settings store so
the next Add uses it without waiting for a re-login or a branch switch:

```ts
settings.set('masters.customer_form_defaults', json);
```

Correct for either scope choice: this session's branch resolves to what was just
written whether the row landed at `COMPANY` or at `BRANCH`. Guard it the same
way §7 of the app-settings plan does — only when the write targeted the context
this session is actually inside. (Here it always is: the scope ids come from the
session, never from a picker. State the guard anyway, so that stays true if a
company picker is ever added.)

---

## 3. `buildDefaults` / `applyDefaults` — pure, and tested first

```
src/features/customer/template/
  buildDefaults.ts        // draft + field specs -> compact JSON string   PURE
  applyDefaults.ts        // JSON string -> Partial<CustomerDraft>        PURE
  excluded.ts            // the identity key list
  SaveAsTemplateButton.tsx
  useSaveTemplate.ts      // the mutation: scope prompt -> POST -> store
```

`buildDefaults` and `applyDefaults` import nothing from React and nothing from
the API layer. They are the whole of the risk and they are testable without
rendering anything. Write them as **generic over a field-spec list**, not
hard-wired to customer: `masters.item_form_defaults` already exists in the
catalog and in the session, and no screen writes or applies it (§5.2). Item
master should get this for the price of a different exclusion list.

### 3.1 The exclusion list

```ts
export const CUSTOMER_TEMPLATE_EXCLUDED = [
  'cusName', 'cusShort', 'cusCode', 'cusGstNo', 'cusPanNo', 'cusAadharNo',
  'cusEcommerceGstin', 'cusContactPerson',
  'cusTel', 'cusPhone1', 'cusPhone2', 'cusWhatsappNo', 'cusEmail',
  'cusNotes', 'cusBirthDate', 'cusMarriageDate',
] as const;
```

Port it verbatim. Every entry is "what identifies ONE customer, never a kind of
customer", and the regional-address block is deliberately **not** in it — a
branch that bills in Tamil may well want the regional country and state seeded.

### 3.2 Build, in order

1. Collect the draft's form-visible values — the same set the form would
   submit.
2. **Pair every dropdown id with its display text.** For each `apiDropdown`
   field with a `displayKey`, if the id is present and the text is not already
   carried, write the selected label. This is the step that was missed once and
   produced a template that looked like it did nothing.
3. Remove the excluded keys.
4. **Strip "not set".** Drop `null`, `undefined`, empty/whitespace strings,
   empty arrays and empty objects. Empty is *not set*, not *set to nothing*:
   carrying it would blank that field on every Add, which is worse than leaving
   it for the operator.
5. Remove any dangling `displayKey` whose id did not survive step 3 or 4 — a
   label on the form with nothing behind it.
6. `JSON.stringify` compact. No pretty-printing: it is a text column, and the
   diff nobody reads is not worth the bytes.

**Known asymmetry, ported deliberately:** `false` and `0` are *kept*. So a
template authored on a form where Credit Allowed happened to be unticked stores
`cusCreditAllowed:false` and forces it off on every new customer. That is
usually what the author meant (they were looking at the form) and it is the only
way a template can express "off" at all. Do not "fix" it with a smarter strip —
if a deployment ever needs "no opinion" on a boolean, the fix is an explicit
include-list per field spec, not a value-shaped guess.

### 3.3 Apply

```ts
applyDefaults(json, fieldSpecs) -> Partial<CustomerDraft>
```

- **Add mode only.** Never call it on a loaded record.
- **A partial template is a partial apply.** Keys the document does not carry
  are left alone. This is where the Qt version has a bug worth *not* porting:
  `fillFromJson` short-circuits `data.contains(key)` for the Custom and Image
  kinds, so a template with no `cusCollectionDays` still pushes a null into the
  collection-days widget and clears it. In React this is a merge over known
  keys and the bug cannot exist:
  `{...blank, ...pick(parsed, knownKeys)}`.
- **Unknown keys are ignored, silently.** A template written by a newer build
  carrying a field this build does not have must not be an error.
- **Coerce at the boundary.** The document is untyped JSON; the draft is typed.
  Convert per field spec — checkbox → `Boolean`, numeric → `Number`, dropdown id
  → `String` — rather than trusting what comes back. `"1"` and `1` both appear
  in live data.
- The two hard-coded fallbacks stay, and stay conditional: Country `India` and
  State `33 / Tamil Nadu` are written **only where the template says nothing**.
  A deployment that has set its own country/state must not have these put back
  on top.

### 3.4 Malformed input is not an outage

Parse failure logs a warning and returns `{}` — the form opens blank and the
operator carries on. A defaults document somebody mistyped must never be the
reason a master cannot be added. Since the column is `TEXT` and the server does
not parse it (§2.1), this is the only place the mistake can be caught.

---

## 4. The button

### 4.1 Where it goes

Footer row, **far left**, with Cancel/Save keeping the right. An occasional
action must not sit under the thumb of the one used forty times a day.

**Not in a menu.** In Qt this started life as a right-click action and was
invisible in practice: `Qt::ActionsContextMenu` only reaches the dialog when the
click lands on a widget that ignores the event, so right-clicking any text field
showed the line edit's own cut/copy/paste instead. React has no such mechanism,
but it has the same failure — an overflow menu the operator never opens is the
same as no feature. A visible, labelled button.

Label: **"Save as Default Template"**. Tooltip: *"New customers at this branch
will start with what is on this form. Existing customers are not touched."*

### 4.2 Who sees it

```ts
canEditTemplates(session)   // session.userType === 'SUPER ADMIN'
```

One helper, in the session module, not a string compare inline in the
component — three other places in the Qt client test the same string and they
should all move together.

This gate is **UI convenience only**. `POST /app-setting-values/create` is not
authorized server-side today, so any authenticated caller can write the
template. See §5.1; do not describe the button as a permission.

### 4.3 The scope prompt

`asd_max_scope` is `BRANCH`, which is the deepest level *allowed* — not a
requirement. Both answers are legal and they mean genuinely different things,
so the operator picks:

| Choice | Scope written | Means |
|---|---|---|
| **All Branches** | `COMPANY` | one house style; a single shop wants this, and so does a chain whose branches agree |
| **This Branch Only** | `BRANCH` | an exception for one place — the resolver prefers it over the company row because it is narrower |
| Cancel | — | nothing is written |

Hardcoding `BRANCH`, as the Qt screen did at first, gives every branch its own
copy and leaves no way to change them all at once.

A real modal with three labelled buttons, not `window.confirm` — a two-way
confirm cannot express a three-way question, and the body text is where
"existing customers are not touched" gets said.

Before prompting: if the session carries neither company nor branch id, warn and
stop — there is nothing to save it against. If "This Branch Only" is chosen and
there is no branch id, warn and stop.

### 4.4 The upsert

```ts
{ data: [{
    asvSettingKey: 'masters.customer_form_defaults',
    asvScope     : branchOnly ? 'BRANCH' : 'COMPANY',
    asvCompanyId : branchOnly ? null : companyId,
    asvBranchId  : branchOnly ? branchId : null,
    asvDeviceId  : null,
    asvUserId    : null,
    asvValue     : json,              // a STRING containing JSON
    asvRemarks   : null,
    asvSyncDate  : null,
    asvCreatedBy : session.userName,
    asvModifiedBy: session.userName,
}] }
```

Cross-reference `plan-react-app-settings.md` §3.2:

- **Trap 1 — `asvValue` is a string.** Here it already is; the trap is the
  opposite mistake, sending the object. `asv_value` is `text`.
- **Trap 2 — no `asvId`, ever.** `/create` upserts on the scope target, and a
  client-minted uuid is by definition an id the server cannot find. This screen
  never reads the override list, so it could not supply a real one. *Caveat:*
  the second save at the same scope depends on that upsert. If the endpoint ever
  stops upserting, this screen must read `/effective` for the key first and pass
  the existing `asvId` — write the test in §5 so the change is loud.
- **Trap 3 — exactly one scope id.** The live `ck_asv_scope_ids` takes the id
  its scope names and nothing else. The share's `10_app_settings.sql` still
  carries the older cumulative version; trust the deployed constraint.

Then: push into the settings store (§2.3), and report what is actually true —
*"Saved for this branch. It takes precedence over any company-wide template."*
or *"Saved for every branch that has no template of its own."*

### 4.5 In flight

Disable the button while the POST is outstanding, and ignore the response if the
form has unmounted (the Qt `QPointer` guard). One click, one row.

### 4.6 The "clicked only" rule, stated as code review checks

- No `useEffect` in this feature has the draft in its dependency array.
- No debounce, no `beforeunload`, no autosave anywhere near
  `masters.customer_form_defaults`.
- `useSaveTemplate` is a mutation with no `enabled`/`refetch` path — it fires
  from `onClick` and from nothing else.
- The handler does **not** call the customer save, and the customer save does
  **not** call it. Pressing Save creates a customer; pressing Save as Default
  Template writes a setting; neither implies the other, and the template button
  does not run the customer form's validation (a template is allowed to be an
  incomplete customer — the live one has GST Type "Unregistered" and no name).

### 4.7 One open decision: edit mode

The Qt button installs regardless of add or edit, so opening an existing
customer and pressing it templates *that customer's* non-identity values.
Useful ("make new customers look like this one") and surprising in equal
measure. **Keep it**, and make the modal body say which record the values are
coming from when the form is in edit mode.

---

## 5. Backend gaps this feature inherits

1. **`/app-setting-values/create` is unauthorized.** The superuser gate is
   client-side only. Anyone who can reach the API can rewrite the template — or
   any other setting. Not this screen's bug; this screen is the one that makes
   it obvious.
2. **`masters.item_form_defaults` is dead.** The catalog row exists
   (`TEXT`/`BRANCH`) and the session reads it, but no screen writes it and no
   screen applies it. Either wire Item Entry with the generic module from §3 or
   retire the key.
3. **Neither `masters.*_form_defaults` key is seeded in the share's schema.**
   The rows exist in the deployed database; `schema/*.sql`, `99_seed_all.sql`
   included, creates neither. `fn_asv_check_scope` raises
   `unknown setting key %` when the catalog row is missing, so **on a fresh
   install this button fails outright.** Add both catalog rows to
   `10_app_settings.sql`.
4. **No way to clear a template from here.** Removal is a `DELETE
   /app-setting-values/delete?asvId=` and this screen never learns the id, so a
   superuser who wants no template must go to the settings screen and reset it
   there. Either read `/effective` for this one key when the form opens (cheap,
   and it also gives §4.4's caveat a way out) and offer "Remove Template", or
   document the settings screen as the only removal path.
5. **The document is never validated server-side** (§2.1). `asd_data_type`
   `JSON` plus a parse check would move the failure to the write, where the
   person who caused it is standing.
6. **No preview.** The operator cannot see what the 29 keys are before agreeing.
   Cheap win: put the count and the four dropdown labels in the modal body.

---

## 6. Tests

**Pure — `buildDefaults` / `applyDefaults` (Vitest, first, no DOM):**

- Round trip: `build(apply(build(draft)))` is byte-identical to `build(draft)`.
- Every excluded key is absent from the output, for a fully populated draft.
- A dropdown with a selection emits **both** id and display text; a dropdown
  with none emits neither.
- Strip rule, one case each: `null`, `""`, `"   "`, `[]`, `{}` dropped;
  `false`, `0`, `[0]` **kept**.
- Dangling display key: exclude `cusGroupId` → `cusGroupName` goes too.
- Apply is partial: a two-key template leaves the collection-days widget and
  every other field untouched (the Qt bug, §3.3).
- Unknown key in the document → ignored, no throw.
- Malformed JSON → `{}` plus one warning, never a throw.
- Coercion: `"1"`, `1`, `"true"`, `true` all land as the field spec's type.
- The Country/Tamil Nadu fallbacks apply only when the template omits them.

**Payload (Vitest):** snapshot the POST body for both scope choices — the pair
of null/non-null ids is the whole of Trap 3, and `asvId` must be absent from
both. A second save at the same scope hits `/create` again with no id (the
upsert caveat, §4.4).

**Component (Testing Library):**

- Non-superuser session → the button does not render.
- Click → modal; Cancel → **zero** network calls.
- "This Branch Only" → one POST, branch id set, company id null, settings store
  updated with the same string that was posted.
- No branch in session + "This Branch Only" → warning, no POST.
- **The clicked-only test:** mount, type into six fields, blur, change tabs,
  unmount — assert the POST mock was never called.
- **The two-writes test:** press Save → the customer create endpoint is called
  and `/app-setting-values/create` is not; press Save as Default Template → the
  reverse.

**Cross-client check:** write a template from the Qt client, open Add in the
React client, and diff the two forms field by field. The stored document is
shared, so this is the cheapest possible guard against a build/apply drift.

---

## 7. Phases

| # | Deliverable | Done when |
|---|---|---|
| 1 | `excluded.ts`, `buildDefaults`, `applyDefaults` | every §6 pure case passes; **no UI exists yet** |
| 2 | Blank-form seeding from the settings store, add mode only | opening Add against the live BRANCH row reproduces the Qt client's form exactly |
| 3 | `SaveAsTemplateButton` + gate + footer placement | superuser sees it, nobody else does, and it sits left of the stretch |
| 4 | Scope modal + `useSaveTemplate` + store write-back | both scope choices write the right row; the next Add uses it without a reload |
| 5 | Gap 3 (catalog seed) raised with the backend | a fresh database accepts the write |

Phase 1 before everything else is deliberate: the two functions are the only
part that can be wrong in a way nobody notices for a month.

---

## 8. Do not port

- The right-click/context-menu placement (§4.1) — it was already replaced in Qt.
- `fillFromJson`'s unconditional Custom/Image branch (§3.3) — a partial template
  must be a partial apply.
- Hard-coded `BRANCH` scope (§4.3).
- The client-minted `asvId` idea — omit the key entirely (§4.4).
- Any notion that this button also saves the customer (§4.6).

