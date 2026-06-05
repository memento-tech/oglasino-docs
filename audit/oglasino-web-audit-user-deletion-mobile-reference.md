# Audit — oglasino-web — user-deletion mobile-adoption reference

**Branch:** `dev`
**HEAD:** `90c74ab`
**Type:** read-only reference audit. No code changed, nothing staged, nothing committed.
**Method:** code read directly; quotes are from the on-disk files at the HEAD above. Where the code diverges from a named doc section it is flagged **CODE SAYS X / DOC SAYS Y**. The code is ground truth.

**Reference docs consulted (read-only, sibling repo):** `oglasino-docs/features/user-deletion.md` (the feature spec, §14 = web UX), `oglasino-docs/features/user-deletion-auth-contract.md` (the auth lifecycle contract, clauses C-1…C-10).

---

## Q1 — Danger Zone

**File:** `app/[locale]/owner/user/page.tsx` — the page's default export is the component `UserInfo` (the account-settings screen). Danger Zone is the last block, **lines 396–423**.

CODE matches DOC §14.1 (which names `app/[locale]/owner/user/page.tsx`). Note the component is named `UserInfo`, not a separate `UserSettingsForm` — the whole settings screen plus the Danger Zone live in this one client component.

Exact rendered structure (lines 396–423):

```tsx
<div className="mt-6 flex w-full flex-col items-center gap-3 rounded-md border border-red-500 p-4 md:items-start">
  <div className="text-destructive flex items-center gap-2">
    <AlertTriangle className="size-5" color="red" />
    <h2 className="text-lg font-semibold">{tDash('danger.zone.label')}</h2>
  </div>
  <h3 className="font-medium">{tDash('delete.account.title')}</h3>
  <ul className="text-primary-mild list-disc pl-6 text-sm">
    <li>{tDash('delete.account.bullet.listings')}</li>
    <li>{tDash('delete.account.bullet.profile.badge')}</li>
    <li>{tDash('delete.account.bullet.messaging')}</li>
    <li>{tDash('delete.account.bullet.signout')}</li>
  </ul>
  <p className="text-primary-mild text-center text-sm italic md:text-start">
    {tDash('delete.account.restore.note')}
  </p>
  <div className="self-center md:self-end">
    <Button
      variant="outline"
      size="lg"
      className="border-red-600 text-red-600"
      disabled={loading}
      onClick={() =>
        openDialog(DialogId.DELETE_ACCOUNT_CONFIRMATION_DIALOG, { userId: user.id })
      }>
      {tButtons('delete.account.label')}
    </Button>
  </div>
</div>
```

- **Heading:** `<h2>` `danger.zone.label` + a red `AlertTriangle` (lucide) icon (`page.tsx:397–399`).
- **Sub-heading:** `<h3>` `delete.account.title` (`:401`).
- **Bullet list:** four `<li>` — `delete.account.bullet.listings`, `.bullet.profile.badge`, `.bullet.messaging`, `.bullet.signout` (`:402–407`).
- **Restore note:** `<p>` `delete.account.restore.note` (`:408–410`).
- **Button:** opens `DialogId.DELETE_ACCOUNT_CONFIRMATION_DIALOG` with prop `{ userId: user.id }` (`:417–419`).

**Translation keys & namespaces:**
- `danger.zone.label`, `delete.account.title`, `delete.account.bullet.listings`, `delete.account.bullet.profile.badge`, `delete.account.bullet.messaging`, `delete.account.bullet.signout`, `delete.account.restore.note` — all via `tDash` = **`DASHBOARD_PAGES`** (`page.tsx:33`).
- `delete.account.label` — via `tButtons` = **`BUTTONS`** (`page.tsx:35`).

**Destructive-emphasis styling:** container `rounded-md border border-red-500 p-4`; heading text `text-destructive`; the trigger `Button` is `variant="outline" size="lg"` with `className="border-red-600 text-red-600"`. (The dialog itself adds `border-destructive/40` — see Q2.) No filled/solid destructive button — the destructive cue is the red border + red text, consistent across the zone and the dialog.

---

## Q2 — Delete-confirmation dialog (core parity reference)

**File:** `src/components/popups/dialogs/DeleteAccountConfirmationDialog.tsx` (197 lines).

**Registration / open mechanism (a dialog registry, not a DialogManager-per-component):**
- Dialog id is the enum `DialogId.DELETE_ACCOUNT_CONFIRMATION_DIALOG = 'deleteAccountConfirmationDialog'` (`src/components/popups/dialogRegistry.ts:27`).
- The component is registered in the single-slot manager `src/components/popups/DialogManager.tsx` (default export `DrawerDialogManager`): import at `:10`, registry entry `deleteAccountConfirmationDialog: DeleteAccountConfirmationDialog` at `:63`. The manager renders exactly one dialog: `const SpecificDialog = DIALOGS[currentDialogId]` → `<SpecificDialog isOpen onClose={closeDialog} {...dialogProps} />` (`DialogManager.tsx:71–80`).
- Open/close state lives in a Zustand store `src/components/popups/store/useDialogStore.tsx` — `openDialog(id, props)` sets `{ currentDialogId, dialogProps }`; `closeDialog()` resets both (single-slot, one dialog at a time). The Danger Zone calls `openDialog(DialogId.DELETE_ACCOUNT_CONFIRMATION_DIALOG, { userId: user.id })`.

> DialogManager is the **dialog registry** the spec §4.1/§14.1 calls "registered via `DialogManager`." Quoted above.

**Provider detection** (`DeleteAccountConfirmationDialog.tsx:26–33`, used at `:55`):

```ts
function resolveProvider(): Provider {
  const providerId = auth.currentUser?.providerData[0]?.providerId;
  if (!providerId) return 'password';
  if (providerId === 'password' || providerId === 'google.com' || providerId === 'facebook.com') {
    return providerId;
  }
  return 'unsupported';
}
```

So web detects provider via **`auth.currentUser.providerData[0].providerId`** (the Firebase client `User`), NOT via `AuthUserDTO.providerId`. `'unsupported'` disables the delete button and shows `system.error` as the description. (Mobile's Google reauth differs — GoogleSignin, not popup — per the brief; this is web's exact shape.)

**Full as-built submit sequence** — `handleDelete` (`:57–143`), in order. PORTABLE/WEB-ONLY labels per step:

Phase A — reauth (`:57–92`):
1. `const currentUser = auth.currentUser; if (!currentUser) { setErrorMessage(reauth.required); return; }` — **PORTABLE** (mobile has a current Firebase user too).
2. `setLoading(true); setErrorMessage('')` (`:64–65`) — **PORTABLE** (UI state).
3. Reauth by provider (`:67–92`):
   - `password`: requires the password field, then `EmailAuthProvider.credential(currentUser.email ?? '', password)` + `await reauthenticateWithCredential(currentUser, credential)` (`:74–75`) — **PORTABLE** (RN Firebase supports `reauthenticateWithCredential`).
   - `google.com`: `await reauthenticateWithPopup(currentUser, googleProvider)` (`:77`) — **WEB-ONLY shape** (mobile uses the GoogleSignin native flow, not a popup; the *intent* — reauth before delete — is portable).
   - `facebook.com`: `await reauthenticateWithPopup(currentUser, facebookProvider)` (`:79`) — **WEB-ONLY shape** (same note; Facebook is "future enablement").
   - else `'unsupported'`: set `reauth.required`, stop — **PORTABLE** (defensive).
   - `catch { setErrorMessage(reauth.required); setLoading(false); return; }` (`:85–92`) — reauth failure keeps the dialog open; Firebase's code is not surfaced verbatim — **PORTABLE**.

Phase B — deletion + teardown (`:94–143`):
4. **`useAuthStore.getState().setDeletionInFlight(true)`** (`:99`) — set **immediately before** `getIdToken(true)` per contract **C-6**. **PORTABLE in concept**, but only matters on web because it gates `UseTokenRefresh`'s cookie/sync listener (see Q12); mobile has its own token-refresh listener if it mirrors C-6.
5. `const freshToken = await currentUser.getIdToken(true)` (`:103`) — forced refresh so `auth_time` is recent — **PORTABLE** (mobile mints a fresh token the same way).
6. `const { scheduledDeletionAt } = await deleteCurrentUser(freshToken)` (`:105`) — the deletion submit with the fresh token (see Q3) — **PORTABLE** (same backend endpoint + header-override pattern).
7. `revalidateUserCache(userId)` (`:106`) — Next.js `revalidateTag`/`updateTag` server action — **WEB-ONLY** (mobile has no Next data cache).
8. **post-success flag write:** `useAuthStore.getState().setAccountJustDeleted({ scheduledDeletionAt })` (`:111`) — the **Zustand store flag** (NOT sessionStorage — see Q4). Set **before** sign-out so the post-deletion dialog opens after navigation — **PORTABLE in concept** (mobile mirrors with a store flag; the exact "before signOut" ordering matters).
9. `await auth.signOut()` (`:112`) — **PORTABLE**.
10. **`await clearFirebaseTokenCookie()`** wrapped in a local `try/catch` that swallows-and-logs (`:117–124`) — contract **C-5**. **WEB-ONLY** (clears the httpOnly `firebase_token` cookie; mobile's token is in-memory, has no SSR cookie). The swallow-and-continue behavior on cookie-clear failure is the C-5 requirement.
11. No explicit `onClose()` — comment (`:125–128`) explains `setAccountJustDeleted` + the subsequent `openDialog(INFO_DIALOG)` from `AccountStateDialogsInit` replaces this dialog via the single-slot store; a manual close would race. — **PORTABLE in concept** (single-slot dialog handoff).
12. `router.replace(\`/${locale}\`)` (`:129`) — **WEB-ONLY shape** (Next router + locale path; mobile uses expo-router/navigation).
13. `catch (error)` (`:130–138`): `isErrorWithCode(error, 'REAUTH_REQUIRED')` → `reauth.required`; `isErrorWithCode(error, 'USER_LOCKED_FROM_DELETION')` → `user.locked.from.deletion` + `setLocked(true)`; else `system.error`. — **PORTABLE** (error-code discrimination).
14. `finally` (`:139–142`): `setDeletionInFlight(false)` + `setLoading(false)`. — **PORTABLE** (flag cleared in finally per C-6).

> **C-5/C-6 ordering confirmed on disk** exactly as the contract specifies: `setDeletionInFlight(true)` is immediately before `getIdToken(true)` (`:99` then `:103`); the success path is submit → `setAccountJustDeleted` → `await auth.signOut()` → `await clearFirebaseTokenCookie()` (swallowed) → `router.replace`; `setDeletionInFlight(false)` is in `finally`.

**Error handling for 403 / 5xx:**
- 403 `REAUTH_REQUIRED` → `tErrors('reauth.required')` (`:131–132`).
- 403 `USER_LOCKED_FROM_DELETION` → `tErrors('user.locked.from.deletion')` and `setLocked(true)` (which disables the password input and the delete button) (`:133–135`).
- 5xx / anything else → `tErrors('system.error')` (`:136–137`). The code discriminates on the **error code** via `isErrorWithCode`, not on HTTP status.
- **403 `USER_BANNED` is NOT handled in this dialog** — it is caught by the global axios response interceptor (Q10) before the dialog's catch runs (that path returns a never-resolving promise, so the dialog's `catch`/`finally` do not fire for it). Worth noting for mobile: the dialog relies on the global interceptor for the banned case.

**Body/labels:**
- Title `tDialog('delete.account.dialog.title')` (DIALOG); description `delete.account.dialog.body.password` or `...body.google` depending on provider (`:145–148`, `:156`); `system.error` if `unsupported`.
- Password field label `delete.account.dialog.password.label` (DIALOG) (`:164`).
- Cancel button `button.cancel.label` (DIALOG) (`:177`); confirm button `delete.account.label` (password) or `reauthenticate.and.delete.label` (google/facebook), both **BUTTONS** (`:187–189`).
- Dialog styling: rendered through `DrawerDialog` with `className="border-destructive/40"`, `addCloseButton={false}`, `closableOutside={!loading}` (`:150–159`).

---

## Q3 — The deletion service call

**File:** `src/lib/service/reactCalls/userService.ts`, function `deleteCurrentUser` (`:102–112`):

```ts
export const deleteCurrentUser = async (
  freshToken: string
): Promise<{ scheduledDeletionAt: string }> => {
  const res = await BACKEND_API.post<{ scheduledDeletionAt: string }>(
    '/secure/user/me/delete',
    {},
    { headers: { Authorization: 'Bearer ' + freshToken } }
  );

  return res.data;
};
```

- **Signature:** `deleteCurrentUser(freshToken: string): Promise<{ scheduledDeletionAt: string }>`.
- **Endpoint:** `POST /secure/user/me/delete`, empty body `{}`.
- **Fresh-token header override:** passed explicitly as `headers: { Authorization: 'Bearer ' + freshToken }`. The request interceptor (Q10) honors a caller-supplied `Authorization` and skips the cached-token attach (`api.ts:111–113`) — this is the documented bypass so the backend reads a recent `auth_time`.
- **Success:** returns `res.data` = `{ scheduledDeletionAt }`.
- **Error:** **no try/catch** — errors propagate to the dialog's `catch`, which discriminates `REAUTH_REQUIRED` / `USER_LOCKED_FROM_DELETION` / else. (The function comment at `:95–101` states this explicitly.)

> Matches DOC spec §11/§14.3 (`axios.post('/secure/user/me/delete', {}, { headers: { Authorization: 'Bearer ' + freshToken } })`). The contract names it `userService.deleteCurrentUser(freshToken)` — confirmed.

---

## Q4 — Post-deletion trigger mechanism (resolves the doc conflict)

**RESOLVED: the on-disk mechanism is a Zustand `accountJustDeleted` flag on `useAuthStore`, NOT `sessionStorage['account-just-deleted']`.**

- Write side: `DeleteAccountConfirmationDialog.tsx:111` — `useAuthStore.getState().setAccountJustDeleted({ scheduledDeletionAt })`.
- Field: `useAuthStore.ts:93` — `accountJustDeleted: { scheduledDeletionAt: string } | null;` (setter `setAccountJustDeleted`, `:109`, `:149`).
- Read side: `AccountStateDialogsInit.tsx:41` subscribes `accountJustDeleted`; the effect (`:79–104`) reads `accountJustDeleted.scheduledDeletionAt`, formats it, opens `INFO_DIALOG`, then clears via `setAccountJustDeleted(null)` in the same effect.

The scheduled date flows: backend returns `scheduledDeletionAt` → dialog destructures it from `deleteCurrentUser` → `setAccountJustDeleted({ scheduledDeletionAt })` → `AccountStateDialogsInit` formats it with `formatDeletionDate(iso, formatterLocale)` (`:15–28`, `:81`) and interpolates into `account.deleted.dialog.scheduled.date` (`:90–92`).

There is **no** `sessionStorage` usage anywhere in the deletion/ban path (grep-confirmed: the only matches for these symbols are the store, the dialog, the init component, the interceptor, and the types).

**CODE-vs-DOC divergence (the conflict the brief flagged):**
- **DOC `user-deletion.md` §14.4 (line ~1272) — MATCHES code:** explicitly documents the store-flag mechanism (`AccountStateDialogsInit` subscribing to `accountJustDeleted`/`restored`/`accountBanned`) and states it "replaces an earlier sessionStorage-based mechanism."
- **DOC `user-deletion.md` §14.3 step 4 (line 1262) and §4.1 step 4 (line 240) — STALE:** still say `sessionStorage.setItem('account-just-deleted', scheduledDeletionAt)`.
- **DOC auth-contract C-5 step 2 (line 109) and the §5 timeline (lines 49, 173, 181) — STALE:** still say `sessionStorage.setItem('account-just-deleted')` / "reads sessionStorage."
- The auth-contract's own §14.4 cross-reference and spec §14.4 are the authoritative as-built description.

> **Mobile mirrors the Zustand-flag mechanism, not sessionStorage.** (Mobile has no sessionStorage anyway; this removes any temptation to emulate one.) The same divergence applies to the ban path — see Q8.

---

## Q5 — `useAuthStore` deletion/ban fields

**File:** `src/lib/store/useAuthStore.ts`. Exact field + setter names and types:

| Field | Type | Setter | Decl lines |
| --- | --- | --- | --- |
| `restored` | `boolean` (default `false`) | `setRestored(value: boolean)` | field `:86`, default `:135`, setter `:108`/`:147` |
| `accountJustDeleted` (the post-deletion field, per Q4) | `{ scheduledDeletionAt: string } \| null` (default `null`) | `setAccountJustDeleted(value)` | field `:93`, default `:137`, setter `:109`/`:149` |
| `accountBanned` | `{ reason: string \| null } \| null` (default `null`) | `setAccountBanned(value)` | field `:98`, default `:139`, setter `:110`/`:151` |
| `deletionInFlight` | `boolean` (default `false`) | `setDeletionInFlight(value: boolean)` | field `:104`, default `:141`, setter `:111`/`:153` |

Notes for mobile parity:
- `restored` is a **plain boolean** (no payload).
- `accountJustDeleted` carries the scheduled date as `{ scheduledDeletionAt: string }`.
- `accountBanned` carries `{ reason: string | null }` — but every web write currently passes `{ reason: null }` (the backend does not send a ban reason on these paths today; see Q8).
- All four are simple `set({...})` actions (`:147–153`), mirroring each other — the contract C-4/Q-4 "mirrors `restored`/`setRestored`" note holds.

---

## Q6 — `AccountStateDialogsInit`

**File:** `src/components/client/initializers/AccountStateDialogsInit.tsx` (138 lines). Default export `AccountStateDialogsInit`, returns `null` (pure effect host).

Subscribes (`:39–44`): `restored`/`setRestored`, `accountJustDeleted`/`setAccountJustDeleted`, `accountBanned`/`setAccountBanned`. Opens dialogs via `useDialogStore.openDialog` (`:37`).

**Reactive-open-then-clear-in-same-effect pattern** — three `useEffect`s, each guarded by `if (!flag) return;`, opens `DialogId.INFO_DIALOG`, then clears its flag at the end of the same effect:

1. **Ban-notice** (`:51–69`):
   - `dialogTitle: tDialog('banned.dialog.title')`
   - body (JSX, three `<p>`): `tDialog('banned.dialog.body.first')`, `tDialog('banned.dialog.body.delete.intro')`, `tDialog('banned.dialog.body.duration')`
   - `closeButtonLabel: tButtons('banned.go.home.label')`
   - `onClose`: `useDialogStore.getState().closeDialog()` then `router.replace(\`/${routingLocale}\`)`
   - then `setAccountBanned(null)` (`:68`).

2. **Post-deletion** (`:79–104`):
   - `dialogTitle: tDialog('account.deleted.dialog.title')`
   - body (two `<p>`): `tDialog('account.deleted.dialog.scheduled.date', { date: formattedDate })`, `tDialog('account.deleted.dialog.restore.instruction')`
   - `closeButtonLabel: tButtons('account.deleted.acknowledge.label')`
   - `onClose`: `closeDialog()` then `router.refresh()`
   - then `setAccountJustDeleted(null)` (`:103`).
   - Date formatting via `formatDeletionDate(iso, formatterLocale)` (`:15–28`); `cnr` falls back to `sr` locale for `toLocaleDateString`.

3. **Restoration** (`:118–134`):
   - `dialogTitle: tDialog('account.restored.title')`
   - `dialogDescription: tDialog('account.restored.subtitle')`
   - `autoDismissAfterMs: RESTORATION_AUTO_DISMISS_MS` (= `10_000`, `:13`)
   - `onClose` (async): `closeDialog()`, then if `user?.id` `await revalidateUserCache(userId)`, then `router.refresh()`
   - then `setRestored(false)` (`:133`).

All three reuse the shared `INFO_DIALOG` (DialogId at `dialogRegistry.ts:7`) with per-call title/description/onClose props — no dedicated PostDeletion/Restoration/BanNotice components exist (the brief's working names map to these three `INFO_DIALOG` invocations).

Namespaces: titles/bodies via `tDialog` = **DIALOG** (`:34`); button labels via `tButtons` = **BUTTONS** (`:35`).

---

## Q7 — Restoration dialog + interceptor read

**Interceptor read of `X-Account-Restored`:** `src/lib/config/api.ts:26–32` (the success branch of the response interceptor):

```ts
instance.interceptors.response.use(
  (response) => {
    if (response.headers['x-account-restored'] === 'true') {
      useAuthStore.getState().setRestored(true);
    }
    return response;
  },
  ...
```

Header name is read lowercase (`x-account-restored`) and compared to the string `'true'`; on match it flips the store via `setRestored(true)`. (Matches contract C-9 step 7 / spec §14.5.)

**Restoration dialog content + keys:** see Q6 item 3 — `account.restored.title` / `account.restored.subtitle` (DIALOG).

**10-second auto-dismiss:** implemented declaratively via the `autoDismissAfterMs` prop passed to `INFO_DIALOG` (`AccountStateDialogsInit.tsx:123`), value `RESTORATION_AUTO_DISMISS_MS = 10_000` (`:13`). The actual timer lives inside `InfoDialog` (the registry-mapped component); the init component only supplies the duration. On dismissal (auto or user), `onClose` busts the Next.js Data Cache for the user (`revalidateUserCache(userId)`) and `router.refresh()`s so the `PENDING_DELETION` badge clears — this cache-bust is **WEB-ONLY** (mobile has no Next Data Cache).

---

## Q8 — Ban-notice dialog + `isErrorWithCode`

**Ban-notice dialog content + keys (the four `banned.dialog.*` in DIALOG):** Q6 item 1 — `banned.dialog.title`, `banned.dialog.body.first`, `banned.dialog.body.delete.intro`, `banned.dialog.body.duration`, plus button `banned.go.home.label` (BUTTONS).

**Both trigger paths (quoted):**

1. **`syncUserToBackend` disabled / banned branch** — `src/lib/service/reactCalls/authService.ts:130–164`:
   - Success-response disabled branch (`:146–150`):
     ```ts
     if (userData.disabled) {
       await auth.signOut();
       useAuthStore.getState().setAccountBanned({ reason: null });
       return { user: null, wasRegister: false };
     }
     ```
   - Rejection branch for `EMAIL_BANNED` / `USER_BANNED` (`:156–162`):
     ```ts
     } catch (error) {
       if (isErrorWithCode(error, 'EMAIL_BANNED') || isErrorWithCode(error, 'USER_BANNED')) {
         await auth.signOut();
         useAuthStore.getState().setAccountBanned({ reason: null });
         return { user: null, wasRegister: false };
       }
       throw error;
     }
     ```
   This is the sign-in / re-registration path. It sets the **store flag**, not `sessionStorage`.

2. **403 `USER_BANNED` interceptor branch** — `src/lib/config/api.ts:51–66` (response-error interceptor):
   ```ts
   if (error.response.status === 403) {
     const errors = (error.response.data as { errors?: Array<{ code?: string }> } | undefined)?.errors;
     if (errors?.[0]?.code === 'USER_BANNED') {
       auth.signOut();
       useAuthStore.getState().setAccountBanned({ reason: null });
       return new Promise(() => {}); // never-resolves
     }
   }
   ```

**Two additional ban surfaces beyond the brief's two** (worth flagging for mobile):
- **Sign-in `auth/user-disabled`** in `useAuthStore.mapAuthError` (`useAuthStore.ts:24–27`): `setAccountBanned({ reason: null })`, returns `null` (suppresses the raw Firebase message). No `signOut` needed (user was never signed in).
- **Cross-browser `auth/user-disabled`** in the request interceptor's token-mint catch (`api.ts:122–134`): `setAccountBanned({ reason: null })` + `await auth.signOut()` then rethrow.

**`isErrorWithCode` helper** — `src/lib/utils/isErrorWithCode.ts` (full):

```ts
export function isErrorWithCode(error: unknown, code: string): boolean {
  if (typeof error !== 'object' || error === null) return false;

  const e = error as {
    response?: { data?: { errors?: Array<{ code?: string }> } };
    data?: { errors?: Array<{ code?: string }> };
  };

  const errors = e.response?.data?.errors ?? e.data?.errors;
  return errors?.[0]?.code === code;
}
```

Shape: tolerates **both** the raw `AxiosError` (`error.response.data.errors[0].code`) **and** the unwrapped rejection shape `BACKEND_API` produces (`error.data.errors[0].code` — because the interceptor rejects with `error.response`, see Q10 `:69`). Reads the **first** error's `code` only. Confirm parity against mobile's Φ1 `isErrorWithCode`: web checks first-error-code across the two envelope shapes; mobile must match `{errors:[{code}]}` first-element semantics.

**CODE-vs-DOC divergence:** DOC §4.7 / §14.10 describe `sessionStorage.setItem('account-banned', '1')` for the ban paths. **CODE SAYS** the `accountBanned` Zustand flag is used everywhere; there is no `sessionStorage('account-banned')` write on disk. (Same family of staleness as Q4; spec §14.4/§14.10's store-flag prose is the authoritative as-built.)

---

## Q9 — Badges and messaging gating

**`ScheduledForDeletionBadge`** — `src/components/client/badges/ScheduledForDeletionBadge.tsx` (full):

```tsx
export default function ScheduledForDeletionBadge() {
  const tCommon = useTranslations(TranslationNamespaceEnum.COMMON);
  return <Badge variant="secondary">{tCommon('user.scheduled.for.deletion.label')}</Badge>;
}
```
- Content/key: `user.scheduled.for.deletion.label` (**COMMON**), shadcn `Badge variant="secondary"`.
- Placement: chat header (`Messages.tsx:167`) and chat-list row (`Chats.tsx:101`), shown when `peerPendingDeletion`.

**Chat-header gate — `Messages.tsx`:**
- Peer-state derivation (`:90–99`):
  ```ts
  const peer = activeChat?.withUser;
  const otherUid = peer?.firebaseUid ?? activeChat?.withUserFirebaseUid;
  const peerDeleted = isDeletedPeer(otherUid, peer);
  const blocking = otherUid ? isBlocking(otherUid) : undefined;
  const blockedBy = otherUid ? isBlockedBy(otherUid) : undefined;
  const peerPendingDeletion = peer?.state === 'PENDING_DELETION';
  const peerBanned = peer?.state === 'BANNED';
  const cannotSend = blocking || blockedBy || peerPendingDeletion || peerBanned || peerDeleted;
  ```
  So `cannotSend` extends to PENDING_DELETION and BANNED (and blocking/blockedBy/deleted). `cannotSend` is passed to `<MessageInput ... disabled={cannotSend} />` (`:322`).
- Header badges (`:167–172`): `{peerPendingDeletion && <ScheduledForDeletionBadge />}`; `{peerBanned && <Badge variant="secondary" className="text-red-600">{tCommon('user.banned.label')}</Badge>}`.
- Notices below the message list (`:278–291`):
  ```tsx
  {peerPendingDeletion && (... {tMessages('user.pending.deletion.notice')} ...)}
  {peerBanned && (... {tMessages('user.banned.notice')} ...)}
  ```
  Keys `user.pending.deletion.notice` and `user.banned.notice` are **MESSAGES_PAGE** (`tMessages`, `:37`); `user.banned.label` is **COMMON** (`tCommon`, `:39`).

**"Deleted User" null-safety — `Messages.tsx`:**
- Avatar + name fall back to `tCommon('user.deleted')` (`:161`, `:165`): `activeChat.withUser?.displayName ?? tCommon('user.deleted')`.
- For a deleted peer the dropdown drops Profile/Report/Block (only Delete remains) — `{peer && !peerDeleted && (...)}` (`:184–223`).

**"Deleted User" null-safety — `Chats.tsx`:**
- `const peerDeleted = isDeletedPeer(chat.withUserFirebaseUid, chat.withUser);` (`:75`)
- `const peerName = peerDeleted ? tCommon('user.deleted') : (chat.withUser?.displayName ?? tCommon('user.deleted'));` (`:76–78`)
- `peerPendingDeletion = chat.withUser?.state === 'PENDING_DELETION'`; `peerBanned = chat.withUser?.state === 'BANNED'` (`:79–80`); badges at `:101–106`.

**`isDeletedPeer` helper** — `src/messages/utils/deletedUser.ts` (full):
```ts
export function isDeletedPeer(firebaseUid: string | undefined, withUser: UserInfoDTO | undefined): boolean {
  if (!withUser) return true;
  if (firebaseUid?.startsWith('deleted:')) return true;
  return false;
}
```
Two deleted signals: a missing `UserInfoDTO` (post-hard-delete the backend returns no `withUser`) OR a `deleted:<uid>` sentinel firebaseUid (written by the cleanup cron). The `common.user.deleted` key the brief names is referenced in code as `tCommon('user.deleted')` (COMMON namespace + `user.deleted` ⇒ `common.user.deleted`).

---

## Q10 — Axios response interceptor (as-built)

**File:** `src/lib/config/api.ts` (139 lines). The instance is built by `createApiInstance` (`:15–74`); both interceptors are installed there / just after.

**Response interceptor** (`:26–71`):
- `X-Account-Restored` read (`:28–30`): on `response.headers['x-account-restored'] === 'true'` → `setRestored(true)`.
- Error branch network/timeout normalization (`:34–42`) and 404/no-response normalization (`:44–49`) — rejects with `{ data: { errorCode }, status }`.
- **403 `USER_BANNED`** (`:51–67`): reads `error.response.data.errors[0].code`; on `USER_BANNED` → `auth.signOut()`, `setAccountBanned({ reason: null })`, and **`return new Promise(() => {})`** (never resolves) so the rejection does not propagate to per-call handlers (spec §14.12). Other 403s fall through.
- Default (`:69`): `return Promise.reject(error.response)` — note it rejects with `error.response` (the unwrapped response), which is why `isErrorWithCode` checks `e.data?.errors` as well as `e.response?.data?.errors` (Q8).

**Cached-token reset on signout (C-7)** (`:78–92`): module-scoped `cachedToken`/`tokenExpiry`, plus an `onIdTokenChanged(auth, user => { if (!user) { cachedToken = null; tokenExpiry = 0; } })` at module-init.

**Request interceptor** (`:94–139`) — relevant for parity: sets `X-Base-Site`/`X-Lang`; returns early for anonymous; **respects a caller-supplied `Authorization` header** (`:111–113`, the deletion override); else attaches/refreshes the cached token; the token-mint `catch` handles cross-browser `auth/user-disabled` ban (`:122–134`).

> Matches contract C-7 / spec §14.12. Mobile's Φ1 interceptor already handles 403 `USER_BANNED`; the web shape to mirror is: read `errors[0].code === 'USER_BANNED'` → signOut + set store flag + never-resolving promise.

---

## Q11 — TS types

**`AuthUserDTO`** — `src/lib/types/user/AuthUserDTO.ts` (full). Deletion/ban fields:
- `disabled: boolean` (`:18`)
- `banReason: string | null` (`:20`)
- `deletionStatus: 'ACTIVE' | 'PENDING_DELETION'` (`:21`)
- `scheduledDeletionAt: string | null` (`:22`)
- (also `wasRegister: boolean` `:19`; no `state` field on this DTO.)

**`UserInfoDTO`** — `src/lib/types/user/UserInfoDTO.ts` (full). Deletion/ban fields:
- `state: 'ACTIVE' | 'PENDING_DELETION' | 'BANNED'` (`:18`)
- `scheduledDeletionAt: string | null` (`:19`)
- (no `disabled` / `banReason` / `deletionStatus` on this DTO.)

So the five fields the brief lists are **split across the two DTOs**: `AuthUserDTO` (the signed-in self) carries `disabled` / `banReason` / `deletionStatus` / `scheduledDeletionAt`; `UserInfoDTO` (a viewed/peer user) carries `state` (which folds ACTIVE/PENDING_DELETION/**BANNED** into one enum) / `scheduledDeletionAt`. There is also a standalone alias `src/lib/types/user/DeletionStatus.ts`: `export type DeletionStatus = 'ACTIVE' | 'PENDING_DELETION';`.

Parity note for mobile: the BANNED state for a peer is expressed only on `UserInfoDTO.state`; the self DTO uses the boolean `disabled` for the same condition. Mobile's DTOs should preserve this split (self = `disabled` boolean + `deletionStatus`; peer = `state` tri-state).

---

## Q12 — Web-only behaviors mobile must NOT mirror

A single consolidated list (each labeled where it lives):

1. **httpOnly `firebase_token` cookie clear (C-5):** `clearFirebaseTokenCookie()` (`src/lib/service/reactCalls/authTokenCookie.ts:45–48`) → `POST /api/auth/token` with `{ token: null }`. Mobile's token is in-memory; there is no SSR cookie to clear. **Skip the whole cookie step** (dialog `:117–124`).
2. **`writeFirebaseTokenCookie` mirroring** (`authTokenCookie.ts:35–39`) and its `onIdTokenChanged` cache-reset (`:19–25`) — the entire `/api/auth/token` route + cookie write is web-SSR plumbing. **Skip.**
3. **The `UseTokenRefresh` cookie-write during deletion** (`src/components/client/initializers/UseTokenRefresh.tsx:59–66`): when `deletionInFlight`, web still writes the cookie but suppresses the `firebase-sync` POST. The **cookie-write half is web-only**; the firebase-sync suppression concept (C-6) is portable if mobile has an analogous token listener.
4. **SSR stale-token race defense (C-3/C-8) and `fetchApi.ts`** — the whole "SSR call carries a stale `firebase_token` cookie" problem and its defenses (filter drop-context, `Bearer null` guard) are a Next-SSR concern. Mobile has no SSR; **no analog**.
5. **`revalidateUserCache(userId)`** — Next.js Data-Cache tag bust, called in the deletion success path (`DeleteAccountConfirmationDialog.tsx:106`) and in the restoration dialog's `onClose` (`AccountStateDialogsInit.tsx:128`). Mobile has no Next Data Cache. **Skip.**
6. **`router.replace` / `router.refresh` + locale-path navigation** — `DeleteAccountConfirmationDialog.tsx:129`, `AccountStateDialogsInit.tsx:65/100/130`. Mobile uses expo-router; mirror the *intent* (navigate to home / refresh the surface) with native navigation, not these Next APIs.
7. **The cached-token reset at module scope (C-7)** (`api.ts:78–92`) — web-specific in-memory axios cache. Mobile's Φ1 interceptor manages its own token lifecycle; only mirror if its caching model matches.
8. **sessionStorage** — **N/A**: Q4 confirms web does **not** use `sessionStorage` for the deletion/ban handoff (it uses Zustand store flags), so there is no sessionStorage mechanism to drop. The store-flag mechanism (`accountJustDeleted` / `accountBanned` / `restored`) **is** the portable contract.

---

## Out-of-scope (admin surfaces) — exist, admin-only, not mobile-relevant

Admin was removed from `oglasino-expo` in chat α; mobile is consumer-only. The following web admin files exist and are **not** mobile surfaces (one line each):
- `src/components/popups/dialogs/AdminBanUserDialog.tsx`, `AdminUnbanUserDialog.tsx`, `AdminLockUserDialog.tsx`, `AdminUnlockUserDialog.tsx`, `AdminUserStateInfoDialog.tsx` — admin-only dialogs (registered in `DialogManager.tsx:4–8`, `64–68`; ids `dialogRegistry.ts:28–32`).
- `src/components/admin/users/UserStateIndicators.tsx`, `EnableDisableButton.tsx`, `EnableDisableIcon.tsx`, `LockUnlockIcon.tsx`, `UserStateInfoIcon.tsx` and `app/[locale]/admin/users/[userId]/page.tsx` — admin Users-page indicators / actions. Admin-only; not mobile-relevant.
- Force-delete (banned-user email path) is a backend admin endpoint with no consumer UI. Not mobile-relevant.

---

## Divergence summary (CODE vs DOC), for the mobile briefs

1. **Post-deletion + ban + restore trigger = Zustand store flags, not sessionStorage.** Spec §14.4 documents this correctly; spec §14.3 step 4, §4.1 step 4, §4.7/§14.10, and auth-contract C-5 step 2 + the C-5 timeline rows still say `sessionStorage('account-just-deleted')` / `('account-banned')`. **Code is ground truth: store flags.** (Q4, Q8.)
2. **Danger Zone host component is `UserInfo` in `app/[locale]/owner/user/page.tsx`** (not a separate `UserSettingsForm`); spec §14.1's file path is correct. (Q1.)
3. **Provider detection = `auth.currentUser.providerData[0].providerId`**, not `AuthUserDTO.providerId`. (Q2.)
4. **403 `USER_BANNED` is handled only by the global interceptor**, never reaching the delete dialog's catch (never-resolving promise). (Q2, Q10.)
5. **BANNED peer-state lives only on `UserInfoDTO.state`**; the self DTO (`AuthUserDTO`) expresses the same via the `disabled` boolean. (Q11.)
6. Two extra ban surfaces beyond the spec's two trigger paths exist on web: `mapAuthError` (`auth/user-disabled` at sign-in) and the request-interceptor token-mint catch (cross-browser `auth/user-disabled`). (Q8.)
</content>
</invoke>
