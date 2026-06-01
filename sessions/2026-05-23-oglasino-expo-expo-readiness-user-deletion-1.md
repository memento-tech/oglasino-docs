# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Task:** Read-only audit of the full user-lifecycle surface on mobile: account deletion, account disabling, token-revocation handling, sign-out behavior, and `disabled` claim enforcement. Identify gaps against `features/user-deletion.md`.

## Implemented

- Produced `.agent/audit-expo-readiness-user-deletion.md` with all 14 sections per the brief.
- No code changes (read-only audit).

## Files touched

- `.agent/audit-expo-readiness-user-deletion.md` (new, audit output)

## Tests

- N/A — read-only audit, no code changes.

## Cleanup performed

none needed — read-only audit.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change. Two trust-boundary findings (push token `userId`, user update `id`/`firebaseUid`) are pending-verification against the backend and would become issues.md entries if confirmed. Drafted in Section 11 "For Mastermind" of the audit.

## Obsoleted by this session

nothing — read-only audit.

## Conventions check

- Part 4 (cleanliness): N/A this session.
- Part 4a (simplicity): N/A — no code changes. See structured evidence below.
- Part 4b (adjacent observations): five cross-feature observations flagged in audit Section 11.
- Part 6 (translations): N/A this session.
- Part 7 (error contract): touched — reviewed 401/403 handling against the error contract. Mobile does not implement `USER_BANNED` code handling.
- Part 11 (trust boundaries): touched — explicit check in audit Section 8. Two findings pending verification.

## Known gaps / TODOs

- none — all gaps are documented in the audit output and flagged for Mastermind.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing — read-only audit.
  - Simplified or removed: nothing — read-only audit.

- **Critical finding: `logoutFirebase` does not call `auth.signOut()`.** Firebase Auth session persists across "logouts." `auth.currentUser` remains non-null, and the axios interceptor continues to attach authenticated tokens. The user appears logged out (Zustand `user: null`) but backend requests are still authenticated. This is the root cause of multiple downstream issues and must be fixed before any user-lifecycle feature ships on mobile.

- **Critical finding: no 403 `USER_BANNED` interceptor.** A banned user's app continues in a degraded-but-authenticated state for up to ~1 hour (until token auto-refresh fails). No ban-notice dialog exists. No sign-out on 403.

- **Trust-boundary findings (pending verification):**
  - `POST /secure/push/token` sends client-supplied `userId` — backend must be verified to ignore this.
  - `POST /secure/user/update` sends client-supplied `id` and `firebaseUid` — backend must be verified to use `SecurityContextHolder` instead.

- **Type definition gaps for mobile adoption:** `AuthUserDTO` lacks `disabled`, `deletionStatus`, `state`, `wasRegister`. `UserInfoDTO` lacks `state`. Both must be extended before the user-deletion adoption chat.

- **`authStorage.ts` is entirely dead code.** Three exported functions (`setAuthToken`, `getAuthToken`, `clearAuthStorage`) with zero callers. Can be deleted.

- **`updateUserAuth` in `userService.ts` is dead code.** `POST /secure/user/update/auth` — zero callers. Can be deleted.

- **`allowPreferenceCookies` field still sent to backend on user update.** The backend removed this field in the Consent Mode v2 feature. Harmless (backend ignores unknown fields) but misleading.

- **`initAuthListener` has no single-flight guard.** Same class of bug as the web's `UseTokenRefresh` pre-fix double-emission (issues.md 2026-05-21). Firebase may emit multiple `onAuthStateChanged` events on cold start, causing duplicate `firebase-sync` POSTs.

- **Recommended prerequisite ordering for mobile user-deletion adoption:**
  1. Fix `logoutFirebase` to call `auth.signOut()` (blocker for everything).
  2. Add global 403 `USER_BANNED` interceptor + ban-notice dialog.
  3. Extend `AuthUserDTO` and `UserInfoDTO` with state/disabled fields.
  4. Add deletion UI (danger zone, reauth, confirmation dialog, post-deletion dialog).
  5. Add `deletionInFlight` flag and single-flight guard on `initAuthListener`.
