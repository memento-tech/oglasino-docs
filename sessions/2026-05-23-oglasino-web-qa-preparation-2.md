# Session summary

**Repo:** oglasino-web
**Branch:** feature/qa-preparation
**Date:** 2026-05-23
**Task:** Author a new QA topic entry `user-deletion-flow` (`type: 'flow'`) — Stop 1: audit and structural draft.

## Implemented

Stop 1 of a two-stop session. No code on disk has changed. `topics.ts` will be appended in Stop 2. Below: the audit of the user-deletion user-facing surfaces, the trust-boundary check, the spec §19.1 reality check, the structural sections of the topic entry, the 2–3 option proposals for the judgment sections, and one Known-issue pitfall draft.

## Audit — user-deletion user-facing surfaces (web)

### Danger Zone — the self-delete entry

- Route: `/[locale]/owner/user` (`app/[locale]/owner/user/page.tsx`). One surface only; no second entry in the dashboard, profile dropdown, or header.
- Position: a red-bordered card at the bottom of the page, after the email / displayName / shortBio / phone-number inputs, the marketplace + region/city block, the five toggle settings (necessary / notifications / emails / promo / phone calls), and the save row. The card is the last element rendered (`page.tsx:396–423`).
- Visual: `AlertTriangle` red icon + `tDash('danger.zone.label')` heading ("Danger Zone"), `tDash('delete.account.title')` sub-heading ("Delete my account" or equivalent), four bullets (`bullet.listings`, `bullet.profile.badge`, `bullet.messaging`, `bullet.signout`), `tDash('delete.account.restore.note')` italic restore reminder, and a red outlined "Delete my account" Button. Right-aligned on `md+`, centered below mobile.
- Click handler: opens `DialogId.DELETE_ACCOUNT_CONFIRMATION_DIALOG` via `useDialogStore.openDialog` with `{ userId: user.id }`.

### Delete-confirmation dialog

`src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx`. A `DrawerDialog` with three provider branches resolved at mount via `resolveProvider()` reading `auth.currentUser?.providerData[0]?.providerId`:

- **`'password'`** (default / email-password / `providerId === 'password'` / no provider): renders a password `Input` (`isPassword`), Cancel + "Delete my account" buttons. Submits call `EmailAuthProvider.credential(currentUser.email, password)` + `reauthenticateWithCredential(currentUser, credential)`.
- **`'google.com'`**: no password field. Single confirm button labelled `tButtons('reauthenticate.and.delete.label')` ("Reauthenticate with Google and delete"). Submit calls `reauthenticateWithPopup(currentUser, googleProvider)`.
- **`'facebook.com'`**: same UI as the Google branch, calls `reauthenticateWithPopup(currentUser, facebookProvider)`. **Shipped but dead — `facebookProvider` is wired but Facebook sign-in is not currently exposed in any sign-in surface I read; the branch is unreachable in production for an unmodified user.** (Per brief and per the Docs/QA summary's "Facebook-as-dead-code" framing.)
- **`'unsupported'`** (any other `providerId`): both `dialogDescription` and the confirm button switch off — only the Cancel button renders, with `errorMessage` derivation for the unsupported case. Reachable only if a future provider gets added without updating `resolveProvider`.

The dialog title (`tDialog('delete.account.dialog.title')`) and body (`delete.account.dialog.body.password` for password, `delete.account.dialog.body.google` for Google; Facebook reuses the Google body) are translated. The dialog is non-dismissible by outside click when `loading` is true (`closableOutside={!loading}`) and has no X-close (`addCloseButton={false}`).

### Reauth wiring + delete request

`DeleteAccountConfirmationDialog.handleDelete`:

1. Reauth (provider branch). On failure, sets `errorMessage = tErrors('reauth.required')` and returns; the dialog stays open for retry. Firebase's raw error code is NOT surfaced (the user knows what they just typed).
2. `useAuthStore.getState().setDeletionInFlight(true)` — flips a Zustand flag the `UseTokenRefresh` listener consults to suppress its redundant firebase-sync POST during the deletion handshake (auth contract "C-6", documented at `DeleteAccountConfirmationDialog.tsx:96–98`).
3. `currentUser.getIdToken(true)` — forced refresh. Per the inline comment, without `true` Firebase returns the cached token whose `auth_time` predates the reauth and the backend rejects with `REAUTH_REQUIRED`.
4. `deleteCurrentUser(freshToken)` (`src/lib/service/reactCalls/userService.ts:118–128`): `BACKEND_API.post('/secure/user/me/delete', {}, { headers: { Authorization: 'Bearer ' + freshToken } })`. The request body is `{}` (empty), and the explicit `Authorization` header bypasses the cached-token interceptor (`src/lib/config/api.ts:111–113`). Returns `{ scheduledDeletionAt: string }`.
5. `revalidateUserCache(userId)` — `await app/actions/revalidateUserCache.ts`'s tag-bust on `user:${userId}` to refresh other viewers' SSR cache.
6. `useAuthStore.getState().setAccountJustDeleted({ scheduledDeletionAt })` — flips the post-deletion flag **before** `signOut` so the home-page mount (after router.replace) sees a populated value when `AccountStateDialogsInit` reads it.
7. `auth.signOut()` — Firebase sign-out fires `onIdTokenChanged(null)`.
8. `clearFirebaseTokenCookie()` — wrapped in try/catch so a cookie-clear blip doesn't pollute the deletion success with a "system error" toast.
9. `router.replace('/'+locale)` — navigates the user out of the protected dashboard.
10. Error branches: `REAUTH_REQUIRED` → `errorMessage = tErrors('reauth.required')`; `USER_LOCKED_FROM_DELETION` → `errorMessage = tErrors('user.locked.from.deletion')` + `setLocked(true)` (disables the password input and the confirm button); other → `tErrors('system.error')`. In the locked branch the only escape is the Cancel button.
11. `finally` clears `deletionInFlight` and the local `loading` state.

There is **no explicit `closeDialog()` call** on success — the team's note (lines 125–128): the post-deletion dialog opens via `AccountStateDialogsInit` flipping the `accountJustDeleted` Zustand flag, and `openDialog(INFO_DIALOG)` replaces the delete-confirmation dialog through the single-slot `useDialogStore`. An explicit `closeDialog` here would race and tear down the freshly-opened post-deletion dialog.

### Post-deletion dialog

Opened by `src/components/client/initializers/AccountStateDialogsInit.tsx`, NOT by a sessionStorage handshake. The effect at `AccountStateDialogsInit.tsx:79–104` subscribes to `useAuthStore.accountJustDeleted`. On non-null, it formats `scheduledDeletionAt` via `formatDeletionDate` (which falls back from `'cnr'` → `'sr'` because `Intl` lacks Montenegrin), opens `DialogId.INFO_DIALOG` with:

- title: `tDialog('account.deleted.dialog.title')`
- body: a two-paragraph `<div>` — `account.deleted.dialog.scheduled.date` interpolating the formatted date, followed by `account.deleted.dialog.restore.instruction`.
- `closeButtonLabel`: `tButtons('account.deleted.acknowledge.label')`
- `onClose`: closes the dialog and calls `router.refresh()` (so the now-signed-out home page replaces the stale signed-in render visible behind the dialog).

The flag is cleared (`setAccountJustDeleted(null)`) inside the same effect, so a refresh after dismissal does not re-open. **The mechanism is a Zustand flag, NOT `sessionStorage.setItem('account-just-deleted', ...)`** — the Docs/QA summary (and spec §14.4) describes a sessionStorage handshake; the live code uses a reactive store-state flip with the explicit rationale at `AccountStateDialogsInit.tsx:71–78` ("the locale layout does not remount on `router.replace` within the same `[locale]` segment, so a mount-only sessionStorage read would never fire"). The tester observation is identical, but the trigger mechanism differs from the spec. Flagged in "For Mastermind."

### Restoration dialog

Opened by the same `AccountStateDialogsInit.tsx` (effect at lines 118–134). Triggered by the `restored` Zustand flag, which is set by the axios response interceptor:

```ts
// src/lib/config/api.ts:26–32
instance.interceptors.response.use((response) => {
  if (response.headers['x-account-restored'] === 'true') {
    useAuthStore.getState().setRestored(true);
  }
  return response;
}, ...)
```

The frontend never calls a "restore" endpoint. The backend's `FirebaseAuthFilter`, on seeing an authenticated request from a `PENDING_DELETION` user, restores the account server-side and sets `X-Account-Restored: true` on the response. The frontend reads the header off any normal authenticated request after sign-in.

On non-null `restored`, the effect opens `INFO_DIALOG` with title `tDialog('account.restored.title')` ("Welcome back" or equivalent), `dialogDescription: tDialog('account.restored.subtitle')`, `autoDismissAfterMs: 10_000` (10 seconds). On dismissal (click or auto-dismiss), `onClose` runs `revalidateUserCache(user.id)` then `router.refresh()` — the inline comment explains that without busting the cached `UserInfoDTO` tag, the PENDING_DELETION badge sticks until the 5-minute revalidate window elapses.

### Ban-notice dialog

Opened by `AccountStateDialogsInit.tsx` (effect at lines 51–69). Triggered by the `accountBanned` Zustand flag — which is set by **three** call sites:

1. **`syncUserToBackend` sign-in branch** (`src/lib/service/reactCalls/authService.ts:148–164`): if `/auth/firebase-sync` returns `userData.disabled === true`, or throws with code `EMAIL_BANNED` / `USER_BANNED`, the function calls `auth.signOut()` and `useAuthStore.getState().setAccountBanned({ reason: null })`.
2. **Mid-session 403 + `USER_BANNED`** via the axios response interceptor (`src/lib/config/api.ts:51–67`): when a 403's first error code is `USER_BANNED`, the interceptor signs out, flips the store flag, and returns a never-resolving Promise (suppresses per-call rejection handling). Other 403 codes fall through.
3. **Cross-browser ban catch** in the axios request interceptor (`api.ts:118–134`): when `user.getIdTokenResult()` throws with Firebase code `auth/user-disabled` (Browser B mints a token after Browser A has banned the user; Firebase has invalidated the refresh token), the request interceptor flips the same store flag and signs out.
4. **(Also)** `useAuthStore.mapAuthError` (`useAuthStore.ts:22–27`): a sign-in attempt where Firebase rejects with `auth/user-disabled` (the credentials are accepted but the account is banned at the Firebase Auth level) flips the same flag and suppresses the raw Firebase message on the sign-in form. So a banned user attempting sign-in via email/password OR Google triggers the same dialog.

On non-null `accountBanned`, the effect opens `INFO_DIALOG` with a three-paragraph body — `tDialog('banned.dialog.body.first')` / `body.delete.intro` / `body.duration` — under title `banned.dialog.title`, with `closeButtonLabel: tButtons('banned.go.home.label')`. The custom `onClose` calls `closeDialog()` then `router.replace('/' + routingLocale)` so the close button actually navigates the user home (the default `INFO_DIALOG` close only calls `onClose`). The flag is cleared inside the same effect.

**The dialog mechanism is the Zustand store, not the `account-banned` sessionStorage flag described in the spec / summary.** Same rationale as the post-deletion case.

### Translation namespaces touched

The user-deletion frontend pulls keys from the following namespaces (each verified against a `useTranslations(TranslationNamespaceEnum.<NS>)` call in the audited components):

- `COMMON` — `user.scheduled.for.deletion.label` (`ScheduledForDeletionBadge.tsx`), `user.deleted` (chat header / message fallbacks), `user.banned.label` (chat Badge in `Messages.tsx:170`).
- `COMMON_SYSTEM` — restoration / general system-level strings; not directly referenced by the dialogs I read, but the spec lists it as a touched namespace.
- `BUTTONS` — `delete.account.label`, `reauthenticate.and.delete.label`, `banned.go.home.label`, `account.deleted.acknowledge.label`.
- `DASHBOARD_PAGES` — `danger.zone.label`, `delete.account.title`, `delete.account.bullet.listings` / `bullet.profile.badge` / `bullet.messaging` / `bullet.signout`, `delete.account.restore.note`.
- `DIALOG` — `delete.account.dialog.title`, `delete.account.dialog.body.password`, `delete.account.dialog.body.google`, `delete.account.dialog.password.label`, `button.cancel.label`, `account.deleted.dialog.title`, `account.deleted.dialog.scheduled.date`, `account.deleted.dialog.restore.instruction`, `account.restored.title`, `account.restored.subtitle`, `banned.dialog.title`, `banned.dialog.body.first`, `banned.dialog.body.delete.intro`, `banned.dialog.body.duration`. **All ban-notice keys are in the `DIALOG` namespace**, NOT a dedicated `BANNED_DIALOG` namespace. The Docs/QA summary lists `BANNED_DIALOG (new namespace per spec §15.7)` as a separate namespace; the live code keeps everything under `DIALOG`. Flagged in "For Mastermind."
- `MESSAGES_PAGE` — `user.pending.deletion.notice` (chat input notice during grace; `Messages.tsx:281`), `user.banned.notice` (chat input notice when banned; `Messages.tsx:288`).
- `ERRORS` — `reauth.required`, `user.locked.from.deletion`, `system.error`. The summary lists `errors.user.banned`, `errors.email.banned`, `errors.user.not.pending.deletion` — those server-emitted codes flow through `isErrorWithCode(err, '<CODE>')` in `authService.syncUserToBackend` and `api.ts`'s 403 interceptor and do not directly resolve to `tErrors(...)` calls on a dialog. The keys exist for completeness; not all are rendered to users.

### "Scheduled for deletion" badge — surfaces

- `src/components/client/badges/ScheduledForDeletionBadge.tsx` — a shadcn `<Badge variant="secondary">` rendering `tCommon('user.scheduled.for.deletion.label')`. Pure presentation; reads no state.
- Profile-page badge: `src/components/client/UserDetails.tsx:124–128` — rendered inside the `UserInfoBlock` immediately after the displayName row and the optional `FollowUserButton`, before the `Rating` row. Conditioned on `userDetails.state === 'PENDING_DELETION'`.
- Chat-header badge: `src/messages/components/Messages.tsx:167` — same component, rendered next to the peer's displayName. Conditioned on `peer.state === 'PENDING_DELETION'`.
- **No `ScheduledForDeletionBadge` on banned users anywhere.** The profile page returns 404 for banned users (see below); the chat-header BANNED case uses a different Badge (`Messages.tsx:168–172`) with a `variant="secondary"` plus inline `text-red-600` and `tCommon('user.banned.label')` ("Banned"). The two badges are visually distinct.

### Chat grace + ban UI

`src/messages/components/Messages.tsx`:

- `peerPendingDeletion = peer?.state === 'PENDING_DELETION'` (line 97).
- `peerBanned = peer?.state === 'BANNED'` (line 98).
- `cannotSend = blocking || blockedBy || peerPendingDeletion || peerBanned || peerDeleted` (line 99) — composes the disabled state for `MessageInput`.
- Chat header (lines 155–173): renders the peer's displayName (or `tCommon('user.deleted')` if `peer.withUser` is null — the hard-deleted case), then the appropriate badge.
- Notice text below the messages list (lines 277–305): one of `user.pending.deletion.notice`, `user.banned.notice`, `blocking.label`, `blocked.by.label`, each in its own bordered red-italic line. Multiple notices stack — a peer who is both pending-deletion and blocked would show two notices side-by-side. The "banned wins over pending-deletion" composition is **server-side only** (`UserState.resolve(disabled, deletionStatus)` per spec §3.5a); the frontend just reads whichever `state` value the backend returns on `UserInfoDTO`.
- The hard-deleted-peer detection lives in `src/messages/utils/deletedUser.ts` (`isDeletedPeer(otherUid, peer)`). The dropdown collapses to "Delete chat" only when `peerDeleted` is true (lines 184–224) — Profile / Report / Block / Unblock are all hidden because there's no living target.

### Profile-page badge + 404 behaviour

- `app/[locale]/(portal)/(public)/user/[userId]/page.tsx`: `getUserForId(userId)` returns `UserInfoDTO | null`. Null branch renders `<NotFound title={tCommon('user.page.not.found.title')} description={tCommon('user.page.not.found.description')} />` (line 64–70). That's the full 404 component.
- The backend returns 404 from `/public/user?id=<id>` for hard-deleted users (no row) AND for banned users (per spec §3.5a — the public profile endpoint returns 404 to mirror the post-hard-delete privacy posture). So a banned user's profile page is observationally indistinguishable from a hard-deleted user's profile page.
- During the 7-day grace window the user's row is still active and `UserInfoDTO.state` is `'PENDING_DELETION'`; the page renders normally with the `ScheduledForDeletionBadge` between displayName and Rating.

### Product-page 404 behaviour

- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`: when `getPortalProductDetails(productId)` returns null, the page renders a **centered "product not found" message** (lines 77–82), NOT the `<NotFound>` component used on the user page:

  ```tsx
  if (!productDetails || !productDetails.id)
    return (
      <div className="flex h-[40vh] w-full items-center justify-center">
        <p>{tCommon('product.page.not.found.title')}</p>
      </div>
    );
  ```

  The product page's not-found surface and the user page's not-found surface are not visually parallel — flagged below as an adjacent observation.
- The backend filters products out of `getPortalProductDetails` once they flip to `INACTIVE` with `deactivated_by_system = true` (Day 0 of self-delete; ban time for banned-user owners; never restored on unban while still pending-deletion). So banned-user product pages and pending-deletion-user product pages both return null and render the centered placeholder.
- A pending-deletion user's products are NOT visible during grace (they flip to INACTIVE at Day 0), but the user's profile IS visible with the badge. Asymmetry by design (per spec §4.3): the profile-with-badge is the affordance for reporting; hidden listings are the affordance for "you can't transact with someone who's leaving."

### Admin-side surfaces — out of scope for this topic

Audit-only confirmation that the admin Users list / detail pages and the ban / unban / lock dialogs all exist:

- `AdminBanUserDialog.tsx` — ships with a required `Textarea` (max 500 chars) reason input. **Confirms spec §19.1 is stale** (see below).
- `AdminUnbanUserDialog.tsx`, `AdminLockUserDialog.tsx`, `AdminUnlockUserDialog.tsx`, `AdminUserStateInfoDialog.tsx` — all present.
- `UserStateIndicators.tsx` — list / detail-page state-indicator block.

None of these surfaces appear in the `user-deletion-flow` topic body — they belong to the future owner-dashboard / admin topic queue per the brief's "Out of scope" section.

## Trust-boundary check (per decisions.md §2)

Spec §16 carries the full trust-boundary audit for user-deletion. Confirming each item the brief flagged:

- **Delete-account endpoint identity derivation.** `src/lib/service/reactCalls/userService.ts:118–128`: the frontend POSTs `/secure/user/me/delete` with body `{}` (empty literal). No client-supplied `userId`. The fresh token in the `Authorization` header is the only client signal; the backend derives identity from `SecurityContextHolder` per spec §16. **Clean — consistent with spec §16.**
- **Restoration mechanism is server-side only.** The frontend has no `restoreAccount` call site anywhere in `src/lib/service/`. The trigger is the axios response interceptor reading `X-Account-Restored: true` (`src/lib/config/api.ts:28`) on a normal authenticated request after the user signs in. The backend runs `cancelDeletionOnLogin` inside `FirebaseAuthFilter` and sets the header. **Clean — consistent with spec §16.**
- **`account-banned` and `account-just-deleted` flags are display triggers only.** They are write-only from the client to itself; no value reaches the server. **Clean — consistent with spec §16, with a mechanism deviation (Zustand, not sessionStorage) noted in "For Mastermind."**

## Spec §19.1 reality check — ban-with-reason

**The ban-with-reason input modal is shipped.** `src/components/popups/dialogs/AdminBanUserDialog.tsx` renders a required `Textarea` (`maxNumOfChars = 500`, `required`, validated as `trimmed.length > 0`) labeled with `tDialog('user.ban.dialog.reason.label')` and described with `tDialog('user.ban.dialog.reason.placeholder')`. The Confirm button is disabled until a non-empty trimmed reason is entered. The reason is passed to `banUser(userId, trimmed)` (`src/lib/admin/lib/service/usersService.ts`).

Spec §19.1's "future work" framing is stale. The manual test cases (`features/user-deletion-test-cases.md` Case 4 step 5 — "The ban-with-reason dialog opens. Enter a reason (free text required)") match the live code; `state.md`'s "ban-with-reason flow" listed in the User Deletion shipped feature also matches reality.

This topic body does not surface the admin-side ban-with-reason UI (admin surfaces are out of scope per the brief), so the topic's tester-facing content is unaffected by which side of the contradiction is true. Flagged in "For Mastermind" as drafted spec correction for Docs/QA — Mastermind decides whether to flip §19.1 to "shipped."

## Structural draft of the topic entry

The non-judgment sections — id, type, route, optionsControls, howToUse, whatToExpect — are drafted directly below. Title, overview, non-known-issue pitfalls, qaChecklist, relatedTopics, and images each carry 2–3 proposals further down for Mastermind selection.

### id

`'user-deletion-flow'`

### type

`'flow'`

### route (primary surface only, per decisions.md §5)

`'/[locale]/owner/user — Danger Zone'`

The Danger Zone card at the bottom of the account-settings page is the only entry to self-deletion. The other user-facing surfaces of the user-deletion feature (the ban-notice dialog's triggers, the restoration dialog's trigger, the post-deletion dialog's trigger, the grace-window badges on profile + chat, the 404 surfaces) are enumerated in `optionsControls`.

### optionsControls (structural — drafted directly)

- '"Delete my account" button inside the Danger Zone card at the bottom of /[locale]/owner/user — the only entry to self-deletion. Red outlined button with a four-bullet "what will happen" preamble (listings hidden, profile shows a "Scheduled for deletion" badge for 7 days, messaging is disabled, you are signed out) and an italic restore reminder. Visible to every signed-in user; absent for signed-out visitors (the page is part of the protected /owner/* tree).'
- 'Delete-confirmation dialog — opens when you click "Delete my account". The dialog cannot be dismissed by clicking outside while a request is in flight, and has no X-close button; Cancel is the only escape from the dialog itself. Three provider branches: an email-password password input + "Delete my account" button; a single "Reauthenticate with Google and delete" button (Google popup); and a Facebook-as-dead-code branch (shipped but unreachable today because Facebook sign-in is not exposed anywhere in the portal).'
- 'Post-deletion dialog — opens on the home page right after a successful delete, with the formatted scheduled-deletion date and a "Go to home" acknowledge button. One-shot: closing it and refreshing does not re-open. The mechanism that opens it is internal to the client; the tester just observes that the dialog appears on the home page after the deletion request completes.'
- 'Restoration dialog — opens automatically on the home page right after a pending-deletion user signs back in. Two-line "Welcome back" / "Your account has been restored. The pending deletion has been cancelled." copy, no buttons other than the close X, and an auto-dismiss after about 10 seconds. The dialog is informational only — the restoration committed server-side before the dialog opened. Closing the dialog refreshes the page so the UserInfoDTO badge clears.'
- 'Ban-notice dialog — opens when a banned user attempts to use the platform. Two trigger paths: at sign-in (the sign-in form rejects without showing the raw Firebase error, and the dialog opens on the next page mount) and mid-session (an authenticated request returns 403 with a USER_BANNED code; the dialog opens after the user is signed out and lands on the home page). Same dialog body in both cases: three paragraphs explaining the ban + the appeals path (support@oglasino.com) + the email-driven deletion path + the 12-month re-registration eligibility, plus a "Go to home" button. No reason is shown; no user is identified.'
- '"Scheduled for deletion" badge on the public profile page — secondary surface inside the UserDetails block on /[locale]/user/[userId], rendered between the displayName row and the Rating row. Only appears while the user is in the 7-day grace window; gone once the user restores or once hard-delete runs.'
- '"Scheduled for deletion" badge in the chat header on /[locale]/messages — secondary surface in the conversation header next to the peer\'s name, with a disabled message input and a red italic "This user has scheduled their account for deletion and cannot receive messages" notice below the messages list. "Banned" badge in the same position when the peer is banned (different label, different notice copy: "This user has been banned and cannot receive messages").'
- '"Deleted User" placeholder in chat (and elsewhere) for hard-deleted peers — after Day 7 the chat dropdown menu collapses to just the Delete-chat action (no Profile, no Report, no Block/Unblock), the avatar fallback renders the localized "Deleted User" label, and the message input is disabled. Previous messages remain readable on the counterparty side.'
- 'Profile / product page 404 surfaces for banned users — direct navigation to /[locale]/user/<bannedId> renders the full Not-Found page; direct navigation to /[locale]/product/<bannedId-owned> renders a centered "product not found" line. The two surfaces are not visually parallel (see Pitfalls).'

### howToUse (structural — drafted directly)

- 'Sign in. Open /[locale]/owner/user (Settings → User in the owner sidebar). Scroll to the bottom — past the email / displayName / phone / region row and the five toggle settings — to the red-bordered Danger Zone card. Read the four bullets and the italic restore reminder.'
- 'Click "Delete my account". The confirmation dialog opens. Confirm the body wording matches the bullet list shown on the page underneath (listings hidden, badge for 7 days, messaging disabled, signed out, restore-by-sign-in within 7 days).'
- 'Reauthenticate. For an email-password account, type your current password and click the red "Delete my account" button. For a Google account, click "Reauthenticate with Google and delete" — a Google popup opens; complete sign-in in the popup. (Facebook would behave identically to Google but the sign-in flow is not currently enabled in the portal, so a Facebook-authenticated user is not reachable here in normal testing.)'
- 'After reauth, the dialog disappears and you land back on the portal home page signed out. The post-deletion dialog opens automatically on top of the home page, showing the formatted scheduled-deletion date and a "Go to home" acknowledge button. Click acknowledge to dismiss.'
- 'To observe the grace window from a counterparty\'s view: switch browsers (or use a private window). Sign in as a different user who had a conversation with the deleting user, or who can see one of their products in their favourites. Open /[locale]/messages and the chat with the deleting user, or open /[locale]/user/<id> for them. Confirm the "Scheduled for deletion" badge appears next to their name on both surfaces, the chat input is disabled with the pending-deletion notice, and the profile renders normally with the badge under the displayName.'
- 'To restore during grace: in a fresh browser session, sign in as the deleting user via the regular sign-in flow. Sign-in completes normally (Firebase has not been told the user is disabled — only their refresh tokens were revoked). On any page that fires an authenticated request, the restoration dialog opens automatically with "Welcome back" / "Your account has been restored." copy. Dismiss the dialog or let it auto-dismiss after about 10 seconds.'
- 'To observe the Day-7 hard delete: set the test environment so the scheduled-deletion timestamp is in the past, or run the user.deletion.hard.delete.cron manually. Reload counterparty views: /[locale]/user/<id> now renders the full Not-Found page; /[locale]/product/<bannedId-owned> renders the centered "product not found" placeholder; the chat with the deleted user still exists on the counterparty side but the peer\'s name now renders as the localized "Deleted User" label and the chat dropdown collapses to Delete-chat only.'
- 'To observe a ban-notice via sign-in: in the admin panel ban a test user. In a fresh browser session, attempt to sign in as that user with their correct credentials. The sign-in form rejects without surfacing any raw Firebase error text, and the ban-notice dialog opens on top of the home page. Read the three paragraphs (appeals, email-deletion path, 12-month re-registration eligibility) and click "Go to home" to dismiss.'
- 'To observe a ban-notice mid-session: have the test user signed in and on a protected page (e.g., /[locale]/messages). In a separate admin browser, ban that user. Wait for the test user\'s next authenticated request (any navigation or in-page action will do). The test user is signed out automatically and the ban-notice dialog opens after the navigation lands on the home page.'
- 'To observe the 12-month re-registration block: with the same banned email, attempt to register a new account (or sign in via Google with the same email). The registration appears to start but the backend rejects on /auth/firebase-sync. The dialog that opens is the same ban-notice dialog — there is no separate "this email is banned" message, by design (privacy).'

### whatToExpect (structural — drafted directly)

- 'The Danger Zone is hard to miss but easy to overshoot — it lives at the bottom of /owner/user, past every other input. On long viewports it may not be visible without scrolling.'
- 'The confirmation dialog cannot be closed by clicking outside it once a request is in flight, and has no X-close. Cancel is the only way to back out before reauth completes. Once you confirm the deletion, the dialog stays open during the reauth + backend call; if reauth fails, the dialog stays open and the error line below the password input changes to a "please re-authenticate again" message.'
- 'If you take longer than about 5 minutes between reauthenticating and clicking the confirm button, the backend rejects with a "reauth required" error and the dialog shows that same message. Reauthenticate again and resubmit.'
- 'If you are an admin-locked user (a user_deletion_locks row exists for you), the confirm step fails with a generic "Account deletion is currently restricted. Please contact support@oglasino.com" message and the confirm button is disabled. The dialog does NOT tell you specifically that you are locked — the reason is admin-internal. Cancel is the only escape.'
- 'After a successful delete you land on the home page signed out, with the post-deletion dialog open on top. The scheduled-deletion date is formatted in your locale (with Montenegrin falling back to Serbian formatting). Refreshing the page after dismissing the dialog does NOT re-open it.'
- 'During the grace window you cannot sign in for any purpose other than to restore the account. Signing in cancels the deletion server-side before the restoration dialog opens — even if you close the tab the instant the API call hits, the cancellation has committed and the next sign-in finds an active account.'
- 'Counterparty views of a pending-deletion user: profile page renders normally with the "Scheduled for deletion" badge under the displayName; chat header shows the badge next to the peer\'s name; the message input is disabled and a red italic notice "This user has scheduled their account for deletion and cannot receive messages" appears below the messages list. The user\'s products are gone from listings (filtered out by the backend). Reports against the user remain open and a counterparty can still file a new one.'
- 'After hard delete (Day 7), the same counterparty views show a fundamentally different picture: profile page returns 404 (full Not-Found component), product pages return a centered "product not found" placeholder, the chat header shows "Deleted User" as the peer name with no badge, the chat dropdown collapses to just Delete-chat, and the message input is permanently disabled.'
- 'The ban-notice dialog is the same dialog in every triggering path — sign-in rejection, mid-session 403, re-registration block. The dialog does NOT identify the specific user, does NOT show a reason, and does NOT confirm any specific email is banned. A would-be enumerator who submits random emails would see the same dialog every time, indistinguishable from "this email is in fact banned." The button label reads "Go to home"; clicking it explicitly navigates to /[locale]/.'
- 'The restoration dialog auto-dismisses after about 10 seconds without any user action. The cache-bust + refresh that runs on dismissal is what clears the "Scheduled for deletion" badge from the page underneath — closing the dialog manually triggers the same refresh.'

## Option proposals (Mastermind picks)

### Title — 3 options

- **A. Plain — "User Deletion Flow"** — symmetric with `Start Message Flow` and `Follow Flow`; sticks to the schema slug.
- **B. Tester-friendlier — "Account Deletion + Ban Lifecycle"** — names the wider surface explicitly (self-delete + ban + restore + 12-month block) rather than just "delete." Reads more concrete for the tester scanning chips.
- **C. Action-led — "Deleting and Banning Accounts"** — leads with what the tester is exercising, paired with the `start-message-flow` precedent ("how a user opens a conversation").

### Overview — 3 versions (different emphasis)

**Version 1 — surface-led (emphasises the Danger Zone entry + the multi-surface UI).**

> The lifecycle of an end-user account from "I want to delete my account" through the 7-day grace window to either restoration (by signing back in) or permanent hard delete on Day 7. The entry is the Danger Zone card at the bottom of /[locale]/owner/user — one red-bordered card behind a "Delete my account" button. The user-facing UI of the rest of the lifecycle lives in four dialogs (delete-confirmation, post-deletion, restoration, ban-notice) and three persistent badges / placeholders (the "Scheduled for deletion" badge on the profile and chat header during grace, the "Banned" badge in chat for banned counterparties, and the "Deleted User" placeholder after hard delete). Banned users — the parallel admin-driven path — see only the ban-notice dialog and cannot use the in-app delete; the right-to-erasure path for them is email to support@oglasino.com.

**Version 2 — timeline-led (emphasises what happens at each phase).**

> How an account exits the platform, end to end. A signed-in user clicks "Delete my account" in the Danger Zone, reauthenticates inside a confirmation dialog (password for email-password accounts; popup for Google), and is signed out. For the next 7 days the user is gone from messaging and their listings are hidden, but their profile remains publicly visible with a "Scheduled for deletion" badge — the affordance for other users to report misconduct against them in the same window. The user can cancel at any point during the 7 days by signing back in: the cancellation happens server-side, and a restoration dialog opens on the next page with a "Welcome back" message. On Day 7 the user is permanently deleted: profile returns 404, listings are gone, and the user\'s name in any remaining chat history renders as "Deleted User." A separate admin-driven path (ban) signs the user out the same way but routes them through the ban-notice dialog instead; banned users cannot self-delete and must email support@oglasino.com for the right-to-erasure path. A banned email is blocked from re-registration for 12 months.

**Version 3 — composition-led (emphasises the states the tester needs to keep straight).**

> The user-deletion lifecycle as the tester observes it across three states: ACTIVE, PENDING_DELETION (7-day grace), and gone (hard-deleted). A fourth state — BANNED — runs in parallel and can compose with PENDING_DELETION; when both apply, "Banned" wins everywhere in the UI. Each state has its own set of badges, dialogs, and 404 surfaces: PENDING_DELETION shows the "Scheduled for deletion" badge on the profile and the chat header with a disabled input; BANNED shows the "Banned" badge in chat (the profile returns 404) and the ban-notice dialog on any sign-in attempt; hard-deleted users render as "Deleted User" with a 404 profile and gone listings. The entry to all of this — for an end user — is the Danger Zone card at the bottom of /[locale]/owner/user. Banned users cannot use the entry; their right-to-erasure path is email to support@oglasino.com.

### Pitfalls — 3 candidate cuts of non-known-issue pitfalls

The brief asks for variant emphasis. The one Known-issue pitfall draft (from open `issues.md` entries) appears below the three cuts.

**Cut A — grace-window state composition traps.**

- 'During the 7-day grace window the user\'s profile remains publicly visible with the "Scheduled for deletion" badge. Their listings are gone. A tester scanning a single surface — say, the profile page — might conclude "the user is fine, just has an unusual badge" and miss that their entire product set has been removed. The badge is the only signal on the profile that the user is mid-deletion; the absence of products has to be cross-checked separately.'
- 'A user can be both pending-deletion AND banned at the same time. When both apply, the counterparty-side UI shows the "Banned" badge in chat and the profile page returns 404 — the "Scheduled for deletion" badge is hidden in this composition. The product-page 404 is also from the ban side, not the pending-deletion side. A tester restoring a banned-and-pending user (i.e., admin unbans while the user is still in grace) will find that the user\'s products do NOT automatically restore to ACTIVE — the pending-deletion intent is preserved and unban only restores products when the user is in deletionStatus = ACTIVE at the time of unban. Easy to read as "unban is broken" when the design is "unban respects pending-deletion."'
- 'The deactivated_by_system flag is load-bearing. Products the user themselves deactivated before pending-deletion stay user-deactivated through grace, hard delete, ban, and unban; only system-deactivated products (the ones flipped at Day 0 or at ban time) restore. A tester verifying restoration must inspect each product\'s prior state to know which should come back.'
- 'The "Scheduled for deletion" badge is rendered from UserInfoDTO.state, which is driven by a backend → web cache revalidation call after state transitions. If that revalidation does not arrive — e.g., the production WEB_REVALIDATE_URL is misrouted (see Known issue) — counterparty views of the badge persist or are absent up to 5 minutes after the actual state change. Hard-refreshing always picks up the fresh state.'

**Cut B — dialog-trigger-timing traps.**

- 'The post-deletion, restoration, and ban-notice dialogs are all triggered by client-side store flags (not URL routes and not sessionStorage). They open on the home page after the delete / sign-in / 403 has completed and the user has been navigated to /[locale]/. A tester expecting a route-level redirect or a separate URL for each dialog will not find one; the dialogs all open on top of the home page.'
- 'The restoration dialog auto-dismisses after about 10 seconds. A tester who steps away mid-test may miss it. The dialog\'s only confirmation that restoration happened is the dialog itself — after dismissal, the only on-page signal that the account is active again is the absence of the "Scheduled for deletion" badge on the user\'s profile.'
- 'The ban-notice dialog can fire in three different timings — at sign-in (synchronously after the sign-in form), mid-session (after the next authenticated request from any tab), and on re-registration with a banned email. All three open the same dialog with the same body. A tester verifying the mid-session case must not rely on staying idle: the trigger is the next request, so a fully-idle tab may not see the dialog open until they click something.'
- 'The delete-confirmation dialog stays open during the reauth + backend call. Reauth failure (wrong password, popup cancelled, network drop on the reauth call) keeps the dialog open with a "please re-authenticate again" message — it does NOT close on error. Easy to read as "the dialog froze" when the design is "we kept it open so you can retry."'
- 'The reauth-freshness check at the backend is 5 minutes from auth_time. A tester who reauthenticates, then steps away for a phone call, then clicks the confirm button will see a "please re-authenticate again" rejection even though the dialog never closed. The dialog re-prompts but does not auto-restart reauth; the user must initiate.'

**Cut C — provider-branch and edge-case traps.**

- 'The delete-confirmation dialog\'s Facebook branch is shipped but unreachable. resolveProvider() handles facebook.com, but Facebook sign-in is not currently exposed in any sign-in surface in the portal. A tester wanting to exercise the Facebook reauth path cannot get there through normal sign-in; the branch can only be reached by manually mocking a Firebase user with providerId === "facebook.com". Worth a single check that the branch renders identically to Google when forced, but cannot be exercised end-to-end.'
- 'The unsupported provider branch (any providerId that is not password / google.com / facebook.com) renders only the Cancel button — both the dialog body and the confirm button are hidden. Reachable today only if a future Firebase provider is wired without updating resolveProvider; not a tester-facing failure mode in normal flows.'
- 'A banned user attempting sign-in with a Google account whose underlying Firebase Auth record has setDisabled = true will see the same ban-notice dialog as an email-password sign-in. There is no Google-specific message. Firebase rejects the credentials with auth/user-disabled, the sign-in form suppresses the raw error, and the dialog opens on the next page mount.'
- 'A banned user signed in on Browser A — when Browser B is open with the same user signed in and an open session, the ban does not propagate immediately to Browser B. The next authenticated request from Browser B triggers either the mid-session 403 path (refresh token still mints) or the cross-browser auth/user-disabled path (refresh token has been invalidated and Browser B\'s token mint throws). In practice the latency is bounded by the next request; an idle Browser B can stay signed in until it next sends one.'
- 'A user whose Firebase account has been deleted directly (e.g., by the user via their Google Account settings) during the 7-day grace window will be caught by the weekly Firebase reconciliation cron and hard-deleted immediately with triggered_by = "firebase_cascade". If they had unfinished reports against them (or more than 3 they had authored), an auto-ban hash is inserted first, so their email is blocked from re-registration for 12 months even though they were never explicitly banned. The user-facing observation is identical to a normal Day-7 hard delete; the audit trail is what differs.'

### Known-issue pitfall draft (cross-references issues.md per decisions.md §7)

- 'Known issue: in production, the backend\'s cache-invalidation call to /api/revalidate may not reach the web layer if WEB_REVALIDATE_URL points at the oglasino.com domain — the Cloudflare router worker routes /api/* to the backend, so the backend POSTs to itself, returns 404, and the SSR cache is never invalidated. Mitigation in flight: Igor will point WEB_REVALIDATE_URL directly at the Vercel deployment URL to bypass the worker. Tracked in issues.md (2026-05-19 entry — "Backend → web cache revalidation may not work in production via oglasino.com domain"); until verified working in production, if a tester observes a stale "Scheduled for deletion" badge, a stale 404, or a stale "Banned" badge persisting past the expected window after a counterparty-side state transition, suspect the cache-revalidation path before filing it as a UI defect. The cache TTL is 5 minutes — hard refresh always picks up the fresh state.'

### qaChecklist — 3 candidates (minimum-viable / standard / exhaustive)

**Candidate A — minimum viable (~10 items, golden paths + one of each composition).**

- 'Sign in as an email-password user. Open /[locale]/owner/user, scroll to the Danger Zone, click "Delete my account". Verify the confirmation dialog opens with the password input, Cancel + "Delete my account" buttons, no X-close.'
- 'Type the password and click "Delete my account". Verify the post-deletion dialog opens on the home page with the formatted scheduled-deletion date and an acknowledge button. Verify the user is signed out (header / sidebar reflect signed-out state).'
- 'Sign in as a Google user. Open the Danger Zone and click "Delete my account". Verify the confirmation dialog opens with no password input and a single "Reauthenticate with Google and delete" button. Click it; complete the Google popup. Verify the post-deletion dialog opens on the home page.'
- 'In a fresh browser session, sign in as a pending-deletion user. Verify the restoration dialog opens automatically with "Welcome back" / "Your account has been restored." copy and auto-dismisses after about 10 seconds.'
- 'After restoration, verify the user\'s profile page no longer shows the "Scheduled for deletion" badge and that the user\'s previously-hidden products are back in listings.'
- 'Trigger a Day-7 hard delete (run the cron manually or fast-forward scheduled_deletion_at). Verify /[locale]/user/<id> returns the full Not-Found page; verify the chat with the deleted user shows "Deleted User" as the peer name with the chat dropdown collapsed to Delete-chat only; verify the previously-active product pages render the centered "product not found" placeholder.'
- 'In the admin panel ban a test user. In a fresh browser session attempt to sign in as that user; verify the ban-notice dialog opens with the three-paragraph body (appeals, email-deletion path, 12-month re-registration) and a "Go to home" button.'
- 'Sign in as the same test user in one browser. In a separate admin browser ban them. Wait for the test user\'s next authenticated request (navigate to /messages or click an action); verify the user is signed out and the ban-notice dialog opens.'
- 'Within the same 12-month window, attempt to register a new account with the same email; verify the ban-notice dialog opens (no separate "this email is banned" message).'
- 'Put a user in pending-deletion, then ban them. Verify the chat header shows "Banned" (NOT "Scheduled for deletion") and the chat notice text reads "This user has been banned and cannot receive messages." Verify the profile page returns 404.'

**Candidate B — standard (~16 items, A plus reauth-freshness + admin-lock + counterparty-grace + Firebase-cascade observations).**

A1–A10 from Candidate A, plus:

- 'In the delete-confirmation dialog, reauthenticate, then wait at least 5 minutes before clicking the confirm button. Verify the backend rejects with the "please re-authenticate again" message and the dialog stays open. Reauthenticate again and verify the resubmission succeeds.'
- 'Admin-lock a test user (via SQL or the lock endpoint). As that user, attempt deletion. Verify the dialog shows the generic "Account deletion is currently restricted. Please contact support@oglasino.com" message and the confirm button is disabled.'
- 'Sign in as User A. Have User A start a chat with User B. Put User B in pending-deletion. Sign in as User A in a fresh browser. Open /[locale]/messages and the chat with User B. Verify the chat header shows the "Scheduled for deletion" badge next to User B\'s name, the message input is disabled, and the red italic "This user has scheduled their account for deletion and cannot receive messages." notice appears below the messages list. Verify User B\'s message history is still readable with User B\'s actual displayName (NOT "Deleted User").'
- 'Repeat the previous check on /[locale]/user/<User B\'s id> — verify the "Scheduled for deletion" badge appears between the displayName and the Rating row, the profile renders normally, and User B\'s products have disappeared from the product grid on that profile.'
- 'Admin-unban a previously-banned user who is also in pending-deletion. Verify the "Banned" badge / 404 surfaces drop, the "Scheduled for deletion" badge returns on the profile and chat header, but the user\'s products do NOT automatically restore to ACTIVE (they stay INACTIVE with deactivated_by_system = true because the user is still in pending-deletion).'
- 'Delete the Firebase Auth user record of a pending-deletion user directly (via Firebase Admin Console). Trigger the weekly Firebase reconciliation cron manually. Verify the user is hard-deleted immediately (counterparty views show "Deleted User" / 404) and that the audit log records triggered_by = "firebase_cascade".'

**Candidate C — exhaustive (~24 items, B plus provider-branch + cross-browser + re-registration variants).**

B1–B16 from Candidate B, plus:

- 'In the delete-confirmation dialog with an email-password user, type a wrong password and click "Delete my account". Verify the dialog stays open and the error line below the password input reads the generic "please re-authenticate again" message (NOT the raw Firebase code).'
- 'In the delete-confirmation dialog with a Google user, dismiss the Google popup without completing reauth. Verify the dialog stays open with the same generic error message and the confirm button re-enables.'
- 'Verify the delete-confirmation dialog cannot be dismissed by clicking outside it while the request is in flight (the loading spinner is visible). Verify it can be dismissed by clicking Cancel before submitting.'
- 'After a successful delete, dismiss the post-deletion dialog and refresh the home page. Verify the dialog does NOT re-open. Confirm the scheduled-deletion date in the dismissed dialog was formatted in the active locale (Montenegrin testers see Serbian formatting per the cnr → sr fallback).'
- 'Sign in as a user on two browsers (A and B). Admin bans the user. On Browser A, click anywhere that triggers an authenticated request; verify the ban-notice dialog opens via the mid-session 403 path. On Browser B, click anywhere; verify the ban-notice dialog also opens — Browser B may take a different code path (cross-browser auth/user-disabled at token mint) but the user-facing dialog is the same.'
- 'In an admin browser, ban a user; in a separate browser, attempt Google sign-in with that user\'s Google email. Verify the ban-notice dialog opens (same dialog body as email-password sign-in), and verify the just-created Firebase user has been cleaned up by the backend (no orphan in Firebase).'
- 'Verify clicking "Go to home" in the ban-notice dialog navigates to /[locale]/ explicitly (the default INFO_DIALOG close just calls onClose; the custom onClose here triggers the navigation).'
- 'Verify the "Banned" badge in chat shows the localized banned label (Common namespace), and the pending-deletion notice text uses the MESSAGES_PAGE namespace. Both notices stack vertically if a peer is simultaneously blocked and pending-deletion (composition: block + pending-deletion or block + banned).'

### relatedTopics — 3 candidates

**Set A — narrowest (counterparty-facing surfaces).**

- `{ topicId: 'messages-page' }` — the chat header badge + disabled input + notices live here.
- `{ topicId: 'user-page' }` — the profile badge + 404 surface lives here.

**Set B — narrow plus product surface.**

- `{ topicId: 'messages-page' }`
- `{ topicId: 'user-page' }`
- `{ topicId: 'product-page' }` — banned / deleted user\'s products return the centered "product not found" placeholder; the topic covers product-side surfaces a counterparty would visit.

**Set C — full cross-reference (every adjacent topic the lifecycle touches).**

- `{ topicId: 'messages-page' }`
- `{ topicId: 'user-page' }`
- `{ topicId: 'product-page' }`
- `{ topicId: 'product-creation-flow' }` — once Stop 2 of that session lands; deleted / banned users cannot create. (Cross-reference will resolve when the parallel topic is appended.)
- `{ topicId: 'follow-flow' }` — following a pending-deletion or banned user; the icon should still flip but the followed user is on their way out.

### images — 3 candidates (variant coverage)

Note on convention: each entry has the `// <!-- Screenshot: ... -->` HTML-comment block + `name` + `description` shape with `imageKey` unset. Igor supplies files after the topic ships.

**Set A — four slots, one per primary surface.**

1. `user-deletion-flow-danger-zone.png` — /[locale]/owner/user scrolled to the bottom showing the red-bordered Danger Zone card with the four bullets, italic restore note, and red "Delete my account" button.
2. `user-deletion-flow-confirmation-dialog-password.png` — the delete-confirmation dialog open with the email-password branch (password input visible, Cancel + "Delete my account" buttons, no X).
3. `user-deletion-flow-grace-profile-with-badge.png` — /[locale]/user/<id> for a pending-deletion user, showing the "Scheduled for deletion" badge between the displayName and the Rating row, and the empty product grid on the right.
4. `user-deletion-flow-ban-notice-dialog.png` — the ban-notice dialog open on the home page with the three-paragraph body and "Go to home" button.

**Set B — five slots (A plus grace chat-header view).**

A1–A4, plus:

5. `user-deletion-flow-grace-chat-header.png` — /[locale]/messages with a chat against a pending-deletion peer: chat header shows the displayName with the "Scheduled for deletion" badge next to it, the message input is disabled, and the red italic pending-deletion notice is visible below the messages list.

**Set C — five slots (A plus Day-7 hard-delete view).**

A1–A4, plus:

5. `user-deletion-flow-hard-delete-chat.png` — /[locale]/messages with a chat against a hard-deleted peer: chat header shows "Deleted User" as the peer name with no badge, the chat dropdown is open showing only the Delete-chat action, the message input is disabled, and the avatar fallback renders the localized "Deleted User" label.

## Files touched

None. Stop 1 produces no on-disk code changes.

## Tests

Not run. Stop 1 does no code work; `npx tsc --noEmit` runs at Stop 2 after the `topics.ts` append.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

(Adjacent observations in "For Mastermind" are flagged for triage but are not drafted as new entries. The §19.1 spec correction is drafted below as a Docs/QA candidate — target: `features/user-deletion.md` §19.1.)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed (no code touched).
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): three flagged in "For Mastermind".
- Part 6 (translations): confirmed (no translation keys added; the namespace inventory is observational only).
- Other parts touched: Part 5 (session-summary template) — applied for filename + sections + closure gate. Part 11 (trust boundaries) — applied for the §16 confirmation check.

## Known gaps / TODOs

- None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — Stop 1 produces a session summary only.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

### Trust-boundary check — short conclusion

Consistent with spec §16. One mechanism deviation noted (Zustand store flags, not sessionStorage) — same trust posture, different implementation. Detail below under "Audit-vs-spec / -summary observations."

### Spec §19.1 reality check — finding + drafted correction

**Reality: AdminBanUserDialog ships with a required free-text reason input (`Textarea`, max 500 chars, `required`, `trimmed.length > 0` validation). The ban-with-reason UI is live, not future work.**

**Drafted spec correction for Docs/QA — target: `oglasino-docs/features/user-deletion.md` §19.1.** The current text reads (paraphrased from the Docs/QA summary's reference): "the existing admin button does not yet pass a reason. The button keeps working with a default placeholder; a future web brief adds a reason-input modal." Suggested replacement: a single short paragraph noting that the admin ban-with-reason input modal shipped during the User Deletion feature, with file pointer `oglasino-web/src/components/popups/dialogs/AdminBanUserDialog.tsx` and the validation contract (`required`, ≤ 500 chars). Mastermind drafts final wording; Docs/QA applies.

### Audit-vs-spec / -summary observations (mechanism deviations)

1. **`account-banned` and `account-just-deleted` are Zustand store flags, NOT sessionStorage flags.** Spec §14 and the Docs/QA summary describe `sessionStorage.setItem('account-just-deleted', scheduledDeletionAt)` and `sessionStorage.setItem('account-banned', '1')`; the live code uses `useAuthStore.setAccountJustDeleted(...)` and `useAuthStore.setAccountBanned(...)` consumed reactively by `AccountStateDialogsInit`. The inline comment at `DeleteAccountConfirmationDialog.tsx:108–110` and `AccountStateDialogsInit.tsx:71–78` explicitly state the rationale: "the locale layout does not remount on `router.replace` within the same `[locale]` segment, so a mount-only sessionStorage read never re-fires." Same trust posture (write-only, browser-local, display trigger), different mechanism. The user-deletion-flow topic body describes the observation only (the dialog appears on the home page) without committing to either mechanism, so the topic stays correct regardless of which the spec lands on. Severity: low (the spec is the older description; the code is the current reality). Suggested action: Docs/QA brief to update spec §14.4 / §14.10–§14.12 to reflect the store-based mechanism.

2. **Ban-notice dialog keys live in the `DIALOG` namespace, not a dedicated `BANNED_DIALOG` namespace.** The Docs/QA summary lists `BANNED_DIALOG (new namespace per spec §15.7)` as one of seven namespaces; the live code reads `tDialog('banned.dialog.title')` etc., placing all four banned-dialog keys (`banned.dialog.title`, `banned.dialog.body.first`, `banned.dialog.body.delete.intro`, `banned.dialog.body.duration`) in the `DIALOG` namespace. The `BANNED_DIALOG` namespace does not exist in `TranslationNamespaceEnum`. Severity: low (purely a documentation drift; tester observation is identical). Suggested action: same Docs/QA brief, fix §15 namespace list.

3. **Product-page null-result UI is not a full Not-Found render.** Spec §4.3 / §4.5 and the test cases describe banned-user / hard-deleted-user product pages as "404." The live code renders a centered single-line "product not found" message in a flex container at 40vh height (`product/[productId]/[productName]/page.tsx:77–82`); it does NOT use the `<NotFound title=... description=... />` component that the user page uses. The HTTP status is not explicitly set in either case (both rely on the `notFound()` Next.js helper or default 200 response). The user-deletion-flow topic Cut A pitfall draft notes the asymmetry; the topic body otherwise describes both as 404 surfaces because the tester observation is "product is unreachable." Severity: low (cosmetic asymmetry between two not-found surfaces). Not flagged for spec correction — the spec's "404" is correct as a description of intent.

### Adjacent observations (conventions Part 4b)

1. **Product-page null-result UI inconsistency with user-page (low).** `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:77–82` renders a centered single-line placeholder, while `app/[locale]/(portal)/(public)/user/[userId]/page.tsx:64–70` renders the full `<NotFound title=... description=... />` component. Two visually-distinct not-found surfaces for symmetric concepts (banned/deleted owner). File paths above. I did not fix this because it is out of scope.

2. **`'unsupported'` provider branch in `DeleteAccountConfirmationDialog` shows the system-error string in `dialogDescription` (low).** `DeleteAccountConfirmationDialog.tsx:156` falls back to `tErrors('system.error')` when `provider === 'unsupported'`. The user gets only a Cancel button and a system-error description; no explanation of what happened. Reachable only if a new Firebase provider is added without updating `resolveProvider`; defensible posture today, but a hard-to-action failure for the tester to interpret if it ever fires. File: `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx:156`. I did not fix this because it is out of scope and the branch is not reachable in normal sign-in flows.

3. **Facebook reauth branch is shipped but unreachable through any sign-in surface (low/cosmetic).** `DeleteAccountConfirmationDialog.tsx:78–79` handles `'facebook.com'` correctly, but the `facebookProvider` is not exposed in any portal sign-in surface I can find (`LoginOptionsDialog.tsx`, etc. — search did not surface a Facebook button reachable from default sign-in). The branch exists for future-enablement; today a Facebook-authenticated user is not reachable. File: `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx:78–79`. I did not fix this because the brief explicitly described the Facebook branch as dead code, so this is consistent with the documented intent; flagging only for awareness.

### Brief vs reality

No structural mismatch between the brief and the code worth stopping the session for. The mechanism deviations (sessionStorage vs Zustand, `BANNED_DIALOG` vs `DIALOG`) are observations recorded for spec routing, not blockers to drafting the topic. The brief explicitly said "Confirm the actual surface" — I have done so, and the topic body is written to match the implementation rather than the description. The spec §19.1 contradiction is resolved (ban-with-reason input modal is shipped); drafted correction sits above for Mastermind.

### `<n>` counting

Listed `oglasino-web/.agent/` for `*-qa-preparation-*.md`. One match: `2026-05-23-oglasino-web-qa-preparation-1.md` (the product-creation-flow Stop-1 summary; despite the slug naming, it sits in the qa-preparation slug per the brief's framing). Following conventions Part 5: `<n> = 1 + 1 = 2`. This session's named file is `2026-05-23-oglasino-web-qa-preparation-2.md`.

### Stop 2 — what's owed

After Mastermind selects from the option proposals above:

- Title — one of A / B / C (or other).
- Overview — one of V1 / V2 / V3 (or rewrite).
- Pitfalls — one of Cut A / B / C (or a custom merge) plus the standing Known-issue pitfall draft for the 2026-05-19 cache-revalidation issue, plus a decision on whether to include the §19.1 spec correction signal in the topic body or leave it as a "For Mastermind" item only (leave-out is my recommendation — the topic body is tester-facing only).
- qaChecklist — one of A / B / C (or a custom merge).
- relatedTopics — one of Set A / B / C.
- images — one of Set A / B / C.
- shortLabel — not pre-drafted; my suggestion is `'Delete Account'` (parallel with `'Start Message'` / `'Follow'`), but Mastermind can pick.

Stop 2 folds the decisions in, appends the final topic entry to `qaTopics` in `app/[locale]/design/topics.ts`, runs `npx tsc --noEmit` from `oglasino-web`, and closes the session.
