# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev (the brief's `dev` slot — Igor's deliberate isolation per the 2026-05-25 "Expo foundation work is tentative" decisions.md entry)
**Date:** 2026-05-28
**Task:** Φ3 Brief 18 — D1 (boot-state hardening of `apiStore.ts`) + D3a (break Cycle A by extracting interceptor-side auth effects from `api.ts` into a registered hook).

## Implemented

- **D1 — apiStore rewrite.** `src/lib/config/apiStore.ts` rewritten to the codes-derived, `globalThis`-pinned singleton shape from Brief 17's design. `state` is a single object pinned on `globalThis[KEY]` with `baseSite`, `lang`, and `waiters[]`. `waitForBootstrap()` derives the barrier from `state.baseSite !== null` on every call — structurally impossible to desync from codes. `setCodes(baseSite, lang)` atomically writes both fields then flushes waiters; `setLangOnly(lang)` updates language without touching the barrier. Old `bootstrapResolve` / `bootstrapPromise` couple and the old `setCurrentBaseSiteCode` / `setCurrentLangCode` setters deleted. `getCurrentBaseSiteCode` / `getCurrentLangCode` / `waitForBootstrap` names preserved so the interceptor at `api.ts:54` needs no edit beyond import rewiring (which already happened via the apiStore module — `api.ts` still imports the same three symbols).
- **D1 — three call sites in `AppContext.tsx` updated.** Import on line 3 swapped to `setCodes, setLangOnly`. Bootstrap success branch (originally lines 127–128) replaced with `setCodes(storedBaseSite.code, language.code)`. Base-site switch action (originally lines 207–208) replaced with `setCodes(site.code, lang.code)` — variable names `site` / `lang` already matched the design's assumption (no Brief vs reality needed). Language-only switch action (originally line 238) replaced with `setLangOnly(lang.code)`.
- **D3a — Cycle A broken.** `src/lib/config/api.ts` no longer imports `useAuthStore`. New `AuthInterceptorHooks` type with `onAccountRestored?` / `onAccountBanned?` callback slots, module-local `authHooks` state, and `configureAuthInterceptorHooks(hooks)` exported. The response interceptor's `x-account-restored` branch (formerly `api.ts:64`) now calls `authHooks.onAccountRestored?.()`. The `USER_BANNED` / `EMAIL_BANNED` branch (formerly `api.ts:88`) now calls `authHooks.onAccountBanned?.()`. The 401 token-refresh path is unchanged — it uses `auth.signOut()` and `auth.currentUser` from the Firebase SDK, no `useAuthStore` involvement.
- **D3a — `src/lib/init/authInterceptors.ts` created.** Idempotent `registerAuthInterceptors()` that imports `useAuthStore` (so the side of the cycle that needed `authStore` now lives outside `api.ts`) and wires the two callbacks to `useAuthStore.getState().setRestored(true)` / `setAccountBanned(true)`. The `registered` guard makes double-registration a no-op.
- **D3a — module-import-time registration from `app/_layout.tsx`.** `registerAuthInterceptors()` is called at module scope (after imports, before `Platform.OS === 'android'` side-effect block). Because `BACKEND_API` is constructed at module-import time and no request fires until React mounts, the hooks are guaranteed registered before any interceptor can trigger.

No feature behavior changes. Auth flows, account-banned dialog, account-restored, product fetch, chat, filters all behave identically — only the plumbing moved.

## Files touched

- `src/lib/config/apiStore.ts` — full rewrite (~29 lines net, replaces 31 lines).
- `src/components/context/AppContext.tsx` — import on line 3 + three call-site edits (net ~ −3 lines).
- `src/lib/config/api.ts` — removed `useAuthStore` import; added `AuthInterceptorHooks` type + `authHooks` module state + `configureAuthInterceptorHooks` export (lines 6–13); swapped two interceptor branches to use `authHooks.onAccountRestored?.()` / `authHooks.onAccountBanned?.()` (lines 72 and 96).
- `src/lib/init/authInterceptors.ts` — new file (12 lines).
- `app/_layout.tsx` — added `registerAuthInterceptors` import + module-scope call.

## Tests

- Ran: `npx tsc --noEmit` — exit 0.
- Ran: `npm run lint` — exit 0, 73 warnings, 0 errors (matches brief ceiling of ≤73).
- Ran: `npm test` — `Test Files 7 passed (7) | Tests 109 passed (109)`. Meets the brief's ≥109 floor exactly.
- New tests added: none. No new code path is added that wasn't already covered by the existing call-site behavior. Adding a unit test for `setCodes` / `waitForBootstrap` was considered (see Part 4a "considered and rejected"); inlined as smoke-only because the freshness derivation is two lines and the integration paths through `AppContext.tsx` exercise it.

## Cleanup performed

- Deleted `bootstrapResolve` / `bootstrapPromise` couple from `apiStore.ts` (replaced by codes-derived `waitForBootstrap`).
- Deleted old `setCurrentBaseSiteCode` / `setCurrentLangCode` exports from `apiStore.ts` (replaced by `setCodes` / `setLangOnly`).
- Deleted `useAuthStore` import from `api.ts` (cycle break — the import is now in `authInterceptors.ts` where it does not feed back into the HTTP layer).
- Deleted the two direct `useAuthStore.getState().setRestored(true)` / `useAuthStore.getState().setAccountBanned(true)` calls from `api.ts` interceptors.

Per the brief's "Do not remove the `[DBG]` log yet" rule, the `console.log('[DBG]', …)` block at `api.ts:43–53` is intentionally preserved. It will be removed in Brief 19's cleanup pass. The misspelled `// TODO: Translations PLUS after meintanence page do better check` in `AppVersionConfigInit.tsx:29` is also out of scope here (Brief 19 cleanup target).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change. The close-out chat for the refactor (likely after Brief 19) will draft a single `decisions.md` entry summarizing the boot-refactor — Brief 17's "For Mastermind" pre-staged the title.
- state.md: no change. Brief 18 is one step inside the in-flight Φ3 trajectory; no backlog row movement.
- issues.md: no change.

## Obsoleted by this session

- Deleted in this session: `bootstrapResolve`, `bootstrapPromise`, `setCurrentBaseSiteCode`, `setCurrentLangCode` from `apiStore.ts`; `useAuthStore` import + direct interceptor calls in `api.ts`.
- Nothing left for follow-up. All deletions targeted by Brief 17 D6 steps 1+2 are complete.

## Conventions check

- Part 4 (cleanliness): confirmed. Zero unused imports / dead code introduced; `tsc`, `lint`, `npm test` all green; touched files have no commented-out code, no orphaned debug logs added (the preserved `[DBG]` block is pre-existing and explicitly kept per the brief).
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): two pre-existing observations re-flagged (low severity each) in "For Mastermind." Neither fixed because both are explicitly out of scope per the brief.
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (architectural defaults) — the new module-import-side-effect call (`registerAuthInterceptors()` at module scope in `app/_layout.tsx`) is the minimal, explicit equivalent of an init wiring file; it follows Brief 17 D3a's "locked call site" decision. Part 5 (session template) — followed below.

## Known gaps / TODOs

- The `[DBG]` log in `api.ts:43–53` is deliberately left for Brief 19's cleanup pass (carries diagnostic signal through this refactor's smoke per the brief).
- `globalThis`-pinning's behavior under a full Metro `r-r` reload is documented in Brief 17's smoke list (case 12) but not exercised in this session — code changes only, no manual smoke. Engineer's expectation matches Brief 17: `r-r` triggers a JS context reset, `globalThis` resets with it, codes start null again, behavior matches a true cold start.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - `AuthInterceptorHooks` type + `authHooks` module-local + `configureAuthInterceptorHooks` export in `api.ts` (~7 lines). Earned: this is the minimal indirection that lets `api.ts` stop importing `useAuthStore`, closing Cycle A. Function-export shape matches surrounding code style (Part 4a "match the surrounding code's style").
    - `src/lib/init/authInterceptors.ts` (12 lines). Earned: it is the only place that holds the `useAuthStore → api.ts` direction of the dependency, registered once at module-import time. The `registered` guard is two lines and prevents double-registration under Fast Refresh re-evaluations of `app/_layout.tsx`.
    - `globalThis`-pin on the `apiStore` singleton (~3 lines, the `KEY` + `g[KEY] ??= …` triple). Earned: standard React-Native / Metro pattern (mirrors zustand and TanStack Query) for surviving Fast Refresh closures over module state. Brief 16's audit established the concrete failure mode this prevents.
  - Considered and rejected:
    - **Adding a unit test for `setCodes` / `waitForBootstrap`.** Considered; rejected for this brief. The freshness derivation is two reads + one push; the actual integration is exercised every cold start through `AppContext.bootstrap`. A unit test on the freshness logic alone would lock in the implementation shape without adding coverage the existing 109 tests don't already provide indirectly via `apiStore` consumers.
    - **Wrapping `registerAuthInterceptors()` in a `try/catch` in `app/_layout.tsx`.** Considered; rejected. The function is two synchronous writes (a boolean flip + an assignment). It cannot throw without indicating a deeper bug; swallowing would mask that bug.
    - **Naming the new file `registerInterceptors.ts` per Brief 17's optional fallback shape.** Considered; rejected. The brief's primary recommendation was `authInterceptors.ts` because the file is auth-specific. If a non-auth interceptor hook is added later, it can live in a sibling file (`pushInterceptors.ts`, etc.); keeping the file name aligned with its content is simpler than a generic registry.
    - **Adding a `unregisterAuthInterceptors()` symmetric counterpart.** Considered; rejected. No call site needs to undo registration. YAGNI.
  - Simplified or removed:
    - Removed the `bootstrapResolve` + `bootstrapPromise` couple in `apiStore.ts`. Net `apiStore.ts` is roughly the same line count but one fewer state-coupling invariant ("resolver fires iff code is set" is now structural, not by-convention).
    - Removed the `useAuthStore` import and two direct `getState()` calls from `api.ts`. The interceptor file is now a pure HTTP-plumbing module with no React-tree coupling.
- **Confirmation requested in the brief:**
  - **No other `useAuthStore` usages in `api.ts` beyond the two named.** Confirmed by grep. The 401 token-refresh path uses `auth.signOut()` and `auth.currentUser` from the Firebase SDK (`firebase/auth`), not `useAuthStore`. Brief vs reality: clean.
  - **`setBaseSiteForCode` / `setLanguageForCode` variable names.** Confirmed — `site` and `lang` in `setBaseSiteForCode`, `lang` in `setLanguageForCode`. Brief vs reality: clean.
  - **`globalThis` typing.** The shape `globalThis as typeof globalThis & { [KEY]?: ApiStoreState }` typechecked cleanly under the project's `tsc` config with no `// @ts-expect-error` or `as any` escape hatch. The `??=` assignment narrows correctly; downstream reads through `state` are `ApiStoreState` (non-optional) because the `??=` immediately assigns a concrete object. Brief vs reality: clean.
- **Metro require-cycle warning count.** Verified structurally rather than by running Metro: `grep -rn "useAuthStore\|from.*authStore" src/lib/config/` returns empty after the edit. Cycle A (`api.ts → authStore → authService → api.ts`) no longer exists at the import-graph level. Brief 17's design doc identifies one Metro warning for Cycle A and two for Cycle B (chat triangle plus authStore's three imports). Expected after this brief: Metro emits **one fewer** require-cycle warning at boot. Igor will see this in the Metro log when next launching the dev build; no Metro run was performed in this session because (a) the cycle break is structural and grep-verifiable, and (b) running Metro is outside the engineer-agent scope.
- **Part 4b adjacent observations (re-flagged, all already in Brief 17's Part 4b):**
  1. `api.ts:44–53` still carries the `console.log('[DBG]', …)` block — deliberately preserved per Brief 18's hard rule, will be removed in Brief 19. Severity: low. **I did not fix this because the brief explicitly forbids removing it in this session.**
  2. `AppVersionConfigInit.tsx:29` still carries the misspelled `// TODO: Translations PLUS after meintanence page do better check`. Severity: low. **I did not fix this because it is out of scope for Brief 18 — Brief 17's design earmarks it for Brief 19 cleanup or engineer's choice; this session is plumbing-only.**
- **Closure gate.** No implicit config-file dependency. Brief 19 (render gate + ProductList guard + cleanup) is the natural next session. After Brief 19 + smoke, a single close-out `decisions.md` entry would be appropriate; Brief 17 pre-staged the title "Boot/HTTP layer refactor: codes-derived barrier, per-screen base-site render gate, api↔authStore cycle broken."
