# Audit — Notifications (web)

**Repo:** oglasino-web
**Date:** 2026-06-02
**Type:** Read-only audit. No code changes.
**Scope:** Brief `audit-notifications` — notifications page/center, data shape, web push, the four events, and seams.

Method: code is ground truth. Every claim below is anchored to `file:line`. Where the
true source of a behavior lives in another repo (backend, Firestore Rules), it is
marked **[needs cross-repo verification]**.

---

## 0. Map of the notification surface

All web-side notification code lives in two places:

- `src/notifications/` — the in-app notification module (hooks, types, push lib, token service).
- `src/components/client/initializers/` — the three client initializers that wire push into the app shell.

```
src/notifications/
  hooks/useNotifications.ts          Firestore listener + paging + mark-seen
  lib/notificationManager.ts         in-app toast dedupe/gating
  lib/notificationActions.ts         categoryId → router.push action
  lib/devicePush.ts                  permission + getToken → backend POST
  lib/fcmClient.ts                   getToken → Firestore (parallel path)
  lib/messaging.ts                   foreground onMessage listener
  service/pushTokenService.ts        BACKEND_API push/token attach+detach
  types/AppNotification.ts           the doc shape the client reads
  types/NotificationCategoryId.ts    'MESSAGE'|'SAVED_PRODUCT'|'PRODUCT_EXPIRATION'|'PRODUCT_EXPIRED'|'NAVIGATION'
  types/NotificationType.ts          'NORMAL'|'SUCCESS'|'DANGER'|'WARNING'

src/components/client/initializers/
  FirebaseWorkerInit.tsx             registers /firebase-messaging-sw.js (mounted in app/layout.tsx)
  ForegroundPushInit.tsx (PushInitializer)  foreground FCM → native Notification + click-nav
  UseTokenRefresh.tsx                onIdTokenChanged → sync → initPushForAuthenticatedUser

public/firebase-messaging-sw.js      background push + notificationclick nav (generated)
public/firebase-messaging-sw.template.js  the edited source for the above
```

Mounting:
- `FirebaseWorkerInit` is rendered in `app/layout.tsx:84` (root, every page).
- `PushInitializer`, `UseTokenRefresh` are rendered inside `AppInit` (`src/components/client/initializers/AppInit.tsx:50,53`), which is mounted from `app/[locale]/layout.tsx`.

---

## 1. The notifications page / center

### Route & component
- **Route:** `/notifications` — `app/[locale]/(portal)/(protected)/notifications/page.tsx`. It is in the `(protected)` route group, so it is auth-gated by the layout, not by the page itself.
- **Component:** `NotificationsPage` (default export, `'use client'`). It calls the hook `useNotificationsStore(router)` (`src/notifications/hooks/useNotifications.ts`) and renders the list.
- A second route exists for manual testing: `app/[locale]/test/notifications/page.tsx` (`TestNotification`). **See §5 — it POSTs to a backend endpoint that `state.md` records as deleted; this page is dead/broken.**

### Source of data — persistent Firestore listener
The page does **not** poll a backend endpoint. It reads a **Firestore subcollection** via a realtime `onSnapshot` listener:

- Collection path: `notifications/{firebaseUid}/userNotifications` (`useNotifications.ts:41,88,113`).
- `firebaseUid` comes from `useAuthStore` (`useNotifications.ts:24-25`).
- Initial load + live updates: `onSnapshot(baseQuery, …)` where `baseQuery` is `orderBy('createdAt','desc'), limit(10)` (`useNotifications.ts:40-73`). `PAGE_SIZE = 10` (`:21`).
- The listener unsubscribes and resets the toast dedupe set on unmount / uid change (`:75-78`).

So the notification store on the web is a **Firestore-backed persistent store**, written by the backend (Admin SDK) and read live by the client. The web client never writes notification documents — only reads them and flips `seen`.

### Read / unseen state
- **Field:** `seen: boolean` on each document (`AppNotification.seen`, read at `useNotifications.ts:56`).
- **Unseen count:** derived client-side from the loaded page — `incoming.filter(n => !n.seen).length` (`useNotifications.ts:68`). **This counts only the first page of 10**, not the true total of unseen across the whole subcollection. The badge therefore caps at the page window. The badge UI renders `'...'` when `unseenCount > 100` (`AuthNotificationButton.tsx:29`), which can never be reached given the 10-item page. **[behavior note — not in scope to fix]**
- **Mark as read ("mark all seen"):** `markNotificationsAsSeen()` (`useNotifications.ts:109-124`) runs a `where('seen','==',false)` query (whole subcollection, not just the page), then `Promise.all` of per-doc `updateDoc(ref, { seen: true })`. It is triggered from the page on mount when `user` is set (`notifications/page.tsx:19-23`). There is no per-notification "mark this one read" — it is mark-all, on page open.
- The realtime listener then re-fires and the badge clears.

### Pagination / load-more
- Yes: `loadMore()` (`useNotifications.ts:84-104`) uses cursor paging — `startAfter(lastDoc)`, `limit(10)`, `getDocs` (one-shot, not live). The page renders a "load more" button gated on `hasMore` (`notifications/page.tsx:65-69`).
- `hasMore` is set `docs.length === PAGE_SIZE` on both initial load (`:66`) and load-more (`:103`).
- **Shape inconsistency between the two read paths** (see §2): the initial `onSnapshot` maps fields explicitly with normalization (`createdAt ?? null`, casts); `loadMore` does `{ id: doc.id, ...doc.data() }` raw with an `as AppNotification` assertion (`:96-99`). Load-more'd rows are not normalized the same way. Cosmetic today but a latent drift if doc fields diverge.

### Two bell buttons (entry points)
- `AuthNotificationButton.tsx` — signed-in bell with the unseen badge; subscribes to `useNotificationsStore(router)` purely for `unseenCount` and routes to `/notifications`. Rendered via `HeaderNavButtons.tsx:34`.
- `NotificationButton.tsx` — signed-out bell; opens the login-options dialog instead. No store.
- **Double Firestore subscription:** because `AuthNotificationButton` and the `/notifications` page each call `useNotificationsStore(router)` independently, when the page is open there are **two live `onSnapshot` listeners** on the same subcollection (the hook is a plain hook, not a shared Zustand store despite the name). Cost/correctness note, not a crash. **[behavior note]**

---

## 2. The notification data shape the web client expects

**Type:** `AppNotification` — `src/notifications/types/AppNotification.ts`:

```ts
interface AppNotification {
  id: string;                       // Firestore doc id
  title: string;
  description: string;
  seen: boolean;
  createdAt: Timestamp | null;      // firebase/firestore Timestamp
  type: NotificationType;           // 'NORMAL'|'SUCCESS'|'DANGER'|'WARNING'
  categoryId: NotificationCategoryId;
  data?: Record<string, any>;       // free-form; consumed per category (see §3/§4)
}
```

- `NotificationType` (`types/NotificationType.ts`) — drives toast style and the page's left-border color (`notifications/page.tsx:35-43`, `notificationManager.ts:32-43`).
- `NotificationCategoryId` (`types/NotificationCategoryId.ts`) — `'MESSAGE' | 'SAVED_PRODUCT' | 'PRODUCT_EXPIRATION' | 'PRODUCT_EXPIRED' | 'NAVIGATION'` — drives click navigation (`notificationActions.ts`).
- **The `data` bag is untyped (`Record<string, any>`).** The fields actually read per category:
  - `SAVED_PRODUCT`: `data.productId` (in-app: `notificationActions.ts:19`), plus `data.productName` in the push handlers (`ForegroundPushInit.tsx:39`, `firebase-messaging-sw.js:50`).
  - `NAVIGATION`: `data.navigate` (`notificationActions.ts:25`, `ForegroundPushInit.tsx:44`, sw `:55`).
  - `MESSAGE`, `PRODUCT_EXPIRATION`, `PRODUCT_EXPIRED`: no `data` fields read; `MESSAGE` routes to `/messages`, the two expiration categories have **no action** (fall through to `undefined`, `notificationActions.ts:31-34`).

**This shape is defined only on the web side; there is no schema/Zod validation of incoming docs.** The doc structure is a contract owned by whoever writes the documents (backend Admin SDK). **[needs cross-repo verification — see §5]**

**Important wire-shape caveat:** the in-app/Firestore path and the FCM push path carry `data` differently. In the Firestore document `categoryId` and `type` are top-level fields; in an FCM `data` message the click handlers read `payload.data.categoryId` / `event.notification.data.categoryId` (string-keyed `data` map). The web reads both, but the field *locations* differ between the two transports. Any backend change must keep both shapes in sync.

---

## 3. Web push

There are three distinct delivery surfaces. All use Firebase Cloud Messaging (FCM web SDK, `firebase/messaging`).

### a) Token registration — TWO parallel paths, TWO different sinks
This is the most important finding for the feature.

**Path 1 — backend (`/secure/push/token`):**
- `devicePush.initPushForAuthenticatedUser(userId)` (`lib/devicePush.ts:29-50`) → requests permission if `default`, gets a token via `getToken({ vapidKey: NEXT_PUBLIC_FIREBASE_VAPID_KEY, serviceWorkerRegistration })` (`:19-27`), then `attachPushTokenToBackend(token, userId)`.
- `attachPushTokenToBackend` → `POST /secure/push/token { token, userId, platform: 'FIREBASE' }` (`service/pushTokenService.ts:3-9`).
- **Called from `UseTokenRefresh.tsx:103`**, after every successful `syncUserToBackend`, inside the single-flight guard. This is the primary, auth-lifecycle-driven registration.
- Detach: `detachPushToken()` (`devicePush.ts:52-65`) → `POST /secure/push/token/detach { pushToken }` (`pushTokenService.ts:11-21`). **[needs cross-repo verification: which sink the backend actually reads to send web push — the `push_token` table or the Firestore `users.fcmToken` field below.]**

**Path 2 — Firestore (`users/{uid}.fcmToken`):**
- `fcmClient.getFcmToken()` (`lib/fcmClient.ts`) → requests permission unconditionally, `getToken(...)` with the same VAPID key, returns the token (or `null`).
- `setUserFcmToken(uid, token)` (`authService.ts:203-…`) → `updateDoc(doc(db,'users',uid), { fcmToken: token })` (with create-on-missing fallback). Writes the token into the Firestore **`users` document**, not the backend.
- Called from **two** places:
  - `ensureUserInFirestore` (`authService.ts:94-97`) — on first-time Firestore user creation.
  - The owner settings page notification toggle (`app/[locale]/owner/user/page.tsx:82-95`) — `handleNotificationToggle` writes the token on enable, `null` on disable.

**Seam / risk:** the web registers the FCM token to **two** independent locations (backend `push_token` and Firestore `users.fcmToken`). The two paths use **two near-duplicate token-fetch implementations** (`devicePush.getFirebaseToken` vs `fcmClient.getFcmToken`) that differ in permission handling (`devicePush` guards on `Notification.permission`; `fcmClient` always prompts and `console.warn`/`console.error`s). It is unclear which sink the backend's web-push sender uses; if it is the backend table, the Firestore `fcmToken` write is dead weight; if it is Firestore, the `/secure/push/token` POST is. **Must be resolved when scoping the feature.** **[needs cross-repo verification]**

### b) Foreground push — `ForegroundPushInit.tsx` (`PushInitializer`)
- Subscribes via `listenForForegroundMessages` (`lib/messaging.ts:21-26` → `onMessage`), guarded by `isSupported()` (`messaging.ts:8-19`).
- On each foreground FCM message (only if `Notification.permission === 'granted'`, `ForegroundPushInit.tsx:16`), it constructs a **native `new Notification(title, { body, icon:'/logo/dark-oglasino.png', requireInteraction:true, data: payload.data })`** (`:21-26`).
- Click handling (`:29-53`): routes by `data.categoryId` — `MESSAGE`→`/messages`; `SAVED_PRODUCT`→`/product/{productId}/{encodeURIComponent(productName)}` (only if both present); `NAVIGATION`→`stripRoutingLocale(data.navigate)`; default `/notifications`. Then `window.focus()` + `notification.close()`.
- Note: the in-app **toast** path (notificationManager) is gated *off* when push permission is granted (`notificationManager.ts:11-13`), so foreground push and in-app toasts are mutually exclusive by permission state.

### c) Background push — service worker `public/firebase-messaging-sw.js`
- Yes, there is a service worker. Registered by `FirebaseWorkerInit.tsx:11` (`navigator.serviceWorker.register('/firebase-messaging-sw.js', { scope:'/' })`).
- It is **generated** from `public/firebase-messaging-sw.template.js` by `scripts/build-firebase-sw.mjs` (per the template header). The generated copy has the **stage** Firebase config inlined (`firebase-messaging-sw.js:11-15`, `projectId: 'oglasino-stage-49abb'`). **[note: the committed generated SW carries stage credentials; confirm the build/deploy step regenerates it with prod config — out of scope to verify here.]**
- `onBackgroundMessage` (`:20-35`) shows a notification with `actions: [open, dismiss]`, `icon:'/favicon.ico'`.
- `notificationclick` (`:37-79`): dismiss → return; otherwise routes by `data.categoryId` with the same category map as the foreground handler, **except NAVIGATION uses `data.navigate` raw** (`:55-57`) — see §3d. Focuses an existing matching client or `openWindow(url)`.

### d) `data.navigate` / click-nav and locale-prefix stripping
This is the known-issue area. Current behavior is **inconsistent across the three handlers:**

| Handler | NAVIGATION url | Locale stripped? |
|---|---|---|
| In-app toast (`notificationActions.ts:23-29`) | `stripRoutingLocale(data.navigate)` | **Yes** |
| Foreground native (`ForegroundPushInit.tsx:44-47`) | `stripRoutingLocale(data.navigate)` | **Yes** |
| Service worker (`firebase-messaging-sw.js:54-57`) | `data.navigate` (raw) | **No** |

- `stripRoutingLocale` (`src/lib/utils/stripRoutingLocale.ts`) strips a leading routing-locale segment so the wrapped `next-intl` router does not double-prefix (the fix from issues.md 2026-05-22). The two in-app/foreground handlers use the **`next-intl` wrapped router** (`@/src/i18n/navigation`), which re-adds the locale — hence the strip.
- The **service worker cannot import the app router**; it calls `clients.openWindow(url)` with the raw `data.navigate`. If the backend emits a locale-prefixed `navigate` (e.g. `/sr/reports?...`), the SW opens it as-is (works), but if it emits an unprefixed path the SW opens an unprefixed URL that then relies on the proxy/middleware to resolve the locale. The foreground/in-app paths and the SW path will therefore **disagree** when `data.navigate` is locale-prefixed vs not. The SAVED_PRODUCT and MESSAGE branches are unaffected (locally constructed, unprefixed). **This is the locale-prefix seam to nail down: standardize what `data.navigate` contains (prefixed or not) and make the SW consistent with the app handlers.** **[needs cross-repo verification of what the backend actually puts in `data.navigate`]**

---

## 4. The four events from the web side

For favorite / follow / message / admin-ban, **the web does nothing notification-related**. Each is a plain authenticated backend call; the notification (Firestore doc + push) is created backend-side. No web code writes a notification document or sends a push for these.

- **Favorite a product:** `addToFavorites(productId)` → `GET /secure/favorites/addRemove?productId=…` (`src/lib/service/reactCalls/favoriteService.ts:23-25`). No notification side-effect web-side.
- **Follow a user:** `markFollowUser(userId)` → `POST /secure/follow/{userId}` (`src/lib/service/reactCalls/followService.ts:5-23`); only side-effect is `revalidateUserCache`. No notification web-side.
- **Send a message:** the messaging flow writes the chat/message to Firestore (`useChatStore`), no notification doc/push from web. Grep of `src/messages` for `notification` is empty.
- **Admin bans a product:** closest web action is admin report resolution — `resolveReport(...)` → `POST /secure/admin/report/resolve` (`src/lib/admin/lib/service/reportsService.ts:38-52`). No notification web-side. (No direct "ban product" service exists in web beyond report resolution; the admin product list service `productsSearchService.ts` only reads.)

**Conclusion:** notifications for all four events are a backend + Firestore concern. Web's role is purely consumer (listen, display, mark seen, navigate on tap) plus token registration. **[needs cross-repo verification that the backend actually emits docs/pushes for all four — web has no visibility into this.]**

---

## 5. Seams — what web assumes, and what to verify

What the web client assumes about the notification source:

1. **Firestore subcollection contract.** Web listens to `notifications/{firebaseUid}/userNotifications`, ordered by `createdAt desc`, documents shaped as `AppNotification` (§2). It assumes the backend Admin SDK writes here. **Verify:** the backend write path and exact field names/types (esp. `createdAt` as a Firestore `Timestamp`, `seen` default `false`, `type` ∈ the 4 enum values, `categoryId` ∈ the 5 enum values, `data` keys per category). **[needs cross-repo verification — backend repo]**

2. **Firestore Rules allow the client to flip `seen`.** `markNotificationsAsSeen` does client-side `updateDoc(ref, { seen: true })` (`useNotifications.ts:121`). This requires a rule permitting the owner to update `seen` on their own `userNotifications` docs (and ideally *only* `seen`). state.md (2026-05-20) notes the notifications **create** rule is locked `if false` — but the **update** rule for owner `seen`-flip must exist or mark-as-read silently fails. **Verify:** `oglasino-firestore-rules` for `notifications/{uid}/userNotifications` update. **[needs cross-repo verification — firestore-rules repo]**

3. **Push token sink ambiguity (the big one).** Web writes the FCM token to **both** the backend (`POST /secure/push/token`, `platform:'FIREBASE'`) and Firestore (`users/{uid}.fcmToken`). **Verify:** which one the backend's web-push sender reads, so the other can be removed. (§3a) **[needs cross-repo verification — backend repo]**

4. **`data.navigate` prefix convention.** The SW does not strip the locale; the app handlers do (§3d). **Verify:** what the backend puts in `data.navigate` (locale-prefixed or not), then make all three handlers consistent.

5. **FCM `data` vs Firestore field placement.** Push payloads read `payload.data.categoryId`/`.productId`/`.navigate`; Firestore docs carry `categoryId`/`type` top-level and the rest under `data`. **Verify:** backend keeps both transports' shapes aligned.

6. **VAPID key + SW config.** Token fetch depends on `NEXT_PUBLIC_FIREBASE_VAPID_KEY` (`devicePush.ts:24`, `fcmClient.ts:22`) and a registered SW. The committed generated SW (`public/firebase-messaging-sw.js`) carries **stage** Firebase config; the template is the edit source. **Verify:** prod build regenerates the SW with prod config (`scripts/build-firebase-sw.mjs`).

---

## Adjacent observations (Part 4b — not in scope to fix here)

- **Dead/broken test page.** `app/[locale]/test/notifications/page.tsx` POSTs to `/public/notification/test` (`:27,48,63`). `state.md` (2026-05-20, messaging Brief 4) records `/api/public/notification/test` as **deleted backend-side** (`NotificationsControllerTest` + `NotificationTestDTO` removed). This page now calls a removed endpoint — it is dead and will error. It also contains `console.error` and hardcoded Serbian copy. Candidate for deletion. **[verify the endpoint is gone — backend repo.]**
- **Duplicate token-fetch implementations.** `devicePush.getFirebaseToken` and `fcmClient.getFcmToken` are near-identical (`getMessaging` + `serviceWorker.ready` + `getToken` w/ same VAPID). `fcmClient` additionally `console.warn`/`console.error`s. If the feature consolidates the token sink (seam #3), these should collapse into one.
- **`unseenCount` page-window cap** (§1) and **double `onSnapshot` subscription** (§1) — both follow from `useNotificationsStore` being a per-call hook, not a shared store, despite the `…Store` name.
- **`loadMore` shape normalization gap** (§1/§2) — raw spread vs explicit mapping on initial load.
- **No client-side validation** of incoming notification docs (no Zod), consistent with repo convention (server is source of truth), but worth noting given the untyped `data` bag.

---

## What I did not do
- Did not read prior notification specs (brief instruction — code is ground truth).
- Did not verify any backend, Firestore-Rules, or build-script behavior — all such points are flagged **[needs cross-repo verification]** above for Igor to route.
- No code changes.
