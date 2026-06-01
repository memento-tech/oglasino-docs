# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Make the running app boot on the new state machine (expo-boot-redesign Part 8 step 7, PHASE 7a — MOUNT only). After this brief the user-visible boot flow is driven by bootStore, not AppContext.bootstrap. The old code stays on disk and is NOT executed; phase 7b deletes it after Igor's live verification.

## Implemented

All five 7a changes (and nothing else):

1. **One mount effect (Invariant 1).** `app/_layout.tsx:40-42` — exactly one `useEffect` with empty deps calling `useBootStore.getState().start()` once on mount. No machine-owned value (`status`, `selectedBaseSite`, codes) appears in any boot-path effect's deps.
2. **Overlay switch (Invariant 2).** The overlay now reads `bootStatus = useBootStore((s) => s.status)` (was AppContext's `status`). Mapping: `booting`/`updating` → `LoadingOverlay`; `maintenance` → `BaseSiteSelector isMaintenance`; `intro-picker` → `BaseSiteSelector isMaintenance={false}`; `ready` → no overlay (portal Stack shows); `update-required` → no overlay branch (see change #5 / Option A). The `AppInit` mount gate was also switched from the old AppContext `status === 'ready'` to `bootStatus === 'ready'` so "ready" is bootStore-driven end-to-end (the portal-init side of the ready transition — flagged for Mastermind below).
3. **Picker re-wire (3 lines, per audit §5.2).** `src/components/init/BaseSiteSelector.tsx` lines 14/37/38 re-pointed from AppContext to bootStore: import → `useBootStore`; `baseSites` → `useBootStore((s) => s.baseSiteOverviews)`; `setBaseSiteForCode` → `useBootStore((s) => s.pickBaseSite)`. The JSX body, styling, copy, INTRO-translations fetch, and tap-handler shape are byte-for-byte unchanged. `BaseSiteOverviewDTO` carries `code`/`flagImageKey`/`labelKey` with the names the JSX reads (verified) — tsc clean, no JSX edit needed.
4. **Active-checker poll.** New `src/components/init/MaintenancePollInit.tsx` (mounted in `_layout.tsx:80`). While `status === 'maintenance'`, a 5s interval pings `checkIfMaintenance`; on a confirmed "off" it calls `reEnter()` (explicit, non-destructive re-entry). The effect reads `status` ONLY to start/stop the interval — it never writes status. Re-uses the existing `checkIfMaintenance` service (no new service). On error/timeout it does nothing and waits for the next interval (`checkIfMaintenance` swallows errors → returns `true`; a `try/catch` backstops any unexpected rejection). Tick logic extracted as the pure `pollMaintenanceOnce()` for unit-testing in the repo's `node` test env.
5. **Preserve the old path + disable the competing render-gate (Option A, Igor's call).** `AppContextProvider` stays mounted (its `bootstrap` and its own 5s poll fire harmlessly; the overlay no longer reads its status, so its status flips are invisible). `AppVersionConfigInit` stays mounted and remains the sole owner of the hard-update screen and soft-update modal (driven by its own version check, which mirrors Gate 2 and is a no-op in dev). Only its **competing render-gate was disabled**: removed the `preparing` state + its now-dead `initial` plumbing + the `<ActivityIndicator>` splash, so it renders children immediately instead of flashing a second loading screen over bootStore's. Its force/optional `Modal` logic is untouched.

**Decision asked of Igor (brief change #2/#5 "REPORT and ask before forking"):** the hard-update screen and soft-update modal are inline JSX inside `AppVersionConfigInit`, not separable components; the hard-update view needs `config.latestVersion` (bootStore exposes only `softUpdate: boolean`, no version string) and the soft modal carries 24h/per-version dismissal memory (bootStore's flag has none). Forking them would need a bootStore machine change (out of the five changes) + stripping AppVersionConfigInit's modals to avoid doubles. Igor chose **Option A**: keep AppVersionConfigInit as owner, disable only its render-gate, overlay does not render the two update states. Zero fork, zero machine change, smallest revert.

## Files touched

- app/_layout.tsx (+59 / −55 region; net mount effect + overlay switch + AppInit gate + poll mount + dropped unused AppContext status plumbing)
- src/components/init/BaseSiteSelector.tsx (3 lines: 14/37/38)
- src/components/internals/AppVersionConfigInit.tsx (−render-gate: `preparing`/`initial`/`ActivityIndicator`)
- src/components/init/MaintenancePollInit.tsx (new, +56)
- src/components/init/MaintenancePollInit.test.ts (new, +56)

(boot-redesign files are untracked on `new-expo-dev` — the feature is uncommitted per state.md; new files show as `??`.)

## Tests

- Ran: `npx vitest run` → **176 passed** (11 files, 0 failed). Baseline was 173; +3 new.
- New tests (`MaintenancePollInit.test.ts`, active-checker `pollMaintenanceOnce`): (1) maintenance OFF → `reEnter` called once; (2) still ON → no `reEnter`; (3) check rejects (network error/timeout) → no `reEnter`, no exception.
- `npx tsc --noEmit`: clean.
- `npx eslint` (touched paths): **0 errors, 2 warnings** — both accepted (see Conventions check).
- `npx expo start` not run (long-running dev server; would hang the session). tsc-clean + full suite green is the build proxy. Igor runs the live device test (below).

## Cleanup performed

- Removed the now-unused AppContext status plumbing from `_layout.tsx`: the local `status`/`setStatus` state, the `AppStateValue` import, and the `onStatusChanged={setStatus}` prop (overlay + AppInit now read `bootStatus`). Dropped the unused `locale` binding (`const [, setLocale]`) while keeping `setLocale`/`onLanguageChanged` wired (preserves the old-path re-render-on-language-change behavior).
- Removed `AppVersionConfigInit`'s `preparing` state, the dead `initial` param across `checkVersion` + its 3 call sites, and the orphaned `ActivityIndicator` import (render-gate carve-out, change #5).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change (the feature-close `decisions.md` entry is owed at the end of 7b, per spec Part 9 — not this session).
- state.md: **no change required this session.** The Expo-backlog `Version checksums` row flip to `mobile-stable` is explicitly deferred to feature close (after 7b teardown + Igor's live regression test), exactly as drafted in session 7's summary. No unstated config-file dependency this session.
- issues.md: no change. (The `[BOOT]`-log cleanup entry stays open — those logs are preserved in 7a and removed in 7b.)

## Obsoleted by this session

- The AppContext-driven overlay path in `_layout.tsx` (old `status` values `loading`/`select-base-site`) — replaced by the bootStore-driven overlay. The AppContext machinery itself is NOT deleted (7b owns deletion); only the overlay's reliance on it is severed.
- `AppVersionConfigInit`'s render-gate (the `preparing` splash) — deleted this session (carve-out). The component otherwise survives intact for 7b to evaluate.
- Nothing else. The Part 7 deletion list (AppContext.bootstrap, apiStore globalThis barrier, initI18n/loadAllNamespaces/loader.ts, AppVersionConfigInit as a gate, `[BOOT]` logs, `<RequireBaseSite>`) is **untouched and confirmed still on disk** — that is 7b's work.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no debug `console.log` added; all new/dead-symbol cleanup listed above. Two pre-existing lint warnings left (see "For Mastermind"), neither introduced by this session's logic.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): flagged in "For Mastermind".
- Part 6 (translations): confirmed — no new translation keys; backend-seeded keys consumed as-is.
- Other parts touched: Part 8 (architectural defaults) — re-uses the existing `/public/maintenance/active` service for the poll; no mobile-specific routes added. Spec Part 1 invariants re-confirmed below.

## Known gaps / TODOs

- **7b is next; the deletion list is deferred.** No Part 7 deletions performed. After Igor runs the boot-loop regression test on a real device and it passes, 7b deletes the old path. If it FAILS, do not proceed to 7b — find the 7a regression first (fast revert: re-point the overlay + AppInit gate back to AppContext `status`, restore the `onStatusChanged` prop, remove the mount effect and `<MaintenancePollInit />`, restore AppVersionConfigInit's `preparing` gate).
- Runtime maintenance-flip *into* maintenance from `ready` is NOT wired in 7a (the poll runs only while in `maintenance`). Per the brief this is intentional and not required (the spec only requires detecting maintenance clearing *off*); I agree with the brief's read. Cold boots into maintenance work. Flagged only for completeness.
- No `TODO`/`FIXME` added.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): one new module, `MaintenancePollInit.tsx` — the active-checker the spec Part 2 transition table promises; it has one mount site and is the cleanest home for the status-watching interval (the brief licenses "a sibling component dedicated to it"). The `pollMaintenanceOnce` pure-function extraction earns its place by making the maintenance-clear logic unit-testable in the repo's `node` env (no React renderer), matching how the bootStore family is tested. Relative imports in this file (not `@/`) match the boot-family/testability pattern (bootStore.ts et al.) — required because vitest has no `@/` alias.
  - Considered and rejected: (a) putting the poll's `useEffect` inline in `_layout.tsx` instead of a component — rejected; a dedicated component keeps the logic testable and the root layout readable. (b) a re-entrancy/in-flight guard on the poll tick — rejected; the 5s interval and `start()`'s synchronous `set({status:'booting'})` (which clears the interval via the effect dep before the next tick) make a double-fire impossible, and the existing AppContext poll has no such guard (match surrounding style). (c) a module-level `started` guard on the mount effect for React StrictMode double-mount — rejected to honor the brief's literal "one effect, calls start() once"; noted as a consideration below.
  - Simplified or removed: deleted AppVersionConfigInit's `preparing`/`initial` render-gate plumbing + orphaned import; removed the redundant AppContext `status` mirror from `_layout.tsx`.
- **Three invariants — re-confirmed:**
  1. ONE effect with `[]` deps (`_layout.tsx:40-42`) is the only effect that calls `start()` directly. The poll effect (`MaintenancePollInit`) reads `status` in its deps only to start/stop the interval and calls `reEnter()` (the spec-sanctioned explicit re-entry) — never `start()` directly, never a status write. The pre-existing Android nav-bar effect is not in the boot path (no machine reference).
  2. `status` is written ONLY by bootStore's `toX`/`set` calls. `_layout.tsx` and the poll only read it. Confirmed by grep.
  3. Re-entry destroys nothing: `start()` (`bootStore.ts:112-118`) only `set({status:'booting'})` and runs gates; it nulls no resolved slot. Proven for the maintenance-clear case by the new poll test (`reEnter` called) plus the existing Invariant-3 test in `bootStore.test.ts`.
- **bootStore + new module import zero auth/chat stores — confirmed.** Grep over `bootStore.ts`, `bootFreshness.ts`, `checksumStorage.ts`, `versionsService.ts`, `MaintenancePollInit.tsx` for `authStore`/`chatStore`/`useAuthStore`/`useChatStore` returns only the doc-comment in `bootStore.ts` describing the defense — no imports.
- **In-scope decision to flag: `AppInit` mount gate switched to `bootStatus === 'ready'`.** The brief enumerates "the overlay" in change #2; `AppInit` (portal init — push notifications, auth listener, chat init) is gated separately. I switched it from old AppContext `status` to `bootStatus` so the ready state is bootStore-driven end-to-end (the TASK line: "the user-visible boot flow is driven by bootStore"). Leaving it on AppContext would split-brain the ready transition. Flagging because it's adjacent to, not literally inside, change #2's wording.
- **Two maintenance polls now run concurrently (old + new).** AppContext's own 5s `setInterval` poll (`AppContext.tsx:161-180`) still fires alongside the new `MaintenancePollInit` — both hit `/public/maintenance/active` every 5s while the app is up. This is the "preserve old path" cost in 7a; 7b deletes AppContext.bootstrap and its poll. Severity low (redundant network call, no user-visible effect). Not fixed — out of 7a scope.
- **Adjacent observations (Part 4b):**
  - `src/components/internals/AppVersionConfigInit.tsx:20` — `ONE_MIN = 60 * 1000` is a pre-existing unused constant (zero references; predates this session). Low severity (dead constant). **Not removed** — the brief's hard rule forbids editing AppVersionConfigInit beyond the render-gate carve-out; recommend folding into 7b cleanup.
  - `MaintenancePollInit.test.ts:17` — `import/first` warning on the import-under-test placed after `vi.mock`. Identical to the accepted vitest mock-hoisting pattern in `bootStore.test.ts` (session 7); kept for consistency. `vi.mock` is hoisted regardless, so behavior is correct.
  - React StrictMode: if the app tree is wrapped in `StrictMode` (dev only), the empty-deps mount effect fires twice, causing two `start()` passes on dev cold-start. Gates are read-only until they write, so this is a benign dev artifact, but if Igor sees a double-boot in dev logs, a module-level `started` guard (or moving `start()` behind a ref) is the fix. Left out to honor the brief's literal "one effect, start() once"; flagging so it isn't mistaken for a regression.
- **Boot-loop regression test — steps for Igor to run on a real device/simulator AFTER commit (spec Part 8 / brief):**
  1. Ensure a base site is already stored (or pick one once to store it).
  2. Cold start the app → it should reach the portal (`ready`): splash (`booting`) → `ready` (stored, current site) with **zero** translation/catalog fetches for a returning user (Gate 4 checksum match).
  3. With the app at `ready`, toggle backend maintenance **ON** (admin). *Note:* 7a does not detect maintenance flipping on while already `ready` (the poll runs only in `maintenance`) — to enter `maintenance` you must cold-restart with maintenance on, OR start from a `maintenance` boot. The clear-detection half of the loop is what 7a wires.
  4. From `maintenance`, toggle backend maintenance **OFF**. Within ~5s the active-checker poll detects it → `reEnter()` → the machine runs the gates → returns to `ready`.
  5. **PASS criteria:** returns to `ready` WITHOUT clearing the stored base site AND with exactly ONE re-entry (no boot→loading→boot loop, no extra pass). If it loops or clears the base site, an invariant was violated — do NOT proceed to 7b; report back.
- **Next session is 7b (teardown).** Only after the regression test passes. 7b deletes the Part 7 list, removes `<RequireBaseSite>` (review), removes the `[BOOT]` logs, and applies the `state.md` Expo-backlog `Version checksums` flip + the `decisions.md` feature-close entry (Docs/QA).
