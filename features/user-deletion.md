# User Deletion

**Status:** approved for implementation
**Owner:** Igor (operator), Mastermind (planning), Backend Engineer + Web Engineer (execution)
**Last updated:** 2026-05-19
**Branch:** `feature/user-deletion`
**Affects:** `oglasino-backend`, `oglasino-web`. Mobile (`oglasino-expo`) adopts post-merge in its own Mastermind chat.

**Manual test cases:** see [`user-deletion-test-cases.md`](user-deletion-test-cases.md).

**Depends on / supersedes:**

- Supersedes the engineering-spec draft dated 2026-05-06. Corrections noted inline where applicable.
- Privacy Policy §8 and §9 (the "Account deletion" and "Your rights" sections of `legal/privacy-policy-draft.md`) are authoritative on user-facing behavior. This spec mirrors them and resolves to them on any disagreement.
- Terms of Use §10 (Termination) is authoritative on the user-vs-admin termination distinction. This spec mirrors it.
- Inherits the constraint from `decisions.md` 2026-05-14 (connection-pool incident): any code path that mutates an auth-relevant field on a `User` row must route through `DefaultUserService.saveUser` so the `redisUserAuth` `@CacheEvict` fires. Hard delete must evict both `redisUserAuth` and `redisUserInfo`.
- Folds in the `account-disabling-enforcement` backlog item from `state.md`. The veto plumbing (auth filter reads `disabled`, syncUserToBackend signals it, frontend signs out) lives here.

**Related features (separate):** Image pipeline general (`state.md` backlog) — not affected by this work. Mobile adoption (post-merge).

---

## 1. Overview

Users can delete their own account from the dashboard. Deletion is **soft for 7 days** (auto-restore on login during the window), then **hard** (irreversible). Two parallel timers run from the deletion request: the 7-day grace period (user can restore) and the 7-day reporting window (other users can still report misconduct against the deleting user). Both expire at the same instant on Day 7, at which point hard deletion runs.

This feature also delivers the **banned-user enforcement** half of the platform's auth model. Today, the backend has a `disabled` column on the user row that is populated by `AuthenticatedUserDTO` but never consulted anywhere. This spec wires it up: the auth filter rejects disabled users with 403, the frontend signs them out, Firebase Auth is told to disable them so existing tokens cannot refresh, and a hashed audit record prevents re-registration with the same email for 12 months.

**Banned users cannot self-delete** (they can't sign in, so the in-app delete flow is inaccessible). The right-to-erasure path for banned users is **email to support@oglasino.com**, processed manually by an admin via a backend endpoint. The ban-notice dialog on the frontend explains this path.

### What this feature is

- Self-service account deletion from the user dashboard.
- 7-day grace period with sign-in-to-restore behavior.
- Permanent hard delete after the grace period, including from Firebase Auth.
- Firebase orphan reconciliation: if a user deletes themselves from Firebase directly (e.g., from Google Account settings), the platform detects and cleans up on a weekly sweep.
- Admin postponement of deletion via the `user_deletion_locks` table (legal investigations, internal abuse investigations).
- Admin-initiated ban with reason, Firebase `setDisabled`, refresh-token revocation, ban-hash insertion.
- Banned-user re-registration prevention via SHA-256 email hashing, 12-month retention.
- Frontend ban-notice dialog explaining the situation to disabled users.
- Admin-only force-delete endpoint for processing email-requested deletions from banned users.

### What this feature is not

- It is **not** a data export ("download my data") feature. That is separate and post-launch.
- It is **not** an admin UI for managing deletion locks. The `user_deletion_locks` table is populated via SQL or via admin-only endpoints with no UI in v1; admin UI is post-launch.
- It is **not** an admin-initiated restore mechanism. Once Day 7 hits, the user is gone. Admins cannot un-delete.
- It is **not** an email-driven self-service deletion. Email is the manual ops path for banned users only; active users use the in-app flow.
- It is **not** a notification cleanup pass on other users' inboxes. A deleted user's name may persist in stale notifications that other users still hold; these are ephemeral and turn over naturally. Notification cleanup of _the deleted user's own_ inbox is in scope.
- It is **not** a chat-message rewrite operation. Message documents in Firestore carry only `senderId` (Firebase UID), not display name. Anonymization happens at read time via the new `state` field on `UserInfoDTO`, not via Firestore mutation.

### Out-of-scope work that this feature deliberately defers

See §19 (Future work) for the full list with rationales. Headlines:

- Frontend admin UI for ban-with-reason (the existing button keeps working; the new `banReason` field is optional at the endpoint and populated by ops via direct API or DB).
- Email service infrastructure (privacy@, support@ mailboxes are pre-launch action items).
- Mobile adoption (separate Mastermind chat post-merge).
- Notification sweep across other users' inboxes for mentions of deleted users.
- Data export.

---

## 2. Compliance context

This implementation is designed to meet GDPR's **right to erasure** (Article 17) requirements while preserving necessary audit data under **legitimate interest** (Article 6(1)(f)) and **legal obligation** (Article 6(1)(c)) for safety and fraud prevention.

The retention model is:

- **Active account data**: kept while the account is active.
- **Soft-delete state**: kept for 7 days, then either restored (cancellation by sign-in) or hard-deleted.
- **Hard-deleted account data**: erased from Postgres, Firestore, R2, Firebase Auth, and Elasticsearch.
- **Normal-deletion audit record (`user_deletion_audit_log`)**: hashed (SHA-256) user id + email + deletion metadata, retained 30 days, then purged.
- **Banned-user audit record (`banned_user_audit`)**: hashed (SHA-256) email + ban reason, retained 12 months, then purged.

Hashed retention is permissible under legitimate interest because:

- The data is one-way hashed; the original email cannot be recovered.
- The retention period is proportionate to the abuse-prevention purpose.
- The processing is disclosed in the Privacy Policy (§8.5).
- After the retention period, even the hash is deleted.

Reviews authored by a deleted user are retained _anonymized_ (with `reviewer_id = NULL` and frontend-rendered "Deleted User") because they reflect the experience of the user who was _reviewed_, whose record is partly composed of those reviews. Pending-approval reviews authored by a deleted user are deleted entirely. Reviews _about_ the deleted user are deleted entirely. See §5 (Data inventory) for the full matrix.

**Privacy Policy and Terms amendments required.** This feature requires two small additions to the legal drafts, both for the banned-user-deletion-via-email path:

- **Privacy Policy §9 (Your rights — Right to erasure)**: add a sentence: "If you are unable to use the self-service deletion flow (for example, because your account has been disabled), contact support@oglasino.com to request deletion. We will respond within 30 days as required by GDPR."
- **Terms of Use §10 (Termination)**: add the same path under "appeals and deletion requests for banned accounts."

These edits land _after_ this feature ships, before launch. Logged in §20 (Pre-launch action items).

---

## 3. Policy decisions

This section captures the decisions made during the Mastermind planning chat that the spec depends on. Each is non-negotiable; changing any of them requires reopening the planning chat.

### 3.1 Profile visibility during the grace period

During the 7-day grace period, the deleted user's profile **remains publicly visible** with a "Scheduled for deletion" badge. Phone number is hidden. Listings are hidden (via `ProductState.INACTIVE`). Reviews about the user remain visible. Reviews authored by the user remain visible. The user is signed out and cannot sign in for any purpose other than restoring the account (signing in cancels the deletion).

The original engineering spec said "profile hidden at Day 0." That was wrong against the legal drafts. The legal drafts win; the visible-profile-with-badge model is the policy.

The reasoning: the badge and the 7-day reporting window together form the affordance for other users to report the deleting user. Hiding the profile destroys that affordance.

### 3.2 Messaging during the grace period

During the grace period, the deleting user **cannot send or receive messages**. Other users see the deleting user's chat history with the deleting user's name still rendering normally — there is no "Deleted User" label during the grace period. The chat header for the conversation with the deleting user shows the "Scheduled for deletion" badge (via the `state` field on `UserInfoDTO`). The message input is disabled.

After hard delete, the deleting user's name in any conversation render becomes the translated "Deleted User" placeholder, served by the chat null-safety branch (when `getUserForFirebaseUid` returns null, render `tCommon('user.deleted')`).

### 3.3 Reviews — four-case split

- **Approved reviews authored by the deleting user**: kept, `reviewer_id` set to `NULL` at hard delete. Frontend renders "Deleted User" as the author.
- **Pending-approval reviews authored by the deleting user**: deleted entirely at hard delete (they may have been approved and become visible months later; better to remove uncertainty now).
- **Disapproved reviews authored by the deleting user (admin-rejected)**: deleted entirely at hard delete. An admin actively rejected these; there is no value in retaining anonymized copies. Without this delete, `userRepository.delete(user)` would fail with an FK violation on any disapproved-authored row.
- **All reviews about the deleting user** (any approval state): deleted entirely at hard delete.

Reviews' images: review images live under `public/products/` in R2 (per backend audit §3 and §10). At hard delete, only the _user's own listing images_ are deleted from R2; review images owned by _other users_ — including any that referenced products that don't exist anymore — are left for the existing `ProductImagesRemovalJob` orphan sweeper to clean up. Review images attached to the deleted user's own authored reviews follow the review: if the review is kept (approved, anonymized), the image stays; if the review is deleted (pending, or review-about-deleted-user), the image is dereferenced and gets cleaned up by the orphan sweeper.

### 3.4 Reauth before destructive action

Before the user can submit the delete-account request, they must reauthenticate. Two flows:

- **Email/password users**: Firebase `reauthenticateWithCredential` with their password.
- **Google users** (and Facebook, when enabled): Firebase `reauthenticateWithPopup` with the relevant `AuthProvider`.

After successful reauth, the frontend **must** call `user.getIdToken(true)` (forced refresh). The fresh token is passed explicitly on the delete request to bypass the axios cached-token interceptor (per Section 12.1).

The backend reads the `auth_time` claim from the verified ID token in the delete-account controller. If `now - auth_time > user.deletion.reauth.max.age.seconds` (default 300, i.e., 5 minutes), the request is rejected with HTTP 403 + error code `REAUTH_REQUIRED`.

### 3.5 Banned-user veto model

Admin-initiated bans go through `DefaultUsersFacade.disableUser(userId, banReason)`. The flow:

1. `User.disabled = true`, `User.banReason = <reason>` — persisted via `saveUser`, which evicts both `redisUserAuth` and `redisUserInfo`.
2. `banned_user_audit` row inserted with SHA-256 hash of `lowercase(trim(email))`, ban reason, `banned_at = now()`, `retention_until = now() + 12 months`.
3. `FirebaseAuth.getInstance().updateUser(new UserRecord.UpdateRequest(uid).setDisabled(true))` — Firebase rejects future sign-ins and rejects refresh-token rotation.
4. `FirebaseAuth.getInstance().revokeRefreshTokens(uid)` — forces existing sessions to fail on next token refresh.

On the next request from this user:

- If their refresh token has expired: Firebase rejects token verification. The backend never sees an authenticated request; the frontend's `onIdTokenChanged` fires with null, signs them out, navigation drops them from any protected page, the ban-notice dialog renders.
- If their refresh token hasn't yet hit Firebase: the backend's `FirebaseAuthFilter` reads `authData.disabled()` from the (just-evicted) Redis cache, sees true, returns 403 with body `{errors: [{field: null, code: 'USER_BANNED', translationKey: 'errors.user.banned'}]}`. The axios global 403 interceptor catches this specific code, signs out, sets a sessionStorage flag, navigation completes, the ban-notice dialog renders.

### 3.5a Banned-user content hiding

When a user is banned (`User.disabled = true`), the system mirrors the pending-deletion content-hiding posture (§3.1 / §3.2 / §14.8):

- All their products with `productState = ACTIVE` flip to `INACTIVE` with `deactivated_by_system = true`.
- The public profile endpoint returns 404 (mirrors post-hard-delete privacy posture).
- Messaging is gated on the counterparty side: badge ("Banned"), disabled input, notice text — see §14.8.
- When admin unbans:
  - If `deletionStatus = ACTIVE`: products with `deactivated_by_system = true` flip back to `ACTIVE`.
  - If `deletionStatus = PENDING_DELETION`: products are NOT restored (pending-deletion intent preserved).

The `deactivated_by_system` marker distinguishes "deactivated by ban / by pending-deletion" from "deactivated by user." Only system-deactivated products are restored on unban; user-deactivated products stay user-deactivated.

### 3.6 Banned users delete via support email

A banned user cannot use the self-service deletion flow (they cannot sign in). The right-to-erasure path for them is email to support@oglasino.com. An admin:

1. Verifies the request matches the banned account (e.g., the email is the same).
2. Calls `POST /api/admin/users/{userId}/force-delete` (admin-authenticated, separate endpoint).
3. The backend runs the hard-delete sequence immediately (no grace period — they cannot restore).
4. The `banned_user_audit` hash row remains until its 12-month retention expires, so the user cannot re-register during that window even after deletion.

The ban-notice dialog tells the user how to do this. See §14.

### 3.7 Un-ban removes the hash

When an admin un-bans a user (`DefaultUsersFacade.enableUser`), the `banned_user_audit` row for that user's email hash is deleted. Rationale: if the admin un-bans, they have decided the user is no longer a threat; re-registration should not be blocked post-un-ban.

Unban actions are logged to the `user_admin_action_audit` table with `action = 'UNBAN'`, hashed user id + email, optional reason, acting admin id, performed-at timestamp, and 12-month retention. The audit row is written **before** `banned_user_audit` deletion so a mid-flow failure cannot leave the system without a record of the attempted unban. The same table will host other admin lifecycle actions as they are added; per §9.3 the weekly audit-purge cron sweeps expired rows from this table alongside `user_deletion_audit_log` and `banned_user_audit`.

### 3.8 Admin postponement is supported but not operationalized at launch

The `user_deletion_locks` table is created and populated by the scheduled-deletion-skip check, but the operator does not use this feature at launch (no live bans, no live legal investigations). The notification path that would tell a locked user that their deletion is delayed is deferred until the email service exists.

### 3.9 Configuration is DB-backed

All user-deletion knobs live in the `Configuration` table (`configurationCache` in `DefaultConfigurationService`), not in `application.yml`. This matches the `product.removal.*` precedent and allows runtime tuning without redeploy. See §7 for the full key list.

Exception: `@Scheduled(cron = ...)` annotations resolve at bean-construction time from the Spring Environment and cannot read from the runtime `Configuration` cache. The three cron expressions (`user.deletion.hard.delete.cron`, `user.deletion.audit.purge.cron`, `user.deletion.firebase.reconciliation.cron`) therefore live in `application*.yaml`, not the Configuration table. Changing schedules requires a redeploy. All other knobs — batch sizes, retention periods, the reconciliation enable flag, the cascade threshold — are DB-tunable without redeploy.

### 3.10 Display-name sentinel: null at hard delete

After hard delete, references to the deleted user (in reviews, chat history, etc.) resolve to _null_ via the existing nullable association on the FK side (review `reviewer_id`) or via the backend returning null from `/api/auth/firebase/{uid}` for a UID whose row no longer exists. The frontend treats null as the cue to render `tCommon('user.deleted')`.

During the grace period (user row still exists), the user's actual `displayName` is **not mutated**. The user's row carries `deletionStatus = 'PENDING_DELETION'` and the `UserInfoDTO` response gains a `state` field set to `'PENDING_DELETION'`. The frontend uses `state` to render the "Scheduled for deletion" badge; the actual display name is preserved so that grace-cancel-by-login restores the original name with no mutation needed.

### 3.11 Adjacent bug status

The Phase 3 audit surfaced two backend bugs that this feature would inherit if left unfixed:

- `DefaultTrustReviewService.canReviewProduct` chat-id self-self bug.
- `DefaultReportService.createReport` self-assignment bug.

Operator fixed both manually outside the formal flow. Confirmed during Phase 3.

### 3.12 Firebase orphan with open reports — auto-ban before hard delete

When the weekly Firebase reconciliation job (§9.2 — `reconcileFirebaseUsers`) detects an orphan — a Postgres `users` row whose `firebase_uid` no longer exists in Firebase — the deletion service does not immediately run hard-delete. It first runs an abuse check:

- Count unfinished reports against this user (`report.reported_user_id = userId AND report.finished = FALSE`).
- Count unfinished reports authored by this user (`report.reporter_id = userId AND report.finished = FALSE`).

If either count is positive AND the per-arm threshold is met (see below), the service inserts a `banned_user_audit` row for this user **before** running hard delete:

- `email_hash` = SHA-256(lowercase+trim(user.email))
- `ban_reason` = `'auto: deletion with open reports'`
- `banned_at` = now
- `retention_until` = now + 12 months

The reports involving this user are then linked to that audit row per §3 reports treatment (kept, FKs anonymized to null, `banned_user_audit_id` set). The user is then hard-deleted with `triggered_by = 'firebase_cascade'`.

The threshold logic:

- **"Any unfinished report against this user"** — 1 or more is enough. Reports about you are evidence of complaints; even one open complaint at deletion time is reason enough to retain the re-registration barrier.
- **"More than N unfinished reports authored by this user"** — N is configured via `user.deletion.firebase.cascade.report.threshold`, default 3. Catches the serial bad-faith-reporter who tries to escape accountability by deleting their Firebase account.

If neither arm trips, the user is hard-deleted with no audit row inserted — they were a clean account that left through Firebase, and we have no operational reason to retain their hash.

---

## 4. User-facing behavior

### 4.1 Initiating deletion

**Location**: account settings page → bottom "Danger Zone" section.
**Path**: `/[locale]/owner/user`.
**Trigger**: explicit "Delete my account" button.
**Confirmation**: dialog (registered via `DialogManager`) requiring re-authentication.

The confirmation dialog:

1. Tells the user what will happen ("Your profile will remain visible for 7 days with a 'Scheduled for deletion' badge. Your listings will be hidden. You will be signed out. You can restore your account at any time during these 7 days by signing back in. After 7 days, your account is permanently deleted.").
2. Asks the user to re-authenticate. For email/password users: a password input + "Delete my account" button. For Google users: a single "Reauthenticate with Google and delete" button that triggers the popup. (Facebook case included for future enablement — see §14.2.)
3. On successful reauth, frontend calls `getIdToken(true)`, passes the fresh token explicitly on the delete request.
4. On success: backend returns `scheduledDeletionAt`. Frontend writes `sessionStorage.setItem('account-just-deleted', scheduledDeletionAt)`, signs the user out (`auth.signOut()`), which triggers `SessionGuard` to redirect from the protected dashboard to `/{locale}/`. On root layout mount, the sessionStorage key is detected, the post-deletion dialog opens with the date, and the key is removed.
5. On reauth failure: dialog shows the error, button remains.
6. On backend rejection with `REAUTH_REQUIRED`: dialog shows "Please re-authenticate again" and resets.
7. On backend rejection with `USER_LOCKED_FROM_DELETION`: dialog shows the generic "Account deletion is currently restricted. Please contact support@oglasino.com" message.

### 4.2 What happens at Day 0 (deletion request)

When the backend accepts the deletion request:

- `User.deletionStatus` flips from `'ACTIVE'` to `'PENDING_DELETION'`.
- `user_deletion_requests` row is inserted (or updated, if a previous one exists in the `'cancelled'` state — see §6.4).
- `user_deletion_audit_log` row is inserted.
- All products owned by the user are flipped from `ACTIVE` to `INACTIVE` (the system-context path; see §8.3). Elasticsearch is reindexed asynchronously per product.
- `FirebaseAuth.revokeRefreshTokens(uid)` is called. The user's existing sessions on other devices stop working as soon as their token refreshes.
- Redis: both `redisUserAuth` and `redisUserInfo` are evicted for this user (via `saveUser`'s `@CacheEvict`).
- Backend returns the deletion request DTO containing `scheduledDeletionAt`.

### 4.3 During the 7-day grace period

**The user (now signed out):**

- Cannot send or receive messages.
- Their profile is publicly visible with a "Scheduled for deletion" badge.
- Their phone number is hidden.
- Their listings are hidden (filtered out of all portal surfaces).
- Reviews about them remain visible.
- Reviews they authored remain visible with their normal name (because the user row is still active in the system, `displayName` is unchanged).
- Reports against them remain open. Any user can still file a report.

**The user can:**

- Sign in to restore the account at any time during the 7 days (see §4.4).

**Other users:**

- See the "Scheduled for deletion" badge on the user's profile.
- See the badge in the chat header for any conversation with the user.
- Cannot send messages to the user (the message input is disabled with a notice: "This user has scheduled their account for deletion and cannot receive messages.").
- Can still report the user (via the existing per-user report flow on the user's profile or in the chat header).
- See the user's existing messages in their chat history as normal (with the user's actual `displayName`, not "Deleted User" — that label appears only after hard delete).

### 4.4 Restoration during the grace period

When the user signs in during the 7 days:

1. Firebase Auth verifies their credentials (the user's Firebase account is still active during grace; we did **not** call `setDisabled` during the deletion request — only `revokeRefreshTokens`).
2. The fresh ID token reaches the backend's `FirebaseAuthFilter`.
3. The filter looks up the user's cached auth data; finding `deletionStatus = 'PENDING_DELETION'`, it calls `UserDeletionService.cancelDeletionOnLogin(user)`:
   - `user_deletion_requests`: `status = 'cancelled'`, `cancelled_at = now()`, `cancelled_reason = 'user_login'`.
   - `user_deletion_audit_log`: `cancelled_at = now()`.
   - `User.deletionStatus = 'ACTIVE'`, persisted via `saveUser` (evicting both Redis caches).
   - All `INACTIVE` products of this user that were flipped at Day 0 are flipped back to `ACTIVE` (system-context, batched, asynchronously reindexed to Elasticsearch).
4. The filter sets header `X-Account-Restored: true` on the response.
5. The filter falls through normally; the request completes successfully.
6. The frontend's axios response interceptor sees `X-Account-Restored: true`, sets a flag in `useAuthStore`. The root layout (which is mounted at the restore point because the user just signed in to the home page or a public page) reads the flag and opens the restoration `InfoDialog` via `DialogManager`. The flag is cleared as the dialog opens.

The restoration is server-side. The frontend dialog is informational only — even if the user closes the tab immediately after the API call hits, the deletion is already cancelled.

### 4.5 At Day 7 (hard delete)

The scheduled job at `user.deletion.hard.delete.cron` (default daily at 02:00) finds all `user_deletion_requests` rows where `status = 'pending'` AND `scheduled_deletion_at <= now()`. For each, the job:

1. Checks for an active deletion lock. If found, skips this user (the row stays `pending`; the lock must be removed for the user to be deleted).
2. Otherwise, runs `UserDeletionService.executeScheduledDeletion(user)`:
   - Deletes Firestore notifications for this user (see §5).
   - Deletes the user's profile image and listing images from R2 (review images are handled by the existing orphan sweeper).
   - Deletes the user's row from Postgres. JPA cascade handles products, product images (FK), product translations, user translations, and (via explicit pre-delete clear) the user_favorite_products join-table rows.
   - The reviews authored by the user that are _approved_: `reviewer_id` set to NULL via an explicit pre-delete UPDATE (see §5 for the SQL shape).
   - The reviews authored by the user that are _pending_: deleted via DELETE.
   - The reviews about the user: deleted via DELETE.
   - Reports authored or referenced by the user: per §3.12 reports treatment (linked to ban hash if banned, deleted if not).
   - `user_follow`, `push_token`, `suggestion` rows: deleted via DELETE.
   - Firebase Auth user is deleted via `FirebaseAuth.deleteUser(uid)`.
   - `user_deletion_requests.status = 'completed'`, `actual_deletion_at = now()`.
   - `user_deletion_audit_log.actual_deletion_at = now()`.
   - If the user was banned at the time of deletion (`User.disabled = true`), the `banned_user_audit` row is already in the table from when they were banned (it gets inserted at ban time, not at delete time — see §3.5). No additional insert needed at delete.

The user row is gone after this step. Any frontend code that subsequently calls `/api/auth/firebase/{uid}` for this deleted user's UID receives a null response. The chat renderer handles null by rendering "Deleted User" via `tCommon('user.deleted')`.

### 4.6 Admin-initiated ban

The admin clicks "Disable" on a user in the admin panel. The existing endpoint runs, extended per §3.5:

- `User.disabled = true`, `User.banReason = <admin-supplied reason>`, persisted via `saveUser` (cache eviction).
- `banned_user_audit` row inserted.
- Firebase `setDisabled(true)` + `revokeRefreshTokens` called.

The user is signed out on next interaction. They see the ban-notice dialog (§4.7).

### 4.7 Ban-notice dialog

A banned user attempts to use the platform. One of two trigger paths fires:

- **At sign-in**: Firebase rejects the credentials (because `setDisabled` is true) or `syncUserToBackend` returns `disabled: true`/`USER_BANNED`/`EMAIL_BANNED`. Frontend signs out and sets `sessionStorage.setItem('account-banned', '1')`.
- **Mid-session**: Backend returns `403 + USER_BANNED` for an authenticated request the user issues. Global axios interceptor signs out and sets the sessionStorage flag.

In both cases, the sign-out causes `SessionGuard` (if on a protected page) to redirect to `/{locale}/`. On the root layout mount of the next route, the sessionStorage flag is detected; the ban-notice dialog opens; the key is removed.

The dialog content:

> **Your account is banned.**
>
> Your account has been disabled by Oglasino administration. If you believe this is in error, you can appeal by emailing **support@oglasino.com**.
>
> **If you want to delete your account permanently**, email **support@oglasino.com** with the subject "Account deletion request." We will respond within 30 days as required by data-protection law.
>
> Each account ban lasts at most 12 months. After that period, the email associated with the banned account becomes eligible for re-registration if you wish to create a new account.

The dialog does **not** identify the specific user, does **not** display a reason, and does **not** confirm any specific email is banned. This is a deliberate privacy choice: a public surface that returned ban context for any submitted email would be an enumeration vector.

The dialog content is fully translated. See §15.

### 4.8 Re-registration attempt with a banned email

A user attempts to register (or sign in via Google with a previously-banned email):

1. Firebase Auth lets them through — Firebase doesn't know about our bans.
2. Frontend's `buildUserSession` → `syncUserToBackend` POSTs to `/auth/firebase-sync`.
3. Backend's `firebase-sync` handler hashes the incoming token's email and checks against `banned_user_audit`. If an unexpired hash matches:
   - Backend deletes the just-created Firebase user (`FirebaseAuth.deleteUser(uid)`), to avoid leaving an orphan in Firebase.
   - Backend returns 403 + `{errors: [{field: null, code: 'EMAIL_BANNED', translationKey: 'errors.email.banned'}]}`.
4. Frontend's `syncUserToBackend` catches the rejection, sets the `account-banned` sessionStorage flag, signs the user out, navigation lands on the home page, the ban-notice dialog opens.

The check happens server-side at the only entry point where a new user record would be created. The frontend has no role in the ban check — it just handles the response.

### 4.9 Edge cases at a glance

Detailed in §17:

- User has pending deletion → admin tries to ban: admin action takes precedence. Pending stays pending; ban is layered on top. At Day 7 the user is hard-deleted with `was_banned = true`.
- User locked from deletion → user requests deletion: rejected with 403 + `USER_LOCKED_FROM_DELETION`.
- User signs in but logs out before dialog shows: restoration is server-side. Dialog is informational only.
- User requests deletion, restores by sign-in, then requests deletion again: same `user_deletion_requests` row updated in place, new 7-day timer, audit log records both attempts.
- Firebase deletion during grace period: detected by the weekly reconciliation job; user is upgraded to immediate hard delete with `triggered_by = 'firebase_cascade'`. Per §3.12, if open reports exist, an auto-ban hash is inserted first.
- Server down when scheduled-deletion cron should run: cron picks up where it left off on next run. 24-hour delay acceptable.

---

## 5. Data inventory and deletion strategy

The complete matrix.

| Data location                                                                              | Day 0 (request)                                                         | During grace (Days 1–7)                                                                | Day 7 (hard delete)                                                                                                                                                                                              | Notes                                                                                                                                               |
| ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `users` row                                                                                | `deletion_status = 'PENDING_DELETION'`, `scheduled_deletion_at` set     | (unchanged)                                                                            | hard delete (JPA cascade handles dependents)                                                                                                                                                                     |                                                                                                                                                     |
| `User.displayName`                                                                         | (unchanged)                                                             | (unchanged)                                                                            | row goes away                                                                                                                                                                                                    | Not mutated. Restoration depends on the original value being preserved.                                                                             |
| `User.favoriteProducts` (join table `user_favorite_products`)                              | (unchanged)                                                             | (unchanged)                                                                            | `user.getFavoriteProducts().clear(); userService.saveUser(user)` before `userRepository.delete(user)`                                                                                                            | Cannot rely on `cascade=ALL` here — would attempt to delete the favorited products themselves. Explicit `clear()` removes only the join-table rows. |
| `products` owned by user                                                                   | `product.productState = INACTIVE` for each (system-context, batched)    | hidden from all portal surfaces via ES filter                                          | hard delete (JPA cascade via `User.products`)                                                                                                                                                                    | The `INACTIVE` state propagates to Elasticsearch async. The ES filter `productState == ACTIVE` already excludes them.                               |
| `product_translations` (under `products`)                                                  | (unchanged)                                                             | (unchanged)                                                                            | hard delete (JPA cascade via `Product.translations` orphanRemoval)                                                                                                                                               |                                                                                                                                                     |
| `product_images` join table (under `products`)                                             | (unchanged)                                                             | (unchanged)                                                                            | hard delete (JPA cascade)                                                                                                                                                                                        | The R2 objects are deleted separately.                                                                                                              |
| R2 listing images                                                                          | (unchanged)                                                             | (unchanged)                                                                            | explicit bulk delete via `R2Service.deleteBulk(imageKeys)`                                                                                                                                                       | Built from `productRepository.findAllImageKeysForOwner(userId)` before the user row is deleted.                                                     |
| R2 profile image                                                                           | (unchanged)                                                             | (unchanged)                                                                            | explicit `R2Service.delete(profileImageKey)`                                                                                                                                                                     | If non-null.                                                                                                                                        |
| `user_translations`                                                                        | (unchanged)                                                             | (unchanged)                                                                            | hard delete (JPA cascade via `User.translations` orphanRemoval)                                                                                                                                                  |                                                                                                                                                     |
| Approved reviews authored by user                                                          | (unchanged)                                                             | (unchanged)                                                                            | `UPDATE review SET reviewer_id = NULL WHERE reviewer_id = :userId AND approved = TRUE`                                                                                                                           | Requires `reviewer_id` schema migration to NULLABLE — see §6.2.                                                                                     |
| Pending-approval reviews authored by user                                                  | (unchanged)                                                             | (unchanged)                                                                            | `DELETE FROM review WHERE reviewer_id = :userId AND approved IS NULL`                                                                                                                                            | Per §3.3.                                                                                                                                           |
| Disapproved reviews authored by user (admin-rejected)                                      | (unchanged)                                                             | (unchanged)                                                                            | `DELETE FROM review WHERE reviewer_id = :userId AND approved = FALSE`                                                                                                                                            | Per §3.3. Without this delete, `userRepository.delete(user)` would FK-fail on disapproved-authored rows.                                            |
| Reviews about user (any approval state)                                                    | (unchanged)                                                             | (unchanged)                                                                            | `DELETE FROM review WHERE target_user_id = :userId`                                                                                                                                                              | Their images get cleaned up by the orphan sweeper.                                                                                                  |
| `review_translations` (under reviews)                                                      | (unchanged)                                                             | (unchanged)                                                                            | cascade via `Review.translations` orphanRemoval (for deleted reviews) / unaffected (for kept anonymized reviews)                                                                                                 |                                                                                                                                                     |
| `review_images` (R2, under `public/products/`)                                             | (unchanged)                                                             | (unchanged)                                                                            | not explicitly touched here; `ProductImagesRemovalJob` orphan sweeper cleans up dereferenced images later                                                                                                        | Review images share-prefix with product images on R2.                                                                                               |
| Reports authored by user (`reporter_id`)                                                   | (unchanged)                                                             | (unchanged)                                                                            | IF user is banned (has `banned_user_audit` row): `UPDATE report SET reporter_id = NULL, banned_user_audit_id = :auditRowId WHERE reporter_id = :userId`. ELSE: `DELETE FROM report WHERE reporter_id = :userId`. | Linked reports survive until the audit row is purged at 12 months (via `ON DELETE CASCADE`).                                                        |
| Reports about user (`reported_user_id`)                                                    | (unchanged)                                                             | (unchanged)                                                                            | IF user is banned: `UPDATE report SET reported_user_id = NULL, banned_user_audit_id = :auditRowId WHERE reported_user_id = :userId`. ELSE: `DELETE FROM report WHERE reported_user_id = :userId`.                | Same retention rule.                                                                                                                                |
| `user_follow` rows (either direction)                                                      | (unchanged)                                                             | (unchanged)                                                                            | `DELETE FROM user_follow WHERE follower_id = :userId OR following_id = :userId`                                                                                                                                  | Explicit in the deletion service.                                                                                                                   |
| `push_token` rows                                                                          | (unchanged)                                                             | (unchanged)                                                                            | `DELETE FROM push_token WHERE user_id = :userId`                                                                                                                                                                 | Explicit.                                                                                                                                           |
| `suggestion` rows                                                                          | (unchanged)                                                             | (unchanged)                                                                            | `DELETE FROM suggestion WHERE user_id = :userId`                                                                                                                                                                 | Explicit.                                                                                                                                           |
| `product_audit` rows                                                                       | (unchanged)                                                             | (unchanged)                                                                            | `UPDATE product_audit SET user_id = NULL WHERE user_id = :userId` OR delete depending on column nullability                                                                                                      | Engineer confirms during implementation.                                                                                                            |
| Firestore `chats/{chatId}` documents                                                       | (unchanged)                                                             | (unchanged)                                                                            | not deleted                                                                                                                                                                                                      | Chat documents are shared with the other participant. Deleting them would erase the other user's chat history.                                      |
| Firestore `chats/{chatId}/messages/*` documents                                            | (unchanged)                                                             | (unchanged)                                                                            | not deleted                                                                                                                                                                                                      | Same reasoning.                                                                                                                                     |
| Firestore `notifications/{firebaseUid}/userNotifications/*` (the deleted user's own inbox) | (unchanged)                                                             | (unchanged)                                                                            | recursive delete via Firestore batch                                                                                                                                                                             | Explicit.                                                                                                                                           |
| Firestore notifications _about_ the deleted user, held by other users                      | (unchanged)                                                             | (unchanged)                                                                            | not swept                                                                                                                                                                                                        | Per §3.9 Phase 1 / §19.4. Deferred.                                                                                                                 |
| Firebase Auth user record                                                                  | (unchanged) + `revokeRefreshTokens(uid)`                                | (gone from session, but Firebase account is still active so sign-in for restore works) | `FirebaseAuth.deleteUser(uid)`                                                                                                                                                                                   | Idempotent on USER_NOT_FOUND.                                                                                                                       |
| `redisUserAuth` (key: firebaseUid)                                                         | evicted via `saveUser`                                                  | (evicted, repopulated lazily on next request with new `deletionStatus`)                | evicted                                                                                                                                                                                                          | The connection-pool decision constraint.                                                                                                            |
| `redisUserInfo` (keys: id, firebaseUid)                                                    | evicted via `saveUser`                                                  | (evicted)                                                                              | evicted                                                                                                                                                                                                          | Same.                                                                                                                                               |
| Elasticsearch product index                                                                | reindex each product on `INACTIVE` flip (async, per product)            | (reflects INACTIVE)                                                                    | reindex each product on hard delete                                                                                                                                                                              | Engineer confirms whether the JPA delete fires an indexer event; if not, explicit delete from the indexer is added to the deletion service.         |
| `user_deletion_requests`                                                                   | insert row                                                              | (visible)                                                                              | `status = 'completed'`, `actual_deletion_at` set                                                                                                                                                                 | One row per user, lifetime — never duplicated.                                                                                                      |
| `user_deletion_audit_log`                                                                  | insert row                                                              | (visible)                                                                              | `actual_deletion_at` set                                                                                                                                                                                         | Hashed user id + email. Retained 30 days post-deletion.                                                                                             |
| `banned_user_audit`                                                                        | not touched at Day 0 (unless ban was already in place at deletion time) | (unchanged)                                                                            | not touched at Day 7 (the row stays for 12 months from `banned_at`)                                                                                                                                              | Inserted at ban time per §3.5.                                                                                                                      |
| `user_deletion_locks`                                                                      | (consulted but not modified)                                            | (consulted)                                                                            | (consulted; if active, deletion skipped)                                                                                                                                                                         | Operational state.                                                                                                                                  |
| `FirebaseAuth.revokeRefreshTokens(uid)`                                                    | called at Day 0                                                         | (existing tokens fail)                                                                 | (Firebase user is deleted)                                                                                                                                                                                       | Forces existing sessions to fail on next refresh.                                                                                                   |

The deletion service does the explicit handling in a specific order to satisfy FK constraints (clear favorites first, anonymize/delete reviews second, link or delete reports third, clear user_follow fourth, delete push_tokens and suggestions fifth, then delete the user row, which fires the JPA cascade for products and translations).

---

## 6. Schema changes

This feature is being implemented before any production deployment. Per the operator's decision dated 2026-05-17, the schema changes below are folded into the existing `V1__init_schema.sql` baseline migration rather than added as a new Flyway migration file. The dependency ordering and column definitions are unchanged; only the delivery mechanism differs.

Engineers editing `V1__init_schema.sql` must confirm with Igor that no preserved environment has V1 applied before making the edit. Local development databases that have V1 applied will need to be re-bootstrapped (drop and recreate database, or `mvn flyway:clean`).

Post-launch, all schema changes go in new Flyway migrations as normal. This pre-production fold is a one-time accommodation.

The configuration keys from §7 are seeded by additions to `data/configuration/*.sql` (a data-seeding mechanism, not a migration).

### 6.1 Modify `users` table CREATE

Add to the `CREATE TABLE users` statement:

```sql
deletion_status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
ban_reason VARCHAR(500),
```

Plus constraints and index:

```sql
CONSTRAINT chk_users_deletion_status
  CHECK (deletion_status IN ('ACTIVE', 'PENDING_DELETION'))
```

```sql
CREATE INDEX idx_users_deletion_status
  ON users(deletion_status)
  WHERE deletion_status != 'ACTIVE';
```

`deletion_status` only has two states from the entity's perspective — `ACTIVE` or `PENDING_DELETION`. The "deleted" state is the absence of the row entirely. `ban_reason` is nullable; populated only when `disabled = TRUE`. Reverted to NULL when admin un-bans (see §13).

### 6.2 `review.reviewer_id` becomes nullable

In the `CREATE TABLE review` statement, `reviewer_id` is created **without** `NOT NULL`. The FK constraint to `users(id)` remains. The corresponding JPA entity annotation changes from `@ManyToOne(fetch=LAZY, optional=false)` to `@ManyToOne(fetch=LAZY)`.

`review.target_user_id` stays `NOT NULL` — reviews about the deleted user are deleted entirely per §3.3.

### 6.3 Modify `report` table CREATE

Add to the `CREATE TABLE report` statement:

```sql
banned_user_audit_id BIGINT NULL,
CONSTRAINT fk_report_banned_user_audit
  FOREIGN KEY (banned_user_audit_id)
  REFERENCES banned_user_audit(id)
  ON DELETE CASCADE
```

```sql
CREATE INDEX idx_report_banned_user_audit ON report(banned_user_audit_id);
```

The FK has `ON DELETE CASCADE` so reports tied to a ban hash automatically purge when the audit row expires at 12 months. Note: this constraint requires `banned_user_audit` to be created before `report`, which is fine within V1 because we control the ordering.

### 6.4 New table — `user_deletion_requests`

```sql
CREATE TABLE user_deletion_requests (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL UNIQUE
    REFERENCES users(id) ON DELETE CASCADE,
  requested_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  scheduled_deletion_at TIMESTAMP WITH TIME ZONE NOT NULL,
  cancelled_at TIMESTAMP WITH TIME ZONE,
  cancelled_reason VARCHAR(50),
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
  CONSTRAINT chk_udr_status
    CHECK (status IN ('PENDING', 'CANCELLED', 'COMPLETED')),
  CONSTRAINT chk_udr_cancelled_reason
    CHECK (cancelled_reason IS NULL
      OR cancelled_reason IN ('user_login', 'admin_override'))
);

CREATE INDEX idx_udr_pending_due
  ON user_deletion_requests(scheduled_deletion_at)
  WHERE status = 'PENDING';
```

Status values are stored uppercase to match the Java enum constant names (`DeletionRequestStatus.PENDING` etc., `@Enumerated(EnumType.STRING)`). The CHECK constraint and partial-index predicate use uppercase string literals accordingly.

`UNIQUE` on `user_id` enforces "one row per user, ever." Restoration updates the row; re-request updates again.

`ON DELETE CASCADE` on the `user_id` FK means hard deletion of the user row also deletes the request row. The audit log survives because it stores hashed identifiers, not FK references.

### 6.5 New table — `user_deletion_audit_log`

```sql
CREATE TABLE user_deletion_audit_log (
  id BIGSERIAL PRIMARY KEY,
  user_id_hash VARCHAR(64) NOT NULL,
  email_hash VARCHAR(64) NOT NULL,
  requested_at TIMESTAMP WITH TIME ZONE NOT NULL,
  scheduled_deletion_at TIMESTAMP WITH TIME ZONE NOT NULL,
  actual_deletion_at TIMESTAMP WITH TIME ZONE,
  cancelled_at TIMESTAMP WITH TIME ZONE,
  triggered_by VARCHAR(30) NOT NULL,
  was_banned BOOLEAN NOT NULL DEFAULT FALSE,
  retention_until TIMESTAMP WITH TIME ZONE NOT NULL,
  CONSTRAINT chk_udal_triggered_by
    CHECK (triggered_by IN ('user_request', 'admin_action', 'firebase_cascade'))
);

CREATE INDEX idx_udal_email_hash ON user_deletion_audit_log(email_hash);
CREATE INDEX idx_udal_retention ON user_deletion_audit_log(retention_until);
```

`retention_until` is set at row insertion: `now() + user.deletion.audit.retention.normal.days` days (default 30).

The original engineering spec had a `ban_reason` column on this table. We're not adding it — `banned_user_audit` is where reason lives. `was_banned` is enough to link the two records by retention window.

### 6.6 New table — `banned_user_audit`

```sql
CREATE TABLE banned_user_audit (
  id BIGSERIAL PRIMARY KEY,
  email_hash VARCHAR(64) NOT NULL UNIQUE,
  ban_reason VARCHAR(500) NOT NULL,
  banned_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  retention_until TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE INDEX idx_bua_email_hash ON banned_user_audit(email_hash);
CREATE INDEX idx_bua_retention ON banned_user_audit(retention_until);
```

`email_hash` is UNIQUE because there can only be one active ban per email at a time.

`retention_until` is `banned_at + user.deletion.audit.retention.banned.months` months (default 12). The audit-purge job deletes rows where `retention_until <= now()`. When the row is deleted, `ON DELETE CASCADE` on `report.banned_user_audit_id` deletes any linked reports.

### 6.7 New table — `user_deletion_locks`

```sql
CREATE TABLE user_deletion_locks (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL UNIQUE
    REFERENCES users(id) ON DELETE CASCADE,
  locked_by_admin_id BIGINT NOT NULL
    REFERENCES users(id),
  reason VARCHAR(500) NOT NULL,
  disclosable_reason VARCHAR(500),
  notified_at TIMESTAMP WITH TIME ZONE,
  locked_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMP WITH TIME ZONE
);

-- The original spec defined this as a partial index with predicate
-- "WHERE expires_at IS NULL OR expires_at > NOW()", but Postgres rejects
-- partial-index predicates that call STABLE functions like NOW() with
-- error 42P17. The shipped V1 uses a plain (non-partial) btree on
-- user_id. The active-vs-expired check runs at query time in
-- UserDeletionLockRepository.existsActiveLockForUser via a :now
-- parameter. Effective lookup is still O(1) because of the UNIQUE
-- constraint on user_id — at most one row per user.
CREATE INDEX idx_udl_user_active ON user_deletion_locks(user_id);
```

`UNIQUE` on `user_id`: at most one active lock per user.

`reason` is the internal full reason. `disclosable_reason` is what we tell the user if we tell them — nullable for cases where the reason cannot be disclosed at all.

`notified_at` is when the user was notified. For launch, this stays NULL because no notification path exists yet.

`expires_at` nullable means indefinite lock. The active-lock check filters `expires_at IS NULL OR expires_at > NOW()`. No cron sweep of expired locks — the lock is simply ignored once expired.

### 6.8 V1 ordering

Within `V1__init_schema.sql`, the order of CREATE statements must be:

1. `banned_user_audit` (no FK dependencies).
2. `user_deletion_audit_log` (no FK dependencies).
3. (existing tables — `users`, `language`, `base_site`, `region`, `city`, etc.)
4. `user_deletion_locks` (FK to `users`).
5. `user_deletion_requests` (FK to `users`).
6. (existing `review`, `report` tables — with `reviewer_id` now nullable and `banned_user_audit_id` FK on `report`).

---

## 7. Configuration

Most deletion knobs live in the `Configuration` table per §3.9. The three cron expressions are the exception — they live in `application*.yaml` (see §3.9 for the reasoning).

| Key                                                  | Type    | Default         | Source              | Description                                                                                                                                                                       |
| ---------------------------------------------------- | ------- | --------------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `user.deletion.grace.period.days`                    | int     | 7               | `Configuration`     | Days between user-initiated deletion request and hard delete.                                                                                                                     |
| `user.deletion.report.window.days`                   | int     | 7               | `Configuration`     | Days during which reports against the deleting user remain open. Currently identical to the grace period — kept as a separate knob so the two timers can diverge later if needed. |
| `user.deletion.audit.retention.normal.days`          | int     | 30              | `Configuration`     | Days that `user_deletion_audit_log` rows are kept after deletion.                                                                                                                 |
| `user.deletion.audit.retention.banned.months`        | int     | 12              | `Configuration`     | Months that `banned_user_audit` rows are kept. Per Privacy Policy / Terms.                                                                                                        |
| `user.deletion.hard.delete.batch.size`               | int     | 50              | `Configuration`     | Max users processed per cron tick of the hard-delete sweep.                                                                                                                       |
| `user.deletion.hard.delete.cron` †                   | string  | `0 0 2 * * *`   | `application*.yaml` | Cron expression for the daily hard-delete sweep.                                                                                                                                  |
| `user.deletion.audit.purge.cron` †                   | string  | `0 0 4 * * SUN` | `application*.yaml` | Cron for the weekly audit-purge job.                                                                                                                                              |
| `user.deletion.firebase.reconciliation.cron` †       | string  | `0 0 3 * * SUN` | `application*.yaml` | Cron for the weekly Firebase orphan reconciliation.                                                                                                                               |
| `user.deletion.firebase.reconciliation.enabled`      | boolean | `true`          | `Configuration`     | Master switch for the reconciliation job.                                                                                                                                         |
| `user.deletion.firebase.reconciliation.page.size`    | int     | 100             | `Configuration`     | Pagination size for iterating active users.                                                                                                                                       |
| `user.deletion.reauth.max.age.seconds`               | int     | 300             | `Configuration`     | Max age of the `auth_time` claim accepted by the delete-account endpoint.                                                                                                         |
| `user.deletion.firebase.cascade.report.threshold`    | int     | 3               | `Configuration`     | Threshold for auto-ban-on-cascade-with-authored-reports per §3.12.                                                                                                                |

† Per the §3.9 exception, `@Scheduled(cron = ...)` resolves at bean-construction time and cannot read from the runtime `Configuration` cache. Changing a cron schedule requires editing `application*.yaml` and redeploying.

`Configuration`-sourced keys are seeded by a `data/configuration/*.sql` file delivered with the migration. Admin can override any of them via the existing admin Configuration endpoints without redeploy.

---

## 8. Backend service contracts

### 8.1 `UserDeletionService` (new — `service.UserDeletionService`)

```java
public interface UserDeletionService {

  /**
   * Phase 1: user-initiated deletion request.
   *
   * Preconditions:
   *   - User is authenticated; user object is loaded from the auth context.
   *   - The ID token used for this request has auth_time within
   *     user.deletion.reauth.max.age.seconds (controller-level check;
   *     see §10).
   *   - User is not locked from deletion. Throws
   *     UserLockedFromDeletionException with 403 + USER_LOCKED_FROM_DELETION
   *     if locked.
   *   - User is not already pending deletion. If they are, the call is
   *     idempotent — returns the existing scheduledDeletionAt without
   *     creating a duplicate row.
   *
   * Side effects:
   *   - Insert or update user_deletion_requests row (status='pending',
   *     scheduled_deletion_at = now + grace.period.days).
   *   - Insert user_deletion_audit_log row.
   *   - Update users.deletion_status = 'PENDING_DELETION' via saveUser
   *     (evicts both Redis caches).
   *   - Flip all user's products from ACTIVE to INACTIVE via
   *     ProductService.changeProductStateAsSystem (§8.3).
   *   - Call FirebaseAuth.revokeRefreshTokens(uid).
   *
   * Note: does NOT call FirebaseAuth.setDisabled(true). User must still be
   * able to sign in to restore.
   *
   * Returns: DeletionRequestResult { scheduledDeletionAt: Instant }
   */
  DeletionRequestResult requestDeletion(User user);

  /**
   * Phase 2: user signs in during grace period — auto-restore.
   * Called by FirebaseAuthFilter after successful token verification when
   * user.deletion_status == 'PENDING_DELETION'.
   *
   * Preconditions:
   *   - user.deletion_status == 'PENDING_DELETION'.
   *
   * Side effects:
   *   - user_deletion_requests: status='cancelled', cancelled_at=now,
   *     cancelled_reason='user_login'.
   *   - users.deletion_status = 'ACTIVE' via saveUser.
   *   - user_deletion_audit_log: cancelled_at=now.
   *   - Flip all INACTIVE products of this user back to ACTIVE via
   *     ProductService.changeProductStateAsSystem.
   *
   * Idempotent if no pending deletion exists (no-op).
   */
  void cancelDeletionOnLogin(User user);

  /**
   * Phase 3: hard delete after grace period. Called by
   * ScheduledDeletionJob.
   *
   * Preconditions:
   *   - user.deletion_status == 'PENDING_DELETION'.
   *   - user_deletion_requests.scheduled_deletion_at <= now.
   *   - No active deletion lock for this user.
   *
   * Side effects (in this exact order to satisfy FK constraints):
   *   1. user.getFavoriteProducts().clear(); save user.
   *   2. linkReportsToBanIfBanned(user) — sets banned_user_audit_id on
   *      reports if banned, deletes reports if not.
   *   3. Anonymize approved authored reviews: reviewer_id = NULL.
   *   4. Delete pending authored reviews.
   *   5. Delete reviews about user.
   *   6. Delete user_follow rows (both directions).
   *   7. Delete push_token rows.
   *   8. Delete suggestion rows.
   *   9. Handle product_audit rows (UPDATE or DELETE depending on
   *      nullability — engineer confirms in implementation).
   *  10. Collect R2 image keys (profile, all listing images for owner).
   *  11. Delete the user's Firestore notification subcollection.
   *  12. userRepository.delete(user). JPA cascade handles products,
   *      product translations, user translations, product_images join
   *      rows. ON DELETE CASCADE on user_deletion_requests.user_id
   *      simultaneously deletes the deletion-request row (see step 15
   *      note).
   *  13. R2Service.deleteBulk(collectedImageKeys). Best-effort.
   *  14. FirebaseAuth.deleteUser(uid). Idempotent on USER_NOT_FOUND.
   *  15. (Formerly: update user_deletion_requests row.) Removed. The
   *      ON DELETE CASCADE on user_deletion_requests.user_id deletes
   *      the request row when step 12 deletes the user. The request
   *      row is intentionally ephemeral (Mastermind Option B); the
   *      audit log at step 16 carries the durable record.
   *  16. Update user_deletion_audit_log: actual_deletion_at=now.
   */
  void executeScheduledDeletion(User user);

  /**
   * Phase 4: immediate hard delete triggered by Firebase orphan detection.
   * Called by FirebaseReconciliationJob when a user row's firebase_uid
   * does not exist in Firebase.
   *
   * Before running, checks per §3.12:
   *   - count unfinished reports against user
   *   - count unfinished reports authored by user
   * If "any against" OR "more than firebase.cascade.report.threshold
   * authored" → insert banned_user_audit row with reason
   * 'auto: deletion with open reports'.
   *
   * Then runs executeScheduledDeletion's steps 1–16 with
   * triggered_by='firebase_cascade'.
   *
   * Note: no grace period — Firebase is already gone, no restore
   * possible.
   */
  void markForImmediateDeletion(User user, String reason);

  /**
   * Phase 5: admin-initiated force delete. Used for the banned-user
   * email-deletion path (§3.6) and for any other admin override case.
   *
   * Preconditions:
   *   - Caller is admin (controller-level check).
   *   - No active deletion lock.
   *
   * Runs executeScheduledDeletion's steps 1–16 immediately with
   * triggered_by='admin_action'. No grace period.
   */
  void forceDeleteByAdmin(Long userId, String adminReason);

  /**
   * Admin lock operation.
   */
  void lockFromDeletion(Long userId,
                        Long adminId,
                        String reason,
                        String disclosableReason,
                        Optional<Instant> expiresAt);

  /**
   * Admin unlock operation. Removes the lock row.
   */
  void unlockFromDeletion(Long userId);
}
```

### 8.2 `UserAuditService` (new — `service.UserAuditService`)

```java
public interface UserAuditService {

  /**
   * SHA-256 hash of lowercase(trim(email)). Consistent across both
   * user_deletion_audit_log.email_hash and banned_user_audit.email_hash.
   */
  String hashEmail(String email);

  /**
   * SHA-256 hash of user.id.toString(). Used for
   * user_deletion_audit_log.user_id_hash.
   */
  String hashUserId(Long userId);

  /**
   * Check if email is in banned_user_audit AND the row has not expired.
   * Called from firebase-sync to reject re-registration attempts.
   */
  boolean isEmailBanned(String email);

  /**
   * Insert a banned_user_audit row. Idempotent on email_hash collision:
   * returns the existing row's id instead of inserting.
   */
  Long recordBannedUser(String email, String banReason);

  /**
   * Delete a banned_user_audit row by email_hash. Called from
   * DefaultUsersFacade.enableUser (admin un-ban). ON DELETE CASCADE on
   * report.banned_user_audit_id deletes linked reports.
   *
   * No-op if no row exists for this email_hash.
   */
  void removeBannedUserRecord(String email);
}
```

### 8.3 `ProductService` extension (modify — `service.ProductService`)

```java
public interface ProductService {

  // ... existing methods ...

  /**
   * System-context state change. Skips the ownership check that
   * changeProductState performs. Used by:
   *   - UserDeletionService.requestDeletion (Day 0 bulk INACTIVE flip)
   *   - UserDeletionService.cancelDeletionOnLogin (restore bulk ACTIVE
   *     flip)
   *
   * Publishes the standard ProductStateUpdateEvent so Elasticsearch
   * reindexes asynchronously.
   */
  void changeProductStateAsSystem(Long productId, ProductState newState);

  /**
   * Helper: find all product IDs owned by a user, in the given state.
   */
  List<Long> findProductIdsByOwnerAndState(Long ownerId, ProductState state);
}
```

The implementation: copy `changeProductState`'s body, remove the ownership check, keep the event publishing. Recommended (not required) to refactor `changeProductState` to delegate to this so the ownership check sits in a wrapper.

### 8.4 `FirebaseUserService` extension (modify — `admin.service.FirebaseUserService`)

```java
public interface FirebaseUserService {

  void disableUser(String firebaseUid);  // existing
  void enableUser(String firebaseUid);   // existing

  /**
   * Hard delete the user from Firebase Auth.
   * Idempotent on USER_NOT_FOUND.
   */
  void deleteFirebaseUser(String firebaseUid);

  /**
   * Check if a user still exists in Firebase Auth.
   * Returns false on USER_NOT_FOUND; throws on any other Firebase error.
   */
  boolean userExistsInFirebase(String firebaseUid);

  /**
   * Revoke all refresh tokens for this user.
   * Idempotent on USER_NOT_FOUND.
   */
  void revokeRefreshTokens(String firebaseUid);
}
```

### 8.5 `FirestoreUserService` (new — `service.FirestoreUserService`)

```java
public interface FirestoreUserService {

  /**
   * Delete the entire notifications/{firebaseUid}/userNotifications
   * subcollection plus the parent notification doc.
   * Idempotent: empty collection is a no-op.
   */
  void deleteUserNotifications(String firebaseUid);
}
```

No method for "anonymize chat sender display names" — per §3.2, we don't mutate chat messages.

No method for "sweep other users' notifications that mention deleted user" — per §19.4, deferred.

### 8.6 `DefaultUsersFacade` modification

```java
public interface UsersFacade {

  void disableUser(Long userId, String banReason);
  void enableUser(Long userId);
}
```

`disableUser`:

1. Load user.
2. Set `user.disabled = true`, `user.banReason = banReason`.
3. `userService.saveUser(user)` (evicts caches).
4. `userAuditService.recordBannedUser(user.email, banReason)`.
5. `firebaseUserService.disableUser(user.firebaseUid)`.
6. `firebaseUserService.revokeRefreshTokens(user.firebaseUid)`.

`enableUser`:

1. Load user.
2. Set `user.disabled = false`, `user.banReason = null`.
3. `userService.saveUser(user)`.
4. `userAuditService.removeBannedUserRecord(user.email)` — cascades any linked reports to deletion.
5. `firebaseUserService.enableUser(user.firebaseUid)`.

Backward-compat: if the existing admin button POSTs without a body, the controller accepts a null `banReason` and defaults to a placeholder string.

### 8.7 `AuthUserDTO` and `UserInfoDTO` extensions

`AuthUserDTO`:

```java
class AuthUserDTO {
  // existing fields ...

  // new
  boolean disabled;
  String banReason;            // null unless disabled
  DeletionStatus deletionStatus;  // 'ACTIVE' or 'PENDING_DELETION'
  Instant scheduledDeletionAt;  // null unless PENDING_DELETION
}
```

`UserInfoDTO`:

```java
class UserInfoDTO {
  // existing fields ...

  // new
  UserState state;  // 'ACTIVE' | 'PENDING_DELETION' | 'BANNED'
  Instant scheduledDeletionAt;
}
```

`BANNED` wins over `PENDING_DELETION` in the composition rule. See `UserState.resolve(boolean disabled, DeletionStatus deletionStatus)` in `UserState.java` for the canonical algorithm — banned-and-pending users surface as `BANNED` on the wire so counterparty UIs render the strongest gating posture.

After hard delete, `/api/auth/firebase/{uid}` returns null (preserves existing 404 behavior per backend audit §2).

### 8.8 Error codes

| Code                        | HTTP | Where emitted                                                          | Translation key                    |
| --------------------------- | ---- | ---------------------------------------------------------------------- | ---------------------------------- |
| `REAUTH_REQUIRED`           | 403  | Delete-account endpoint when `auth_time` is too old.                   | `errors.reauth.required`           |
| `USER_BANNED`               | 403  | Auth filter when `disabled=true`; also from `firebase-sync` if banned. | `errors.user.banned`               |
| `EMAIL_BANNED`              | 403  | `firebase-sync` when registration attempt matches banned hash.         | `errors.email.banned`              |
| `USER_LOCKED_FROM_DELETION` | 403  | Delete-account endpoint when user has active lock.                     | `errors.user.locked.from.deletion` |
| `USER_NOT_PENDING_DELETION` | 400  | Defensive — if a restore is somehow called on a non-pending user.      | `errors.user.not.pending.deletion` |

All use the existing `{errors: [{field, code, translationKey}]}` envelope.

### 8.9 Cache invalidation pattern (backend → web revalidation)

Backend → web cache invalidation fires after every user state transition via `UserStateChangedEvent` + `DefaultWebRevalidationService` POSTing to `/api/revalidate` with header `X-Revalidate-Secret` and body `{"tags": ["user:<userId>"]}`. The event is published from `DefaultUsersFacade` (ban / unban) and `DefaultUserDeletionService` (request / cancel / hard-delete) after the transactional write commits.

Wire-up uses `@TransactionalEventListener(AFTER_COMMIT, fallbackExecution = true)` so the revalidation call happens only after the state-transition write has been durably committed; `fallbackExecution = true` covers the no-active-transaction case (admin tooling, tests). Failure is logged and swallowed — the 5-minute SSR `revalidate: 300` window on `UserInfoDTO` is the upper bound on staleness, so a missed revalidate is recoverable without operator action.

Auth: shared secret in the `X-Revalidate-Secret` header. Same value on both sides (backend `REVALIDATE_SECRET` env, web `REVALIDATE_SECRET` env). On dev, the secret can stay empty — backend defaults to disabled and the web route fails closed with 503.

See `decisions.md` 2026-05-19 for the design pattern decision (best-effort fire-and-forget vs explicit post-commit method calls). See `issues.md` 2026-05-19 for the production routing caveat (the `oglasino.com` domain vs direct Vercel URL) and the secret-inventory entry for `REVALIDATE_SECRET` and `WEB_REVALIDATE_URL`.

---

## 9. Scheduled jobs

Three new jobs in `UserDeletionScheduledJobs`. Pattern mirrors `ProductRemovalJob`.

### 9.1 Daily hard-delete sweep

```java
@Scheduled(cron = "${user.deletion.hard.delete.cron:0 0 2 * * *}")
public void processScheduledDeletions() {
  int batchSize = configurationService.getRequiredIntConfig(
      "user.deletion.hard.delete.batch.size");

  Page<UserDeletionRequest> due = deletionRequestRepository
      .findPendingDueByDate(LocalDateTime.now(), PageRequest.of(0, batchSize));

  for (UserDeletionRequest req : due) {
    try {
      if (deletionLockRepository.existsActiveLockForUser(req.getUserId())) {
        log.info("Skipping deletion for locked user {}", req.getUserId());
        continue;
      }
      User user = userRepository.findById(req.getUserId()).orElse(null);
      if (user == null) {
        log.warn("Pending deletion row {} has no user; cleaning up",
                 req.getId());
        deletionRequestRepository.delete(req);
        continue;
      }
      userDeletionService.executeScheduledDeletion(user);
    } catch (Exception e) {
      log.error("Deletion failed for user {}; will retry next run",
                req.getUserId(), e);
    }
  }
}
```

### 9.2 Weekly Firebase reconciliation

```java
@Scheduled(cron = "${user.deletion.firebase.reconciliation.cron:0 0 3 * * SUN}")
public void reconcileFirebaseUsers() {
  if (!configurationService.getBooleanConfig(
      "user.deletion.firebase.reconciliation.enabled")) return;

  int pageSize = configurationService.getRequiredIntConfig(
      "user.deletion.firebase.reconciliation.page.size");

  int pageNum = 0;
  Page<User> page;
  do {
    page = userRepository.findAllActive(PageRequest.of(pageNum, pageSize));
    for (User user : page) {
      if (user.getFirebaseUid() == null) continue;
      try {
        if (!firebaseUserService.userExistsInFirebase(user.getFirebaseUid())) {
          log.warn("Orphaned user in Firebase: {}", user.getId());
          userDeletionService.markForImmediateDeletion(user, "firebase_cascade");
        }
      } catch (Exception e) {
        log.error("Firebase check failed for user {}, skipping",
                  user.getId(), e);
      }
    }
    pageNum++;
  } while (!page.isEmpty() && page.hasNext());
}
```

`findAllActive` excludes already-pending and already-deleted users: `WHERE deletion_status = 'ACTIVE'`.

`getBooleanConfig` returns `false` on missing — the safer default for a destructive batch job. The Configuration seed at Phase 3 sets this key to `'true'` explicitly so the default path never fires in normal operation.

The reconciliation does NOT do the reverse check (DB row missing, Firebase user still exists). See §17.9 / §19.10.

### 9.3 Weekly audit purge

```java
@Scheduled(cron = "${user.deletion.audit.purge.cron:0 0 4 * * SUN}")
public void purgeExpiredAuditRecords() {
  int deletedAudit = auditLogRepository.deleteExpired(LocalDateTime.now());
  int deletedBans = bannedUserAuditRepository.deleteExpired(LocalDateTime.now());
  int deletedAdminActions = adminActionAuditRepository.deleteExpired(LocalDateTime.now());
  log.info("Purge: audit={}, bans={}, adminActions={}", deletedAudit, deletedBans, deletedAdminActions);
}
```

Three retention purges: `user_deletion_audit_log` rows past their 30-day retention, `banned_user_audit` rows past their 12-month retention, and `user_admin_action_audit` rows past their 12-month retention (per §3.7). There is no purge for `user_deletion_requests` — the FK cascade on user delete handles cleanup of completed and cancelled requests; the table only ever contains in-flight pending rows.

The FK CASCADE on `report.banned_user_audit_id` deletes linked reports automatically when their parent audit row goes away.

---

## 10. Auth filter changes

`FirebaseAuthFilter` gains two new checks after token verification:

1. Verify Firebase ID token (existing).
2. Look up cached AuthenticatedUserDTO (existing).
3. NEW: If authData.disabled == true:
   - Return HTTP 403 with body {errors: [{field: null, code: 'USER_BANNED', translationKey: 'errors.user.banned'}]}.
   - Short-circuit; do not invoke filterChain.doFilter.
4. NEW: If authData.deletionStatus == 'PENDING_DELETION':
   - Load full User entity (single DB hit — we need to do the cancellation work).
   - Call userDeletionService.cancelDeletionOnLogin(user).
   - Set response header X-Account-Restored: true.
   - Continue with auth context.
5. Build OglasinoAuthentication (existing).
6. Set SecurityContextHolder (existing).
7. filterChain.doFilter (existing).

`cancelDeletionOnLogin` is idempotent. After the first call within a session, `deletion_status` is `ACTIVE` and subsequent requests skip the branch; the `X-Account-Restored: true` header is only set on the first request.

The `firebase-sync` endpoint exemption (filter doesn't run token verification for `/api/auth/firebase-sync`) is preserved. The disabled-check and ban-check happen in the `firebase-sync` handler itself.

### 10.1 `firebase-sync` handler changes

1. Verify Firebase ID token.
2. Extract email from token.
3. Check userAuditService.isEmailBanned(email).
   - If banned:
     - Delete the Firebase user just created (best-effort).
     - Return HTTP 403 with body {errors: [{field: null, code: 'EMAIL_BANNED', translationKey: 'errors.email.banned'}]}.
4. getOrCreateUser as today.
5. NEW: if user.disabled == true:
   - Return HTTP 403 + USER_BANNED.
6. NEW: if user.deletionStatus == 'PENDING_DELETION':
   - Call cancelDeletionOnLogin (this is the restore path for sign-in).
   - Set X-Account-Restored: true.
7. Build and return AuthUserDTO with the new fields populated.

### 10.2 Delete-account endpoint and reauth freshness

POST /api/secure/user/me/delete
Authorization: Bearer <fresh ID token>
Body: {} (empty)

No `userId` in the body. User derived from auth context.

Controller logic:

```java
@PostMapping("/me/delete")
public ResponseEntity<DeletionRequestResultDTO> deleteMe(
    @AuthenticationPrincipal OglasinoAuthentication auth) {
  FirebaseToken token = (FirebaseToken)
      SecurityContextHolder.getContext().getAuthentication().getCredentials();

  Long authTimeSeconds = (Long) token.getClaims().get("auth_time");
  if (authTimeSeconds == null
      || Instant.now().getEpochSecond() - authTimeSeconds
         > configurationService.getRequiredIntConfig(
             "user.deletion.reauth.max.age.seconds")) {
    throw new ReauthRequiredException();
  }

  User user = userService.findById(auth.getUserId()).orElseThrow();
  DeletionRequestResult result = userDeletionService.requestDeletion(user);
  return ResponseEntity.ok(new DeletionRequestResultDTO(result.scheduledDeletionAt));
}
```

The `auth_time` claim is a Unix epoch in seconds.

**Token exposure to controller.** Per backend audit §2 the current `FirebaseAuthFilter` doesn't stash the verified token. The filter is modified to set `OglasinoAuthentication.credentials = firebaseToken` so the controller can retrieve it as above. (Engineer's option to re-verify in the controller is also acceptable; the recommended path is stash-in-filter.)

---

## 11. Firebase Auth integration — summary

Already covered in §8.4. Call sites:

| Call                        | Source                                                 | Purpose                             |
| --------------------------- | ------------------------------------------------------ | ----------------------------------- |
| `disableUser(uid)`          | `DefaultUsersFacade.disableUser`                       | Admin ban                           |
| `enableUser(uid)`           | `DefaultUsersFacade.enableUser`                        | Admin un-ban                        |
| `revokeRefreshTokens(uid)`  | `DefaultUsersFacade.disableUser`                       | After admin ban                     |
| `revokeRefreshTokens(uid)`  | `UserDeletionService.requestDeletion`                  | At Day 0                            |
| `deleteFirebaseUser(uid)`   | `UserDeletionService.executeScheduledDeletion` step 14 | Day 7                               |
| `deleteFirebaseUser(uid)`   | `firebase-sync` handler when ban-hash matches          | Clean up just-created Firebase user |
| `userExistsInFirebase(uid)` | `UserDeletionScheduledJobs.reconcileFirebaseUsers`     | Weekly orphan check                 |

No call to `setDisabled` at user-initiated deletion request — the user must be able to sign in to restore.

---

## 12. Reauth contract

### 12.1 Frontend responsibilities

Between "user clicks Delete" and "frontend sends the delete request":

1. **Invoke reauth.** Email/password: `await reauthenticateWithCredential(currentUser, EmailAuthProvider.credential(email, password))`. Google: `await reauthenticateWithPopup(currentUser, googleProvider)`. Facebook: same with `facebookProvider` (the case is implemented for future enablement).
2. **Force-refresh the ID token.** `const freshToken = await currentUser.getIdToken(true)`. The `true` parameter is critical — without it, the cached token is returned with old `auth_time`.
3. **Pass the fresh token explicitly on the delete request.** Per the recommendation in §14.3: `axios.post('/secure/user/me/delete', {}, { headers: { Authorization: 'Bearer ' + freshToken } })`. This bypasses the axios interceptor's cached token.
4. **Submit the deletion request.** Empty body.

If step 2 is omitted, the backend's reauth-freshness check rejects with `REAUTH_REQUIRED`. The frontend handles this by showing a "please reauthenticate again" message in the dialog.

### 12.2 Backend responsibilities

1. **Verify the ID token** (already done in filter; token stashed in credentials per §10.2).
2. **Read `auth_time` claim** from the verified token.
3. **Check freshness.** `now - auth_time <= user.deletion.reauth.max.age.seconds`. If not, throw `ReauthRequiredException` → 403 + `REAUTH_REQUIRED`.
4. **Proceed with deletion.**

No tolerance for missing `auth_time` — if the claim is absent, reject.

---

## 13. Banned user lifecycle

### 13.1 Active user, never banned, deletes themselves

Day 0: `requestDeletion`. `deletion_status = 'PENDING_DELETION'`. Products INACTIVE. Refresh tokens revoked. Days 1–7: user can sign in to restore. Day 7: `executeScheduledDeletion`. User row deleted. `triggered_by = 'user_request'`. `was_banned = false`. No `banned_user_audit` row. Day 7 + 30: audit log row purged.

### 13.2 Active user, banned by admin

Day 0: admin `disableUser(userId, banReason)`. `user.disabled = true`, `banReason` set. `banned_user_audit` row inserted. Firebase `setDisabled(true)`. `revokeRefreshTokens`. Day 0+: user attempts to sign in, sees the ban-notice dialog.

The ban can persist indefinitely on our side. Re-registration is blocked for 12 months from `banned_at`.

### 13.3 Banned user, admin un-bans

Day N: admin `enableUser(userId)`. `user.disabled = false`, `banReason = null`. `banned_user_audit` row deleted (and any linked reports via cascade). Firebase `setDisabled(false)`. Day N+: user can sign in normally.

### 13.4 Banned user, requests deletion via email

Day 0 of ban: admin disables user. Day M: user emails support. Day M+: admin verifies, calls `POST /api/admin/users/{userId}/force-delete`. The endpoint runs `forceDeleteByAdmin`. `triggered_by = 'admin_action'`, `was_banned = true`. `banned_user_audit` row is NOT deleted — persists until original 12 months from `banned_at`.

### 13.5 Banned user, banned_user_audit purges before 12 months

If admin un-bans before 12 months, the hash row is deleted (§13.3). If admin never un-bans, audit-purge deletes the row at 12 months. After purge, email is re-registrable.

### 13.6 Firebase-cascade with active ban

Day 0: admin bans user. Day N: user deletes themselves from Firebase. Next Sunday: reconciliation detects orphan. Per §3.12: user already has audit row, skip auto-insert. Run `markForImmediateDeletion`. User row gone. `banned_user_audit` row persists until original 12 months.

### 13.7 Firebase-cascade without ban, but with unfinished reports

Per §3.12. Deletion service auto-inserts a `banned_user_audit` row before `markForImmediateDeletion`. Reports get linked to the new audit row.

---

## 14. Frontend UX

### 14.1 Danger Zone on `/[locale]/owner/user`

The settings page at `app/[locale]/owner/user/page.tsx` gains a new section at the bottom, below the existing "set/update user location" row. Visual styling: separator above (`border-border-mild`), section heading, prose, button.

DANGER ZONE
Delete your account
This action will:
• Hide your listings from all users
• Show a "Scheduled for deletion" badge on your profile for 7 days
• Disable messaging to and from your account
• Sign you out
You can sign in within 7 days to restore your account. After 7 days, your account is permanently deleted.
[ Delete my account ]

Translation keys land in `DASHBOARD_PAGES` and `BUTTONS` per §15.

The button is styled with destructive emphasis (red border, red text). The codebase has no `text-destructive` utility per web audit §4; engineer picks the right Tailwind class set.

### 14.2 Confirmation dialog

Clicking "Delete my account" opens a dialog registered through `DialogManager` (per the existing `DialogManager.ts` and `dialogRegistry.ts`). The dialog reuses the existing `InfoDialog` component (or a dialog component shaped similarly — engineer reads the registry first).

The dialog's content branches on the user's authentication provider, determined the same way the codebase already determines provider elsewhere (engineer audits the existing detection — likely from `AuthUserDTO.providerId` per web audit §1, or via Firebase's `providerData[0].providerId`):

- **Email/password user**: password input + Cancel + Delete buttons.
- **Google user**: a single "Reauthenticate with Google and delete" button.
- **Facebook user**: same shape, with "Reauthenticate with Facebook and delete" — implemented so the case exists, but Facebook is not currently enabled per web audit §1. The dialog code includes the case; the trigger condition is dead code at launch.

The provider-branch is structural: a switch (or equivalent) on the resolved provider value, with a default branch that throws or logs.

### 14.3 Post-reauth deletion flow

After successful reauth, the dialog runs:

1. Force-refresh the ID token: `const freshToken = await currentUser.getIdToken(true)`.
2. Submit the deletion request with the fresh token explicit on the request: `axios.post(BACKEND_API + '/api/secure/user/me/delete', {}, { headers: { Authorization: 'Bearer ' + freshToken } })`.
3. Handle response:
   - HTTP 200 + `{ scheduledDeletionAt }`: continue to step 4.
   - HTTP 403 + `REAUTH_REQUIRED`: dialog shows "Please re-authenticate again," resets.
   - HTTP 403 + `USER_LOCKED_FROM_DELETION`: dialog shows the generic restricted message; button disabled.
   - HTTP 5xx: generic error with retry.
4. Write `sessionStorage.setItem('account-just-deleted', scheduledDeletionAt)`. (This is the **primary** trigger for the post-deletion dialog. See §14.4.)
5. Sign the user out: `await auth.signOut()`.
6. `SessionGuard` reacts to `auth.currentUser` becoming null and redirects from the protected `/owner/user` page to `/{locale}/`.

The post-deletion dialog opens on the next page's mount via the sessionStorage flag. The current confirmation dialog dismisses when the user is signed out.

### 14.4 Post-deletion confirmation dialog

After successful deletion, the user has been signed out and redirected away from any protected page.

The post-deletion, restoration, and ban-notice dialogs are opened by a single client-side initializer (`oglasino-web/src/components/client/initializers/AccountStateDialogsInit.tsx`) that subscribes to three Zustand store flags on `useAuthStore`: `accountJustDeleted`, `restored`, and `accountBanned`. Each flag is set by its respective trigger path (the delete-confirmation dialog's `handleDelete` for `accountJustDeleted`; the axios response interceptor reading `X-Account-Restored: true` for `restored`; the `syncUserToBackend` sign-in branch, the axios 403-`USER_BANNED` interceptor, the cross-browser `auth/user-disabled` request-interceptor branch, and `useAuthStore.mapAuthError` for `accountBanned`). The initializer reactively opens `DialogId.INFO_DIALOG` with the appropriate copy and clears the flag inside the same effect, so dismissal + refresh does not re-open. This replaces an earlier sessionStorage-based mechanism — the locale layout does not remount on `router.replace` within the same `[locale]` segment, so a mount-only sessionStorage read would not fire reliably.

For the post-deletion case, the delete-confirmation dialog's `handleDelete` sets `accountJustDeleted` (carrying the scheduled hard-delete date) before sign-out and redirect. `AccountStateDialogsInit` reactively opens `InfoDialog` with content:

✓ Your account has been scheduled for deletion.
It will be permanently deleted on {date}.
To restore your account, sign in within 7 days. After that, your account and all data are permanently removed.
[ Go to home ]

The date interpolates from the store value. The initializer clears `accountJustDeleted` in the same effect that opens the dialog — one-shot, does not re-open across refreshes.

If the user has never deleted their account, the flag is never set and no dialog renders. Other users are unaffected.

The dialog's "Go to home" button (and X button) dismiss the dialog; the user is already on the home page so no navigation is needed.

Translation keys in `DASHBOARD_PAGES` (under `dashboard.pages.account.deleted.dialog.*` — see §15).

### 14.5 Restoration dialog

The user has signed in and is on a public or home page (signing in does not require leaving a public page). The axios response interceptor inspects every response for `X-Account-Restored: true`. When detected, the interceptor sets a flag in `useAuthStore`: `useAuthStore.getState().setRestored(true)`.

`useAuthStore` carries a `restored: boolean` field and `setRestored(bool)` action.

`AccountStateDialogsInit` (per §14.4) subscribes to this flag. On flag flip to `true`:

1. Opens `InfoDialog` via `DialogManager` with restoration content (§15.2 keys).
2. Clears the flag in the same effect: `useAuthStore.getState().setRestored(false)`.

Dialog content:

✓ Welcome back.
Your account has been restored. The pending deletion has been cancelled.
[ Close ]

Auto-dismiss after 10 seconds via `setTimeout` (or via the dialog's native auto-dismiss option if `InfoDialog` supports one).

The dialog opens once per restore. Subsequent requests in the same session don't trigger the flag because `deletion_status` is now `ACTIVE` and the auth filter no longer sets `X-Account-Restored`.

The restoration dialog can open on the current page (the user is signed in; no protected-route redirect happens).

### 14.6 Profile page — "Scheduled for deletion" badge

When viewing another user's profile, if the response's `UserInfoDTO.state === 'PENDING_DELETION'`, a badge renders below the user's display name.

New component `ScheduledForDeletionBadge`:

```typescript
// src/components/client/badges/ScheduledForDeletionBadge.tsx
"use client";
import { Badge } from "@/components/shadcn/ui/badge";
import { useTranslations } from "next-intl";
import { TranslationNamespaceEnum } from "@/translations/types/TranslationNamespaceEnum";

export function ScheduledForDeletionBadge() {
  const t = useTranslations(TranslationNamespaceEnum.COMMON);
  return (
    <Badge variant="secondary" className="text-warning">
      {t("user.scheduled.for.deletion.label")}
    </Badge>
  );
}
```

Placement in `UserDetails.tsx` (per web audit §5): between the `displayName` row (line 117) and the `Rating` row (line 122). The component receives `userDetails: UserInfoDTO` and renders conditionally on `userDetails.state === 'PENDING_DELETION'`.

### 14.7 Phone number — already hidden for PENDING_DELETION

Per web audit §5, phone number is fetched lazily by `CallUserButton` via `getUserPhoneNumber(userId)` → backend `/secure/user/phoneNumber`. The endpoint should now also reject `PENDING_DELETION` users with a generic null response or 403. The frontend's `CallUserButton` is also gated client-side by `owner.allowPhoneCalling` and now also by `owner.state === 'ACTIVE'` (frontend gate is UX-only; backend check is load-bearing).

### 14.8 Messaging during grace period — chat header badge and disabled input

Per §3.2, during grace:

- "Scheduled for deletion" badge in chat header.
- Message input disabled.
- Existing messages render normally with actual name.

Per web audit §6, chat header is in `Messages.tsx`. Extend the `blocked` boolean at `Messages.tsx:90`:

```typescript
const cannotSend =
  blocking || blockedBy || activeChat.withUser.state === "PENDING_DELETION";
```

Add a corresponding notice (pattern from lines 246-259) for the new state:

```tsx
{
  activeChat.withUser.state === "PENDING_DELETION" && (
    <p className="text-warning">
      {t("messages.page.user.pending.deletion.notice")}
    </p>
  );
}
```

### 14.9 Chat — "Deleted User" rendering after hard delete

Per web audit §6 finding (high severity) and §3.10. Two call sites in `Messages.tsx` need null-safety:

- Line 225: `group.sender.firebaseUid === user.firebaseUid` → guard with `group.sender?.firebaseUid === user.firebaseUid`. Render `tCommon('user.deleted')` if null.
- Line 148: `activeChat.withUser.displayName` → when null, render the translated "Deleted User" string.

```tsx
const displayName = group.sender?.displayName ?? tCommon("user.deleted");
```

Translation key in `COMMON`: `common.user.deleted`.

The brief should also instruct the engineer to audit `Chats.tsx` (the conversation list) and apply the same pattern.

### 14.10 Ban-notice dialog

Two triggers for the dialog. Both call `auth.signOut()` and then set the `accountBanned` flag on `useAuthStore` via `setAccountBanned(true)`; `AccountStateDialogsInit` (per §14.4) reactively opens the dialog.

**Trigger 1 — during `syncUserToBackend`** (per §14.11): on `disabled: true` response OR `USER_BANNED`/`EMAIL_BANNED` rejection:

```typescript
await auth.signOut();
useAuthStore.getState().setAccountBanned(true);
// SessionGuard or natural navigation drops the user from protected pages.
// AccountStateDialogsInit opens the dialog when the flag flips and clears it.
```

**Trigger 2 — global axios interceptor** (per §14.12): on `403 + USER_BANNED` mid-session:

```typescript
auth.signOut();
useAuthStore.getState().setAccountBanned(true);
// Navigation completes; AccountStateDialogsInit opens the dialog.
```

When `accountBanned` flips to `true`, `AccountStateDialogsInit`:

1. Opens `InfoDialog` via `DialogManager` with ban-notice content.
2. Clears the flag in the same effect: `useAuthStore.getState().setAccountBanned(false)`.

Dialog content:

> **Your account is banned.**
>
> Your account has been disabled by Oglasino administration. If you believe this is in error, you can appeal by emailing **support@oglasino.com**.
>
> **If you want to delete your account permanently**, email **support@oglasino.com** with the subject "Account deletion request." We will respond within 30 days as required by data-protection law.
>
> Each account ban lasts at most 12 months. After that period, the email associated with the banned account becomes eligible for re-registration if you wish to create a new account.

The dialog does NOT identify the specific user, does NOT display a reason. Privacy reasoning per §4.7.

The dialog's "Go to home" button (or X button) dismisses; the user is already on the home page so no navigation.

Translation keys are the four `banned.dialog.*` keys in the `DIALOG` namespace — see §15.

**Race-safety**: if both triggers fire in rapid succession, both call `auth.signOut()` and both call `setAccountBanned(true)`. The Zustand flag is idempotent and the dialog opens once.

### 14.11 `syncUserToBackend` — read disabled and trigger ban dialog

Per web audit §1, `syncUserToBackend` calls `/auth/firebase-sync` and stores the resulting `AuthUserDTO` in the auth store. The flow in `authService.ts:118-130` is modified:

```typescript
export async function syncUserToBackend(
  firebaseUser: User,
  allowPreferenceCookies: boolean
): Promise<AuthUserDTO | null> {
  const token = await firebaseUser.getIdToken();
  await writeFirebaseTokenCookie(token);

  try {
    const response = await BACKEND_API.post<AuthUserDTO>(
      "/auth/firebase-sync",
      { allowPreferenceCookies }
    );
    const userData = { ...response.data, firebaseUid: firebaseUser.uid };

    if (userData.disabled) {
      await auth.signOut();
      useAuthStore.getState().setAccountBanned(true);
      return null;
    }

    return userData;
  } catch (error) {
    if (
      isErrorWithCode(error, "EMAIL_BANNED") ||
      isErrorWithCode(error, "USER_BANNED")
    ) {
      await auth.signOut();
      useAuthStore.getState().setAccountBanned(true);
      return null;
    }
    throw error;
  }
}
```

The `isErrorWithCode` helper inspects the axios error response for `{errors: [{code: 'X'}]}` in the body. Engineer implements this — no current helper exists per web audit §11.

### 14.12 Global 403 + `USER_BANNED` interceptor

Per web audit §10, today's axios interceptor doesn't handle 403 globally. The deletion feature adds:

```typescript
api.interceptors.response.use(
  (response) => {
    if (response.headers["x-account-restored"] === "true") {
      useAuthStore.getState().setRestored(true);
    }
    return response;
  },
  (error) => {
    if (error.response?.status === 403) {
      const errorBody = error.response.data;
      if (errorBody?.errors?.[0]?.code === "USER_BANNED") {
        auth.signOut();
        useAuthStore.getState().setAccountBanned(true);
        // Let navigation happen. Don't reject (we don't want every caller
        // to handle the error). Return a Promise that never resolves —
        // the redirect will unmount the caller.
        return new Promise(() => {});
      }
    }
    return Promise.reject(error);
  }
);
```

Other 403s (admin probes, review-eligibility) continue to fall through to per-call handling. Only `USER_BANNED` triggers global sign-out.

### 14.13 Review-image upload scope

Per web audit §8 and Phase 1 item E. The current `uploadImages(data.images, 'product', undefined, uploadOptions)` call in `reviewService.ts:68` changes to `uploadImages(data.images, 'review', undefined, uploadOptions)`.

Frontend adds `'review'` to the `UploadScope` union in `uploadImages.ts:52`:

```typescript
type UploadScope = "product" | "profile" | "chat" | "report" | "review";
```

The storage-key construction with the `review-` prefix is backend-side.

This is a future-data change — existing reviews keep non-prefixed keys.

### 14.14 `AuthUserDTO` and `UserInfoDTO` type updates

```typescript
export interface AuthUserDTO {
  // existing
  id: number;
  firebaseUid: string;
  displayName: string;
  email: string;
  // ...

  // new
  disabled: boolean;
  banReason: string | null;
  deletionStatus: "ACTIVE" | "PENDING_DELETION";
  scheduledDeletionAt: string | null;
}

export interface UserInfoDTO {
  // existing
  displayName: string;
  profileImageKey: string | null;
  // ...

  // new
  state: "ACTIVE" | "PENDING_DELETION" | "BANNED";
  scheduledDeletionAt: string | null;
}
```

`BANNED` wins over `PENDING_DELETION` in the wire-side state composition (mirrors the backend `UserState.resolve(disabled, deletionStatus)` algorithm — see §8.7). Counterparty UIs key off the strongest gating posture: when the user is both banned and pending-deletion, render "Banned," not "Scheduled for deletion."

These changes flow through the codebase via TypeScript compile errors until all consumers handle the new states.

---

## 15. Translations

### 15.1 `COMMON` namespace

| Key                                        | English value            |
| ------------------------------------------ | ------------------------ |
| `common.user.deleted`                      | "Deleted User"           |
| `common.user.scheduled.for.deletion.label` | "Scheduled for deletion" |

### 15.2 `COMMON_SYSTEM` namespace

| Key                                       | English value                                                              |
| ----------------------------------------- | -------------------------------------------------------------------------- |
| `common.system.account.restored.title`    | "Welcome back"                                                             |
| `common.system.account.restored.subtitle` | "Your account has been restored. The pending deletion has been cancelled." |

### 15.3 `BUTTONS` namespace

| Key                                       | English value                           |
| ----------------------------------------- | --------------------------------------- |
| `buttons.delete.account.label`            | "Delete my account"                     |
| `buttons.reauthenticate.and.delete.label` | "Reauthenticate with Google and delete" |
| `buttons.banned.go.home.label`            | "Go to home"                            |
| `buttons.account.deleted.go.home.label`   | "Go to home"                            |

### 15.4 `DASHBOARD_PAGES` namespace

| Key                                                          | English value                                                                                                    |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `dashboard.pages.danger.zone.label`                          | "Danger Zone"                                                                                                    |
| `dashboard.pages.delete.account.title`                       | "Delete your account"                                                                                            |
| `dashboard.pages.delete.account.bullet.listings`             | "Hide your listings from all users"                                                                              |
| `dashboard.pages.delete.account.bullet.profile.badge`        | "Show a 'Scheduled for deletion' badge on your profile for 7 days"                                               |
| `dashboard.pages.delete.account.bullet.messaging`            | "Disable messaging to and from your account"                                                                     |
| `dashboard.pages.delete.account.bullet.signout`              | "Sign you out"                                                                                                   |
| `dashboard.pages.delete.account.restore.note`                | "You can sign in within 7 days to restore your account. After 7 days, your account is permanently deleted."      |
| `dashboard.pages.delete.account.modal.title`                 | "Delete Account"                                                                                                 |
| `dashboard.pages.delete.account.modal.body.password`         | "This action is permanent after 7 days. Re-enter your password to confirm."                                      |
| `dashboard.pages.delete.account.modal.body.google`           | "This action is permanent after 7 days. We need to verify your identity before deleting your account."           |
| `dashboard.pages.delete.account.modal.password.label`        | "Password"                                                                                                       |
| `dashboard.pages.delete.account.modal.cancel.label`          | "Cancel"                                                                                                         |
| `dashboard.pages.account.deleted.dialog.title`               | "Your account has been scheduled for deletion."                                                                  |
| `dashboard.pages.account.deleted.dialog.scheduled.date`      | "It will be permanently deleted on {date}."                                                                      |
| `dashboard.pages.account.deleted.dialog.restore.instruction` | "To restore your account, sign in within 7 days. After that, your account and all data are permanently removed." |

### 15.5 `MESSAGES_PAGE` namespace

| Key                                          | English value                                                                     |
| -------------------------------------------- | --------------------------------------------------------------------------------- |
| `messages.page.user.pending.deletion.notice` | "This user has scheduled their account for deletion and cannot receive messages." |

### 15.6 `ERRORS` namespace

| Key                                | English value                                                                    |
| ---------------------------------- | -------------------------------------------------------------------------------- |
| `errors.reauth.required`           | "Please re-authenticate to continue."                                            |
| `errors.user.banned`               | "Your account has been banned."                                                  |
| `errors.email.banned`              | "This email address is not eligible for registration."                           |
| `errors.user.locked.from.deletion` | "Account deletion is currently restricted. Please contact support@oglasino.com." |
| `errors.user.not.pending.deletion` | "Your account is not in a pending-deletion state."                               |

### 15.7 `DIALOG` namespace

The four banned-dialog keys (`banned.dialog.title`, `banned.dialog.body.first`, `banned.dialog.body.delete.intro`, `banned.dialog.body.duration`) live in the `DIALOG` namespace alongside the post-deletion, restoration, and delete-confirmation dialog keys. No dedicated `BANNED_DIALOG` namespace exists in `TranslationNamespaceEnum`.

| Key                               | English value                                                                                                                                                                                |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `banned.dialog.title`             | "Your account is banned."                                                                                                                                                                    |
| `banned.dialog.body.first`        | "Your account has been disabled by Oglasino administration. If you believe this is in error, you can appeal by emailing support@oglasino.com."                                               |
| `banned.dialog.body.delete.intro` | "If you want to delete your account permanently, email support@oglasino.com with the subject 'Account deletion request.' We will respond within 30 days as required by data-protection law." |
| `banned.dialog.body.duration`     | "Each account ban lasts at most 12 months. After that period, the email associated with the banned account becomes eligible for re-registration if you wish to create a new account."        |

### 15.8 Translation key collision check

Per conventions Part 6 Rule 2. All new keys are leaves; no parent-leaf collisions exist.

---

## 16. Trust boundaries (Part 11 audit)

Per conventions Part 11, every value used in moderation, authorization, or state-transition decisions must be derived from auth, read from the server's DB, or otherwise unforgeable by the client.

### 16.1 The delete-account request

**Endpoint:** `POST /api/secure/user/me/delete`. Body: empty `{}`.

| Value used                      | Source                                                                    | Verification                                             |
| ------------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------------- |
| User identity                   | Derived from verified Firebase ID token → `OglasinoAuthentication.userId` | Server-derived. Frontend has no way to specify a userId. |
| `auth_time` for freshness check | Read from verified Firebase ID token claims                               | Server-derived from a cryptographically verified token.  |
| Whether user is locked          | Read from `user_deletion_locks` table                                     | Server-only data.                                        |
| Whether user is already pending | Read from `user.deletion_status` column                                   | Server-only data.                                        |

✓ All inputs are server-derived.

### 16.2 The restore-on-login flow

| Value used                    | Source                                                 | Verification    |
| ----------------------------- | ------------------------------------------------------ | --------------- |
| User identity                 | Derived from verified Firebase ID token                | Server-derived. |
| User's pending-deletion state | Read from cached `AuthenticatedUserDTO.deletionStatus` | Server-derived. |

Frontend has no role in triggering restore — it's a pure server-side reaction. ✓

### 16.3 The ban-on-registration check

| Value used         | Source                                    | Verification      |
| ------------------ | ----------------------------------------- | ----------------- |
| Email for hashing  | Extracted from verified Firebase ID token | Server-derived.   |
| Ban hash existence | Read from `banned_user_audit` table       | Server-only data. |

✓ Trust boundary intact.

### 16.4 The admin force-delete endpoint

| Value used      | Source                                                       | Verification                                                                   |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| Target userId   | Path parameter                                               | Client-supplied — but controller is admin-gated. Admin has implicit authority. |
| Caller identity | Derived from verified token + `userRole == ROLE_ADMIN` check | Server-derived.                                                                |

✓ Admin's role is the gate, not the userId.

### 16.5 The ban-notice dialog

No backend call for ban context. Static content rendered through translations. No trust boundary issue. ✓

### 16.6 What the client can never lie about

- Cause itself to be deleted by injecting a userId — request body is empty; auth context wins.
- Cause a banned user to register by lying about their email — email comes from verified Firebase token.
- Cause a restore-on-login by sending a request header `X-Account-Restored: true` — server reads response, not request.
- Cause a deletion to be locked or unlocked — admin-gated.
- Skip the reauth freshness check by sending an old token — server reads `auth_time` from cryptographically verified token.

### 16.7 The known weakness (acknowledgment, not violation)

**`ReviewRequestDTO.skipTrust` is client-controlled.** Per backend audit §11 and Phase 3 finding 2.3, the client controls whether the trust check runs and whether the resulting review is marked `verified`. Operator confirmed this is intentional. The deletion feature inherits this — when a deleted user's review survives anonymized, the `verified` flag is preserved.

---

## 17. Edge cases and policies

### 17.1 User has pending deletion → admin tries to disable

Admin disable succeeds. `User.disabled = true`, `banReason` set, `banned_user_audit` row inserted. Firebase `setDisabled(true)` + `revokeRefreshTokens`. Pending deletion remains pending. User cannot sign in to restore (Firebase rejects).

On Day 7, `executeScheduledDeletion` runs. User hard-deleted. `was_banned = true`. The `banned_user_audit` row remains for 12 months. Reports linked to the audit row.

### 17.2 User locked from deletion → user requests deletion

Per §3.8 / §8.1: lock check before any state change. Returns 403 + `USER_LOCKED_FROM_DELETION`.

### 17.3 User has pending deletion → admin places a deletion lock

Lock takes effect. Scheduled-deletion job skips locked users. `deletion_status` remains `PENDING_DELETION`. Once admin removes lock, next cron picks up the request.

### 17.4 User signs in but logs out before dialog shows

Restoration is server-side. First authenticated request triggers `cancelDeletionOnLogin`. State is `ACTIVE` regardless of client state. Dialog is informational only.

### 17.5 Multiple deletion attempts

User requests deletion → cancels via sign-in → requests deletion again. `user_deletion_requests` row updated, not duplicated. `user_deletion_audit_log` gets one new row per attempt (historical record).

### 17.6 Firebase deletion during grace period (covered in §3.12)

Reconciliation job detects orphan, runs `markForImmediateDeletion`. If open reports exist, auto-ban-hash inserted first; then hard delete with `triggered_by = 'firebase_cascade'`.

### 17.7 Server is down when scheduled-deletion cron should run

Cron picks up where it left off on next run. Pending requests where `scheduled_deletion_at <= now()` processed in batch. Missing one day = 24-hour delay. Acceptable.

If broken for extended period, batch size caps per-run processing. Operator monitors and increases batch size or cron frequency if backlog builds.

### 17.8 R2 deletion fails after DB commit

DB transaction commits (steps 1–12). R2 bulk delete fails. User is gone from DB; some images persist in R2 as orphans. Existing weekly orphan sweeper detects and cleans them up within its grace-period window. Acceptable.

### 17.9 Firebase deletion fails after DB commit

DB commits; Firebase `deleteUser(uid)` fails. Firebase user remains as orphan. No automated cleanup exists in this direction.

**Known gap.** Manual ops: operator runs a script to find Firebase users with no Postgres counterpart, deletes them via Firebase Admin SDK. Not in scope for v1. See §19.10.

Severity: low. Orphaned Firebase user can't do anything; if they sign in, `firebase-sync` creates a fresh row (or hits a banned-email block).

### 17.10 User has been hard-deleted, then their UID is referenced in a chat

Per §14.9 — chat null-safety branch renders "Deleted User."

### 17.11 User has been hard-deleted, then someone tries to view their profile

The profile page request to `/public/user/{userId}` returns 404 (existing behavior). Frontend handles 404 by showing "user not found" content. Indistinguishable from "user never existed."

### 17.12 Admin un-bans a user whose audit row has linked reports

Per §13.3 — un-banning removes the `banned_user_audit` row. The FK `ON DELETE CASCADE` on `report.banned_user_audit_id` deletes linked reports. Operator should be aware: un-banning erases the report history preserved by the ban.

### 17.13 Email is changed by user after Firebase sign-in

Currently the user-update flow doesn't allow email change. The deletion feature inherits this. The email used for ban-hashing is whatever was on the user row at hash insertion time. If email-change becomes a feature post-launch, the ban-hash update must be wired in.

### 17.14 Concurrent restore-on-login and admin-ban race

Theoretical race: user submits first request of a session at the exact moment an admin clicks Disable. If the cache version pre-eviction is what the filter sees, the filter sees `disabled=false` and falls through to `cancelDeletionOnLogin`. Meanwhile the admin's disable commits.

Outcome: user is `disabled=true`, `deletion_status='ACTIVE'`. Next request returns 403 `USER_BANNED`. Admin's intent is preserved; the user's restore happened but they can't do anything because they're banned.

Acceptable.

### 17.15 Deleted user's listings — Elasticsearch reindex lag

At Day 0, all listings flip to `INACTIVE`. Reindex is async. A user browsing the catalog at the exact moment Day 0 fires might see listings for a few seconds before reindex propagates. Acceptable — seconds-level lag.

---

## 18. Migration and rollout

Per §6, all schema goes into `V1__init_schema.sql` directly (pre-production decision).

Configuration seed: append `data/configuration/{user-deletion-keys}.sql` with all keys from §7.

No data migrations required (no existing data to migrate).

Rollout sequence:

1. Backend brief delivered, engineer implements.
2. Backend session ends with all tests passing.
3. Web brief delivered, engineer implements.
4. Web session ends with all tests passing.
5. Igor manually tests end-to-end locally: register, list a product, request deletion, see badge on profile, sign in to restore, see banner dialog, request deletion again, wait for cron (or trigger manually), confirm hard delete.
6. Igor manually tests ban flow: admin disable user, attempt sign-in, see ban-notice dialog, ban hash blocks re-registration, un-ban, sign-in works again.
7. Docs/QA session: update `state.md`, archive sessions, route adjacent observations to `issues.md`, draft `decisions.md` correction noted in §20.
8. Feature merged to `main`.
9. Mobile chat opens post-merge for `oglasino-expo` adoption.
10. Privacy Policy §9 and Terms §10 amendments applied (per §2). Lawyer review.
11. Pre-launch action items (§20) completed.

No staged or canary rollout — pre-production means deploy and iterate freely.

---

## 19. Future work

### 19.1 Admin UI for ban-with-reason

The admin ban-with-reason input modal is shipped at `oglasino-web/src/components/popups/dialogs/AdminBanUserDialog.tsx`. The dialog renders a required `Textarea` (max 500 characters, validated as `trimmed.length > 0`) for the ban reason, and the Confirm button is disabled until a non-empty trimmed reason is entered. The reason is passed server-side via `banUser(userId, reason)` and is stored in `users.ban_reason` and `banned_user_audit.ban_reason`. The reason is admin-internal — the banned user is not shown the reason at any point in the user-facing UI.

**Trigger to revisit:** n/a — shipped.

### 19.2 Admin UI for managing deletion locks

`user_deletion_locks` is populated via direct backend endpoints or SQL. No frontend UI.

**Trigger to revisit:** when operator has actual legal preservation requests or active abuse investigations.

### 19.3 User notification when locked

Privacy Policy commits to telling a user that their deletion is delayed. The `notified_at` field exists. The notification path does not.

**Trigger to revisit:** when email service is operational.

### 19.4 Notification sweep across other users' inboxes

Per §5 / Phase 1 item J: notifications other users hold that mention the deleted user remain with frozen text.

**Trigger to revisit:** when complaints arise about names persisting in stale notifications. Implementation requires adding `senderFirebaseUid` to notification docs going forward, plus a Day-7 sweep using that field.

### 19.5 Data export ("download my data") before deletion

GDPR right to data portability. Currently manual ops via email request.

**Trigger to revisit:** when manual export requests become time-consuming.

### 19.6 Multi-step deletion confirmation via email link

Some platforms send an email confirmation link before processing deletion. Oglasino's flow uses password-reauth in lieu of email confirmation. Both are valid GDPR approaches.

**Trigger to revisit:** if user feedback indicates the in-app flow is too easy (accidental deletions).

### 19.7 Granular per-data-type retention controls

The spec has one grace period and two retention periods. No way to say "delete my listings now but keep my reviews."

**Trigger to revisit:** unlikely to be needed; deferred indefinitely.

### 19.8 Pre-existing report-submit trust-boundary check

Per Phase 3 finding 2.4, the report-submit endpoint has no authorization check on the reporter-to-target relationship. Routed to `issues.md` as a separate bug.

**Trigger to revisit:** when ops sees abuse of the report system.

### 19.9 Mobile adoption

`oglasino-expo` adopts the user-deletion feature in a separate Mastermind chat post-merge.

**Trigger to revisit:** automatic after merge.

### 19.10 Reverse Firebase orphan cleanup

Per §17.9. The reconciliation job only handles Postgres→Firebase. Reverse direction is manual.

**Trigger to revisit:** when an audit reveals non-trivial accumulation of one-sided Firebase users.

---

## 20. Pre-launch action items

Required before public launch. Owner: Igor.

### 20.1 Set up support@oglasino.com mailbox

The ban-notice dialog directs banned users here. Privacy Policy and Terms also reference it.

### 20.2 Set up privacy@oglasino.com mailbox

Privacy Policy commits to 30-day response window for privacy rights requests.

### 20.3 Privacy Policy §9 amendment

Applied 2026-05-19: `legal/privacy-policy-draft.md` §9 "Right to erasure" now includes:

> "If you are unable to use the self-service deletion flow — for example, because your account has been disabled — contact support@oglasino.com to request deletion and we will respond within 30 days as required by GDPR."

Send to lawyer for review.

### 20.4 Terms of Use §10 amendment

`legal/terms-of-use-draft.md` §10 (Termination) already references the support@oglasino.com appeals path. Lawyer review covers the existing wording; no further textual delta required.

### 20.5 Provision `REVALIDATE_SECRET`

On Vercel (prod + preview) and DigitalOcean droplet — same value on both. Generate via `openssl rand -base64 32`. See `infra/overview/secret-inventory.md`.

### 20.6 Provision `WEB_REVALIDATE_URL`

On DigitalOcean droplet — direct Vercel deployment URL (bypasses Cloudflare router worker per `issues.md` 2026-05-19 entry). See `infra/overview/secret-inventory.md`.

### 20.7 Lawyer review of legal drafts

Privacy Policy §9 (deletion rights) and Terms §10 (ban + deletion rules) drafts. See `legal/lawyer-handoff.md`.

### 20.8 Native-translator review

Native-translator review of placeholder RS/CNR/RU translations for User Deletion keys.

### 20.9 Translations seeded for all four locales

Per §15. Backend SQL seed inserts all keys for SR, EN, RU, CNR.

### 20.10 OpenAI translation provider — for any new keys

For the new namespace `BANNED_DIALOG` and any other new keys, ensure translations exist in all four locales.

### 20.11 Operator-side training on ban-with-reason endpoint

Until admin UI ships (post-launch per §19.1), bans run via Postman or direct API. Operator needs documented commands.

### 20.12 Decision log entry at ship time

Applied 2026-05-19 — see `decisions.md` 2026-05-19 entries (Backend → web cache revalidation, Banned-state UI visibility, Backlog-triage closed, Cache-invalidation profile fix, Pre-production schema fold).

### 20.13 `decisions.md` 2026-05-15 correction

Per Phase 3 finding 2.5, the 2026-05-15 entry that overstated the Report FK problem needs correction. Docs/QA handles in post-merge cleanup.

### 20.14 Update `state.md` backlog items

When feature ships:

- Move "User deletion" from backlog to `shipped`.
- Remove "Account-disabling & token-revocation enforcement" from backlog (subsumed).

---

End of spec.
