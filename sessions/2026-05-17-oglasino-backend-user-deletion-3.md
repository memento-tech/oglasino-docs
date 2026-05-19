# Session — User deletion (backend), Phase 6

Date: 2026-05-17
Repo: oglasino-backend
Branch: dev
Feature: user-deletion
Phase: 6 (Auth filter + controllers)
Slug-sequence: 3rd session for this slug

## Status

**Phase 6 complete.** Remaining work: Phase 7 (scheduled jobs) and Phase 8 (tests).

## Implemented

- **Extended `AuthenticatedUserDTO`** (the Redis-backed `redisUserAuth` payload) with
  `deletionStatus`. The Hibernate JPQL constructor projection in
  `UserRepository.findAuthDataByFirebaseUid` now selects `u.deletionStatus` so the auth filter
  can see deletion state without hitting the User entity on every request. Old serialised
  entries deserialise with `null` — treated as `ACTIVE` by the equality check, so no eviction
  step is required at deploy.
- **Extended `AuthUserDTO`** (firebase-sync response) and `UserInfoDTO` (public user-info
  payload) with the new spec §8.7 fields: `disabled`, `banReason`, `deletionStatus`,
  `scheduledDeletionAt` on the auth DTO; `state` (new public-facing `UserState` enum:
  `ACTIVE`/`PENDING_DELETION`) and `scheduledDeletionAt` on the info DTO. `UserInfoProjection`
  and its query were extended to source `state` from `u.deletionStatus` and pull
  `scheduledDeletionAt` from `user_deletion_requests` via subquery (only the `PENDING` row).
- **`FirebaseAuthFilter` (spec §10)** — after the existing `getCachedAuthData` lookup, added
  two short-circuits and a credentials stash. Banned (`disabled=true`) ⇒ HTTP 403 with
  `USER_BANNED` envelope written inline (no `filterChain.doFilter`). `PENDING_DELETION` ⇒
  reload User, call `userDeletionService.cancelDeletionOnLogin`, set `X-Account-Restored:
  true` header. After the auth object is built, the verified `FirebaseToken` is stashed via
  `authentication.setCredentials(decoded)` so the delete-account controller can read
  `auth_time` without re-verifying. `OglasinoAuthentication` was extended with a mutable
  `credentials` slot (it previously returned `null` from `getCredentials()`).
- **`AuthController.firebaseSync` (spec §10.1)** — verifies the token once in the controller
  to read the email for the ban check (then re-verified internally by `getOrCreateUser`, which
  is cheap), then four-stage flow: email-banned → 403 `EMAIL_BANNED` + best-effort Firebase
  user delete; `getOrCreateUser` as today; disabled → 403 `USER_BANNED`; pending-deletion →
  cancelDeletionOnLogin + `X-Account-Restored: true` header. ModelMapper auto-maps the new
  AuthUserDTO fields by name from the User entity.
- **`POST /api/secure/user/me/delete` (spec §10.2)** — new handler in `UserController`. Reads
  the stashed `FirebaseToken` from the auth principal's credentials; reads `auth_time` from
  the verified claims; compares against `user.deletion.reauth.max.age.seconds` config (300s).
  Stale or missing `auth_time` → `ReauthRequiredException` → 403 `REAUTH_REQUIRED`. Loads
  User by `auth.getUserId()` (never from request body), invokes
  `userDeletionService.requestDeletion`, returns `DeletionRequestResultDTO`
  ({scheduledDeletionAt}).
- **`POST /api/secure/admin/users/{targetUserId}/force-delete` (spec §3.6 / §8.1)** — new
  handler in the admin `UsersController`. Calls `userDeletionService.forceDeleteByAdmin`
  with the optional reason from the request body (placeholder fallback when omitted).
  Admin-gated by the class-level `@PreAuthorize("hasRole('ADMIN')")` already present on the
  controller; bypasses the facade since there is no orchestration beyond the single service
  call.
- **`/api/secure/user/phoneNumber` (spec §14.7)** — appended
  `AND u.deletionStatus = ACTIVE` to `UserRepository.getUserPhoneNumberAllowed`. Pending-
  deletion users now return empty Optional → null on the wire, which the frontend already
  treats as "phone not available."

## Files touched

| File | Δ | Notes |
|------|---|-------|
| `security/auth/AuthenticatedUserDTO.java` | +9 net | added `DeletionStatus deletionStatus` record component + javadoc |
| `security/auth/OglasinoAuthentication.java` | +14 net | `credentials` slot (replaces hardcoded `null`) |
| `security/filter/FirebaseAuthFilter.java` | +49 net | disabled short-circuit, pending-deletion auto-restore branch, credentials stash, inline 403 envelope helper |
| `ApplicationConfig.java` | +8 net | filter bean wiring updated (2 new deps) |
| `dto/AuthUserDTO.java` | +47 net | 4 new fields + getters/setters + javadoc |
| `dto/UserInfoDTO.java` | +18 net | `UserState state`, `scheduledDeletionAt` + accessors |
| `dto/DeletionRequestResultDTO.java` | +10 (new) | wire-shape record |
| `admin/dto/AdminForceDeleteRequest.java` | +9 (new) | wire-shape record |
| `entity/UserState.java` | +12 (new) | public-facing enum (distinct from internal `DeletionStatus`) |
| `repository/UserRepository.java` | +17 net | `findAuthDataByFirebaseUid` + `findUserInfoById` + `findUserInfoByFirebaseUid` extended; `getUserPhoneNumberAllowed` adds `ACTIVE` filter |
| `repository/projections/UserInfoProjection.java` | +14 net | `getDeletionStatus`, `getScheduledDeletionAt` |
| `service/impl/DefaultUserService.java` | +10 net | `mapProjectionToUserInfo` sets `state` + `scheduledDeletionAt` |
| `controller/AuthController.java` | +71 net | new ban/restore flow in `firebaseSync` |
| `controller/UserController.java` | +52 net | `/me/delete` endpoint |
| `admin/controller/UsersController.java` | +28 net | `/{targetUserId}/force-delete` endpoint |

Totals: ~360 net lines added; 3 new files.

## Tests

`./mvnw spotless:check` — pass.
`./mvnw test` — **360 tests, 0 failures, 0 errors.**

No new tests added in this phase. Phase 8 is the dedicated test phase.

## Cleanup performed

None needed — no commented-out code, no unused imports introduced (one transient import was
removed during the FirebaseAuthFilter edit), no debug logging, no TODO/FIXME left behind.

## Obsoleted by this session

Nothing.

## Brief vs reality

Read the brief and the relevant code paths before implementing. No new blocking discrepancies
beyond what was raised in the Phase 5 summary (and resolved by Mastermind there). One small
shape difference between brief and existing code:

1. **Force-delete path variable name**
   - Brief says: `@PathVariable Long userId` at `/{userId}/force-delete`.
   - Code says: existing admin endpoints in `UsersController` use `@PathVariable Long
     targetUserId`.
   - Why this matters: trivial — Spring binds by name in the URL template, not by the Java
     identifier. But the existing controller is consistent on `targetUserId`, so I matched
     it. Not a correctness issue.
   - Recommended resolution: keep `targetUserId` for consistency with `disableUser` /
     `enableUser` siblings. Done.

Did not stop for this — it is a naming preference, not a code/contract mismatch.

## Known gaps / TODOs

- **`auth_time` source for Postman testing path.** The Postman bypass in `FirebaseAuthFilter`
  does not stash a token, so `auth.getCredentials()` is null. The delete-account endpoint
  treats this as "not authenticated for re-auth" and throws `ReauthRequiredException`. That
  is correct for prod paths, but it means Postman tests cannot exercise the delete-account
  endpoint end-to-end. Phase 8 tests will need to either hit the real auth path or stash a
  synthetic `FirebaseToken` mock in the test setup. Flagged so the Phase 8 author plans
  accordingly.
- **Double `verifyIdToken` on `firebase-sync`.** The controller verifies once (to read email
  for the ban check); `getOrCreateUser` verifies again internally. Firebase Admin caches
  Google's public keys so the second verify is JWT parsing + signature check only — measured
  in microseconds. Documented inline in the controller. If this ever shows up in a perf
  trace, the fix is a new `getOrCreateUser(LoginRequest, FirebaseToken)` overload; not worth
  the surface-area change pre-prod.
- **AuthUserDTO `scheduledDeletionAt` is never set** by any current code path. On firebase-
  sync the auto-restore branch flips the user back to ACTIVE before ModelMapper runs, so the
  field is always null on the wire. That's correct per the "null unless PENDING_DELETION"
  contract on the DTO. Other endpoints that return `AuthUserDTO` (none today) would need to
  populate it explicitly if they did not auto-restore. Phase 6 brief does not introduce any
  such endpoint.

## For Mastermind

1. **`cancelDeletionOnLogin` is called from both `FirebaseAuthFilter` and
   `AuthController.firebaseSync`.** They serve different paths — the filter handles every
   authenticated request after sync, the controller handles the sync handshake itself. The
   filter skips `firebase-sync` (`request.getServletPath().contains("firebase-sync")` guard
   on line 79), so they don't both fire on a single request. However, the FIRST request
   after sync will still trigger the filter's pending-deletion check if the freshly-synced
   user was in PENDING_DELETION — except by then the controller has already restored them,
   so the filter sees ACTIVE in cache (the controller's `userService.saveUser` call inside
   `cancelDeletionOnLogin` evicts the `redisUserAuth` cache entry). Net: idempotent and
   correct, but the double-call window depends on cache eviction firing before the next
   request. Worth verifying in Phase 8 integration tests.

2. **`@PreAuthorize("hasRole('ADMIN')")` on the admin force-delete endpoint** is at the
   class level, inherited from the existing `UsersController`. The brief proposed
   per-method annotation but the codebase convention is class-level. I matched the
   codebase. The brief left this as "or whatever the existing admin auth annotation is" so
   this is implementation latitude not a deviation.

3. **`/api/secure/user/phoneNumber` query still has a likely-inverted boolean.** The query
   selects `WHERE u.allowPhoneCalling = false` — the field name reads as "user allows phone
   calling" so the filter should be `= true`. I noted this in the Phase 2 summary and the
   Phase 5 summary as an adjacent observation; not fixed because the brief's Phase 6.6
   explicitly scopes the change to appending the `deletionStatus` filter. Carry-forward: if
   the bug is confirmed, a one-line fix changing `false` → `true` is enough. Phase 8 tests
   will surface the actual semantics.

4. **Phase 6.1 backward-compat for old `AuthenticatedUserDTO` cache entries.** Serialised
   payloads written before `deletionStatus` was added will deserialise with `null`. The new
   equality check `authData.deletionStatus() == DeletionStatus.PENDING_DELETION` returns
   false on null, so old entries flow through as if `ACTIVE` — safe. No cache flush
   required at deploy. (Phase 6.1 was completed in Session 3 before context compaction; the
   point still holds.)

5. **Trust-boundary audit (spec §10 / Conventions Part 11):**
   - `POST /api/secure/user/me/delete` — userId comes from `auth.getUserId()`. No path
     variable, no body field, no header. `auth_time` is read from the Firebase-signed
     token's claims (stashed by the filter, originally produced by Firebase). CHECK.
   - `POST /api/secure/admin/users/{targetUserId}/force-delete` — `targetUserId` is a
     path variable. Class-level `@PreAuthorize("hasRole('ADMIN')")` gates entry. An admin
     is authorised to act on any user, so accepting userId from the path is the correct
     boundary here. CHECK. (The other three admin path-userId endpoints —
     `disableUser`, `enableUser`, `getUserDetails` — follow the same pattern.)
   - `POST /api/auth/firebase-sync` — email comes from the verified `FirebaseToken`, not
     from the request body. UID likewise. CHECK.
   - `FirebaseAuthFilter` disabled + pending-deletion checks — both read from
     `AuthenticatedUserDTO` which is populated from a JPQL projection of the DB row, never
     from request headers/body. CHECK.
   - No CRITICAL flag.

## Conventions check

- **Part 4 (cleanliness)** — confirmed. No dead code, no TODO/FIXME, no debug println.
- **Part 4a (simplicity)** — confirmed. One judgment call: I bypassed the admin facade for
  `force-delete` because the operation is a single service call with no orchestration. The
  alternative (a facade pass-through method) would be ceremony. Discussed in "For
  Mastermind" §2 above.
- **Part 4b (adjacent observations)** — flagged in "For Mastermind" §3 (the inverted
  `allowPhoneCalling` filter).
- **Part 5 (session summary)** — this file + a copy at `last-session.md`.
- **Part 6 (translations)** — no new translation rows in this phase; Phase 4 covered the
  user-deletion namespace.
- **Part 7 (error contract)** — confirmed. New error codes (`USER_BANNED`, `EMAIL_BANNED`,
  `REAUTH_REQUIRED` via `ReauthRequiredException`) emit the unified
  `{errors:[{field, code, translationKey}]}` envelope, either through
  `GlobalExceptionHandler.handleUserDeletion` (for the exception path) or through inline
  helpers in `FirebaseAuthFilter` / `AuthController` (for filter-level and pre-handler
  short-circuits where exception-throwing doesn't fit the flow).
- **Part 11 (trust boundaries)** — audited above. CHECK across all four new/modified
  endpoints.

## Config-file impact

No required edits to `conventions.md`, `decisions.md`, `state.md`, or `issues.md` from this
session. Carry-forward items from earlier sessions (banned-word seed file split, TLS
constraint, etc.) are still owned by their original session summaries.

## Next checkpoint

**Phase 7 — Scheduled jobs.** Implement `UserDeletionScheduledJobs` per spec §9 with the
three cron-driven methods (`processScheduledDeletions`, `reconcileFirebaseUsers`,
`purgeExpiredAuditRecords`), plus yaml-level cron expressions in `application*.yaml`. Then
**Phase 8 — Tests.** No code blockers.
