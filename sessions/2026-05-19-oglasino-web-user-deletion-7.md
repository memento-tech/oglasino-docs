# Session summary

**Repo:** oglasino-web
**Branch:** feature/user-deletion
**Date:** 2026-05-19
**Task:** Web read-only audit — Facebook sign-in provider coverage. Verdict (A/B/C) for Task 8 scoping (catch `auth/user-disabled` in sign-in path and open `useAuthStore.accountBanned`).

## Implemented

- Read-only audit. No implementation.

## Files touched

- None (audit only).

## Tests

- None run (read-only).

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no production code touched.
- Part 4a (simplicity) / Part 4b (adjacent observations): one adjacent observation flagged in "For Mastermind" (the dead Facebook sign-in button in `LoginOptionsDialog`).
- Part 6 (translations): N/A this session.
- Other parts touched: none.

## Known gaps / TODOs

- None.

## For Mastermind

### Verdict — **C, mixed state**

Facebook is partially present in a way that doesn't fit either A or B cleanly. The non-UI half of the Facebook integration is live and complete; the UI sign-in trigger is dead. Per the brief's strict definition of "wired" ("there exists a code path that calls a Firebase Facebook sign-in method that would execute if the user clicked the right button"), Facebook sign-in is **not wired today** — there is no reachable button. But declaring Verdict B would understate how much Facebook-aware code is live elsewhere (the reauth branch in the delete-account dialog, the helper in `authService`, the store action in `useAuthStore`).

#### Evidence — what is live

| Layer | File:line | Live? |
| --- | --- | --- |
| Provider object | `src/lib/config/firebaseClient.ts:43-45` — `new FacebookAuthProvider()`, `setCustomParameters({ prompt: 'select_account' })`, `addScope('email')` | Yes |
| Sign-in helper | `src/lib/service/reactCalls/authService.ts:192-195` — `loginWithFacebookFirebase` → `signInWithPopup(auth, facebookProvider)` → `buildUserSession` | Yes (no live caller from a UI surface) |
| Store action | `src/lib/store/useAuthStore.ts:234-251` — `loginWithFacebook`, with proper `mapAuthError` wrapping and `initPushForAuthenticatedUser` follow-up | Yes (no live caller from a UI surface) |
| Reauth branch (delete-account flow) | `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx:23,28,77-78` — `Provider` type includes `'facebook.com'`; `resolveProvider()` returns it; `handleDelete` calls `reauthenticateWithPopup(currentUser, facebookProvider)` | Yes — reachable for any user whose `auth.currentUser?.providerData[0]?.providerId === 'facebook.com'` |
| Provider resolution (delete-account) | `DeleteAccountConfirmationDialog.tsx:26` reads `auth.currentUser?.providerData[0]?.providerId`, not `AuthUserDTO.providerId` | Yes |

#### Evidence — what is dead

- **The only UI trigger that would invoke `loginWithFacebook` is commented out.** `src/components/popups/dialogs/LoginOptionsDialog.tsx:65-71`:

  ```tsx
  {/* TODO Facebook login is not currently available, needs to be set up correctly */}
  {/* <button
    onClick={loginWithFacebook}
    className="border-border-mild flex w-[80%] cursor-pointer ...">
    <FacebookIcon height={20} />
    {tDialog('login.options.login.facebook.label')}
  </button> */}
  ```

  The `loginWithFacebook` destructuring is also absent from the live `useAuthStore()` call at `LoginOptionsDialog.tsx:31` (only `loginWithGoogle` is pulled). Consistent with the button being commented out.

- The grep hits for `loginWithFacebook` outside the store/helper are all inside this JSX comment block — there is no live caller.

- `LogInDialog.tsx` and `RegisterDialog.tsx` use email/password only (`login` / `register` actions). Neither references any social provider.

#### Provider-resolution coverage across the auth lifecycle

- `useAuthStore.initAuthListener` and `UseTokenRefresh` do not branch on provider — they treat the Firebase user uniformly.
- The settings page (`app/[locale]/owner/user/page.tsx:218-219`) reads `user.providerId` (from `AuthUserDTO`) to disable the email field for OAuth users — generic, not Facebook-specific.
- The only place provider-specific code lives is `DeleteAccountConfirmationDialog`. Its `Provider` discriminated union explicitly enumerates `'password' | 'google.com' | 'facebook.com' | 'unsupported'`. Facebook is treated as a possible existing provider here — consistent with the assumption that some users may carry `providerData[0].providerId === 'facebook.com'` (legacy sign-ups from before the button was commented out, or future re-enablement).

#### Recommendation for Task 8 scoping

The Task 8 catch is on the sign-in path. Mapped to the three providers:

1. **Email/password** — live UI (`LogInDialog` → `useAuthStore.login` → `loginUserFirebase` → `signInWithEmailAndPassword`). Catch needed. Currently `mapAuthError` (`useAuthStore.ts:11-46`) does **not** handle `auth/user-disabled` — falls through to the generic `(err as { message?: string })?.message ?? 'Login failed. Please try again.'`. Confirmed.
2. **Google** — live UI (`LoginOptionsDialog` → `useAuthStore.loginWithGoogle` → `loginWithGoogleFirebase` → `signInWithPopup(googleProvider)`). Catch needed.
3. **Facebook** — UI trigger commented out. Adding the catch to `loginWithFacebookFirebase` / `loginWithFacebook` today is a **no-op at runtime** because there is no live execution path.

Two viable Task 8 shapes:

- **Tight:** Task 8 covers email/password + Google only. Facebook stays as a backlog entry; when the button is uncommented in a future session, that session adds the catch.
- **Symmetric:** Task 8 lands the catch in `mapAuthError` (or wherever the central handler ends up), which automatically covers all three store actions since they all funnel through `mapAuthError` in their `catch` block. Cost ≈ same as the tight shape because the handler is one map entry. No risk: the dead Facebook helper stays dead, but the day someone uncomments the button the behaviour is already correct.

If the Task 8 implementation puts the catch inside `mapAuthError` (likely the natural spot — that's where every other `auth/*` code is mapped), the symmetric shape is essentially free. If Task 8 instead puts the catch at the action-level (separate `try` block per `login` / `loginWithGoogle`), the tight shape is the lower-risk pick and Facebook becomes a one-line backlog.

#### Out-of-scope-for-Task-8 observations (Part 4b)

Surfaced during the grep, flagged here, not fixed:

1. **`LoginOptionsDialog` carries dead UI** — `LoginOptionsDialog.tsx:65-71`. The TODO is the kind of thing that quietly rots: either Facebook gets re-enabled and the comments come off, or the feature is parked and the comments are deleted with the import / icon. The matching translation key `login.options.login.facebook.label` may or may not still be seeded. Severity: low. Out of scope for Task 8.
2. **`bodyKey` in `DeleteAccountConfirmationDialog`** — `DeleteAccountConfirmationDialog.tsx:144-147`. The dialog has live Facebook reauth (`provider === 'facebook.com'` → popup), but the body-text key collapses all OAuth providers to `'delete.account.dialog.body.google'`. A user whose existing provider is `facebook.com` sees "Google" prose. Severity: low (cosmetic, only affects existing Facebook-signed-in users — likely none in practice today given the sign-in button is dead). Out of scope for Task 8.
3. **`accountBanned` plumbing is already in place** — `useAuthStore.ts:86-89` defines `accountBanned: { reason: string | null } | null` with a comment that already names the three trigger sources: sync-time disabled flag, sync-time `EMAIL_BANNED`/`USER_BANNED`, and mid-session 403 `USER_BANNED`. Task 8 adds the **fourth** trigger source — `auth/user-disabled` on the sign-in path. Worth confirming the existing dialog consumer (`AccountStateDialogsInit`) treats the `reason: null` case acceptably, since the `auth/user-disabled` code carries no reason payload from Firebase. Not an audit finding per se — just a flag that the new trigger source produces `reason: null` and the existing dialog needs to render that gracefully. Out of scope for Task 8 only if `AccountStateDialogsInit` is already null-tolerant; if not, the Task 8 brief should cover it.

---

**Summary in one line.** Verdict C. The dead Facebook button means Task 8 cannot meaningfully attach the catch to a live UI sign-in path for Facebook; the cheapest correct answer is to land the catch in the shared `mapAuthError` so all three store actions inherit it (free symmetry, zero runtime change for Facebook today, correct on the day the button is uncommented). If Mastermind prefers tight scoping, email/password + Google now, Facebook as a one-line backlog when the button is re-enabled.
