# User Deletion — PR Review

**Date:** 2026-05-18
**Branch:** `dev` (spec names `feature/user-deletion` — see Spec-vs-code §1)
**Files reviewed:** 75 (new: 39, modified: 36, deleted: 0)
**Findings:** 18 (critical: 2, high: 3, medium: 6, low: 7)

## Verdict

**REQUEST CHANGES.** The feature is structurally sound: trust boundaries are clean, the
cache-eviction invariant from the 2026-05-14 connection-pool decision is honoured everywhere
auth-relevant state mutates, tests are comprehensive (~78 new tests, all 441 pass), and
spotless is green. The blocking issues are two race / FK windows that turn a hard-delete
into either a data-loss event or an aborted transaction that the cron will keep retrying:

1. **The scheduled cron does not re-verify deletion state inside the hard-delete
   transaction.** A user who signs in after the cron's page load but before the row is
   processed will be restored in the DB, but the cron has already captured a stale
   `User` object with `deletion_status = PENDING_DELETION` and proceeds to hard-delete
   them.
2. **Reviews authored by the deleting user with `approved = false`** (i.e. the
   "rejected by admin" branch) are neither anonymized, deleted, nor cascaded. The FK
   `review.reviewer_id → users.id` is `ON DELETE NO ACTION`, so the user delete at
   step 12 raises a constraint violation and the entire hard-delete transaction
   rolls back. Re-runs of the cron will fail every time for the same user.

Both are fixable in one short engineer session. Cheapest failure mode if shipped today:
the second user with a disapproved review who reaches Day 7 produces a hard-delete that
hangs in the cron forever — log noise, no data loss but no progress either. The first
issue is harder to detect in the wild but produces silent data loss if it fires.

## Findings by severity

### Critical

#### C-1. Cron does not check deletion state before hard-delete; restore race deletes ACTIVE users
- **File:** `src/main/java/com/memento/tech/oglasino/jobs/UserDeletionScheduledJobs.java:78-89`
- **What's wrong:** The cron loads a page of `UserDeletionRequest` rows where `status =
  PENDING`, then iterates. For each row it calls `userRepository.findById(req.getUserId())`
  and unconditionally invokes `userDeletionService.executeScheduledDeletion(user)`. There
  is no check that `user.getDeletionStatus() == PENDING_DELETION` or that the request
  row's status is still `PENDING` at hard-delete time. A user who restores via login
  between the page-load and the per-row call (i.e. while the cron is still inside its
  for-loop) will already have `deletion_status = ACTIVE` and `user_deletion_requests.status
  = CANCELLED` in the DB — the cron will hard-delete them anyway.
- **Evidence — code:**
  ```java
  User user = userRepository.findById(req.getUserId()).orElse(null);
  if (user == null) { ... }
  userDeletionService.executeScheduledDeletion(user);
  ```
  And `executeScheduledDeletion` → `runHardDelete(user, TRIGGER_USER_REQUEST)` performs
  no precondition check on `user.getDeletionStatus()` (`DefaultUserDeletionService.java:255`).
- **Evidence — spec §8.1:** "Phase 3 preconditions: user.deletion_status ==
  'PENDING_DELETION'. user_deletion_requests.scheduled_deletion_at <= now. No active
  deletion lock for this user." The first precondition is not enforced.
- **Fix shape:** Inside `runHardDelete`'s `writeTx.executeWithoutResult` block, re-read
  the user and request rows and abort cleanly if either is no longer pending — either by
  re-fetching with a SELECT FOR UPDATE on the request row, or by adding a
  `deletionRequestRepository.findByUserId(userId)` check on `status = PENDING` at the top
  of the transaction. Symmetrically, `forceDeleteByAdmin` and `markForImmediateDeletion`
  also reach `runHardDelete` without a deletion-state precondition; for the admin path
  this is acceptable (admin override), but the firebase-cascade path could benefit from
  the same defence.

#### C-2. Disapproved reviews authored by deleted user are not handled; FK blocks user delete
- **File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionService.java:284-286`
- **What's wrong:** Three review-cascade statements run in `runHardDelete`:
  - `updateReviewerIdToNullForApproved(userId)` — only `approved = true`
  - `deleteByReviewerIdAndPending(userId)` — only `approved IS NULL`
  - `deleteByTargetUserId(userId)` — all reviews about the user
  Reviews where the deleting user is the reviewer **and** `approved = false`
  (disapproved by admin via
  `DefaultAdminReviewService.disapproveReview` at line 61) are left untouched. The FK
  `review.reviewer_id → users(id)` is plain `FOREIGN KEY ... REFERENCES public.users(id)`
  with no `ON DELETE` clause (`V1__init_schema.sql:1351`), so the cascade default is
  `NO ACTION` — `userRepository.delete(user)` at step 12 will fail with a referential-
  integrity violation, the transaction rolls back, the cron logs `Deletion failed for
  user ...; will retry next run`, and the next run hits the same wall.
- **Evidence — spec §3.3:** explicitly lists the three review cases (approved authored,
  pending authored, about user). Disapproved authored is not in the matrix at §5 either.
  The spec is silent on this third state. The code reflects the spec; the spec is
  incomplete.
- **Fix shape:** Decide the policy (delete disapproved reviews vs anonymize them via
  reviewer = NULL), add the corresponding statement to `runHardDelete`, and update
  spec §3.3 / §5 so this branch is documented. Quickest fix that matches the existing
  philosophy ("approved survives, everything else goes") is to delete the disapproved row
  in the same DELETE statement family — `DELETE FROM review WHERE reviewer_id = :userId
  AND approved = false`. The `reviewer_id` FK should also be considered for an
  `ON DELETE SET NULL` change as a belt-and-braces measure, but that's a wider blast
  radius — primary fix is the missing SQL.

### High

#### H-1. Audit-log `triggered_by` is inherited, not overwritten, when admin force-delete runs over a user-initiated pending request
- **File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionService.java:381-404` (`ensureAuditLogRow`)
- **What's wrong:** `ensureAuditLogRow` looks up an existing open audit-log row by
  `user_id_hash` (no other filter) and returns it if found. So:
  1. User runs `requestDeletion` → audit row inserted with `triggered_by = 'user_request'`.
  2. Admin runs `forceDeleteByAdmin` before Day 7.
  3. `ensureAuditLogRow` finds the existing row and returns its id.
  4. The bestEffort post-commit `auditLogRepository.findById(...).ifPresent(...)` updates
     only `actual_deletion_at` and `was_banned` — `triggered_by` stays `'user_request'`.

  The audit trail records the admin action as if the user did it themselves. Same
  shape applies to the firebase-cascade path running over an existing pending request:
  `triggered_by` stays `'user_request'` instead of becoming `'firebase_cascade'`.
- **Evidence — spec §8.1:** explicitly states `triggered_by = 'admin_action'` for
  `forceDeleteByAdmin` and `triggered_by = 'firebase_cascade'` for
  `markForImmediateDeletion`. The audit row is the only durable record of *which path*
  triggered the deletion — compliance hinges on it being correct.
- **Fix shape:** In the bestEffort `actual_deletion_at` update at lines 338-348,
  also `audit.setTriggeredBy(triggeredBy)` when `triggeredBy != audit.getTriggeredBy()`.
  Alternatively, change `ensureAuditLogRow` to insert a *new* row when the triggers
  don't match — but that produces multiple audit rows per user, complicating the
  retention story. Overwriting on the close step is the smaller change.

#### H-2. Audit-purge can erase the audit row before `actual_deletion_at` is written
- **File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionService.java:338-348` and `src/main/java/com/memento/tech/oglasino/jobs/UserDeletionScheduledJobs.java:170-172`
- **What's wrong:** The audit-log's `actual_deletion_at` is written inside
  `bestEffort(...)` *after* the DB transaction commits. If the audit save throws
  (transient DB outage, connection pool exhaustion), the user is hard-deleted from
  Postgres but the audit row is never finalised. Meanwhile, `retention_until` was set
  at request time to `scheduledDeletionAt + retention.normal.days` (default 30) — so
  the weekly audit-purge job (`purgeExpiredAuditRecords`) deletes the row at
  `retention_until` with `actual_deletion_at` still null. The compliance trail then
  shows "deletion was scheduled but never completed" for a user who was in fact
  hard-deleted.
- **Evidence:** `runHardDelete` step "Audit log actual_deletion_at" wrapped in
  `bestEffort`, comment: "best-effort post-commit". `deleteExpired` query:
  `DELETE FROM UserDeletionAuditLog l WHERE l.retentionUntil <= :now`.
- **Fix shape:** Move the `actual_deletion_at` write *inside* the transaction (drop the
  `bestEffort`) — this couples the audit row to the user-delete commit so they succeed
  or fail together. If the engineer wants to keep the bestEffort shape, the audit-purge
  query needs to exclude rows with `actual_deletion_at IS NULL` so unfinished rows
  aren't reaped. Either fix is one line.

#### H-3. Firestore notification deletion happens *during* the JPA transaction; failure leaves orphans + user still in DB
- **File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionService.java:298-300`
- **What's wrong:** Step 11 of the transactional block calls
  `firestoreUserService.deleteUserNotifications(firebaseUid)`. The Firestore service
  catches its own exceptions (`DefaultFirestoreUserService.java:49-54`) so a Firestore
  failure does not roll back the DB transaction — *but* the Firestore deletion has
  already deleted some notifications by the time the failure happens, and even on
  success the Firestore write happens before step 12 (`userRepository.delete(user)`).
  If step 12 fails (the C-2 FK case, or any other DB issue), the user row is still in
  the DB, but the user's Firestore notifications are gone.
- **Evidence:** `DefaultUserDeletionService.java:298-300`,
  `DefaultFirestoreUserService.java:39-54` (batch loop runs even if the second batch
  throws).
- **Fix shape:** Move the Firestore call into the post-commit `bestEffort` block
  (where R2 and Firebase Auth deletes already live). Notifications outliving a
  half-rolled-back hard-delete is a worse outcome than the rare orphan-on-DB-success.

### Medium

#### M-1. Spec-vs-code: deletion cron knobs live in `application.yaml`, not the `Configuration` table
- **Files:** `src/main/resources/application-{dev,prod,stage}.yaml`,
  `src/main/resources/data/configuration/data-configuration.sql:73-86`
- **What's wrong:** Spec §3.9 / §7 states "All user-deletion knobs live in the
  `Configuration` table … not in `application.yml`." The implementation puts the three
  cron expressions (`user.deletion.hard.delete.cron`,
  `user.deletion.audit.purge.cron`, `user.deletion.firebase.reconciliation.cron`) in
  yaml and the non-cron knobs in the DB. The engineer correctly reasoned (comments in
  both files) that `@Scheduled(cron = "${…}")` resolves at bean-construction from the
  Spring `Environment` and cannot read from the runtime `Configuration` cache. The
  code is right; the spec needs updating.
- **Fix shape:** Spec correction, not a code change. Flagged for Docs/QA in
  "For Mastermind."

#### M-2. `DeletionRequestStatus` and `CHECK` constraint use uppercase; spec uses lowercase
- **Files:** `src/main/java/com/memento/tech/oglasino/entity/DeletionRequestStatus.java`,
  `src/main/resources/db/migration/V1__init_schema.sql:1853`
- **What's wrong:** Spec §6.4 specifies `status IN ('pending', 'cancelled',
  'completed')`. Implementation stores `'PENDING' | 'CANCELLED' | 'COMPLETED'`
  (JPA `EnumType.STRING` matches the constant name). CHECK constraint and the
  `idx_udr_pending_due` partial-index predicate both use uppercase. Internally
  consistent, but it diverges from spec — could mislead anyone querying the table
  manually.
- **Fix shape:** Spec correction. Implementation is correct.

#### M-3. Postman testing path silently breaks the delete-account endpoint
- **File:** `src/main/java/com/memento/tech/oglasino/controller/UserController.java:58-63`
- **What's wrong:** When the Postman bypass is active (`X-Postman: true` plus dev
  profile), `FirebaseAuthFilter.validateForPostmanTesting` builds the
  `OglasinoAuthentication` *without* setting credentials. The delete endpoint detects
  null credentials and throws `ReauthRequiredException` (HTTP 403 + `REAUTH_REQUIRED`).
  This is the safe behaviour, but it gives a misleading error code — dev manual tests
  using Postman against `/me/delete` will see "REAUTH_REQUIRED" with no obvious cause.
  Acceptable for shipping (only affects dev manual tests), worth a comment.
- **Fix shape:** Either add a Postman-aware bypass for the freshness check in dev only,
  or improve the existing inline comment so the next dev knows to call the endpoint
  via real Firebase auth, not Postman. Comment-only fix is cheapest.

#### M-4. `requestDeletion` runs `safeRevokeRefreshTokens` *after* commit; failure leaves user in PENDING but with active tokens
- **File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionService.java:152-157`
- **What's wrong:** Spec §4.2 step 5 lists `FirebaseAuth.revokeRefreshTokens(uid)` as
  one of the Day 0 effects. The code performs the revoke *after* the transaction
  commits, in a `try/catch` that just logs. If the Firebase call fails for any
  reason (network blip, Firebase outage, quota), the user is `PENDING_DELETION` in
  Postgres but their existing tokens still work on other devices — they can
  effectively bypass the "existing sessions stop working" property. The auth filter
  *does* reject `disabled = true`, but a deleting user is not banned, only pending.
  So they retain auth.
- **Evidence:** `safeRevokeRefreshTokens` catches all exceptions; the user state is
  already committed.
- **Fix shape:** Either retry the revoke (background retry queue) or include the
  revoke inside the transaction and let the call's failure roll back the deletion
  request — but that has its own downsides (Firebase outage prevents users from
  deleting accounts). The current trade-off is defensible; flag this so future ops
  knows the property is best-effort, not guaranteed.

#### M-5. `markForImmediateDeletion` ignores its `reason` parameter
- **Files:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionService.java:219-239`,
  `src/main/java/com/memento/tech/oglasino/jobs/UserDeletionScheduledJobs.java:141`
- **What's wrong:** The cron passes `"firebase_cascade"` as the reason; the method
  signature accepts it but never uses it. The auto-ban hash is hard-coded to
  `"auto: deletion with open reports"` and the `triggered_by` is hard-coded to
  `TRIGGER_FIREBASE_CASCADE`. Either the parameter should be removed, or the value
  should flow through into the ban reason / audit row.
- **Fix shape:** Remove the parameter from the interface + implementation + caller.
  One-line cleanup. Spec §8.1 signature has `String reason` so spec needs the same
  trim if the parameter is removed.

#### M-6. Cancel-on-login audit-log lookup is "find first matching" — risks tagging the wrong row
- **File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionService.java:190-197`
- **What's wrong:** `cancelDeletionOnLogin` matches the audit-log row to update by
  "first open row by user_id_hash, ordered by created_at desc." Per spec §17.5 a
  user can re-request after a cancellation, creating a *new* audit row per attempt.
  The repository query
  `findFirstByUserIdHashAndActualDeletionAtIsNullAndCancelledAtIsNullOrderByCreatedAtDesc`
  finds the most-recent open row, which is the current pending one. ✓ — but the
  reliance on "most recent open by createdAt desc" is fragile if a future code path
  ever creates two open audit rows. Worth a stronger invariant — e.g., a unique
  partial index on `(user_id_hash) WHERE actual_deletion_at IS NULL AND cancelled_at
  IS NULL`. Same query is used in `ensureAuditLogRow` (H-1).
- **Fix shape:** Add a partial unique index, or stop relying on createdAt ordering by
  joining through `user_deletion_requests.user_id` when possible. Not blocking.

### Low

#### L-1. Spec divergence: spec says `feature/user-deletion` branch; work is on `dev`
- **Severity:** Low. Workflow-level, not code.
- **Fix shape:** None for the engineer. Confirm with Igor whether the branch should
  be rebased into a feature branch before merge.

#### L-2. Untracked `start-session.sh` at repo root
- Looks like a working-tree-only helper script that got staged into the repo root
  alongside the deletion changes. Probably accidental.
- **Fix shape:** Delete or `.gitignore`.

#### L-3. Pre-existing `// TODO pageable and join left with collection is not good...` in `ReviewRepository.java:31`
- Not introduced by this feature; flagged per conventions Part 4b (adjacent
  observation). Out of scope.

#### L-4. `getBaseSiteCode()` ↔ query alias `baseSiteId` mismatch (pre-existing)
- **File:** `src/main/java/com/memento/tech/oglasino/repository/projections/UserInfoProjection.java:23` (pre-existing line; not introduced).
- The interface-projection method `getBaseSiteCode()` does not match either query
  alias (`bs.id AS baseSiteId`) in `UserRepository.findUserInfoById` /
  `findUserInfoByFirebaseUid`. Spring's interface-projection convention matches by
  getter name → alias, so this getter resolves to null in practice. The downstream
  mapping in `DefaultUserService.mapProjectionToUserInfo` already handles a null
  baseSiteCode via `Optional.ofNullable`. Pre-existing, out of scope.

#### L-5. `idx_udr_pending_due` partial index references uppercase status — comment in spec was lowercase
- See M-2. Documentation drift; cosmetic.

#### L-6. `Firestore` user notifications service's batch loop continues past partial failure
- **File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultFirestoreUserService.java:33-46`
- If batch 1 deletes 500 docs successfully and batch 2 throws, the catch at line 49
  swallows it, but the partial state is left in Firestore (some docs deleted, some
  not). Acceptable in the "user is being deleted anyway, orphans are best-effort"
  posture; worth a comment that this is intentional.

#### L-7. `bestEffort` post-commit Audit-log `setWasBanned(ctx.wasBanned)` is redundant
- **File:** `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserDeletionService.java:344-347`
- `wasBanned` is captured at the start of `runHardDelete` from `user.isDisabled()` and
  set on the audit row created at `requestDeletion` time (with the user's then-current
  `disabled` flag). The post-commit step sets it again. If the user got banned
  *between* requestDeletion and Day 7, the post-commit overwrite captures that
  transition. If they were already banned, the value is the same. Worth a one-line
  comment explaining the late-capture intent — or drop the redundant write when the
  in-tx audit row already had the right value.

## Findings by dimension

### 1. Spec conformance
- M-1, M-2, M-5, L-1, L-5.
- Spec is mostly faithfully implemented. The three divergences are the cron-in-yaml
  necessity (spec wrong), uppercase enum storage (spec wording wrong), and the
  missing third review state (C-2 — both sides incomplete).

### 2. Trust boundaries (conventions Part 11)
- **Clean.** User identity flows from `OglasinoAuthentication` (set by
  `FirebaseAuthFilter` after token verify); `auth_time` from the stashed
  `FirebaseToken` credentials slot; admin endpoint gated by class-level
  `@PreAuthorize("hasRole('ADMIN')")`. Delete-account endpoint takes empty body —
  no userId from the client. firebase-sync extracts email from the verified
  token. Ban-hash lookup uses server-controlled hash. No `oldName`-shaped bug
  reappears here. ✓

### 3. Cascade correctness
- C-2 (disapproved authored reviews → FK failure).
- H-3 (Firestore deletion ordering risk).
- M-6 (audit-log open-row lookup fragility).
- Otherwise the cascade order is exactly what spec §8.1 dictates: favorites → reports
  → reviews → user_follow / push_token / suggestion / product_audit → image
  collection → Firestore → user delete → R2 → Firebase → audit closure.

### 4. Cache eviction (per the 2026-05-14 decision)
- **Clean.** Every auth-relevant `User` mutation in this feature routes through
  `DefaultUserService.saveUser`, whose `@Caching` annotation evicts `redisUserInfo`
  (id + firebaseUid) and `redisUserAuth` (firebaseUid). Specifically:
  - `DefaultUserDeletionService.requestDeletion` → line 141
  - `DefaultUserDeletionService.cancelDeletionOnLogin` → line 200
  - `DefaultUserDeletionService.runHardDelete` (favorites clear) → line 278
  - `DefaultUsersFacade.disableUser` / `enableUser` → both call `saveUser`
- A tiny race exists between `saveUser` (which evicts) and the final
  `userRepository.delete(user)` 40 lines later — another request can repopulate the
  cache in that window with a now-doomed user. Firebase Auth deletes the user at step
  14, so subsequent tokens fail verification — the stale cache entry is harmless
  unless someone hits the filter with a valid pre-revocation token in the window.
  Acceptable.

### 5. External system coordination
- **Firebase Auth:** Day 0 revoke is `safeRevokeRefreshTokens` post-commit (M-4 trade-
  off). Day 7 delete is idempotent on `USER_NOT_FOUND`. ✓
- **Firestore:** H-3 ordering concern.
- **Cloudflare R2:** profile + bulk product images deleted post-commit; failures
  fall through to orphan sweeper. ✓ per spec §17.8.
- **Elasticsearch:** product state flip publishes `ProductStateUpdateEvent` per
  product; reindex is async. ✓
- **Redis:** redisUserAuth + redisUserInfo evicted on every mutation. ✓

### 6. Error handling and the wire contract (conventions Part 7)
- **Clean.** New error codes (`REAUTH_REQUIRED`, `USER_BANNED`, `EMAIL_BANNED`,
  `USER_LOCKED_FROM_DELETION`, `USER_NOT_PENDING_DELETION`) live in a new
  exception hierarchy (`UserDeletionException`), not in `ProductErrorCode`.
  Wire shape is the unified `{errors:[{field, code, translationKey}]}` envelope.
  HTTP statuses: 403 for auth-related, 400 for `USER_NOT_PENDING_DELETION`. The
  comment in `UserDeletionException.java:11-13` explicitly references the open
  `issues.md` HIGH entry about `ProductErrorCode` being a dumping ground — this
  feature consciously did not contribute to that smell. ✓

### 7. Code quality (Part 4 + Part 4a)
- spotless:check passes, all 441 tests pass.
- No commented-out code, no `System.out.println`. Logger usage is consistent with
  surrounding code.
- L-2 (`start-session.sh`) and L-3 (pre-existing TODO).
- Comments lean explanatory — `DefaultUserDeletionService.java`'s 6-line preamble on
  `TransactionTemplate` choice is a model of the Part 4a explain-the-non-obvious rule.

### 8. Concurrency and safety
- C-1 (cron does not re-check deletion state).
- Otherwise robust: `existsActiveLockForUser` takes a `:now` parameter (clean,
  re-runnable across DST); idempotent guards in `requestDeletion` and
  `cancelDeletionOnLogin`; restore-vs-ban race acknowledged in spec §17.14 and
  handled.

### 9. Tests
- Coverage: 78 new tests across 10 files (FirebaseAuthFilter 6, UserDeletionService
  21, UserDeletionScheduledJobs 11, AuthController 5, UserController 5, UsersFacade
  4, UsersController force-delete 3, UserAuditService 12, FirestoreUserService 3,
  FirebaseUserService 8).
- Happy paths, idempotency, lock-throws, banned-with-audit, banned-without-audit (the
  defensive fall-through), threshold-trip vs threshold-not-trip, R2 failure tolerance,
  Firebase failure tolerance — all covered.
- **Gaps:**
  - No test for C-1 (the cron-restore race) — would need a thread-coordinated test
    or an integration test that mutates state between page-load and per-row processing.
  - No test for C-2 (disapproved-authored review surviving the cascade and breaking
    the FK).
  - No test for H-1 (admin force-delete inheriting `triggered_by = 'user_request'` from
    a pre-existing audit row).
  - No test for H-3 (Firestore-during-tx ordering).

### 10. Adjacent observations
- See L-3 (pre-existing TODO), L-4 (pre-existing baseSiteCode mismatch). Per
  conventions Part 4b — flagged but out of scope.

## Spec-vs-code divergences

1. **Spec §3.9 / §7:** "All user-deletion knobs live in the `Configuration` table … not
   in `application.yml`." **Code:** Cron knobs in yaml; non-cron knobs in DB.
   **Right side:** Code. Spring `@Scheduled` cannot read cron from a runtime cache.
   **Why:** Engineer's call is technically correct and well-commented.

2. **Spec §6.4:** `status IN ('pending', 'cancelled', 'completed')`. **Code:** Uppercase
   throughout (`PENDING`, `CANCELLED`, `COMPLETED`) — JPA `EnumType.STRING`, CHECK
   constraint, partial-index predicate. **Right side:** Code, on consistency grounds —
   matching the storage to the enum constant name is the JPA idiom. Spec needs
   correcting.

3. **Spec §3.3 / §5:** review-handling matrix lists three cases for reviews authored by
   the deleting user (approved → anonymize, pending → delete, about-user → delete). It
   does **not** address `approved = false` (disapproved). **Code:** Also doesn't
   handle this case. **Right side:** Spec is incomplete; this is C-2.

4. **Spec §8.1:** "Phase 3 preconditions: user.deletion_status == 'PENDING_DELETION'."
   **Code:** Precondition not enforced inside the transaction. **Right side:** Spec.
   This is C-1.

5. **Spec branch:** `feature/user-deletion`. **Code:** Work is on `dev`. **Right side:**
   Workflow choice for Igor.

6. **Spec §8.1 `markForImmediateDeletion(User user, String reason)`:** `reason` is a
   passed-through value. **Code:** Parameter ignored; both `triggered_by` and the
   ban-hash reason are hard-coded. **Right side:** Code, with parameter trim (M-5).

## Adjacent observations

- **`ReviewRepository.java:31`** — pre-existing `// TODO pageable and join left with
  collection is not good...`. Severity low. Out of scope, not fixed.
- **`UserInfoProjection.java:23`** — pre-existing `String getBaseSiteCode()` getter
  whose name does not match either query alias (`bs.id AS baseSiteId`). The downstream
  mapper handles the resulting null, but the projection getter is effectively dead.
  Severity low. Out of scope, not fixed.
- **`start-session.sh`** untracked at repo root, alongside CLAUDE.md modifications.
  Looks accidental. Severity low. Out of scope.
- **`AuthController.firebaseSync`** — `decoded.getEmail()` is guarded by
  `StringUtils.isNoneBlank(email)` before the ban check. If a Firebase provider were
  ever to return a token with a blank email claim, the ban hash would not be checked
  for that user. Normal providers (Google, Email/Password) always populate email, so
  this is a theoretical concern. Severity low. Out of scope.

## What I did not review

- **Runtime behaviour** of the scheduled jobs against a real Postgres + Redis
  combination — judged purely statically. The cron-restore race (C-1) is therefore
  judged by reading the code, not by reproducing it.
- **Firebase Admin SDK** integration behaviour against a live Firebase project — only
  the call shapes and idempotency handling were reviewed.
- **Frontend** (`oglasino-web`) interactions — the spec mentions extensive
  frontend work but that's not in this repo. The wire contracts (DTOs, error codes,
  response headers, X-Account-Restored) were reviewed; the frontend's handling of them
  was not.
- **The `oglasino-router` worker** for any maintenance/admin-bypass interaction with
  the new auth-filter 403 path — not in this repo.
- **Test internals** were skimmed for coverage but not asserted against the
  implementation line-by-line.

## Test results

- `./mvnw spotless:check` — **passed.**
- `./mvnw test` — **passed.** `Tests run: 441, Failures: 0, Errors: 0, Skipped: 0`.
  Output is noisy with expected warnings (Elasticsearch probe failures, language
  detector mismatches, deliberate exception-handling-path tests) — none are
  regressions.
- Coverage notes: see "Tests > Gaps" above. Two of the three blocking findings
  (C-1, C-2) and one of the high findings (H-1) have no dedicated test.

## Notes for Mastermind

- **Blocking issues:** C-1 (cron-restore race) and C-2 (disapproved-authored
  reviews break FK). Both need a backend brief — small in scope, large in blast
  radius if unaddressed. Recommend one combined brief that:
  1. Adds the precondition check inside `runHardDelete`.
  2. Adds the `DELETE FROM review WHERE reviewer_id = :userId AND approved = false`
     statement (or equivalent policy decision — see C-2 fix-shape).
  3. Adds focused tests for both.
- **Spec corrections needed (Docs/QA):**
  - Spec §3.9 / §7: cron knobs cannot live in the `Configuration` table — call this
    out explicitly to avoid the same misunderstanding next time.
  - Spec §6.4: status values are uppercase, matching the JPA enum constants.
  - Spec §3.3 / §5: define the policy for reviews authored by the deleting user with
    `approved = false`.
  - Spec §8.1: either keep the `reason` parameter on `markForImmediateDeletion` and
    wire it through, or remove it from the signature.
- **Branch convention:** spec names `feature/user-deletion`; work is on `dev`.
  Confirm with Igor whether to rebase before merge or update the spec.
- **Strength to call out:** the trust-boundary discipline on this feature is
  excellent — the delete-account endpoint takes an empty body, `auth_time` flows
  through the verified-token credentials slot (not from anything the client can
  craft), the firebase-sync ban check runs on the verified-token email. The
  2026-05-14 cache-eviction invariant is satisfied at every mutation site. These
  are the things that would have produced the most embarrassing bugs if done
  wrong; they were done right.
- **Issues.md entry candidate:** the pre-existing `getBaseSiteCode()` ↔ alias
  mismatch (L-4) is a real bug; the downstream null-tolerance is masking it. Worth a
  HIGH/LOW entry depending on whether anyone relies on baseSiteCode being populated.
