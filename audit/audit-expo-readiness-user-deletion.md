# Audit — Expo release readiness: user deletion and account-disabling enforcement

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Mode:** read-only audit. No code changes.

---

## Section 1 — Account deletion surface

**No user-initiated "delete my account" UI exists on mobile today.**

There is no deletion endpoint call, no confirmation flow, no danger zone, no reauth flow. The user settings screen (`app/owner/dashboard/user.tsx`) contains profile editing (avatar, display name, email, phone number, bio, region/city) and preference toggles, but no deletion-related UI whatsoever.

Grep for `delete.*account`, `danger.*zone`, `deleteAccount`, `deleteUser`, `account.*delet`, and `self.delet` across `src/` and `app/` returned zero matches.

The user-deletion spec (`features/user-deletion.md`) explicitly states: "Mobile (`oglasino-expo`) adopts post-merge in its own Mastermind chat." This is expected — the feature is in the Expo backlog table with status `not-started`.

---

## Section 2 — Account disabling — runtime enforcement

### Does mobile read the `disabled` claim from the Firebase JWT?

**No.** Mobile does not read any custom claims from the Firebase ID token. The `AuthUserDTO` type (`src/lib/types/user/AuthUserDTO.ts`) does not carry a `disabled` field. The `syncUserToBackend` call (`src/lib/services/authService.ts:125-137`) receives the backend's `AuthUserDTO` response, which on web now includes user state information — but the mobile type definition does not declare `disabled`, `deletionStatus`, or `state`.

### Does mobile have any logic that signs the user out when `disabled` is detected?

**No.** There is no code anywhere in `src/` or `app/` that checks for a `disabled` field, a `USER_BANNED` error code, or any `state` field on the user object.

### Does the axios interceptor handle 401/403 from a disabled-user rejection?

**No.** The response interceptor (`src/lib/config/api.ts:33-62`) handles only:
- `ERR_NETWORK` / `ECONNABORTED` → connection timeout error
- No response → network unreachable error
- 404 → not found error

There is **no handling for 401 or 403**. A 403 `USER_BANNED` response from the backend would propagate as an unhandled axios error to whatever call site made the request. Each call site handles errors independently (most swallow with `logServiceError` or `console.error`), but none inspect the error body for `USER_BANNED` or trigger a sign-out.

### Does mobile call `revokeRefreshTokens` or equivalent?

**No.** Mobile has no equivalent. It relies entirely on the server-side `revokeRefreshTokens` call made by the backend during the ban flow (spec §3.5 step 4). After revocation, the Firebase SDK will fail to refresh the ID token when it expires (within ~1 hour). The `onAuthStateChanged` listener would then fire with `null`, which sets `user: null` in the auth store. However, during the window where the existing ID token is still valid, **mobile has no defense**.

### Confirming: no explicit 401-retry interceptor exists.

**Confirmed.** The axios response interceptor at `src/lib/config/api.ts:33-62` has no 401 or 403 handling. The request interceptor at `src/lib/config/api.ts:21-30` calls `auth.currentUser.getIdToken()` (no `true` argument — auto-refresh only) on every request, which handles token expiry automatically via Firebase's internal refresh. But explicit 401/403-triggered sign-out does not exist.

**Implication for disabled-user enforcement:** A user banned by an admin will continue to be able to use the app until their current Firebase ID token expires (~1 hour max). During this window:
- The backend's `FirebaseAuthFilter` will return 403 with `USER_BANNED` on every authenticated request.
- Mobile's axios interceptor does not catch this, so every API call will fail silently (each call site logs the error and returns null/false).
- The user will see a broken but not-signed-out experience: API calls fail, data doesn't load, but the app still shows the authenticated UI.
- Eventually (within ~1 hour), the Firebase SDK attempts to refresh the token, fails (refresh tokens are revoked), and `onAuthStateChanged` fires with `null`, setting `user: null`.

**There is no ban-notice dialog on mobile.** Web has `AccountStateDialogsInit` that detects the `USER_BANNED` 403 and shows an explanatory dialog. Mobile has nothing equivalent.

---

## Section 3 — Sign-out behavior

### Where is `logoutFirebase` defined?

`src/lib/services/authService.ts:233-235`.

### What does it actually do?

```typescript
export const logoutFirebase = async (): Promise<void> => {
  await GoogleSignin.signOut();
};
```

The function body contains **exactly one call**: `GoogleSignin.signOut()`. That's it.

**It does NOT call:**
- `auth.signOut()` — Firebase Auth client-side sign-out. **This is the critical missing call.**
- `AsyncStorage.clear()` or any AsyncStorage cleanup.
- Any Zustand store reset (besides `useViewTokenStore.clear()` which is done in the `authStore.logout` caller, not here).
- Push token deregistration (handled separately in `PushNotificationsInit` via user effect).

### Consequence of not calling `auth.signOut()`

Firebase Auth state (`auth.currentUser`) persists across "logouts." After `logoutFirebase()`:
- `GoogleSignin.signOut()` clears the Google Sign-In SDK state, so the user won't auto-sign-in via Google on the next attempt.
- But `auth.currentUser` remains non-null. Firebase considers the user still signed in.
- The `onAuthStateChanged` listener does NOT fire with `null` because Firebase Auth state hasn't changed.
- The axios interceptor continues to attach `Authorization: Bearer <token>` to every request because `auth.currentUser` is still set.

The only thing that makes "logout" appear to work is `authStore.logout()` (`src/lib/store/authStore.ts:121-145`), which:
1. Calls `logoutFirebase()` (Google SDK only).
2. Clears `useViewTokenStore`.
3. Sets `user: null` in the Zustand store.

Setting `user: null` updates the UI (authenticated components check `user` from the store). But Firebase Auth is still alive in the background. The request interceptor reads `auth.currentUser` directly (not the store), so **authenticated API calls continue to work even after "logout"** if any code path bypasses the store's `user` check.

The `initAuthListener` at `src/lib/store/authStore.ts:150-162` listens via `onAuthStateChanged`. Since `auth.signOut()` is never called, this listener never fires with `null` during a user-initiated logout. The listener only fires on app cold start if Firebase's persisted auth has expired.

---

## Section 4 — Sign-in surface

### Sign-in methods supported today

1. **Email/password** — `loginUserFirebase` at `src/lib/services/authService.ts:150-153`. Uses `signInWithEmailAndPassword`.
2. **Google** — `loginWithGoogleFirebase` at `src/lib/services/authService.ts:174-195`. Uses `@react-native-google-signin/google-signin` + `signInWithCredential` with `GoogleAuthProvider`.
3. **Facebook** — `loginWithFacebookFirebase` at `src/lib/services/authService.ts:200-228`. **Entirely commented out, returns `null`.** The function body is a block comment. Confirmed.

### Custom claims from the Firebase ID token

**None.** For all sign-in methods, mobile calls `syncUserToBackend` (`src/lib/services/authService.ts:125-137`) which POSTs to `/auth/firebase-sync`. The backend response is an `AuthUserDTO` object. Mobile does not inspect or extract any custom claims from the Firebase ID token itself — it uses `getIdToken()` only for the Authorization header value. The `AuthUserDTO` type does not include `disabled`, `deletionStatus`, `state`, or any claim-derived field.

---

## Section 5 — Auth state and Zustand wiring

### Where is auth state held?

**Zustand only** — `useAuthStore` at `src/lib/store/authStore.ts`. There is no React Context for auth. The store is persisted to AsyncStorage via `zustand/middleware/persist`, partializing only `{ user }`.

The `AppContext` (`src/components/context/AppContext.tsx`) handles base-site selection and app initialization state, not auth.

### How does mobile react when `onAuthStateChanged` fires with `null`?

`initAuthListener` at `src/lib/store/authStore.ts:150-162`:

```typescript
listenAuthState(async (firebaseUser: FirebaseUser | null) => {
  if (!firebaseUser) return set({ user: null });
  // ...sync...
});
```

When `firebaseUser` is `null`, the store sets `user: null`. This triggers:
- **ChatsInit** (`src/components/init/ChatsInit.tsx:16-19`): `unsubscribeAll()` and `unsubscribeBlocks()` — stops Firestore listeners for chats and blocks.
- **NotificationsInit** (`src/notifications/components/NotificationsInit.tsx:9-13`): `reset()` — clears notification state.
- **PushNotificationsInit** (`src/notifications/components/PushNotificationsInit.tsx:133-145`): when `user` becomes `null`, `currentUserId` goes to `null`, and `unregisterFromPush()` is called — detaches the push token from the backend.
- **InitFavoritesStore** (`src/components/init/InitFavoritesStore.ts:11-15`): `clear()` — clears favorite IDs.

However, the store does **not** clear:
- `useFilterStore` — filter state persists.
- `useCatalogStore` — catalog cache persists.
- `useCardSizeStore` — card size preferences persist (reasonable — these are device-level preferences).
- `useViewTokenStore` — only cleared explicitly in `authStore.logout()`, not by the auth listener.
- `useUploadProgressStore` — not cleared (by design per the comment in `authStore.logout`).
- AsyncStorage entries (auth-storage key `auth-storage`, catalog cache, card-size preferences, i18n cache, base-site selection) — none are cleared.

### Is there a listener for `disabled` claim changes while the app is running?

**No.** Custom claims update when the token refreshes (~1 hour), but mobile has no code that reads custom claims at all. Even if the Firebase SDK refreshes the token and the new token carries `disabled: true` in its claims, mobile would never notice. The `onAuthStateChanged` listener fires when the user signs in or out, but **not** when claims change on an existing session. Mobile would need `onIdTokenChanged` (not used) plus claim-inspection logic (not implemented) to detect a mid-session disable.

---

## Section 6 — Token-refresh path

### Does the axios interceptor ever call `getIdToken(true)` to force a refresh?

**No.** The request interceptor at `src/lib/config/api.ts:27-28` calls:

```typescript
await currentUser.getIdToken()
```

No `true` argument. This relies on Firebase's automatic refresh (the SDK returns a cached token if it's still valid, or refreshes if it's near expiry). It never forces a refresh.

### Is there any code path where mobile forces a token refresh?

**No.** Searching for `getIdToken(true)` across `src/` and `app/` returns zero matches. Mobile never forces a token refresh.

### Firebase ID token lifetime

Firebase ID tokens have a **1-hour lifetime** by default. The Firebase SDK auto-refreshes ~5 minutes before expiry. Mobile relies entirely on this auto-refresh. There is no manual refresh anywhere in the codebase.

---

## Section 7 — Push notification deregistration

### Where is the push token registered?

`src/notifications/lib/pushNotificationRegister.ts:13-43` (`registerForPush`). The function:
1. Gets Expo push token via `Notifications.getExpoPushTokenAsync`.
2. Calls `attachPushTokenToBackend(token, userId)` — `POST /secure/push/token` with `{ token, userId, platform: 'EXPO' }`.

### On sign-out, is the token deregistered?

**Partially.** `PushNotificationsInit` (`src/notifications/components/PushNotificationsInit.tsx:133-145`) reacts to `user` changes. When `user` becomes `null`, it calls `unregisterFromPush()` (`src/notifications/lib/pushNotificationRegister.ts:45-58`), which:
1. Calls `detachPushTokenFromBackend(currentToken)` — `POST /secure/push/token/detach` with `{ pushToken: token }`.
2. Clears module-scoped `currentToken` and `currentUserId`.
3. Removes token refresh and permission-revoked listeners.

**However**, this deregistration depends on the `user` effect in `PushNotificationsInit` firing with `null`. Since `logoutFirebase` does not call `auth.signOut()`, the `onAuthStateChanged` listener does not fire, and `user` in the auth store only becomes `null` because `authStore.logout()` explicitly sets it. The `PushNotificationsInit` component does react to the store's `user` field, so the deregistration does happen on user-initiated logout via the store path.

For admin-initiated bans, the push token deregistration would only happen when Firebase Auth eventually fails to refresh the token (~1 hour) and `onAuthStateChanged` fires with `null`.

### On account deletion, is the token deregistered?

**N/A** — no account deletion flow exists on mobile. If deletion were added, the push token deregistration would need to be part of the flow.

### Active-token-per-device or many-tokens-per-user?

The `pushTokenService.ts` endpoint `POST /secure/push/token` sends `{ token, userId, platform: 'EXPO' }`. The backend manages the token-to-user mapping. From mobile's perspective, there is one Expo push token per device, attached to one user at a time. The `registerForPush` function skips re-registration if the token and userId haven't changed (line 35).

---

## Section 8 — Trust boundary check

### User-lifecycle requests mobile makes today

**1. `POST /auth/firebase-sync`** (sign-in / session sync)
- **Fields sent:** `{ idToken, profileImageKey: null, providerId }`
  - `idToken` — derived from `firebaseUser.getIdToken()` (Firebase SDK).
  - `profileImageKey` — hardcoded `null`.
  - `providerId` — from `firebaseUser.providerData?.[0]?.providerId` (Firebase SDK).
- **Authorization header:** `Bearer ${idToken}` — derived from auth.
- **Trust boundary:** clean. The backend derives the user identity from the verified Firebase ID token, not from any client-supplied user ID.

**2. `POST /secure/user/update`** (profile update)
- **Fields sent:** `{ id, firebaseUid, email, displayName, profileImageKey, shortBio, phoneNumber, regionAndCity, allowPreferenceCookies, allowEmails, allowPromoEmails, allowPhoneCalling }`
  - `id` — from `user.id` (Zustand store, populated by backend's `AuthUserDTO`).
  - `firebaseUid` — from `user.firebaseUid` (Zustand store).
  - Other fields — user input or local state.
- **Trust boundary concern:** `id` and `firebaseUid` are client-supplied on the wire. However, the backend's `POST /secure/user/update` endpoint reads the authenticated user from `SecurityContextHolder` (per Part 11 conventions), so the client-supplied `id` should be ignored server-side. **This should be verified during mobile adoption** — the mobile DTO includes `id` and `firebaseUid` on the update request, which the web client may or may not send.

**3. `POST /secure/push/token`** (push token registration)
- **Fields sent:** `{ token, userId, platform: 'EXPO' }`
  - `token` — from Expo push token API.
  - `userId` — from `user.id` (Zustand store).
  - `platform` — hardcoded `'EXPO'`.
- **Trust boundary concern:** `userId` is client-supplied. The backend should derive the user from the JWT, not trust the client-supplied `userId`. **Potential trust-boundary issue** — depends on whether the backend endpoint validates `userId` against the authenticated principal.

**4. `POST /secure/push/token/detach`** (push token deregistration)
- **Fields sent:** `{ pushToken: token }`
  - `pushToken` — from module-scoped `currentToken`.
- **Trust boundary:** the token itself identifies the device-user binding. Acceptable — deregistering by token is the standard pattern.

**5. Account deletion — N/A.** No deletion endpoint is called. When it is added, the spec (§14.3, auth contract C-5) requires: the server derives the user from the JWT, not from a client-supplied user ID.

---

## Section 9 — Local state cleanup on auth changes

### AsyncStorage entries

| Key | On logout (`authStore.logout()`) | On `onAuthStateChanged(null)` | Assessment |
|-----|----------------------------------|-------------------------------|------------|
| `auth-storage` (persisted Zustand user) | Set to `{ user: null }` via Zustand persist | Set to `{ user: null }` via Zustand persist | Cleaned |
| `access_token` (authStorage.ts) | **NOT cleared** | **NOT cleared** | Stale — but also never written (dead code, see Section 10) |
| `card-size-preferences` | NOT cleared | NOT cleared | **Acceptable** — device-level preference |
| `BASE_SITE` | NOT cleared | NOT cleared | **Acceptable** — device-level preference |
| `SPPL` (push prompt timestamp) | NOT cleared | NOT cleared | **Acceptable** — device-level throttle |
| `translations_*` | NOT cleared | NOT cleared | **Acceptable** — cache, not user data |
| `APP_VERSION_CONFIG` | NOT cleared | NOT cleared | **Acceptable** — app config |
| Firebase Auth persistence (via `getReactNativePersistence(AsyncStorage)`) | **NOT cleared** because `auth.signOut()` is never called | Would be cleared if `auth.signOut()` were called | **BUG** — Firebase Auth session persists across "logouts" |

### Zustand stores

| Store | On logout (`authStore.logout()`) | On `onAuthStateChanged(null)` | Assessment |
|-------|----------------------------------|-------------------------------|------------|
| `useAuthStore` (`user`) | `user: null` | `user: null` | Cleaned |
| `useViewTokenStore` | `clear()` explicitly called | **NOT cleared** | Inconsistent — cleaned on user-initiated logout but not on Firebase-driven sign-out |
| `useChatStore` | Cleaned via `ChatsInit` effect on `user` change | Cleaned via `ChatsInit` effect | Cleaned |
| `useChatBlockStore` | Cleaned via `ChatsInit` effect on `user` change | Cleaned via `ChatsInit` effect | Cleaned |
| `useFavoritesStore` | Cleaned via `InitFavoritesStore` listener | Cleaned via `InitFavoritesStore` listener | Cleaned |
| `useNotificationStore` | Cleaned via `NotificationsInit` effect | Cleaned via `NotificationsInit` effect | Cleaned |
| `useFilterStore` | **NOT cleared** | **NOT cleared** | **Gap** — filter state is not user-scoped so acceptable, but admin-specific filter states (`selectedProductStates`, `selectedModerationStates`) could leak across users |
| `useCatalogStore` | **NOT cleared** | **NOT cleared** | Acceptable — catalog data is public |
| `useCardSizeStore` | **NOT cleared** | **NOT cleared** | Acceptable — device preference |
| `useDialogStore` | **NOT cleared** | **NOT cleared** | Acceptable — transient UI state |
| `useUploadProgressStore` | **NOT cleared** (by design) | **NOT cleared** | Acceptable — bounded by `start()/finish()` lifecycle |

### Push notification subscription

On user-initiated logout: cleaned via `PushNotificationsInit` effect reacting to `user` change → `unregisterFromPush()`.

On `onAuthStateChanged(null)`: same path — `user` in store becomes `null`, `PushNotificationsInit` effect fires.

### In-memory user data

`auth.currentUser` (Firebase Auth) is **NOT cleared** on user-initiated logout because `auth.signOut()` is never called. This is the most significant cleanup gap.

---

## Section 10 — Dead code

### `loginWithFacebookFirebase` (known)

`src/lib/services/authService.ts:200-228`. Entirely commented out, returns `null`. Confirmed.

### `authStorage.ts` — all functions have zero callers

`src/lib/storage/authStorage.ts` exports `setAuthToken`, `getAuthToken`, and `clearAuthStorage`. **None of these are imported or called anywhere in the codebase.** Grep for `authStorage`, `setAuthToken`, `getAuthToken`, and `clearAuthStorage` across `src/` and `app/` shows only the definitions themselves.

The file manages an `access_token` key in AsyncStorage, but mobile's actual auth token flow uses `auth.currentUser.getIdToken()` directly (via the axios interceptor). This file appears to be a leftover from an earlier auth implementation pattern.

### `testConnection` in `firebaseClient.ts`

`src/lib/client/firebaseClient.ts:38-50`. A `testConnection` function that subscribes to the `users` collection and logs document count, then unsubscribes after 5 seconds. The call at line 52 is commented out (`// testConnection()`). Debug utility, dead.

### `console.error` and `console.warn` calls in auth paths

- `authService.ts:47` — `console.error('Image download failed:', err)` in `imageUrlToOglasinoImage`.
- `authService.ts:116` — `console.warn(...)` in `retryWithDelay`.
- `authStore.ts:65` — `console.error('Email login failed', err)` in `login`.
- `authStore.ts:81` — `console.error('Register failed', err)` in `register`.
- `authStore.ts:96` — `console.error('Google login failed', err)` in `loginWithGoogle`.
- `authStore.ts:112` — `console.error('Facebook login failed', err)` in `loginWithFacebook`.
- `authStore.ts:139` — `console.error('Logout failed', err)` in `logout`.
- `authStore.ts:159` — `console.error('Auth listener failed', err)` in `initAuthListener`.

These are ad-hoc debug logging, not the project-standard logger. Per conventions Part 4, these should not exist. Flagged for the general audit (out of scope for this audit).

### `updateUserAuth` in `userService.ts`

`src/lib/services/userService.ts:9-24` — `POST /secure/user/update/auth`. Not called anywhere in the codebase except its own definition. Dead code.

---

## Section 11 — For Mastermind

### Trust-boundary findings

1. **`POST /secure/push/token` receives client-supplied `userId`.** `pushTokenService.ts:3-8` sends `{ token, userId, platform: 'EXPO' }` where `userId` comes from the Zustand store. If the backend trusts this value instead of deriving the user from the JWT, a malicious client could attach their push token to another user's account. **Severity: medium-pending-verification.** The backend endpoint should be audited to confirm it ignores the client-supplied `userId` and uses the authenticated principal. This is the same class of issue as the `reportedUserId` finding in `issues.md` 2026-05-16.

2. **`POST /secure/user/update` sends `id` and `firebaseUid` on the wire.** `userService.ts:36-48` includes `user.id` and `user.firebaseUid` in the request body. If the backend reads `id` from the body instead of from `SecurityContextHolder`, a user could modify another user's profile. **Severity: medium-pending-verification.** Same class as finding 1. Backend endpoint audit needed.

### Cases where a disabled or deleted user could continue using the app

1. **A banned user can continue using the app for up to ~1 hour after server-side ban.** Mobile has no 403 `USER_BANNED` interceptor. The backend rejects every authenticated request with 403, but mobile's axios interceptor does not catch 403. The user sees a degraded experience (all API calls fail silently) but remains in the authenticated UI. The app does not sign the user out or show a ban notice until Firebase's token auto-refresh fails (when the revoked refresh token is attempted). **This is the highest-severity finding.**

2. **After user-initiated "logout," Firebase Auth session persists.** `logoutFirebase()` calls `GoogleSignin.signOut()` only, not `auth.signOut()`. `auth.currentUser` remains non-null. If any code path bypasses the Zustand store's `user` check and reads `auth.currentUser` directly, it will see an authenticated user. The axios interceptor reads `auth.currentUser` directly — so authenticated API calls continue to work. The user's Zustand `user` is `null` (UI shows unauthenticated), but the backend sees authenticated requests. **This is a real leak** — the user appears logged out but their session is still alive.

3. **A deleted user's Firebase Auth session would persist identically to finding 2.** If/when mobile adoption adds the deletion flow, the `auth.signOut()` gap means the deletion could be reversed by a stale SSR-equivalent call (the exact race that the auth contract was written to prevent, though mobile doesn't have SSR).

### Sign-out completeness gaps

1. **`logoutFirebase` does not call `auth.signOut()`.** This is the root cause of findings 2 and 3 above. The fix is one line: add `await auth.signOut()` before `await GoogleSignin.signOut()` in `logoutFirebase`.

2. **`useViewTokenStore` is cleared on user-initiated logout but NOT on `onAuthStateChanged(null)`.** If Firebase Auth fires `null` (e.g., after a ban's token expiry), view tokens from the previous session survive until the next user-initiated logout.

3. **`useFilterStore` is never cleared on sign-out.** Admin-scoped filter state (`selectedProductStates`, `selectedModerationStates`) persists across users. Low severity — filter state is not sensitive, but could confuse a subsequent non-admin user if the device is shared.

4. **AsyncStorage `access_token` key is never cleared.** However, this is moot because the key is never written either (dead code in `authStorage.ts`).

### Divergence from the user-deletion spec

1. **No `disabled` field on `AuthUserDTO`.** The mobile type at `src/lib/types/user/AuthUserDTO.ts` lacks `disabled`, `deletionStatus`, and `state`. These are required for the ban-notice dialog, the post-deletion dialog, and the restoration dialog.

2. **No `state` field on `UserInfoDTO`.** The mobile type at `src/lib/types/user/UserInfoDTO.ts` lacks `state`. This field is needed to render the "Scheduled for deletion" badge and the "Banned" badge in chat headers and user profiles.

3. **No 403 `USER_BANNED` interceptor.** The auth contract (§C-4) specifies the backend returns 403 with `USER_BANNED` for disabled users. Web handles this in a global 403 interceptor that signs out and shows the ban-notice dialog. Mobile has no equivalent.

4. **No `deletionInFlight` flag on `useAuthStore`.** The auth contract (§C-6) specifies this flag to prevent redundant `firebase-sync` POSTs during the deletion flow. Mobile's store doesn't have it.

5. **No `clearFirebaseTokenCookie()` equivalent.** Mobile doesn't use cookies, so this is structurally different. But the equivalent concern (ensuring no stale auth state after deletion) applies — `auth.signOut()` is the mobile equivalent, and it's not called.

6. **`initAuthListener` calls `syncUserToBackend` on every `onAuthStateChanged` emission.** This is the mobile equivalent of `UseTokenRefresh.tsx`. It has no single-flight guard, no `deletionInFlight` check. On cold app start, Firebase may emit multiple `onAuthStateChanged` events, causing duplicate `firebase-sync` POSTs (same class of bug as the web's pre-fix `UseTokenRefresh` double-emission issue in `issues.md` 2026-05-21).

### Cross-feature observations (one line each)

- **Image pipeline audit scope:** `ensureUserInFirestore` writes `displayName` to Firestore `users/<uid>` — the general audit flagged this collection's `allow read: if true` was tightened to owner-only. Mobile's Firestore write still works (owner writing their own doc), but any read of other users' Firestore docs will fail post-rule-change.
- **Consent mode v2 scope:** `app/owner/dashboard/user.tsx` still renders `allowPreferenceCookies` toggle and sends it to the backend. The backend removed this field in the Consent Mode v2 feature. Mobile is sending a field the backend no longer reads.
- **Backend calls reduction scope:** `authService.ts:125-137` calls `getIdToken()` and then also passes `idToken` in the request body to `/auth/firebase-sync`. The interceptor also attaches the token as an Authorization header. The token is sent twice per request.
- **Admin removal scope:** `EnableDisableButton.tsx` calls `disableUser`/`enableUser` but does not send `banReason`. The backend's updated ban flow expects an optional `banReason`. Low severity — the field is optional.

### Questions whose answers would change scope of a future mobile-user-deletion adoption chat

1. Does the backend's `POST /secure/push/token` endpoint validate the `userId` parameter against the authenticated principal, or trust it from the body?
2. Does the backend's `POST /secure/user/update` endpoint read `id`/`firebaseUid` from the request body, or derive the user from `SecurityContextHolder`?
3. Should mobile adopt the `initAuthListener` → `syncUserToBackend` cascade with a single-flight guard before or during the deletion adoption, or as a separate brief?
4. Should mobile add a global 403 interceptor (for `USER_BANNED` and potentially other codes) as a prerequisite to the deletion feature, or can the deletion feature ship first?

---

## Section 12 — Cleanup performed

`none needed` — read-only audit.

---

## Section 13 — Obsoleted by this session

`nothing` — read-only audit.

---

## Section 14 — Conventions check

- Part 4 (cleanliness): N/A this session.
- Part 6 (translations): N/A this session.
- Part 7 (error contract): touched — reviewed 401/403 handling (or lack thereof) in the axios interceptor against the Part 7 error shape. Mobile does not implement the error contract's `USER_BANNED` code handling.
- Part 11 (trust boundaries): touched — explicit check in Section 8. Two pending-verification findings on `POST /secure/push/token` and `POST /secure/user/update`.
- Other parts touched: Part 4a (simplicity) — N/A, no code changes. Part 4b (adjacent observations) — five cross-feature observations flagged in Section 11.
