# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Step 4 of `features/expo-boot-redesign.md` Part 8 — replace the Gate 1 (maintenance) and Gate 2 (version) STUBS in `src/lib/store/bootStore.ts` with real logic. Introduce the shared timeout-and-divert wrapper (`withGateTimeout`) that expresses the spec's single backend-down rule. Convert the sync stubs to async real gates WITHOUT changing the call-flow shape proven in step 3. Gates 3 and 4 remain stubs; the machine still reaches `ready` on the happy path through them.

> **Reconstruction note (per Part 5 honest-record-keeping).** The engineer who did the implementation work for this step crashed (VS Code lost) before writing the session summary. This summary was reconstructed by a second engineer in a separate session, working from the on-disk artifacts only: the brief at `.agent/brief.md`, the four touched files (`bootGate.ts`, `bootGate.test.ts`, `bootStore.ts`, `bootStore.test.ts`), the step-3 summary as a template, and a re-run of the test / `tsc` / lint commands to confirm the state described below. The voice may differ from prior boot-redesign session summaries by the same engineer; the recorded facts are verified against the working tree as of 2026-05-28 14:30 local. Flagging here so Docs/QA at archive time knows why this summary's tone is not continuous with sessions 1–3.

## Implemented

- New file `src/lib/store/bootGate.ts` carrying the shared timeout wrapper:
  - `GATE_TIMEOUT_MS = 5000` — single named constant for the spec's T (Part 2). Constant, not config, per Part 4a — no foreseeable second value.
  - `class GateTimeoutError extends Error` — a distinct, typed sentinel so gates can match on `instanceof` and route a timeout specifically. The class form (over a string-coded `Error`) gives `instanceof` matching plus a stable `.name` for logs, and costs nothing extra.
  - `withGateTimeout<T>(fn: () => Promise<T>): Promise<T>` — races `fn()` against a 5s timer. fn settles first → resolves/propagates its result; timer wins → rejects with `GateTimeoutError`. Mechanism only — does NOT itself call `toMaintenance`. The wrapper is the racer; the gate is the policy.
  - Implementation note: written as a single outer Promise (not `Promise.race`) so a never-settling `fn()` does not leave a dangling timeout-promise rejection after the outer promise has already settled. The timer is `clearTimeout`-ed in both fn-resolve and fn-reject branches.
- `src/lib/store/bootStore.ts` — Gate 1 (`runMaintenanceGate`) replaced. Now `async`:
  - Calls `checkIfMaintenance()` wrapped in `withGateTimeout`.
  - Maintenance on (`true`) → `toMaintenance()`, STOP.
  - Any throw — timeout or otherwise — → `toMaintenance()`, STOP. (The wrapper is the backstop the maintenance service's internal `catch → return true` cannot provide; a genuine HANG would never reach the service's catch.)
  - Off (`false`) → `await runVersionGate()`.
- `src/lib/store/bootStore.ts` — Gate 2 (`runVersionGate`) replaced. Now `async`:
  - Calls `getAppVersionConfig()` wrapped in `withGateTimeout`.
  - Timeout (`GateTimeoutError`) → `toMaintenance()`, STOP. Spec rule: a version-call HANG is maintenance, never silently "you're fine."
  - Resolved DTO: `forceUpdate` checked first (defensive — force wins over optional even if both somehow set) → `toUpdateRequired()`, STOP. Else `optionalUpdate` → set `softUpdate: true` (flag, not status) and advance. Else advance.
  - Resolved `undefined` (the service's catch-return for a swallowed non-timeout error the timer did not catch) → advance, do NOT block, do NOT set `softUpdate`. Documented decision below.
  - Non-timeout throw: structurally unreachable today (the service swallows its own errors), but a defensive fall-through in the `catch` advances without blocking — same posture as the swallowed-undefined branch.
  - After the `try/catch`, the gate advances via `await get().runBaseSiteGate()`. (Stub is still sync; `await` of a sync return is a no-op. When step 5 makes it async the call site does not change.)
- `src/lib/store/bootStore.ts` — async-signature updates on `start()`, `reEnter()`, `runMaintenanceGate`, `runVersionGate`. `start()` now `await`s `runMaintenanceGate()`. `reEnter()` `await`s `start()`. Resolved slots are NOT touched on entry (Invariant 3 preserved).
- New file `src/lib/store/bootGate.test.ts` — 3 tests for the wrapper in isolation: resolves-before-timeout, timer-wins-throws-GateTimeoutError, non-timeout-rejection-propagates-unchanged. Fake timers (`vi.useFakeTimers()`) — no actual 5s wait.
- `src/lib/store/bootStore.test.ts` — step-3 tests updated for the now-async gates 1/2 (added `await` on `start()` / `reEnter()` calls; existing assertions kept, none weakened). New tests added:
  - Gate 1 (3): maintenance-false-advances-to-version, maintenance-true-diverts, maintenance-timeout-diverts. Mocks via `vi.hoisted` + `vi.mock` on `maintenanceService` and `appVersionConfigService`; fake timers for the timeout case.
  - Gate 2 (6): forceUpdate-true-diverts (softUpdate stays false; baseSite gate not called), optionalUpdate-true-sets-softUpdate-and-advances, neither-flag-advances-without-softUpdate, undefined-advances-without-softUpdate (the documented swallowed-non-timeout decision), force-wins-over-optional (both true → update-required, defensive check order), version-timeout-diverts.
  - Invariant 3 re-confirmed with real gates 1/2 actually running (mocked services return clean values; resolved slots survive `start()` re-entry).
  - Happy-path reach-ready test still green with real gates 1/2 and stubbed gates 3/4 — maintenance false → version neither → stubbed baseSite → stubbed freshness → `'ready'`.

## Files touched

- src/lib/store/bootGate.ts (+49 / −0, new)
- src/lib/store/bootGate.test.ts (+38 / −0, new)
- src/lib/store/bootStore.ts (modified — Gate 1 body replaced, Gate 2 body replaced, signatures async, imports added for the two services + the wrapper)
- src/lib/store/bootStore.test.ts (modified — `await` added on async-gate calls in step-3 tests; Gate-1 / Gate-2 real-logic describe blocks added; service mocks declared at module top via `vi.hoisted` + `vi.mock`)

No other source files modified. Old boot path (`AppContext.tsx`, `apiStore.ts`, `_layout.tsx`, `AppVersionConfigInit.tsx`, `src/i18n/loader.ts`) untouched per the "build alongside" rule.

## Tests

- Ran: `npx vitest run src/lib/store/bootGate.test.ts src/lib/store/bootStore.test.ts`
- Result: 29 passed, 0 failed (2 files) — 26 in `bootStore.test.ts` (17 carried from step 3 + 9 new) and 3 in `bootGate.test.ts`.
- Ran: `npm test` (full suite).
- Result: 138 passed, 0 failed (9 files). Up from the step-3 baseline of 126 by exactly the 12 new tests (3 wrapper + 9 gate-real).
- Ran: `npx tsc --noEmit`. Clean (zero errors).
- Ran: `npx expo lint -- --quiet src/lib/store/bootGate.ts src/lib/store/bootGate.test.ts src/lib/store/bootStore.ts src/lib/store/bootStore.test.ts`. Zero warnings on touched paths. (Repo-wide pre-existing warning count unchanged.)
- `npx expo-doctor` not run — no dependency changes this session.
- No real network calls in tests; both services mocked via `vi.hoisted` + `vi.mock`. Fake timers (`vi.useFakeTimers()`) for every timeout case, so no test waits 5s of wall time.

## Cleanup performed

- None needed. Pure additions plus in-place replacement of two stub bodies with their real bodies. No commented-out code, no `console.log`, no TODO/FIXME, no unused imports, no dead helpers. The two real-gate bodies fully replace their step-3 stub bodies — no orphaned stub code left behind.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change. The undefined-handling posture for Gate 2 is a brief-internal decision rationalized below in "For Mastermind"; it is implementation detail of the spec's existing Part 3 fail-open rule, not a new project-wide decision.
- state.md: no change. The expo-boot-redesign feature is mid-implementation (this is step 4 of 7 in Part 8 of the spec). The Φ3 "verifying" status referenced in state.md for the boot-loading issue stays unchanged; it flips when step 7 ships and the old path is torn out.
- issues.md: no change. The pre-existing `[BOOT]` instrumentation entry remains open and is owed to step 7 (portal wiring + teardown), not this brief.
- No Expo backlog row applies. This is in-progress engineering on the boot redesign, not a mobile adoption of a `web-stable` / `shipped` feature.

## Obsoleted by this session

- The two STUB bodies of `runMaintenanceGate` and `runVersionGate` from step 3. Their stub comments (which named "step 4" as the brief that fills them in) are gone; the new bodies stand on their own. The other two stubs (`runBaseSiteGate`, `runFreshnessGate`) remain in place — step 5 and step 6 land their real bodies.
- Nothing else. The old boot path stays live until step 7 per the brief's hard rules.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no unused imports/variables/functions, no `console.log`, no TODO/FIXME. `tsc` clean. Lint clean on the four touched paths. New tests pass; full suite green at 138 (up by exactly 12 over the step-3 baseline of 126).
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): N/A. Nothing surfaced this session that is not already catalogued by `.agent/audit-expo-boot-redesign.md` or tracked in `issues.md`.
- Part 6 (translations): N/A. No user-facing strings added. The maintenance / update-required / etc. surfaces are wired in step 7 with existing translation keys.
- Spec Parts 1, 2, 3 — confirmed:
  - **Invariant 1** (one effect, empty deps): structural — this module still has no effect and no internal subscription. `start()` remains the single entry point designed for one `useEffect` with `[]` deps; wiring is step 7.
  - **Invariant 2** (machine writes status; view reads it): every `status` write in the file is from one of the four `toX()` helpers or the freshness gate's final `set({ status: 'ready' })`. No reactive trigger, no internal subscription, no effect added.
  - **Invariant 3** (re-entry destroys nothing): `start()` sets `status: 'booting'` and runs the gates; it does NOT touch `selectedBaseSite` / `language` / `config` / `softUpdate`. Both Invariant-3 tests (via `start()` and via `reEnter()`) pass with real gates 1/2 actually executing — the seeded resolved-slot references are identity-equal after re-entry.
  - **Spec Part 2 single backend-down rule**: expressed in exactly one place — `withGateTimeout` in `bootGate.ts`. Gate 1 and Gate 2 both use it. Gates 3 and 4 adopt it in their own briefs (steps 5, 6).
  - **Spec Part 3 version classification**: forceUpdate-first defensive ordering; optionalUpdate is a flag, never a status; undefined and non-timeout swallow are non-blocking. Documented in "For Mastermind."
  - **Structural defense (Part 1)**: `bootStore.ts` imports `zustand`, `BaseSiteDTO`, `LanguageDTO`, `ConfigMap`, `withGateTimeout` + `GateTimeoutError`, `checkIfMaintenance`, `getAppVersionConfig` — and nothing else. Verified `bootStore.ts` and `bootGate.ts` contain ZERO imports matching `authStore|useActiveChatStore|useChatListStore|useChatNavStore|useChatBlockStore` (the only grep hit is the doc-comment line at `bootStore.ts:34` that names what is NOT imported). The boot store remains a leaf w.r.t. the `authStore ↔ chat-store` require cycle documented in the Expo audit §5.1.

## Known gaps / TODOs

- **Active-checker / maintenance-poll deferred to step 7.** The maintenance-on case is handled at boot (Gate 1 diverts to `maintenance` and stops). The poll that pings while maintenance is on and calls `reEnter()` on clear is NOT in this brief — it lives with the mount-effect wiring in step 7. `reEnter()` is exposed (behaviourally identical to `start()`) so the step-7 active-checker has a stable named entry point. This brief only does the boot-time Gate 1 check.
- **Stubs remain in Gate 3 (base-site) and Gate 4 (freshness).** Real bodies land in steps 5 and 6. Both stubs still advance synchronously (Gate 3 → Gate 4 → `set({ status: 'ready' })`), so the happy path reaches `ready` end-to-end after the real Gate 1 / Gate 2 pass. When steps 5 / 6 make them async, the existing `await get().runBaseSiteGate()` call site in Gate 2 does not change shape.
- **Mount-effect wiring (step 7).** Single `useEffect` with empty deps that calls `start()`. Out of scope here.
- **softUpdate UI / modal.** `softUpdate: true` is set by Gate 2 in the optional-update path; the consumer modal ships in a later brief.
- **Old boot path remains live.** Step 7 reviews `<RequireBaseSite>` against the `portal-mounts-only-in-ready` invariant and tears down the old orchestration (Promise.all in `AppContext`, `globalThis` barrier in `apiStore.ts`, `AppVersionConfigInit` as render-gate, 19-namespace burst, inert helpers, `[BOOT]` logs).

## For Mastermind

- **AppVersionConfig type matches B2 — no challenge raised.** `src/lib/types/AppVersion.ts` declares `AppVersionConfig = { latestVersion: string; minSupportedVersion: string; forceUpdate: boolean; optionalUpdate: boolean }`. The brief asked to confirm before proceeding (since the type predated B2). It is correct. The Gate 2 logic uses `forceUpdate` and `optionalUpdate` directly, no shimming.

- **Documented decision — Gate 2 on resolved `undefined`:** The brief flagged the choice as worth raising rather than deciding unilaterally. The session adopts the brief's recommended posture: **resolved-`undefined` → advance, do NOT set `softUpdate`, do NOT divert to maintenance.**
  - Why: `getAppVersionConfig()` returns `undefined` from its own `catch` clause for any thrown service error the timer did not catch (a fast non-2xx, a network spike that resolved before 5s, etc.). The spec's Part 3 fail-open posture says the backend never errors on unknown platform / bad data; it returns a real DTO with `forceUpdate: false, optionalUpdate: false`. A client-side `undefined` is therefore an ambiguous swallowed error, not an authoritative "no update" signal and not a "backend down" signal. Treating it as "advance without blocking" matches the brief's rationale: the version gate must never block boot on an ambiguous swallowed failure, and a genuine backend-down is already covered by the timeout branch (which DOES divert to maintenance).
  - The non-timeout `catch` arm in Gate 2 takes the same posture (advance, no flag, no divert) as a defensive backstop for any future service variant where a non-`GateTimeoutError` reaches the gate. Today the service swallows its own errors, so the `catch` arm is structurally unreachable; if a later change unswallows the service, this arm preserves the documented "ambiguous failure does not block boot" posture without a code change.
  - If Mastermind disagrees and wants `undefined` (or non-timeout throws) to divert to maintenance, the change is two lines per arm. Raising it here rather than deciding unilaterally per the brief.

- **STEP-7 REMINDER (services must have exactly one caller after teardown).** After step 7 deletes the old orchestration, the two services this brief calls — `src/lib/services/maintenanceService.tsx` (`checkIfMaintenance`) and `src/lib/services/appVersionConfigService.tsx` (`getAppVersionConfig`) — must each have exactly one caller: the new boot machine (`bootStore.ts`). The step-7 brief must grep for both function names and confirm the old call sites in `AppContext.tsx` and `AppVersionConfigInit.tsx` (and any other surfaces) are removed. The services themselves are not edited by this brief; they remain stable.

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - **`withGateTimeout` as a separate module (`bootGate.ts`), not inlined in `bootStore.ts`.** Earned: the spec calls out the single backend-down rule as expressed in one place. A separate small module is the most direct expression — Gates 3 and 4 import the same wrapper in steps 5 / 6 with zero code duplication, and the wrapper can be unit-tested in isolation (3 dedicated tests; the gate tests don't have to re-prove the timeout mechanism, only that the gate routes on it correctly). The module is 49 lines including the doc-comment; it has one named const, one typed error, one function. No surface for accidental misuse.
    - **`GateTimeoutError` as a typed class (not a string-coded `Error` or a sentinel symbol).** Earned: the gate matches with `e instanceof GateTimeoutError`, which is the standard JS/TS pattern, plays well with the `try/catch` in Gate 2, and gives `.name === 'GateTimeoutError'` for any future logging without an opaque string compare. Cost: one class with one zero-arg constructor.
    - **Single-outer-Promise implementation of `withGateTimeout` (instead of `Promise.race`).** Earned: a `Promise.race` with a never-settling `fn` leaves the timeout-promise rejection pending in the microtask queue after the outer promise has resolved, which Node and vitest both flag as unhandled in some flows. The single-outer-Promise form `clearTimeout`s in both branches and never rejects an inner promise that nobody is observing. Five extra lines, fewer footguns.
  - Considered and rejected:
    - **Re-using `Promise.race` with a `setTimeout(() => reject(...), T)` inline.** Rejected — see above; cleaner inner timing under fake timers, no spurious unhandled-rejection signal.
    - **Routing the wrapper itself to `toMaintenance`.** Rejected — would put the divert policy inside the mechanism. The wrapper is mechanism (race + throw), the gate is policy (catch → decide). Keeping policy in the gate means Gates 3 and 4 can route their timeouts the same way without forking the wrapper.
    - **A `BootError` discriminated union with codes for each failure mode.** Rejected — exactly one mode matters at the gate boundary (timeout vs not), and the gate's `try/catch` already classifies. A union would either be premature (we'd carry codes through layers that don't use them) or invite each gate to invent its own codes. The simpler shape — one typed timeout error, everything else handled at the gate's `catch` — is sufficient through step 6.
    - **Auto-retry on timeout inside the wrapper.** Rejected — the spec is explicit: timeout → maintenance, not retry. The maintenance screen IS the retry surface for the user. Retry would burn the 5s budget multiple times before showing the user a state.
    - **Threading a `signal: AbortSignal` into `checkIfMaintenance` / `getAppVersionConfig` so the wrapper cancels the in-flight HTTP call on timeout.** Rejected this brief — the services don't accept signals today and the brief forbids modifying them ("Do NOT modify the service files' internal logic"). The timeout still correctly diverts to maintenance; the in-flight HTTP call resolves in the background and its result is discarded by the gate's already-passed `try/catch`. If a future brief adds signals to the services, the wrapper can be extended without changing the gate call sites.
    - **`forceUpdate` and `optionalUpdate` as exclusive types (`type VersionStatus = 'force' | 'optional' | 'none'`).** Rejected — the backend wire is two booleans; mapping them at the gate would require a transformer and would obscure the "force wins over optional" defensive check that this brief explicitly wants visible in the gate. The if/else ladder in the gate is three lines and reads exactly as the spec writes the rule.
  - Simplified or removed: nothing this session. Pure additions plus two stub bodies replaced in place.

- **Closure gate (verified):** No drafted `conventions.md` / `decisions.md` / `state.md` / `issues.md` edits this session. The boot-redesign feature is mid-implementation; the closing `state.md` flip and the `issues.md` `[BOOT]`-log close land at step 7, applied by Docs/QA on Mastermind's draft. No Expo-backlog row applies to this engineering brief.
