# Session summary

**Repo:** oglasino-backend
**Branch:** dev (the brief named `feature/user-deletion`; see "For Mastermind" — I did not switch branches per CLAUDE.md hard rule and verified all user-deletion files are present on `dev` as modified/untracked entries)
**Date:** 2026-05-18
**Task:** Read-only audit of the user-deletion auth lifecycle contract — answer Q-1 through Q-6 in `.agent/audit-user-deletion-auth-lifecycle.md` to verify the contract's assumptions against the on-disk backend code before Mastermind finalises it.

## Implemented

- Wrote `.agent/audit-user-deletion-auth-lifecycle.md` (~ 9 sections). Traced `FirebaseAuthFilter`, `SecurityConfig`, `OglasinoAuthentication`, `AuthController.firebaseSync`, `UserController.deleteMyAccount`, `UsersController.forceDeleteUser`, `ProductSearchController`, `DefaultUserDeletionService.requestDeletion`/`cancelDeletionOnLogin`, `DefaultUserService.saveUser`, `DefaultFirebaseAuthService.getCachedAuthData`, and the projection JPQL on `UserRepository.findAuthDataByFirebaseUid`.
- Verdict on each clause the contract draft turns on: **C-3 implementable as drafted** (Q-1). **`/api/public/**` is not exempt; filter runs on every path except `firebase-sync`/OPTIONS/Postman-bypass/blank-token** (Q-2). **`requestDeletion` evicts via `saveUser` cleanly; the 2026-05-18 race was *not* a cache bug** (Q-3). **`firebase-sync` handler is self-contained and correct under C-9** (Q-4). **Two production callers of `cancelDeletionOnLogin` today; the service is fully idempotent on already-ACTIVE users** (Q-5). **Test deltas isolated to two delete + one rewrite in `FirebaseAuthFilterTest`; `AuthControllerFirebaseSyncTest` and `DefaultUserDeletionServiceTest` unchanged** (Q-6).
- Surfaced two adjacent observations (low severity), neither blocking C-3: (A) filter exception-branch suppresses 401 only for `/api/public`, not all `permitAll` paths; (B) the `cancelDeletionOnLoginIsCalledTwiceWhenTwoRequestsRaceTheSameStaleCache` test's framing describes the wrong race shape vs the 2026-05-18 incident.
- No code changes; no test changes; no spec or contract amendments.

## Files touched

- `.agent/audit-user-deletion-auth-lifecycle.md` (new, +373 / -0)
- `.agent/2026-05-18-oglasino-backend-user-deletion-auth-lifecycle-1.md` (new, this file)
- `.agent/last-session.md` (overwritten with this file's content)

No `src/` files touched. No `pom.xml`, no SQL, no YAML.

## Tests

- Ran (baseline only): `./mvnw spotless:check`, `./mvnw test`.
- Result: spotless clean, **444 tests passed, 0 failures, 0 errors, 0 skipped**, BUILD SUCCESS.
- New tests added: none (read-only audit).
- Post-session test run not required since nothing changed.

## Cleanup performed

- none needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (User Deletion is `backend-stable` per the 2026-05-18 entry, and this audit does not alter status. The contract draft remains a draft; engineering briefs are downstream.)
- issues.md: no change. (The two adjacent observations are low-severity; flagged in "For Mastermind" rather than as `issues.md` entries — Mastermind's call whether to promote.)

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed. No new files in `src/`; no commented-out code; no TODOs added; `spotless:check` and `./mvnw test` ran green at session start.
- Part 4a (simplicity): N/A — no code written.
- Part 4b (adjacent observations): two observations flagged below.
- Part 5 (session summary template): all required sections present, including Obsoleted, Conventions check, and Config-file impact with explicit values.
- Part 11 (trust boundaries): explicitly checked. The audit confirms `OglasinoAuthentication` in `SecurityContextHolder` is the trust authority on `userId`, `userRole`, `subscriptionType`, `subscriptionActive`, `baseSiteId`, and `preferredLanguage`. C-3 preserves this; the change is to *not* populate the context when the user is PENDING_DELETION, which is the right outcome (a request mid-deletion has no legitimate authenticated identity).

## Known gaps / TODOs

- none.

## For Mastermind

### Branch mismatch (process)

The brief says "**Branch:** `feature/user-deletion` (checkout if not already there)". The repo is on `dev`, and CLAUDE.md hard rule #1 forbids me from `git checkout` to a different branch. I verified all user-deletion code is present on `dev` as modified + untracked entries in the working tree (see `git status` — `SecurityConfig.java`, `FirebaseAuthFilter.java`, `AuthController.java`, `UserController.java`, `DefaultUserDeletionService.java` are all `M` or `??` against `dev`). The audit was performed against on-disk state. If `feature/user-deletion` is meant to be a separate branch with possibly different content, please re-issue the brief with explicit guidance on which branch the engineer should read; I cannot do this lookup without a `git checkout`. If the convention is now "user-deletion work lives on `dev`," consider updating the brief template's "Branch:" field to match — this is a small ambiguity that the next audit/engineer session will hit too.

### Adjacent observations (Part 4b)

1. **Filter's 401-on-exception suppression is `/api/public`-only, not all `permitAll` paths.** `FirebaseAuthFilter.java:125` — the exception branch returns 401 unless `path.startsWith("/api/public")`. But `SecurityConfig.java:77` also permits `/api/auth/**` and `/internal/**`. A malformed token on `/api/auth/firebase/{id}` (a `GET` lookup endpoint at `AuthController.java:112-119`) currently returns 401 even though the URL rule would have served it. Severity: low. Recommendation: when implementing C-3, broaden the exception-branch suppression to match the `permitAll` set, or simplify to "always drop context, let URL rules decide" — same shape C-3 is moving the PENDING_DELETION branch toward. I did not fix this because it is out of scope for an audit.

2. **`FirebaseAuthFilterTest.cancelDeletionOnLoginIsCalledTwiceWhenTwoRequestsRaceTheSameStaleCache` (line 140-168) describes the wrong race in its inline comment.** The test's rationale at lines 141-145 frames the race as "two near-simultaneous requests sharing a stale PENDING_DELETION snapshot **before** the first one's eviction propagates." The 2026-05-18 SSR race is structurally different: the eviction had already happened (per the 2026-05-18 logs showing a cache-miss DB hit), and the cache was correctly populated with PENDING_DELETION when the SSR call arrived — the bug is the filter's state-mutating *response* to seeing PENDING_DELETION, not a stale-cache window. Severity: low (documentation/comment only). The whole test is being deleted under C-3 so this is mostly historical, but worth noting in case the engineering brief reuses the comment text. I did not fix this because the test is going away anyway.

### Confirmation of contract-draft assumptions (summary)

The audit verifies, against on-disk code:

- C-1 (Postgres is the sole authority): consistent with current code — `deletion_status` column + `user_deletion_requests` row govern; `redisUserAuth` is a derived cache populated via the JPQL projection at `UserRepository.java:46-65`.
- C-2 (restoration is sign-in only): no current code path independently invents a restoration trigger; the two callers of `cancelDeletionOnLogin` are explicit.
- C-3 (filter drops context on PENDING_DELETION): **implementable as drafted.** Spring's authorization chain handles the downstream behaviour cleanly. No additional principal needs to be constructed; the existing chain's anonymous-authentication handles the missing principal. Precedent for the "drop context, let public paths serve" pattern is already in the filter's exception branch (FirebaseAuthFilter.java:122-129).
- C-4 (filter still 403s on disabled): unchanged; preserved verbatim.
- C-9 (`firebase-sync` retains its restore responsibility): unchanged; the handler is structurally self-contained and matches spec §10.1 step-for-step.
- C-10 (refresh-token revocation as courtesy): verified at `DefaultUserDeletionService.java:156-158` — revocation is called inside the requestDeletion tx, so the 1-hour token-validity window is real but bounded.

### Suggested next steps for the engineering brief

- Single-file change in `FirebaseAuthFilter.java`: remove lines 100-104 (the PENDING_DELETION mutation), and either (a) keep `SecurityContextHolder` untouched and `return` early to fall through anonymous, or (b) explicitly clear context to mirror the existing exception branch. (a) is simpler; (b) is more explicit. The engineer's call.
- Test deltas: delete two tests, rewrite one, add one (named in the audit's Q-6 section).
- Spec amendments to `user-deletion.md` §10 step 4 per contract §10.
- Suggested adjacent fix-in-passing: broaden the 401-suppression to all `permitAll` paths (observation A) — small enough to fold in if Mastermind agrees.

### Drafted config-file text

None required this session.
