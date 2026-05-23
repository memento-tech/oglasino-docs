# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-21
**Task:** Single-flight guard in UseTokenRefresh for duplicate firebase-sync and push/token POSTs.

## Implemented

- Added a single-flight + recent-completion guard inside `src/components/client/initializers/UseTokenRefresh.tsx`'s `onIdTokenChanged` callback. The guard holds `{ inFlightUid, lastSyncedUid, lastSyncedAt }` in a `useRef`. While a sync for uid X is in flight, a second same-uid emission's cascade is dropped; for `SAME_UID_REPEAT_WINDOW_MS = 2000` ms after a completed sync for uid X, a same-uid emission's cascade is dropped.
- Hoisted `writeFirebaseTokenCookie(token)` into the listener body, before the guard. The cookie mirror runs on every signed-in emission (including the second one whose backend cascade is dropped), preserving the audit-1 adjacent-#4 observation that the rotated token must reach the SSR cookie. The redundant `writeFirebaseTokenCookie` call inside `syncUserToBackend` (`authService.ts:136`) is a noop on the first emission via its existing token-string dedupe (`authTokenCookie.ts:36`).
- Reset the guard's ref state on every signed-out emission so a sign-out → sign-in handshake always re-fires a fresh cascade. The deletion-in-flight branch (auth contract C-6) is untouched; it still short-circuits before the guard, with its own cookie write.
- `lastSyncedAt` is only set on the success path. If `syncUserToBackend` throws, the next emission can retry immediately — failed cascades do not earn a dedupe credit.

## Files touched

- src/components/client/initializers/UseTokenRefresh.tsx (+50 / -3)

## Tests

- Ran: `npx tsc --noEmit` — clean.
- Ran: `npx eslint src/components/client/initializers/UseTokenRefresh.tsx` — clean.
- Ran: `npm run lint` — 0 errors, 175 pre-existing warnings (none in the touched file).
- Ran: `npm test` — 206 passed (17 files). No tests exist for `UseTokenRefresh.tsx`; the touched paths have no test coverage to add to. The brief's verification is entirely runtime-observable (backend access-log counts + browser cookie inspection), not unit-test-shaped, and the brief did not request new tests.
- **Manual verification NOT performed.** The brief's verification steps (hard refresh `/rs-sr` and observe 1× sync + 1× push/token in the backend access log; sign-out → sign-in cycle; cookie-mirror inspection through the cold-load sequence) require a real browser session, a running backend with log visibility, and Firebase sign-in credentials. I cannot drive any of those as a Claude Code agent. Additionally, Igor has WIP on `app/[locale]/(portal)/(public)/page.tsx` (cookies-V2 refactor) in the cold-load path, so starting the dev server now would mix this fix with that WIP in any observation. Flagging this clearly so Igor runs the manual verification himself before considering the brief closed.

## Cleanup performed

- None needed. The change is additive within a single file: one new `useRef`, one new constant, one hoisted `writeFirebaseTokenCookie` call, and a `try/finally` wrapping the existing backend cascade. No code was obsoleted, no imports removed, no commented-out lines left behind. The existing in-file comment block on `onIdTokenChanged` semantics is preserved.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

The `decisions.md` 2026-05-21 "sole hydrator" contract is preserved — the listener remains the sole caller of `syncUserToBackend` on Firebase emissions, and `refreshUser` (the only other caller, user-action-triggered from `UserBasicDataSelectorDialog`) is untouched.

## Obsoleted by this session

- Audit-1 Q2 (firebase-sync fires twice on hard refresh) and Q3 (push/token POSTs twice on hard refresh) findings in `.agent/2026-05-21-oglasino-web-portal-multi-search-investigation-1.md` close once Igor confirms the manual verification. The root cause those questions identified is now guarded; nothing else in the audit's reasoning is obsoleted.
- Nothing in the codebase was deleted by this session.

## Conventions check

- Part 4 (cleanliness): confirmed.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one observation flagged in "For Mastermind."
- Part 6 (translations): N/A this session.
- Other parts touched: none.

## Known gaps / TODOs

- Manual runtime verification of the four behavioral cases (hard-refresh count drop, normal-refresh non-regression, sign-out → sign-in re-fire, cookie mirror through the cold-load sequence) is owed by Igor. The brief's "Definition of done" requires it before closure.
- The cascade can still in principle dedupe-block a *legitimate* same-uid re-sync if it arrives within the 2s window from a non-listener path — but the only non-listener caller of `syncUserToBackend` is `useAuthStore.refreshUser`, which the brief explicitly excludes from the guard and which lives entirely outside the listener closure. No real-world flow hits this case today.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):**
    - One `useRef` carrying three fields (`inFlightUid`, `lastSyncedUid`, `lastSyncedAt`) — earned: the listener callback must observe its own prior emissions across renders, and the listener is the closure-bound function inside a `useEffect`. A ref is the idiomatic React way to do this. Considered the module-scoped alternative the brief named; rejected because the component mounts exactly once (`AppInit.tsx:28`) and a `useRef` reads more naturally next to the `useEffect` that wires the listener, without needing a separate explanatory comment for "why module-level."
    - One module constant `SAME_UID_REPEAT_WINDOW_MS = 2000` — earned: per Part 4a, a value with one setting and no foreseeable second setting belongs as a constant, not config. The brief said the same. Placed inline at the top of the file rather than in a separate `constants.ts` because there is no related constant to group with and the file is the only consumer.
    - Hoisting `writeFirebaseTokenCookie(token)` into the listener — earned: the brief's constraint that the cookie mirror must run on every emission, combined with the fact that the cookie write currently lives inside `syncUserToBackend`, forces the hoist. The token-string dedupe inside `writeFirebaseTokenCookie` makes the redundant call safe (the first-emission path executes the noop dedupe inside `syncUserToBackend`'s internal call).
  - **Considered and rejected:**
    - **Move `writeFirebaseTokenCookie` out of `syncUserToBackend` entirely** and call it only from the listener (and from `refreshUser`). Rejected: bigger blast radius across two files (`authService.ts` + `useAuthStore.ts`), and the brief asks the fix to be confined to `UseTokenRefresh.tsx`. The dedupe inside `writeFirebaseTokenCookie` makes the in-file solution safe today.
    - **Set `lastSyncedAt` in `finally` regardless of error.** Rejected: a thrown cascade should not buy a 2s drop window for the next emission; it should be retryable immediately. Only the success path sets the timestamp.
    - **Compare `lastSyncedUid` and uid via a single boolean rather than a struct.** Rejected: three separate fields document intent better than packing them into one tuple; refactoring later is trivial if the shape proves wrong.
    - **A module-scoped equivalent of the ref.** Mentioned by the brief as acceptable. Rejected for the readability reason above; both have the same single-mount lifetime in this app.
  - **Simplified or removed:** nothing.
- **Adjacent observation (Part 4b):**
  - The redundant `writeFirebaseTokenCookie(token)` call inside `syncUserToBackend` (`src/lib/service/reactCalls/authService.ts:136`) is now strictly defensive — every caller of `syncUserToBackend` on the listener path now writes the cookie first. The only other caller, `useAuthStore.refreshUser` (`src/lib/store/useAuthStore.ts:281`), does *not* write the cookie before calling `syncUserToBackend`, so the in-function write is still load-bearing for that path. Severity: low. I did not refactor it because it is out of scope and currently correct; if a future brief consolidates the cookie write into a single owner, the listener already does the right thing.
- **Structural note for Mastermind (called out as "new information" per the brief):**
  - The brief stated: *"The brief assumes the cookie mirror is independent; if it's not, that's new information."* It is not independent — `writeFirebaseTokenCookie(token)` is called from inside `syncUserToBackend` (`authService.ts:136`), not directly in the listener. The brief covered this case ("the engineer should trace the call order before applying the guard and flag in 'For Mastermind' if the structure forces a less-clean implementation than expected"). My implementation hoists a `writeFirebaseTokenCookie` call into the listener before the guard; the in-`syncUserToBackend` call dedupes itself via the token-string guard on the first emission of a pair, and is correct standalone for the `refreshUser` caller. Net result: the guard is confined to `UseTokenRefresh.tsx`, the cookie mirror still runs on every emission, and no cross-file refactor was needed. Flagging in case the structural duplication ("cookie write in two places") feels untidy enough to warrant a follow-up brief to consolidate; in my read, it is correct and load-bearing for the `refreshUser` path, so I'd leave it.
- **Four constraints from the brief — covered:**
  1. **Same-uid in-flight drop.** Lines 80–80 of the new file (`if (state.inFlightUid === uid) return;`). The `inFlightUid` is set before the `await syncUserToBackend` and cleared in `finally`.
  2. **Same-uid recent-completion drop.** Lines 81–84 of the new file. `lastSyncedAt` is set after `setUser` + `initPushForAuthenticatedUser`; the window check is `Date.now() - lastSyncedAt < SAME_UID_REPEAT_WINDOW_MS`.
  3. **Different uid passes through.** Both guard checks key on `uid === state.inFlightUid` / `state.lastSyncedUid`; a sign-out → sign-in as a different user trips neither check. (The sign-out emission also resets the ref state to defaults, so the next signed-in emission for any uid starts fresh.)
  4. **Sign-out passes through.** The `if (!firebaseUser)` branch returns before the guard runs and clears the ref state, so the next sign-in emission is never blocked by stale guard data from the prior user.
- **`refreshUser` (constraint 5) untouched.** The guard lives entirely in the listener closure inside `UseTokenRefresh.tsx`. `useAuthStore.refreshUser` is not affected; `UserBasicDataSelectorDialog` remains the only caller of `refreshUser`, and its user-action-triggered semantics are intact.
- **Manual verification still owed.** I cannot drive a browser sign-in flow; Igor's WIP on `app/[locale]/(portal)/(public)/page.tsx` (cookies-V2 refactor) in the cold-load path argues against me starting the dev server independently. Igor should run the four-step verification from the brief before considering this brief closed.
