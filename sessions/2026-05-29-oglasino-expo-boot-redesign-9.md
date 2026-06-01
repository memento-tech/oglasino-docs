# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Task:** expo-boot-redesign step 7a.1 — a tactical follow-up to 7a. Remove ONLY the `apiStore` `globalThis` bootstrap barrier (one specific Part 7 deletion-list item) to unblock the cold-boot / no-stored-site DEADLOCK: the picker's INTRO translation fetch was queued at the request interceptor behind `waitForBootstrap()` and never reached the wire, so the picker spun forever. Everything else in the Part 7 deletion list (AppContext.bootstrap, AppVersionConfigInit, initI18n/loader.ts, the `[BOOT-START]` log in bootStore, `<RequireBaseSite>`) STAYS for 7b.

## Brief vs reality

Nothing worth stopping for. One expected wrinkle, handled (not a challenge): the brief's "existing 176 tests still pass" co-exists with "remove every `setCodes` call site." Removing `setCodes` from `bootStore.ts` necessarily falsifies the `bootStore.test.ts` assertions that asserted `setCodes` was called. That is fallout of the instructed change, not a contradiction in the brief — I updated those tests (removed the `apiStore` mock and the `setCodes` expectations) so the suite stays green at 176. No test was added; assertions on a now-deleted mechanism were removed. Documented under "Files touched" and "Obsoleted."

## Implemented

The one fix, and only it:

1. **Deleted `src/lib/config/apiStore.ts` entirely.** Every export is gone: `getCurrentBaseSiteCode`, `getCurrentLangCode`, `setCodes`, `setLangOnly`, `waitForBootstrap`, the `globalThis`-pinned `ApiStoreState`, and the waiter queue.

2. **`src/lib/config/api.ts` request interceptor re-sourced to bootStore.**
   - Removed the `import { getCurrentBaseSiteCode, getCurrentLangCode, waitForBootstrap } from './apiStore'`; added `import { useBootStore } from '../store/bootStore'`.
   - Removed the `waitForBootstrap()` await and the `!getCurrentBaseSiteCode() && !config._bootstrap` branch — there is no barrier to skip, so the `_bootstrap` flag is moot in the interceptor (no branching on it remains).
   - Removed the `[BOOT]` `console.log` line (with its `willAwait` computation).
   - Headers now read from bootStore, conditionally (no more setting the header to `null` before codes exist):
     ```ts
     const { selectedBaseSite, language } = useBootStore.getState();
     if (selectedBaseSite) config.headers.set('X-Base-Site', selectedBaseSite.code);
     if (language) config.headers.set('X-Lang', language.code);
     ```
   - Kept the `_bootstrap?: boolean` type augmentation (lines 15-18) — service call sites still pass `_bootstrap: true`; removing the type would break their compile. Both are the 7b cosmetic sweep.

3. **`src/lib/store/bootStore.ts` — `setCodes` removed from the new path.**
   - Dropped the `import { setCodes } from '../config/apiStore'`.
   - Deleted the `setCodes(storedSite.code, lang.code)` call in `runBaseSiteGate` (stored-site path) and the `setCodes(fullDto.code, lang.code)` call in `pickBaseSite`. The slots (`selectedBaseSite` / `language`) are now the sole source the interceptor reads — the explicit "open the barrier" semantic is gone because there is no barrier.
   - Updated the two surrounding doc comments that said "call setCodes" to describe the slots-as-source model (functions I was already editing; kept accurate).

4. **`src/components/context/AppContext.tsx` — narrow carve-out (old path, slated for 7b).** Stripped references to the deleted exports JUST ENOUGH to compile; the surrounding functions/file are untouched:
   - Removed `import { setCodes, setLangOnly } from '@/lib/config/apiStore'`.
   - Removed `setCodes(storedBaseSite.code, language.code)` in `bootstrap()` (stored-site branch).
   - Removed `setCodes(site.code, lang.code)` in `setBaseSiteForCode`.
   - Removed `setLangOnly(lang.code)` in `setLanguageForCode`.
   These four line-removals are the entirety of the AppContext edit — no logic, no surrounding structure changed. 7b's deletion brief should know AppContext is already partially edited here.

5. **`src/lib/store/bootStore.test.ts` — kept green.** Removed `mockSetCodes` (hoisted decl + factory entry + `mockReset`), the `vi.mock('../config/apiStore', ...)`, and the six `expect(mockSetCodes)...` assertions across the Gate-3 and pickBaseSite describes (two test titles reworded to drop "setCodes"). No tests removed; assertion count on the deleted mechanism dropped. Suite stays 176.

## Files touched

- `src/lib/config/apiStore.ts` — **deleted**.
- `src/lib/config/api.ts` — interceptor re-sourced to bootStore; `waitForBootstrap` + `[BOOT]` log removed; import swapped.
- `src/lib/store/bootStore.ts` — `setCodes` import + 2 call sites removed; 2 doc comments updated.
- `src/components/context/AppContext.tsx` — import + 3 call sites removed (narrow carve-out; old path).
- `src/lib/store/bootStore.test.ts` — `apiStore` mock + `setCodes` assertions removed.

(All boot-redesign files are untracked on `new-expo-dev` — the feature is uncommitted per state.md.)

## Tests

- `npx vitest run` → **176 passed** (11 files, 0 failed). Unchanged count from 7a.
- `npx tsc --noEmit` → clean.
- `npx eslint` (touched paths) → **0 errors, 4 warnings**, all pre-existing and none introduced by this session:
  - `api.ts:31` `Array<T>` forbidden — on the untouched `pendingRequests` declaration.
  - `bootStore.test.ts:121-122` `import/first` — the accepted vitest mock-hoisting pattern (import-under-test after `vi.mock`), same as before.
  - `bootStore.ts:427` `i18n.use` named-as-default-member — pre-existing Gate-4 i18n init line.
- Did NOT add an interceptor test (brief: low value for the disruption; future cleanup may add one).
- Did NOT run `npx expo start` (long-running dev server). Igor runs the live device verification below.

## Cleanup performed

- Removed the `[BOOT]` diagnostic `console.log` in the `api.ts` interceptor (closes the issues.md 2026-05-28 cleanup obligation for that line — see Config-file impact).
- Deleted `apiStore.ts` whole (refactor obsoleted it; deleted in the same session per conventions Part 4).
- Removed the now-dead `apiStore` mock plumbing in `bootStore.test.ts`.
- No commented-out code, no `TODO`/`FIXME` added.

## Config-file impact

- **issues.md:** the 2026-05-28 "Live `[BOOT]` diagnostic instrumentation in `api.ts` request interceptor" cleanup obligation is now **satisfiable** — the `[BOOT]` log it tracks was removed this session. Docs/QA is the sole writer; draft for the entry's status: *fixed (2026-05-29, session `oglasino-expo-boot-redesign-9`) — the `[BOOT]` interceptor log was removed when the apiStore barrier was deleted.* I did not edit issues.md.
  - Note: the 2026-05-28 "always-mounted `<Stack>` exposes pre-bootstrap API calls (root cause)" entry references the apiStore barrier as Φ3's mitigation (1). That mitigation no longer exists; the structural risk it described is now governed by bootStore's slot population + the gates, not a queue. Flagged for Docs/QA — no edit made.
- **state.md:** no change required this session. The Expo-backlog `Version checksums` row flip to `mobile-stable` remains deferred to feature close (after 7b + Igor's live regression), exactly as 7/7a left it. **Closure-gate check:** there is no implicit config-file dependency this session leaves unstated — apiStore removal does not change any feature's mobile-adoption status.
- **conventions.md / decisions.md:** no change.

## Obsoleted by this session

- `src/lib/config/apiStore.ts` and all its exports — **deleted** (this was one specific item on the Part 7 deletion list; the rest of that list is untouched and still owed by 7b).
- The interceptor's queue-behind-`waitForBootstrap` mechanism and its `[BOOT]` instrumentation.
- The `setCodes`/`setLangOnly` "open the barrier" call sites in both the new path (bootStore) and, narrowly, the old path (AppContext).
- The `bootStore.test.ts` assertions on `setCodes`.

## Conventions check

- **Part 4 (cleanliness):** confirmed. No commented-out code, no debug logging left (the one debug log this brief targeted was removed). No unused imports — verified by tsc + grep (only remaining apiStore-string match is a stale doc comment in `versionsService.ts`, see Part 4b). No `TODO`/`FIXME`.
- **Part 4a (simplicity):** the change is net-subtractive — a whole module and a queueing mechanism deleted, headers sourced from the one place that already owns them (bootStore slots). No abstraction added. The conditional `if (selectedBaseSite)` header set is simpler and more correct than the old unconditional `set(..., null)`.
- **Part 4b (adjacent observations):** see "For Mastermind."
- **Part 6 (translations):** no new keys; backend-seeded keys unaffected.
- **Spec Part 1 invariants + structural defense:** re-confirmed below.

## Known gaps / TODOs

- No `TODO`/`FIXME` added. The `_bootstrap: true` flags at service call sites and the stale `versionsService.ts` doc comment that references the "apiStore bootstrap barrier" are intentionally left for 7b's cosmetic sweep (the brief explicitly licenses leaving the flags).
- The `[BOOT-START] start() called` `console.log` in `bootStore.ts` `start()` (line ~113) is NOT this brief's `[BOOT]` target (that was the interceptor) and is out of 7a.1 scope — left in place; flagging for 7b cleanup.

## For Mastermind

- **Three invariants — re-confirmed:**
  1. **One mount effect, `[]` deps:** unchanged. This brief touched no effects. The interceptor is not an effect.
  2. **Machine writes status, view reads status:** holds. The interceptor now *reads* `selectedBaseSite`/`language` from bootStore via `getState()` — it never writes `status` and never writes a slot. bootStore remains the only writer.
  3. **Re-entry destroys nothing:** holds. Removing the barrier removed only a queueing mechanism; the gates' slot population is unchanged. The Invariant-3 tests (and the real-Gate-4 re-entry test) still pass.
- **Structural defense — re-confirmed (one-way edge, no Cycle B):** `api.ts` now imports `useBootStore` from `bootStore.ts` directly. Verified `bootStore` and its whole transitive closure reachable from `api.ts` (`bootFreshness`, `bootGate`, `checksumStorage`, `baseSitesService`, `versionsService`, `fetchNamespace`, `maintenanceService`, `appVersionConfigService`, `configurationService`, `firebaseClient`) import **no** `authStore` and **no** chat-store. So `api.ts` is **not** pulled into Cycle B (authStore ↔ chat-store). The only `authStore`/`chat` strings in that closure are the doc-comment in `bootStore.ts` describing the defense.
- **New benign require cycle introduced (report, per brief):** `api.ts → bootStore.ts → {baseSitesService, versionsService, fetchNamespace, maintenanceService, appVersionConfigService, configurationService} → api.ts (BACKEND_API)`. Before this session `api.ts` imported `apiStore` (a leaf) so this loop did not exist. It is **benign**, same character as the documented Cycle B (issues.md 2026-05-28): every cyclic edge is used lazily, never at module-evaluation time —
  - the interceptor reads `useBootStore.getState()` at request time;
  - bootStore calls the services inside gate-function bodies;
  - the services touch `BACKEND_API` inside their async functions.
  No module reads a cyclic binding at eval time, so ESM live bindings resolve it with no load-order hazard. Tests (which import `api.ts` transitively) and tsc are clean. No action needed; noting it so a future engineer who runs a cycle detector isn't surprised, and so 7b is aware. If it ever needs breaking, the lazy `getState()` read makes a `require`-on-call swap trivial.
- **Adjacent observations (Part 4b):**
  - `src/lib/init/versionsService.ts:13` — doc comment still says `_bootstrap: true` "exempts the call from the apiStore bootstrap barrier." The barrier is gone, so the comment is stale. NOT edited (beyond the narrow carve-out, which is compile-only; comments don't block compile, and the `_bootstrap` flags themselves are explicitly deferred to 7b). Recommend folding into 7b's `_bootstrap`-flag sweep.
  - `bootStore.ts` `start()` `[BOOT-START]` log — see Known gaps; 7b cleanup.

## Manual verification (Igor runs after commit)

1. Cold-restart on a real device/simulator with **NO stored base site** (clear storage / fresh install).
2. The picker should appear within a few seconds (welcome card + flag buttons). The picker's own `<ActivityIndicator />` early-return should now resolve, because its `fetchNamespace(INTRO, 'sr')` call is no longer queued — with the barrier gone it reaches the wire immediately (sending no `X-Base-Site`, which `/public/translations` allowlists for the lang-only path).
3. Backend access log should show `/translations?namespace=INTRO&lang=sr` returning **200** (the call that was invisible for 10+ minutes before).
4. Tap a base site → freshness gate runs → portal renders.
5. **Boot-loop regression (spec Part 8, the test the deadlock was blocking):** cold-restart with maintenance ON → see maintenance screen → toggle maintenance OFF → app returns to `ready` within ~5s WITHOUT clearing the stored base site and WITHOUT extra boot passes.

If the picker still spins forever, the deadlock has a second cause not addressed here — report back rather than poking further.

## Step-7b reminder (updated)

- **`apiStore` is already gone** (deleted this session) — strike it from the Part 7 deletion list.
- **AppContext.tsx is already partially edited** — the `setCodes`/`setLangOnly` import and its 3 call sites were removed (narrow carve-out). The rest of AppContext (bootstrap, the Promise.all, the catch-all-to-maintenance, the duplicate maintenance poll, `setBaseSiteForCode`/`setLanguageForCode`/`reBootstrap`) is intact and still owed.
- Still owed by 7b (unchanged): AppContext.bootstrap teardown, AppVersionConfigInit as a gate, initI18n/loadAllNamespaces/loader.ts, `<RequireBaseSite>`, the `_bootstrap: true` flags at service call sites + the stale `versionsService.ts` barrier comment, the `[BOOT-START]` log in `bootStore.start()`, the 3 spec amendments, the state.md `Version checksums` flip, the decisions.md feature-close entry.
