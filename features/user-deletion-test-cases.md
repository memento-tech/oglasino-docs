# User Deletion — Manual Test Cases

**Purpose:** end-to-end manual verification of the User Deletion feature before launch.
**Audience:** Igor running QA against a live local or stage stack.
**Status:** authored 2026-05-19; tracks the feature as shipped at backend session 13 + frontend session N.

---

## Prerequisites

Before running any case, confirm the test environment is in a clean state:

1. Backend running locally with Postgres, Redis, Elasticsearch, Firebase Admin SDK.
2. Web running locally with Next.js dev server.
3. `REVALIDATE_SECRET` env var set identically on both backend and web.
4. `WEB_REVALIDATE_URL` env var set on the backend pointing at the local web (`http://localhost:3000/api/revalidate`).
5. Cron jobs disabled or scheduled so they don't fire mid-test (recommend disabling `user.deletion.hard.delete.cron`, `user.deletion.audit.purge.cron`, and `user.deletion.firebase.reconciliation.cron` during testing, or accepting their daily cadence).
6. Test accounts ready:
   - **Email/password account** (User EA) with one active product, one inactive product (user-deactivated), and at least one chat with another user.
   - **Google account** (User GA) with one active product.
   - **Counterparty account** (User C) — separate from EA and GA — used for cross-viewer observations.
   - **Admin account** (User Admin) with admin role.
   - Optionally a **Facebook account** if Facebook sign-in is enabled (currently dead per the audit; skip if not enabled).
7. Browser devtools handy with Network tab open for cache-invalidation observations.

After each test case, return the system to a clean state before running the next (delete or un-modify test users as needed).

---

## Case 1 — Self-deletion by email/password user

**Goal:** verify a user can request deletion, see the confirmation dialog, and be signed out into the post-deletion dialog.

**Steps:**
1. Sign in as User EA via email/password.
2. Navigate to `/{locale}/owner/user`.
3. Scroll to the Danger Zone section at the bottom.
4. Click "Delete my account".
5. The confirmation dialog opens. Confirm the bullet list reads:
   - "Hide your listings from all users"
   - "Show a 'Scheduled for deletion' badge on your profile for 7 days"
   - "Disable messaging to and from your account"
   - "Sign you out"
6. Confirm the restore note below reads: "You can sign in within 7 days to restore your account. After 7 days, your account is permanently deleted."
7. Enter the password.
8. Click "Delete my account" (the dialog's confirm button).
9. Expected outcome:
   - The dialog closes.
   - The post-deletion dialog opens on the home page.
   - The dialog body says "Your account has been scheduled for deletion." with the scheduled date displayed.
10. Click "Go to home" in the post-deletion dialog. Confirm it dismisses and you remain on the home page.

**Backend state to verify:**
- `users.deletion_status = 'PENDING_DELETION'` for User EA.
- `user_deletion_requests` row inserted for User EA with `status = 'PENDING'` and `scheduled_deletion_at` set to ~7 days from now.
- `user_deletion_audit_log` row inserted with `triggered_by = 'user_request'`.
- All of User EA's products that were `ACTIVE` are now `INACTIVE` with `deactivated_by_system = true`.
- Products that were already `INACTIVE` before deletion remain `INACTIVE` with `deactivated_by_system = false`.

---

## Case 2 — Self-deletion by Google user

**Goal:** verify the Google sign-in user can request deletion through the popup-reauth flow.

**Steps:**
1. Sign in as User GA via Google.
2. Navigate to `/{locale}/owner/user`.
3. Click "Delete my account" in the Danger Zone.
4. The confirmation dialog opens with a single "Reauthenticate with Google and delete" button (no password field).
5. Click the button. A Google popup opens.
6. Complete the Google reauth in the popup.
7. After the popup closes, expected outcome:
   - The post-deletion dialog opens on the home page.
   - User GA is signed out.

**Verify backend state:** same as Case 1.

---

## Case 3 — Restoration by sign-in during grace period

**Goal:** verify a `PENDING_DELETION` user can restore their account by signing back in.

**Setup:** User EA in `PENDING_DELETION` state from Case 1.

**Steps:**
1. Open a fresh browser session (or new private window).
2. Navigate to the home page.
3. Click "Sign in".
4. Enter User EA's credentials and sign in.
5. Expected outcome:
   - Sign-in completes successfully.
   - The restoration dialog opens.
   - The dialog reads "Welcome back" and "Your account has been restored. The pending deletion has been cancelled."
6. Click "Close" or wait for the auto-dismiss (~10 seconds).
7. Navigate to `/{locale}/owner/user`. Confirm you are signed in normally.

**Backend state to verify:**
- `users.deletion_status = 'ACTIVE'` for User EA.
- `user_deletion_requests` row for User EA now has `status = 'CANCELLED'`, `cancelled_at` set, `cancelled_reason = 'user_login'`.
- `user_deletion_audit_log` row has `cancelled_at` set.
- All of User EA's products that were flipped to `INACTIVE` with `deactivated_by_system = true` are now back to `ACTIVE` with `deactivated_by_system = false`.
- Products that were `INACTIVE` with `deactivated_by_system = false` remain `INACTIVE` (user-deactivated, not restored).

---

## Case 4 — Admin bans an active user

**Goal:** verify admin ban flips the user out, hides their content, and produces the audit record.

**Setup:** User EA in `ACTIVE` state with at least one active product.

**Steps:**
1. Sign in as User Admin.
2. Navigate to the admin Users page.
3. Find User EA in the list.
4. Click the ban action (button or icon).
5. The ban-with-reason dialog opens. Enter a reason (free text required).
6. Submit the ban.
7. Expected outcome:
   - The user's row indicator shows "Banned" (or equivalent label).
   - A success toast appears.

**Backend state to verify:**
- `users.disabled = true` for User EA.
- `users.ban_reason` populated with the entered text.
- `banned_user_audit` row inserted with `email_hash` matching User EA's email (SHA-256 of `lowercase(trim(email))`), `ban_reason`, `banned_at`, `retention_until` set ~12 months from now.
- All of User EA's products are now `INACTIVE` with `deactivated_by_system = true`.
- Firebase Auth user record's `disabled` flag is `true`.
- `redisUserAuth` and `redisUserInfo` caches are evicted for User EA.

**Cross-viewer observation (still as User Admin, but representative of any signed-out browser):**
8. Open a separate browser or private window (signed out).
9. Navigate to `/{locale}/user/<User EA's id>` directly via URL.
10. Expected outcome: 404 page (no profile rendered, indistinguishable from "user does not exist").
11. Navigate to `/{locale}/product/<one of User EA's product ids>/<slug>` directly via URL.
12. Expected outcome: 404 page (banned-user products no longer accessible via direct link).

---

## Case 5 — Banned user attempts to sign in

**Goal:** verify the ban-notice dialog fires on sign-in attempts.

**Setup:** User EA is banned (from Case 4).

**Steps:**
1. Open a fresh browser session.
2. Navigate to the home page.
3. Click "Sign in".
4. Enter User EA's credentials.
5. Click the sign-in button.
6. Expected outcome:
   - Sign-in fails.
   - The login form does NOT display the raw Firebase error "auth/user-disabled" or any similar raw error text.
   - The ban-notice dialog opens.
7. Verify the dialog content:
   - Title: "Your account is banned."
   - Body mentions appealing via `support@oglasino.com`.
   - Body mentions emailing `support@oglasino.com` with subject "Account deletion request" for account deletion.
   - Body mentions 12-month re-registration eligibility.
8. Click "Go to home". The dialog dismisses; you remain on the home page, signed out.

**Repeat for Google sign-in:** the same dialog should fire when User EA (if previously Google-authenticated and now banned) attempts Google sign-in.

---

## Case 6 — Admin unbans a previously-banned user (not in pending deletion)

**Goal:** verify unban restores the user, restores their products, and creates the unban audit row.

**Setup:** User EA banned (from Case 4); not in pending deletion.

**Steps:**
1. Sign in as User Admin.
2. Navigate to the admin Users page.
3. Find User EA.
4. Click the unban action.
5. The unban dialog opens (with optional reason field).
6. Optionally enter a reason. Submit the unban.
7. Expected outcome:
   - User EA's row no longer shows "Banned".
   - Success toast appears.

**Backend state to verify:**
- `users.disabled = false` for User EA.
- `users.ban_reason` cleared (null).
- `banned_user_audit` row for User EA deleted.
- `user_admin_action_audit` row inserted with `action = 'UNBAN'`, hashed user id + email, the optional reason (if provided), admin id of User Admin, `performed_at` set.
- All of User EA's products that were `INACTIVE` with `deactivated_by_system = true` are now `ACTIVE` with `deactivated_by_system = false`.
- Products that were `INACTIVE` before the ban (user-deactivated) remain `INACTIVE`.

**Cross-viewer observation:**
8. Open a separate signed-out browser.
9. Navigate to `/{locale}/user/<User EA's id>` directly.
10. Expected outcome: profile renders normally.
11. Navigate to `/{locale}/product/<one of User EA's product ids>/<slug>`.
12. Expected outcome: product page renders normally.

---

## Case 7 — Admin bans a user already in pending deletion

**Goal:** verify ban can layer on top of pending deletion; banned wins over pending in the UI.

**Setup:**
- User EA in `ACTIVE` state (re-registered or restored from earlier cases).
- Put User EA into `PENDING_DELETION` (run Case 1 again).

**Steps:**
1. Sign in as User Admin.
2. Find User EA in the admin Users page.
3. Confirm User EA's row shows the "Pending deletion" indicator.
4. Ban User EA (run Case 4 from step 4).
5. Expected outcome:
   - User EA's row now shows the "Banned" indicator (banned wins over pending-deletion in the UI per `UserState.resolve` composition).
   - The "Pending deletion" indicator may or may not still show alongside — depends on the admin UI's composition rule.
6. Sign out of admin. Open a separate signed-out browser.
7. Navigate to `/{locale}/user/<User EA's id>`. Expected: 404 (banned user 404 wins over pending-deletion badge).
8. Navigate to `/{locale}/product/<User EA's product>`. Expected: 404.

**Backend state to verify:**
- `users.disabled = true` AND `users.deletion_status = 'PENDING_DELETION'`.
- Both `banned_user_audit` row exists AND `user_deletion_requests` row exists.

---

## Case 8 — Admin unbans a user still in pending deletion (the edge case)

**Goal:** verify unban does NOT restore products when the user is still in pending deletion. This is the explicit B2 design exception.

**Setup:** User EA in both `disabled = true` AND `deletion_status = 'PENDING_DELETION'` (from Case 7).

**Steps:**
1. Sign in as User Admin.
2. Find User EA.
3. Unban User EA.
4. Expected outcome:
   - User EA's row no longer shows "Banned".
   - User EA's row still shows "Pending deletion".
   - The unban toast fires.

**Backend state to verify (the critical assertion of this case):**
- `users.disabled = false` for User EA.
- `users.ban_reason` cleared.
- `banned_user_audit` row for User EA deleted.
- `user_admin_action_audit` row inserted with `action = 'UNBAN'`.
- `users.deletion_status` still `PENDING_DELETION`.
- **User EA's products remain `INACTIVE` with `deactivated_by_system = true`.** They are NOT restored to `ACTIVE` because the user is still in pending deletion.

5. The user can still restore the account by signing in (Case 3 logic): if User EA signs in within the remaining grace period, products restore correctly.

---

## Case 9 — Hard delete after grace period

**Goal:** verify Day 7 hard delete works end-to-end.

**Setup:** User EA in `PENDING_DELETION` for 7 days (or manually adjust `scheduled_deletion_at` to a past timestamp for testing).

**Steps:**
1. Trigger the hard-delete cron manually (or wait for the scheduled run).
2. Confirm logs show "Deletion executed for user <ID>".

**Backend state to verify:**
- User EA's `users` row deleted (no row exists for that id).
- All of User EA's products deleted via JPA cascade.
- All of User EA's product images deleted from R2.
- User EA's Firebase Auth record deleted.
- Approved reviews authored by User EA: kept with `reviewer_id = NULL`.
- Pending-approval reviews authored by User EA: deleted.
- Disapproved reviews authored by User EA: deleted.
- Reviews about User EA: deleted.
- `user_deletion_requests` row for User EA: deleted (cascaded via FK).
- `user_deletion_audit_log` row: updated with `actual_deletion_at` set.

**Cross-viewer observation:**
3. Sign in as User C (counterparty who had a chat with User EA).
4. Navigate to `/{locale}/messages`.
5. Open the chat with User EA (which still exists in Firestore).
6. Expected outcome:
   - User EA's name renders as "Deleted User" (or the localized equivalent).
   - The chat history is still readable.
   - The message input is disabled.

7. Navigate to `/{locale}/user/<User EA's id>` directly.
8. Expected outcome: 404.

---

## Case 10 — Banned-user deletion via support email

**Goal:** verify admin can force-delete a banned user via the admin-only endpoint (the right-to-erasure path for banned users).

**Setup:** User EA banned, in `ACTIVE` state (not pending deletion). User EA emails support@oglasino.com requesting account deletion.

**Steps:**
1. Sign in as User Admin.
2. Call `POST /api/admin/users/{User EA's userId}/force-delete` via Postman or admin tooling (no UI in v1).
3. Provide the admin reason in the request body.

**Expected outcome:**
- Force-delete runs immediately (no grace period).
- All of User EA's data is deleted per the same sequence as Case 9.
- `user_deletion_audit_log` row inserted with `triggered_by = 'admin_action'`, `was_banned = true`.
- `banned_user_audit` row remains in the database — persists until original 12-month retention expires from `banned_at`.

---

## Case 11 — Re-registration with a banned email

**Goal:** verify the 12-month re-registration prevention works.

**Setup:** User EA's email is in `banned_user_audit` from Case 4 or Case 10. User EA's account row may or may not exist (depending on whether case 10 ran).

**Steps:**
1. Open a fresh browser session.
2. Navigate to `/sign-up` (or wherever registration lives).
3. Attempt to register a new account with User EA's email (and a different password).
4. Expected outcome:
   - Registration appears to start but immediately fails.
   - The ban-notice dialog opens (same dialog as Case 5).
5. Verify backend state:
   - The Firebase Auth user that was just created via `createUserWithEmailAndPassword` has been deleted (no orphan in Firebase).
   - No new `users` row was created in Postgres.

**Repeat for Google sign-in:** if User EA's email is a Google account, attempting Google sign-in with that email should produce the same ban-notice dialog and clean up the Firebase user.

---

## Case 12 — Firebase orphan reconciliation (without open reports)

**Goal:** verify the weekly reconciliation job hard-deletes a user whose Firebase account is gone, when no open reports exist.

**Setup:**
- User EA in `ACTIVE` state with no open reports authored by or against them.
- Manually delete User EA's Firebase Auth record via Firebase Admin Console or Admin SDK.

**Steps:**
1. Trigger the reconciliation cron manually (or wait for the weekly run).
2. Confirm logs show "Orphaned user in Firebase: <User EA's id>" and subsequent immediate-deletion call.

**Expected outcome:**
- User EA's `users` row deleted.
- All cascaded data deleted (per Case 9 sequence).
- `user_deletion_audit_log` row inserted with `triggered_by = 'firebase_cascade'`, `was_banned = false`.
- No `banned_user_audit` row created (because no abuse threshold tripped).

---

## Case 13 — Firebase orphan reconciliation (with open reports)

**Goal:** verify the auto-ban-on-cascade rule per spec §3.12 — open reports trigger a ban-hash insertion before hard delete.

**Setup:**
- User EA in `ACTIVE` state.
- Create at least one open report against User EA (`report.reported_user_id = <User EA's id>`, `finished = FALSE`).
- Manually delete User EA's Firebase Auth record.

**Steps:**
1. Trigger the reconciliation cron manually.
2. Confirm logs show the orphan-detected message and the auto-ban-then-delete sequence.

**Expected outcome:**
- `banned_user_audit` row inserted with `ban_reason = 'auto: deletion with open reports'`, hashed email, 12-month retention.
- User EA's `users` row deleted.
- The open report's `reporter_id` and `reported_user_id` are now nullified, with `banned_user_audit_id` linking to the just-inserted audit row.
- User EA's email cannot be re-registered for 12 months (run Case 11 to verify).

---

## Case 14 — Cache-invalidation cross-viewer test

**Goal:** verify backend → web cache invalidation works on user state transitions.

**Setup:**
- User EA in `ACTIVE` state.
- User C (different user) is signed in on a separate browser.
- User C navigates to User EA's profile at `/{locale}/user/<User EA's id>` (this populates the SSR cache for that page).

**Steps:**
1. Sign in as User Admin in a third browser.
2. Ban User EA (Case 4).
3. **Immediately** (within seconds, not minutes) refresh User C's page (normal browser refresh, NOT hard refresh).
4. Expected outcome: User C now sees the 404 page for User EA's profile. The SSR cache invalidated correctly.

5. Unban User EA (Case 6).
6. **Immediately** refresh User C's page.
7. Expected outcome: User C now sees User EA's profile rendering normally.

**Network observation:**
- On the backend logs during steps 2 and 5, observe one POST to `/api/revalidate` with body `{"tags": ["user:<User EA's id>"]}`. Expected response status: 200.

If the cache does NOT invalidate without a hard refresh, there's still a cache-config bug somewhere — investigate before launch.

---

## Case 15 — Messaging gating during pending deletion (counterparty view)

**Goal:** verify counterparties cannot message a user in pending deletion.

**Setup:**
- User EA has a chat with User C.
- Put User EA in `PENDING_DELETION` (Case 1).

**Steps:**
1. Sign in as User C.
2. Navigate to `/{locale}/messages`.
3. Open the chat with User EA.
4. Expected outcome:
   - Chat header shows "Scheduled for deletion" badge next to User EA's name.
   - Message input is disabled.
   - Notice "This user has scheduled their account for deletion and cannot receive messages." renders below the input.
   - Previous messages are still readable.
   - User EA's name renders as their actual `displayName` (NOT "Deleted User" — that label is post-hard-delete only).

5. Sign in as User EA. Expected: signed out (cannot sign in while in pending deletion without restoring).
6. Restore User EA via sign-in (Case 3).
7. Sign in as User C and refresh the messages page. Expected: badge gone, input enabled, notice gone.

---

## Case 16 — Messaging gating after ban (counterparty view)

**Goal:** verify counterparties cannot message a banned user.

**Setup:**
- User EA has a chat with User C (in `ACTIVE` state).
- Admin bans User EA (Case 4).

**Steps:**
1. Sign in as User C.
2. Navigate to `/{locale}/messages`.
3. Open the chat with User EA.
4. Expected outcome:
   - Chat header shows "Banned" badge next to User EA's name.
   - Message input is disabled.
   - Notice "This user has been banned and cannot receive messages." renders below the input.
   - Previous messages are still readable with User EA's actual `displayName`.

5. Admin unbans User EA (Case 6).
6. User C refreshes the messages page. Expected: badge gone, input enabled, notice gone.

---

## Case 17 — Messaging gating when banned wins over pending deletion (counterparty view)

**Goal:** verify the composition rule — when a user is both pending-deletion AND banned, the UI shows "Banned" not "Scheduled for deletion".

**Setup:**
- User EA has a chat with User C.
- Put User EA in `PENDING_DELETION` (Case 1).
- Admin bans User EA (Case 4 from step 4).

**Steps:**
1. Sign in as User C.
2. Navigate to `/{locale}/messages`.
3. Open the chat with User EA.
4. Expected outcome:
   - Chat header shows "Banned" badge (NOT "Scheduled for deletion").
   - Banned notice renders below the input (NOT the pending-deletion notice).
   - Message input is disabled (same as Cases 15 and 16).

---

## Case 18 — Admin Users page state indicators (list view)

**Goal:** verify the list-page row indicators render correctly for each state.

**Steps:**
1. Sign in as User Admin.
2. Navigate to the admin Users page.
3. Filter / scroll to find users in each state:
   - Active user → no state indicator on the row.
   - Banned user → "Banned" indicator visible.
   - Pending-deletion user → "Pending deletion" indicator visible.
   - Locked-from-deletion user → "Locked" indicator visible.
   - Banned + pending-deletion user → "Banned" indicator (composition rule).
   - Banned + locked user → both "Banned" and "Locked" indicators.

4. Apply the "Pending deletion" filter chip. Confirm only pending-deletion users show.
5. Apply the existing "disabled" filter chip alongside. Confirm the AND composition works.

---

## Case 19 — Admin Users page state info popup

**Goal:** verify the info popup displays correct state metadata.

**Setup:** A banned user with a recorded `ban_reason` and a known banning admin id. A locked user with a recorded `reason` and admin id.

**Steps:**
1. Sign in as User Admin.
2. Open the info popup for the banned user.
3. Expected:
   - Shows ban metadata: who banned, when, reason.
   - Shows deletion state if applicable.
   - Shows lock state if applicable.

4. Open the popup for the locked user.
5. Confirm lock metadata displays (who locked, reason, expiry if set).

6. Verify the "system-banned" fallback copy renders when `bannedByAdminId IS NULL` (i.e., Firebase-cascade auto-ban per spec §3.12).

---

## Case 20 — Admin Users detail page state indicators

**Goal:** verify the just-added detail-page indicators mirror the list page.

**Setup:** Users in each of the four states (banned, pending-deletion, locked, multi-state).

**Steps:**
1. Sign in as User Admin.
2. From the Users list, click into each test user's detail page.
3. Expected outcome: the same state indicators render on the detail page (in the row beneath the user's identity block, with the info-popup icon nearby).
4. The indicators row does NOT render at all for a clean user with no applicable state.

---

## Case 21 — Reauthentication freshness window (negative test)

**Goal:** verify the 5-minute reauth window is enforced.

**Setup:** User EA signed in with a session token older than 5 minutes (or manually decrement `auth_time` in the token if possible).

**Steps:**
1. Navigate to `/{locale}/owner/user`.
2. Click "Delete my account" in the Danger Zone.
3. The confirmation dialog opens. (No reauth has been done yet in this session beyond the original sign-in.)
4. Click the delete button (do NOT enter password / reauth).
5. Expected outcome: backend returns 403 `REAUTH_REQUIRED`, the dialog displays "Please re-authenticate again" or equivalent.
6. Reauthenticate, then complete the deletion. Now it succeeds.

---

## Case 22 — Deletion lock (admin override)

**Goal:** verify a locked user cannot self-delete.

**Setup:**
- Sign in as User Admin.
- Lock User EA from deletion via the admin lock dialog.

**Steps:**
1. Sign out of admin, sign in as User EA.
2. Navigate to `/{locale}/owner/user`.
3. Attempt to delete (Case 1 steps).
4. Expected outcome: backend rejects with 403 `USER_LOCKED_FROM_DELETION`. The dialog displays the generic restricted message ("Account deletion is currently restricted. Please contact support@oglasino.com").

5. Sign back in as admin; unlock User EA.
6. As User EA, attempt deletion again. Now it succeeds.

---

## Case 23 — Locked user during scheduled deletion

**Goal:** verify the scheduled-deletion cron skips locked users.

**Setup:**
- User EA in `PENDING_DELETION` with `scheduled_deletion_at` in the past (override for testing).
- Admin locks User EA from deletion.

**Steps:**
1. Trigger the hard-delete cron manually.
2. Expected outcome: cron logs "Skipping deletion for locked user <User EA's id>". User EA's row remains in `PENDING_DELETION`, products remain INACTIVE.

3. Admin removes the lock.
4. Trigger the cron again.
5. Expected outcome: hard delete now runs to completion.

---

## Case 24 — Audit purge cron

**Goal:** verify the weekly audit-purge cron deletes expired records.

**Setup:**
- A `user_deletion_audit_log` row with `retention_until` in the past.
- A `banned_user_audit` row with `retention_until` in the past.
- A `user_admin_action_audit` row with `retention_until` in the past.
- A linked report row pointing at the `banned_user_audit` row.

**Steps:**
1. Trigger the audit-purge cron manually.
2. Expected outcome:
   - All three expired audit rows deleted.
   - The linked report deleted via cascade from `banned_user_audit`'s `ON DELETE CASCADE` FK.
   - Log line "Purge: audit=N, bans=N, adminActions=N" with non-zero counts.

---

## Case 25 — Concurrent restore + admin ban race (acceptance check)

**Goal:** verify the documented race outcome from spec §17.14.

**Setup:** User EA in `PENDING_DELETION` state.

**Steps:**
1. Set up two browsers: one for User EA (about to sign in), one for User Admin (about to ban).
2. Coordinate timing: trigger both the User EA sign-in and the User Admin ban as close to simultaneously as possible.
3. Possible outcomes (any of these is acceptable):
   - A. User EA restores first, then User Admin bans. Final state: `disabled = true`, `deletion_status = 'ACTIVE'`, User EA cannot use the platform.
   - B. User Admin bans first, then User EA tries to sign in but fails (Firebase rejects). Final state: `disabled = true`, `deletion_status = 'PENDING_DELETION'`, banned-user dialog shows.
   - C. Both succeed in some interleaving. Final state should still converge to `disabled = true` with consistent admin intent preserved.

The acceptance criterion is **admin intent is always preserved** — User EA cannot escape a ban by racing to sign in.

---

## What this doc does NOT test

- Mobile (Expo) flows — separate Mastermind chat post-merge.
- Cross-browser/cross-device sessions in detail (Case 25 is a simplified version).
- Performance under load (out of scope for manual QA).
- The Cloudflare router worker's involvement in revalidation routing — see issues.md 2026-05-19 "Backend → web cache revalidation may not work in production via oglasino.com domain" for tracking.

---

## When to consider this feature shipped

Every case above passes. Any failure routes to a bug fix, then re-test. Mobile adoption opens its own Mastermind chat post-feature-merge.