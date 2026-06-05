# Audit — oglasino-expo — user-deletion current-state (mobile adoption)

**Type:** read-only current-state audit. No code changed, no edits staged, no commits.
**Branch:** `new-expo-dev`
**HEAD:** `b67627c` (Merge pull request #1 from memento-tech/feature/image-pipeline-v2)
**Date:** 2026-05-31

Scope note: this captures what exists in `oglasino-expo` *today* so the implementation briefs are written against the real layout. Where the feature expects something that is absent, it is flagged **NOT PRESENT**. Stub/dead/no-op code is quoted and flagged. No fixes proposed.

---

## Q1 — authStore current shape

**Path:** `src/lib/store/authStore.ts` (confirmed — single file, no alternate location).

**Full `AuthState` interface** (`authStore.ts:38-61`):

```ts
export interface AuthState {
  user: AuthUserDTO | null;
  loading: boolean;
  error: string | null;
  _hasHydrated: boolean;
  restored: boolean;
  accountBanned: boolean;

  resetAuthError: () => void;
  setUser: (user: AuthUserDTO | null) => void;
  setHasHydrated: (v: boolean) => void;
  setRestored: (value: boolean) => void;
  setAccountBanned: (value: boolean) => void;
  isAuthenticated: () => boolean;
  isLoading: () => boolean;

  login: (userData: LoginUserRequestDTO) => Promise<void>;
  register: (userData: RegisterUserRequestDTO) => Promise<void>;
  loginWithGoogle: () => Promise<void>;
  loginWithFacebook: () => Promise<void>;
  logout: () => Promise<void>;

  initAuthListener: () => void;
}
```

**Field-by-field presence:**

| Field / setter | Present? | Type & default |
| --- | --- | --- |
| `restored` / `setRestored` | **PRESENT** | `restored: boolean`, default `false` (`authStore.ts:43,71`); `setRestored: (value: boolean) => void` → `set({ restored: value })` (`:49,76`) |
| `accountBanned` / `setAccountBanned` | **PRESENT** | `accountBanned: boolean`, default `false` (`authStore.ts:44,71`); `setAccountBanned: (value: boolean) => void` → `set({ accountBanned: value })` (`:50,77`) |
| `accountJustDeleted` / `setAccountJustDeleted` | **NOT PRESENT** | — (this feature adds it; see Q2) |
| `deletionInFlight` / `setDeletionInFlight` | **NOT PRESENT** | — (this feature may add it; see Q7) |

**⚠️ Shape divergence from web (worth flagging for the implementation brief).** The brief notes web uses `accountBanned: { reason: string | null } | null`. **Mobile does NOT match** — mobile's `accountBanned` is a plain `boolean` (`authStore.ts:44`), and `restored` is a plain `boolean` (`:43`), not an object. The ban-notice dialog on mobile is deliberately reason-less (it renders fixed `banned.dialog.*` keys regardless of reason — see Q2), so the boolean carries no reason payload. This is internally consistent on mobile but is a real contract delta vs. web if the brief assumes the object shape.

**`logout()` action in full** (`authStore.ts:147-210`) — the reference for a full user-scoped store cleanup on this codebase:

```ts
logout: async () => {
  try {
    set({ loading: true });

    set({ user: null });

    useViewTokenStore.getState().clear();

    try {
      useChatListStore.getState().clearChatList();
    } catch (error) {
      logServiceError('authStore.logout.clearChatListStore', error);
    }

    try {
      useActiveChatStore.getState().clearActiveChat();
    } catch (error) {
      logServiceError('authStore.logout.clearActiveChatStore', error);
    }

    try {
      useChatNavStore.getState().clearChatNav();
    } catch (error) {
      logServiceError('authStore.logout.clearChatNavStore', error);
    }

    try {
      clearUserCache();
    } catch (error) {
      logServiceError('authStore.logout.clearUserCache', error);
    }

    try {
      useFavoritesStore.getState().clear();
    } catch (error) {
      logServiceError('authStore.logout.clearFavoritesStore', error);
    }

    try {
      useNotificationStore.getState().reset();
    } catch (error) {
      logServiceError('authStore.logout.resetNotificationStore', error);
    }

    try {
      usePortalFilterStore.getState().clearAllFilters();
    } catch (error) {
      logServiceError('authStore.logout.clearPortalFilterStore', error);
    }

    try {
      useDashboardFilterStore.getState().clearAllFilters();
    } catch (error) {
      logServiceError('authStore.logout.clearDashboardFilterStore', error);
    }

    await logoutFirebase();
  } catch (err: any) {
    logServiceError('authStore.logout', err);
    set({ error: err.message });
  } finally {
    set({ loading: false });
  }
},
```

Note: `logout()` does **not** touch `restored` or `accountBanned` (they are managed by `AccountStateDialogsInit` via the open-then-clear pattern). Persistence: only `user` is persisted (`partialize: (state) => ({ user: state.user })`, `authStore.ts:259`); `restored`/`accountBanned` are in-memory only.

---

## Q2 — AccountStateDialogsInit and the accountJustDeleted stub

**Path:** `src/components/init/AccountStateDialogsInit.tsx` (confirmed). Mounted in `AppInit.tsx:22`.

**Full component** (`AccountStateDialogsInit.tsx:1-53`):

```tsx
import { useEffect } from 'react';
import { DialogId } from '@/components/dialog/dialogRegistry';
import { useDialogStore } from '@/components/dialog/store/useDialogStore';
import { TranslationNamespace } from '@/i18n/types';
import { useTranslations } from '@/i18n/useTranslations';
import { useAuthStore } from '@/lib/store/authStore';

export default function AccountStateDialogsInit() {
  const tCommonSystem = useTranslations(TranslationNamespace.COMMON_SYSTEM);
  const tDialog = useTranslations(TranslationNamespace.DIALOG);
  const tButtons = useTranslations(TranslationNamespace.BUTTONS);

  const openDialog = useDialogStore((s) => s.openDialog);

  const restored = useAuthStore((s) => s.restored);
  const setRestored = useAuthStore((s) => s.setRestored);
  const accountBanned = useAuthStore((s) => s.accountBanned);
  const setAccountBanned = useAuthStore((s) => s.setAccountBanned);

  // Stub: accountJustDeleted will be added to AuthState by chat E (user-deletion adoption).
  // Chat E replaces this constant with a store selector and adds dialog logic to the effect below.
  const accountJustDeleted: string | null = null;

  useEffect(() => {
    if (!restored) return;
    openDialog(DialogId.INFO_DIALOG, {
      dialogTitle: tCommonSystem('common.system.account.restored.title'),
      dialogDescription: tCommonSystem('common.system.account.restored.subtitle'),
    });
    setRestored(false);
  }, [restored, openDialog, setRestored, tCommonSystem]);

  useEffect(() => {
    if (!accountBanned) return;
    openDialog(DialogId.INFO_DIALOG, {
      dialogTitle: tDialog('banned.dialog.title'),
      dialogDescription: [
        tDialog('banned.dialog.body.first'),
        tDialog('banned.dialog.body.delete.intro'),
        tDialog('banned.dialog.body.duration'),
      ].join('\n\n'),
      closeButtonLabel: tButtons('buttons.banned.go.home.label'),
    });
    setAccountBanned(false);
  }, [accountBanned, openDialog, setAccountBanned, tDialog, tButtons]);

  useEffect(() => {
    if (!accountJustDeleted) return;
    // Chat E adds the post-deletion dialog logic here.
  }, [accountJustDeleted]);

  return null;
}
```

**The `accountJustDeleted` stub** (`AccountStateDialogsInit.tsx:20-22`, `47-50`): it is a **local constant**, not a store selector — `const accountJustDeleted: string | null = null;`. The effect body is **present-but-empty** (a single comment, no `openDialog`). It is dead/no-op today: the guard `if (!accountJustDeleted) return;` always returns because the const is always `null`. Typed `string | null`, matching web's intent of carrying the `scheduledDeletionAt` string.

**`restored` slot** (`:24-31`): opens `DialogId.INFO_DIALOG` with `{ dialogTitle: tCommonSystem('common.system.account.restored.title'), dialogDescription: tCommonSystem('common.system.account.restored.subtitle') }`, then `setRestored(false)` **in the same effect** (open-then-clear pattern confirmed). Namespace: `COMMON_SYSTEM`.

**`accountBanned` slot** (`:33-45`): opens `DialogId.INFO_DIALOG` with title `tDialog('banned.dialog.title')`, description = three `DIALOG` keys joined by `\n\n` (`banned.dialog.body.first`, `banned.dialog.body.delete.intro`, `banned.dialog.body.duration`), and `closeButtonLabel: tButtons('buttons.banned.go.home.label')`. Then `setAccountBanned(false)` **in the same effect** (open-then-clear confirmed).

**Does `accountJustDeleted` activation still require the 3 handoff steps?** **Yes — unchanged since the Chat-E handoff.** The three steps remain exactly as named:
1. Add `accountJustDeleted` + `setAccountJustDeleted` to `AuthState` (NOT PRESENT — see Q1).
2. Swap the local const (`:22`) for a store selector.
3. Add `openDialog(...)` + the clear (`setAccountJustDeleted(null)`) inside the empty effect (`:47-50`).

The surrounding scaffold (`restored`/`accountBanned` slots, the `openDialog` import, the namespace hooks) is live and exemplary — the post-deletion slot follows the same shape the other two already use.

---

## Q3 — Dialog infrastructure

**How dialogs open on mobile.** A single Zustand store holds the active dialog id + props; one `<Modal>` host renders the mapped component.

- **API** (`src/components/dialog/store/useDialogStore.ts:1-16`):
  ```ts
  openDialog: (id: DialogId, props?: Record<string, any>) => void;
  closeDialog: () => void;
  // openDialog: (id, props = {}) => set({ currentDialogId: id, dialogProps: props })
  ```
  Confirmed signature: `useDialogStore.getState().openDialog(id, props)` (or via selector `useDialogStore((s) => s.openDialog)`). One dialog at a time (`currentDialogId: DialogId | null`).

- **`DialogId` enum** (`src/components/dialog/dialogRegistry.ts:1-23`) — full value list:
  `LOGIN_OPTIONS_DIALOG`, `REGISTER_DIALOG`, `LOGIN_DIALOG`, `SUGGEST_CATEGORY_DIALOG`, `NEW_PRODUCT_DIALOG`, `INFO_DIALOG` (`'infoDialog'`), `REPORT_DIALOG`, `PREVIEW_PRODUCT_DIALOG`, `DASHBOARD_PRODUCT_FUNCTIONS_DIALOG`, `VIEW_MESSAGE_IMAGES_DIALOG`, `PRODUCT_REVIEW_DIALOG`, `PRODUCT_REVIEW_NOT_ALLOWED_DIALOG`, `PRODUCT_ALREADY_REVIEWED_DIALOG`, `PRODUCT_SUCCESS_DIALOG`, `CARD_SELECTION_DIALOG`, `CURRENCY_SELECTION_DIALOG`, `FILTERS_DIALOG`, `APP_VERSION_CONFIGURATION_DIALOG`, `SOFT_PUSH_PERMISSION_DIALOG`, `CHAT_USER_FUNCTIONS_DIALOG`, `PORTAL_CONFIG_DIALOG`.
  **No `DELETE_ACCOUNT`/`CONFIRM_DELETE` value exists** (NOT PRESENT — this feature adds one if a bespoke dialog is needed).

- **Registry/host** (`src/components/dialog/DialogManager.tsx:32-96`): `DIALOGS: Record<string, React.ElementType>` maps the string id → component (`infoDialog: InfoDialog`, `reportDialog: ReportDialog`, etc.). `DialogManager` reads `currentDialogId`/`dialogProps`, renders a single `<Modal>` wrapping `DialogWrapper` which spreads `{...dialogProps}` into the chosen component and injects an `onClose` that calls `dialogProps.onClose` if provided else `closeDialog()`. `VIEW_MESSAGE_IMAGES_DIALOG` is registered in the enum but commented out in the map (`DialogManager.tsx:53`). `DialogManager` is mounted in `app/_layout.tsx`.

**Is there an `INFO_DIALOG` equivalent to web's title/description/closeButtonLabel/onClose/autoDismiss shape?** Yes — `InfoDialog`. **Prop interface** (`src/components/dialog/dialogs/InfoDialog.tsx:9-18`):

```ts
interface InfoDialogProps {
  onClose: () => void;
  onContinue?: () => Promise<boolean>;
  continueButtonLabel?: string;
  closeButtonLabel?: string;
  onContinueError?: string;
  type?: 'info' | 'error' | 'warning';
  dialogTitle: string;
  dialogDescription: string;
}
```

- `closeButtonLabel?: string` — **PRESENT** (`:13`, falls back to `tDialog('button.close.label')` at `:64`). Matches the Φ1 Brief 4B addition (already consumed by the ban dialog at `AccountStateDialogsInit.tsx:42`).
- An auto-dismiss prop (`autoDismissAfterMs` or similar) — **NOT PRESENT**. `InfoDialog` has no timer/auto-close logic anywhere in the file. If the post-deletion dialog needs auto-dismiss, it must be added.
- `onContinue?: () => Promise<boolean>` + `continueButtonLabel` give a confirm/cancel two-button mode with an async action returning success (`:37-52`) — usable for a simple confirm, but it has **no input field**.

**Confirmation dialog with a password input (for the delete reauth flow)?** **No existing reusable shape.** `InfoDialog` renders only title + description + up to two buttons (`InfoDialog.tsx:54-86`), no input. The dialog system is bespoke-per-id, not a generic body-slot host. The closest precedents that combine *body content + a text input + submit/cancel* are:
- `ReportDialog` (`src/components/dialog/dialogs/ReportDialog.tsx`) — `TextInput` + option select + submit + error/success local state (`:14,45-50`).
- `LoginDialog` / `RegisterDialog` — email/password inputs (these already do Firebase email/password auth).

So the email/password reauth + confirm dialog would be a **new bespoke dialog component** registered under a new `DialogId`, following the `ReportDialog`/`LoginDialog` pattern. NOT PRESENT today.

---

## Q4 — The user settings screen (where the Danger Zone goes)

**Actual path:** `app/owner/dashboard/user.tsx` (confirmed — the handoff's guess is correct; the Consent Mode C+G work reshaped this exact file, it was not moved). Component `UserScreen`.

**Current section structure, top-to-bottom** (inside `<ScrollView><View className="gap-5 px-3">`, `user.tsx:217-356`):

1. `<AvatarUpload>` (`:220-224`).
2. **Identity card** — `border` group (`:225-246`): email `Input` (disabled when `user.providerId` set), username `Input`, short-bio `Textarea`, phone-number `Input`.
3. **Region/city card** — `border` group (`:248-265`): region `Input` (disabled), city `Input` (disabled), `<CitySelector>`.
4. **Toggles block** — `<View className="w-full items-end pb-10">` (`:266-348`), four toggle rows then the consent toggle:
   - `allowNotifications` toggle (`:267-280`) — labels from `COOKIES` (`notifications.label/description/warning`).
   - `allowEmails` toggle (`:282-291`) — `COOKIES` (`email.*`).
   - `allowPromoEmails` toggle (`:293-306`) — `COOKIES` (`email.promo.*`).
   - `allowPhoneCalling` toggle (`:308-321`) — `DASHBOARD_PAGES` (`phone.number.setup.*`).
   - **Analytics consent toggle** (`:322-347`) — `COOKIES` (`mobile.consent.settings.label/description`); device-local, reads `useConsentStore`, writes-through on flip (Consent Mode G addition).
5. **Save button block** — `<View className="pb-3">` (`:349-355`): `<Button onPress={saveChanges}>` with `tDash('user.submit.label')` + an inline `errorMessage` text below.

**Bottom of the screen (where a Danger Zone would be appended)** (`user.tsx:322-356`):

```tsx
          {/* Analytics consent — a standalone, device-local control ... */}
          <View className="mt-5 w-full flex-row items-center gap-4 rounded-md border border-border-mild p-2">
            <View className="flex-1">
              <ToggleSettingItemLabel title={tCookies('mobile.consent.settings.label')}>
                <Text className="text-center">{tCookies('mobile.consent.settings.description')}</Text>
              </ToggleSettingItemLabel>
            </View>

            <Switch
              className="self-start"
              value={consentDecision?.analytics === true}
              disabled={!consentHydrated}
              onValueChange={(next) =>
                setConsentDecision({
                  analytics: next,
                  decidedAt: Math.floor(Date.now() / 1000),
                  version: 1,
                })
              }
            />
          </View>
        </View>
        <View className="pb-3">
          <Button variant="outline" onPress={saveChanges} disabled={loading}>
            <ActivityIndicator animating={loading} />
            <Text>{tDash('user.submit.label')}</Text>
          </Button>
          <Text className="mt-1 text-center italic text-red-600">{errorMessage}</Text>
        </View>
      </View>
    </ScrollView>
```

A Danger Zone would sit below the Save button block (`:349-355`), as the last child of the outer `gap-5 px-3` `View`.

**Existing "delete account" UI?** **NONE — confirmed absent.** No delete button, no danger-zone section, no reference to deletion anywhere in `user.tsx`.

**The `allowNotifications` dead toggle** (so the brief doesn't collide): `const [allowNotifications, setAllowNotifications] = useState(false);` (`user.tsx:47`). It is **dead** — it is never seeded from the fetched `details` (the `getUserDetails().then(...)` block at `:64-78` seeds `allowEmails`/`allowPromoEmails`/`allowPhoneCalling` but **not** `allowNotifications`), it is **not** in the change-detection comparison (`:95-106`), and it is **not** in the `updateUser({...})` save body (`:170-182`). The `<Switch>` (`:275-279`) flips local state that goes nowhere.

**The consent toggle Consent Mode left** (`user.tsx:322-347`): the analytics-consent `<Switch>` described in §4 above. It is **deliberately separate** from the save flow — reads/writes `useConsentStore` directly, device-local, NOT part of `saveChanges`/change-detection/the `updateUser` body (comment at `:53-58, 323-327`). The deletion brief must not fold either toggle into the delete flow.

---

## Q5 — Service layer + the Φ4 surface-via-throw convention

**`parseServiceError` helper — PRESENT.** Location `src/lib/utils/parseServiceError.ts`. Signatures:

```ts
export type ServiceFieldError = { field: string | null; code: string; translationKey?: string };
export type ParsedServiceError = { errors: ServiceFieldError[]; byField: Record<string, ServiceFieldError> };

export function parseServiceError(error: unknown): ParsedServiceError            // :40
export function findSystemError(errors: ServiceFieldError[]): ServiceFieldError | null  // :84
```

It reads `(error as AxiosError).response?.data?.errors` (`parseServiceError.ts:45`), tolerates non-contract errors with an empty neutral result, never throws, never touches translations. `findSystemError` returns the first `field === null` entry (the `__system` slot).

**`userService` — PRESENT.** `src/lib/services/userService.ts`. Existing functions:
- `updateUserAuth(user)` (`:9`)
- `getUserDetails(user)` (`:19`) — returns `UpdateUserDTO`
- `updateUser(user)` (`:29`)
- `getUserForId(userId)` (`:39`)
- `getUserFollows(paging)` (`:49`)
- `getUserPhoneNumber(userId)` (`:59`)
- `getUserForFirebaseUid(firebaseUid)` (`:69`) — returns `null` on 404 (chat "Deleted User" fallback path)

**`deleteCurrentUser`-equivalent — NOT PRESENT.** No delete/erase function in `userService.ts` or anywhere else (grep clean). This feature adds it.

**Representative Φ4 surface-via-throw services** (house style for the new `deleteCurrentUser`):

`userService.updateUser` (`userService.ts:29-37`):
```ts
export const updateUser = async (user: UpdateUserDTO) => {
  try {
    const res = await BACKEND_API.post('/secure/user/update', user);
    return res.status >= 200 && res.status < 300;
  } catch (err) {
    logServiceError('user.updateUser', err);
    throw err;
  }
};
```

`userService.getUserFollows` (`userService.ts:49-57`):
```ts
export const getUserFollows = async (paging: PagingDTO): Promise<UserFollowingsDTO> => {
  try {
    const res = await BACKEND_API.post<UserFollowingsDTO>('/secure/user/follows', paging);
    return res.data;
  } catch (err) {
    logServiceError('user.getUserFollows', err);
    throw err;
  }
};
```

Pattern: `try { await BACKEND_API... } catch (err) { logServiceError('<dotted.tag>', err); throw err; }`. The caller then runs `parseServiceError(err)` to render per-field/`__system` messages.

**Per-call Authorization header override (the web `getIdToken(true)` → explicit-header trick).** ⚠️ **This is a platform gotcha worth flagging.**

The request interceptor **unconditionally overwrites** `Authorization` whenever a Firebase user is signed in (`src/lib/config/api.ts:41-52`):

```ts
instance.interceptors.request.use(async (config) => {
  const currentUser = auth.currentUser;

  const { selectedBaseSite, language } = useBootStore.getState();
  if (selectedBaseSite) config.headers.set('X-Base-Site', selectedBaseSite.code);
  if (language) config.headers.set('X-Lang', language.code);

  if (currentUser && config.headers) {
    config.headers.set('Authorization', `Bearer ${(await currentUser.getIdToken()).toString()}`);
  }
  return config;
});
```

- It calls `currentUser.getIdToken()` **without force-refresh** and **clobbers** any caller-supplied `Authorization` header. So mobile's interceptor does **NOT** respect a per-call `Authorization` override the way web's does — a `{ headers: { Authorization: 'Bearer ' + freshToken } }` passed by a caller is discarded.
- Mitigating nuance: Firebase caches the token from a prior `getIdToken(true)`, so a subsequent un-forced `getIdToken()` on the *same* `currentUser` returns that fresh token while it is valid. So if the delete flow does reauth → `getIdToken(true)` → immediately fire the request, the interceptor's own `getIdToken()` returns the just-refreshed token and the `auth_time` claim is fresh. But the *explicit per-call header* itself is not honored — the brief must not assume web's pattern transfers verbatim.
- Precedent that "works around" it by relying on the same-user/cached-token behavior: `authService.syncUserToBackend` (`authService.ts:125-137`) passes `Authorization: Bearer ${idToken}` explicitly on `/auth/firebase-sync`; that header is clobbered by the interceptor but the value is equivalent because it is the same `currentUser`'s token.

---

## Q6 — isErrorWithCode parity

**`isErrorWithCode`** (`src/lib/utils/isErrorWithCode.ts:1-9`):

```ts
import { AxiosError } from 'axios';

export function isErrorWithCode(error: unknown, code: string): boolean {
  if (typeof error !== 'object' || error === null) return false;

  const e = error as AxiosError<{ errors?: { code?: string }[] }>;
  const errors = e.response?.data?.errors;
  return errors?.[0]?.code === code;
}
```

- **First-element semantics — CONFIRMED:** reads `errors[0].code` only (`:8`).
- **Envelope tolerated:** only `error.response.data.errors`. It does **NOT** tolerate the bare `error.data.errors` shape that web also accepts. (Single shape, not the dual shape.)

**Does this line up with what mobile's axios interceptor rejects with?** **Yes.** The interceptor (`api.ts:54-147`) always rejects with the axios error object — either the raw `error` (which carries `.response`), or a spread `{ ...error, status, errorCode }` that **preserves `.response`** (e.g. `:63-67`, `:71-75`, `:138-142`). So `error.response.data.errors` remains reachable in every reject path. `parseServiceError` reads the identical `response?.data?.errors` path (`parseServiceError.ts:45`). They line up.

- One exception that does **not** reject at all: the `USER_BANNED`/`EMAIL_BANNED` 403 branch returns `new Promise(() => {})` (never resolves; `api.ts:90-94`) after firing the ban hook — so `isErrorWithCode` is consumed *inside* the interceptor there, not by a downstream caller. The downstream callers that use `isErrorWithCode` against rejected errors are `authStore.initAuthListener` (`authStore.ts:244`) and `ForegroundRevalidationInit` (`ForegroundRevalidationInit.tsx:38`), both reading the `firebase-sync` rejection — which is a normal rejected axios error with `.response` intact. Consistent.

---

## Q7 — Auth listener, logout, 403 interceptor (Φ1 scaffolds)

**`logoutFirebase` / logout path** (`src/lib/services/authService.ts:233-240`):

```ts
export const logoutFirebase = async (): Promise<void> => {
  try {
    await auth.signOut();
  } catch (e) {
    logServiceError('auth.logoutFirebase.signOut', e);
  }
  await GoogleSignin.signOut();
};
```

`auth.signOut()` **is called** (Φ1 F2 confirmed), followed by `GoogleSignin.signOut()`. Invoked from `authStore.logout()` (`authStore.ts:203`).

**`initAuthListener` / `onIdTokenChanged` handler** (`authStore.ts:215-254`):

```ts
initAuthListener: () => {
  if (listenerState.unsubscribe) return;

  listenerState.unsubscribe = onIdTokenChanged(auth, async (firebaseUser: FirebaseUser | null) => {
    if (!firebaseUser) {
      listenerState.inFlightUid = null;
      listenerState.lastSyncedUid = null;
      listenerState.lastSyncedAt = 0;
      set({ user: null });
      return;
    }

    const uid = firebaseUser.uid;

    if (listenerState.inFlightUid === uid) return;
    if (
      listenerState.lastSyncedUid === uid &&
      Date.now() - listenerState.lastSyncedAt < SAME_UID_REPEAT_WINDOW_MS
    ) {
      return;
    }

    listenerState.inFlightUid = uid;
    try {
      const backendUser = await syncUserToBackend(firebaseUser);
      set({ user: backendUser });
      listenerState.lastSyncedUid = uid;
      listenerState.lastSyncedAt = Date.now();
    } catch (err) {
      if (isErrorWithCode(err, 'USER_BANNED') || isErrorWithCode(err, 'EMAIL_BANNED')) {
        set({ accountBanned: true });
        await auth.signOut();
        return;
      }
      logServiceError('authStore.initAuthListener', err);
    } finally {
      listenerState.inFlightUid = null;
    }
  });
},
```

**Does it read `deletionInFlight` and skip `firebase-sync` when true (contract C-6)? NOT PRESENT.** There is no `deletionInFlight` flag anywhere (grep clean), and the listener always runs `syncUserToBackend` for a signed-in user. This feature must add it if C-6 is in scope — the relevant guard point is before `syncUserToBackend` at `authStore.ts:238`. Note the dedup machinery (`inFlightUid`, `lastSyncedUid`, `SAME_UID_REPEAT_WINDOW_MS`) is adjacent and would interact with any added skip.

**The axios 403 interceptor** (`api.ts:89-95`):

```ts
if (error.response.status === 403) {
  if (isErrorWithCode(error, 'USER_BANNED') || isErrorWithCode(error, 'EMAIL_BANNED')) {
    auth.signOut();
    authHooks.onAccountBanned?.();
    return new Promise(() => {});
  }
}
```

- **`USER_BANNED` (and `EMAIL_BANNED`) — CONFIRMED:** signs out (`auth.signOut()`) and calls `authHooks.onAccountBanned?.()`. That hook is wired in `src/lib/init/authInterceptors.ts:11` to `useAuthStore.getState().setAccountBanned(true)`, which drives `AccountStateDialogsInit`'s ban dialog. Returns a never-resolving promise to halt the chain.

- **`X-Account-Restored` → `restored`** — handled in the **success-response** interceptor, not the 403 branch (`api.ts:54-60`):
  ```ts
  (response: AxiosResponse) => {
    if (response.headers?.['x-account-restored'] === 'true') {
      authHooks.onAccountRestored?.();
    }
    return response;
  },
  ```
  `onAccountRestored` is wired to `useAuthStore.getState().setRestored(true)` (`authInterceptors.ts:10`), driving the restored dialog. **CONFIRMED at the interceptor layer** (Φ1 Brief 4A), reading the lowercased header `x-account-restored === 'true'`.

**Does the listener participate in restoration?** **No.** Restoration is purely the `firebase-sync` (and any other authed) response-header path read by the success interceptor (`api.ts:56`). The `onIdTokenChanged` listener has no restoration branch — matching backend C-2/C-3 (the backend sets `X-Account-Restored` on the response of the request whose token reaches `FirebaseAuthFilter`; on mobile that is the `syncUserToBackend` POST, whose response flows through the success interceptor and trips `onAccountRestored`). The boolean-flag mechanism (not sessionStorage) is the mobile mirror.

---

## Q8 — Reauth capability on mobile

Mobile uses the **Firebase JS SDK** (`firebase/auth`), via the modular `initializeAuth` + `getReactNativePersistence(AsyncStorage)` (`src/lib/client/firebaseClient.ts:46-50`) — **not** `@react-native-firebase`. So reauth APIs are the JS-SDK modular functions.

**Email/password reauth.** `reauthenticateWithCredential` + `EmailAuthProvider.credential` are exported by `firebase/auth` (the package already in use). **No existing usage** of either in the repo (grep `reauthenticate`/`EmailAuthProvider` → no matches). The codebase already imports sibling JS-SDK functions from `firebase/auth` (`signInWithEmailAndPassword`, `signInWithCredential`, `createUserWithEmailAndPassword`, `GoogleAuthProvider`, `onAuthStateChanged` — `authService.ts:5-12`), so adding `reauthenticateWithCredential` + `EmailAuthProvider` to that import is the expected path.

**Google reauth.** Web's `reauthenticateWithPopup` is web-only and does **not** exist on RN. Mobile's Google sign-in mechanism is **`@react-native-google-signin/google-signin`** (`GoogleSignin`). Existing sign-in flow (`authService.ts:1, 169-195`):

```ts
import { GoogleSignin, statusCodes } from '@react-native-google-signin/google-signin';
...
GoogleSignin.configure({
  webClientId: firebaseConfig?.webClientId,
  offlineAccess: true,
});

export const loginWithGoogleFirebase = async (): Promise<AuthUserDTO> => {
  try {
    await GoogleSignin.hasPlayServices();

    const userInfo = await GoogleSignin.signIn();
    const idToken = userInfo.data?.idToken;
    const credential = GoogleAuthProvider.credential(idToken);
    const userCredential = await signInWithCredential(auth, credential);

    return buildUserSession(userCredential.user);
  } catch (error: any) {
    if (error.code === statusCodes.SIGN_IN_CANCELLED) { ... }
    ...
  }
};
```

The reauth mirror would be: `GoogleSignin.signIn()` → `GoogleAuthProvider.credential(idToken)` → `reauthenticateWithCredential(auth.currentUser, credential)` (instead of `signInWithCredential`). `GoogleSignin.signOut()` is also called in `logoutFirebase` (`authService.ts:239`). Facebook is stubbed/commented out and returns `null` (`authService.ts:200-228`) — not enabled.

**Provider detection.** `auth.currentUser.providerData[0].providerId` is **available on mobile** — the JS SDK exposes `providerData`, and it is already read at `authService.ts:130` (`firebaseUser.providerData?.[0]?.providerId`) to populate the `providerId` sent to `firebase-sync`. The backend value also lands on `AuthUserDTO.providerId` (optional; `AuthUserDTO.ts:12`) and is read in `user.tsx:230-233` to lock the email field. **No dedicated provider-detection helper exists** — the only detection site is the inline `providerData?.[0]?.providerId` read; the brief can reuse that idiom or the stored `user.providerId`.

---

## Q9 — Badges, messaging gating, deleted-user rendering on mobile

**`ScheduledForDeletionBadge` equivalent — NOT PRESENT.** No "scheduled for deletion" / grace-period badge anywhere. The only chat badge primitive is the **block badge** in the chat list (`Chats.tsx:55-58`: renders `tMessages('blocking.label')` or `tMessages('blocked.by.label')`) and the in-chat `blocked.by.label` notice (`Messages.tsx:238-242`). There is no generic `Badge` component import for user-state; the grep hit on "Badge" in `Chats.tsx` is the inline block-badge view.

**Chat header / chat list — do they read `withUser.state`? NO.** Neither component reads any `state` field (and `UserInfoDTO` has none — see Q10). State-related reads that *do* exist, all from the messaging adoption (`oglasino-expo-messaging-adoption`):

- **`isDeletedPeer`** — single deleted-detection helper (`src/components/messages/utils.ts:16-21`):
  ```ts
  export function isDeletedPeer(
    withUser: UserInfoDTO | null | undefined,
    peerUid: string | null | undefined
  ): boolean {
    return !withUser || !!peerUid?.startsWith('deleted:');
  }
  ```
  This is a **post-hard-delete** guard only: `withUser` null (backend 404) **or** the `deleted:<uid>` cleanup-cron sentinel. It does **not** model the grace-period `PENDING_DELETION` state.
- Chat list row (`Chats.tsx:28,41-47`): `peerDeleted` → renders `tCommon('user.deleted')` instead of name/avatar.
- Chat detail header (`Messages.tsx:65,200-205`): same `peerDeleted` → `tCommon('user.deleted')` fallback.
- `ChatUserFunctionsDialog` (`ChatUserFunctionsDialog.tsx:47,66,79`): `peerDeleted` guards hide block/report actions.

**`cannotSend`-equivalent gate? NO grace-period gate.** The message-input disable is purely block-based (`Messages.tsx:65-70, 263`):
```ts
const peerDeleted = isDeletedPeer(...);
const blocking = otherUid ? !!blockingMap[otherUid] : undefined;
const blockedBy  = otherUid ? !!blockedByMap[otherUid] : undefined;
const blocked = blocking || blockedBy;
...
disabled={blocked}   // line 263 — input/send disabled on block, not on peer deletion-state
```
There is no "this user has scheduled their account for deletion and cannot receive messages" notice and no input-disable keyed to a pending-deletion `state`. The deletion feature would need to add that (and it depends on a `state` field reaching `UserInfoDTO` — see Q10).

**`user.deleted` fallback — PRESENT** (`COMMON` namespace), consumed at `Chats.tsx:47` and `Messages.tsx:205`.

**Summary of what messaging adoption already wired (so the deletion work doesn't duplicate):** hard-delete deleted-peer fallback (`isDeletedPeer` + `user.deleted`), block badges/notices, block-based input disable. **What it did NOT wire (deletion feature's job):** the grace-period `PENDING_DELETION` badge, the pending-deletion send gate/notice, and any `UserInfoDTO.state` reading.

---

## Q10 — TS DTO types on mobile

**`AuthUserDTO`** (self DTO, `src/lib/types/user/AuthUserDTO.ts:4-17`):

```ts
export interface AuthUserDTO {
  id: number;
  firebaseUid: string;
  displayName: string;
  email: string;
  baseSite: BaseSiteDTO;
  regionAndCity: RegionAndCityDTO;
  profileImageKey: string;
  providerId?: string;
  allowNotifications?: boolean;
  allowEmails?: boolean;
  allowPromoEmails?: boolean;
  allowPhoneCalling: boolean;
}
```

Self-DTO deletion/ban fields:
| Field | Present? |
| --- | --- |
| `disabled` | **NOT PRESENT** |
| `banReason` | **NOT PRESENT** |
| `deletionStatus` | **NOT PRESENT** |
| `scheduledDeletionAt` | **NOT PRESENT** |

**`UserInfoDTO`** (peer DTO, `src/lib/types/user/UserInfoDTO.ts:4-18`):

```ts
export interface UserInfoDTO {
  id: number;
  firebaseUid: string;
  displayName: string;
  profileImageKey: string;
  shortBio: string;
  rating: number;
  isVerified: boolean;
  activeProducts: number;
  iamActive: boolean;
  allowPhoneCalling: boolean;
  isFollowingCurrent: boolean;
  baseSiteOverview: BaseSiteOverviewDTO;
  regionAndCity: RegionAndCityDTO;
}
```

Peer-DTO deletion fields:
| Field | Present? |
| --- | --- |
| `state` | **NOT PRESENT** |
| `scheduledDeletionAt` | **NOT PRESENT** |

**All six deletion/ban DTO fields are NOT PRESENT.** Neither the messaging adoption nor any earlier feature added `state`/`scheduledDeletionAt` to `UserInfoDTO` — confirming why the chat layer keys deleted-detection off `withUser == null` / the `deleted:` UID sentinel rather than a `state` field (Q9). The deletion feature must add `state` (+ `scheduledDeletionAt`) to `UserInfoDTO` for the grace-period badge/gate, and the relevant self-DTO fields if any self-screen needs them.

---

## Q11 — Translation key availability

**Fetch mechanism.** Mobile fetches/registers translations during boot via the freshness gate in `src/lib/store/bootStore.ts`. The namespace set is **every value of the `TranslationNamespace` enum** — the freshness/fetch loop iterates `Object.values(TranslationNamespace)` (`bootStore.ts:451`), and i18n registers `ns: Object.values(TranslationNamespace)` (`:541`). So there is no curated subset; the full enum is fetched per active language.

**`TranslationNamespace` enum** (`src/i18n/types.ts:6-38`): `COMMON`, `COMMON_SYSTEM`, `ERRORS`, `VALIDATION`, `PAGING`, `BUTTONS`, `INPUT`, `DIALOG`, `HEADER`, `FOOTER`, `NAVIGATION`, `INTRO`, `EXTRA_PRODUCTS`, `COOKIES`, `MESSAGES_PAGE`, `DASHBOARD_PAGES`, `ABOUT_PAGE`, `FREE_ZONE_PAGE`, `PRICING_PAGE`, `METADATA`.

**Are the six deletion-needed namespaces fetched?** **All present in the enum, therefore all fetched:**
| Namespace | In enum / fetched? |
| --- | --- |
| `DASHBOARD_PAGES` | ✅ (`types.ts:31`) |
| `BUTTONS` | ✅ (`types.ts:15`) |
| `DIALOG` | ✅ (`types.ts:17`) |
| `MESSAGES_PAGE` | ✅ (`types.ts:30`) |
| `COMMON` | ✅ (`types.ts:8`) |
| `ERRORS` | ✅ (`types.ts:10`) |

**No namespace the feature needs is missing from mobile's fetch set.**

**Spot-check (key-level).** Mobile reads the same backend-seeded tables at runtime; the repo holds no static fallback table, so key *values* can't be asserted from source alone. What the code already *consumes successfully today* proves the namespace + several keys resolve:
- `banned.dialog.title` (+ `banned.dialog.body.first/.delete.intro/.duration`) — `DIALOG` — already consumed in `AccountStateDialogsInit.tsx:36-40`. ✅ resolves.
- `user.deleted` — `COMMON` — already consumed in `Chats.tsx:47`, `Messages.tsx:205`. ✅ resolves.
- `danger.zone.label` (`DASHBOARD_PAGES`), `delete.account.label` (`BUTTONS`), `reauth.required` (`ERRORS`) — namespaces are fetched, but these specific keys are not yet referenced in mobile code; whether the backend has them seeded for all four languages is a backend/seed concern, not a mobile-fetch concern. Per the brief, the deletion feature seeds **no** new keys (backend already has them); mobile just needs the namespaces, which it has.

Caveat: `common.system.account.restored.title/.subtitle` use `COMMON_SYSTEM` (`AccountStateDialogsInit.tsx:27-28`), also fetched. The error keys from the backend delete endpoint arrive **without** the `errors.` prefix (`reauth.required`, `user.locked.from.deletion`); rendering them is the screen's job via the `ERRORS` namespace lookup on the parsed `translationKey`/`code`.

---

## Q12 — Navigation + sign-out-and-redirect

**Mobile's redirect-after-sign-out mechanism (the `SessionGuard` analog).** There is no `(secured)/_layout.tsx` (it was deleted on this branch — `git status` shows `D app/(portal)/(secured)/_layout.tsx`). Guards are **inline per protected surface**, each redirecting to `/` when `user` is null:

- Owner dashboard layout (`app/owner/_layout.tsx:16-29`):
  ```ts
  const user = useAuthStore((s) => s.user);
  const hasHydrated = useAuthStore((s) => s._hasHydrated);
  ...
  if (!hasHydrated) return null;
  if (!user) return <Redirect href="/" />;
  ```
- Portal secured screens, each with its own guard: `app/(portal)/(secured)/favorites.tsx:10`, `messages.tsx:14`, `notifications.tsx:100` — all `if (!user) return <Redirect href="/" />;`.

So when `auth.signOut()` fires, `onIdTokenChanged` sets `{ user: null }` (`authStore.ts:223`), every mounted guard re-renders and `<Redirect href="/" />` drops the user off any protected screen to home — the mobile equivalent of web's `router.replace(\`/${locale}\`)` + `SessionGuard`. The settings screen `user.tsx` itself short-circuits with `if (!user) return <></>;` (`user.tsx:82`) and sits under the owner layout's redirect.

**Existing sign-out-then-navigate site (the logout flow).** `UserMenu.tsx:162-168`:
```tsx
onPress={async () => {
  useAuthStore.getState().logout();
  setOpen(false);
  router.push('/');
}}
```
`logout()` clears stores + calls `logoutFirebase()` (→ `auth.signOut()`); the menu also does an explicit `router.push('/')`. This is the reference for the delete success path: clear/sign-out then push home. (Note `router.push` here, vs. the guard-driven `<Redirect>`; the delete path can rely on the guard redirect like web's `SessionGuard`, optionally with an explicit `router.replace('/')`.)

**How the boot/gate machine reacts to `signOut`.** `bootStore` governs boot/freshness/maintenance/offline gating, **not** auth — signing out does **not** re-enter the boot machine or reset the gate (the boot gate has already reached `ready`; `auth.signOut()` only flips `useAuthStore.user` to null). So sign-out is purely an auth-store transition that the inline route guards react to; there is no boot re-gate to contend with. The delete success path therefore needs only: (1) set the post-deletion store flag (the future `accountJustDeleted` per Q2), (2) `auth.signOut()` (via `logout()` or directly), (3) let the guard redirect (and/or `router.replace('/')`) — the post-deletion dialog then opens from `AccountStateDialogsInit` on the home surface, mirroring web's sessionStorage-key-on-root-mount behavior with the store-flag mechanism instead.

---

## Cross-cutting NOT-PRESENT summary (what this feature must add on mobile)

1. `authStore`: `accountJustDeleted`/`setAccountJustDeleted` (Q1, Q2); `deletionInFlight`/`setDeletionInFlight` if C-6 is in scope (Q1, Q7).
2. `AccountStateDialogsInit`: activate the empty `accountJustDeleted` effect (3 handoff steps, unchanged) (Q2).
3. Dialog infra: no auto-dismiss prop on `InfoDialog`; no reauth/confirm-with-input dialog — new bespoke dialog + `DialogId` value needed (Q3).
4. Settings screen: no Danger Zone / delete UI (Q4).
5. Service layer: no `deleteCurrentUser` in `userService` (Q5); the interceptor clobbers per-call `Authorization` overrides — web's explicit-fresh-token-header pattern does not transfer verbatim (Q5).
6. Auth listener: no `deletionInFlight` skip of `firebase-sync` (Q7).
7. Chat: no grace-period `PENDING_DELETION` badge or send-gate/notice (Q9).
8. DTOs: `disabled`/`banReason`/`deletionStatus`/`scheduledDeletionAt` on `AuthUserDTO`; `state`/`scheduledDeletionAt` on `UserInfoDTO` (Q10).

## What is already in place (do not rebuild)

- Store-flag handoff mechanism + `restored`/`accountBanned` slots wired end-to-end (interceptor hooks → `authStore` setters → `AccountStateDialogsInit` open-then-clear dialogs) (Q2, Q7).
- `parseServiceError`/`findSystemError` + surface-via-throw service house style (Q5).
- `isErrorWithCode` (first-element, `response.data.errors`) aligned with the interceptor reject shape (Q6).
- 403 `USER_BANNED`/`EMAIL_BANNED` sign-out + ban dialog; `X-Account-Restored` → restored dialog (Q7).
- Google sign-in via `@react-native-google-signin` + `GoogleAuthProvider.credential`; `providerData[0].providerId` available (Q8).
- Hard-delete deleted-peer rendering (`isDeletedPeer` + `user.deleted`) and block gating in chat (Q9).
- All required translation namespaces fetched at boot (Q11).
- Inline per-screen auth guards redirect to `/` on sign-out (SessionGuard analog) (Q12).
