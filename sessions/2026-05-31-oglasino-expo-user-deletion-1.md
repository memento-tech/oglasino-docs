# Session summary — oglasino-expo — user-deletion — 1

**Date:** 2026-05-31
**Repo:** oglasino-expo
**Slug:** user-deletion
**Order:** 1 (first session for this slug; no prior `*-user-deletion-*.md` in `.agent/`)
**Branch:** `new-expo-dev` (HEAD `b67627c`)
**Type:** read-only current-state audit. No code changed, no edits staged, no commits.

## Task (one sentence)

Capture the *current state* of `oglasino-expo` against the shipped user-deletion contract (Q1–Q12) so the mobile-adoption implementation briefs are written against the real file layout, flagging every NOT-PRESENT / stub / no-op.

## What I did

Read the auth/dialog/service/chat/navigation surfaces the user-deletion feature touches and answered all twelve audit questions with file:line citations. Wrote the deliverable to `.agent/audit-user-deletion-current-state.md`.

Files examined (read-only): `src/lib/store/authStore.ts`, `src/components/init/AccountStateDialogsInit.tsx`, `src/components/init/AppInit.tsx`, `src/components/init/ForegroundRevalidationInit.tsx`, `src/components/dialog/store/useDialogStore.ts`, `src/components/dialog/dialogRegistry.ts`, `src/components/dialog/DialogManager.tsx`, `src/components/dialog/dialogs/InfoDialog.tsx` (+ ReportDialog shape), `src/lib/utils/parseServiceError.ts`, `src/lib/utils/isErrorWithCode.ts`, `src/lib/services/userService.ts`, `src/lib/services/authService.ts`, `src/lib/client/firebaseClient.ts`, `src/lib/config/api.ts`, `src/lib/init/authInterceptors.ts`, `src/lib/store/bootStore.ts` (namespace/fetch loop), `src/i18n/types.ts`, `src/lib/types/user/AuthUserDTO.ts`, `src/lib/types/user/UserInfoDTO.ts`, `app/owner/dashboard/user.tsx`, `app/owner/_layout.tsx`, `app/(portal)/(secured)/*.tsx`, `src/components/messages/{Chats,Messages,utils}.tsx`, `src/components/dialog/dialogs/ChatUserFunctionsDialog.tsx`, `src/components/user/UserMenu.tsx`.

## Key findings (headlines; full detail in the deliverable)

- **`authStore`:** `restored`/`accountBanned` present as plain **booleans** (web uses `accountBanned: {reason}|null` — shape delta). `accountJustDeleted`/`deletionInFlight` NOT PRESENT.
- **`AccountStateDialogsInit`:** `accountJustDeleted` is a dead local `const … = null` with an empty effect; the 3 handoff steps to activate it are unchanged. `restored`/`accountBanned` slots fully wired with open-then-clear.
- **Dialogs:** `InfoDialog` has `closeButtonLabel?` but **no auto-dismiss** and **no input**; a reauth/confirm-with-input dialog is a new bespoke component (ReportDialog/LoginDialog are the precedents).
- **Service layer:** `parseServiceError`/surface-via-throw house style present; **no `deleteCurrentUser`**. ⚠️ The request interceptor **unconditionally overwrites `Authorization`** — web's explicit per-call fresh-token header does NOT transfer verbatim (mitigated by Firebase's cached-token behavior on the same `currentUser`).
- **Listener/interceptor:** `auth.signOut()` confirmed; 403 `USER_BANNED`/`EMAIL_BANNED` + `X-Account-Restored` paths confirmed and store-flag-wired. **No `deletionInFlight` skip** of `firebase-sync` (C-6).
- **Reauth:** JS SDK (`firebase/auth`) — `reauthenticateWithCredential`/`EmailAuthProvider` available, no existing usage; Google via `@react-native-google-signin`; `providerData[0].providerId` available.
- **Chat/DTOs:** no grace-period badge or send-gate; only hard-delete `isDeletedPeer`. `UserInfoDTO` has **no `state`/`scheduledDeletionAt`**; `AuthUserDTO` has none of `disabled`/`banReason`/`deletionStatus`/`scheduledDeletionAt`.
- **Translations:** all required namespaces (DASHBOARD_PAGES, BUTTONS, DIALOG, MESSAGES_PAGE, COMMON, ERRORS) are fetched (full enum fetched at boot).
- **Navigation:** no `(secured)/_layout`; inline per-screen `if (!user) <Redirect href="/" />` guards are the SessionGuard analog; `bootStore` does not re-gate on sign-out.

## Brief vs reality

Nothing. The brief was an audit request and matched the code as described; the divergences I found (boolean vs object store flags, interceptor header-clobber, missing DTO fields) are reported as audit findings in the deliverable, not as brief errors — there was no instruction to write code against a wrong contract.

## Obsoleted by this session

Nothing. (The pre-build readiness audit `audit-expo-readiness-user-deletion.md`, 2026-05-23, is a different artifact — a web/backend-readiness pre-foundation audit — and remains valid in its own scope; this new deliverable is the expo current-state audit the implementation briefs need. Not deleted.)

## Cleanup performed

None needed — read-only audit, no source touched.

## Conventions check

- **Part 4 (cleanliness):** no source files changed; nothing to lint/type-check/test. No `console.log`/TODO/FIXME introduced. (No `npm run lint`/`tsc`/`test` run — no touched code paths to gate.)
- **Part 4a (simplicity):** N/A — no code produced.
- **Part 4b (adjacent observations):** Noted but not filed as issues (audit, not fix): (a) the `allowNotifications` toggle in `app/owner/dashboard/user.tsx:47` is fully dead (never seeded, never compared, never saved); (b) the interceptor's unconditional `Authorization` overwrite (`api.ts:48-50`) means any future per-call auth-header override is silently clobbered — a latent gotcha beyond this feature. Both are recorded in the deliverable for Mastermind/Docs-QA triage rather than fixed here.
- **Hard rules:** no commits/pushes/branch changes; no edits to `app.config.ts`/`eas.json`/native config; no cross-repo edits; no writes to the four `oglasino-docs` config files.

## Config-file impact

No config-file edit made (correctly — engineer agents do not write the four files). **For Docs/QA to consider:** the User-Deletion Expo-backlog row in `state.md` could note that the mobile current-state audit (this deliverable) is complete and the implementation briefs can be authored against `new-expo-dev` (HEAD `b67627c`). I am not asserting a status change — drafted in "For Mastermind" below.

## For Mastermind

1. **Implementation-brief inputs are ready.** The deliverable's "Cross-cutting NOT-PRESENT summary" enumerates the eight additions the feature requires and the "What is already in place" list flags the scaffolds to reuse — both can seed the brief breakdown.
2. **Store-flag shape decision needed.** Mobile's `restored`/`accountBanned` are plain booleans, not web's object shape. The mobile ban dialog is intentionally reason-less, so a boolean `accountJustDeleted: string | null` (carrying `scheduledDeletionAt`) is the natural fit — confirm the brief should follow the mobile boolean/string-flag convention rather than porting web's object shape.
3. **Auth-header gotcha.** The brief for the delete request must NOT specify "pass the fresh token as a per-call `Authorization` header" (web's pattern) — mobile's interceptor clobbers it. The brief should instead require reauth → `getIdToken(true)` immediately before the call so the interceptor's own `getIdToken()` returns the refreshed token on the same `currentUser`.
4. **Possible `state.md` Expo-backlog note** (Docs/QA to apply): User Deletion mobile — current-state audit complete (`oglasino-expo` `.agent/audit-user-deletion-current-state.md`), implementation briefs pending.
