# Audit — `oglasino-backend` deploy workflow

**Repo:** `oglasino-backend`
**Branch:** `dev` (read-only audit; no edits)
**Date:** 2026-05-15
**Scope:** `.github/workflows/` only, plus any in-repo scripts those workflows call. Read-only.

---

## Section 1 — Workflow inventory

`.github/workflows/` contains three files. (`.github/dependabot.yml` exists but is not a workflow and is out of scope.)

| Filename | Trigger | Purpose | Touches |
| --- | --- | --- | --- |
| `ci-dev.yml` | `push` to `dev`; `pull_request` targeting `dev`, `stage`, `main` | Compile + Spotless format check + SpotBugs static analysis on backend code, with a `verify-summary` aggregator job. | Neither (CI only) |
| `deploy-stage.yml` | `push` to `stage`; manual `workflow_dispatch` | Build a Docker image, flip stage maintenance KV on, SSH-deploy to stage droplet, verify backend health, flip stage maintenance KV off. | Stage |
| `deploy-backend.yml` | `push` to `main`; manual `workflow_dispatch` | Build a Docker image, flip prod maintenance KV on, SSH-deploy to prod droplet, verify backend health. Maintenance is **not** flipped off automatically. | Prod |

---

## Section 2 — Deploy workflows in detail

### 2A. `deploy-stage.yml` (stage)

**File-level config**
- `concurrency: group=deploy-backend-stage, cancel-in-progress=false` — serializes stage deploys (lines 8–10).
- `env.IMAGE_NAME = ghcr.io/${{ github.repository }}` (line 13).
- `env.MAINTENANCE_KEY = maintenance.active` (line 14).

#### Job: `build` (lines 17–49)

| # | Step name | Action / command | Env / secrets | Notes |
| --- | --- | --- | --- | --- |
| 1 | `actions/checkout@v4` | action | — | Default checkout. |
| 2 | `meta` | inline `run` (lines 28–31) — computes `TAG="$(date -u +%Y%m%dT%H%M)-${GITHUB_SHA::7}"`, exports as `steps.meta.outputs.tag`. | — | Job output `image_tag` is wired from this step (line 23). |
| 3 | `docker/setup-buildx-action@v3` | action | — | — |
| 4 | `docker/login-action@v3` (lines 34–39) | action | `secrets.GITHUB_TOKEN`; `username = github.actor` | Logs in to `ghcr.io`. |
| 5 | `docker/build-push-action@v5` (lines 41–49) | action | — | `push: true`. Tags pushed: `ghcr.io/<repo>:<tag>` and `ghcr.io/<repo>:stage`. `cache-from: type=gha`, `cache-to: type=gha,mode=max`. |

`continue-on-error` / non-fatal conditionals: none in this job.

#### Job: `maintenance-on` (lines 51–79) — `needs: build`

| # | Step name | Action / command | Env / secrets | Notes |
| --- | --- | --- | --- | --- |
| 1 | `Set maintenance.active = true (stage KV)` (lines 55–67) | inline `curl -fsSL -X PUT https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/${MAINTENANCE_KEY}` with `Authorization: Bearer ${CF_API_TOKEN}`, `Content-Type: text/plain`, body `true`. | Env: `CF_API_TOKEN=${{ secrets.CF_API_TOKEN }}`, `CF_ACCOUNT_ID=${{ secrets.CF_ACCOUNT_ID }}`, `CF_KV_NAMESPACE_ID=${{ secrets.CF_KV_NAMESPACE_ID_STAGE }}`. Inherits `MAINTENANCE_KEY` from workflow `env`. | `set -euo pipefail`; `curl -f` so a non-2xx KV PUT fails the step. |
| 2 | `Wait for KV propagation` (line 70) | `sleep 0` | — | No real wait. Comment line 69 names it as "wait for KV propagation" but it sleeps zero seconds. |
| 3 | `Verify maintenance is live at stage edge (advisory)` (lines 72–79) | inline `curl -s -o /dev/null -w "%{http_code}" --max-time 10 https://api-stage.oglasino.com/health || echo "000"`; if status==503 echoes confirmation; else `::warning::` and continues. | — | No `set -e`; uses `||` to swallow curl failure. **Advisory only — never fails the workflow.** No retry loop. |

`continue-on-error` / non-fatal conditionals: step 3 is intentionally non-fatal (no `exit 1` path).

#### Job: `deploy` (lines 81–156) — `needs: [build, maintenance-on]`

| # | Step name | Action / command | Env / secrets | Notes |
| --- | --- | --- | --- | --- |
| 1 | `SSH deploy to stage droplet` (lines 85–102) | `appleboy/ssh-action@v1.0.3`. Remote script: `set -euo pipefail; cd /opt/oglasino; sed -i "s\|^IMAGE_TAG=.*\|IMAGE_TAG=${IMAGE_TAG}\|" .env; docker compose pull backend; docker compose up -d --wait backend; docker image prune -f`. | `host=${{ secrets.STAGE_HOST }}`, `port=${{ secrets.STAGE_SSH_PORT }}`, `username=deploy`, `key=${{ secrets.STAGE_DEPLOY_SSH_KEY }}`, `envs=IMAGE_TAG`. Step `env: IMAGE_TAG=${{ needs.build.outputs.image_tag }}`. | `docker compose up -d --wait` blocks until the droplet's compose healthcheck reports healthy (or times out). The compose healthcheck is on the droplet, not in this repo. |
| 2 | `Wait for backend to settle` (lines 104–106) | `sleep 60`, gated by `if: success()`. | — | — |
| 3 | `Verify backend is healthy via Caddy` (lines 108–133) | inline retry loop, `if: success()`. Up to 10 attempts, 6s sleep between. Per attempt: `curl -sS -o /dev/null -w "%{http_code}" --max-time 10 https://api-origin-stage.oglasino.com/actuator/health/liveness \|\| echo "000"`. After loop, `if [ "$status" != "200" ]; then echo "::error::..."; exit 1; fi`. | — | **Fails the workflow on non-200**. See Section 3. |
| 4 | `Maintenance OFF — set Cloudflare KV to false (only on deploy success)` (lines 135–156) | `if: success()`. inline `curl -sS -o /tmp/cf_resp.json -w "%{http_code}" -X PUT https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/${MAINTENANCE_KEY}` with `Authorization: Bearer ${CF_API_TOKEN}`, body `false`. Captures HTTP status; `::warning::` on non-200, no `exit 1`. | Env: `CF_API_TOKEN=${{ secrets.CF_API_TOKEN }}`, `CF_ACCOUNT_ID=${{ secrets.CF_ACCOUNT_ID }}`, `CF_KV_NAMESPACE_ID=${{ secrets.CF_KV_NAMESPACE_ID_STAGE }}`. | Auto flip-off is **stage-only** behaviour. A non-2xx KV PUT only warns, leaving stage maintenance ON without failing the job. |

`continue-on-error` / non-fatal conditionals: every step in `deploy` carries `if: success()` (skip on prior failure). Step 4 also intentionally swallows its own non-2xx.

**Deploy step itself:** `appleboy/ssh-action@v1.0.3` to `secrets.STAGE_HOST:secrets.STAGE_SSH_PORT` as user `deploy` with `secrets.STAGE_DEPLOY_SSH_KEY`. The remote command is a docker-compose pull + `up -d --wait` against `/opt/oglasino` on the stage droplet, using the image tag produced by the `build` job.

#### Job: `notify` (lines 158–170) — `needs: deploy`, `if: always()`

| # | Step | Notes |
| --- | --- | --- |
| 1 | `Summary` | Reads `needs.deploy.result`; `::notice::` on success, `::error::` on failure. No deploy mutation. |

---

### 2B. `deploy-backend.yml` (prod)

**File-level config**
- `concurrency: group=deploy-backend, cancel-in-progress=false` (lines 8–10).
- `env.IMAGE_NAME = ghcr.io/${{ github.repository }}` (line 13).
- `env.MAINTENANCE_KEY = maintenance.active` (line 14).

#### Job: `build` (lines 17–48)

| # | Step name | Action / command | Env / secrets | Notes |
| --- | --- | --- | --- | --- |
| 1 | `actions/checkout@v4` | action | — | — |
| 2 | `meta` | inline `run` (lines 27–30) — computes `TAG="$(date -u +%Y%m%dT%H%M)-${GITHUB_SHA::7}"`, exports as `steps.meta.outputs.tag`. (Stage logs the tag too; prod does not.) | — | Job output `image_tag` (line 23). |
| 3 | `docker/setup-buildx-action@v3` | action | — | — |
| 4 | `docker/login-action@v3` (lines 34–38) | action | `secrets.GITHUB_TOKEN`; `username=github.actor` | Logs in to `ghcr.io`. |
| 5 | `docker/build-push-action@v5` (lines 40–48) | action | — | `push: true`. Tags pushed: `ghcr.io/<repo>:<tag>` and `ghcr.io/<repo>:latest`. `cache-from: type=gha`, `cache-to: type=gha,mode=max`. |

`continue-on-error` / non-fatal conditionals: none.

#### Job: `maintenance-on` (lines 50–89) — `needs: build`

| # | Step name | Action / command | Env / secrets | Notes |
| --- | --- | --- | --- | --- |
| 1 | `Set maintenance.active = true (Cloudflare KV)` (lines 54–65) | inline `curl -fsSL -X PUT https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/${MAINTENANCE_KEY}` with `Authorization: Bearer ${CF_API_TOKEN}`, body `true`. | Env: `CF_API_TOKEN=${{ secrets.CF_API_TOKEN }}`, `CF_ACCOUNT_ID=${{ secrets.CF_ACCOUNT_ID }}`, `CF_KV_NAMESPACE_ID=${{ secrets.CF_KV_NAMESPACE_ID }}`. Inherits `MAINTENANCE_KEY` from workflow `env`. | `set -euo pipefail`; `curl -f` so a non-2xx KV PUT fails the step. |
| 2 | `Wait for KV propagation` (lines 67–70) | `sleep 60` | — | Comment cites CF KV eventual consistency across edge POPs and notes 60s is double the worker `cacheTtl`. |
| 3 | `Verify maintenance is live at the edge` (lines 72–89) | inline retry loop. Up to 5 attempts with `sleep 10` between. Per attempt: `curl -s -o /dev/null -w "%{http_code}" --max-time 10 https://api.oglasino.com/health \|\| echo "000"`. On `503` → `exit 0`. After the loop falls through, emits `::warning::Edge returned HTTP $status (expected 503). Proceeding — KV may still be propagating.` | — | **Advisory after retries — never `exit 1`.** Unlike the prod liveness check, this one doesn't fail the workflow when it can't confirm. |

`continue-on-error` / non-fatal conditionals: step 3 is intentionally non-fatal on the post-retry path.

#### Job: `deploy` (lines 91–138) — `needs: [build, maintenance-on]`

| # | Step name | Action / command | Env / secrets | Notes |
| --- | --- | --- | --- | --- |
| 1 | `SSH deploy to backend droplet` (lines 95–116) | `appleboy/ssh-action@v1.0.3`. Remote script: `set -euo pipefail; cd /opt/oglasino; sed -i "s\|^IMAGE_TAG=.*\|IMAGE_TAG=${IMAGE_TAG}\|" .env; docker compose pull backend; docker compose up -d --wait backend; docker image prune -f`. Inline comment (lines 109–112) explains why `--no-deps` is *not* used: backend depends_on redis service_healthy, so without `--no-deps` compose ensures redis is up before starting backend. | `host=${{ secrets.PROD_HOST }}`, `port=${{ secrets.PROD_SSH_PORT }}`, `username=deploy`, `key=${{ secrets.DEPLOY_SSH_KEY }}`, `envs=IMAGE_TAG`. Step `env: IMAGE_TAG=${{ needs.build.outputs.image_tag }}`. | — |
| 2 | `Verify backend is healthy via Caddy` (lines 118–138) | `if: success()`. Inline retry loop. Up to 10 attempts, 6s sleep between. Per attempt: `curl -sS -o /dev/null -w "%{http_code}" --max-time 10 https://api-origin.oglasino.com/actuator/health/liveness \|\| echo "000"`. After loop, `if [ "$status" != "200" ]; then echo "::error::Backend did not become healthy after ${max} attempts. Maintenance stays ON."; exit 1; fi`. | — | **Fails the workflow on non-200.** See Section 3. |

`continue-on-error` / non-fatal conditionals: step 2 has `if: success()`. There is **no** auto maintenance-off step in prod — that is deliberate and called out in the `notify` job (lines 145–149).

**Deploy step itself:** `appleboy/ssh-action@v1.0.3` to `secrets.PROD_HOST:secrets.PROD_SSH_PORT` as user `deploy` with `secrets.DEPLOY_SSH_KEY`. Remote command: docker-compose pull + `up -d --wait` against `/opt/oglasino` on the prod droplet, using the image tag produced by the `build` job.

#### Job: `notify` (lines 140–153) — `needs: deploy`, `if: always()`

| # | Step | Notes |
| --- | --- | --- |
| 1 | `Summary` | On success: `::notice::Backend deployed and healthy. Maintenance is STILL ON.` plus a hint to run `/opt/oglasino/scripts/maintenance-off.sh` on the droplet. On failure: `::error::Deploy FAILED. Maintenance is STILL ON.` plus a rollback hint. The maintenance-off script lives on the droplet, not in this repo. |

---

## Section 3 — The failing backend liveness check

There are two structurally identical liveness checks, one per deploy workflow:

| Where | File | Lines | URL | Method | Headers | Timeout | Retries | Behaviour on failure |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Stage | `.github/workflows/deploy-stage.yml` | 108–133 | `https://api-origin-stage.oglasino.com/actuator/health/liveness` | `GET` (default `curl`) | none | per-attempt `--max-time 10` | up to 10 attempts, 6s sleep between | After the loop, `if [ "$status" != "200" ]; then echo "::error::Backend did not return 200 after ${max} attempts"; exit 1; fi`. **Fails the step → fails the workflow.** Maintenance-OFF step (lines 135–156) is gated by `if: success()` and is therefore skipped. |
| Prod | `.github/workflows/deploy-backend.yml` | 118–138 | `https://api-origin.oglasino.com/actuator/health/liveness` | `GET` (default `curl`) | none | per-attempt `--max-time 10` | up to 10 attempts, 6s sleep between | After the loop, `if [ "$status" != "200" ]; then echo "::error::Backend did not become healthy after ${max} attempts. Maintenance stays ON."; exit 1; fi`. **Fails the step → fails the workflow.** Triggers the `notify` job's failure branch. |

**Why the engineer believes it fails (per the brief):** the stage and prod backends are fronted by Caddy, which is locked down to accept only Cloudflare-fronted traffic and applies bot protection. The `api-origin*.oglasino.com` hostnames the workflow targets bypass the Cloudflare worker (they are origin hostnames), so requests from a GitHub-hosted runner reach Caddy directly; Caddy then rejects them as non-Cloudflare / bot traffic. The workflow file itself does not state this — it carries no comment about origin lockdown — so the conclusion comes from the brief plus the fact that the URLs end in `api-origin*` rather than the public `api.*` host.

**Visible from the workflow file:** the request is unauthenticated (`curl` default), no headers, no Cloudflare auth, no `X-Forwarded-*` shaping. There is nothing the runner adds that would let it pass an "only Cloudflare may call this origin" gate.

**Discrepancy with the brief — flagged in Section 7:** the brief states "The check logs an error but doesn't fail the workflow, producing red in Actions for no reason." Both checks as currently written **do** fail the workflow on non-200 (`exit 1` after the retry loop). They are not "log-only." Mastermind should confirm whether the brief reflects an outdated revision of these files, a mis-description, or a runtime detail (e.g. status sometimes happens to come back 200 despite the lockdown).

---

## Section 4 — Existing Cloudflare-touching steps

Every Cloudflare-touching step uses the same REST endpoint shape: `PUT https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/<key>`. No `wrangler`, no DNS, no Worker code mutation, no purge. All steps are **active** (no commented-out blocks, no conditional skips).

| File | Lines | Step | Action / command | Secrets |
| --- | --- | --- | --- | --- |
| `deploy-stage.yml` | 55–67 | `Set maintenance.active = true (stage KV)` | `curl -fsSL -X PUT .../namespaces/${CF_KV_NAMESPACE_ID}/values/maintenance.active` body `true` | `CF_API_TOKEN`, `CF_ACCOUNT_ID`, `CF_KV_NAMESPACE_ID_STAGE` |
| `deploy-stage.yml` | 135–156 | `Maintenance OFF — set Cloudflare KV to false (only on deploy success)` | `curl -sS -X PUT .../namespaces/${CF_KV_NAMESPACE_ID}/values/maintenance.active` body `false`; non-2xx → `::warning::` only | `CF_API_TOKEN`, `CF_ACCOUNT_ID`, `CF_KV_NAMESPACE_ID_STAGE` |
| `deploy-backend.yml` | 54–65 | `Set maintenance.active = true (Cloudflare KV)` | `curl -fsSL -X PUT .../namespaces/${CF_KV_NAMESPACE_ID}/values/maintenance.active` body `true` | `CF_API_TOKEN`, `CF_ACCOUNT_ID`, `CF_KV_NAMESPACE_ID` |

**Important absence:** no workflow currently writes `admin.bypass.disabled` or `use.backend.check`. The brief's target state — "every deploy flips both flags to `true` before deploying" — is not yet expressed anywhere in `.github/workflows/`. Only `maintenance.active` is touched.

The `https://api*.oglasino.com/health` probes (`deploy-stage.yml:74`, `deploy-backend.yml:81`) read state from the Cloudflare worker but do not mutate Cloudflare; they are listed in Section 2/3 rather than here.

---

## Section 5 — Secrets used by deploy workflows

Pulled from `secrets.*` references across `deploy-stage.yml` and `deploy-backend.yml` only (CI-only `secrets.GITHUB_TOKEN` references in `ci-dev.yml` are excluded; the same `GITHUB_TOKEN` is also consumed by both deploy workflows and is listed below).

| Secret | Workflow / step | Best-guess purpose (from context) |
| --- | --- | --- |
| `GITHUB_TOKEN` | `deploy-stage.yml` (build → docker login, line 39); `deploy-backend.yml` (build → docker login, line 38) | Auto-issued GHA token used to authenticate `docker/login-action@v3` against `ghcr.io` for image push. |
| `CF_API_TOKEN` | `deploy-stage.yml` (maintenance-on line 57, maintenance-off line 138); `deploy-backend.yml` (maintenance-on line 56) | Cloudflare API token with KV write scope. Single token reused for stage and prod KV namespaces. |
| `CF_ACCOUNT_ID` | same three steps as above | Cloudflare account ID — path component in the KV REST URL. |
| `CF_KV_NAMESPACE_ID` | `deploy-backend.yml` (maintenance-on, line 58) | KV namespace ID for the **prod** worker. |
| `CF_KV_NAMESPACE_ID_STAGE` | `deploy-stage.yml` (maintenance-on line 59, maintenance-off line 140) | KV namespace ID for the **stage** worker. Distinct namespace from prod. |
| `STAGE_HOST` | `deploy-stage.yml` (deploy → ssh, line 88) | Stage droplet hostname or IP for SSH. |
| `STAGE_SSH_PORT` | `deploy-stage.yml` (deploy → ssh, line 89) | Stage droplet SSH port (kept as a secret because it is non-default). |
| `STAGE_DEPLOY_SSH_KEY` | `deploy-stage.yml` (deploy → ssh, line 91) | Private SSH key for the stage droplet's `deploy` user. |
| `PROD_HOST` | `deploy-backend.yml` (deploy → ssh, line 98) | Prod droplet hostname or IP for SSH. |
| `PROD_SSH_PORT` | `deploy-backend.yml` (deploy → ssh, line 99) | Prod droplet SSH port. |
| `DEPLOY_SSH_KEY` | `deploy-backend.yml` (deploy → ssh, line 101) | Private SSH key for the prod droplet's `deploy` user. Naming asymmetry with stage — see Section 6. |

---

## Section 6 — Adjacent observations

Per conventions Part 4b. None fixed; brief is read-only. Severity guess in parens.

1. **`sleep 0` named "Wait for KV propagation"** — `deploy-stage.yml:69–70`. The step does nothing; the comment promises a wait. Either the wait is intentionally absent on stage (in which case the step name lies) or the value should be non-zero. **(low)** — Out of scope.

2. **Maintenance-off step on stage swallows non-2xx KV writes** — `deploy-stage.yml:135–156`. If the KV PUT to flip stage maintenance back to `false` returns non-2xx, the step emits a warning and exits 0. Stage will silently stay in maintenance until someone notices. **(medium)** — Out of scope.

3. **Stage edge verification has no retry loop** — `deploy-stage.yml:72–79`. Single curl, no retries, no `set -e`. Prod's equivalent (lines 72–89) retries 5×. The two are inconsistent. **(low)** — Out of scope.

4. **Prod edge-verification advisory wording** — `deploy-backend.yml:89`. The fall-through warning says "Proceeding — KV may still be propagating." But the next job (`deploy`) only `needs: [build, maintenance-on]`, so the deploy will run regardless of whether the warning fires. The warning's "Proceeding" wording is accurate but obscures that there is no gate at all on this verification. **(low)** — Out of scope.

5. **Inconsistent secret naming across stage / prod SSH** — stage uses `STAGE_DEPLOY_SSH_KEY`; prod uses `DEPLOY_SSH_KEY` (no `PROD_` prefix). Stage uses `STAGE_HOST` / `STAGE_SSH_PORT`; prod uses `PROD_HOST` / `PROD_SSH_PORT`. The prod SSH key is the only secret in the deploy pair that lacks an environment prefix. **(low)** — Out of scope; flag for the secret-inventory doc.

6. **`MAINTENANCE_KEY` defined as workflow `env` but never overridden** — both deploy workflows. It exists as `env.MAINTENANCE_KEY = maintenance.active` (lines 14 in each). Centralising the key name is fine; just noting it was clearly anticipated to be templated, and the upcoming `admin.bypass.disabled` work would benefit from following the same pattern. **(low)** — Out of scope.

7. **Stage deploy `Wait for backend to settle` (`sleep 60`) sits *before* the health probe** — `deploy-stage.yml:104–106`. Prod has no such fixed pre-probe sleep — it goes straight from the SSH step into the retry loop. The 60s pre-sleep on stage extends every stage deploy by a minute even when the container is already healthy from `docker compose up -d --wait`. **(low)** — Out of scope.

8. **`docker image prune -f` on the droplet inside the SSH script** — both deploy workflows. Hard prune with no filter; relies on the just-pulled tag being referenced by the running container so it survives. Correct in practice, but worth knowing if rollback ever depends on a previous image still being on disk locally. **(low)** — Out of scope.

9. **Help-text in `ci-dev.yml` recommends `git commit --amend && git push --force-with-lease`** — `ci-dev.yml:78–80`. This is contributor guidance, not a workflow action, but it conflicts with the engineer hard rule against amending and force-pushing. Cosmetic for CI; flag in case you want it reworded. **(low)** — Out of scope.

---

## Section 7 — Open questions for Mastermind

1. **Brief vs reality on the liveness check.** The brief says the post-deploy backend liveness check "logs an error but doesn't fail the workflow, producing red in Actions for no reason." Both `deploy-stage.yml:130–133` and `deploy-backend.yml:135–138` end the retry loop with `exit 1` on non-200, which **does** fail the step and the workflow. Three possibilities I cannot distinguish from the file alone:
   - The brief is describing an older revision and the `exit 1` was added since.
   - "Red in Actions for no reason" refers to the inline `::error::` annotations being visible on the run summary page (which would be cosmetic if the job somehow still passed) — but the `exit 1` rules that out.
   - The runner sometimes happens to receive a 200 (e.g. a fraction of probes get through Caddy's filter), so the workflow goes green often enough that the description undercounts the failure mode.
   Which is it? The fix brief depends on whether we're removing a working failure gate or replacing a noisy advisory.

2. **Should the liveness check be replaced or removed?** Two options the brief does not pre-decide:
   - Replace with a probe through the public `api.*.oglasino.com` host (going through Cloudflare → worker → backend), accepting that this only verifies the whole edge path and not Caddy specifically.
   - Remove it entirely and rely on `docker compose up -d --wait`'s readiness gate on the droplet.
   Confirm preference before the fix brief is drafted.

3. **`admin.bypass.disabled` semantics during deploy.** The brief says both `maintenance.active` and `admin.bypass.disabled` should flip to `"true"` pre-deploy. For `maintenance.active`, "true" means the worker serves the maintenance response. For `admin.bypass.disabled`, the intended polarity at deploy time should be confirmed: is `"true"` the value that *blocks* admin bypass during the maintenance window (so even logged-in admins see maintenance)? The fix brief should pin this down before the curl command is written.

4. **Symmetry between stage and prod for the new flips.** The brief's wording is "every deploy" — stage and prod both. Stage currently auto-flips `maintenance.active` back to `false` after a successful deploy (lines 135–156). Should it also auto-flip `admin.bypass.disabled` back, or leave it as a manual cleanup like prod? The fix brief should state explicitly per workflow.

5. **Single-job vs split-jobs for the two flag flips.** Should both KV PUTs land in the existing `maintenance-on` job (still one wait, still one verify), or be split into a new `pre-deploy-flags` job? The current `maintenance-on` job's verify step only checks `maintenance.active` via `/health`; verifying `admin.bypass.disabled` from a runner is not obviously possible without an admin token. Confirm whether the verify step is expected to grow, stay maintenance-only, or be dropped.

6. **Worker `cacheTtl=30s` doubled for safety = `sleep 60` on prod.** The brief restates this. Is the same `sleep 60` desired on stage (currently `sleep 0`)? If yes, the fix touches stage's wait step too.
