# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-25
**Task:** Extend the axios response interceptor in `src/lib/config/api.ts` to handle 401 (token-refresh-and-retry) and 403 USER_BANNED / EMAIL_BANNED (sign-out and ban dialog).

## Implemented

- **403 USER_BANNED / EMAIL_BANNED handler** (api.ts:68-74): On a 403 response whose body matches `code === 'USER_BANNED'` or `code === 'EMAIL_BANNED'` (checked via existing `isErrorWithCode` helper), the interceptor calls `auth.signOut()` (fire-and-forget), sets `useAuthStore.getState().setAccountBanned(true)`, and returns a never-resolving promise to prevent the caller from receiving the error and double-handling. Matches web's pattern at `user-deletion.md` §14.12. Other 403s fall through to `Promise.reject(error)` unchanged.
- **401 single-flight token-refresh-and-retry** (api.ts:76-114): On a 401 response, the interceptor checks a `_retry` flag on the request config to detect retried requests. If not retried and `auth.currentUser` exists, it force-refreshes the token via `getIdToken(true)` and retries the original request. Concurrent 401s are queued via a `pendingRequests` array — only one `getIdToken(true)` call runs at a time; queued requests retry with the new token once it resolves. On refresh failure, queued requests are drained with null (causing them to reject with their original errors), and `auth.signOut()` is called. On second 401 (retried request fails again), `auth.signOut()` is called without setting `accountBanned` — 401 means token problem, not ban.
- **All existing handlers preserved unchanged**: `ERR_NETWORK`/`ECONNABORTED`, missing response, 404, and Brief 4A's `X-Account-Restored` success-side handler all remain in their original positions with original logic.
- **Module augmentation** (api.ts:7-11): `InternalAxiosRequestConfig` extended with `_retry?: boolean` via `declare module 'axios'` for type-safe retry-flag access on request configs.

## Files touched

- `src/lib/config/api.ts` (+58 / -2)

## Tests

- `npx tsc --noEmit`: 10 pre-existing errors (unchanged from Brief 6 baseline). Zero new errors.
- `npm run lint`: 18 errors + 83 warnings (Brief 6 baseline: 18 errors + 82 warnings). Zero new errors/warnings from `api.ts`. The 1-warning variance is in unrelated files.
- `npm test`: 109 passed (unchanged from Brief 6 baseline).
- `npx expo-doctor`: 17/18 passed, 1 pre-existing version-mismatch failure (unchanged from Brief 6 baseline).
- No unit test added for the interceptor this session (optional per brief; manual end-to-end verification deferred to Phi1-close stage).

## Cleanup performed

None needed. No commented-out code, no unused imports, no console.log, no TODO/FIXME added.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (Phi1 closure and Risk Watch row flip are for Mastermind to draft after Phi1's last brief ships)
- issues.md: no change

## Obsoleted by this session

- **Finding 5 ("No 401/403 response interceptor")** from the structural audit becomes closeable. F5 was the last of seven Phi1 findings. With all seven findings addressed (F1=Brief 2, F2=Brief 3, F3=Brief 3, F4=Brief 4A/4B, F5=this brief, F13=Brief 7, F22=Brief 4A), Phi1 closes.
- The `uploadImages.ts:282-298` local 401+TOKEN_EXPIRED retry remains in place per brief scope. It handles image-upload-specific retry against R2 (different error body shape: `{error: {code: 'TOKEN_EXPIRED'}}` vs the backend's Part 7 envelope `{errors: [{code: ...}]}`). The global interceptor does NOT overlap because `uploadImages` uses `uploadPrimitive.ts` (raw `fetch`, not axios). Left for follow-up (Phi3/Omega scope at earliest).

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no unused imports/variables, no console.log, no TODO/FIXME.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): flagged in "For Mastermind."
- Part 7 (error contract): confirmed. The interceptor matches on the Part 7 envelope `{errors: [{code: ...}]}` via `isErrorWithCode`. Trust boundary confirmed — the codes matched (`USER_BANNED`, `EMAIL_BANNED`) are server-derived from `FirebaseAuthFilter.java` and `AuthController.java`. Client cannot forge a response.
- Part 11 (trust boundaries): confirmed. F5 is a clean trust-enforcement point. No client-supplied value influences the trust decision. The 403 status and body code are server-set; the 401 retry uses `getIdToken(true)` which returns a cryptographically signed Firebase ID token validated by the backend.

## Known gaps / TODOs

- **Local 401 retry in `uploadImages.ts` could be consolidated.** The image pipeline uses raw `fetch` (not axios), so the global interceptor doesn't apply there. The two retry mechanisms operate on different HTTP clients against different endpoints (backend API vs R2 direct upload). Consolidation is out of Phi1 scope; noting for Phi3/Omega.
- **No "session expired" dialog on second 401.** Per brief, sign-out alone is sufficient in Phi1. A dedicated dialog is Phi3/Omega scope at earliest.
- **Auth listener's null-path cleanup is minimal.** When the interceptor calls `auth.signOut()`, the `onIdTokenChanged` listener fires and sets `user: null` but does not clear chat/favorites/notification/filter stores — only the `logout()` action does full store cleanup. This is the intended design (matches web's pattern), but means those stores retain stale data until the user navigates to the public surface. Not F5's scope to fix.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity):
    - Module augmentation for `_retry` on `InternalAxiosRequestConfig` (api.ts:7-11) — earned because TypeScript's `AxiosRequestConfig` has no index signature; without the augmentation, the `_retry` flag would require `as any` casts at every access point. One declaration, two access sites.
    - Single-flight `isRefreshing` / `pendingRequests` module-scoped state (api.ts:19-20) — earned because concurrent 401s (e.g., parallel API calls when a token expires) would each independently call `getIdToken(true)`, producing redundant Firebase round-trips and potential race conditions. The brief explicitly specifies this pattern.
  - Considered and rejected:
    - A dedicated `TokenRefreshManager` class encapsulating the single-flight logic — rejected because the logic has exactly one consumer (the interceptor) and the module-scoped variables are simpler. No second consumer foreseeable.
    - Extracting the 403 ban-handler and 401 refresh-handler into separate functions — rejected because each is used once, the interceptor reads top-to-bottom clearly, and extraction would scatter the flow across the file.
    - Awaiting `auth.signOut()` in the 403 handler — rejected per web's pattern: `auth.signOut()` is fire-and-forget because the never-resolving promise means the caller never receives control back anyway. Awaiting would delay `setAccountBanned(true)` unnecessarily.
  - Simplified or removed: nothing.

- **Part 4b adjacent observations:**
  - **Auth listener null-path store cleanup gap** — `authStore.ts:198-204` (`initAuthListener`'s null-path) only calls `set({ user: null })`. The `logout()` action (lines 144-189) additionally clears viewTokenStore, chatStore, favoritesStore, notificationStore, and filter stores. When `auth.signOut()` is called from the interceptor (or any non-`logout()` path), those stores retain stale data. Severity: low (user is immediately navigated to public surface; stale data in stores is invisible). File: `src/lib/store/authStore.ts:198-204`. I did not fix this because it is out of scope — it affects the auth listener, not the interceptor. A future brief could either add store cleanup to the listener's null-path or route interceptor sign-outs through the `logout()` action.
  - **`uploadImages.ts` local retry overlap** — already noted in Known gaps. The image pipeline's 401 retry (uploadImages.ts:282-298) handles a different error shape (`{error: {code: 'TOKEN_EXPIRED'}}`) against R2 direct upload (raw `fetch`, not axios). No functional overlap with the global interceptor, but the parallel retry mechanisms are worth noting for future consolidation. Severity: low (no user-facing impact). File: `src/lib/images/uploadImages.ts:282-298`. I did not fix this because the brief explicitly says not to.

- **`_retry` flag shape decision:** Used axios module augmentation (`declare module 'axios'` extending `InternalAxiosRequestConfig`) rather than `(config as any)._retry`. Rationale: type-safe at both read and write sites; standard axios community pattern for custom config properties; zero runtime cost (compile-time only).

- **Single-flight queueing shape decision:** Module-scoped `isRefreshing: boolean` + `pendingRequests: Array<(token: string | null) => void>`. The `pendingRequests` array holds resolver functions; each queued 401 returns a Promise that waits for the refresh to complete, then retries with the new token (or rejects if null). Rationale: matches the brief's specified shape exactly; minimal state surface; `finally` block guarantees `isRefreshing = false` even on exception paths.

- **Circular import confirmation:** The `api.ts` → `authStore.ts` → `authService.ts` → `api.ts` circular dependency (flagged by Brief 4A) continues to work without issue. F5 adds no new imports to the cycle — `useAuthStore` was already imported (line 3). The new `isErrorWithCode` import is outside the cycle entirely. The `useAuthStore.getState()` pattern is lazy (evaluated at call time, not import time), so the circular module resolution is unaffected.

- **Closure gate verification:** No implicit config-file dependency. F5 does not introduce new translation keys, new backend routes, or new state.md entries. The Phi1 closure (marking F5 as shipped, flipping the Risk Watch row about "Mobile lifecycle defects") is for Mastermind to draft after confirming all seven findings are addressed; the draft goes through Docs/QA per Part 3.
