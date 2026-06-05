# Session summary — `oglasino-router` worker maintenance-split audit

**Repo:** oglasino-router
**Branch:** stage
**Date:** 2026-05-15
**Task:** Read-only audit of the worker maintenance-split change — produce the exact strings, semantics, and code locations the propagation briefs will reference.

---

## Section 1 — KV keys the worker reads

All KV access is via the single binding `CONFIG` (`[[kv_namespaces]] binding = "CONFIG"` in `wrangler.toml`; stage id `0b9a66b980a14bcfb5587b7480ba1a2b`, production id `dafb45ca859f470682a34245e253fad9`). Three keys are read; no others.

### 1.1 — `maintenance.active`

- **Exact key string:** `maintenance.active`
- **Binding:** `CONFIG`
- **Code path:** `src/index.ts:78` (inside `Promise.all` at lines 77–80)
- **Read options:** `{ cacheTtl: 30 }` (30 second edge cache)
- **Expected value type:** string. Only the literal string `"true"` is treated as truthy; everything else is treated as `false`.
- **Behavior on each input:**
  - Value `"true"` → `maintenanceActive = true` (`src/index.ts:81`).
  - Value `"false"` → `maintenanceActive = false`.
  - Any other string (`"True"`, `"1"`, `"yes"`, empty string, whitespace, JSON, etc.) → treated as `false` (strict `=== "true"` comparison at line 81).
  - Absent (`null` returned by `CONFIG.get`) → treated as `false`.
  - The `Promise.all` throws (KV unreachable for either key) → both flags forced to `false` by the `catch` block at `src/index.ts:83–87`. Fail-open.

### 1.2 — `admin.bypass.disabled`

- **Exact key string:** `admin.bypass.disabled`
- **Binding:** `CONFIG`
- **Code path:** `src/index.ts:79` (inside the same `Promise.all` as 1.1)
- **Read options:** `{ cacheTtl: 30 }`
- **Expected value type:** string. Only the literal string `"true"` is treated as truthy.
- **Behavior on each input:**
  - Value `"true"` → `adminBypassDisabled = true` (`src/index.ts:82`).
  - Value `"false"` → `adminBypassDisabled = false`.
  - Any other string → treated as `false` (strict `=== "true"` comparison at line 82).
  - Absent (`null`) → treated as `false`.
  - `Promise.all` throws → forced to `false` by the same `catch` block (`src/index.ts:83–87`). Fail-open.

### 1.3 — `use.backend.check`

- **Exact key string:** `use.backend.check`
- **Binding:** `CONFIG`
- **Code path:** `src/index.ts:91–93`. This read happens **only if** `maintenanceActive` is still `false` after the read above (`src/index.ts:90` guard `if (!maintenanceActive)`).
- **Read options:** `{ cacheTtl: 30 }`
- **Expected value type:** string. Only `"true"` is treated as truthy.
- **Behavior on each input:**
  - Value `"true"` → run the backend liveness probe (`src/index.ts:94`).
  - Any other value, or absent → skip the probe.
  - This read is **not** inside the fail-open `try`/`catch` from 1.1/1.2. If `CONFIG.get("use.backend.check")` itself throws, the exception propagates and the request fails. (See Section 8, finding 8.4.)

If the probe runs and either returns non-ok or throws, `maintenanceActive` is forced to `true` (`src/index.ts:99` and `:101`). This is the only place in the worker that flips `maintenanceActive` after the KV read.

No other KV keys are read. There is no `portal.maintenance.active`, no `api.maintenance.active`, no `web.maintenance.active`. The split is `maintenance.active` × `admin.bypass.disabled` — not a per-surface key split.

---

## Section 2 — Gating logic per path

### Decision order (top to bottom, first match wins)

1. **`www` → apex 301 redirect** — `src/index.ts:46–51`. Fires when `env.WWW_HOST` is non-empty and `host === env.WWW_HOST`. Production: `www.oglasino.com` → `https://oglasino.com${path}${search}` 301. Stage: `WWW_HOST = ""`, so this branch never fires (`www.stage.oglasino.com` falls through to the unknown-host 404).
2. **Unknown host 404** — `src/index.ts:56–58`. If the host is neither `APEX_HOST` nor `API_HOST`, return `new Response("Not found", { status: 404 })`. No body shape beyond the literal `"Not found"`, no special headers.
3. **Admin-request classification** — `src/index.ts:62–71`. Sets a boolean `isAdminRequest` (does not branch yet); the value feeds the maintenance gate below. A request is `isAdminRequest === true` if any of:
   - Path matches `/^\/[a-z]{2}-[a-z]{2}\/admin(\/|$)/i` (case-insensitive locale-prefixed admin route — `/rs-sr/admin`, `/en-us/admin/users`, etc.).
   - `isApi` is true (request hit `API_HOST`).
   - Path starts with `/api/` (Next.js API routes on the apex host).
   - Path starts with `/_next/` (Next.js static assets and chunks).
   - Path is exactly `/favicon.ico`.
4. **KV read + (conditional) backend probe** — `src/index.ts:74–104`. See Section 1.
5. **Maintenance gate** — `src/index.ts:110–115`. Evaluated only if `maintenanceActive === true`:
   - `shouldBlock = adminBypassDisabled || !isAdminRequest` (`src/index.ts:111`).
   - If `shouldBlock`, return `maintenanceResponse(isApi, path, url.search, env)` (`src/index.ts:113`).
   - Otherwise fall through to the forward.
6. **Forward to upstream** — `src/index.ts:119–134`. If `isApi`, forward to `BACKEND_ORIGIN`; otherwise (apex host) forward to `FRONTEND_ORIGIN`. `addNoIndex` is set to `env.ENVIRONMENT === "stage"` and is applied inside `forwardToOrigin` to overwrite `X-Robots-Tag` regardless of upstream.

### Per-path summary (matrix axes: maintenance × bypass)

| Path / host                       | maintenance=off          | maintenance=on, bypass=off (admin bypass enabled) | maintenance=on, bypass=on (full lockdown) |
|-----------------------------------|--------------------------|---------------------------------------------------|-------------------------------------------|
| `api.oglasino.com/*` (any path)   | Forward to `BACKEND_ORIGIN` | Forward to `BACKEND_ORIGIN` (API counts as admin) | 503 JSON (see Closed response, API)       |
| `oglasino.com/{xx-xx}/admin[/...]` | Forward to `FRONTEND_ORIGIN` | Forward to `FRONTEND_ORIGIN`                     | 503 HTML from `MAINTENANCE_ORIGIN`        |
| `oglasino.com/api/*`              | Forward to `FRONTEND_ORIGIN` | Forward to `FRONTEND_ORIGIN` (counts as admin)   | 503 HTML from `MAINTENANCE_ORIGIN`        |
| `oglasino.com/_next/*`            | Forward to `FRONTEND_ORIGIN` | Forward to `FRONTEND_ORIGIN` (counts as admin)   | 503 HTML from `MAINTENANCE_ORIGIN`        |
| `oglasino.com/favicon.ico`        | Forward to `FRONTEND_ORIGIN` | Forward to `FRONTEND_ORIGIN` (counts as admin)   | 503 HTML from `MAINTENANCE_ORIGIN`        |
| `oglasino.com/*` (everything else, apex) | Forward to `FRONTEND_ORIGIN` | 503 HTML from `MAINTENANCE_ORIGIN`           | 503 HTML from `MAINTENANCE_ORIGIN`        |
| `www.oglasino.com/*` (prod only)  | 301 → apex               | 301 → apex (redirect is evaluated before maintenance) | 301 → apex                            |
| Unknown host                      | 404 `"Not found"`        | 404 (host check is evaluated before maintenance)  | 404                                       |

### Closed response, API path

`maintenanceResponse(isApi=true, …)` → `src/index.ts:144–153`.

- Status: `503`
- Body: the constant `MAINTENANCE_JSON` (`src/index.ts:26–31`) — `{"status":"maintenance","message":"Oglasino is undergoing maintenance. Please try again in a few minutes.","retryAfter":120}`
- Headers:
  - `Content-Type: application/json; charset=utf-8`
  - `Retry-After: 120`
  - `Cache-Control: no-store`
  - `X-Oglasino-Maintenance: true`

### Closed response, frontend path

`maintenanceResponse(isApi=false, …)` → `src/index.ts:156–166`.

- The worker fetches `${env.MAINTENANCE_ORIGIN}${path}${search}` (production and stage: `https://oglasino-maintenance.pages.dev`).
- Status: `503` (overrides whatever the maintenance page returned).
- `statusText: "Service Unavailable"`.
- Headers: upstream headers, plus the worker forces `X-Oglasino-Maintenance: true`, `Cache-Control: no-store`, `Retry-After: 120`.
- Body: streamed directly from the maintenance origin.

### Open response (forwarded path)

`forwardToOrigin(originBase, path, search, originalRequest, addNoIndex)` → `src/index.ts:169–206`.

- Target URL: `${originBase}${path}${search}` (no rewriting of path or query).
- Method, headers, body: preserved from `originalRequest`.
- Worker sets `X-Forwarded-Host` (from incoming `Host` header) and `X-Forwarded-Proto: https` if a Host header is present.
- Redirects: `redirect: "manual"` — upstream 3xx responses are passed through unchanged.
- On stage (`addNoIndex === true`): worker reconstructs the response and overwrites `X-Robots-Tag` with the constant `NOINDEX_HEADER` (`"noindex, nofollow, noarchive, nosnippet"`, `src/index.ts:33`). This applies to **both** the apex and the API host on stage.

### Default branch

There is no fall-through default outside the path/host classification. Once host is established as `APEX_HOST` or `API_HOST`, every request takes the maintenance-gate or forward path. The `isAdminRequest` predicate is closed — anything not matched by its five clauses is non-admin.

---

## Section 3 — Default behavior when a KV key is absent

The worker treats `CONFIG.get(...) === null` identically to `"false"` for all three keys (strict `=== "true"` comparison, line 81 / 82 / 94).

| Key                       | Absent → flag value | Effective behavior of the surface                                                                 | Up / down |
|---------------------------|---------------------|----------------------------------------------------------------------------------------------------|-----------|
| `maintenance.active`      | `false`             | Maintenance gate not engaged — both admin and non-admin traffic forward to upstreams.              | **Up**    |
| `admin.bypass.disabled`   | `false`             | If maintenance is engaged, admin requests bypass; non-admin requests are blocked. If maintenance is not engaged, this flag has no observable effect. | Bypass enabled (admin **up**, non-admin governed by `maintenance.active`) |
| `use.backend.check`       | `false`             | No probe. The backend's liveness has no effect on routing.                                         | Probe **off**     |

**Net default with an empty KV namespace:** everything is up, the admin bypass is enabled (but inert because maintenance is off), and the backend probe is disabled. This is the "happy path" for a fresh environment.

**Important for the propagation plan:** the worker's default-when-absent is **up** for the maintenance surface. Any backend/web reader of the new key(s) must agree: `null` from KV means "the surface is up." If backend or web defaults `null` to "down," users would see a maintenance UI while the worker is happily proxying traffic — the half-state Igor's brief is trying to close.

The two flags read different defaults from the same "absent" state, but they are not conflicting: the worker's logic only consults `adminBypassDisabled` after `maintenanceActive === true`, so the two flags compose into a single 3-row matrix (off / admin-on / full-lockdown), not a 4-row one.

---

## Section 4 — KV writes

**The worker performs no KV writes.** Grep for write methods (`put`, `delete`, `list`, `getWithMetadata`) on the `CONFIG` binding returns zero matches in `src/index.ts`. Only three `CONFIG.get(...)` reads exist (lines 78, 79, 91).

The worker is strictly a reader of `CONFIG`. Whatever writes `maintenance.active`, `admin.bypass.disabled`, and `use.backend.check` lives outside this repo (backend and/or web admin tooling).

---

## Section 5 — Other worker surface changes

The brief asks whether the maintenance-split commit (`f1ee682`) changed anything else. Comparing `f1ee682` against its parent `38ff063`:

- **Routes (`wrangler.toml`):** No change. Routes are configured via the Cloudflare Dashboard (see the comment at the top of `wrangler.toml`, lines 5–9); the file does not declare them. No new `[[routes]]` blocks.
- **Env vars (`env.*`):** No change. The `Env` interface (`src/index.ts:15–24`) lists the same eight bindings before and after: `CONFIG`, `FRONTEND_ORIGIN`, `BACKEND_ORIGIN`, `MAINTENANCE_ORIGIN`, `APEX_HOST`, `WWW_HOST`, `API_HOST`, `ENVIRONMENT`.
- **KV namespace bindings, secret bindings, service bindings, R2 bindings, Durable Object bindings:** No change. Only the single `CONFIG` KV namespace binding exists — same as before the commit.
- **Maintenance response shape:**
  - API path: unchanged — same `MAINTENANCE_JSON` body, same 503, same four headers.
  - Frontend path: unchanged — still fetches `MAINTENANCE_ORIGIN`, still overrides to 503, still sets the same three forced headers (`X-Oglasino-Maintenance`, `Cache-Control`, `Retry-After`).
- **Request headers set, read, or forwarded:** No change. `X-Forwarded-Host`, `X-Forwarded-Proto`, `X-Robots-Tag` (stage only), and the response-side `X-Oglasino-Maintenance` are all unchanged.
- **`package.json` dependencies:** No change. The commit modified only `src/index.ts` (per `git show --stat f1ee682`: 1 file, +72 / −19). No `package.json` or `package-lock.json` touched.

**The only surface change introduced by `f1ee682` is the new `admin.bypass.disabled` KV key and the two-flag block-decision (`shouldBlock = adminBypassDisabled || !isAdminRequest`).** Everything else — the matrix-comment block at the top of the file, the headers, the upstream wiring, the `redirect: "manual"`, the stage noindex — is unchanged from the pre-split version (or restructured but semantically equivalent).

The `Env` interface comment for `WWW_HOST` is `"// empty string when env has no www variant (e.g., stage)"`. That is unchanged.

---

## Section 6 — Tests

- **Test runner:** Vitest (`vitest` 1.6.1; `npm test` runs `vitest run`).
- **Test files:**
  - `tests/router.test.ts` — single file, covers host routing, the full 3-row matrix, the backend liveness probe, KV error handling, the stage noindex header, and forwarding semantics (X-Forwarded-* headers, method preservation, query-string preservation). 32 tests total, organized into 7 `describe` blocks.

**Test run result (this session):**

```
$ npm test
> vitest run
 ✓ tests/router.test.ts  (32 tests) 10ms
 Test Files  1 passed (1)
      Tests  32 passed (32)
   Duration  298ms
```

Zero failures. The test suite explicitly asserts the new two-flag matrix:

- "matrix: maintenance off — allow everyone" (3 tests)
- "matrix: maintenance on + bypass enabled — allow admin + API" (7 tests; covers `/xx-xx/admin/...`, `/xx-xx/admin` root, `/api/...` on apex, `/_next/`, `/favicon.ico`, API host, and a negative case where non-admin gets 503)
- "matrix: maintenance on + bypass disabled — full lockdown" (5 tests; admin API, admin frontend, non-admin frontend, `/_next/` — all blocked)

Plus matching coverage for the KV-error fail-open path (`KV throws on maintenance.active`, `KV throws on admin.bypass.disabled` — both go to normal routing) and for the backend liveness probe.

`tests/router.test.ts` is currently **uncommitted** — `git status` shows it as modified. The modification expanded the suite from the pre-split version (the diff is the full restructure into matrix-rowed `describe` blocks plus new assertions for `admin.bypass.disabled`). The tests against the deployed worker source pass cleanly. See Section 7 for the working-tree-vs-deployed observation.

`npm run lint` (`tsc --noEmit`) also passes — clean.

---

## Section 7 — Deployed state vs. checked-out state

**Current branch:** `stage`. **HEAD:** `f1ee6820a0d4a630550d3aa3db67f8de9c4c5dd0` — `Admin allowed to access admin portal after admin.bypass.disabled is set to false` (2026-05-15 13:48:42 +0200 = 11:48:42 UTC). This commit is the one Igor's brief refers to.

**Working-tree state (uncommitted):**

- `M tests/router.test.ts` — full restructure into matrix-rowed describe blocks (the tests in Section 6). Does not affect the deployed worker code; the tests are not bundled. Should be committed alongside the change to keep the test suite synchronized with the live behavior.
- `M .gitignore` — adds `.agent` to ignore. Unrelated to the maintenance split.
- `?? CLAUDE.md` — new untracked file (the engineer-agent instructions). Unrelated to the maintenance split.

`src/index.ts` is **clean** (no uncommitted modifications). `wrangler.toml`, `package.json`, `tsconfig.json` are also clean. So the worker code and config match `HEAD`.

**Deployed worker timestamps (`wrangler deployments list`):**

| Env / worker name             | Latest deployment (UTC)         | Version id                              |
|-------------------------------|---------------------------------|-----------------------------------------|
| production / `oglasino-router-prod`  | `2026-05-15T11:39:29.376Z`      | `de29d03b-8b3c-440f-857f-3e85c40b2bb0`  |
| stage / `oglasino-router-stage`      | `2026-05-09T23:13:55.146Z`      | `3b7fd687-1542-451b-9026-2e0c32a6829f`  |

**Drift findings.** Two are real.

1. **Stage has not received the maintenance-split change.** The latest stage deployment is dated `2026-05-09T23:13:55Z`, six days before commit `f1ee682` (`2026-05-15T11:48Z`). The stage worker is running the pre-split code — single-flag maintenance, no `admin.bypass.disabled` read. Igor's brief states the change is "deployed to production and stage"; observable wrangler state contradicts that for stage. **Flagged as a high-severity finding in Section 8 (8.1).** This is the one place the audit could describe behavior other than the live system: anything the audit says about `admin.bypass.disabled` on stage is true of `HEAD` but not of the running worker.

2. **Production's latest deployment timestamp predates the commit by ~9 minutes.** Production deployment `de29d03b` was created at `2026-05-15T11:39:29Z`; commit `f1ee682` is timestamped `2026-05-15T11:48:42Z`. The same day shows four production deploys (11:20, 11:26, 11:38, 11:39 UTC) — Igor was iterating live. The discrepancy is consistent with deploying from an uncommitted working tree and committing after the final deploy, which would mean the deployed bytes match the eventual commit content despite the timestamp ordering. Wrangler's `deployments list` does not expose source SHA, so this cannot be verified from the CLI. **Flagged as a medium-severity finding in Section 8 (8.2)** — worth confirming, not a defect by itself.

3. **`tests/router.test.ts` uncommitted.** The test-suite restructure is on disk but not in git. If anyone else clones the repo and runs `npm test`, they get the pre-split tests (which still pass against the post-split worker because the pre-split tests are a subset, but they do not assert the matrix as the brief expects). Low-severity, easy fix — Igor commits the file.

---

## Section 8 — Findings that need fixing

### 8.1 — Stage is running the pre-split worker

- **Where:** Cloudflare worker `oglasino-router-stage` (id `0b9a66b980a14bcfb5587b7480ba1a2b`-bound)
- **Severity:** high
- **Description:** Latest stage deploy is 2026-05-09, six days before the split-flag commit. Stage is running the single-flag worker; `admin.bypass.disabled` does nothing on stage. Anyone testing the new behavior on stage would conclude the split is broken when in fact the deploy never happened.
- **Scope:** out of scope for code-side propagation (backend + web). The fix is a deploy, not a code change. Routes to Mastermind for an explicit "deploy stage" step in the propagation plan. Engineer-side rule "no deploys" applies — this audit does not deploy. Igor would run `npm run deploy:stage` (= `wrangler deploy --env stage`) himself.

### 8.2 — Production's most-recent deploy timestamp predates the commit

- **Where:** production worker `oglasino-router-prod`, deployment `de29d03b` at `2026-05-15T11:39:29Z`
- **Severity:** medium
- **Description:** Production was deployed 9 minutes before commit `f1ee682` was timestamped. Most likely benign (deploy-from-working-tree, commit-after), but cannot be verified from `wrangler deployments list` alone since it does not expose the source SHA. If the deploy was made from a working tree that diverged from the eventual commit, production may differ from what `src/index.ts` says at HEAD.
- **Scope:** confirm via `wrangler tail` or the Cloudflare dashboard's source-view, or by deploying the current HEAD on top to remove ambiguity. Routes to Mastermind. Not in scope for backend/web propagation.

### 8.3 — `tests/router.test.ts` updates uncommitted

- **Where:** `tests/router.test.ts` working tree
- **Severity:** low
- **Description:** The test suite expansion (matrix-rowed describes, `admin.bypass.disabled` assertions) is on disk but not in any commit on `stage`. The brief explicitly forbids editing tests in this session, so the audit does not change it. Routes to Mastermind / `issues.md` as a "commit pending."
- **Scope:** Igor commits the file. Not a code change.

### 8.4 — `use.backend.check` read is outside the fail-open `try`/`catch`

- **Where:** `src/index.ts:91–93`
- **Severity:** medium
- **Description:** The two split-flag reads (`maintenance.active`, `admin.bypass.disabled`) at lines 77–80 are wrapped in `try`/`catch` with explicit fail-open (lines 76–87). The follow-up read of `use.backend.check` at lines 91–93 is **not** inside that `try`. If `CONFIG.get("use.backend.check")` throws (e.g. transient KV outage that recovers between the first call and the second), the unhandled exception aborts the request — the opposite of the documented fail-open behavior.
- **Scope:** outside the propagation plan's KV-key split. Routes to `issues.md` as an adjacent observation. Trivial fix (move the read inside the existing `try`, or wrap with its own `try`/`catch`), but not in scope here. The matrix comment block (`src/index.ts:1–13`) does not mention `use.backend.check` at all, so this inconsistency is invisible to a reader skimming the top of the file.

### 8.5 — Matrix-comment block does not document `use.backend.check`

- **Where:** `src/index.ts:1–13`
- **Severity:** low
- **Description:** The header comment lists the two split flags and the 3-row matrix, but does not mention `use.backend.check`, which can force `maintenanceActive` to `true` independently of the KV flag. A reader treating the comment as the spec would miss a real input to the gate.
- **Scope:** adjacent observation. Routes to `issues.md`. The propagation plan is about backend + web reading the same flags; `use.backend.check` is router-internal.

### 8.6 — `Env.WWW_HOST` is a sentinel "empty string when absent" rather than `string | undefined`

- **Where:** `src/index.ts:21`
- **Severity:** low
- **Description:** `WWW_HOST` is typed `string`, with the convention "empty string when env has no www variant." Wrangler-bound vars cannot easily be optional, so this is a known idiom — flagged here only because the propagation plan may add per-surface flags whose absent-on-some-envs nature deserves the same treatment, and it should not be reinvented. Adjacent observation.
- **Scope:** routes to `issues.md` if Mastermind wants it; otherwise wontfix.

### 8.7 — Locale-prefixed admin regex hardcodes `xx-xx` shape

- **Where:** `src/index.ts:64` (`/^\/[a-z]{2}-[a-z]{2}\/admin(\/|$)/i`)
- **Severity:** low (documented in `CLAUDE.md` as a critical-care area — flagged here per Part 4b for completeness, not as a defect)
- **Description:** The regex assumes locale segments are exactly two-letter-dash-two-letter (`rs-sr`, `en-us`). If the web's locale set ever grows to a 3-letter language tag (or `me-cnr`, which is 2-3, currently aliased — see conventions Part 9), the regex would silently miss `/me-cnr/admin/...` and route it through the non-admin block path. Conventions list locales as EN, SR, RU with `me/cnr` aliasing to SR; the worker's regex would not match `/cnr-cnr/...` or any 3-letter form. If a future locale uses anything other than `xx-xx`, this regex needs to change in lockstep.
- **Scope:** known by `CLAUDE.md`, not a present defect. Flagged so the propagation plan does not assume the admin path is a stable matcher; if the web team adds a new locale prefix as part of a future feature, the worker needs a paired change.

---

## Section 9 — Open questions for Mastermind

- **What writes `admin.bypass.disabled`?** The worker only reads. The brief states "backend and web admin controls still read and write the old key name through their existing maintenance toggles." Which surface (backend admin endpoint? web admin UI calling backend?) writes `admin.bypass.disabled`, and is it being written at all today, or is the key currently unset everywhere? Code-only audit cannot answer this — needs backend/web inspection (which Mastermind will commission the next round of briefs on).
- **What is the actual current value of each key in stage and production KV?** Worker is read-only and the engineer has no live KV access per `CLAUDE.md` hard rules. If a key is currently set to a non-`"true"`/non-`"false"` malformed value, the worker silently treats it as `false`. The propagation plan should know the live values; not visible from this repo.
- **Production deployment timestamp ordering (Section 8.2).** Does Mastermind want this verified before the propagation plan ships, or accepted as "almost certainly benign"?
- **Was a stage deploy intended as part of the split-flag rollout and skipped accidentally?** The brief asserts both environments are deployed; the wrangler list contradicts that. Need confirmation: is this a "deploy was missed" gap, or has the propagation always been planned to deploy stage later? If the former, that deploy is a precondition for the rest of the propagation.
- **Should the matrix comment block (`src/index.ts:1–13`) be updated to mention `use.backend.check` as a third input to `maintenanceActive`?** Editing the comment is in scope for the next code-touching brief, not this audit, but it would help any reader reasoning about the gate.

---

## Files touched

- `.agent/audit-worker-maintenance-split.md` (new, this file)
- `.agent/last-session.md` (new, byte-identical copy of this file)

No files outside `.agent/` were modified by this session. Pre-existing uncommitted modifications to `tests/router.test.ts` and `.gitignore`, plus the untracked `CLAUDE.md`, are observed and documented in Section 7 / Section 8 but were not changed.

## Tests

- Ran: `npm test` (= `vitest run`) and `npm run lint` (= `tsc --noEmit`)
- Result: 32 passed, 0 failed; lint clean
- New tests added: none (read-only audit; the brief forbids test edits)

## Cleanup performed

- none needed (read-only audit)

## Obsoleted by this session

- nothing

## Known gaps / TODOs

- The `wrangler dev --env stage --local` boot succeeded (verified: server starts on `http://localhost:8788` with the eight expected bindings shown). Curl-against-localhost matrix smoke tests were not performed, because (a) KV is empty in local mode so the matrix sits at the off / off row by default, (b) requests to upstream `vercel.app` / `api-origin-stage.oglasino.com` from local would either reach real upstreams (out of scope per `CLAUDE.md`) or fail in confusing ways, and (c) the test suite already exercises the matrix exhaustively against the same worker handler with mocked KV. Runtime gating matches the code-read; no Section 8 high-severity contradiction surfaced. Flagged here so the choice is visible.

## Conventions check

- Part 4 (cleanliness): confirmed — read-only audit, no code or comment changes, no new files outside `.agent/`, no logging or TODOs introduced.
- Part 4a (simplicity) / Part 4b (adjacent observations): confirmed — six adjacent observations flagged in Section 8 with file paths, severity guesses, and explicit "out of scope" notes.
- Part 5 (session summary): this file follows the template — `Repo`, `Branch`, `Date`, `Task`, the audit body in place of "Implemented," `Files touched`, `Tests`, `Cleanup performed`, `Obsoleted by this session`, `Known gaps / TODOs`, `Conventions check`, `For Mastermind`. The byte-identical `last-session.md` copy is also written per Part 5.
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): N/A this session — no DTOs, no auth-derived decisions in the worker. The maintenance flag is operationally-supplied KV state, not user-supplied input.

## For Mastermind

### Brief vs reality

1. **Stage is not running the split-flag worker.**
   - Brief says: "The change is deployed to production and stage."
   - Code says / I observed: `wrangler deployments list --env stage` shows the most recent stage deploy as `2026-05-09T23:13:55Z`, six days before commit `f1ee682`. The deployed stage worker is pre-split: it does not read `admin.bypass.disabled` and would behave as the single-flag worker.
   - Why this matters: the propagation plan presumes the worker contract on stage matches `HEAD`. If backend or web start writing `admin.bypass.disabled` against the stage KV today, the stage worker would silently ignore it — appearing to be a backend/web bug when it is a missing deploy.
   - Recommended resolution: add an explicit "Igor runs `npm run deploy:stage`" step at the front of the propagation plan, before backend/web start adopting the new key. The engineer cannot deploy (hard rule); Igor confirms-then-deploys.

2. **Production deploy timestamp predates the commit by 9 minutes.**
   - Brief says: production is on the split-flag worker.
   - Code says / I observed: latest prod deploy `2026-05-15T11:39:29Z`, commit `f1ee682` `2026-05-15T11:48:42Z`. Same day, four production deploys close together. Most likely Igor deployed from a working tree before committing, in which case the prod bytes are consistent with `HEAD`. Cannot be confirmed from `deployments list` alone.
   - Why this matters: if there is any drift between the deployed prod worker and `HEAD`, the propagation plan should know.
   - Recommended resolution: either Igor confirms "yes, I deployed from the eventual-commit working tree," or a fresh prod deploy from `HEAD` is run as the first step of the propagation. Engineer side does not deploy.

### Adjacent observations not raised as Section 8 findings

- The matrix comment block (`src/index.ts:1–13`) is the closest thing this worker has to a spec. The propagation plan touches the *meaning* of `maintenance.active` (if it stays single-key) and adds `admin.bypass.disabled` semantics that downstream code must agree on — both should be reflected in the matrix block in lockstep with whatever code change the next router-side brief makes. The comment block is currently accurate for the deployed prod behavior, but it omits `use.backend.check` (8.5) and does not mention that the two flags compose into a 3-row matrix rather than a 4-row one explicitly.
- The brief's Definition-of-done line "git status shows only the two new files in `.agent/` as additions; nothing else" cannot be satisfied as-is: `M tests/router.test.ts`, `M .gitignore`, and `?? CLAUDE.md` are pre-existing working-tree changes outside `.agent/`. The audit does not touch them (read-only). And separately, `.gitignore` now lists `.agent` itself, which would suppress both new audit files from `git status` entirely. Both observations are surfaced so Mastermind / Igor can choose how to reconcile.

### Surprised by

- The worker is genuinely 207 lines and has no logger, no metrics, no telemetry — exactly as `CLAUDE.md` promised. The hardest decision in this audit was what *not* to add to Section 8 (resist flagging absent-but-defensible patterns). I held to the brief's "exact strings, exact semantics, exact code locations" framing.
- The `use.backend.check` read sitting outside the fail-open `try`/`catch` (8.4) is the one finding I would push hardest on if asked. The matrix comment promises fail-open; one of three KV reads does not honor it. It is medium-severity rather than high only because KV outages tend to fail consistently — if the first two reads succeed, the third usually does too — but it is a real correctness gap in the documented contract.
