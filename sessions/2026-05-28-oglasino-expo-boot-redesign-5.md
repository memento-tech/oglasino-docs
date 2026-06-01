# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Step 5 of `features/expo-boot-redesign.md` Part 8 — replace the Gate 3 STUB (`runBaseSiteGate`) in `src/lib/store/bootStore.ts` with real logic, and add the `pickBaseSite(code)` action the picker will eventually call. Gate 4 (freshness) REMAINS a stub; the machine still reaches `ready` through it on the happy path. The 3-line picker re-wire in `BaseSiteSelector.tsx` was DEFERRED to step 7 per the transitional-risk confirm (see below).

## Implemented

- `src/lib/init/baseSitesService.ts` — two new thin service functions added alongside `fetchBaseSites`:
  - `fetchBaseSiteOverviews()` → `GET /public/baseSite/overviews` → `BaseSiteOverviewDTO[]`. `_bootstrap: true`. THROWS on non-2xx and lets axios's native throw propagate on transport failure. No swallow.
  - `fetchBaseSiteByCode(code)` → `GET /public/baseSite/${encodeURIComponent(code)}` → `BaseSiteDTO`. `_bootstrap: true`. Same throw-on-failure semantics. `encodeURIComponent` is defence against arbitrary code strings reaching the URL even though the picker only invokes with backend-supplied codes — it costs nothing.
  - File-leading comment explains why the new pair throws while the legacy `fetchBaseSites` (about to be torn out in step 7) swallows: the gate's `try/catch` + `withGateTimeout` is the one place the spec's "fails/times out → maintenance" rule lives, so the inner services must propagate failures to it. Both rules in the spec table (Part 2: `intro-picker | GET /baseSite/overviews fails/times out | maintenance`) are now uniformly served by the gate.
- `src/lib/store/bootStore.ts` — Gate 3 (`runBaseSiteGate`) replaced. Now `async`. Body:
  - Reads `getStoredBaseSite()` first (the "honest first paint" moment per spec Part 5).
  - **Stored-site path:** reads `getStoredLanguage()`; picks `lang` = stored language if present AND in `storedSite.allowedLanguages`, else `storedSite.defaultLanguage` (mirrors the legacy `AppContext.setBaseSiteForCode` language-pick rule at `AppContext.tsx:193-195`). `set({ selectedBaseSite, language })`, `setCodes(storedSite.code, lang.code)`, advance to `runFreshnessGate()`. No re-fetch: the stored DTO is trusted until Gate 4 proves it stale (Invariant 3 + spec freshness ownership). No `initI18n` — translation loading is Gate 4 territory (step 6), and the OLD path's `initI18n` still drives translations end-to-end on the running app until step 7 tears down both paths together.
  - **No-stored-site path:** `withGateTimeout(() => fetchBaseSiteOverviews())`. On success: store the list in the new `baseSiteOverviews` slot, `toIntroPicker()`. On ANY throw (timeout or non-timeout): `toMaintenance()`. Empty picker is never shown — the spec's `fails/times out → maintenance` rule covers both arms with a single `catch`.
- `src/lib/store/bootStore.ts` — new action `pickBaseSite(code)`. Body:
  - Defensive `find` against `baseSiteOverviews` for the tapped code. If missing, no-op (the picker shouldn't surface an unknown code; this is belt-and-braces).
  - `withGateTimeout(() => fetchBaseSiteByCode(code))`. On ANY throw: `toMaintenance()`, return — nothing persisted, nothing advanced.
  - Language pick mirrors `AppContext.setBaseSiteForCode` lines 193-195: previously-selected `language` survives if it's in `fullDto.allowedLanguages`; else fall back to `fullDto.defaultLanguage`. This is the same rule used by Gate 3 stored-site path, just with the picker's previously-selected `language` slot replacing AsyncStorage's stored language.
  - `set({ selectedBaseSite: fullDto, language })`, `setCodes(fullDto.code, lang.code)`, `await setStoredBaseSite(fullDto)`, `await setStoredLanguage(lang)`, `runFreshnessGate()`.
  - No `initI18n` (Gate 4 / step 6 owns translation loading on the new path). No checksum co-storage (Gate 4 / step 6 owns checksums per spec Part 5).
- `src/lib/store/bootStore.ts` — `BootStore` interface updates: new `baseSiteOverviews: BaseSiteOverviewDTO[]` slot (init `[]`), `runBaseSiteGate` return type changed `void → Promise<void>`, new `pickBaseSite: (code: string) => Promise<void>` action. Module imports `getStoredLanguage`/`setStoredLanguage` from `../../i18n/storage`, `setCodes` from `../config/apiStore`, and the four base-site service functions from `../init/baseSitesService`. The leading docstring's "step" line updated to reflect Gates 1/2/3 real, Gate 4 still stubs. No new imports of any auth-store or chat-store module (structural defense Part 1 preserved).
- `src/lib/store/bootStore.test.ts` — extensive test-side updates:
  - New mocks via `vi.hoisted` + `vi.mock` for `baseSitesService` (4 functions), `i18n/storage` (2 functions), and `apiStore.setCodes`. Default Gate 3 path is now stored-site (mock returns `DEFAULT_STORED_SITE = buildSiteDTO()`); legacy "no-stored-site" branch is opted into per-test via `mockGetStoredBaseSite.mockResolvedValue(null)`. This default keeps the original transition-table happy-path tests reaching `'ready'` via stored-site → freshness (stub) — the call-flow shape proven in step 3 survives unchanged at the assertion level.
  - `buildSiteDTO(over)` and `buildOverview(over)` factory helpers so each test produces full DTOs without 12 lines of inline shaping.
  - Invariant 3 tests rewritten so the seeded slot identities match what storage returns (identity-equal across re-entry, as the original test asserted with `.toBe`). The thin `seededBaseSite`/`seededLanguage` placeholders are gone; the test now persists a real `buildSiteDTO()` reference into the mock, runs the gate to land it in slots, layers `softUpdate` + `seededConfig` on the slots no gate owns yet, then re-enters via `start()` / `reEnter()` and verifies all four resolved-or-seeded fields survive. Invariant 3 is asserted against the **real** Gates 1+2+3 stored-site path.
  - Existing override stubs for `runBaseSiteGate` updated `() → async ()` to match the new return type (call sites at tests for ordering, stub-divert, and the gates-in-order trace).
  - New `bootStore — Gate 3 (base-site), real logic` block (6 tests):
    - stored-site no-stored-language path: slots populated from stored DTO, language falls back to `defaultLanguage`, `setCodes` called, advances to freshness (overviews fetch NOT invoked);
    - stored-site stored-language present and in `allowedLanguages`: uses stored language;
    - stored-site stored-language NOT in `allowedLanguages`: falls back to `defaultLanguage`;
    - no-stored-site happy path: overviews fetched into `baseSiteOverviews` slot, `toIntroPicker`, `setCodes`/freshness NOT called, `selectedBaseSite` stays null;
    - no-stored-site overviews fetch rejects (non-timeout) → `toMaintenance`;
    - no-stored-site overviews fetch hangs → 5s timeout → `toMaintenance` (fake timers, attach-handler-before-advance to avoid unhandled-rejection signal, same pattern step 4 used).
  - New `bootStore — pickBaseSite action` block (6 tests):
    - happy path: full DTO fetched, slots populated, `setCodes`, `setStoredBaseSite`/`setStoredLanguage` called with the right values, freshness invoked;
    - language preservation: previously-selected language is in `allowedLanguages` → preserved;
    - language fallback: previously-selected language NOT in `allowedLanguages` → falls back to `defaultLanguage`;
    - unknown code (not in `baseSiteOverviews`) is a no-op: no fetch, no persist, no `setCodes`, slots untouched;
    - full-DTO fetch rejects (non-timeout) → `toMaintenance`, nothing persisted, freshness NOT called;
    - full-DTO fetch hangs → 5s timeout → `toMaintenance`, nothing persisted, freshness NOT called (fake timers).

## Files touched

- src/lib/init/baseSitesService.ts (+25 / −1) — two new functions; one pre-existing `_bootstrap: true` argument on `fetchBaseSites` was already in the working tree from prior session, unmodified here.
- src/lib/store/bootStore.ts (modified: +6 imports, new `baseSiteOverviews` slot + init, `runBaseSiteGate` signature `void → Promise<void>`, Gate 3 stub body replaced with real body, new `pickBaseSite` action, two docstring lines tightened to reflect Gate 3 now real)
- src/lib/store/bootStore.test.ts (modified: 7 new mock factories added to the `vi.hoisted` block + 3 new `vi.mock(...)` calls; `beforeEach` reset and default-seeded all new mocks; helpers `buildSiteDTO`/`buildOverview`/`DEFAULT_STORED_SITE` added; Invariant 3 tests rewritten; 4 sync `runBaseSiteGate` override literals upgraded to `async`; two new top-level `describe` blocks added for Gate 3 real logic and `pickBaseSite`)

No other source files modified. Old boot path (`AppContext.tsx`, `apiStore.ts`, `_layout.tsx`, `AppVersionConfigInit.tsx`, `src/i18n/loader.ts`, `BaseSiteSelector.tsx`) untouched per the "build alongside" rule and per the deferred-re-wire decision.

## Tests

- Ran: `npx vitest run src/lib/store/bootGate.test.ts src/lib/store/bootStore.test.ts`. Result: 41 passed, 0 failed (2 files) — 38 in `bootStore.test.ts` (26 carried from step 4 + 12 new: 6 Gate 3 + 6 `pickBaseSite`) and 3 in `bootGate.test.ts`.
- Ran: `npm test` (full suite). Result: 150 passed, 0 failed (9 files). Up from the step-4 baseline of 138 by exactly the 12 new tests.
- Ran: `npx tsc --noEmit`. Clean (zero errors).
- Ran: `npx expo lint -- --quiet src/lib/store/bootStore.ts src/lib/store/bootStore.test.ts src/lib/init/baseSitesService.ts`. Zero warnings on touched paths.
- `npx expo-doctor` not run — no dependency changes this session.
- No real network calls in tests; every backend-touching service is mocked via `vi.hoisted` + `vi.mock`. Fake timers (`vi.useFakeTimers()`) for every timeout case; no test waits 5s of wall time.

## Cleanup performed

- Removed the now-unused `seededBaseSite` / `seededLanguage` test constants (the `as unknown as BaseSiteDTO` placeholders) when the Invariant 3 tests stopped needing them — the rewrites use full `buildSiteDTO()` references seeded into the storage mock instead. `seededConfig` retained because the rewritten Invariant 3 tests still use it for the slot no gate owns.
- No commented-out code, no `console.log`, no TODO/FIXME, no unused imports, no dead helpers. The Gate 3 stub body fully replaced; no orphaned stub code.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change. The "throw-on-failure" posture in the two new boot-redesign services is a brief-internal implementation choice grounded in the existing spec rule (Part 2: "fails/times out → maintenance"); not a new project-wide decision.
- state.md: no change. The expo-boot-redesign feature is mid-implementation (this is step 5 of 7 in Part 8 of the spec). The Φ3 "verifying" status referenced in state.md for the boot-loading issue stays unchanged; it flips when step 7 ships and the old path is torn out.
- issues.md: no change. The pre-existing `[BOOT]` instrumentation entry remains open and is owed to step 7 (portal wiring + teardown).
- No Expo backlog row applies. This is in-progress engineering on the boot redesign, not a mobile adoption of a `web-stable` / `shipped` feature.

## Obsoleted by this session

- The Gate 3 STUB body of `runBaseSiteGate` (a one-liner advancing straight to freshness). Replaced. The stub comment naming "step 5" as the brief that fills it in is gone; the new body and its docstring stand on their own.
- The placeholder `seededBaseSite` / `seededLanguage` thin-mock test constants. Replaced by full DTOs via `buildSiteDTO()` so identity-equality assertions across Gate 3 re-entry hold against a realistic-shape DTO.
- The original "Invariant 3 — re-entry destroys nothing (with real gates 1+2)" describe-block phrasing. Now reads "(with real gates 1+2+3 stored-site path)" because Gate 3 actually runs during the test.
- Gate 4 (`runFreshnessGate`) STUB remains in place — step 6 lands the real body.
- The OLD path (`AppContext.tsx` `setBaseSiteForCode`, the `BaseSiteSelector.tsx` AppContext-driven data-in / action-out, `apiStore.ts` `globalThis`-pinned barrier, `initI18n` on pick, `AppVersionConfigInit` as a render-gate, the 19-namespace burst, the inert version/cache helpers, the no-op `setStoredBaseSite` re-write, the `[BOOT]` instrumentation in `api.ts`) — STILL LIVE. Step 7 owns the teardown.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no unused imports/variables/functions, no `console.log`, no TODO/FIXME. `tsc` clean. Lint clean on the three touched paths. New tests pass; full suite green at 150 (up by exactly 12 over the step-4 baseline of 138). Two now-unused test constants removed in the same session.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one observation surfaced — see "For Mastermind." It is `BaseSiteSelector.tsx`'s `INTRO_FALLBACK` hardcoded-Serbian object (low) which the seam audit already flagged as out-of-scope for the seam analysis but worth pinning here since the picker re-wire will inherit it in step 7.
- Part 6 (translations): N/A this session. No user-facing strings added or changed. The picker's existing translation pathway (`fetchNamespace(INTRO, 'sr')` + the hardcoded `INTRO_FALLBACK`) is untouched per the UI constraint.
- Spec Parts 1, 2, 5 — confirmed:
  - **Invariant 1** (one effect, empty deps): structural — this module still has no effect and no internal subscription. `pickBaseSite` is an action invoked by user input (the picker's tap), not a reactive trigger. Mount-effect wiring is step 7.
  - **Invariant 2** (machine writes status; view reads it): every `status` write in the file remains owned by `toX()` helpers or `runFreshnessGate`'s final `set({ status: 'ready' })`. Gate 3 calls `toMaintenance`/`toIntroPicker`; `pickBaseSite` calls `toMaintenance` on failure. No reactive trigger, no internal subscription added.
  - **Invariant 3** (re-entry destroys nothing): `start()` still does NOT touch `selectedBaseSite`/`language`/`config`/`softUpdate`/`baseSiteOverviews` on entry. Gate 3's stored-site path REPOPULATES `selectedBaseSite`/`language` from storage on every entry — the same references survive because storage returns the same persisted JSON-parsed shape each pass. The Invariant 3 tests now assert this end-to-end through real Gates 1+2+3, with `softUpdate` and `config` (slots no gate owns yet) layered on after the first pass and surviving the second.
  - **Spec Part 2 single backend-down rule**: every backend call from this module — `checkIfMaintenance` (Gate 1), `getAppVersionConfig` (Gate 2), `fetchBaseSiteOverviews` (Gate 3), `fetchBaseSiteByCode` (`pickBaseSite`) — wraps in `withGateTimeout`. Every `catch` arm in these four sites routes timeout → `toMaintenance`; Gate 3 and `pickBaseSite` extend that to any non-timeout throw too (per spec table's `fails/times out → maintenance`). Gate 2 keeps its narrower posture (only timeout → maintenance) per the documented Part 3 fail-open rule.
  - **Spec Part 5 storage model**: `pickBaseSite` writes `setStoredBaseSite(fullDto)` + `setStoredLanguage(lang)` together; Gate 3 reads `getStoredBaseSite` + `getStoredLanguage` together. Checksum co-storage (per Part 5 table) is NOT in this brief — Gate 4 owns checksums, and Gate 4 is step 6. The Part 5 "honest first paint" instant is preserved: `booting` only owns the async storage read in Gate 3; no fetch, no logic beyond it. The `_bootstrap: true` flag on the two new service functions guarantees they don't queue behind `waitForBootstrap()` (apiStore.ts) — boot calls bypass the barrier on the old transport, the barrier itself is removed in step 7.
  - **Structural defense (Part 1)**: `bootStore.ts` now imports — beyond step-4's set — `getStoredLanguage`/`setStoredLanguage` from `../../i18n/storage`, `setCodes` from `../config/apiStore`, four functions from `../init/baseSitesService`, and the `BaseSiteOverviewDTO` type. None of these is an auth-store or chat-store module. Verified by `grep` — `bootStore.ts` and `bootGate.ts` together contain ZERO imports matching `authStore|useActiveChatStore|useChatListStore|useChatNavStore|useChatBlockStore` (only doc-comment line `bootStore.ts:43-45` names what is NOT imported). The boot store remains a leaf w.r.t. the `authStore ↔ chat-store` require cycle (Expo audit §5.1).

## Known gaps / TODOs

- **`BaseSiteSelector.tsx` re-wire DEFERRED to step 7** (per pre-implementation confirm with Igor). The picker still reads its list and tap handler from `useAppState()` / `useAppActions()` (lines 14, 37, 38). The bootStore-side machinery (`baseSiteOverviews` slot, `pickBaseSite` action) is fully built and tested but unmounted to the running app. Step 7 owns the 3-line swap as part of the path switchover, so the picker is never broken-on-old-path between briefs.
- **`runFreshnessGate` remains a stub** (advances straight to `ready`). Real body lands in step 6 — checksum compare, slice-only refetch of stale catalog/translation namespaces, checksum co-storage. Spec Part 4 + Part 5 govern the design.
- **`initI18n` / translation burst on the new path** — not fired by Gate 3 or `pickBaseSite`. Step 6 / Gate 4 owns translation loading on the new path; until then, the OLD path's `AppContext.setBaseSiteForCode` is what drives translations end-to-end on the running app. New path's `pickBaseSite` does NOT load translations, which is why step 7 cannot mount the new path until step 6 (gate 4) plus a translation-load wiring lands.
- **Mount-effect wiring (step 7).** Single `useEffect` with empty deps that calls `start()`. Out of scope here.
- **Active-checker / maintenance-poll (step 7).** The maintenance-clear path that re-enters via `reEnter()` still lives in step 7's wiring; `reEnter()` is exposed for that consumer.
- **Old boot path remains live.** Step 7 reviews `<RequireBaseSite>` against the `portal-mounts-only-in-ready` invariant and tears down the old orchestration (Promise.all in `AppContext`, `globalThis` barrier in `apiStore.ts`, `AppVersionConfigInit` as render-gate, 19-namespace burst, inert helpers, `[BOOT]` logs).

## For Mastermind

- **Transitional-risk decision recorded (option `a`).** Per pre-implementation confirm: the 3-line `BaseSiteSelector.tsx` re-wire is DEFERRED to step 7. Rationale (as stated in the brief): re-wiring now would leave the running app with a picker that reads from a bootStore slot the old path never populates, breaking the picker on the old path between this brief and step 7. Building only the machine side (Gate 3 + `pickBaseSite` + 12 tests + 2 services) keeps the app whole today and gives step 7 a clean one-shot path switchover. Test coverage is identical either way — the picker is exercised by unit tests, not the running UI.

- **Service-function decision recorded.** Two new thin functions (`fetchBaseSiteOverviews`, `fetchBaseSiteByCode`) added to `src/lib/init/baseSitesService.ts` per pre-implementation confirm. Throw-on-failure semantics chosen (departing from `fetchBaseSites`'s legacy swallow) so the gate's `try/catch` + `withGateTimeout` is the one place the spec's "fails/times out → maintenance" rule lives. File-leading comment documents the intentional inconsistency with `fetchBaseSites` (which gets deleted in step 7).

- **STEP-7 REMINDERS — consolidated checklist (additive over step 4's list).** After step 7 deletes the old orchestration, the engineer doing step 7 must:
  1. **Services one-caller verification.** Grep that the four services this brief depends on each have exactly one caller, the bootStore: `checkIfMaintenance`, `getAppVersionConfig`, `fetchBaseSiteOverviews`, `fetchBaseSiteByCode`, `getStoredBaseSite`, `setStoredBaseSite`, `getStoredLanguage`, `setStoredLanguage`. The legacy `fetchBaseSites` (heavy `/baseSite/details`) should have ZERO callers and be deleted. The legacy `setBaseSiteForCode` in `AppContext.tsx` (and its `initI18n` call) is also deleted in step 7.
  2. **Picker re-wire (the 3 lines).** `BaseSiteSelector.tsx` line 14 (import), line 37 (data-in: `baseSites` → `baseSiteOverviews`), line 38 (action-out: `setBaseSiteForCode` → `pickBaseSite`). The JSX body lines (76-140) DO NOT change — the picker reads `baseSite.code/flagImageKey/labelKey` from each item, and `BaseSiteOverviewDTO` carries all three with the same names (`src/lib/types/catalog/BaseSiteOverviewDTO.ts`). Confirmed.
  3. **`apiStore.ts` teardown.** The `globalThis`-pinned barrier + `waitForBootstrap()` go away. `setCodes` (called by Gate 3 stored-site path and `pickBaseSite`) becomes obsolete with the barrier; step 7 decides whether `bootStore` exposes `baseSiteCode/langCode` selectors for the `X-Base-Site` / `X-Lang` interceptor reads OR whether the interceptor reads from `bootStore` directly. The current `api.ts:43-65` interceptor reads from `apiStore.getCurrentBaseSiteCode()` / `getCurrentLangCode()` — those need a new source.
  4. **`initI18n` on freshness (step 6) before step 7.** Step 7 mounts the new path; the new path expects translations loaded by Gate 4. So step 6 must wire `initI18n` (or its successor that loads only the stale namespaces) into the freshness gate BEFORE step 7 switches the overlay over. Otherwise the new path mounts the portal with no translations loaded and shows raw keys.
  5. **`[BOOT]` instrumentation removal.** The `console.log` in `src/lib/config/api.ts:43-51` request interceptor goes with the apiStore teardown. Closes the matching `issues.md` 2026-05-28 cleanup obligation.

- **Part 4b — one adjacent observation.**
  - `BaseSiteSelector.tsx` `INTRO_FALLBACK` is a hardcoded-Serbian object (lines 21-34) used when `fetchNamespace(INTRO, 'sr')` is unresolved or fails. Low severity. The seam audit already flagged the hardcoded `'sr'` in the namespace fetch as a known scope-out (audit §5.4 / adjacent observations). This adjacent observation is the SAME-class issue on the fallback side — the fallback is also Serbian-only. Step 7's picker re-wire inherits both. Not in scope here, not fixed; file path: `src/components/init/BaseSiteSelector.tsx:21-34, 45`. "I did not fix this because it is out of scope" — Igor's UI directive ("DO NOT change the UI") and the deferred-re-wire decision both forbid edits to this component this brief.

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - **The `baseSiteOverviews: BaseSiteOverviewDTO[]` slot on the store.** Earned: the picker UI needs the list as data-in, and the action that resumes the machine (`pickBaseSite`) needs to look up the picked code in the same list to validate it. Without a store slot, the gate would have to either (a) re-fetch the list inside `pickBaseSite` (wasteful, doubled timeout exposure), or (b) leak the list through a prop chain that doesn't exist on this layout shape. The slot is the single place that connects the two halves of Gate 3 (the fetch on entry, the lookup on user tap).
    - **`pickBaseSite` as an action on the store (not a free function).** Earned: it reads three store slots (`baseSiteOverviews`, `language`) and writes four (`selectedBaseSite`, `language`, plus indirectly the `status` via `runFreshnessGate`/`toMaintenance`), persists two things to AsyncStorage, and calls `setCodes` on the barrier. A free function with closure over the store would be the same code with a worse name; an action keeps the dispatch surface uniform with `start`/`reEnter`/`runX`.
    - **Throw-on-failure semantics in the two new services.** Earned: the spec's "fails/times out → maintenance" is one rule, and the gate's `try/catch` + `withGateTimeout` is the one place it is expressed. Inner services that swallow into sentinel returns would force every gate to invent a sentinel check, duplicating the rule per-gate. The departure from `fetchBaseSites`'s swallow is documented in the file-leading comment and is temporary — step 7 deletes `fetchBaseSites`.
    - **`encodeURIComponent(code)` in `fetchBaseSiteByCode`.** Earned: defensive URL hygiene against any future code that contains URL-reserved characters. Costs zero today and rules out a class of footgun if the backend ever ships a code with `/`, `?`, `#`, etc.
    - **`buildSiteDTO`/`buildOverview` test-factory helpers.** Earned: every Gate 3 / `pickBaseSite` test needs a real-shaped DTO and most tests vary one field. Without factories the tests would be 10 lines of `{ id, code, domain, ... }` each, obscuring the assertion. Factories let each test write `buildSiteDTO({ allowedLanguages: [langSr] })` and the reader sees the test's claim immediately.
    - **`DEFAULT_STORED_SITE` constant in `beforeEach`.** Earned: keeps the original transition-table tests reaching `'ready'` without per-test setup, because the default Gate-3 path is now stored-site. Tests that need the no-stored-site branch opt in with `mockGetStoredBaseSite.mockResolvedValue(null)` — that opt-in is explicit and local, matching the brief's "tests overide as needed" idiom from step 4.
  - Considered and rejected:
    - **A separate `bootBaseSiteService.ts` file for the two new functions.** Rejected — they belong with the existing base-site fetch in `baseSitesService.ts`. The throw-vs-swallow distinction is documented in the file-leading comment; a separate file would force a future reader to compare the patterns across files.
    - **Returning `null` / empty array on overviews failure (matching `fetchBaseSites`).** Rejected — see "Throw-on-failure semantics" above. Would force Gate 3 to invent a sentinel check on top of `try/catch`, duplicating the spec's single rule.
    - **Surfacing the soft-update modal in `pickBaseSite`.** Rejected — `softUpdate` is set by Gate 2 and consumed by the view layer (modal mounted by step 7's overlay). `pickBaseSite` doesn't read or write the flag.
    - **A `<RequireBaseSite>`-style check inside `pickBaseSite` for "the user picked the SAME code as the stored one".** Rejected — that's a freshness optimisation (skip re-fetch if `fullDto.code === selectedBaseSite?.code`) but the brief calls for `pickBaseSite` to always fetch the full DTO on pick (since the picker only fires after `intro-picker`, where there is no stored site). The same-code case is the freshness gate's job (step 6).
    - **An `AbortSignal` threaded into the two new services to cancel an in-flight HTTP call on timeout.** Same posture as step 4: rejected, services don't accept signals today; the timeout still correctly diverts; the inflight call's eventual result is discarded by the gate's `try/catch` having already routed. Future brief can wire `signal` without changing the gate call sites.
    - **A `pickBaseSiteIfNeeded(code)` that no-ops when `code === selectedBaseSite?.code`.** Rejected — Gate 3 reaches `intro-picker` only when no stored site exists, so `selectedBaseSite` is null when `pickBaseSite` runs. The no-op check would be a contract that never fires.
    - **Auto-clearing `baseSiteOverviews` after a successful pick.** Rejected — `baseSiteOverviews` is harmless to retain (a small list of pointers), and clearing it would be one more write on the action's critical path. If a future brief wants to free memory, a `cleanup-after-pick` change is one line.
    - **Folding `runBaseSiteGate`'s stored-site read of language into `pickBaseSite`'s "previously-selected" lookup so both code paths use one helper.** Rejected — the two paths differ semantically: Gate 3 reads from AsyncStorage's persisted language; `pickBaseSite` reads from the current bootStore slot. Sharing a helper would muddle the source (storage vs slot) without saving more than two lines. The mirror to `AppContext.tsx:193-195` is preserved in both call sites verbatim.
  - Simplified or removed:
    - **Two now-unused test constants `seededBaseSite` / `seededLanguage`** deleted from the test file when the Invariant 3 tests no longer needed thin placeholder DTOs.
    - **Two stub-override `runBaseSiteGate: () => { ... }` literals** upgraded from sync to `async () => { ... }` to match the new signature — clarifies that the gate is async at every call site.

- **Closure gate (verified):** No drafted `conventions.md` / `decisions.md` / `state.md` / `issues.md` edits this session. The boot-redesign feature is mid-implementation; the closing `state.md` flip and the `issues.md` `[BOOT]`-log close land at step 7, applied by Docs/QA on Mastermind's draft. No Expo-backlog row applies to this engineering brief. The transitional-risk decision (option `a` — defer re-wire) is recorded above; no config-file edit is required for it.
