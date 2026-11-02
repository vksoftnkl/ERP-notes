---
name: gst-config-module
description: "src/modules/gst (notes 79 R1–R9, 2026-10-02) — GST provider/credential masters + Verify; needs GST_CRED_KEY env or secret saves/Verify 503; menus found by name"
metadata:
  node_type: memory
  type: project
  originSessionId: 9a575c74-3aa9-4652-8e4f-e271163504b6
  modified: 2026-10-02T11:20:51.127Z
---

Notes 79 (share `gst/notes-79-gsp-config-routes.md`) built 2026-10-02 as `src/modules/gst/`:
`/gst/providers|provider-services|provider-endpoints|provider-field-maps|provider-error-maps|provider-accounts|company-credentials/*`
on the `public.gst_*` tables. README in the module.

Non-obvious facts:
- **Secrets need `GST_CRED_KEY`** (32 bytes, hex or base64) in the server env, else 503 `GST_CRED_KEY_MISSING`.
  Since 2026-10-03 (notes 82) the dev box's `.env` (192.168.0.106) has one, v1, generated in place and never
  printed; the user was told to back it up off-box. NEVER regenerate/overwrite it: every stored secret dies.
  Other servers (VPS, CloudJiffy) still need their own.
  Ciphertext is `gcm:v<n>:iv:tag:ct`; `*_key_version` = the key version (`GST_CRED_KEY_VERSION`,
  default 1), NOT a write counter — notes 79 said "bump"; interpreted as "stamp current version".
- **Menus resolved by (parent 60, name)** 'GST Providers' / 'GST Credentials' — migration
  `20261002150000_gst_menus`, hidden. Local ids are 269/270 (a BEGIN/ROLLBACK dry run burned 267/268:
  sequences don't roll back), the share file predicted 267/268.
- **Rights:** create for a new row, edit for update/restore, delete, view for get/status, post for Verify.
- **Verify** refuses before the portal: 409 `GST_AUTH_BUSY` (lease held) and `GST_AUTH_RATE_LIMIT`
  (4 AUTH rows in gst_api_log per company+branch in 15 min — an addition beyond notes 79, because NIC
  blocks a GSTIN at 5). First sign-in inserts a `gas_token_type='PENDING'` placeholder session row
  to have something to lease.
- NIC auth body = RSA-PKCS1(base64(JSON)) — written from memory of the v1.04 spec, never run against the
  real sandbox (no credentials exist).

**How to apply:** for the later phases (document routes, worker) reuse `GstCryptoService` and
`GstAuthService` (exported). See [[gstin-search-runs-on-gsp-rows]] for /gst/search, which now runs on these rows, and
[[concurrent-claude-sessions]].

**Seed fixes never reach an existing box** — `prisma/seed/Gsp_Providers.sql` is the whole catalogue and
returns early once provider NIC exists (runs every boot as a no-op). A config correction (e.g. notes 83:
CHARTERED sandbox https, AUTH on `/eivital/dec/v1.04/auth` with `eInvPwd`, success path NULL, errors at
`$.error.*`, auth_token ← `$.Data.AuthToken`) must ALSO be applied to deployed servers via the R-routes or a
guarded UPDATE migration. The Qt side's "11_gst_provider.sql / 99_seed_all.sql" do not exist in this repo.
Dry-run recipe: BEGIN; rename gpv_code NIC/CHARTERED to *X; \i the seed; EXCEPT-diff against *X; ROLLBACK.

**Notes 88 (2026-10-03): no active provider, no GST call.** `client/gst-route-guard.ts` refuses 503
`GST_SWITCHED_OFF` naming the first row off: provider → service → endpoint → account (only when the
endpoint's templates need {aspId}/{aspPassword}/{apiKey}, or a {clientId}/{clientSecret} the credential
lacks) → credential. Verify NO LONGER tries inactive rows (README's "Verify tests an inactive provider"
is gone). `GstHttpClient.send` REQUIRES `request.route` and re-checks it before I/O — any new outbound
call must pass its rows. Guard runs before GSTIN check / budget / lease, so no gst_api_log row.
Live sessions are left alone. Test a provider by switching it on (or in a rolled-back tx: notes-88
checks recipe flips gpv/gps/gpe/gpa is_active inside `prisma.$transaction` with a transactional proxy).

