# Notes 82 — GST_CRED_KEY is not set on the box (every GST secret save is a 503)

**From:** Qt client (Viji / Claude) · **Date:** 2026-10-03 · **For:** Prathap

## What happened

We are setting up the TaxPro (CHARTERED) **SANDBOX** account with the sandbox details Chartered gave
us. Saving the provider account from GST Provider (menu 269) answers:

```
503  GST credential key unavailable
• GST_CRED_KEY — GST_CRED_KEY is not set on this server, so no GST secret can be stored or read
```

This is the guard in `gst-crypto.service.ts`, working as designed. The API's environment on
192.168.0.106 has no `GST_CRED_KEY`. Until the API has one, we can't save the ASP id or password,
a credential's portal password, or anything else that is a secret. **Verify (R8) is blocked too**,
because it has to decrypt those secrets.

## The ask

On the API server (192.168.0.106):

1. Generate the key once:
   ```
   openssl rand -hex 32
   ```
2. Add it to the API's environment (`.env`, or wherever the service reads its env):
   ```
   GST_CRED_KEY=<the 64 hex characters>
   GST_CRED_KEY_VERSION=1
   ```
3. Restart the API.

**Keep a copy of the key outside the box** (a password manager, for example). If the key is lost or
regenerated, every stored ASP or portal secret becomes unreadable and has to be entered again. If it
ever has to change, rotate it the way the gst README describes (new key + version, the old one kept
as `GST_CRED_KEY_V1`), not by overwriting it.

While you're restarting the API, the `GST_LOOKUP_*` variables from notes 79 §7 (GSTIN fetch, still
answering 503) can go in at the same time.

## How we'll check it

- GST Provider → CHARTERED → SANDBOX account: enter Client ID (aspid) and Client secret, then Save.
  The card should then show `set · key v1` for both.
- GST Credential (menu 270): add a CHARTERED / EINVOICE / SANDBOX login with its portal password,
  Save, and press **Verify**. If Verify still fails, it should be the sandbox's own answer, not 503.

Nothing else is needed from the client side. The Qt screens already send the secrets write-only.
