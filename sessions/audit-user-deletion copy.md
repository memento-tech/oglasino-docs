# User deletion audit — oglasino-web

**Date:** 2026-05-17
**Branch:** dev
**Repo:** oglasino-web

Read-only audit. No code changes. All findings are state of the code on disk at the time of audit.

---

## 1. Auth flow and session sync

**Firebase init.** `src/lib/config/firebaseClient.ts:1-80`. Singleton app (`getApps().length === 0 ? initializeApp(...) : getApp()`). Exports `auth`, `db`, `storage`, `googleProvider`, `facebookProvider`. Persistence is set lazily by `initFirebaseAuth()` (`browserLocalPersistence`). All seven Firebase config values come from `NEXT_PUBLIC_FIREBASE_*` env vars.

**Login end-to-end.** Entry points are the methods on `useAuthStore` (`src/lib/store/useAuthStore.ts:124-200`): `login`, `register`, `loginWithGoogle`, `loginWithFacebook`. Each delegates to the matching helper in `src/lib/service/reactCalls/authService.ts`:

- `loginUserFirebase` (`authService.ts:143`) — `signInWithEmailAndPassword` → `buildUserSession`.
- `registerUserFirebase` (`authService.ts:148`) — `createUserWithEmailAndPassword` → `buildUserSession`.
- `loginWithGoogleFirebase` (`authService.ts:156`) — `signInWithPopup(googleProvider)` → `buildUserSession`.
- `loginWithFacebookFirebase` (`authService.ts:161`) — `signInWithPopup(facebookProvider)` → `buildUserSession`.

`buildUserSession` (`authService.ts:135-138`) does two things in order: `ensureUserInFirestore(firebaseUser)` (creates the Firestore `users/<uid>` doc if missing, attempting an avatar mirror to R2 from the OAuth `photoURL`), then `syncUserToBackend(firebaseUser)`.

`syncUserToBackend` (`authService.ts:118-130`): fetches `firebaseUser.getIdToken()` (no forced refresh), POSTs the token to `/api/auth/token` via `writeFirebaseTokenCookie` (Next.js route handler at `app/api/auth/token/route.ts:6-29` which sets `firebase_token` as `httpOnly; secure; sameSite=lax`), then POSTs to backend `/auth/firebase-sync` with body `{ allowPreferenceCookies }`. The backend response is typed `AuthUserDTO` and the returned object is spread with `firebaseUid` overwritten from the local Firebase user. The store sets this as `user`.

The store's `login` / `register` / `loginWithGoogle` / `loginWithFacebook` are user-triggered (called from sign-in dialogs). The `initAuthListener` flow at `useAuthStore.ts:226-237` covers refresh: `onAuthStateChanged` → if `firebaseUser`, call `syncUserToBackend` and set `user` in the store. After every login/oauth login (but not register), `initPushForAuthenticatedUser(backendUser.id)` runs.

Additionally, `UseTokenRefresh` (`src/components/client/initializers/UsetTokenRefresh.tsx:9-30`) subscribes to `onIdTokenChanged` and on every change writes the new token to the cookie via `/api/auth/token` AND re-POSTs `/auth/firebase-sync` with `{ allowPreferenceCookies }`. So `firebase-sync` is called on every token rotation, not just at login.

**User state stores.** The single store of record for the authenticated user is `useAuthStore` (`src/lib/store/useAuthStore.ts:91-268`), Zustand `create()`. Fields it holds:

- `user: AuthUserDTO | null` — backend's authoritative user record after sync. `AuthUserDTO` (`src/lib/types/user/AuthUserDTO.ts:4-18`): `id`, `firebaseUid`, `displayName`, `email`, `baseSite`, `regionAndCity`, `profileImageKey`, optional `providerId`, optional preference-toggle booleans, `allowPhoneCalling`.
- `loading`, `error: string | null`.
- `isAdmin`, `isAdminLoaded`, `isAdminLoading` — populated lazily by `loadIsAdmin()` hitting `/secure/admin`.

Cookies: `firebase_token` (httpOnly, set by `/api/auth/token` POST). `globalCookie` for cookie-consent and per-user UI preferences (read via `getGlobalCookie`).

Other relevant stores:
- `useChatStore` (`src/messages/store/useChatStore.ts:94-756`) holds chat data and a `userCache: Record<firebaseUid, UserInfoDTO>` filled by `getUserForFirebaseUid`. Cleared on `clearChatStore`.
- `useChatBlockStore` (`src/messages/store/useChatBlockStore.ts`) holds Firestore-backed user-to-user block lists; subscribed on login (via `ChatsInit`), unsubscribed on logout.
- `useViewTokenStore` (`src/lib/stores/viewTokens.ts`) — chat view-tokens, cleared on logout.

There is no Context for the user. There is no localStorage write of user data by the app (Firebase SDK uses IndexedDB / localStorage for its own persistence per `browserLocalPersistence`).

**Sign-out flow.** `useAuthStore.logout` (`useAuthStore.ts:205-221`) runs in this order:

1. `detachPushToken()` (FCM detach for the current user; errors logged, not thrown).
2. `logoutFirebase()` → `signOut(auth)` (`authService.ts:166-168`). The `onIdTokenChanged` listener in `UseTokenRefresh` fires with `null`, which calls `writeFirebaseTokenCookie(null)` (which DELETEs the httpOnly cookie via the route handler).
3. `notificationManager.reset()`.
4. `useViewTokenStore.getState().clear()` — drops cached chat view tokens.
5. `set({ user: null, isAdmin: false, isAdminLoaded: false, isAdminLoading: false })`.

Note: `useChatStore` and `useChatBlockStore` are NOT cleared inside `logout`. They are cleared by the `ChatsInit`/`ChatsWatcher` initializers reacting to `user === null`. (Not audited in detail here — out of deletion's strict scope, but flagged below as a "watch this when wiring delete-then-redirect.")

---

## 2. Handling of disabled/banned users

There is **no current frontend handling of "this user is disabled / banned" at the auth layer.**

Searches performed for `disabled`, `banned`, `\bban\b`, `blocked`, `userDisabled`, `isDisabled`, `isBanned`, `accountDisabled`, `user-disabled` across `src/` and `app/` returned only:

- Admin tooling that flips a user's disabled state: `src/components/admin/users/EnableDisableButton.tsx:1-68`, `src/components/admin/users/EnableDisableIcon.tsx`, `src/components/admin/users/UsersTable.tsx`, `src/components/admin/users/UserFilters.tsx`. These consume `UserOverviewDTO.disabled: boolean` (`src/lib/types/user/UserOverviewDTO.ts:7`) and call `enableUser`/`disableUser` admin endpoints. They do not affect the disabled user's own session.
- Chat blocking between users: `src/messages/store/useChatBlockStore.ts` (Firestore `userblocks` / `userblocksReverse` collections, plus the FirestoreUser `blocked: string[]` field at `src/lib/types/user/FirestoreUser.ts:6`). This is **user-to-user blocking, not platform-level banning** — affects only `MessageInput` `disabled` (via `Messages.tsx:90` `const blocked = blocking || blockedBy`) and the "blocking" / "blocked.by" notices on the messages page.
- The shadcn `disabled` HTML prop on buttons/inputs — irrelevant.
- `Switch` toggles in `app/[locale]/owner/user/page.tsx` for cookie/email/notification prefs.

The `AuthUserDTO` returned by `/auth/firebase-sync` does **not** carry a `disabled` field (`AuthUserDTO.ts`). `UserInfoDTO` (the public profile DTO) does not carry one either; it has `iamActive` and `isVerified` but neither maps to "this account is banned." Only `UserOverviewDTO` (admin-side) carries `disabled`.

There is no auth-store flow that signs a user out when the backend signals they are banned. There is no UI surface that tells the user "your account has been disabled." There is no client-side route guard that consults a "banned" flag.

---

## 3. Reauth utilities

**Reauthentication helpers — none.** Grep of `reauthenticateWithCredential`, `reauthenticateWithPopup`, `EmailAuthProvider` across `src/` and `app/` returns **zero** matches. `GoogleAuthProvider` and `FacebookAuthProvider` appear only in `src/lib/config/firebaseClient.ts:3-44` for initial sign-in, not for reauthentication.

**Forced-refresh token getter — none.** Grep of `getIdToken(true)` returns **zero** matches. Every `getIdToken()` call (`UsetTokenRefresh.tsx:19`, `authService.ts:119`, `authService.ts:177`) uses the cache-allowed form.

The axios interceptor at `src/lib/config/api.ts:56-76` caches the ID token via `user.getIdTokenResult()` and rotates 1 minute before `expirationTime` — so the token in flight is at most ~59 minutes old. There is no path that asks Firebase for a fresh token mid-session at the application's discretion.

---

## 4. Settings page — `/[locale]/owner/user`

**Page file:** `app/[locale]/owner/user/page.tsx` (363 lines, `'use client'`).

**Sections, in render order (lines 207-360):**

1. Avatar (`<AvatarUpload>`, lines 208-212) — uploads via `uploadImages([avatarFile], 'profile')` on submit (line 151), orphan cleanup if user-record save fails (line 197).
2. Email input (213-219) — disabled if `user.providerId` (i.e., OAuth users can't edit email).
3. Display-name input (220).
4. Short bio textarea (221).
5. Phone-number input (222-226).
6. Marketplace flag + region/city section (227-265) — region and city are read-only here; only updated through `USER_BASIC_DATA_SELECTOR_DIALOG`.
7. Cookie / notification / email / promo-email / phone-calling toggle bank (267-335) — six switches (`required` is read-only ON, the other five are user-editable).
8. Submit + "set/update user location" buttons + error label (336-359).

**Danger zone — does not exist.** No "danger zone" section, no analogous block. The natural slot is at the very bottom of the page, after the "set/update user location" row, inside the existing `div.flex.w-full.flex-col` (line 207) — i.e., outside the cookie/preference bank and below the submit-button cluster, with its own visual separator. There is currently no styled "danger" CSS utility — the page uses neutral `border-border-mild` borders for every section.

**Delete-account button — does not exist anywhere.** Grep for `deleteAccount`, `deleteUser`, `delete-account`, `delete.*account`, `removeAccount`, `DangerZone`, `danger.zone`, `danger-zone` across `src/` and `app/` returns **zero** matches.

**Form structure on this page.** Plain `useState` per field with prop-drilled `onChange={setX}` on each `<Input>` / `<Textarea>` / `<Switch>`. Submit (`saveChanges`, lines 97-204) reads all the `useState`s, optionally uploads an avatar, then POSTs an `UpdateUserDTO` to `/secure/user/update` via `updateUser()`. Not React Hook Form. This matches the convention noted in `CLAUDE.md` ("Forms in this repo are useState with prop-drilled onChange(partial)…") — though this one uses scalar onChange handlers rather than the partial-spread pattern referenced in CLAUDE.md.

The `getUserDetails(user)` call (line 61) returns an `UpdateUserDTO` from `/secure/user/update` (GET) — used to seed the form's initial state.

---

## 5. Profile rendering — "Scheduled for deletion" badge target

**Profile page file.** `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` (90 lines, async server component). It fetches `userDetails: UserInfoDTO` via `getUserForId(userId)` (next-call wrapping `/public/user/{id}` — not audited byte-for-byte but the type is `UserInfoDTO | null`). Renders `<UserDetails userDetails={...} isOnUserPage={true}>` plus the user's products via `<SelectableFilterProductListWrapper>`, then `<ExtraProductsComponent>` for similar.

**Fields rendered.** Inside `<UserInfoBlock>` (`src/components/client/UserDetails.tsx:26-224`):

- `OglasinoAvatar` (profileImageKey, displayName).
- `displayName` (line 117).
- `FollowUserButton` (only when auth resolved and not viewing your own profile).
- `Rating` (line 123).
- `baseSiteOverview.labelKey` translated via `INTRO` (128-130).
- `regionAndCity.city.labelKey` translated via `COMMON_SYSTEM` (132-136).
- `shortBio` (140-142).
- Verification indicator (`isVerified` + `ShieldCheck` / `ShieldX` + translated label) (144-158).
- `activeProducts` count (160-165).
- Action buttons (167-220): "see user products" / "send message" / `ReportButton(ReportType.USER)` / "share product."
- `ReviewsList` (221) — only when `isOnUserPage`.

**Phone number on the public profile.** `UserDetails.tsx` does **not** render `phoneNumber` at all. The phone is fetched lazily by `CallUserButton` (`src/components/client/buttons/CallUserButton.tsx:14-86`) via `getUserPhoneNumber(userId)` → `/secure/user/phoneNumber?userId=…`. The call button is rendered by `ProductFunctions` (`src/components/client/ProductFunctions.tsx:44`), gated by `owner.allowPhoneCalling`. Hiding the phone is enforced server-side (the endpoint is `/secure/…`) and the frontend additionally gates the button on the owner's `allowPhoneCalling` flag (`UserInfoDTO.allowPhoneCalling`). There is no per-call check that the owner is not in deletion grace.

**Existing badge / chip / status-indicator component.** `src/components/shadcn/ui/badge.tsx` exists — a thin wrapper around shadcn's `cva` badge with `variant` support. No application-level `UserBadge` / `StatusBadge` / "scheduled for deletion" component anywhere. The "verified user" / "unverified user" indicator (Shield icons + text) is inline JSX, not a reusable badge.

**Where the badge would attach.** Inside `UserInfoBlock`, between line 121 (the `displayName` row) and line 122 (the `Rating` row), so the badge sits under the name in both desktop and mobile layouts. The `userDetails: UserInfoDTO` is the prop entry point — a new optional field on `UserInfoDTO` (e.g., `scheduledForDeletionAt`) is the natural carrier.

---

## 6. Message page — sender display name and "cannot send" state

**Files.**

- Page: `app/[locale]/(portal)/(protected)/messages/page.tsx` (38 lines) — renders `<Chats>` and `<Messages>` side-by-side (desktop) or one-at-a-time (mobile, switching on `activeChatId`).
- Conversation list: `src/messages/components/Chats.tsx` (not opened — out of strict scope; see "For Mastermind" if needed).
- Single-conversation view: `src/messages/components/Messages.tsx` (272 lines).
- Input: `src/messages/components/MessageInput.tsx` (301 lines).
- Store: `src/messages/store/useChatStore.ts` (757 lines).

**Sender display name resolution.** Each Firestore `Message` carries `senderId` (firebaseUid) and `receiverId`, no embedded display name. Inside `useChatStore.subscribeToMessages` (lines 229-312) and `loadMoreMessages` (lines 314-370), the store enriches each message via `getUserData(uid)` (lines 115-124) which hits `getUserForFirebaseUid(firebaseUid)` → backend `/auth/firebase/{firebaseUid}` and caches the result in `userCache` for the lifetime of the store. The same cache services `Chats` (via `subscribeToChats`, lines 126-171).

**Consequence: if John is renamed or deleted, chat shows the *current* state** (whatever the backend's `/auth/firebase/{uid}` currently returns), not the historical state, because the lookup is keyed on `firebaseUid` and the messages themselves carry no name snapshot. The cache is per-store-instance and persists across re-renders inside a session; a logout-then-login refills it.

A non-existent / deleted user would manifest as `getUserForFirebaseUid` returning `null` (the service returns `null` on 404 without warning per `userService.ts:101-104`). The store would then set `sender: undefined` on every message in that group, which would crash the renderer at `Messages.tsx:225` (`group.sender.firebaseUid === user.firebaseUid`) and at `Messages.tsx:148` (`activeChat.withUser.displayName`). No null-safe fallback for `sender`/`withUser` exists in the current code.

**Cannot-send UI state.** `MessageInput`'s `disabled` prop (`MessageInput.tsx:33`) gates everything: textarea (252), image picker (264), send button (293), `send()` short-circuit (118). It's driven from `Messages.tsx:266`: `disabled={blocked}` where `blocked = blocking || blockedBy` (line 90). The visible notice for the user is two `<p>` elements (lines 246-259) rendered conditionally on `blocking` / `blockedBy`, using `MESSAGES_PAGE` keys `blocking.label` and `blocked.by.label`. **That is the only "cannot send" state today** — there is no equivalent for "the other user has been deleted" or "you cannot send because target account is gone." A new state would slot naturally beside lines 246-259 with a new conditional and a corresponding `disabled` term in the boolean at line 90 (or a wider `cannotSend = blocked || otherUserDeleted` derived in the same place).

**Send-message button + input.** `Messages.tsx:262-268` renders `<MessageInput onSend={(content) => sendMessage(activeChatId, content)} chatId={activeChat?.chatId} disabled={blocked} />`. `sendMessage` is on `useChatStore` (lines 372-603) — the full message-write flow (optimistic UI, Firestore batch on new chat, append on existing chat).

**Report-message and report-user buttons.** The conversation header has a three-item dropdown (`Messages.tsx:151-192`):

- `func.profile` → `router.push('/user/{withUser.id}')`.
- `func.delete` → `onDelete()` (deletes the per-user chat ref from Firestore — soft, not a global delete).
- `func.report` → opens `REPORT_DIALOG` with `reportType: ReportType.USER` and `reportedUserId: activeChat.withUser.id` (lines 165-177).
- Block / Unblock (178-191).

**There is no per-message report.** Only per-user report from this UI. Grep for `ReportType.MESSAGE` returns zero matches; `ReportType.USER` is used here, in `UserDetails.tsx:191`, and in `ReceivedReviewCard.tsx`. The brief says "report-message and report-user buttons … confirm they exist." Report-user exists in this dropdown. **Report-message does not exist on this page.**

---

## 7. Listing visibility on portal pages

**Where `ProductState` is referenced on the frontend** (`ProductState` is the local enum at `src/lib/types/product/ProductState.ts:1-6`: `ACTIVE | INACTIVE | DELETED` — note the brief uses `ProductStatus.INACTIVE`; the actual type is `ProductState.INACTIVE`):

- `src/components/popups/dialogs/DashboardProductFunctionsDialog.tsx:46, 99, 134` — dashboard-only dialog that conditions buttons (e.g., re-activate, edit) on the product's current state.
- `src/components/client/filters/SelectableFiltersWrapper.tsx:45` — passes `productStateOptions={[ProductState.ACTIVE, ProductState.INACTIVE]}` to the dashboard filter (so the owner can filter their own dashboard listings by state).
- `src/components/client/product/UniversalProductCard.tsx:46-60` — renders the "INACTIVE" / "BANNED" red badge **only when `isDashboard` is true**. Portal-mode (`isDashboard=false`) cards never render the badge.
- `src/lib/store/useFilterStore.test.ts:21` — test seed.
- `app/[locale]/owner/products/[productId]/page.tsx:322, 554, 577` — owner's own product-edit page (status pill + re-activation flow).

**On portal pages — no `ProductState.INACTIVE` filtering in the renderer.** Catalog (`app/[locale]/(portal)/(public)/catalog/[[...slugs]]`), search, user-profile listings (`(public)/user/[userId]`), related (`<ExtraProductsComponent>`), and sitemap (`app/sitemap.ts`) all consume `getPortalProducts` / `/public/product/search`. The frontend renders every product it receives. `UniversalProductCard` does not branch on `productState` in non-dashboard mode.

**The backend filter is exhaustive for portal pages.** All five surfaces hit the same `/public/product/search` (with shape `{productsFilter, paging}` per `productsSearchService.ts:74-117`); the frontend filter object does not carry a `productState` constraint for portal queries. The honest contract today is "backend returns only what's public, frontend renders what it gets." No frontend code re-asserts `state === ACTIVE`.

---

## 8. Review-image upload

**Where a review with images is uploaded.** `src/lib/service/reactCalls/reviewService.ts:59-98` (`reviewProduct`). It calls `uploadImages(data.images, 'product', undefined, uploadOptions)` (line 68) — scope is **`'product'`, not `'review'`**. There is no `'review'` scope; `UploadScope` is the literal union `'product' | 'profile' | 'chat' | 'report'` (`src/lib/images/uploadImages.ts:52`). After upload, the call POSTs `{ ...rest, imageKeys: uploadedNewKeys }` to `/secure/review/product`.

**Storage-key naming convention.** Keys are built **server-side**, not on the frontend. The flow (`uploadImages.ts:106-172`):

1. Frontend POSTs `/secure/images/upload-tokens` with `{scope, count, contentTypes, chatId?}` (line 200).
2. Backend returns `tokens: [{token, key, uploadUrl, expiresAt}, …]` — each `key` already includes whatever prefix the backend chose.
3. Frontend PUTs the bytes directly to the returned `uploadUrl` and stores `key` against the entity.

So the frontend has no visibility into the prefix convention (`profile/`, `product/`, `chat/<chatId>/` or whatever) — that's a backend-only concern. The only client-side bit is the `scope` literal it sends; the chat scope additionally passes `chatId`.

**Where a `review-` prefix duplication would slot.** Because keys are produced server-side, the duplication has to happen on the backend — either at token-issue time (issue a token with a `review-`-prefixed key for review scope, and add a `'review'` scope variant) or at review-create time (the `/secure/review/product` endpoint copies the `product/...` objects to `review-...` keys and stores the new keys on the review row). The frontend's contribution is at most: (a) a new `'review'` literal in `UploadScope` (`uploadImages.ts:52`) and (b) calling `uploadImages(data.images, 'review', ...)` from `reviewProduct`. The actual key naming is entirely backend.

---

## 9. Translation key usage

**Library + lookup pattern.** `next-intl` (per `next.config`'s plugin and per imports across the codebase). The convention pattern is `useTranslations(TranslationNamespaceEnum.<NAMESPACE>)` then `t('key.name')`. Namespaces are an enum at `src/translations/types/TranslationNamespaceEnum.ts:1-34` — current values: `COMMON, COMMON_SYSTEM, ERRORS, VALIDATION, PAGING, BUTTONS, INPUT, DIALOG, HEADER, FOOTER, NAVIGATION, INTRO, EXTRA_PRODUCTS, COOKIES, MESSAGES_PAGE, DASHBOARD_PAGES, ADMIN_PAGES, ABOUT_PAGE, FREE_ZONE_PAGE, PRICING_PAGE, METADATA`.

The conventions Part 6 lists 22 namespaces grouped under CORE/UI/LAYOUT/GLOBAL FEATURES/PAGES/META/BACKEND. The frontend enum is missing `BACKEND_TRANSLATIONS` (the bypass namespace) — expected, since the frontend doesn't consume it.

**Translation loading.** `src/translations/lib/translationsCache.ts:23-92` (`getNamespaceTranslations`). Fetches `/public/translations?namespace=X&lang=Y` with `X-Base-Site: tenant`, flattens the `[{key, value}]` array and reshapes dotted keys to nested objects. Wrapped in `unstable_cache` (`['translations-v1']`, `revalidate: 86400`, tag `'translations'`).

`src/i18n/internalRequest.ts:4-21` (`loadAllNamespaces`) iterates `Object.values(TranslationNamespaceEnum)` and calls `getNamespaceTranslations` for each. Called by `src/i18n/request.ts:7-23` (`getRequestConfig`) on every request — but `unstable_cache` covers the actual network calls.

**`common.user.deleted` flow (the candidate key for "Deleted User").**

1. Backend SQL seed (Backend Engineer's responsibility) inserts a row into the translations table with `namespace='COMMON'`, `key='user.deleted'`, `value='Deleted User'` (or locale-equivalent) for each supported `lang`.
2. On first request after deploy, `/public/translations?namespace=COMMON&lang=…` returns the row.
3. `getNamespaceTranslations` flattens it; `toNested` reshapes so the messages tree gets `COMMON.user.deleted = "Deleted User"`.
4. `unstable_cache` stores the result for 24h or until `revalidateTag('translations')`.
5. Any component does `const tCommon = useTranslations(TranslationNamespaceEnum.COMMON); tCommon('user.deleted')`.

The key `common.user.deleted` is not present in the frontend code today (grep returns zero hits). Adding the rendering side is a one-line `tCommon('user.deleted')` at the call site; adding the data side is a Backend Engineer SQL append.

**Caution — key collision.** `COMMON` already has keys like `user.page.not.found.title`, `user.page.no.products.found`, `user.details.verified.user.label`, `user.details.unverified.user.label`, `user.details.active.products`. Adding `user.deleted` (a leaf) alongside any existing leaf or nested key under `user` is fine. But adding both `user.deleted` and `user.deleted.something` would trip the parent/child collision rule (Conventions Part 6 Rule 2) — preferred remediation `user.deleted.label`.

**Display-name consistency.** Every code path the deletion spec touches reads the user's display name from a `displayName` string field:

- Chat sender: `Messages.tsx:148, 225` reads `activeChat.withUser.displayName` and `group.sender.firebaseUid` (display name lookup actually happens on `UserInfoDTO.displayName` for the chat header, and on `group.sender.displayName` via `Message.tsx` if referenced).
- Review author: `ReviewDTO.reviewer.displayName` (`src/lib/types/review/ReviewDTO.ts:1-5`).
- Profile name: `UserDetails.tsx:117` reads `userDetails.displayName` (`UserInfoDTO.displayName`).
- Owner-name on product card / detail flows: indirectly through `UserInfoDTO`-shaped objects.

All three converge on a `displayName: string` field. None of them today reads any deletion-state field. Substituting the rendered display with `tCommon('user.deleted')` at deletion time is straightforward — either the backend swaps the field on the wire (cheapest), or the frontend branches per call site (more refactor). The audit observes that the backend-swap approach requires fewer FE changes because every call site already trusts `displayName` as a final string.

---

## 10. Auth filter on the frontend — protected routes

**Route protection.** Three layouts wrap protected route trees with `<SessionGuard>` (`src/components/client/SessionGuard.tsx:1-59`):

- `app/[locale]/owner/layout.tsx:27` — `<SessionGuard isAdminRoute={false}>`.
- `app/[locale]/admin/layout.tsx:28` — `<SessionGuard isAdminRoute={true}>`.
- `app/[locale]/(portal)/(protected)/layout.tsx:14` — `<SessionGuard isAdminRoute={false}>` (covers `messages`, `favorites`, `notifications`).

`SessionGuard` runs on the client. It awaits `auth.authStateReady()`, then:

1. If `auth.currentUser` is null → `router.replace('/<locale>')`.
2. If `isAdminRoute` and not admin → `router.replace('/<locale>')`.
3. Otherwise sets local `loading=false` and renders children.

**This check runs on mount of the guard** (each navigation that mounts the guard re-runs the effect). It does **not** re-consult the backend per navigation about "is this user still valid." It only checks that a Firebase user exists locally, and for admin routes calls `/secure/admin` (per-React-cache, so cross-request safe — see `src/lib/auth/serverAuthCheck.ts:23-31` for the server variant). There is no FE call that asks "is the auth token holder still permitted in" on every nav.

**Global 403 handling — none.** The axios interceptor (`api.ts:24-46`) handles only `ERR_NETWORK`, `ECONNABORTED` (timeout), missing response, and 404. **403 falls into the generic `Promise.reject(error.response)`** — no global sign-out, no global redirect, no toast. Each service file handles its own 403 (e.g., `reviewService.ts:37-43` for `/secure/review/product` "not eligible", `cache/BackendCachePanel.tsx:37` for admin forbidden). `serviceLog.ts:17-20` downgrades a 403 from `error` to `warn` because of the `/secure/admin` probe.

**Consequence for deletion.** A backend signal that the current token's user is now disabled/deleted (e.g., the planned `account-disabling-enforcement` feature returning 403 from a normally-2xx endpoint) does **not** propagate to a sign-out today. The user would see a per-call error (or, for endpoints that catch+swallow 403, no error at all), then the next page render that mounts `SessionGuard` would still consider them logged in because `auth.currentUser` is still populated. This is the deletion seam the audit is meant to surface (see Seams).

---

## 11. Error handling — codes vs messages

**Wire shape today.** The unified validation envelope is `{errors: [{field, code, translationKey}]}`, per `src/lib/types/product/ProductErrorResponse.ts:1-9`:

```ts
interface ProductFieldError { field: string | null; code: string; translationKey: string; }
interface ProductErrorResponse { errors: ProductFieldError[]; }
```

Parser: `src/lib/utils/parseProductValidationErrors.ts:1-24` — first-error-per-field wins, `field: null` reserved as `SYSTEM_ERROR_KEY = '__system'`. Renderer call sites: `BasicInfoProductDialog.tsx:284-286`, `UploadedProductDialog.tsx:105-108` — they read `validationErrors[field]` (the translationKey) and pass to `tErrors(...)`. Conformant to the conventions: backend ships a code + translationKey; frontend translates via `tErrors`.

**Coverage.** This shape is currently bound to **product validation only** — the type is literally named `ProductErrorResponse`. User-related endpoints (`/secure/user/update`, `/secure/user/update/auth`, `/secure/user/update/region-city`, `/secure/user/phoneNumber`, `/auth/firebase-sync`, `/auth/firebase/{uid}`) all return `boolean | DTO` and the service layer collapses any non-2xx into `false` / `null` with a `logServiceWarn` (e.g., `userService.ts:14-23, 36-50`). There is no `UserErrorResponse` type and no codified user-error code list on the frontend. A `USER_BANNED` / `EMAIL_BANNED` arriving from the backend today would land as `res.errorCode` on `ApiResponse<T>` from `fetchApi.ts:65-71` (server side) or as a generic axios rejection (client side) and would surface as a generic toast or be silently swallowed.

**Where `USER_BANNED` / `EMAIL_BANNED` would surface.** There is **no generic error renderer**. Each call site handles its own. For the deletion-spec endpoints (TBD-named "delete me," "register email" rejection), the natural pattern would be: extend `ProductErrorResponse` into a shared `BackendErrorResponse` (or just use it directly with the same `{field, code, translationKey}` triple) and have the relevant component (the danger-zone confirm dialog for delete, the registration form for ban-on-reregister) consume it through `parseProductValidationErrors` plus an `ERRORS`-namespace `tErrors(translationKey)` lookup.

The simpler path (and the one most consistent with the existing pattern) is: backend sends `{errors: [{field: null, code: 'EMAIL_BANNED', translationKey: 'errors.email.banned'}]}` on the failing registration response; the registration form treats `field: null` errors as `SYSTEM_ERROR_KEY` and renders the translationKey through `useTranslations(TranslationNamespaceEnum.ERRORS)`. Same for `USER_BANNED` on any secure endpoint that wants to surface it. No new shape needed.

---

## 12. Configuration system

**Environment-specific values.** Two mechanisms:

1. **`NEXT_PUBLIC_*` env vars**, read directly via `process.env`:
   - `NEXT_PUBLIC_API_URL` (used by `api.ts:5`, `fetchApi.ts:3`, `translationsCache.ts:28`, `getConfig.ts:7`).
   - `NEXT_PUBLIC_FIREBASE_*` × 7 (used by `firebaseClient.ts:16-22`).
   These are baked into the bundle at build time. Any change requires a redeploy.

2. **Backend-served runtime config**, via `GET /public/config` (`src/configuration/getConfig.ts:3-24`). Returns a `Record<string, string>` cached for 24h (`unstable_cache`-style `next: { revalidate: 86400, tags: ['config'] }`). Consumed via `<ConfigProvider config={await getConfig()}>` in `app/layout.tsx:45-47` and read in client components via `const cfg = useConfig(); cfg('some.key')` (`src/configuration/ConfigProvider.tsx:1-27`).

The runtime-config bag is the natural place for any deletion-related config that the FE needs to display (e.g., a `grace.period.days` constant if the UI shows it without a server date).

**Where the deletion-date would come from.** If the backend's response to "delete me" carries a concrete `scheduledForDeletionAt` ISO timestamp, the FE just renders it (the cleanest contract). If the FE has to compute it (Day 0 + `grace.period.days`), the constant has to be available via either: (a) the backend response embedding it, or (b) `useConfig('grace.period.days')` if the backend already publishes it on `/public/config`. The audit observes that the backend-supplied date is cheaper and removes one trust-boundary hop — the FE would otherwise need a clock and a config value, and a discrepancy between FE clock and server clock would produce a wrong date in the confirmation UI.

---

## Trust boundaries

Per conventions Part 11. Each boundary check is explicit:

- **"Delete my account" request body.** Code does not exist today — there is no `deleteAccount`/`deleteUser` service or call site in the FE. The recommended shape (per the convention) is **a POST with an empty body**, identity carried by the Firebase ID token (already attached automatically by `api.ts:56-76` for `BACKEND_API` requests, and by `fetchApi.ts:38-45` for server-side requests via the `firebase_token` cookie). The FE has **no business sending a `userId`**. If a future brief proposes the FE send `{userId}` in the delete body, that is a Part 11 violation and would be flagged.
- **"Restore by signing in" flow.** The current login flow is `signInWithEmailAndPassword` → `buildUserSession` → `/auth/firebase-sync` with body `{ allowPreferenceCookies }`. The body carries no flags that could short-circuit a state transition — `allowPreferenceCookies` is a UX preference, not a state command. Restore must be a server-side derivation: when `/auth/firebase-sync` runs for a user whose record is in the deletion-pending state, the backend cancels the deletion and returns the now-active user. **The FE never sends a "please restore" payload, and the audit confirms it has no current vector to do so.**
- **Fresh ID token after reauth.** The FE does **not** force-refresh tokens. Every `getIdToken()` call uses the cached form (`UsetTokenRefresh.tsx:19`, `authService.ts:119, 177`); the interceptor caches the token until ~1 minute before its `expirationTime` (`api.ts:66-72`). For the deletion request to carry a token whose `auth_time` claim is fresh (the standard "reauthenticate before destructive action" pattern), the FE **must explicitly call `firebaseUser.getIdToken(true)` after the reauth step**. There is no helper for this today (see Section 3 — no reauth utilities at all). **This is a gap, not a violation — flagging here so the spec/brief that introduces reauth includes the forced refresh.**

No CRITICAL trust-boundary violations were found in the current code, because the delete feature does not exist yet. The audit confirms the existing auth-sync paths do not give the client a vector to misrepresent identity.

---

## Seams

- **Backend's `disabled` flag on `/auth/firebase-sync`.** `AuthUserDTO` has no `disabled` / `bannedAt` / `scheduledForDeletionAt` fields today. The audit assumption is that when the deletion feature ships, `AuthUserDTO` gains at least one such field and the auth store reads it to short-circuit normal navigation (e.g., redirect-to-deletion-pending-page, hide product-create CTA, …). The counterpart backend audit needs to confirm: which field, with what type (`boolean disabled` vs `Instant scheduledDeletionAt`), and which response carries it. The web counterpart hooks then naturally land in `useAuthStore.initAuthListener` (line 226) after `syncUserToBackend` returns.
- **Backend rejecting banned users with 403.** Today the FE's axios interceptor (`api.ts:24-46`) and `FETCH_BACKEND_API` (`fetchApi.ts:46-79`) both treat 403 as a per-call rejection. There is **no global "got 403, sign out" mechanism**. If the planned `account-disabling-enforcement` backend change starts rejecting requests with 403 for disabled users, the FE will not sign them out — the user would see broken UI (failed product create, failed message send) but stay "logged in" until the next reload. The counterpart audit (backend) should confirm: which 403 error code identifies "banned/disabled" specifically (so the FE can distinguish from existing 403s like `/secure/admin` not-admin or `/secure/review/product` not-eligible) and whether the response body uses the `{errors: [{field, code, translationKey}]}` envelope so the FE can introspect.
- **Firebase reauth fresh ID token.** Firebase SDK auto-rotates tokens via `onIdTokenChanged`, but the axios interceptor caches the result of `getIdTokenResult()` for ~59 minutes (`api.ts:53-75`). Calling `firebaseUser.reauthenticateWith...` on Firebase **does** mint a fresh token internally and fires `onIdTokenChanged`, which `UsetTokenRefresh` listens to and which would invalidate the cookie. However, the axios interceptor's `cachedToken` is a module-scoped variable — it is not invalidated by `onIdTokenChanged`. So an in-memory request that fires shortly after reauth could still attach the stale token. **The safe pattern is `await user.getIdToken(true)` immediately before the delete request and pass the resulting token explicitly**, or invalidate `cachedToken` / `tokenExpiry` in `api.ts` on `onIdTokenChanged`. The latter is a wider change; the former is local to the delete flow. The counterpart audit (backend) confirms whether `auth_time` (or `iat`) is the claim the deletion endpoint reads to enforce "recent reauth."
- **"Scheduled deletion date" string source.** The cleanest contract is the backend's delete response returning `{ scheduledForDeletionAt: ISO }`, the FE renders it via `Intl.DateTimeFormat` or `next-intl` date formatting. The alternative (FE computes `now + GRACE_DAYS`) requires `GRACE_DAYS` on `/public/config` and a synchronized clock. The audit recommends backend-supplied (Section 12).
- **"Deleted User" translated string.** The audit assumes this lands as `common.user.deleted` (or any namespaced equivalent) seeded via the backend SQL translations seed and consumed by `useTranslations(TranslationNamespaceEnum.COMMON)`. No existing FE code references the key. The frontend change is a leaf rendering decision; the data lives on the backend.

---

## For Mastermind

**Conventions confirmation.** I read `meta/conventions.md` end-to-end. Part 11 (Trust boundaries) was applied explicitly in the Trust boundaries section above — three boundary checks were performed (delete-me request body, restore-on-login derivation, fresh-token after reauth) with the verdict that no current code violates Part 11 (because the feature doesn't exist) plus three forward-looking conditions a future brief must honor. Part 8 (errors as codes) was applied in Section 11 — the existing `{field, code, translationKey}` envelope is the obvious extension point for `USER_BANNED` / `EMAIL_BANNED`.

**Questions and observations not covered above:**

1. **Chat-sender "deleted user" renderer is fragile.** Confirmed in Section 6: a `getUserForFirebaseUid` returning `null` for a deleted user would crash `Messages.tsx` at `group.sender.firebaseUid` and `activeChat.withUser.displayName`. **Severity: high.** A future deletion-related brief that ships before chat is hardened will hit this. Either the deletion flow ensures the backend's `/auth/firebase/{uid}` keeps returning a placeholder `UserInfoDTO` for deleted users (with `displayName: "<deleted-user-placeholder>"` so the FE renders the translation), or `useChatStore` needs a null-safe pathway for `sender` / `withUser`. Backend-placeholder is the cheaper path and aligns with the "scheduled for deletion" badge concept — the placeholder carries enough fields for the chat header to render.

2. **No per-message report exists.** The brief assumed "report-message and report-user buttons" both exist on the messages page. **Confirming: report-user exists in the conversation-header dropdown; report-message does not exist anywhere.** Grep for `ReportType.MESSAGE` is empty. If the deletion spec relies on per-message reporting surviving the grace period, the spec needs to either drop that assumption or add the report-message UI as in-scope.

3. **`MessagesPage` and the audit-route's `(protected)/messages` layout.** The conversation list (`src/messages/components/Chats.tsx`, 7-line `<Chats />` reference) was not opened. If Mastermind needs the Chats-side detail (e.g., how the list represents a deleted partner, whether the conversation entry disappears), I can fetch it on follow-up — out of strict deletion scope but adjacent.

4. **`AuthUserDTO.firebaseUid` is set client-side**, by spreading the backend response with the local `firebaseUid` (`authService.ts:129`). This is harmless today — the backend has the source of truth, the FE just overlays the value it already trusts (the locally-authenticated Firebase user's UID). But it's a fragile pattern: a refactor that lets `firebaseUid` arrive from the backend would shadow this local value silently. Worth a note for the next refactor; not a deletion blocker. **Severity: low (adjacent observation, Part 4b).**

5. **`UseTokenRefresh` re-syncs every token rotation.** Each `onIdTokenChanged` fires a fresh `firebase-sync` POST. With Firebase's hourly rotation that's one redundant POST/hour per active client; combined with the auth listener doing the same on initial mount, the first session minute does two `firebase-sync` calls back-to-back (`AuthInit` → `initAuthListener` → `syncUserToBackend` AND `UseTokenRefresh` → `onIdTokenChanged` immediately fires once on mount → `firebase-sync`). Not deletion-relevant but worth knowing if Mastermind ever scopes a perf or cache audit. **Severity: low (adjacent observation, Part 4b).**

6. **The deletion confirmation/restore page route.** No `/owner/deletion-pending` (or equivalent) route exists today. Where this page lands is a UX decision out of audit scope, but: a deleted user navigating to `/owner/user` would currently pass `SessionGuard` (because they have a Firebase user) and crash or render half-broken because the backend may reject `/secure/user/update`. The deletion brief needs to either: (a) introduce a wrapper above `SessionGuard` that checks for the deletion-pending state and redirects, or (b) extend `SessionGuard` itself. Implementation-side: `useAuthStore.user` is the obvious carrier — once `AuthUserDTO` gains the field, `SessionGuard` reads it on line 32 alongside `auth.currentUser`.

7. **Convention vs CLAUDE.md form-pattern note.** CLAUDE.md says "Forms in this repo are `useState` with prop-drilled `onChange(partial)` spread-merge." The audited settings page (`/owner/user`) uses **scalar** `useState` and scalar `onChange` callbacks (`setEmail`, `setDisplayName`, etc.) — not the partial-merge pattern. Not a discrepancy to fix; flagging so the deletion brief, if it adds a confirm-dialog form (e.g., type-your-email-to-confirm), can match the actual local pattern rather than the CLAUDE.md description. **Severity: low.**
