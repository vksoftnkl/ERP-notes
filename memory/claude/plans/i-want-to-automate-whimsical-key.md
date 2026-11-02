# Auto-deploy the frontend to the live VPS on push (with approval)

## Context

Today a push to `dev` does **not** reach production. The live site
(https://169-58-213-171.sslip.io/, PM2 app `erp-front` on :3001, user `erp-front`)
is updated only by hand:

```
ssh root@169.58.213.171 'cd /opt/erp-front && sudo -u erp-front HOME=/opt/erp-front /opt/erp-front/bin/deploy.sh dev'
```

The repo's `.github/workflows/ci.yml` still has a `deploy` job wired to the
**retired CloudJiffy node** (secrets `CLOUDJIFFY_*`, pipes
`scripts/deploy-on-node.sh` over SSH). It can never succeed, and its `build`
job is red on every push because `npm run lint` fails before any change
(verified locally today: typecheck 0, test 0, lint 2).

Goal, as decided with the user:

| Decision | Choice |
| --- | --- |
| Scope | **Frontend only** (this repo). Server keeps its manual `deploy.sh`. |
| Trigger | Push to `dev` → build gate → **pause for the user's approval** in GitHub → deploy to the VPS. |
| Gate | **Only the production build must succeed.** No typecheck/lint/test gate. |

Everything the VPS needs already exists there (`/opt/erp-front/bin/deploy.sh`
does fetch → reset → `npm ci --omit=dev` when needed → build into `.next-build`
→ swap → `pm2 reload` → 90 s health gate, see README "Deploying a new version").
The work is: let GitHub Actions *trigger* that script safely, and make the
pipeline honest about what it deployed.

Constraint discovered while planning: this session's permission classifier
blocks SSH reads of the VPS ("Production Reads"). The VPS steps below are
therefore written as commands for the user to run (or to approve one by one);
nothing on the box can be verified from here beforehand.

## Design

```
push to dev ──► job build  (ubuntu-latest, node 22, npm ci --omit=dev,
                            NEXT_DIST_DIR=.next-build node scripts/ensure-build.js --strict)
                   │  green
                   ▼
              job deploy  environment: production  ──► ⏸ "Review deployments" → user approves
                   │
                   ▼
              ssh -i <deploy key> root@169.58.213.171      (key is restricted to ONE forced command)
                   └─► /opt/erp-front/bin/ci-deploy.sh
                          └─► sudo -u erp-front … /opt/erp-front/bin/deploy.sh dev   (unchanged script)
                   ▼
              assert the log shows the tip of origin/dev went live, curl public URL = 200
```

Why these choices:

- **Build gate mirrors the VPS exactly** (`npm ci --omit=dev` + `ensure-build.js --strict`,
  i.e. plain `next build` = Turbopack on Next 16.1.6, not `--webpack` as the old job did). The VPS installs
  production deps only, so the classic trap — a build-time package landing in
  `devDependencies` — is caught on the runner, before anyone is asked to approve.
- **Approval = GitHub Environment `production` with Required reviewers.** The
  run shows "Waiting for review"; the user clicks *Approve* in the Actions tab
  (or from the e-mail). Environment-scoped secrets mean the SSH key is only
  readable by an approved job. Works on the free plan because the repo is public;
  if it is ever made private, required reviewers need GitHub Pro/Team.
- **Forced-command SSH key, not a shell key.** The key lands in
  `/root/.ssh/authorized_keys` with `restrict,command="/opt/erp-front/bin/ci-deploy.sh"`.
  Whatever command the client asks for, sshd runs only the wrapper — no shell,
  no pty, no forwarding. A leaked `VPS_SSH_KEY` can redeploy `dev`, nothing else.
  (Putting the key on the `erp-front` user instead would need that system user
  to gain a login shell; root + forced command changes nothing about users.)
- **Host key pinned** via a `VPS_KNOWN_HOSTS` secret instead of `ssh-keyscan` at
  run time (the old job trusted whatever answered).
- **The VPS deploy script is left as-is.** It always deploys the *tip* of
  `origin/dev`, regardless of which run was approved. The workflow therefore
  asserts against `git ls-remote origin dev` at deploy time and prints a warning
  when that differs from the run's own commit (= a newer push already landed).
  Stale "waiting" runs should simply be rejected.

## Changes

### 1. Repo: `.github/workflows/ci.yml` (rewrite)

- `on`: `push` → `[dev, main]`, `pull_request` → `[dev, main]`, `workflow_dispatch` (redeploy without a push).
- Top-level `permissions: contents: read`.
- **job `build`** — `timeout-minutes: 20`, `concurrency: ci-build-${{ github.ref }}` with
  `cancel-in-progress: true` (a superseded push's build is pointless).
  Steps: checkout → `actions/setup-node@v4` (22.x, `cache: npm`) →
  `npm ci --omit=dev` → `NEXT_DIST_DIR=.next-build NODE_OPTIONS=--max-old-space-size=4096 node scripts/ensure-build.js --strict`
  with `NEXT_PUBLIC_API_BASE: ${{ vars.NEXT_PUBLIC_API_BASE || 'https://169-58-213-171.sslip.io/api/v1' }}`
  and `NEXT_TELEMETRY_DISABLED: 1`; then `test -f .next-build/BUILD_ID`.
  Drop the typecheck / lint / test steps (user's decision; they can be re-added
  as blocking later with one step each — typecheck and test are green today).
- **job `deploy`** — `needs: build`,
  `if: github.event_name != 'pull_request' && github.ref == 'refs/heads/dev'`,
  `environment: { name: production, url: https://169-58-213-171.sslip.io/ }`,
  `concurrency: { group: deploy-vps-front, cancel-in-progress: false }`
  (never interrupt a deploy mid-flight; queue instead), `timeout-minutes: 25`.
  Steps:
  1. Guard: fail fast if `secrets.VPS_SSH_KEY` / `secrets.VPS_KNOWN_HOSTS` are empty.
  2. Write `~/.ssh/deploy_key` (600) and `~/.ssh/known_hosts` from the secrets.
  3. `expected=$(git ls-remote https://github.com/${{ github.repository }} refs/heads/dev | cut -c1-7)`;
     warn (`::warning::`) if `$expected` ≠ `${GITHUB_SHA:0:7}`.
  4. `ssh -i ~/.ssh/deploy_key -o IdentitiesOnly=yes -o ConnectTimeout=20 -o ServerAliveInterval=30 ${{ vars.VPS_SSH_TARGET }} deploy 2>&1 | tee deploy.log`
     (the word `deploy` is ignored by the forced command; it's there for readability).
  5. `grep -q "$expected" deploy.log` — the VPS script prints the commit it
     brought up; if the short sha is not in the log, fail the job.
  6. Public check: poll `https://169-58-213-171.sslip.io/` for HTTP 200 (up to ~100 s).
- Remove everything CloudJiffy-specific (secrets, port 3022, the stdin-piped script, the cloudjiffy URL check).

### 2. Repo: `deploy/vps/ci-deploy.sh` (new — reference copy of the VPS wrapper)

Same pattern as the server repo's `deploy/` folder: the file that really runs
lives on the VPS, the repo keeps the source of truth. ~15 lines:

```bash
#!/usr/bin/env bash
# Forced command for the GitHub Actions deploy key (see README "CI/CD").
# sshd runs this whatever the client asked for; SSH_ORIGINAL_COMMAND is ignored
# on purpose so the key can only ever redeploy dev.
set -euo pipefail
cd /opt/erp-front
exec sudo -u erp-front HOME=/opt/erp-front /opt/erp-front/bin/deploy.sh dev
```

Plus `deploy/vps/authorized_keys.example` with the one-line template:
`restrict,command="/opt/erp-front/bin/ci-deploy.sh" ssh-ed25519 <PUBKEY> github-actions:ERP-frontend`.

### 3. Repo: delete `scripts/deploy-on-node.sh`

Only the old `deploy` job used it. (`ecosystem.config.cjs` is also CloudJiffy-only
and the cause of the eslint crash, but it is out of scope here — leave it, note it.)

### 4. Repo: `README.md`

- "Deploying a new version" (L190–197): replace "Pushing to `dev` does **not** update
  production …" with the new flow: push → Actions → *Review deployments* → Approve →
  live in ~3–5 min; manual `deploy.sh` command stays as the fallback; `workflow_dispatch`
  for a redeploy without a push.
- "CI" (L252–261): describe the two jobs, the gate, the environment, the secrets/variables.
- Table at L178–184: add `/opt/erp-front/bin/ci-deploy.sh` and the authorized_keys line.
- "Known issues" (L271): drop the CloudJiffy deploy-job bullet.
- "Legacy: CloudJiffy" (L273–279): now only `ecosystem.config.cjs`.
- Project-structure line L132: `scripts/` no longer has the legacy deploy script.

### 5. VPS (user runs as root, or approves each command)

1. **Generate the key on the laptop**, never on the VPS:
   `ssh-keygen -t ed25519 -N "" -C "github-actions:ERP-frontend" -f /tmp/claude-1000/-home-vk-Dev-erp-ERP-client/bc76631b-ce09-416f-b8e1-f525d160e035/scratchpad/gha-erp-front`
2. **Install the wrapper**: `scp deploy/vps/ci-deploy.sh root@169.58.213.171:/opt/erp-front/bin/ci-deploy.sh && ssh root@169.58.213.171 'chmod 755 /opt/erp-front/bin/ci-deploy.sh'`
3. **Authorize the key** (append, don't overwrite):
   `ssh root@169.58.213.171 'printf "restrict,command=\"/opt/erp-front/bin/ci-deploy.sh\" %s\n" "$(cat)" >> /root/.ssh/authorized_keys' < …/gha-erp-front.pub`
4. **Prove the restriction**: `ssh -i …/gha-erp-front -o IdentitiesOnly=yes root@169.58.213.171 id` must run a deploy (prints `… -> … on dev`, `healthy`), *not* print `uid=0`.
5. **Host key for pinning**: `ssh-keyscan -t ed25519 169.58.213.171` → value for `VPS_KNOWN_HOSTS`.
6. Afterwards delete the private key from the laptop (`shred -u`) — it exists only in the GitHub secret.

### 6. GitHub (web UI; `gh` is not installed locally)

Repo `vksoftnkl/ERP-frontend` → **Settings**:

1. **Environments → New environment → `production`**
   - *Required reviewers*: add your own account (that is the approval click).
   - *Deployment branches and tags*: Selected branches → `dev`.
   - Environment **secrets**: `VPS_SSH_KEY` (contents of the private key file), `VPS_KNOWN_HOSTS` (ssh-keyscan line).
   - Environment **variable**: `VPS_SSH_TARGET` = `root@169.58.213.171`.
2. (Optional) repo variable `NEXT_PUBLIC_API_BASE` = `https://169-58-213-171.sslip.io/api/v1` — only affects the gate build; the live bundle still takes it from the VPS's `.env.production`.
3. Delete the dead `CLOUDJIFFY_SSH_KEY` / `CLOUDJIFFY_SSH_TARGET` secrets and the `CLOUDJIFFY_SSH_PORT` variable if present.

## Verification (end-to-end)

1. Commit the repo changes on `dev` (workflow + `deploy/vps/` + README + deletion) and push.
2. Actions tab: `build` goes green (watch that `ensure-build.js --strict` with
   prod-only deps builds on the runner; if Next's default bundler fails there for a
   runner-specific reason, fall back to `--webpack` and say so in the README).
3. `deploy` shows **Waiting for review** → click *Review deployments* → Approve.
4. The job log shows the VPS script's own lines (`<old> -> <new> on dev`,
   `healthy on :3001 at <sha>`), the sha-assert step passes, public URL returns 200.
5. Cross-check from the laptop: `ssh root@169.58.213.171 "git -c safe.directory='*' -C /opt/erp-front/app log -1 --oneline"` = the pushed commit; open the site, hard-refresh, confirm the change.
6. Negative checks: open a PR → only `build` runs, no `deploy`; *Reject* a waiting run → nothing changes on the VPS.
7. After it works: update memory `erp-frontend-vps-deploy.md` (CI/CD path, secrets names, forced-command key) and mark the CI/CD line in `erp-cloudjiffy-deploy.md` obsolete.

## Out of scope / follow-ups

- Server repo CI/CD (user chose frontend only). Same recipe applies: forced-command key → `/opt/erp-server/bin/deploy.sh` on branch `DEV-v1`.
- One-click rollback from Actions (would need a second allowed word in `SSH_ORIGINAL_COMMAND` + a rollback script on the VPS; today rollback stays the documented manual `mv .next.previous .next && pm2 reload`).
- Re-adding typecheck/test as blocking gates; fixing lint so it can gate too.
- Removing `ecosystem.config.cjs` (fixes the eslint crash as a side effect).
