# Session summary

**Repo:** oglasino-web
**Branch:** feature/user-deletion
**Date:** 2026-05-18
**Task:** Web Engineering Brief B — User Deletion Danger Zone, dialogs, and trust-boundary close. Phases 2–8: `CallUserButton` correction from Brief A, review-image upload scope, `BANNED_DIALOG` namespace registration, Danger Zone section on settings page, deletion confirmation dialog with provider-branch reauth, three root-layout dialogs (post-deletion, restoration, ban-notice), and web half of the §16 trust-boundary audit.

## Implemented

- **Phase 2 (`CallUserButton` correction).** `ProductFunctions.tsx` no longer conditionally wraps `<CallUserButton>` — the button is always rendered, with `callingAllowed={(owner.allowPhoneCalling || false) && owner.state === 'ACTIVE'}` (the conjunction the brief specified). UX result: a `PENDING_DELETION` user's call icon renders in its disabled state, identical to any user who has phone calling toggled off. `CallUserButton` itself stays state-agnostic.
- **Phase 3 (review-image upload scope).** Added `'review'` to the `UploadScope` union in `src/lib/images/uploadImages.ts`; changed the call in `reviewService.ts` from `'product'` to `'review'`. No client-side prefix dispatch exists (verified by reading `uploadImages.ts:174-204`) — the scope is passed straight through to the backend's `/secure/images/upload-tokens` endpoint, so only the union update was needed.
- **Phase 4 (`BANNED_DIALOG` namespace).** Added `BANNED_DIALOG = 'BANNED_DIALOG'` to `TranslationNamespaceEnum`. The frontend's only parallel allowlist is `Object.values(TranslationNamespaceEnum)` in `src/i18n/internalRequest.ts:8` — automatically picks up the new entry.
- **Phase 5 (Danger Zone).** Added a new section at the bottom of `app/[locale]/owner/user/page.tsx`, below the existing submit-and-location row, separated by a `border-border-mild` top border. Heading + sub-heading + four-bullet list + restore note + `<Button variant="destructive">` matching the existing precedent (admin `CacheEvictionPanel`, `BackendCachePanel`). The button opens the deletion-confirmation dialog via `DialogManager`.
- **Phase 6 (deletion confirmation dialog).**
  - New component `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx`, registered as `DialogId.DELETE_ACCOUNT_CONFIRMATION_DIALOG` and wired into `DialogManager.tsx`'s `DIALOGS` map. Built on top of `DrawerDialog` (same shape `InfoDialog` uses) — couldn't use `InfoDialog` directly because the password branch needs a body slot for the input, and `InfoDialog`'s API is title+description+continue only.
  - Provider detection reads `auth.currentUser?.providerData[0]?.providerId ?? 'password'` (Firebase as source of truth). The codebase's only existing provider check (`/owner/user/page.tsx:217`) reads `user.providerId` from `AuthUserDTO` for an "is this OAuth user" gate — but that value is undefined for email/password users, which doesn't help branch into `'password'`. Firebase's `providerData[0].providerId` returns the spec's exact branch values (`'password'` / `'google.com'` / `'facebook.com'`). The `'unsupported'` branch renders Cancel only.
  - Three branches: email/password uses `EmailAuthProvider.credential + reauthenticateWithCredential`; Google uses `reauthenticateWithPopup(currentUser, googleProvider)`; Facebook uses the same with `facebookProvider` (case present, dead code at launch per spec §3.5 / §14.2).
  - After reauth: `await currentUser.getIdToken(true)` (forced refresh) → `axios.post(DELETE_ENDPOINT, {}, { headers: { Authorization: 'Bearer ' + freshToken } })` with **raw axios**, not `BACKEND_API`. `BACKEND_API`'s request interceptor unconditionally overwrites `Authorization` with its cached token (`api.ts:88-95`), which would defeat the forced refresh — the backend would see the stale `auth_time` claim. Raw axios is the spec-quoted approach (Brief 6.4 / spec §12.1 / §14.3).
  - Response handling per brief: 200 → `sessionStorage.setItem('account-just-deleted', scheduledDeletionAt)` BEFORE `auth.signOut()` (so `SessionGuard`'s redirect doesn't race the key write); 403 + `REAUTH_REQUIRED` → keep dialog open with `errors.reauth.required`; 403 + `USER_LOCKED_FROM_DELETION` → show `errors.user.locked.from.deletion` and lock the Delete button; 5xx → generic `system.error`, keep open. Discrimination uses Brief A's `isErrorWithCode` helper.
  - Reauth failure (wrong password, popup cancelled, network) is caught and surfaces a generic `errors.reauth.required` — Firebase's specific error codes aren't shown verbatim because the user knows what they just typed.
- **Phase 7 (three root-layout dialogs).**
  - New initializer `src/components/client/initializers/AccountStateDialogsInit.tsx`, mounted from `AppInit.tsx` alongside `UseTokenRefresh`, `ChatsInit`, etc.
  - Followed the existing pattern (e.g., `CallUserButton`, `ProductFunctions`): reuse `DialogId.INFO_DIALOG` with different props per call rather than registering three new DialogIds for what is structurally the same dialog. See "For Mastermind" note 1 — this is a judgment call.
  - **Post-deletion** triggers on mount-time `sessionStorage.getItem('account-just-deleted')`; one-shot, key removed on open. Description is a two-paragraph ReactNode containing the locale-formatted date and the restore instruction. Date formatted via `toLocaleDateString(locale, ...)` with the active next-intl locale — the codebase's only existing date helper hardcodes `'sr-RS'` (`admin/chats/Chat.tsx:5-15`), so I rolled the locale-aware version inline.
  - **Restoration** triggers on `useAuthStore.restored === true` (Brief A wired this from the response interceptor). The effect flips it back to `false` on open. Uses the new `autoDismissAfterMs={10000}` prop on `InfoDialog`.
  - **Ban-notice** triggers on mount-time `sessionStorage.getItem('account-banned')`; one-shot, key removed on open. Three-paragraph body. Static — does not identify the user or display a reason (privacy reason per spec §4.7).
  - `InfoDialog.tsx` gained two small optional props: `autoDismissAfterMs?: number` (mount-effect `setTimeout` calling `onClose`) and `closeButtonLabel?: string` (overrides the default `tDialog('button.close.label')`). The spec's "Go to home" button labels (`buttons.banned.go.home.label`, `buttons.account.deleted.go.home.label`) flow in via the second prop.
  - **Mount-order note.** `DialogManager` is a single-slot store (`useDialogStore` is keyed by `currentDialogId` with `closeDialog` resetting; opening a new dialog REPLACES the current one). If both sessionStorage keys are somehow present on the same mount, the second `openDialog` overwrites the first. The init reads ban first, then post-deletion second — so on the (impossible-in-practice) double-trigger, post-deletion wins, which is the brief's stated precedence (a deleting user is not simultaneously banned mid-session). Restoration runs on a separate effect (deps `[restored]`), independent of the mount-time pair.
- **Phase 8 (trust-boundary audit, web half).** Written to `oglasino-web/.agent/trust-boundary-audit-user-deletion.md`. Seven user-facing surfaces audited (Danger Zone submission, post-deletion sessionStorage trigger, restoration `X-Account-Restored` header, ban-notice sessionStorage trigger, global 403+`USER_BANNED` interceptor, profile-page badge rendering, `CallUserButton` gate). Zero CRITICAL findings. Two informational-only sessionStorage triggers are technically self-writable but no state transition follows from them. Five response- or server-data-driven surfaces are unforgeable client-side. Closes §16 of the feature spec from the frontend side.

## Files touched

- `src/components/client/ProductFunctions.tsx` (+4 / -3)
- `src/lib/images/uploadImages.ts` (+1 / -1)
- `src/lib/service/reactCalls/reviewService.ts` (+1 / -1)
- `src/translations/types/TranslationNamespaceEnum.ts` (+3 / -0)
- `src/components/popups/dialogRegistry.ts` (+1 / -0)
- `src/components/popups/DialogManager.tsx` (+2 / -0)
- `src/components/popups/dialogs/InfoDialog.tsx` (+12 / -2)
- `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx` (NEW, +166)
- `src/components/client/initializers/AccountStateDialogsInit.tsx` (NEW, +98)
- `src/components/client/initializers/AppInit.tsx` (+2 / -0)
- `app/[locale]/owner/user/page.tsx` (+24 / -0)
- `.agent/trust-boundary-audit-user-deletion.md` (NEW — audit deliverable)

## Tests

- Ran: `npm test` (vitest)
- Result: 10 files / 154 tests passing, 0 failing.
- New tests added: none. All touched paths are dialog wiring, scope-string changes, type extensions, and pure React glue — the patterns in this repo do not have unit-test coverage for `DialogManager`-mediated dialog flows or for `useEffect`-driven `sessionStorage` initializers. Noted in "Known gaps" below.
- Ran: `npx tsc --noEmit` — clean.
- Ran: `npm run lint` — 0 errors, 208 warnings, all pre-existing (same count as session start; this brief added 0 new warnings).

## Cleanup performed

- None needed. No commented-out code, no `console.log` introduced, no dead imports, no unused vars in any of the touched files. The four new files (`DeleteAccountConfirmationDialog.tsx`, `AccountStateDialogsInit.tsx`, `trust-boundary-audit-user-deletion.md`, and the touched config files) are all referenced by something — the dialog is registered in `DialogManager`, the initializer is mounted in `AppInit`, the audit file is referenced from the spec §16 and from this summary.
- Brief A's "For Mastermind" observation 4 (`CallUserButton` hidden at call site) is resolved by Phase 2 of this brief — the conditional wrap is gone; the button gates via the existing `callingAllowed` prop. That observation is now closed.

## Config-file impact

- **conventions.md**: no change.
- **decisions.md**: no change. (Per spec §20.8, the shipped-feature `decisions.md` entry is drafted post-merge by a Docs/QA wrap session, not in this engineer session.)
- **state.md**: no change. (Status flip from `backend-stable` to `in-progress-web` to `web-stable` is Mastermind/Docs/QA's call once they verdict this session and Brief A.)
- **issues.md**: no change. Adjacent observations are flagged in "For Mastermind" for Mastermind to route.

## Obsoleted by this session

- Nothing. Brief A's `ProductFunctions.tsx` conditional wrap on `<CallUserButton>` was reverted (Phase 2) — that was a Brief A artifact corrected by this brief per its own self-description; not separate dead code, deleted in the same edit.

## Conventions check

- **Part 4 (cleanliness):** confirmed. No `console.log` added, no commented-out blocks, no orphan files, no new `TODO`/`FIXME`.
- **Part 4a (simplicity):** confirmed. The two `InfoDialog` prop additions (`autoDismissAfterMs`, `closeButtonLabel`) are tiny optional extensions to an existing component, both with a concrete present-day caller in this brief. The new `DeleteAccountConfirmationDialog` is a new component because `InfoDialog`'s API has no body slot for the password input; not a parallel abstraction. The new initializer follows the existing `*Init.tsx` pattern (`ChatsInit`, `BaseSiteInit`, `PushInitializer`, etc.).
- **Part 4b (adjacent observations):** three observations logged in "For Mastermind."
- **Part 6 (translations):** confirmed. New namespace `BANNED_DIALOG` registered in `TranslationNamespaceEnum` per spec §15.7. No parent/leaf collisions — all consumed keys are leaves (verified mentally against the four `BANNED_DIALOG` keys and the `DASHBOARD_PAGES` / `COMMON_SYSTEM` / `BUTTONS` / `ERRORS` keys this brief consumes). Spec §15.8 explicitly confirms the leaf-only structure.
- **Part 7 (error contract):** confirmed. `DeleteAccountConfirmationDialog` reads `{errors: [{field, code, translationKey}]}` via `isErrorWithCode`. The four discriminated codes (`REAUTH_REQUIRED`, `USER_LOCKED_FROM_DELETION`, plus 5xx and generic) match spec §8.8.
- **Part 11 (trust boundaries):** confirmed and audited end-to-end in `.agent/trust-boundary-audit-user-deletion.md`. Every state transition reacts to a server signal; no client-set value gates any authorization or moderation decision.

## Known gaps / TODOs

- **No automated test added** for the deletion-confirmation dialog or the three root-layout dialog triggers. The repo has no `@testing-library/react` precedent (per `issues.md` 2026-05-14 "Web component-render test coverage gap"), so dialog rendering and sessionStorage-driven mount effects aren't testable today without infrastructure work. Behaviour verified by reading.
- **Backend translation seeds.** This brief consumes 23 translation keys across `DASHBOARD_PAGES`, `BUTTONS`, `ERRORS`, `COMMON_SYSTEM`, and the new `BANNED_DIALOG` namespace (per spec §15.1–§15.7). If the backend hasn't seeded a key, the UI renders the raw dotted string. Per brief Phase 4 wording, do not stub client-side — it resolves once Backend seeds. Drafting that list (so Igor can pass it to Backend) is below in "For Mastermind."
- **Date format.** `toLocaleDateString(locale, ...)` with day/month/year. If next-intl's `useFormatter` is preferred for consistency with a future shared formatter, this is the trivial point of change. Today the codebase has only one date helper and it's hardcoded to `'sr-RS'` — adopting `useFormatter` would be its own small consistency PR, out of scope here.
- **Facebook reauth is dead code at launch.** Per spec §3.5 / §14.2, the case is implemented (imports `facebookProvider`, calls `reauthenticateWithPopup(currentUser, facebookProvider)`) so it lights up when Facebook is enabled backend-side. Today no user has `providerData[0].providerId === 'facebook.com'` because Facebook sign-in is not enabled per Brief A.

## For Mastermind

1. **Judgment call: reusing `INFO_DIALOG` instead of three new DialogIds for the root-layout dialogs.** The brief says "three registrations against the same component, not three new components." I read this as "don't build three new UI components with different markup," which the literal `INFO_DIALOG`-reuse-with-props pattern (matching `CallUserButton`'s idiom) satisfies. Alternative interpretation would have me add three entries (`ACCOUNT_DELETED_DIALOG`, `ACCOUNT_RESTORED_DIALOG`, `ACCOUNT_BANNED_DIALOG`) to `dialogRegistry.ts` and three entries in `DialogManager.tsx`'s `DIALOGS` map, all pointing at `InfoDialog`. That would give each dialog a discoverable named identity at the call site. Trade-off: more registry rows, no behaviour change. Engineer's choice; mention if Mastermind would prefer the three-named-IDs approach.

2. **`InfoDialog` API drift.** I added two optional props (`autoDismissAfterMs?: number` and `closeButtonLabel?: string`) to an existing shared component to satisfy the restoration dialog's auto-dismiss and the post-deletion / ban-notice dialogs' "Go to home" button labels. Both are tiny, fully-additive, and have concrete callers today. The brief explicitly permitted this (Phase 7.2: "If `InfoDialog` has an `autoDismissAfterMs` (or similar) option, use it. Otherwise wire a `setTimeout` in the dialog's mount effect"). Flagging because it widens an existing component's API rather than introducing a wrapper.

3. **Adjacent observation: `BACKEND_API_URL` is duplicated.** `process.env.NEXT_PUBLIC_API_URL as string` is read in five places: `api.ts:6`, `fetchApi.ts:3`, `translationsCache.ts:28`, `getConfig.ts:7`, and now `DeleteAccountConfirmationDialog.tsx:18`. Each reads the env directly because `api.ts` keeps `BACKEND_API_URL` as a module-local constant (not exported). Small extraction opportunity: export the constant from `api.ts` (or a tiny `src/lib/config/backendUrl.ts`). **Severity: low** (Part 4b). File path: cross-cutting. Out of scope this brief; flagging.

4. **Adjacent observation: the codebase has one date helper, hardcoded to `'sr-RS'`.** `src/components/admin/chats/Chat.tsx:5-15`'s `formatDate` always renders Serbian-locale dates regardless of the active site language. Surfaced because Phase 7.1 needed a locale-aware date for the post-deletion dialog and I had to write the formatter inline (`new Date(iso).toLocaleDateString(locale, ...)` with the active next-intl locale). A small project-wide cleanup would extract a shared locale-aware formatter (probably via `next-intl`'s `useFormatter` for client components, plain `Intl.DateTimeFormat` for server). **Severity: low** (Part 4b). File path: `src/components/admin/chats/Chat.tsx:5-15`. Out of scope this brief; flagging.

5. **Spec example precision: `BACKEND_API + URL` vs raw axios.** Spec §14.3 / Brief 6.4 quote `axios.post(BACKEND_API + '/api/secure/user/me/delete', {}, { headers: { Authorization: 'Bearer ' + freshToken } })`. Strictly read, that uses `BACKEND_API` as a string concatenation — but `BACKEND_API` in this codebase is the axios *instance* (`api.ts:71`), not a string. The string constant is `BACKEND_API_URL` (module-local). I read the spec example as illustrative (concatenated string + raw axios with explicit header to bypass the BACKEND_API interceptor's token override) and implemented accordingly with raw `axios` + `process.env.NEXT_PUBLIC_API_URL`. The intent of the spec (bypass the cached-token interceptor with an explicit fresh token) is preserved.

6. **Translation keys this brief consumes that Backend needs to seed.** If the user-deletion SQL seed didn't include these yet, list them for Backend:
   - **`COMMON_SYSTEM`**: `common.system.account.restored.title`, `common.system.account.restored.subtitle`, `system.error` (likely existing).
   - **`BUTTONS`**: `buttons.delete.account.label`, `buttons.reauthenticate.and.delete.label`, `buttons.banned.go.home.label`, `buttons.account.deleted.go.home.label`.
   - **`DASHBOARD_PAGES`**: `dashboard.pages.danger.zone.label`, `dashboard.pages.delete.account.title`, `dashboard.pages.delete.account.bullet.listings`, `dashboard.pages.delete.account.bullet.profile.badge`, `dashboard.pages.delete.account.bullet.messaging`, `dashboard.pages.delete.account.bullet.signout`, `dashboard.pages.delete.account.restore.note`, `dashboard.pages.delete.account.modal.title`, `dashboard.pages.delete.account.modal.body.password`, `dashboard.pages.delete.account.modal.body.google`, `dashboard.pages.delete.account.modal.password.label`, `dashboard.pages.delete.account.modal.cancel.label`, `dashboard.pages.account.deleted.dialog.title`, `dashboard.pages.account.deleted.dialog.scheduled.date` (with `{date}` interpolation), `dashboard.pages.account.deleted.dialog.restore.instruction`.
   - **`ERRORS`**: `errors.reauth.required`, `errors.user.locked.from.deletion`, `system.error` (likely existing — the existing `tErrors('system.error')` is used at `MessageInput` and others).
   - **`BANNED_DIALOG`**: `banned.dialog.title`, `banned.dialog.body.first`, `banned.dialog.body.delete.intro`, `banned.dialog.body.duration`.
   Mapping is 1:1 with spec §15.1–§15.7. Per spec §20.5 / §20.6 all four locales (EN, SR, RU, CNR) need values.

7. **`UsetTokenRefresh` rotation-time-disabled gap unchanged.** Per the brief Phase 7.5 / Brief A's "For Mastermind" observation 2: `UsetTokenRefresh.tsx` still fires `BACKEND_API.post('/auth/firebase-sync', ...)` and discards the result, so a `200 + disabled: true` on token rotation isn't read. The next backend call closes the window via the global 403 + `USER_BANNED` interceptor that Brief A added. Not in scope per the brief; flagging that it stays as-is.

(or: nothing flagged) — six adjacent observations and one judgment call above; otherwise the implementation hewed to the brief and spec.
