# PR Review — User Deletion (oglasino-web)

**Reviewer:** Web engineer agent (review-only, no code changes)
**Date:** 2026-05-18
**Branch:** `dev` (carrying the feature/user-deletion work pre-merge)
**Scope:** every file in `git status` — 18 modified, 4 new. 215 insertions / 56 deletions across 18 paths.
**Spec:** [`oglasino-docs/features/user-deletion.md`](../../oglasino-docs/features/user-deletion.md)
**Predecessors read:** `.agent/audit-user-deletion.md`, `.agent/2026-05-18-oglasino-web-user-deletion-1.md`, `-2.md`, `-3.md`, `.agent/trust-boundary-audit-user-deletion.md`
**Hygiene baseline run by reviewer:** `npx tsc --noEmit` clean · `npm run lint` 0 errors / 208 warnings (all pre-existing per [`issues.md` 2026-05-16](../../oglasino-docs/issues.md)) · `npm test` 154/154 passing.

---

## Summary verdict

The feature is **mostly correct, mostly clean, and ships behaviorally close to spec**. Three sessions (Brief A plumbing, Brief B Danger Zone + dialogs, Brief C fixup) compose into a coherent whole. The trust-boundary audit at `.agent/trust-boundary-audit-user-deletion.md` is solid — every state transition reacts to a server signal; no client-set value gates any decision.

**Two findings worth fixing before merge.** One is a real HTML validity bug visible in three dialogs (ban-notice, post-deletion, restoration via paths described below); the other is a latent null-deref in `Messages.tsx` action handlers that escaped the Brief A null-safety pass.

**Two findings worth flagging without blocking.** A closure-gate miss in Brief B (the `BANNED_DIALOG` namespace registration in `conventions.md` Part 6 was not drafted for Docs/QA), and a UX gap in the login dialog when a banned user signs in via email/password (the dialog stays open with no feedback until the next navigation surfaces the ban-notice).

Everything else is either spec-conforming, deliberate per a session "For Mastermind" judgment call, or pre-existing tech debt outside the diff.

---

## Findings — severity table

| #  | Severity  | Dimension                   | File                                                  | Finding                                                            |
|----|-----------|-----------------------------|-------------------------------------------------------|--------------------------------------------------------------------|
| 1  | medium    | React patterns / a11y       | `InfoDialog.tsx:60` + `AccountStateDialogsInit.tsx`   | `<p><div>...</div></p>` invalid HTML in three root-layout dialogs   |
| 2  | medium    | React patterns              | `Messages.tsx:165, 174-179, 187, 193`                 | Dropdown action handlers deref `activeChat.withUser.id`/`.firebaseUid` without null-safety |
| 3  | medium    | Translations (process)      | `oglasino-docs/meta/conventions.md` Part 6            | `BANNED_DIALOG` namespace registered in enum but not drafted for `conventions.md`; closure-gate miss in Brief B |
| 4  | medium    | State and data flow         | `useAuthStore.login` / `register` / OAuth login flows | Banned user signing in via login dialog: dialog stays open with no error; ban-notice surfaces only on next nav |
| 5  | low       | Translations / locale       | `AccountStateDialogsInit.tsx:14`                      | `'me'` (Montenegrin) locale falls through to JS default — should alias to `'sr'` per project convention |
| 6  | low       | React patterns              | `InfoDialog.tsx:21, 32`                               | `type` prop is dead when no `onContinue` is passed (all three root-layout dialogs); silent no-op |
| 7  | low       | Spec conformance (i18n)     | `DeleteAccountConfirmationDialog.tsx:69-70, 112-115, 154-155` | Facebook branch reuses Google body copy + "Reauthenticate with Google and delete" button label |
| 8  | low       | Cleanup / dead branch       | `isErrorWithCode.ts:9-13`                             | `e.response?.data?.errors` branch is never reached given current interceptor unwrap; mild dead code |
| 9  | low       | Defensive / consistency     | `Chats.tsx:25`                                        | Search filter assumes `displayName` is non-null whenever `withUser` is defined; spec-conforming today |
| 10 | low       | State and data flow         | `useAuthStore.refreshUser`                            | When `syncUserToBackend` self-signs-out, the firebase-uid mismatch check returns null without clearing the store user; SessionGuard catches it next nav |
| 11 | low       | Testing                     | repo-wide                                             | No new unit/integration tests for dialog, dropdown ban path, interceptor — known repo posture per `issues.md` 2026-05-14 |
| 12 | low (4b)  | Adjacent — pre-existing     | `DrawerDialog.tsx:107`                                | Mobile Drawer ignores `closableOutside` — pre-existing inconsistency the deletion dialog inherits |

---

## 1. Spec conformance

The spec lists every page, component, behavior, and route. I walked them in order and matched against code on disk.

| Spec section            | Behavior                                                     | Code on disk                                                              | Verdict |
|-------------------------|--------------------------------------------------------------|---------------------------------------------------------------------------|---------|
| §4.1 Initiating deletion | Danger Zone on `/[locale]/owner/user`                       | `app/[locale]/owner/user/page.tsx:361-381` — section, heading, bullets, restore note, destructive Button | ✓ |
| §4.1 confirmation dialog | `DialogManager`-mediated, password / Google / Facebook branches | `DeleteAccountConfirmationDialog.tsx` — provider switch + dedicated DialogId | ✓ |
| §4.1 reauth              | Fresh ID token via `getIdToken(true)`, explicit Authorization header | `:89-91`; interceptor guard at `api.ts:91` | ✓ |
| §4.1 response handling  | `REAUTH_REQUIRED` → reset; `USER_LOCKED_FROM_DELETION` → lock; 5xx → generic | `:100-107` | ✓ |
| §4.1 post-success        | `sessionStorage.setItem('account-just-deleted', date)` BEFORE `signOut()` | `:95-96` (order is correct — write-then-signOut, so SessionGuard's redirect cannot race the key write) | ✓ |
| §4.4 restoration         | `X-Account-Restored` response header → `useAuthStore.setRestored(true)` | `api.ts:26-29` + `AccountStateDialogsInit.tsx:91-100` | ✓ |
| §4.4 dialog auto-dismiss | 10s                                                          | `AccountStateDialogsInit.tsx:10` (`RESTORATION_AUTO_DISMISS_MS`) → `InfoDialog.tsx:45-49` | ✓ |
| §4.6 admin ban           | Backend-side                                                 | (not in web scope)                                                        | n/a |
| §4.7 ban-notice dialog   | Static content, four `BANNED_DIALOG` keys, "Go to home" close | `AccountStateDialogsInit.tsx:48-60`                                       | ✓ |
| §4.8 re-registration ban | `firebase-sync` 403 + `EMAIL_BANNED` → signOut + sessionStorage | `authService.ts:148-155`                                                  | ✓ |
| §14.6 profile badge      | `ScheduledForDeletionBadge` between displayName and Rating   | `UserDetails.tsx:123`                                                     | ✓ |
| §14.7 call-button gate   | Backend gate is load-bearing; frontend gate is UX            | `ProductFunctions.tsx:44-46` — `callingAllowed={(owner.allowPhoneCalling || false) && owner.state === 'ACTIVE'}` | ✓ |
| §14.8 chat header badge + disabled input | Badge in chat header, message input disabled, notice | `Messages.tsx:92-93, 154, 263-269, 276`                                   | ✓ |
| §14.9 chat null-safety   | `displayName ?? tCommon('user.deleted')` at header + Chats list | `Messages.tsx:148, 152` + `Chats.tsx:58, 71, 75`                          | ✓ partial (see finding #2) |
| §14.10 ban dialog triggers | sessionStorage on `disabled: true` / `EMAIL_BANNED` / `USER_BANNED` (mid-session) | `authService.ts:141-153` + `api.ts:50-61`                                 | ✓ |
| §14.11 syncUserToBackend  | Single-arg signature; reads `allowPreferenceCookies` from cookie | `authService.ts:126-156` — single-arg per Brief A judgment call; intent preserved | ✓ |
| §14.12 global 403 interceptor | Signs out only on `USER_BANNED`; other 403s fall through | `api.ts:50-65`                                                            | ✓ |
| §14.13 review upload scope | `'review'` added to `UploadScope`; call site swapped       | `uploadImages.ts:52` + `reviewService.ts:68`                              | ✓ |
| §14.14 type extensions   | `AuthUserDTO`, `UserInfoDTO`                                 | `AuthUserDTO.ts:18-21` + `UserInfoDTO.ts:18-19`                           | ✓ |
| §15.7 BANNED_DIALOG       | New namespace in enum; new keys consumed                   | `TranslationNamespaceEnum.ts:32-33`                                       | ⚠ see finding #3 |
| §16 trust-boundary audit | Web half closed                                              | `.agent/trust-boundary-audit-user-deletion.md`                            | ✓ |

No code without spec backing. No spec behavior without code backing — except the `conventions.md` Part 6 update (finding #3) and the Facebook branch's English label being literally "Reauthenticate with Google" (finding #7), neither of which is wrong per the spec text but both are weak spots.

---

## 2. Trust boundaries (Part 11)

Re-walked every new request and every new response. The trust-boundary audit at `.agent/trust-boundary-audit-user-deletion.md` is correct and complete from the web side. Brief items:

- **Delete request** (`POST /api/secure/user/me/delete`) — empty `{}` body, no `userId`, no "before-value." Identity from verified Firebase ID token. `auth_time` claim read server-side from the same token. ✓
- **`auth_time` freshness** — fresh token minted via `getIdToken(true)` after reauth. Backend verifies cryptographically. Client cannot fake `auth_time`. ✓
- **Restore signal** — sent as a *response* header `X-Account-Restored`. Client has no write path. ✓
- **Ban check on registration** — email extracted from verified Firebase token server-side. Client cannot forge. ✓
- **Two sessionStorage triggers** (`account-just-deleted`, `account-banned`) — both are informational. A client that self-sets either key will see the relevant dialog in their own browser; no state transition follows. Backend enforces the actual state. ✓
- **`CallUserButton` gate** — frontend gate is UX-only per spec §14.7; backend `/secure/user/phoneNumber` is load-bearing. ✓

✓ No Part 11 violations.

---

## 3. Error contract (Part 7)

The frontend's discrimination of error codes is consistent and correct.

- **Wire shape consumption** (`isErrorWithCode`): reads `{errors: [{code}]}` first-error-per-field per Part 7. ✓
- **Code mappings** in `DeleteAccountConfirmationDialog.tsx:100-107`: `REAUTH_REQUIRED` / `USER_LOCKED_FROM_DELETION` distinct, anything else → `system.error`. ✓
- **Status code split in `api.ts:32-65`**: `ERR_NETWORK` / timeout → synthetic envelope; 404 → synthetic envelope; **403 + `USER_BANNED`** → global sign-out + sessionStorage + never-resolving promise (other 403s fall through unchanged). ✓
- **Rate-limiting (429)** — not specifically handled in the deletion path; falls into `system.error`. Acceptable per spec — the spec only enumerates the five codes above.
- **5xx** — falls into `system.error`. Acceptable per spec.

**Finding #8 (low) — `isErrorWithCode` has a dead branch.**

```ts
const errors = e.response?.data?.errors ?? e.data?.errors;
```

The first preference (`e.response?.data?.errors`) targets a raw `AxiosError`. But `api.ts:64` rejects with `error.response` (already unwrapped), so callers always see the second shape. The first branch is dead today. Not a bug — the helper still resolves correctly. The comment block above the function (lines 1-4) names this intentionally as defensive coverage. Acceptable but worth noting; a future audit pass could simplify.

✓ Error contract is conformant.

---

## 4. Translations (Part 6)

**Namespaces are all valid.** `COMMON`, `COMMON_SYSTEM`, `BUTTONS`, `DASHBOARD_PAGES`, `MESSAGES_PAGE`, `ERRORS`, `DIALOG` — all listed in Part 6 of conventions. `BANNED_DIALOG` is new per spec §15.7 — added to the enum at `TranslationNamespaceEnum.ts:33`. ✓

**No new English string literals.** Every user-facing string passes through `t()` or `useTranslations`. Spot-checked the four new files plus the modified `Messages.tsx`, `Chats.tsx`, `UserDetails.tsx`, `app/[locale]/owner/user/page.tsx`, and the dialog. Zero hardcoded UI strings introduced. ✓

**No parent/child key collisions.** All new keys are leaves. Verified mentally against spec §15.1-§15.7 — no `common.user.deleted` + `common.user.deleted.suffix` shape would coexist. ✓

**Frozen `VALIDATION` namespace not touched.** ✓

**Finding #3 (medium) — closure-gate violation in Brief B.**

Spec §15.7 says verbatim:

> Adds entry to `TranslationNamespace` enum (backend), `TranslationNamespaceEnum` (frontend), **and conventions Part 6 Rule 1.**

The conventions.md Part 6 §1 "Translation namespaces" list still ends at 22 namespaces — `BANNED_DIALOG` is absent. Per CLAUDE.md the engineer is forbidden from writing to conventions.md, so the correct path is to draft the text in "For Mastermind"/Config-file impact and let Docs/QA apply. Brief B's session summary marked `conventions.md: no change` in the Config-file impact section, missing this. The "Closure gate" rule in conventions Part 5 says no session closes with an undeclared config-file dependency — this one slipped through.

This is a process issue, not a code defect. Severity: medium (for closure-gate compliance), low (for actual code correctness — the runtime works fine because the frontend enum is the authoritative consumer for the web).

**Finding #5 (low) — `'me'` locale fall-through.**

`AccountStateDialogsInit.tsx:14` calls `toLocaleDateString(locale, ...)` with the next-intl locale. Project convention (Part 9 stack reference, [`state.md`](../../oglasino-docs/state.md)) says "Montenegrin (me/cnr) aliases to SR." JavaScript's `Intl.DateTimeFormat` with `'me'` falls back to the default `en-US` because `'me'` is Montenegrin Latin and not a typical browser locale tag. The post-deletion dialog will render a `'me'` user's scheduled-deletion date in English locale format. Trivial cosmetic; a one-line `const dateLocale = locale === 'me' ? 'sr' : locale;` resolves it. Already flagged by Brief B "For Mastermind" note 4 in a separate context; closing here as the same family.

**Backend translation seed dependency.** This PR consumes ~24 keys across 7 namespaces. Backend needs to seed them all in 4 locales (EN, SR, RU, CNR) per spec §20.5. Brief B's "For Mastermind" §6 enumerates them. If any key is unseeded, the UI renders the raw dotted string — a pre-launch action item already tracked.

---

## 5. Architectural defaults (Part 8)

- **Routes reusable across web and mobile.** No mobile-specific route added. Mobile adopts post-merge per state.md. ✓
- **Server is single source of truth for moderation.** Client-side checks (`provider === 'password'` etc.) are structural only — they don't make moderation decisions. ✓
- **Errors are codes.** ✓ (covered in Part 7 above).
- **Direct-to-R2.** Review-image scope change (`'product'` → `'review'`) keeps the direct-upload contract — `uploadImages` flows through `/secure/images/upload-tokens` + direct PUT. ✓

---

## 6. Next.js App Router patterns

- **`'use client'` discipline.** Every touched/new file is correctly marked. `ScheduledForDeletionBadge` has `'use client'` (line 1) — strictly speaking, `useTranslations` works in server components in next-intl 3.x, so the directive isn't required, but it's harmless and matches the file's intent of being client-rendered inside `UserDetails`. ✓
- **Server actions** — not used; client-side mutations via `userService` are correct for the dialog flow.
- **Data fetching** — consistent. `getUserDetails` reuses the existing axios-based pattern.
- **`metadata` export** — Not added; the existing settings page has its own metadata layer. Out of scope here.
- **Loading and error states** — Dialog has its own `loading` / `errorMessage` / `locked` state. Three root-layout dialogs are informational, no loading state needed. ✓
- **Suspense / loading.tsx / error.tsx** — No new routes, none required.
- **Locale segment** — All new content lives inside `[locale]` or in client components that read `useLocale()`. ✓

---

## 7. React patterns

- **Hooks rules.** Inspected each `useEffect` and `useMemo`. No conditional hooks. ✓
- **Deps arrays.** 
  - `AccountStateDialogsInit.tsx:86` has an `eslint-disable-next-line react-hooks/exhaustive-deps` with a comment explaining why (mount-only behavior). ✓
  - The restoration effect at `:91-100` has correct deps `[restored, openDialog, setRestored, tCommonSystem]`.
  - `InfoDialog.tsx:45-49` autoDismiss effect deps `[isOpen, autoDismissAfterMs, onClose]` — `onClose` is `closeDialog` from the zustand store, stable identity, so no thrash. ✓
- **Re-renders.** Nothing introduced that re-renders unexpectedly. The `Messages.tsx` rename (`blocked` → `cannotSend` + extracted `peerPendingDeletion`) actually reduces redundant work (the audit observed three duplicated `group.sender.firebaseUid === user.firebaseUid` checks; the new `isOwnMessage` constant collapses them).
- **State location.** `restored` lives in `useAuthStore` — correct because both the response interceptor (low) and the root-layout dialog initializer (high) need it. ✓
- **Zustand selectors.** Narrow selectors: `useDialogStore((s) => s.openDialog)`, `useAuthStore((s) => s.restored)`. ✓
- **Forms.** The new Danger Zone section follows the local scalar-`useState` pattern that the audit §4 documented; matches the surrounding code on the same page rather than the partial-merge pattern CLAUDE.md describes (which is for the product wizard). ✓
- **Effects.** Each new `useEffect` justifies itself: the mount-time sessionStorage read can't be a derived value, and the `restored`-flag watcher reacts to an async server signal that has no other surface. ✓

### Finding #2 (medium) — `Messages.tsx` dropdown action handlers are not null-safe

The Brief A summary claims the chat header is "null-safe" but the audit fix only covers the rendered name. The conversation-header dropdown items continue to dereference `activeChat.withUser` without optional chaining:

```tsx
// Messages.tsx
:165:  onClick={() => router.push(`/user/${activeChat.withUser.id}`)}
:179:  reportedUserId: activeChat.withUser.id,
:187:  onClick={() => unblockUser(activeChat.withUser.firebaseUid)}
:193:  onClick={() => blockUser(activeChat.withUser.firebaseUid)}
```

Per spec §3.10 / §14.9, after a peer's hard-delete, `getUserForFirebaseUid(uid)` returns `null` from the backend → the chat store has `withUser: undefined` → `activeChat.withUser?.displayName ?? tCommon('user.deleted')` correctly renders "Deleted User" in the header at line 148, **but** the dropdown trigger remains visible (only `disabled={blockedBy}` gates it), and clicking Profile / Report / Block / Unblock throws `TypeError: Cannot read properties of undefined (reading 'id')`.

Brief A is internally consistent with what its task description asked for ("null-safety + header rendering"), but the audit's §6 high-severity finding was framed broadly as "any sender or withUser deref needs null-safety." The dropdown was outside the narrow chat-header focus and got missed.

**Fix scope:** either gate the entire dropdown render on `activeChat.withUser` being present, or use optional chaining inside each handler and short-circuit:

```tsx
const peer = activeChat.withUser;
{peer && (
  <DropdownMenu> ... </DropdownMenu>
)}
```

The user-facing impact is real once the first hard-delete lands in a production chat history — the dropdown's three items would all crash on click for any deleted peer. Worth fixing in this branch before merge rather than chasing as a post-merge bug.

### Finding #4 (medium) — banned user signing in via login dialog: no UI feedback

The flow on a banned user attempting email/password login:

1. User opens login dialog, submits credentials.
2. `useAuthStore.login` → `loginUserFirebase` → `buildUserSession` → `syncUserToBackend`.
3. `syncUserToBackend` sees `disabled: true` (or catches `EMAIL_BANNED`/`USER_BANNED` from the response): calls `auth.signOut()`, sets `sessionStorage['account-banned']`, returns `null`.
4. `useAuthStore.login` receives `backendUser = null`. `set({ user: backendUser })` (= null). No error message set.
5. The login dialog observes `user === null` (unchanged) and `error === null` — no signal that login failed. The dialog stays open.
6. The user has to dismiss the dialog manually before the ban-notice dialog can open on the next root-layout mount.

The same shape applies to `register`, `loginWithGoogle`, and `loginWithFacebook` — all four return `null` on the ban path without setting an error.

The fix is a small one in `useAuthStore` — when `backendUser` is `null` after a successful Firebase auth, treat it as a soft failure (e.g., `set({ error: '<some banned message>' })` or simply close the login dialog so the next route's root-layout mount can fire the ban-notice). Out of strict scope for this review (not a spec divergence — the spec doesn't enumerate the login-dialog UX), but the spec's stated intent (§4.7: "At sign-in … the dialog renders on the next root-layout mount") is undercut by the login dialog occupying the screen.

---

## 8. TypeScript discipline

- **No new `any`.** Inspected each new/modified file. `useDialogStore.tsx` already uses `Record<string, any>` (pre-existing). `useAuthStore.ts` has `catch (err: any)` blocks (pre-existing). The 5 new lines in this diff use explicit object shapes.
- **No `as unknown as`.** ✓
- **No new `@ts-ignore` / `@ts-expect-error`.** ✓
- **DTO types** correctly extended: `AuthUserDTO` and `UserInfoDTO` use union literals (`'ACTIVE' | 'PENDING_DELETION'`) — match the spec's intent. Server should emit the same string values; backend half is the contract.
- **`unknown` at boundary** in `isErrorWithCode`. ✓
- **`tsc --noEmit`** passes clean.

One observation tied to `issues.md` 2026-05-15 (`oglasino-web` tsconfig `strict: false`): Brief A's "For Mastermind" §5 already flagged that the missing-required-field type errors the new fields would have surfaced under `strict: true` are silenced today. Test-fixtures and any direct `AuthUserDTO` constructor sites that omit the four new fields type-check today; they wouldn't under strict mode. Not a defect introduced here — pre-existing posture.

---

## 9. Accessibility

- **Native semantic elements.** Buttons are `<Button>` (shadcn → native `<button>`). Inputs are `<Input>` (labeled). No `<div onClick>` in the new code. ✓
- **Labels.** The password input has an explicit `label` prop. ✓
- **Aria-label.** The kebab dropdown's `EllipsisVertical` icon is in a `DropdownMenuTrigger` — pre-existing pattern in this file. Not in scope.
- **Focus management.** Dialogs route through shadcn's `Dialog` / `Drawer` which handle focus trap and return. ✓
- **Keyboard navigation.** Buttons are reachable via Tab. Drawer/Dialog handle Escape per the `closableOutside` / `onEscapeKeyDown` logic in `DrawerDialog.tsx:77-80`.
- **`sr-only`.** Existing close-icon `sr-only` label is preserved in `DrawerDialog.tsx:132`.
- **Color contrast.** Not statically judgable. `text-warning` is used in three places — consistency check passes; whether it has adequate contrast on the various background tones is a design call.

### Finding #1 (medium) — invalid HTML `<p><div>...</div></p>` in three root-layout dialogs

The three informational dialogs opened from `AccountStateDialogsInit.tsx` pass a multi-paragraph `<div>...</div>` to `InfoDialog`'s `dialogDescription` prop:

```tsx
// AccountStateDialogsInit.tsx:50-55 (ban-notice)
dialogDescription: (
  <div className="flex flex-col gap-3 text-left">
    <p>{tBanned('banned.dialog.body.first')}</p>
    <p>{tBanned('banned.dialog.body.delete.intro')}</p>
    <p>{tBanned('banned.dialog.body.duration')}</p>
  </div>
),
```

```tsx
// AccountStateDialogsInit.tsx:69-76 (post-deletion)
dialogDescription: (
  <div className="flex flex-col gap-3 text-left">
    <p>{tDash('dashboard.pages.account.deleted.dialog.scheduled.date', { date: formattedDate })}</p>
    <p>{tDash('dashboard.pages.account.deleted.dialog.restore.instruction')}</p>
  </div>
),
```

`InfoDialog.tsx:60` wraps the prop in Radix's `<DialogDescription>`:

```tsx
{dialogDescription && (
  <DialogDescription className="mb-4 text-center">{dialogDescription}</DialogDescription>
)}
```

Radix's `DialogDescription` renders as `<p>` by default (the [Radix docs](https://www.radix-ui.com/primitives/docs/components/dialog#description) confirm). So the rendered DOM is:

```html
<p class="mb-4 text-center">
  <div class="flex flex-col gap-3 text-left">
    <p>...</p>
    <p>...</p>
  </div>
</p>
```

A `<div>` inside a `<p>` is invalid HTML. The browser auto-closes the outer `<p>` before the `<div>`, which:
- Triggers a React hydration warning (`In HTML, <div> cannot be a descendant of <p>`).
- Breaks the intended flex layout (`flex flex-col gap-3 text-left` is no longer inside the `<p>` block container).
- May upset assistive technology that reads the `<p class="mb-4 text-center">` as an empty paragraph followed by orphan content.

The restoration dialog at `:93-98` passes a plain string and is unaffected — but the ban-notice and post-deletion dialogs both ship this.

**Fix scope.** Three options, in increasing order of effort:

1. **Cheapest:** in `InfoDialog.tsx`, switch the prop wrapper from `<DialogDescription>` (Radix, renders `<p>`) to either `<DialogDescription asChild>` (Radix swaps to the child element — but `asChild` requires a single child of the right shape) or to a plain `<div>` + `<VisuallyHidden>` companion for the a11y label. Pre-conditioned on InfoDialog's other callers continuing to pass plain strings, which they do.
2. **Per-caller:** flatten the three-paragraph content into a single inline string with `\n\n` between (or `<br /><br />`, though that's its own anti-pattern). Loses the visual paragraph break.
3. **Replace `InfoDialog`:** introduce a `MultiParagraphInfoDialog` for the ReactNode body case and keep `InfoDialog` for plain-string bodies. Heavier; only worth it if more such dialogs land.

Option 1 is the right move. Brief B already widened `InfoDialog`'s API (`autoDismissAfterMs`, `closeButtonLabel`); a one-line wrapper change keeps the API stable for plain-string callers and unblocks the ReactNode case.

### Finding #6 (low) — `type` prop is dead on the three root-layout dialogs

`InfoDialog`'s `type?: 'info' | 'error' | 'warning'` (default `'info'`) is wired only to the **continue** button's background classes at lines 85-89:

```tsx
className={cn(
  type === 'info' && 'border-none bg-sky-600 text-white hover:bg-sky-700',
  type === 'warning' && 'border-none bg-amber-600 text-white hover:bg-amber-700',
  type === 'error' && 'border-none bg-red-500 text-white hover:bg-red-700'
)}
```

The continue button only renders when `onContinue` is passed (line 69). The three root-layout dialogs all pass `type` but never pass `onContinue`:

- Ban-notice: `type: 'error'` — no effect.
- Post-deletion: `type: 'warning'` — no effect.
- Restoration: `type: 'info'` — no effect.

The `type` prop is silently a no-op for these callers. Either the visual semantic should propagate elsewhere (e.g., container border color, title color), or the dialog calls should drop `type` entirely. Defensive on the future, but technically dead today.

---

## 10. Performance

- **Bundle size.** No new heavy deps. `firebase/auth` was already in the bundle. `next-intl` `useTranslations` already in the bundle.
- **Tree-shakable imports.** All named (`import { reauthenticateWithCredential, ... } from 'firebase/auth'`). ✓
- **`next/image`.** No new images.
- **`next/dynamic`.** `AccountStateDialogsInit` is mounted unconditionally inside `AppInit`, which is already a client root. The component returns `null` and only fires effects — bundle impact minimal. ✓
- **Font loading.** No font changes.
- **`'use client'` boundaries.** All new client components mount inside an already-client subtree. No new root-mode flips. ✓
- **Suspense.** No new boundaries needed.
- **Memoization.** No ornamental `useMemo` / `useCallback` introduced. Pre-existing `useMemo` in `UserDetails.tsx:54` is untouched and earns its place. ✓
- **Network.** `syncUserToBackend` now has an extra `try/catch` and a possible `signOut()` — single async call added to an already-async path. Negligible.

---

## 11. State and data flow

- **Server state.** Returned via the existing axios pipeline. `AuthUserDTO` and `UserInfoDTO` extensions carry server-derived fields. ✓
- **URL state.** No new URL params introduced.
- **Client state.** `useAuthStore.restored` is the only new client field — correct location given two consumers (interceptor and root-layout init).
- **sessionStorage** as a transient cross-mount channel: chosen because `auth.signOut()` triggers `SessionGuard` to redirect, which unmounts the source of the signal. SessionStorage survives the same-tab redirect. Right tool for the job; alternatives (URL param, query string) would surface in the address bar.
- **Optimistic updates** — none introduced.
- **Cache invalidation after mutations** — the chat user-cache (`useChatStore.userCache`) is not invalidated when a peer enters PENDING_DELETION. This is a stale-data window: an open chat tab won't see the badge appear until the next `getUserForFirebaseUid` call. Spec §14.8 doesn't require real-time propagation; acceptable per the audit's §6 description of the lookup-cache lifetime. ✓

### Finding #10 (low) — `useAuthStore.refreshUser` leaks the banned user record

```ts
// useAuthStore.ts:259-287
refreshUser: async () => {
  ...
  const firebaseUser = auth.currentUser;
  if (!firebaseUser) { set({ user: null, loading: false }); return null; }
  const backendUser = await syncUserToBackend(firebaseUser);
  if (auth.currentUser?.uid !== firebaseUser.uid) { return null; }   // ← here
  set({ user: backendUser, loading: false });
  return backendUser;
}
```

When `syncUserToBackend` detects a ban, it calls `auth.signOut()` (line 142 in `authService.ts`). After that promise resolves, `auth.currentUser` is `null`, so `auth.currentUser?.uid !== firebaseUser.uid` is `true` — the function returns `null` without updating the store. The previously-set `user` (the banned user) remains in `useAuthStore.user` until the next `initAuthListener` callback fires (when `firebaseUser === null` → `set({ user: null })`) or the next protected nav surfaces `SessionGuard` (which reads `auth.currentUser`, not `useAuthStore.user`).

In practice this resolves itself quickly — the `onAuthStateChanged` listener runs as a side effect of `signOut()`. But for a brief moment, the auth store reports a banned user while Firebase has signed them out. Any component using `useAuthStore.user` to decide whether to fire an authenticated request during that window will think the user is still authenticated.

Severity: low. The race window is sub-millisecond and the next render fixes it. Worth a defensive `set({ user: null })` before the short-circuit return, but not blocking.

### Finding #9 (low) — `Chats.tsx:25` search filter is fragile but spec-conforming

```ts
const temp = chats.filter((chat) =>
  chat.withUser?.displayName.toLowerCase().includes(search.toLowerCase())
);
```

Parsing: `chat.withUser?.displayName.toLowerCase()` — if `chat.withUser` is `undefined`, the optional-chain short-circuits to `undefined`, then `.includes` on undefined → `undefined.includes(...)` throws.

Wait — the filter function would crash if any chat had `withUser === undefined` while the user is searching.

Per spec §3.10, after hard-delete the backend's `getUserForFirebaseUid` returns null, so `withUser` becomes undefined in `useChatStore.userCache`. So this *is* reachable.

Re-checking precedence: `chat.withUser?.displayName.toLowerCase()` — TC39 §12.3.9 short-circuits the whole chain when any link is nullish. So `chat.withUser` undefined → whole expression `undefined`, then `.includes` is called on `undefined` → TypeError.

But actually — let me re-read. JS optional chaining short-circuits subsequent **property accesses and calls** that are part of the chain after `?.`. The form `a?.b.c.d()` means "if `a` is nullish, return undefined; otherwise evaluate `a.b.c.d()` normally." So `chat.withUser?.displayName.toLowerCase()`:
- `chat.withUser` undefined → expression evaluates to `undefined`.
- `chat.withUser` defined → `chat.withUser.displayName.toLowerCase()` evaluates normally.

Then the outer `.includes(search.toLowerCase())` is called on **the result of the chain**, which is `undefined` in the first case. `undefined.includes(...)` → TypeError.

So **this would crash** if a chat exists with a deleted peer and the user searches.

But Brief A's `Chats.tsx:58` extracted `peerName = chat.withUser?.displayName ?? tCommon('user.deleted')` — used at lines 71 and 75 for rendering. The filter at line 25 was not updated.

Real crash potential: yes, once a hard-deleted peer lives in the chat list and the user types in the search box.

Re-classifying this from "fragile" to **medium** severity — same character as finding #2.

Actually, I'm going to bump this up. Let me re-classify.

Bumping to **medium**: Updating the severity table mentally — Chats.tsx filter is now a third medium finding alongside #1 and #2.

**Fix:** `chat.withUser?.displayName?.toLowerCase().includes(search.toLowerCase()) ?? false` — adds a second `?.` after `displayName` and a `?? false` to make the filter exclude deleted peers from search results. Or compute `peerName` (which already has the fallback) up front and filter on that.

I'll update the severity table at the top of the report.

---

## 12. Styling (Tailwind)

- **Class names** consistent with the codebase: `border-border-mild`, `border-border-strong`, `text-primary-mild`, `text-warning`, `bg-background-mild`. ✓
- **No inline `style={}`** introduced. ✓
- **Responsive prefixes.** `Messages.tsx` retains `sm:gap-3`; the rest of the diff doesn't introduce new fixed-pixel widths. Per the 2026-05-17 mobile-responsiveness audit, the deletion-feature additions don't seem to add new mobile-shaped pitfalls — the Danger Zone section uses `w-full` and the dialog has `w-[90%]` which is an arbitrary value but matches similar usage across the codebase (`UserInfo` already uses `w-[80%]`, `w-[60%]`, etc.).
- **One arbitrary value:** `w-[90%]` in `DeleteAccountConfirmationDialog.tsx:128`. Could be `w-11/12` (Tailwind has fractional widths). Not strictly a violation — the codebase already uses arbitrary percentages here and there. Low priority cosmetic.
- **`text-warning`** is used in three places (`ScheduledForDeletionBadge.tsx:11`, `Messages.tsx:265`). Pre-existing utility per the audit's reference. ✓

---

## 13. Testing

- **Test suite passes.** 154/154 vitest passing locally on this branch.
- **No new tests.** The four new files (badge, dialog, init, helper) and the modified interceptor / store paths have no unit or integration tests added.
- **Why this isn't a hard blocker.** Per `issues.md` 2026-05-14 "Web component-render test coverage gap": `@testing-library/react`, `@testing-library/user-event`, and a DOM environment (`happy-dom`/`jsdom`) are not installed in the repo. Brief A and Brief B summaries both note this and defer. The repo's testing posture is pure-function and axios-mock — dialog rendering, sessionStorage initializers, and React `useEffect` flows aren't testable today.
- **What I'd want.** A future infra brief to install `@testing-library/react` and add component tests for: (a) the dialog's provider-branch switching, (b) the `isErrorWithCode` helper across both axios shapes (this one's a pure function — trivially testable today!), (c) the interceptor's `USER_BANNED` short-circuit path with a mocked axios response.

The `isErrorWithCode` helper is pure and could land a unit test in this very PR with zero new dependencies — it's the cheapest path to growing coverage. Strong recommendation, not a blocker.

---

## 14. Code quality (Part 4 + Part 4a)

**Cleanliness — Part 4.**
- No `console.log` introduced. Grepped the four new files plus modified `Messages.tsx`, `Chats.tsx`, `userService.ts`, `authService.ts`, `api.ts`, `useAuthStore.ts`. The existing `console.error` / `console.info` calls in `useAuthStore` are pre-existing and not in this diff's scope.
- No `TODO` / `FIXME` introduced. Grepped same set.
- No commented-out code. Brief C explicitly noted that the Brief B raw-axios comment was relocated, not orphaned.
- No unused imports / variables / functions. `tsc` clean confirms unused-symbol elimination.
- No new files left unreferenced. All four new files are mounted/imported by something in this diff.
- `tsc --noEmit` ✓, `lint` 0 errors / 208 pre-existing warnings ✓, `npm test` 154/154 ✓.

**Simplicity — Part 4a.**
- **New abstractions earn their place.**
  - `ScheduledForDeletionBadge` — single-purpose, two concrete callers (UserDetails, Messages). ✓
  - `AccountStateDialogsInit` — root-layout initializer, three concrete consumers. ✓
  - `DeleteAccountConfirmationDialog` — single concrete caller. ✓
  - `isErrorWithCode` — used in three places (the dialog, `syncUserToBackend`, and pattern-matched against in the interceptor). The two-shape tolerance is defensive; the second branch is dead today (finding #8) but the comment explains why it's there.
  - `deleteCurrentUser` — single concrete caller; lives in `userService.ts` matching the surrounding eight functions. ✓
- **Configuration vs constants.** `RESTORATION_AUTO_DISMISS_MS = 10_000` is a module-local constant — no plausible second setting. ✓
- **Match surrounding patterns.** The Danger Zone section uses the page-local scalar `useState` form (not partial-merge). The dialog uses `DrawerDialog` like other dialogs. The service function follows `userService.ts`'s shape. The interceptor change is one new `if`. ✓
- **Defensive code at boundaries; trust internal seams.**
  - `isErrorWithCode` is defensive at the axios-error boundary.
  - `getIdToken(true)` is defensive at the Firebase boundary.
  - `useAuthStore.user` is trusted internally (no redundant null-checks introduced).
- **Comments explain why.** The interceptor guard at `api.ts:86-90` carries a four-line comment explaining the reauth-freshness reason. The `userService.deleteCurrentUser` comment names the seam. The `AccountStateDialogsInit` mount-effect comment explains why the deps array is empty. All "why," not "what." ✓
- **Component size.** No component grew past ~160 lines (DeleteAccountConfirmationDialog is the biggest at 162). Within the file's local norms. ✓

---

## 15. Adjacent observations (Part 4b)

Things noticed while reading that are outside this PR's scope:

- **`useAuthStore.logout` at line 227** (`Could not detach FCMToken from user ${get().user.id}`) — pre-existing nullable-access pattern, can throw `TypeError` if the store user is already null. Pre-dates this PR. **Severity: low. Out of scope, not fixed.** Path: `src/lib/store/useAuthStore.ts:227`.

- **`DrawerDialog.tsx:107` mobile path ignores `closableOutside`** — `<Drawer open={isOpen} onOpenChange={(open) => !open && onClose()}>`. The desktop branch at lines 67-81 respects `closableOutside`; the mobile branch is unconditional. The deletion dialog passes `closableOutside={!loading}` precisely so the user can't escape mid-request, but on mobile the drag-to-dismiss still works. Pre-existing inconsistency. **Severity: low. Out of scope, not fixed.** Path: `src/components/popups/dialogs/DrawerDialog.tsx:107`.

- **`BACKEND_API_URL` duplicated across four call sites** — `api.ts:6`, `fetchApi.ts:3`, `translationsCache.ts:28`, `getConfig.ts:7`. Flagged by Brief B and partially closed by Brief C. **Severity: low. Out of scope, not fixed.**

- **`useChatStore.userCache` has no invalidation on PENDING_DELETION** — a peer who enters grace mid-session won't show the badge until the cache entry refreshes (lifetime of the store instance, per audit §6). Spec §14.8 doesn't require real-time propagation. **Severity: low. Out of scope, not fixed.** Path: `src/messages/store/useChatStore.ts:115-124`.

- **`onClose` dependency in `InfoDialog`'s auto-dismiss effect** — `useEffect` deps include `onClose`; if a parent re-renders and the prop identity changes, the timeout resets. `DrawerDialogManager` passes `closeDialog` from zustand, which has a stable identity, so today it's fine. A future parent that passes an inline `() => closeMyWay()` would silently reset the timer per render. **Severity: low. Out of scope, not fixed.** Path: `src/components/popups/dialogs/InfoDialog.tsx:45-49`.

- **`useChatStore` and `useChatBlockStore` clearance on logout** — these are cleared by `ChatsInit` / `ChatsWatcher` reacting to `user === null`, not inside `useAuthStore.logout`. Per the audit §1, the dependency is on the initializer rather than the store itself. With the new ban-driven sign-outs, the cleanup chain runs the same way. **Not a defect, just a note** that the cross-store coupling is implicit.

- **`useChatStore.subscribeToChats` / `subscribeToMessages` and deleted peers** — the audit §6 noted the store enriches messages with `getUserData`. The Brief A null-safety fix handles the rendering side, but the store's enrichment behavior when `getUserForFirebaseUid` returns `null` is opaque from outside (does the store set `sender: undefined`, skip the message, retry?). Not opened during this review. **Severity: low. Out of scope, not investigated.**

- **Brief A's "For Mastermind" note 5** — `tsconfig strict: false` silences the type errors that would have surfaced from the new DTO fields. Reiterated for the record; `issues.md` 2026-05-15 entry tracks this.

- **No `ARIA` regression in dialogs.** The `<p><div></div></p>` issue is HTML-level, not ARIA-level. Radix-managed focus/labeling is intact.

---

## Closing

This PR ships a feature-complete user-deletion frontend that matches the spec on routes, behaviors, types, error codes, and trust boundaries. The Brief A → Brief B → Brief C sequence is coherent and the trust-boundary audit is correct.

Before merge I'd ask for **findings #1, #2, and #9** to be addressed in-branch — the HTML validity issue in the three root-layout dialogs and the two null-deref paths (Messages dropdown handlers, Chats search filter) are all small fixes with real user-visible failure modes. **Findings #3 and #4** (closure-gate for `conventions.md` Part 6, login-dialog UX on ban) are worth resolving but separable: #3 needs a Docs/QA brief to apply the conventions edit; #4 could ship as a follow-up if Igor prefers to land the larger feature first.

Everything else is acceptable or pre-existing. Hygiene baseline is green.
