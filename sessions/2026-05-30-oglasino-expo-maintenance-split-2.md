# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-30 (session opened 2026-05-29)
**Slug:** expo-maintenance-split (`<n>=2`; session 1 was the read-only audit)
**Task:** Implement the Expo side of `expo-maintenance-split` (feature §5): route mobile API
traffic through the worker's `/api/mobile/*` label, detect the worker maintenance signal
(503 + `X-Oglasino-Maintenance`) in the response interceptor, remove the dedicated
`/public/maintenance/active` poll (boot Gate 1 + exit poller), and delete `maintenanceService`.

---

## Implemented

Five coordinated changes (feature §5.1–5.4), all on the existing single axios chokepoint /
boot-gate machine:

1. **change-2 — Response interceptor maintenance branch** (`src/lib/config/api.ts`).
   Added `onMaintenanceDetected?: () => void` to `AuthInterceptorHooks`, and a branch in the
   response error handler: `503 && headers['x-oglasino-maintenance'] === 'true'` → calls
   `authHooks.onMaintenanceDetected?.()` then re-rejects the original error. Branch is keyed
   off the header, not the bare 503 (a real backend 5xx blip is not a maintenance window).
   Placed right after the no-response guard, before the 403/401/404 branches (no overlap).

2. **Hook wiring** (`src/lib/init/authInterceptors.ts`). Wired
   `onMaintenanceDetected: () => useBootStore.getState().toMaintenance()` alongside the existing
   `onAccountRestored` / `onAccountBanned`. `bootStore` stays a leaf module (it imports no
   HTTP/auth code); the dependency points authInterceptors → bootStore, never the reverse.

3. **change-3 — Boot Gate 1** (`src/lib/store/bootStore.ts`). `runMaintenanceGate` no longer
   calls `checkIfMaintenance()`. It now runs a lightweight liveness probe `healthCheck()`
   (through `/api/mobile/*`) wrapped in `withGateTimeout`: backend down (probe `false`) OR
   timeout OR any throw → `toMaintenance()` + STOP; reachable → populate the non-critical
   `config` slot (unchanged) → advance to Gate 2. Added `import healthCheck from
   '../services/healthCheckService'`; removed the `maintenanceService` import.

4. **change-4 — Exit poller** (`src/components/init/MaintenancePollInit.tsx`).
   `pollMaintenanceOnce` re-pointed from `checkIfMaintenance()` to `healthCheck()`:
   `backendUp === true` → `reEnter()`; otherwise stay. The 5s-while-in-maintenance interval and
   the non-destructive `reEnter()` are unchanged.

5. **Deletions / dead-code resolution.** Deleted `src/lib/services/maintenanceService.tsx`
   (zero callers after 3 & 4, grep-confirmed). Wired the previously-**dead**
   `healthCheckService` (`GET /public/health/check`, default export `healthCheck`) into real
   use as the Gate-1 + poll probe — resolving the Part 4b dead-code observation from session 1;
   removed its unused `catch (error)` binding.

**change-1 (`/api/mobile/*` routing) is env-only — no code change.** The base URL is already
fully tier-specific via `EXPO_PUBLIC_API_URL` (read verbatim in `api.ts`). Routing through the
worker is achieved by Igor setting the per-tier env values (below); `api.ts` stays a dumb
reader. The `.env.*` files are git-ignored and were not edited.

Tests added/updated (vitest): new `api.interceptor.test.ts` (3 cases: 503+header → hook;
bare 503 → no hook; non-503+header → no hook — driven end-to-end through the real interceptor
via an adapter swap); rewrote `MaintenancePollInit.test.ts` (healthCheck up → reEnter, down →
stay, reject → stay); updated `bootStore.test.ts` (Gate-1 mock `maintenanceService` →
`healthCheckService`, liveness-boolean semantics, three titles).

## Brief vs reality

This did not block implementation (the mobile DoD and feature §5 were coherent), but two items
are worth surfacing:

1. **`.agent/brief.md` mixes backend content into the mobile brief.**
   - Brief says: the top half ("FORK 1 … New controller in the admin.controller package …
     report the final path string so the web brief quotes it"; "FORK 2 … DefaultCloudflareKvService
     … @PreAuthorize … @WebMvcTest"; "ALSO: log the underlying gap … issues.md entry text … in
     oglasino-backend") is all **oglasino-backend** engineer content.
   - Code says: my repo is `oglasino-expo`; none of that is actionable here. The mobile-relevant
     content is the `DEFINITION OF DONE (continued)` and `VERIFY BEFORE WRITING CODE` sections.
   - Why this matters: the backend `issues.md` draft ("No method-security behavioral test
     coverage in oglasino-backend …") is routed to the wrong repo's brief — it is the Backend
     engineer's to draft, not mine. I have not actioned it.
   - Recommended resolution: confirm the mobile brief should carry only §5 mobile scope; route
     the FORK/issues.md drafts to the Backend session. (Drafted the backend issues.md text under
     "For Mastermind" for traceability only.)

2. **`healthCheckService` shape differs from the session-1 audit's framing.** It is a **default
   export named `healthCheck`** hitting `GET /public/health/check` (not a named
   `healthCheckService`). Consumed it as such. No contract impact.

**VERIFY-before-writing-code results (all confirmed; no blocking mismatch):**
- *Tier signal in code:* `APP_ENV` IS readable at runtime via `Constants.expoConfig.extra.env`
  (`app.config.ts` threads `env: ENV`), but `api.ts` does not read it. This confirmed change-1
  can be — and is — **env-only** rather than a code tier-check.
- *Interceptor shape / `configureAuthInterceptorHooks` signature:* hooks object
  `{ onAccountRestored?, onAccountBanned? }`, registered by `registerAuthInterceptors()` (with a
  `registered` guard, `@/lib/...` alias imports). Extended with `onMaintenanceDetected?`.
- *`checkIfMaintenance` caller count after changes 3 & 4:* was 2 (Gate 1 + MaintenancePollInit) →
  now **0** → `maintenanceService.tsx` deleted; repo-wide grep for `checkIfMaintenance` /
  `maintenanceService` returns nothing.

## Files touched

- `src/lib/config/api.ts` (M) — hook type field + 503/header maintenance branch.
- `src/lib/init/authInterceptors.ts` (M) — `onMaintenanceDetected` → `toMaintenance` wiring.
- `src/lib/store/bootStore.ts` (M) — Gate 1 `checkIfMaintenance` → `healthCheck` liveness probe;
  import swap.
- `src/components/init/MaintenancePollInit.tsx` (M) — poll probe re-pointed to `healthCheck`.
- `src/lib/services/healthCheckService.ts` (M) — removed unused `catch (error)` binding (now live).
- `src/lib/services/maintenanceService.tsx` (**D**) — deleted (zero callers).
- `src/lib/config/api.interceptor.test.ts` (new) — interceptor maintenance-branch tests.
- `src/components/init/MaintenancePollInit.test.ts` (M) — re-pointed to `healthCheck`.
- `src/lib/store/bootStore.test.ts` (M) — Gate-1 probe mock + liveness semantics.
- `.agent/2026-05-30-oglasino-expo-maintenance-split-2.md`, `.agent/last-session.md` (this summary).

(Note: many of these appear untracked in `git status` because the whole boot-redesign/Φ-foundation
work is uncommitted on `new-expo-dev` per Igor's isolation strategy — not new this session.)

## Tests

- `npx tsc --noEmit` — clean (exit 0).
- `npx eslint <touched files>` — **0 errors**, 6 warnings, all pre-existing baseline patterns
  (the established `vi.mock`-then-`import` ordering in test files; the untouched `Array<T>` on
  `api.ts:29`; the untouched i18n `no-named-as-default-member` on `bootStore.ts` 535/563).
- `npx vitest run` — **280 passed (280), 17 files passed**. Includes the 3 new interceptor tests,
  the rewritten poll tests, and the updated bootStore Gate-1 tests.
- `npx expo-doctor` — not run; no dependency change this session (see below).

## Cleanup performed

- Deleted obsolete `maintenanceService.tsx` and its `/public/maintenance/active` call.
- Removed the dead-code status of `healthCheckService` by wiring it into Gate 1 + the poll.
- Removed an unused `catch (error)` binding in `healthCheckService.ts`.
- No commented-out code, no `console.log`/debug logging, no `TODO`/`FIXME` introduced.
- Removed a false-start set of stray test files I created mid-session in incorrect `__tests__/`
  directories before re-establishing the repo's real layout (siblings, vitest); none remain.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- issues.md: no change by me. (The brief's `issues.md` draft is a **backend** entry — see
  "For Mastermind"; it routes to the Backend session / Docs/QA, not from this repo.)
- state.md: no oglasino-docs config-file change is required by the code itself (code-only, as the
  brief predicted). **Closure-gate note:** the Expo backlog row (Order 3 "Expo maintenance split —
  cleared to start") is now code-complete on the Expo side, pending Igor's env values + preview
  rehearsal. Drafted the suggested state.md wording under "For Mastermind" for Docs/QA — I do not
  edit state.md.

## Obsoleted by this session

- `src/lib/services/maintenanceService.tsx` and the `GET /public/maintenance/active` poll
  (boot-time + exit). Per feature §7 sequencing step 5, this is what unblocks the backend deleting
  `GET /public/maintenance/active` + the `maintenance.active` config row — **no Expo code path
  calls that endpoint after this session** (grep-confirmed).
- The dead-code flag on `healthCheckService` (session-1 Part 4b observation) — now a live caller.

## Conventions check

- **Part 4 (cleanliness):** clean — file deleted in the same session that obsoleted it; unused
  import binding removed; no stray debug; tsc/lint/tests green for touched paths.
- **Part 4a (simplicity):** see evidence in "For Mastermind". Net −18 lines of product code
  (whole `maintenanceService` removed); each addition mirrors an existing pattern.
- **Part 4b (adjacent observations):** resolved the prior dead-`healthCheckService` observation.
  No new adjacent issues surfaced.
- **Part 6 (translations):** N/A — no translation keys touched. Maintenance UI
  (`<BaseSiteSelector isMaintenance />`) unchanged per §5.5.
- **Part 8 (architectural defaults):** the worker-is-edge-boundary principle is preserved — the
  `/api/mobile/*` label carries no authority; mobile keys off the worker's 503+header contract.
- **Waiting rule:** satisfied — state.md greenlights the Expo side per brief override ("expo side
  cleared to start"); router/backend are in progress, and end-to-end verification is Igor's
  preview-against-stage step (engineers do not deploy).

## Known gaps / TODOs

- None in code. End-to-end behavior (worker 503 → maintenance; clear → exit via re-pointed poll)
  is verified by unit tests at the gate/poll/interceptor seams; the full cross-repo rehearsal is
  Igor's preview build against the stage worker (DoD §9 / feature §7), which I cannot run.

## For Mastermind

- **Per-tier `EXPO_PUBLIC_API_URL` values Igor must set** (git-ignored `.env.*`; I cannot edit
  them). From feature §5.2:
  - **development:** `http://10.184.52.91:8080/api`  — unchanged; local backend, **no** `/mobile`
    (no worker in dev; boot relies on the 5s gate timeout to show maintenance). Update the LAN IP
    as needed for your machine.
  - **preview:** `https://api-stage.oglasino.com/api/mobile`  — through the **stage worker**
    (replaces the raw LAN IP `http://172.21.173.91:8080/api`).
  - **production:** `https://oglasino.com/api/mobile`  — through the **prod worker** (replaces
    `https://oglasino.com/api`).

- **change-1 mechanism — env-only (not a code tier-check).** `APP_ENV` is readable at runtime
  (`Constants.expoConfig.extra.env`), so a code branch was possible, but the base URL is already
  tier-specific via `EXPO_PUBLIC_API_URL`; appending `/mobile` for preview/prod is a per-tier env
  value change with **zero** code in the single chokepoint. A code tier-check would duplicate tier
  knowledge already in the env var and add a conditional for no behavioral gain (Part 4a). Chosen:
  env-only.

- **change-3 shape — Gate 1 keeps a liveness call (not "fold into the next gate").** The boot
  machine advances unconditionally between gates and gates do not read `status`, so relying solely
  on the interceptor's `toMaintenance()` would be unsafe — a later gate (baseSite/freshness) could
  overwrite `'maintenance'` with `'ready'`. So Gate 1 keeps an explicit probe that **stops** the
  sequence on backend-down. I rejected folding the signal into `getAppConfiguration` because
  config is deliberately **non-critical** (its failure must not brick boot; the two
  `getConfiguration` readers tolerate missing keys) — making it the liveness gate would change
  documented behavior. `healthCheck` (`/public/health/check` through `/api/mobile/*`) is the
  probe; the interceptor (change-2) is defense-in-depth for any other `/api/mobile/*` call that
  returns 503+header mid-boot or mid-session.

- **Part 4a simplicity evidence (required):**
  - *Added (earned complexity):* one interceptor branch + one optional hook field + one wiring
    line (mirroring `onAccountRestored`/`onAccountBanned`); a one-for-one probe swap in Gate 1 and
    in the poll. No new abstraction, store, or module.
  - *Considered and rejected:* (a) a code `APP_ENV` tier-check for `/api/mobile` routing — rejected
    for env-only; (b) interceptor-only maintenance during boot — rejected (later gates could
    overwrite `status`); (c) making `getAppConfiguration` the Gate-1 liveness gate — rejected
    (would make config critical); (d) a new HTTP-mock harness for the interceptor test — used a
    minimal adapter swap on the existing axios instance instead (no new infra, consistent with the
    brief's FORK-2 "don't stand up parallel test infra" guidance).
  - *Simplified / removed:* deleted `maintenanceService.tsx` wholesale; gave the previously-dead
    `healthCheckService` a real caller; removed an unused `catch` binding. Net product-code
    reduction.

- **Suggested state.md Expo-backlog wording (Docs/QA — I did not edit it):** Order 3 "Expo
  maintenance split" → *"expo side code-complete on `new-expo-dev` (interceptor 503+header branch,
  Gate-1 liveness probe via `/api/mobile/*` health check, exit poller re-pointed, dedicated
  `/public/maintenance/active` poll + `maintenanceService` removed; tsc/lint/280 tests green).
  Remaining: Igor sets per-tier `EXPO_PUBLIC_API_URL` (dev local no-`/mobile`, preview
  api-stage.oglasino.com/api/mobile, prod oglasino.com/api/mobile) and runs the preview-against-
  stage rehearsal (DoD §9). Unblocks backend cleanup step 5 — Expo no longer polls
  `/public/maintenance/active`."*

- **Backend `issues.md` draft (mis-routed to the mobile brief; not mine to author — forwarded for
  the Backend session):** *"No method-security behavioral test coverage in oglasino-backend; admin
  `@PreAuthorize` gates verified only by annotation presence + manual check. Future
  testing-infrastructure decision. (low)"*

- **Cross-repo contract consumed:** mobile keys off the worker's mobile-maintenance response —
  `503` + header `X-Oglasino-Maintenance: true` (feature §3.5). If the router freezes a different
  header name/case, the interceptor branch (`x-oglasino-maintenance`, lowercased by axios) and the
  per-tier hosts must be reconciled before the preview rehearsal.
