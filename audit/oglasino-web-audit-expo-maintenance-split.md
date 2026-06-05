# Audit — Web's role in the maintenance KV split

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-29
**Type:** read-only audit (no code changes)
**Brief:** Inventory web's role in the `maintenance.active` → `maintenance.web.active` Cloudflare KV rename. Report deploy-workflow KV flips, every maintenance/admin.bypass reference in the repo, and confirm web does not read maintenance state from the worker/KV at runtime.

---

## TL;DR

- Web flips the Cloudflare KV maintenance flag from **two** GitHub Actions workflows, not one: `deploy-stage.yml` (push to `stage`) and `deploy-prod.yml` (push to `main`). The brief's "we believe web deploys from `stage`" is true for stage but incomplete — prod deploys from `main` and flips the same key name.
- Each workflow writes **two** KV keys on every deploy: `maintenance.active` and `admin.bypass.disabled`, both set to the string `"true"`. The key strings live in `env:` blocks (`MAINTENANCE_KEY`, `ADMIN_BYPASS_KEY`).
- The flip is **one-way**: the workflows only ever set the keys to `true`. Turning maintenance **off** is a manual operator step after the deploy — the workflows print a reminder, they never PUT `false`.
- The single point the rename touches in web is the `MAINTENANCE_KEY:` env value in **both** workflows (`deploy-stage.yml:14`, `deploy-prod.yml:13`). `ADMIN_BYPASS_KEY` is unaffected by the brief's rename.
- **Web does not read maintenance state from the worker/KV at runtime.** Confirmed — no maintenance route, no `X-Oglasino-Maintenance` / `503` handling in app code. The only in-app maintenance logic is the admin `MaintenanceToggle`, which hits the **backend** `/public/maintenance/*` endpoints — a *separate* maintenance concept from the edge KV flag (see §3, important).

---

## 1. Deploy workflow KV flips

Two workflows write to Cloudflare KV. Both share an identical flip step shape.

### Triggering branches

| Workflow | File | Trigger | KV namespace secret |
| --- | --- | --- | --- |
| Deploy - Stage | `.github/workflows/deploy-stage.yml` | `push` to `stage` (`:4-5`) | `CF_KV_NAMESPACE_ID_STAGE` (`:44`) |
| Deploy - Production | `.github/workflows/deploy-prod.yml` | `push` to `main` (`:4-5`) | `CF_KV_NAMESPACE_ID` (`:42`) |

(A third workflow, `.github/workflows/ci-dev.yml`, exists but was not in the maintenance/KV grep — it does no KV writes.)

### Key strings (quoted exactly)

Stage (`deploy-stage.yml:13-18`):
```yaml
  STAGE_URL: https://stage.oglasino.com
  MAINTENANCE_KEY: maintenance.active
  ADMIN_BYPASS_KEY: admin.bypass.disabled
  MAINTENANCE_HEADER: X-Oglasino-Maintenance
  # Worker reads KV with cacheTtl=60, so 60s covers the per-edge cache.
  KV_PROPAGATION_WAIT_SECONDS: '60'
```

Prod (`deploy-prod.yml:13-16`):
```yaml
  MAINTENANCE_KEY: maintenance.active
  ADMIN_BYPASS_KEY: admin.bypass.disabled
  # Worker reads KV with cacheTtl=60, so 60s covers the per-edge cache.
  KV_PROPAGATION_WAIT_SECONDS: '60'
```

So the exact KV key strings written are:
- `maintenance.active`  ← the one being renamed to `maintenance.web.active`
- `admin.bypass.disabled`  ← **not** in scope of the brief's rename

### Steps that write KV

Both workflows have a single step `Maintenance ON — flip Cloudflare KV` (`deploy-stage.yml:52-82`, `deploy-prod.yml:50-80`) that issues **two** PUTs — one per key — then sleeps.

The curl/API call shape (quoted from `deploy-stage.yml:56-61`; prod `:54-59` is identical apart from the namespace secret):
```bash
response=$(curl -sS -o /tmp/cf_resp.json -w "%{http_code}" \
  -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_KV_NAMESPACE_ID}/values/${MAINTENANCE_KEY}" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: text/plain" \
  --data "true")
```
The second PUT (`deploy-stage.yml:69-74`, `deploy-prod.yml:67-72`) is the same call with `${ADMIN_BYPASS_KEY}` in place of `${MAINTENANCE_KEY}`. Both write the literal body `true`. Each PUT fails the job (`exit 1`) if the HTTP code is not `200`.

The value written is always the string `"true"`. **Neither workflow ever writes `false`.**

### Wait / sleep timing

After both PUTs succeed (`deploy-stage.yml:81-82`, `deploy-prod.yml:79-80`):
```bash
echo "KV writes OK. Sleeping ${KV_PROPAGATION_WAIT_SECONDS}s ..."
sleep "${KV_PROPAGATION_WAIT_SECONDS}"
```
`KV_PROPAGATION_WAIT_SECONDS = '60'` in both, with the comment "Worker reads KV with cacheTtl=60, so 60s covers the per-edge cache."

### Verify-live step (differs between the two)

- **Stage** (`deploy-stage.yml:84-106`, advisory, `max=1`): probes `GET https://stage.oglasino.com/` with `curl -I`, parses the `X-Oglasino-Maintenance:` response header, expects lowercased value `true`. Does not fail the deploy.
- **Prod** (`deploy-prod.yml:82-99`, `max=5`, `sleep 10` between attempts): probes `GET https://api.oglasino.com/health` and checks for HTTP `503` (the worker returns 503 when `maintenance.active=true`). Comment at `:83-84` notes header parsing was unreliable on GH egress, so prod uses status code instead. On non-503 after 5 attempts it emits a `::warning::` and proceeds (does not fail).

### Flip-OFF is manual (not in any workflow)

Both workflows end with notify steps that instruct the operator to flip the keys back by hand:
- `deploy-stage.yml:120-128` / `deploy-prod.yml:113-121`:
  - success: `"... Maintenance is still ON — verify the site, then manually flip both ${MAINTENANCE_KEY} and ${ADMIN_BYPASS_KEY} to false in Cloudflare KV."`
  - failure: `"... Maintenance is still ON. Investigate, then manually flip both ... to false ... when ready."`

There is no automated maintenance-OFF anywhere in the repo. If the rename lands, the **manual** off-flip the operator performs must also target the new key name — that's a runbook/operator concern, not a code change here, but worth flagging to whoever owns the rename.

---

## 2. Every maintenance / admin.bypass reference in the repo

Grep across `*.ts/tsx/js/yml/yaml/json/sh` (node_modules and `.next` excluded).

### Cloudflare KV key strings (`maintenance.active`, `admin.bypass.disabled`)

| File:line | Reference | Notes |
| --- | --- | --- |
| `.github/workflows/deploy-stage.yml:14` | `MAINTENANCE_KEY: maintenance.active` | **rename target** |
| `.github/workflows/deploy-stage.yml:15` | `ADMIN_BYPASS_KEY: admin.bypass.disabled` | not renamed |
| `.github/workflows/deploy-stage.yml:16` | `MAINTENANCE_HEADER: X-Oglasino-Maintenance` | header name, used only by the stage verify probe |
| `.github/workflows/deploy-prod.yml:13` | `MAINTENANCE_KEY: maintenance.active` | **rename target** |
| `.github/workflows/deploy-prod.yml:14` | `ADMIN_BYPASS_KEY: admin.bypass.disabled` | not renamed |
| (both workflows, many lines) | `${MAINTENANCE_KEY}` / `${ADMIN_BYPASS_KEY}` interpolations | All resolve through the two env vars above — see §1. No raw literal `maintenance.active` outside the `env:` definitions; everything else is the `${MAINTENANCE_KEY}` variable. Same for admin bypass. So the rename is a **single-line edit per workflow** (the `env:` value), not a multi-site find/replace. |

`admin.bypass` also appears in docs prose (not KV writes, no code):
- `docs/07-pre-lunch-tasks.md:110` — `# 5. Verify admin bypass works`
- `docs/07-pre-lunch-tasks.md:262` — manual smoke note about hitting `/admin` while the gate is up.

### The word "maintenance" elsewhere in the repo (non-KV)

| File:line | What it is | Relevant to the split? |
| --- | --- | --- |
| `src/lib/admin/lib/service/configService.ts:48-62` | `isMaintenanceActive()` → `BACKEND_API.get('/public/maintenance/active')` | **No — backend endpoint, not KV.** See §3. |
| `src/lib/admin/lib/service/configService.ts:64-78` | `toggleMaintenance()` → `BACKEND_API.post('/public/maintenance/toggle')` | **No — backend endpoint, not KV.** See §3. |
| `src/components/client/buttons/MaintenanceToggle.tsx` (whole file) | Admin wrench toggle calling the two functions above | No — drives the backend maintenance flag, not the edge KV. |
| `app/[locale]/admin/config/page.tsx:3,24` | Imports & renders `<MaintenanceToggle />` on the admin config page | No — same backend flag. |
| `src/components/icons/dynamic/RepairsMaintenanceCategoryIcon.tsx` + `index.ts:103` | A product-category icon ("Repairs & Maintenance") | No — unrelated, name collision only. |

---

## 3. Does web read maintenance state from the worker/KV at runtime? — No (with an important nuance)

**Confirmed: web has no runtime dependency on the edge maintenance KV key or the worker's maintenance response.**

- No app route or page named/handling maintenance (`find app -iname '*maintenance*'` → none).
- No code reads the `X-Oglasino-Maintenance` header — it appears only in `deploy-stage.yml` (the CI verify probe). No app code parses it.
- No app code keys behavior off a `503` from the edge. (The only `503` in app code is `app/api/revalidate/route.ts:54`, an unrelated `{ error: 'not_configured' }` guard; remaining `503` grep hits are SVG `d="..."` path data.)
- This matches conventions Part 8: "The Cloudflare router worker is the edge boundary. Maintenance state, admin bypass, and origin forwarding live there." Maintenance is enforced entirely at the edge, in front of the Vercel origin; the Next app never sees a maintenance-mode request.

**Important nuance — there are two distinct "maintenance" concepts, and web touches both via different mechanisms:**

1. **Edge maintenance (Cloudflare KV `maintenance.active`)** — the flag this rename is about. Web's *only* interaction is the deploy workflows writing it (§1). Nothing in the running app reads it.

2. **Backend maintenance (`/public/maintenance/active` + `/public/maintenance/toggle`)** — a separate server-side flag, surfaced in the admin UI by `MaintenanceToggle` / `configService`. This is *not* the Cloudflare KV key and is *not* affected by the `maintenance.active` → `maintenance.web.active` KV rename. An admin clicking the wrench toggles the **backend** flag; it does **not** flip the edge KV. If the project intends the admin toggle to control the edge maintenance gate, that wiring does not exist in web today — the two are independent. Flagging so the rename isn't assumed to cover the admin toggle.

---

## Notes for whoever owns the rename (factual, from the code)

- The rename touches exactly two lines in this repo: `deploy-stage.yml:14` and `deploy-prod.yml:13` (`MAINTENANCE_KEY: maintenance.active` → `maintenance.web.active`). Because both workflows reference the key only through `${MAINTENANCE_KEY}`, no other line changes.
- The brief's framing ("the web deploy workflow") is singular; there are two (stage + prod). Both must change together or stage and prod will diverge on the key name.
- The manual OFF-flip described in the notify steps (and any external runbook) must switch to the new key name too — there is no automated OFF path to update.
- `admin.bypass.disabled` is written in lockstep with the maintenance key on every deploy but is out of scope for this rename; leave it.
- The worker owns the contract `cacheTtl=60` ↔ `KV_PROPAGATION_WAIT_SECONDS=60` and the `maintenance.active=true → 503` behavior the prod verify step relies on. If the router repo renames the key, the prod verify step's 503 probe keeps working only if the worker reads the new key under the same 503 semantics — a router-repo concern, noted here because the web prod workflow depends on it.
