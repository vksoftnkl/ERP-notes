---
name: erp-client-ui-browser-verification
description: How to drive the ERP client UI headlessly (playwright-core + cached chromium + sessionStorage auth seed) for end-to-end verification
metadata: 
  node_type: memory
  type: reference
  originSessionId: 00dbddb0-5613-44bc-b7ac-95ec2bd94ac4
  modified: 2026-10-26T05:21:34.964Z
---

To verify client changes in a real browser against the running dev stack (client `next dev` on **https://127.0.0.1:3001** as of 2026-09-21, API :3011):

1. **Browser:** no playwright in the repo. `npm i playwright-core` in the scratchpad, then drive **snap chromium** (`/snap/bin/chromium` — the old `~/.cache/ms-playwright` download is gone as of 2026-07-28). Snap confinement blocks profile dirs outside the snap's home, so use `launchPersistentContext("/home/vk/snap/chromium/common/<name>", {executablePath: "/snap/bin/chromium", headless: true, args: ["--no-sandbox"]})`; a `/tmp` or `~/.cache` user-data-dir fails with `SingletonLock: Permission denied`. Delete the profile afterwards — it holds the seeded session.
2. **Auth bypass:** mint a JWT + Redis session per [[erp-auth-endpoint-verification]], then seed the client session before navigation with `page.addInitScript`: set `sessionStorage["erp_client_redux_state"]` to `{auth: {initialized:true, isAuthenticated:true, token, refreshToken:null, userId, userInfo:{...nullable fields}, recentPages:[], businessContext:null}}` — the route guard rehydrates Redux from this key and lets protected routes render.
3. Observe network via `page.on("request")`; master list pages hit `/configured-grid-sql/run?grid_id=N&page=&limit=`. The client's API base is `NEXT_PUBLIC_API_BASE=http://127.0.0.1:3011/api/v1` (`/api` alone 404s), and the DB password is in `ERP server/.env` `DATABASE_URL` (not `postgres`).
4. A recent successful login user id can be read from `audit.user_login_sessions` (uls_user_id); don't touch `public.user_master` (PII, permission-denied).

Known pre-existing bug seen while probing: account-ledger page footer shows "Showing 0 to 0 of 0 entries" even when rows render (its local `totalEntries` state never gets set from the response); present before the auto-fit change.


## Faking the API instead of the session (2026-09-21)

Minting a JWT needs `JWT_SECRET`, and **reading it is refused in auto mode**
(credential materialization). When the thing under test is client-side, skip the
real session entirely:

- `context.route("**/api/v1/**", ...)` and `route.fulfill` every call. Send
  `access-control-allow-origin: <the page origin>` + `-headers`/`-methods` on
  every reply and answer `OPTIONS` with 204, or the preflight for a JSON POST
  with an `Authorization` header kills the request.
- Any token-shaped string works in the seeded session, since nothing verifies it.
  Serve 401 for the stale one and 200 for the refreshed one to drive a whole
  expiry/refresh/retry chain.
- **Seed sessionStorage once, not once per load**: `addInitScript` re-runs on
  every navigation, so it hands the session back after a logout redirect and the
  guard bounces straight back to the page under test — which looks exactly like
  "the redirect never happened". Guard the seed with a marker key that
  `clearAuthSession` does not touch.
- State and toasts die with the page on that redirect (`window.location.replace`),
  so mirror anything to be asserted into sessionStorage as it happens (a
  MutationObserver on `.Toastify__toast`) and read it after the navigation.
- A throwaway page under `app/dev/<name>/page.tsx` (precedent: `app/dev/nex-dropdown`)
  exercises a hook directly in the real app — far cheaper than driving a master
  screen's UI. Delete it, and the chromium profile, afterwards.

**Update 2026-09-24 — no playwright needed.** `playwright-core` is not installed
in the client checkout, and installing it touches a `node_modules` other
sessions share. Node 22 has a global `WebSocket`, so snap chromium can be driven
over CDP directly: launch `/snap/bin/chromium --headless=new
--remote-debugging-port=9333 --ignore-certificate-errors --window-size=...`,
read the page target from `http://127.0.0.1:9333/json/list`, then
`Page.enable` → navigate to the origin → `Runtime.evaluate` to seed
`sessionStorage["erp_client_redux_state"]` with `{auth:{token, userId, userInfo,
businessContext, isAuthenticated:true, initialized:true}}` → navigate to the
page → `Page.captureScreenshot`. About 40 lines, no dependency. The JWT is
minted as in [[erp-auth-endpoint-verification]] (note the server's real
`JWT_SECRET` is the SECOND assignment in its `.env`, not the first).

**Cleanup trap (2026-09-25): snap chromium cannot be killed from here.**
`kill`/`pkill` get EPERM even outside the sandbox (AppArmor snap confinement),
and a CDP driver whose `finally` calls `chrome.kill()` leaks one whole headless
browser per run (the later ones cannot even bind the debug port). End every
driver with CDP `Browser.close` on `/json/version`'s websocket. For leaks that
already happened: read each PID's `/proc/<pid>/cgroup` for its
`snap.chromium.chromium-<uuid>.scope`, confirm the scope holds only the
verification instance, then `systemctl --user stop <scope>` (needs the sandbox
off). A throwaway `features/**/zz-*.tmp.test.ts` run with vitest makes a
convenient driver: it can import the TS fixtures directly and fulfil
`Fetch.requestPaused` with them.


**Seeded `userInfo.deviceId` must be a real `device_master.dev_id` (2026-10-26).** A made-up or
non-v7 id makes `/app-setting-values/effective` 400 ("deviceId must be a valid UUID"), and every
screen silently falls back to setting DEFAULTS (e.g. textile columns stay hidden). Find the device
table's schema via information_schema (it is not `public`). The minting script can read `.env`
itself and never print the secret — that worked in auto mode.

## Fake-API CDP recipe that worked end to end (2026-10-02)

Used for company/branch ([[erp-company-branch-qt-parity]]); scripts were
`cdp.mjs` + `fixtures.mjs` in that session's scratchpad (gone now — rebuild):
- Launch with `--disable-web-security --user-data-dir=/home/vk/snap/chromium/common/<name>`
  and CORS/preflight stop mattering; `Fetch.enable` on `*/api/v1/*` and
  `Fetch.fulfillRequest` everything. Close with `Browser.close`, then rm the profile.
- Leave `/menu-masters/usermenu` PAUSED (never fulfil): the permission gate
  fails open while the menu "loads", so governed master routes render.
- Default reply `{success:true,data:[]}`; grid/dropdown runs want
  `{data:{items,meta:{total}}}`. The shell also calls grid ids 34/38/43
  (directories), `/app-setting-values/effective`, `/grid-details/get`.
- Form DOM: fields are `[data-erp-modal-field-name]`; selects (searchable or
  not) are `role=combobox` inputs whose `.value` is the LABEL; pick an option
  by dispatching `mousedown` on the input, then on `#<id>-search-list [role=option]`.
  Only the ACTIVE tab's fields exist in the DOM; tab text gains an error marker
  after a failed save (match by prefix); checkbox-group marks invalid by class,
  not `aria-invalid`. The message popup is `[role=alertdialog]`; dismiss with
  Enter only while it is open, or the Enter lands on the form.


## Grid responses are cached ~1 s server-side (2026-10-07) — trap FIXED 2026-10-15

(Since 2026-10-15 every successful write purges the whole server HTTP cache before it
answers, so the stale-after-delete trap below no longer happens; kept for history.)


`configured-grid-sql/run` has `@CacheTTL(1)` (cache-manager 7, in-process — no
Redis keys). Opening a shell modal fires a freshness-bus list re-read, so a
script that confirms a delete within ~1 s gets the CACHED pre-delete rows back
and the deleted row "stays". Pause >1.5 s before confirming; humans never hit it.

## Seeding the fake session: use an init script (2026-10-03)

Seeding via `Runtime.evaluate` after a first navigation FAILS two ways:
navigating to `/favicon.ico` lands on a 404 error document where
`sessionStorage` throws SecurityError, and navigating to `/login` runs the
app, whose store re-persists `token:null` 250 ms later over the seed. What
works: `Page.addScriptToEvaluateOnNewDocument` that writes
`erp_client_redux_state` (guarded by a `__seeded` marker) before any app code,
then navigate straight to the route. Voucher screens open on the list —
click the toolbar's "Add" to reach the entry. Headless chromium shows native
date inputs as mm/dd/yyyy even with `--lang=en-IN` or
`Emulation.setLocaleOverride`, so don't judge date format from a headless shot.

## Close the fake-API gap: run captured bodies through the server DTOs (2026-10-07)

A faked API accepts any body, so record every POST body in the fake and then
validate them with the server's own DTOs + the API's pipe options (`whitelist`,
`forbidNonWhitelisted`, `enableImplicitConversion`): a tsx script that imports
`src/modules/.../dto/*.ts`, `plainToInstance` + `validate`. Run it FROM the server
repo as `NODE_PATH="<server>/node_modules" npx tsx --tsconfig tsconfig.json <script>`
— without NODE_PATH a script outside the repo cannot resolve `reflect-metadata`.
Fixtures pulled read-only from the dev DB (`json_agg` over the grid's own SQL with
tokens sed-substituted) make the screens render real rows. Used for
[[erp-gst-config-screens]]. In `waitFor` expressions wrap in `!!(…)`: a DOM node is
not serializable by `returnByValue` and reads as falsy.

## Sale bill entry under the fake API (2026-10-08)

`/sales/sale-bill` opens on the list; click the button whose text is exactly
"Add". The draft needs a company/branch or customer picks are refused: seed
`auth.businessContext {companyId, branchId, …}` AND fake
`/master-lookups/name-id/all-masters` (`[{compId, compName}]`),
`/master-lookups/branches/by-company/<co>` (`[{brId, brName}]`),
`/master-lookups/fiscal-years/by-company/<co>` (`[{id, name, isCurrent}]`) and
`/company-masters/get` (`{compStateCode}`). The server's tsconfig is CommonJS,
so a DTO-validation tsx script must wrap its `await`s in an async IIFE.
