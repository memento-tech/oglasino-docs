# Session summary

**Repo:** oglasino-docs
**Branch:** main (single-branch workflow)
**Date:** 2026-05-23
**Task:** Produce a tester-perspective summary of the user-facing surface of the user-deletion feature. This summary becomes an input to a forthcoming web-engineer audit brief for the `user-deletion-flow` QA topic in `oglasino-web`. It is a Mastermind input, not a topic entry — no edit to `topics.ts` in this session.

## Implemented

- Read `features/user-deletion.md` (1924 lines), `features/user-deletion-test-cases.md`, and the relevant `decisions.md` entries (2026-05-18 User Deletion shipped, 2026-05-19 Banned-state UI / cache-invalidation / schema-fold / backlog-triage, 2026-05-20 Messaging entry's user-deletion touchpoints).
- Produced the tester-perspective summary below.
- No file edits, no `topics.ts` work, no config-file work — read-and-summarise only per brief.

---

## Tester-perspective summary — user-facing surface of user-deletion

### Entry points

A tester finds the user-facing surfaces in five places:

1. **Self-deletion start.** Account settings page, "Danger Zone" section at the bottom (below the existing set/update-user-location row). One surface only — there is no second "delete" entry point in the dashboard, profile dropdown, or anywhere else.
2. **Restoration.** Any sign-in surface — the existing portal sign-in button. Restoration is *implicit*: signing in during the grace window cancels the deletion. There is no dedicated "restore my account" link or button. The system restores server-side as soon as the auth filter sees a `PENDING_DELETION` user's verified token.
3. **Re-registration block (banned email).** Sign-up form *and* Google sign-in (and Facebook, when enabled). The block fires from `/auth/firebase-sync` on the backend; the user sees the ban-notice dialog as a result.
4. **Ban-notice dialog.** Two trigger paths: at sign-in (any provider — Firebase rejects credentials when `setDisabled = true`, or backend returns `USER_BANNED` / `EMAIL_BANNED`) and mid-session (any authenticated request returning `403 + USER_BANNED` via the global axios interceptor). The dialog itself is rendered by the root layout on the next mount via a sessionStorage flag.
5. **Admin-side surfaces.** Admin Users list page (state indicators, ban / unban / lock actions, info popup with ban metadata) and the admin Users detail page (mirrors the list-page indicators, info popup). Admin force-delete (for the support-email path) and deletion-lock are admin-only.

### The user-facing flow itself

The self-deletion flow:

1. Click "Delete my account" in the Danger Zone.
2. **Confirmation dialog** opens. Content branches by auth provider:
   - **Email/password user**: bullet list (hide listings, show "Scheduled for deletion" badge for 7 days, disable messaging, sign you out), restore note ("You can sign in within 7 days to restore your account. After 7 days, your account is permanently deleted."), **password input**, Cancel + "Delete my account" buttons.
   - **Google user**: same prose, single "Reauthenticate with Google and delete" button (no password input). Clicking opens the Google popup.
   - **Facebook user**: shipped as dead code (case implemented, trigger condition is dead — Facebook not currently enabled).
3. On successful reauth, the dialog force-refreshes the ID token (`getIdToken(true)`) and POSTs the empty-body delete request.
4. The dialog closes; the user is signed out automatically; `SessionGuard` redirects from the protected dashboard to `/{locale}/`.
5. On the home page mount, the **post-deletion dialog** opens via a sessionStorage flag with the scheduled date ("Your account has been scheduled for deletion. It will be permanently deleted on {date}."), a "Go to home" button, and an X to dismiss. One-shot — does not re-open across refreshes.

Cancellation surfaces inside the dialog itself: Cancel button + X. The dialog can be dismissed before reauth without side effects (no deletion request is sent until the reauth step completes and the user clicks the confirm button).

### The 7-day grace window — what's visible

Per the spec's foundational policy decision (`decisions.md` 2026-05-19 "Banned-state UI visibility" / spec §3.1):

**The deleting user (now signed out):**
- Cannot sign in for any purpose other than restoring the account (signing in cancels the deletion).
- Cannot send or receive messages.

**Other users viewing the deleting user's surface:**
- Profile page **remains publicly visible** with a "Scheduled for deletion" badge rendered below the display name. Phone number is hidden. Listings are hidden (filtered via Elasticsearch `productState == ACTIVE` — products flipped to `INACTIVE` with `deactivated_by_system = true` at Day 0). Reviews about the user remain visible. Reviews authored by the user remain visible with their normal display name (because the user row is still active during grace).
- In chat: the conversation continues to render the user's actual display name in existing messages. The **chat header shows the "Scheduled for deletion" badge**. The message input is disabled with notice text: "This user has scheduled their account for deletion and cannot receive messages."
- Other users can still file reports against the deleting user (the 7-day reporting window runs in parallel with the grace window).

The "profile-stays-visible-with-status-badge" decision is the foundation: the badge + reporting window together form the affordance for other users to report misconduct. Hiding the profile would destroy that affordance.

### Cancel / restore during the grace window

There is **no explicit "cancel" UI button**. Cancellation is signing in:

1. The user opens the portal in any browser (fresh session or new private window).
2. Clicks "Sign in" and authenticates normally. (Firebase Auth still accepts their credentials during grace — we did *not* call `setDisabled` at Day 0, only `revokeRefreshTokens`.)
3. Sign-in completes; the backend's `FirebaseAuthFilter` detects `deletionStatus = 'PENDING_DELETION'`, calls `cancelDeletionOnLogin`, flips the user back to `ACTIVE`, and restores all system-deactivated products to `ACTIVE`. The response carries header `X-Account-Restored: true`.
4. The frontend's axios response interceptor reads the header and sets a Zustand flag; the root layout opens the **restoration dialog**: "Welcome back. Your account has been restored. The pending deletion has been cancelled." Auto-dismiss after ~10 seconds (or Close button).

Restoration is server-side and idempotent — even if the user closes the tab immediately after the API call hits, the deletion is already cancelled and the next sign-in finds an active account. The dialog is informational only.

### What "hard delete" looks like from a tester's view (Day 7 vs Day 6)

Day 6 (last day of grace): profile visible with "Scheduled for deletion" badge; listings hidden; messaging gated with badge + notice. User can still restore by signing in.

Day 7 (after the `user.deletion.hard.delete.cron` runs — default daily at 02:00):

- **Profile page** at `/{locale}/user/<id>` returns 404. Indistinguishable from "user never existed."
- **All listings** gone (cascade-deleted from Postgres; reindexed out of Elasticsearch).
- **Chat history** with this user still exists on the counterparty's side, but the deleted user's name now renders as the translated **"Deleted User"** placeholder (key `common.user.deleted`) wherever it appeared. The message *content* is preserved.
- **Reviews authored by the deleted user** that were *approved*: kept, but the author renders as "Deleted User."
- **Reviews authored by the deleted user** that were *pending* or *disapproved*: gone.
- **Reviews about the deleted user**: all gone (any approval state).
- **Sign-in attempt** with the deleted user's credentials: Firebase rejects (the Firebase Auth record was deleted at hard-delete). No ban-notice dialog (they weren't banned, just deleted) — the user sees the standard Firebase-rejected-credentials error.

A subtle distinction the tester should look for: during grace, the badge surfaces "Scheduled for deletion" via `UserInfoDTO.state === 'PENDING_DELETION'`. After hard delete, the `/api/auth/firebase/{uid}` endpoint returns null (the row is gone), and the frontend interprets null as the cue to render "Deleted User" in chat / review contexts. Two different mechanisms; the tester should confirm both render correctly.

### Re-registration with a banned email (within 12 months)

A user (or a different person who knows the banned email) attempts to sign up or sign in via Google with a previously-banned email:

1. Firebase Auth lets them through — Firebase has no knowledge of our bans.
2. Backend's `/auth/firebase-sync` hashes the token's email and checks `banned_user_audit`. If an unexpired hash matches, backend deletes the just-created Firebase user and returns `403 + EMAIL_BANNED`.
3. Frontend signs them out, sets the `account-banned` sessionStorage flag, the next page-mount opens the **ban-notice dialog**.

The dialog content is the same as the regular ban-notice (described next). Crucially, the dialog **does not identify the specific user, does not confirm any specific email is banned, does not display a reason** — this is a deliberate privacy choice to prevent the dialog from becoming an email-enumeration vector. A would-be enumerator submitting random emails would see the same dialog every time, indistinguishable from "this email is in fact banned."

After 12 months from the original ban (`retention_until`), the audit-purge cron removes the hash row and the email becomes re-registrable.

### Ban-with-reason — admin-internal or user-facing?

**The ban reason is admin-internal only.** The user is never told why they were banned.

- The reason is captured in an admin-side dialog at ban time (per Case 4: "The ban-with-reason dialog opens. Enter a reason (free text required).").
- The reason is stored in `users.ban_reason` and `banned_user_audit.ban_reason`.
- The reason surfaces on the admin Users page in the state info popup (admin-only).
- The banned user, when they encounter the ban-notice dialog, sees only generic text — no specific user identification, no reason, no banning admin. They are pointed to `support@oglasino.com` for appeals and for the email-driven deletion path (since banned users cannot use the self-service flow).

The ban-notice dialog content (testable verbatim):

> **Your account is banned.**
>
> Your account has been disabled by Oglasino administration. If you believe this is in error, you can appeal by emailing **support@oglasino.com**.
>
> **If you want to delete your account permanently**, email **support@oglasino.com** with the subject "Account deletion request." We will respond within 30 days as required by data-protection law.
>
> Each account ban lasts at most 12 months. After that period, the email associated with the banned account becomes eligible for re-registration if you wish to create a new account.

A "Go to home" button (and X) dismisses. The user is already on the home page so no navigation happens.

### Edge cases the tester needs to know

From the PR-review patches and decisions log (spec §3 / §17, decisions.md 2026-05-18 + 2026-05-19):

- **Locked-from-deletion users.** A user covered by a `user_deletion_locks` row attempting self-deletion gets the generic message "Account deletion is currently restricted. Please contact support@oglasino.com." The user is *not* told they are specifically locked (the lock reason is admin-internal). No admin lock-management UI ships at launch — locks are created via direct backend endpoint or SQL.
- **Reauth freshness window.** If the user's `auth_time` claim is older than 5 minutes when they submit the delete request, the backend rejects with `REAUTH_REQUIRED` → dialog shows "Please re-authenticate again" and resets. The dialog itself does not automatically re-prompt for reauth — the user has to retry.
- **Banned + pending-deletion composition.** If a user is both banned and in pending-deletion, counterparty UIs render the strongest gating posture: **"Banned" wins over "Scheduled for deletion"** on the wire (`UserState.resolve(disabled, deletionStatus)`). Chat header shows "Banned" badge with banned-notice text — NOT "Scheduled for deletion."
- **Admin unban while user is pending-deletion.** Products are *not* auto-restored to `ACTIVE`. The pending-deletion intent is preserved; only `deactivated_by_system = true` products get restored when the user is in `deletionStatus = ACTIVE` at the time of unban.
- **Banned-user direct URLs.** A banned user's profile returns 404 on direct URL (`/{locale}/user/<id>`). Their product pages also return 404 (their products flipped to `INACTIVE` with `deactivated_by_system = true` at ban time).
- **Firebase-side deletion during grace.** If the user deletes themselves out of Firebase directly (e.g., from Google Account settings) during grace, the weekly reconciliation cron detects the orphan, upgrades to immediate hard delete with `triggered_by = 'firebase_cascade'`. If unfinished reports exist against the user (or more than the threshold authored by them — default 3), an auto-ban hash is inserted *first* so the 12-month re-registration block applies.
- **Multiple deletion attempts.** A user can request deletion, restore by sign-in, then request deletion again. The `user_deletion_requests` row is updated in place (UNIQUE on `user_id`); the audit log gets a new row per attempt for historical record.
- **Concurrent restore + admin ban race** (acceptance check): admin intent always wins. A user racing to sign in cannot escape a ban.
- **`deactivated_by_system = true` flag is load-bearing.** It distinguishes system-deactivations (restorable on unban) from user-deactivations (not restored). A tester restoring an unbanned user should confirm that user-deactivated products stay deactivated while system-deactivated ones flip back to active.
- **Banned-user deletion via support email.** Banned users cannot use the in-app flow (cannot sign in). They email `support@oglasino.com`; an admin verifies the request matches the banned account and calls `POST /api/admin/users/{userId}/force-delete`. Hard-delete runs immediately, no grace period. The `banned_user_audit` row persists until its original 12-month retention (so the email stays blocked for the full window even after deletion).
- **Hard-delete-of-a-banned-user marker.** `user_deletion_audit_log.was_banned = true` records the historical fact; no behavioral consequence at the audit-log layer.

### Translation namespaces touched

The user-deletion frontend pulls keys from seven namespaces (spec §15):

- `COMMON` — `common.user.deleted` ("Deleted User"), `common.user.scheduled.for.deletion.label` ("Scheduled for deletion")
- `COMMON_SYSTEM` — restoration-dialog title + subtitle ("Welcome back" / "Your account has been restored…")
- `BUTTONS` — `buttons.delete.account.label`, `buttons.reauthenticate.and.delete.label`, `buttons.banned.go.home.label`, `buttons.account.deleted.go.home.label`
- `DASHBOARD_PAGES` — Danger Zone section, modal bullets / titles / labels, post-deletion dialog title + scheduled-date interpolation + restore-instruction
- `MESSAGES_PAGE` — `messages.page.user.pending.deletion.notice` (chat input notice during grace)
- `ERRORS` — `errors.reauth.required`, `errors.user.banned`, `errors.email.banned`, `errors.user.locked.from.deletion`, `errors.user.not.pending.deletion`
- `BANNED_DIALOG` (new namespace per spec §15.7) — `banned.dialog.title`, `banned.dialog.body.first`, `banned.dialog.body.delete.intro`, `banned.dialog.body.duration`

EN copy is final. RS / CNR / RU are placeholders pending native-translator review (per `state.md` "Tasks remaining" and spec §20.8).

---

## Files touched

- None. Read-and-summarise session.

## Tests

- None. No code or docs changes.

## Cleanup performed

- None needed (read-and-summarise session).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no files added, no dead links introduced.
- Part 4a (simplicity): N/A — no code, no abstractions added/considered/removed.
- Part 4b (adjacent observations): two observations flagged in "For Mastermind."
- Part 5 (session summary template): confirmed — summary written to both `.agent/2026-05-23-oglasino-docs-qa-preparation-20.md` and `.agent/last-session.md` with identical content. `<n>=20` reasoning explained below under "For Mastermind."
- Part 6 (translations): N/A this session (no translation work; namespace list is summarised from the spec for the downstream web audit).
- Part 11 (trust boundaries): N/A — spec §16 already carries the trust-boundary audit; not re-litigated here.

## Known gaps / TODOs

- None — the brief was scoped tightly and the output is the summary above.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-and-summarise session, no code or abstractions.
  - Considered and rejected: nothing — no design decisions in scope.
  - Simplified or removed: nothing.

- **`<n>` determination.** The brief instructs determining `<n>` by listing `.agent/` for `*-qa-preparation-*.md` "including archive-pointer stubs per conventions Part 5 amendment 2026-05-18." My `.agent/` folder contains **zero** such files (no archive-pointer stubs were left for sessions 1–19; only `2026-05-22-oglasino-docs-cookies-closing-5.md` and `2026-05-23-oglasino-docs-google-analytics-v1-2.md` are present). `sessions/` contains 19 archived `oglasino-docs-qa-preparation-*.md` files numbered 1 through 19. I chose **n=20** to preserve sequential ordering and avoid collision with the existing archives. The strict-literal reading of the rule would have produced n=1, which would have collided with `sessions/2026-05-14-oglasino-docs-qa-preparation-1.md`. Worth a Mastermind decision: backfill archive-pointer stubs for sessions 1–19, or accept the gap as the amendment-was-formalised-after-those-sessions consequence.

- **Possible spec contradiction — admin ban-with-reason UI status.** Spec §19.1 says the admin UI for ban-with-reason is **future work** ("the existing admin button does not yet pass a reason. The button keeps working with a default placeholder; a future web brief adds a reason-input modal."). But the manual test cases at `features/user-deletion-test-cases.md` Case 4 step 5 expect a **shipped** ban-with-reason dialog with a free-text required reason input ("The ban-with-reason dialog opens. Enter a reason (free text required)."). `state.md` describes "ban-with-reason flow" as live in the User Deletion shipped feature. The web audit will confirm what actually shipped; flagging as an inconsistency the spec text has not been updated to reflect.

- **Heads-up for the web audit — surfaces I read as shipped but cannot verify from docs alone.** All sourced from the spec and the test cases; the web audit will confirm each landed on disk:
  - The "Scheduled for deletion" badge on the profile page (spec §14.6, between `displayName` row and `Rating` row in `UserDetails.tsx`).
  - The chat header "Scheduled for deletion" badge + disabled input + pending-deletion notice text (spec §14.8).
  - The "Banned" badge in chat for counterparty view when `state === 'BANNED'`, including the BANNED-wins-over-PENDING-DELETION composition behaviour (test Case 17).
  - The post-deletion dialog opening via the `account-just-deleted` sessionStorage flag on root-layout mount (spec §14.4).
  - The restoration dialog opening via the `X-Account-Restored: true` axios response header (spec §14.5).
  - The ban-notice dialog opening via the `account-banned` sessionStorage flag, with two trigger paths (spec §14.10–§14.12).
  - The 404 behaviour for banned-user direct URLs to product pages (test Case 4 step 11–12, decisions.md 2026-05-19 "Banned-state UI visibility").

- **Adjacent observations flagged (Part 4b) — not fixing in this session:**
  - **Spec §19.1 appears stale.** Per the contradiction above. Severity: medium (it could mislead a future reader into thinking the ban-reason UI is still to come). File: `features/user-deletion.md` §19.1. I did not fix this because the change is substantive (a status flip from "future work" to "shipped"), which requires an upstream drafter per conventions Part 3; this is also outside the scope of a read-and-summarise session.
  - **Archive-pointer stubs missing for sessions 1–19** of the `qa-preparation` slug in `oglasino-docs/.agent/`. Per conventions Part 5 amendment 2026-05-18, the stubs should be present to keep the `<n>` count correct. Severity: low (the system works because Docs/QA-on-Docs/QA can fall back to `sessions/` for sequential counting, as I did this session). File: `oglasino-docs/.agent/` directory. I did not fix because the amendment may have been intended to apply prospectively only.

- **Closure gate.** No config-file edits drafted this session and none required by the work. Read-only docs session.
