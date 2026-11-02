# Plan: keep unsaved entries when the user leaves a screen

## Context

When a user is halfway through a sale bill, opens another menu screen and then comes back, the bill is gone. Qt avoided this because QMdiArea keeps every window alive. In React, each menu click is a `router.push` (`components/layout/erp-header.tsx:680`), and the App Router unmounts the page.

- **Testbill** keeps the bill in a `useReducer` (`features/sales/testbill/state/use-bill-draft.ts:609`), so the bill is lost when the screen unmounts.
- **Sale Bill** (menu 12) keeps it in Redux, so it does survive. However, the entry view's mount effect calls `api.clear()` every time it mounts (`features/sales/salebill/components/sale-bill-entry-view.tsx:367-394`).
- Both pages also start on the list (`page.tsx`, `useState<View>({screen:"list"})`).

**User decisions:**

| Topic | Decision |
|---|---|
| Approach | Remember the draft only; no MDI tab strip |
| Scope | Every screen that holds unsaved entry work, both bill screens first |
| On return | Restore with a small one-time message, e.g. "Your unsaved bill was restored." |
| Lifetime | Kept until logout or user change. Being in memory, it is also lost on reload or tab close; the IndexedDB crash recovery still covers new bills. No TTL. |

Next 16's built-in `cacheComponents` / `<Activity>` was ruled out: it is a global switch and keeps only 3 pages (`node_modules/next/dist/client/components/bfcache.js:13`).

## Core: in-memory screen memory

**`lib/screen-memory.ts`** (new). Build it as `createScreenMemory({ currentUserId })`, the same factory shape as `features/sales/till/shared/session-store.ts`, so it can be tested.

- **Scope and key.** `ScreenScope = { companyId, branchId, accYear }`. The internal key is `${userId}|${screenKey}|${company}|${branch}|${year}`, giving one slot per tenant: a draft started in branch A comes back when you return to branch A and never appears in B.
- **API:**
  - `remember(key, scope, value)` does nothing when the user is null or the scope is incomplete.
  - `recall(key, scope)` is read-only and pure, because StrictMode runs lazy initialisers twice.
  - `has(key, scope)`.
  - `forget(key, scope?)`; with no scope it drops every scope.
  - `forgetAll()`.
  - `onAuthChange(userId)` clears everything unless it is the same user; token refreshes fire the same event.
- **Storage.** Values are stored by reference, with no serialisation. An LRU cap of about 40 entries stops growth. There is no TTL.
- **Auth wiring.** A module-level `window.addEventListener(AUTH_SESSION_EVENT, …)` (`lib/auth/session.ts:4`) calls `onAuthChange`, copying `session-store.ts:193-198`. Logout is a `router.replace` and does not reload, so this listener is what clears memory.

**`lib/use-screen-memory.ts`** (new). These are thin glue hooks; all decisions live in pure functions so the node-environment vitest can test them.

- `useRecalledOnce(memory, scope)` reads memory once in a `useState` initialiser.
- `useRememberWrite(memory, value, { scopeOf, shouldKeep, armed? })` is an effect on `value`. It calls `remember` when `shouldKeep(value)` is true and `forget` otherwise. The cost is one Map set per change.
- `useForgetIfWritingOnUnmount(memory, busyRef, scopeOf)` drops the draft if a save or hold was still running when the screen unmounted, so a finished save can't come back as a draft and be billed twice. It is StrictMode-safe because nothing is busy at mount.

Screens opt in by passing `memory={{ key, resume }}`. Deep-link routes such as `/accounts/receipt/[company]/…` and the vouchers' `initialKeys` never touch memory.

**`features/sales/document-memory.ts`** (new) holds the shared predicate for bill, quotation and sale order:

```
mode === "entry" && !isDeleted && (holdId || isDirty || (!docId && lines.some(l => l.itemId)))
```

- Kept: a resumed held cart, copy-as-new, an accepted recovery, an imported quotation or order.
- Not kept: a blank bill with only the walk-in customer or auto charges, a browsed bill, a clean saved draft. Those reload from the server.

**Restore message.** When a resume finishes, show one `notify` info message through `@/lib/notify` (memory: there is no react-toastify), e.g. "Your unsaved bill was restored." Each screen supplies its own noun.

## Phase 1: both bill screens (the reported bug)

### Testbill

**`features/sales/testbill/page.tsx`**
- Export `useBillDraftScope` from the hook's `useDraftContext` (`use-bill-draft.ts:596`).
- The view initialiser becomes `screenMemory.has("sales/testbill", scope) ? {screen:"entry", mode:"entry", resume:true} : {screen:"list"}`.
- Use `key={view.resume ? "resume" : view.document?.sbId ?? "new"}` and pass `memory={{ key, resume }}`.
- `onCreate` and `onOpen` leave `resume` false.

**`components/sale-bill-screen.tsx`**
- Add the `memory` prop and call `useSaleBillDraft({ memory })`.
- In the mount effect (366-393), before `api.clear()`, add `if (memory?.resume && api.resumed) { void api.finishResume(); return; }`. This skips clear, load and `findRecovery`.
- In `runGuarded("back")` (~1154, also reached by Esc), call `api.forgetRemembered()` before `onClose()`. That is a confirmed discard.
- Fix the stale "global slice" comment at 362-364.

**`state/use-bill-draft.ts`**
- `useSaleBillDraft(opts = {})`; the reducer initialiser becomes `recalled ?? createBillDraft(...)`.
- `seededTenant` (763) starts at the current scope signature when the draft was recalled. The first seed then doesn't re-stamp the restored draft, and a later company switch hits the existing "this bill keeps its tenant" branch.
- Add a `busyRef` that mirrors `setBusy` (715), and call `useForgetIfWritingOnUnmount` for the busy states "saving" and "holding".
- Call `useRememberWrite(memory, draft, { scopeOf, shouldKeep: isLineDocumentWorthRemembering })`.
- New `finishResume()`: run `reconcileRestored` (2580), then `verifyHoldLease()` if there is a `holdId`, then show the restore message.
- Return `resumed`, `finishResume` and `forgetRemembered`.
- Unit options refetch on their own (`lineItemIds` effect, ~975). Promotions recompute through their signature effect.

### Sale Bill (Redux)

**`features/sales/salebill/page.tsx`**
- Same as the testbill page with key `"sales/sale-bill"`.
- The view initialiser also requires `canResumeSaleBill(store.getState().saleBill, scope)`. If a save finished after the screen unmounted, Redux is clean and the user should land on the list.

**`components/sale-bill-entry-view.tsx`**
- Same resume branch in the mount effect (367-394), and `forgetRemembered()` in `runGuarded("back")` (~1177).

**`use-sale-bill-draft.ts`**
- Compute `resumed` once at init, from the live Redux draft plus the predicate plus a scope match. `useAppStore` is already imported.
- Pre-seed `seededTenant` (~670) the same way as testbill.
- Add a write gate: a `memoryArmed` ref that is true when resumed or after `clear()` (1416) / `loadDocument` (2215). Without it, the previous session's slice contents get written before the open decision is made.
- `finishResume()` runs `refreshPartyContext()` (1240), `refreshOpenCredits()` (1503) when there are adjustments, then `verifyHoldLease()`, then the message.
- No busy-on-unmount rule is needed: Redux still receives the save result after unmount, and the page re-checks it.

### Shared hold-lease check

**`features/sales/hold-lease.ts`** (new) holds `verifyHoldLease`, shared by both bills and the quotation, with the RTK triggers passed in.

- Re-read the hold.
- If the hold is LOCKED to this device and `holdLeaseIsLive` (`features/sales/quotation/quotation.hold.ts:359`) is true, rebuild `lockedHold` with `holdLockScope`.
- If the lease lapsed, re-take it with `/resume`.
- On 403, 404 or 409, or if the hold was converted: dispatch `holdSet({holdId:null, holdNo:""})` and warn with `holdLockMessage`. The bill continues as an ordinary new bill.
- Also add a warning in `convertHoldIfAny` when conversion fails; today it swallows the error silently.

### Autosave interplay

Both bill screens share the IndexedDB database `erp-sale-bill` and the device key. Skip the "Recover the bill…" offer when the other bill screen has a remembered draft (`screenMemory.has(otherKey, scope)`); otherwise the same cart could end up on both screens.

### Discard paths

| Action | Memory |
|---|---|
| F7 Clear, F8 picking another bill, save / post / tender then clear, hold (F9) then clear, delete, cancel | Forgotten automatically, because the predicate turns false |
| Back / Esc after a confirmed discard | Forgotten explicitly |
| Resumed hold, copy-as-new, import, accepted recovery | Kept |

## Phase 2: other transaction entry screens

The pattern is the same for each: page view from `has()`, `memory` prop, recall into the form state, write-through, a resume-guarded mount, `forget` on confirmed back.

| Screen | Notes |
|---|---|
| Quotation (`features/sales/quotation`, Redux) | Mount reset is in `quotation-entry-view.tsx:236-264`; the file has non-text bytes, so use `grep -a`. The page already lands on a new entry. Uses the shared predicate and `verifyHoldLease` (`lockedHold` 312, `seededTenant` 326). |
| Sale order (`features/sales/sale-order`, Redux) | Mount reset at `sale-order-entry-view.tsx:283-299`; `seededTenant` 277; shared predicate. |
| Receipt (`features/accounts/receipt`, `useReducer`) | Predicate is `draft.dirty`. Start `seededTender` (267) at true on restore. Remember `tempCreditMobile` (321-322) with the draft. With a `?collect=` hand-off as well, resume first and then collect: `collectTempCredit` already asks before discarding. |
| Payment (`features/accounts/payment`) | `draft.dirty`; `seededTender` 197. |
| Expense (`features/accounts/expense`) | Remember `{ typed, autoGstin }`; predicate is `dirty && editable`. |
| Opening balance | Remember `{ mode, draft }`. **Guard** the `LOADED` effect (`use-opening-draft.ts:92-107`): it wipes a dirty restore as soon as the cached list arrives. Extract `shouldApplyLoaded()`. |
| Vouchers (8 types + register) | Key `accounts/vouchers/${typeCode}`. Remember `{ doc, lines, bills, dirty, typeCode }` from `use-voucher-entry.ts:399-404`. |
| Physical stock | `dirty && editable`. Don't wait for settings when resuming. |
| Stock adjustment / Opening stock | Write memory inside the existing `commit` / `apply` funnels, so it keeps working after unmount. Skip the stock adjustment reasons effect (249-259) once after a restore. |
| Selling price | Remember `{ rows, addedItems, filter, scopeChoice, lastLevel }` when any row has changed. Skip the filter-driven load on restore. |
| Settlement, cheques, temp credits | **Excluded.** These are action screens with no unsaved entry. |

## Phase 3: masters

**CrudMasterPage modals.**

Turn on `modalDraftKey` page by page, not by default. A key derived from the route would collide for embedded uses, and pages with derived state need `onModalDraftApplied`.

New auto-reopen in `components/master/crud-master-page.tsx`, only when `modalDraftKey` is set:
- Around `onModalOpenChange` (~4917), store `{variant:"new"} | {variant:"edit", id}` in screen memory on open and forget it on close. A navigation unmount doesn't close the modal, so the marker survives.
- When the page is ready and permissions have loaded, recall the marker. If `readFormDraft(...)` (`lib/form-drafts.ts:117`) exists, call `openCreateModal()` (3260) or `openUpdateModalById(id)` (3305), checking create or edit rights.
- The existing "restored" bar with its Discard button serves as the one-time message.

Rollout tiers:
- **Tier A** (only `modalDraftKey` needed): charge, unit, ledger bank account, ledger shipping address, supplier groups, area, customer groups, freight charges, loading charges, state, tender, employee designation, employee department, GSP provider, GSP service, widget master, item section. Companies and branches modules still need auditing.
- **Tier B** (need an `onModalDraftApplied` re-sync like Customer's at `customer/page.tsx:1843`): city, item brand, item category, item group, ledger groups, tax rate, device list, GSP company service, employee, suppliers, godown, item master.
- **Custom master screens:**
  - account ledger (`{ open, editingId, formValues, section }`)
  - item qty price (`{ activeItem, rows }` when dirty)
  - module administration (`{ visibility, original }` when dirty)
  - still to audit: user entry dialog, app settings, promotion loyalty
  - excluded: templates and the dropdown, grid and UI-table designers

**Reports** (day summary, ledger statement, party outstanding) are excluded. They hold filters only, and restoring an old "as on" date would mislead.

## Tests (vitest, node environment)

- `lib/screen-memory.test.ts`, in the style of `session-store.test.ts`:
  - recall returns the same reference;
  - a different scope or user returns null;
  - `forget` with a scope versus without one;
  - `forgetAll`;
  - `onAuthChange` with the same user versus another user or null;
  - LRU eviction;
  - remember with a null user or incomplete scope does nothing;
  - recall has no side effects.
- `features/sales/document-memory.test.ts`: a predicate table built from the testbill factories (`createBillDraft`, `draftFromBillHold`, `copyBillDraftAsNew`, `resetBillDraft`, `draftFromAutosave`; `CONTEXT` fixture from `state/hold.test.ts`).
- `features/sales/testbill/state/remember.test.ts`:
  - a restored draft survives `tenantSet(sameScope)` with dirty state and lines intact;
  - edits still apply;
  - a reset makes the predicate false.
  - Salebill twin: `saleBillReducer(restored, tenantSet(same))` returns the **same reference**.
- Pure helpers: `canResumeSaleBill`, `shouldApplyLoaded`.

## Verification

1. `npx vitest run lib/screen-memory.test.ts features/sales`, then `npx tsc --noEmit`. Lint only the touched files: `eslint .` crashes on the existing baseline.
2. In the browser, using the snap-chromium CDP recipe from memory, on `/sales/testbill` and `/sales/sale-bill`:
   - Create a bill, add 2 items and a customer, open another menu screen, return. Expect the entry view with the same lines and the "restored" message.
   - F7 Clear, leave, return. Expect the list.
   - Back with a confirmed discard, then Create. Expect a blank bill.
   - Save, leave, return. Expect the list, with no resurrected draft.
   - Resume a held cart (F10), leave, return. Expect the cart back and the lease re-verified. Then save and confirm the hold converts.
   - Switch branch while away, return. The draft is not shown; switch back and it returns.
   - Logout and log back in. Nothing is restored.
3. Repeat the add, leave, return check for each Phase 2 screen, and for a Tier A master modal (open Create, type, leave, return: the modal reopens with the values).
