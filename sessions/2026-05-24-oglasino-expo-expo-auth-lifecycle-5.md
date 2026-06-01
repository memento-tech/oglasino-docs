# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-24
**Task:** Wire `initAuthListener` from `AppInit.tsx` with single-flight guard and add `restored`/`accountBanned` flags to `useAuthStore`

## Implemented

- Added `restored: boolean` and `accountBanned: boolean` flags to `useAuthStore` with simple setters (`setRestored`, `setAccountBanned`). Initial values `false`.
- Rewrote `initAuthListener` with a module-scoped single-flight guard (`listenerState`) following the web `UseTokenRefresh` pattern: `unsubscribe` check (no-op on second call), `inFlightUid` (prevents duplicate sync for same uid), `lastSyncedUid`+`lastSyncedAt` (2-second drop window for same-uid repeat emissions), reset on sign-out (null path).
- Switched the listener from `onAuthStateChanged` (via `listenAuthState`) to `onIdTokenChanged` (direct import), matching web's pattern — fires on sign-in, sign-out, and token rotation.
- The listener's catch block detects `USER_BANNED`/`EMAIL_BANNED` error codes, sets `accountBanned: true`, and calls `auth.signOut()`. Other errors log via `logServiceError`.
- Added `X-Account-Restored` header reading in the axios response interceptor's success handler in `api.ts`. On any 2xx response with the header, `setRestored(true)` is called.
- Added `isErrorWithCode` utility helper in `src/lib/utils/isErrorWithCode.ts` (mirrors web's shape).
- Wired `initAuthListener` from `AppInit.tsx` via a `useEffect` gated on `_hasHydrated`. The single-flight guard inside the listener is defense-in-depth.

## Files touched

- `src/lib/store/authStore.ts` (+88 / -19) — flags, setters, rewritten `initAuthListener` with single-flight guard
- `src/lib/config/api.ts` (+6 / -2) — `X-Account-Restored` header reading in response interceptor
- `src/components/init/AppInit.tsx` (+10 / -0) — hydration-gated call to `initAuthListener`
- `src/lib/utils/isErrorWithCode.ts` (+8 / new) — error code detection helper

## Tests

- `npx tsc --noEmit`: 10 pre-existing errors, 0 new (matches F22 baseline)
- `npm run lint`: 18 errors + 82 warnings, 0 new (matches F22 baseline)
- `npm test`: 109 passed, 0 failed (matches F22 baseline)
- `npx expo-doctor`: 17/18 passed, 1 pre-existing failure (package versions) (matches F22 baseline)

## Cleanup performed

- Removed the `listenAuthState` import from `authStore.ts` (no longer needed — listener uses `onIdTokenChanged` directly). `listenAuthState` remains exported from `authService.ts` and is still consumed by `InitFavoritesStore.ts`.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- The audit's Finding 1 ("`initAuthListener` is defined but never called") — resolved; the listener is now wired from `AppInit` with a single-flight guard. The finding is closeable after this session.
- The `listenAuthState` wrapper is no longer used by the auth store (but still used by `InitFavoritesStore`), so it is NOT deletable.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no unused imports, no console.log, no TODO/FIXME.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): flagged in "For Mastermind"
- Part 11 (trust boundaries): confirmed — trust boundary intact (see below)

## Known gaps / TODOs

- Dialog rendering (`AccountStateDialogsInit`) is Brief 4B's responsibility. Flags are wired but no dialog reads them yet.
- The 401/403 interceptor (F5) is blocked on Q2 and lives in Brief 7.
- The `accountJustDeleted` flag is chat E's territory; not added in this session.
- The foreground re-validation (F4, AppState listener) is Brief 6.
- Login/register/Google flows still call `syncUserToBackend` directly via `buildUserSession`; they do not use the listener path. This is pre-existing architecture. The listener will fire independently and its single-flight guard will drop the redundant sync (same uid within 2s). Ban handling from those paths relies on the listener catching the ban on its own sync attempt.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - Module-scoped `listenerState` object with four fields — required by the single-flight pattern from the brief. Matches web's `useRef` shape adapted for Zustand (no React hooks available inside store actions).
    - `isErrorWithCode` utility helper — one caller today (the listener), but the brief explicitly authorizes it and Brief 7 (401/403 interceptor) will use it too. Two callers visible.
  - Considered and rejected:
    - Extracting a shared `clearUserScopedStores()` helper for both `logout()` and the listener's null path — rejected because the null path only sets `user: null` and lets component effects handle cleanup (the existing secondary mechanism). Duplicating cleanup in the null path would couple the two code paths without benefit at this stage.
    - Adding `disabled` field check on the response body (like web does) — rejected because the mobile `AuthUserDTO` type doesn't declare it, and the backend returns 403 with error code for disabled users on `firebase-sync`. The error-code detection in the catch is sufficient.
  - Simplified or removed:
    - Removed the `listenAuthState` import from `authStore.ts` — the old listener wrapper is replaced by direct `onIdTokenChanged` usage, reducing one indirection layer.

- **Header-read placement decision:** chose the axios response interceptor in `api.ts`. Rationale: (1) consistent with web's pattern, (2) keeps `syncUserToBackend` simple (just returns `AuthUserDTO`), (3) the header is automatically read on any response from any endpoint (future-proof if other endpoints set it). Introduces a circular module dependency (`api.ts` → `authStore.ts` → `authService.ts` → `api.ts`) but all cross-module accesses are inside callbacks (runtime only, not module-evaluation time), which is safe. Tests confirm no runtime issue.

- **AppContext-vs-_hasHydrated determination:** AppContext readiness (`status === 'ready' || 'loading'`) does NOT imply Zustand `_hasHydrated`. AppContext waits for a backend config fetch; `_hasHydrated` fires when AsyncStorage rehydrates the auth store (local, fast, independent). Added an explicit `_hasHydrated` gate in `AppInit`'s effect as defense-in-depth. In practice, `_hasHydrated` fires before AppContext readiness (local storage < network), so the gate will almost never actually delay the listener subscription.

- **Trust boundary confirmation:** Intact per spec §8. `restored` is set by reading the server-set `X-Account-Restored` header (server-derived, client cannot forge response headers). `accountBanned` is set by reading the server-returned error code `USER_BANNED`/`EMAIL_BANNED` (server-derived). The persisted `user` in the store is never used in trust decisions — every authenticated request goes through the axios interceptor which reads `auth.currentUser.getIdToken()` (Firebase's verified token).

- **Part 4b adjacent observations:**
  - `InitFavoritesStore.ts` still uses `listenAuthState` (which wraps `onAuthStateChanged`). This means there are now TWO Firebase auth listeners: `onIdTokenChanged` (from `initAuthListener`) and `onAuthStateChanged` (from `InitFavoritesStore`). They fire on overlapping events but with different timing for token rotations. The favorites listener doesn't call `syncUserToBackend`, so the duplication is benign but worth noting. File: `src/components/init/InitFavoritesStore.ts`. Severity: low. I did not fix this because it is out of scope (the brief says "Component-driven cleanup remains in place as a secondary safeguard").
  - The `console.error` calls in `login`, `register`, `loginWithGoogle`, `loginWithFacebook` actions (`authStore.ts:89,104,119,134`) violate Part 4 cleanliness ("No console.log or ad-hoc debug logging"). Pre-existing. Severity: low. I did not fix this because it is out of scope for this brief.
  - The `loginWithFacebookFirebase` function in `authService.ts` is entirely commented-out code that returns `null` — violates Part 4 ("No commented-out code"). Pre-existing. Severity: low. I did not fix this because it is out of scope.
