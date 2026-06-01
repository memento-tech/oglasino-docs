# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Build the standalone boot state-machine store — the skeleton only. The four gates are STUBS in this brief. Real maintenance/version/base-site/freshness logic lands in later briefs (steps 4–6). This brief proves the machine's shape and the three loop-prevention invariants in isolation, with stubbed gates.

## Implemented

- New file `src/lib/store/bootStore.ts`: Zustand store carrying `status` (initial `'booting'`), `softUpdate` (initial `false`), and the three resolved-data slots `selectedBaseSite` / `language` / `config` (all initial `null`). Slot types reuse existing repo types (`BaseSiteDTO` from `lib/types/catalog/BaseSiteDTO.ts`, `LanguageDTO` from `lib/types/catalog/LanguageDTO.ts`, `ConfigMap` from `lib/services/configurationService.tsx`).
- Single entry point `start()` re-enters the machine non-destructively: it sets `status` back to `'booting'` and kicks `runMaintenanceGate()`, but does NOT touch the resolved slots or `softUpdate`. `reEnter()` is exposed as a named alias for the maintenance-clear handler (later brief) to call — behaviourally identical to `start()`.
- Four gate functions as STUBS (`runMaintenanceGate`, `runVersionGate`, `runBaseSiteGate`, `runFreshnessGate`), each ending by calling the next via `get().runNextGate()`. `runFreshnessGate` ends by setting `status: 'ready'`. Each stub carries a comment naming the later brief that fills in the real body (step 4 for maintenance + version; step 5 for base-site; step 6 for freshness) and the diverting destinations the real body will use.
- Four transition helpers `toMaintenance` / `toUpdateRequired` / `toIntroPicker` / `toUpdating` — pure `set({ status })` actions. Built now so later briefs and the test suite share the same surface for diversion.
- The three loop-prevention invariants (spec Part 1) documented as a top-of-file contract: (1) one effect with empty deps, (2) machine writes status / view reads status — no internal subscription or reactive trigger inside the store, (3) re-entry destroys nothing.
- New file `src/lib/store/bootStore.test.ts`: 17 tests covering initial state, happy-path reach-ready, `reEnter()` equivalence, gate ordering (`maintenance → version → baseSite → freshness`), each `toX()` transition, two re-entry non-destructive tests for Invariant 3 (one via `start()`, one via `reEnter()` with seeded `selectedBaseSite` / `language` / `config` / `softUpdate`), four per-gate diversion tests (each gate can divert via `toX()` and stops the sequence with later gates not called), and three ordering tests proving no gate fires before its predecessor passes.

## Files touched

- src/lib/store/bootStore.ts (+121 / −0, new)
- src/lib/store/bootStore.test.ts (+215 / −0, new)

No other source files modified. Old boot path (`AppContext.tsx`, `apiStore.ts`, `_layout.tsx`, `AppVersionConfigInit.tsx`, `src/i18n/loader.ts`) untouched per the brief's "build alongside" rule.

## Tests

- Ran: `npx vitest run src/lib/store/bootStore.test.ts`
- Result: 17 passed, 0 failed (one file).
- Ran: `npm test` (full suite).
- Result: 126 passed, 0 failed (8 files). Up from the 109-test baseline by exactly the 17 new tests.
- Ran: `npx tsc --noEmit`. Clean (zero errors).
- Ran: `npx expo lint`. 72 pre-existing warnings repo-wide; zero warnings touching the two new files.
- `npx expo-doctor` not run — no dependency changes this session (conventions Part 4: "`npx expo-doctor` when relevant").

## Cleanup performed

- None needed. No prior boot-store scaffolding existed; the new files are pure additions. No commented-out code, no debug `console.log`, no TODO/FIXME, no unused imports, no dead helpers.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change. The expo-boot-redesign feature is mid-implementation; the feature-close `state.md` flip is owed by Docs/QA at feature completion per the spec's Definition of Done, not per-session.
- issues.md: no change. The pre-existing `[BOOT]` instrumentation entry remains open and will be closed during step 7 (portal wiring + teardown), not this brief.

## Obsoleted by this session

- Nothing. Skeleton-only addition; the old boot path stays in place and is still the live boot, per the brief's explicit out-of-scope list (no teardown until step 7).

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no unused imports/variables/functions, no `console.log`, no TODO/FIXME, no new files unreferenced. `tsc` clean; lint clean on touched paths; new tests pass; existing suite green at 126.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): N/A this session. No new observations beyond what the Phase-2 audit (`.agent/audit-expo-boot-redesign.md`) already catalogues and what `issues.md` already tracks.
- Part 6 (translations): N/A this session — no user-facing strings.
- Other parts touched: spec Parts 1, 2, 5 — confirmed. The three invariants are documented and tested (Invariant 1: `start()` is the single entry point designed for one `useEffect` with `[]` deps — wiring is step 7; Invariant 2: structural — there is no effect, no internal subscription, and no reactive trigger anywhere in `bootStore.ts`, verified by reading the file end-to-end; Invariant 3: tested by the two non-destructive re-entry tests). Structural defense — zero imports of `authStore` or any chat store in `bootStore.ts` and `bootStore.test.ts` (verified via grep; the only matches are the doc-comment references at lines 31–32 of `bootStore.ts` that name what is NOT imported).

## Known gaps / TODOs

- Stubbed gates only. Real maintenance + version logic lands in step 4; real base-site gate in step 5; real freshness gate in step 6. Each stub's body is replaced without changing the advancement contract (each gate ends by calling the next via `get().runX()`).
- No timeout logic. The single backend-down rule (any gate that cannot reach the backend within its 5s timeout → `toMaintenance`) lands with the real gate bodies in steps 4–6. `toMaintenance` is the destination when timeouts land — built now, exercised by the per-gate-divert tests.
- No mount-effect wiring into the app tree. Step 7 (portal wiring) mounts the single `useEffect` with empty deps that calls `start()`.
- No softUpdate UI wiring. The `softUpdate` flag is built into the store with the correct initial value (`false`); the modal that consumes it ships in a later brief.
- `<RequireBaseSite>` and the old boot path remain live. Step 7 reviews `<RequireBaseSite>` against the `portal-mounts-only-in-ready` invariant and tears down the old path (Promise.all, globalThis barrier, AppVersionConfigInit as render-gate, 19-namespace burst, inert helpers, `[BOOT]` logs).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - **Gates as store methods (not free functions).** Earned: the brief's tests require per-gate divert proofs (override `runVersionGate` and verify later gates are not called). Methods sit on the store so `useBootStore.setState({ runVersionGate: vi.fn() })` works directly; free functions would need a mock harness or a parallel registry. The same dispatch style also lets later briefs replace the stub body in place without touching the call site.
    - **`get().runNextGate()` call style inside each gate.** Earned: identical surface for stubs and real async gates. When step 4 replaces `runMaintenanceGate` with `async () => { const r = await fetch(...); if (timeout) get().toMaintenance(); else get().runVersionGate(); }`, the call site at `start()` and the inter-gate calls do not change shape.
    - **`BootStatus` exported as a named string-literal union type.** Earned: the view (step 7) switches on this discriminated union; centralizing the literals prevents drift between the machine and the view and gives `tsc` a check-all-arms guarantee at the switch.
  - Considered and rejected:
    - **A `toReady()` transition helper.** Rejected: the brief lists exactly four diverting transitions (`toMaintenance` / `toUpdateRequired` / `toIntroPicker` / `toUpdating`). Reaching `ready` is the normal end of `runFreshnessGate` and the only legitimate writer of `'ready'`; promoting it to a public helper would invite calling it from outside the freshness gate, which would invite skipping the gate sequence.
    - **A `reset()` / `clear()` action on the store.** Rejected: Invariant 3 forbids destructive entry from production code. Tests use `useBootStore.setState(initialState, true)` in `beforeEach`, which is the test-scope equivalent of a reset without exposing a destructive surface to the runtime.
    - **Async / `Promise<void>` gate signatures up front.** Rejected: the stubs are sync and the brief is explicit that timeouts land in steps 4–6. Adopting async now would either require fake-resolving in every test or change the call-flow shape twice (once now, once in step 4). The `get().runX()` call style does not change when X becomes async.
    - **Selector helpers / `useShallow` imports in this file.** Rejected: no view consumers wire in this brief. Selectors and shallow subscriptions live with the consumer per the Φ3 convention, not in the store module.
    - **A `set` of the resolved slots inside `start()` to their existing values (no-op write).** Rejected: a no-op write is dead code that drifts when slots are added. The test asserts identity-equality on the seeded references after `start()`, which is the actual invariant.
  - Simplified or removed: nothing. Pure addition. The old boot path is untouched per the brief's hard rules; its teardown is step 7.

- **Brief vs reality:** nothing material to flag. The brief's resolved-slot type names (`BaseSiteDTO`, `LanguageDTO`, `ConfigMap`) map cleanly to existing repo types and were imported from their existing paths. The brief's store path (`src/lib/store/bootStore.ts`) matches the sibling-file naming convention — co-located with `authStore.ts` (kebab-noun) and `useFavoritesStore.ts` (use-prefixed); the file is `bootStore.ts` to match `authStore.ts`, and the exported hook is `useBootStore` to match the rest. No challenge to the brief warranted.

- **Confirmation per the brief's structural-defense clause:** `bootStore.ts` imports `zustand`, `BaseSiteDTO`, `LanguageDTO`, and `ConfigMap` — and nothing else. Grep against `authStore|useActiveChatStore|useChatListStore|useChatNavStore|useChatBlockStore` returned zero import lines (the only two grep hits are the doc-comment lines naming what is NOT imported). The boot store is a leaf w.r.t. the `authStore ↔ chat-store` require cycle (Expo audit §5.1) and cannot be caught in it.

- **Step-7 reminder for the resolving brief:** the spec's Definition of Done item "everything in Part 7 is deleted, including the `[BOOT]` instrumentation" closes the standing `issues.md` 2026-05-28 entry on `src/lib/config/api.ts`'s request interceptor. This brief did NOT touch that file (it is the old path; this brief built alongside).

- **Closure gate (verified):** No drafted `conventions.md` / `decisions.md` / `state.md` / `issues.md` edits this session. The boot-redesign feature is mid-implementation; the closing `state.md` flip and `decisions.md` entry land at feature close per the spec's Definition of Done, applied by Docs/QA on Mastermind's draft. No Expo-backlog row applies (this is in-progress engineering on the boot redesign, not a mobile adoption of a `web-stable`/`shipped` feature).
