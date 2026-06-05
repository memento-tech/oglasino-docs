# Audit — Notifications (in-app + mobile push)

**Repo:** `oglasino-expo` · **Branch:** `new-expo-dev` · **Date:** 2026-06-02
**Type:** Phase-2 read-only audit. No code changed. Code is ground truth; no prior specs read.

Scope: the notifications screen, push configuration inventory, the notification data
shape, notification-click / deep-link handling, the four events, and the seams expo
assumes about source + shape.

---

## 0. File inventory (everything notification-related)

| File | Role |
| ---- | ---- |
| `app/(portal)/(secured)/notifications.tsx` | The notifications screen (route `/notifications`, secured tab) |
| `src/notifications/components/NotificationsInit.tsx` | App-wide init: initial load + realtime subscribe, mounted in `AppInit` |
| `src/notifications/components/PushNotificationsInit.tsx` | Push permission prompt, token registration trigger, tap/deep-link handling |
| `src/notifications/lib/pushNotificationRegister.ts` | Permission → Expo push token → backend register; token-refresh + permission-revoked listeners |
| `src/notifications/service/pushTokenService.ts` | `POST /secure/push/token` (attach) and `POST /secure/push/token/detach` |
| `src/notifications/store/useNotificationStore.ts` | Zustand store: `notifications`, `unseenCount`, `lastDoc`, `loading` + actions |
| `src/lib/client/firebaseNotifications.ts` | Firestore client: subscribe / fetch-more / mark-seen / mark-shown; `FirebaseNotification` type + two enums |
| `src/components/navigation/BottomBar.tsx` | Bell tab + unseen-count badge (reads `unseenCount`) |
| `src/components/dialog/dialogs/SoftPushPermissionDialog.tsx` | Soft pre-permission prompt (`DialogId.SOFT_PUSH_PERMISSION_DIALOG`) |
| `src/components/init/AppInit.tsx` | Mounts `<PushNotificationsInit />` and `<NotificationsInit />` |

No notification tests exist anywhere in the repo (searched `*notif*test*` / `*test*notif*`).

---

## 1. THE NOTIFICATIONS SCREEN

### Route / component
- Route: `/notifications` — file `app/(portal)/(secured)/notifications.tsx`, default export
  `NotificationsScreen`. It is the Bell tab in `BottomBar` (`TAB_NOTIFICATIONS =
  '(secured)/notifications'`).
- Renders a `FlatList` of notification cards. Each card: a colored left border keyed off
  `item.type` (SUCCESS green / DANGER red / WARNING amber / else blue, `notifications.tsx:55-64`),
  a timestamp (`item.createdAt.toDate().toLocaleString()`), bold `title`, `description`,
  and an optional action button:
  - `categoryId === SAVED_PRODUCT` → "see product" button → `router.push(getDashboardNormalizedProductUrl(item.data.productId))` (`:77-85`)
  - `categoryId === NAVIGATION` → "see notification" button → `router.push(item.data.navigate)` (`:87-93`)
- Unseen rows get `bg-background-mild`, seen rows `bg-background` (`:69`).
- Empty state + `BackButton` live in `ListHeaderComponent`; an `ActivityIndicator` in
  `ListFooterComponent` while `loading` (`:114-124`).

### Where it reads from
- **Firestore listener**, not backend HTTP and not AsyncStorage. Source collection:
  `notifications/{firebaseUid}/userNotifications`, `orderBy('createdAt','desc')`
  (`firebaseNotifications.ts:50-54, 75-80`).
- The screen itself does **not** load data. It only reads `notifications / loading / lastDoc`
  from `useNotificationStore` via `useShallow` (`:22-28`) and, on mount, opens a **second**
  realtime subscription `subscribe(user.firebaseUid)` (`:32-38`). The data it shows is
  populated by the app-wide `NotificationsInit` (`initialLoad` + `subscribe`), which is
  mounted in `AppInit` and runs regardless of whether the screen is open.

### Read/unread (seen) + mark-as-read
- "Unread" = `seen === false`. The unseen badge in `BottomBar` reads `unseenCount`
  (`BottomBar.tsx:31, 115-127`; capped display "9+").
- Mark-as-read is automatic on screen view, **once per mount**: an effect gated by
  `initialMarkDone` ref collects `notifications.filter(n => !n.seen)` and calls
  `markAsSeen(user.firebaseUid, unseenIds)` (`notifications.tsx:40-51`). There is no
  per-item tap-to-read and no "mark all read" button — opening the screen marks the
  currently-loaded unseen set.
- `markAsSeen` writes `{seen:true}` to each Firestore doc (`firebaseNotifications.ts:93-99`)
  and optimistically flips `seen` in the store (`useNotificationStore.ts:76-85`).

### The "unstable / flickers in and out" symptom — likely cause
Reading the subscription + store wiring, this is **listener re-subscribe driven by `user`
object identity**, compounded by a **wholesale array replacement on every re-subscribe**.
Evidence:

1. **Two effects re-run on every `user` reference change.** `NotificationsInit` depends on
   `[hasHydrated, user]` (`NotificationsInit.tsx:20`); the screen depends on `[user]`
   (`notifications.tsx:38`). The dependency is the `user` *object*, not its id.
2. **`user` is replaced with a brand-new object on routine, content-identical events.**
   `authStore` calls `set({ user: backendUser })` from `onIdTokenChanged` token refreshes
   (`authStore.ts:264`), `ForegroundRevalidationInit` foreground re-sync (`:36`), and every
   login path (`:98,116,130,147`). `backendUser` is a freshly-deserialized DTO each time, so
   even when the logical user is unchanged the reference differs. (There is a same-uid
   dedup window at `authStore.ts:253-259`, so churn is bounded, not eliminated — at minimum
   one swap fires shortly after boot, when the persisted-hydrated user is replaced by the
   first `syncUserToBackend` result.)
3. **Each swap tears down the Firestore `onSnapshot` listener and re-runs `initialLoad`.**
   `initialLoad` does `set({ loading:true })` then **replaces** `notifications` wholesale
   with a fresh `fetchMoreNotifications(uid, null)` of ≤20 docs (`useNotificationStore.ts:40-53`).
   If the user had scrolled and `loadMore`'d past 20, those older rows **vanish** on the next
   swap — the literal "visible one moment, gone the next." During the in-flight gap the list
   also repaints.
4. **A transient `set({ user:null })`** (`authStore.ts:237` when `onIdTokenChanged` briefly
   yields a null firebase user during re-auth, or sign-out) makes `NotificationsInit` hit its
   `!user` branch and call `reset()` (`NotificationsInit.tsx:11-14`) → the array is emptied →
   the screen goes blank → user repopulates → `initialLoad` refills. Same signature.

**Ruling out the brief's other two candidates:**
- *Whole-store subscription* — **not** the cause. Selectors are scoped: the screen uses
  `useShallow` over three slices (`notifications.tsx:22-28`); `BottomBar` selects only
  `unseenCount` (`BottomBar.tsx:31`). No component subscribes to the whole store.
- *Hydration race* — only a minor contributor. The screen returns `null` until
  `_hasHydrated` (`notifications.tsx:100`) and `NotificationsInit` gates on `hasHydrated`.
  The race that exists is the boot-time user swap (hydrated persisted user → fresh backend
  user) that kicks the first re-subscribe/`initialLoad` cycle.

**Contributing structural issues that amplify the flicker:**
- **Double subscription.** `NotificationsInit` subscribes app-wide *and* the screen subscribes
  again on mount. Two live `onSnapshot` listeners write to the same store slice. The screen's
  subscription is redundant — it never calls `initialLoad`, so it relies on the global init for
  data anyway.
- **Realtime path only ever *adds* docs; it never reconciles updates or deletes.** `subscribe`'s
  merge computes `newOnes = notifications not already in state` and returns `state` unchanged if
  none (`useNotificationStore.ts:90-107`). A `seen` flip or a delete made on another device is
  never reflected by the realtime path — only the next `initialLoad` (i.e. the next re-subscribe)
  corrects it, which is itself the flicker event.
- **Mixed page sizes corrupt the cursor.** `initialLoad`/`loadMore` page with `getDocs`
  `limit(20)` (`firebaseNotifications.ts:71-91`); the realtime `subscribe` uses `onSnapshot`
  `limit(10)` (`:50-54`). When any new doc arrives, `subscribe` overwrites `lastDoc` with its own
  10th-doc cursor (`useNotificationStore.ts:103`, `lastDoc ?? state.lastDoc` — its `lastDoc` is
  non-null so it wins). That can make `loadMore` skip or duplicate rows on scroll.
- **`initialLoad` passes `null` to a `startAfter`-based query.** `initialLoad` calls
  `fetchMoreNotifications(uid, null as any)` (`useNotificationStore.ts:43`), and
  `fetchMoreNotifications` always does `startAfter(lastDoc)` (`firebaseNotifications.ts:78`).
  `startAfter(null)` is not the intended "from the top" form; verify on-device it does not
  throw / return an unexpected page. A dedicated first-page query (no `startAfter`) would be
  the clean shape.

### Pagination / load-more
- `FlatList onEndReached` → if `lastDoc && !loading`, `loadMore(user.firebaseUid)`
  (`notifications.tsx:109-113`).
- `loadMore` pages with `startAfter(lastDoc)` `limit(20)`, appends to the tail, recomputes
  `unseenCount` (`useNotificationStore.ts:58-71`). Note `loadMore` does **not** set/clear
  `loading`, so the footer spinner only shows during `initialLoad`, and the `!loading`
  guard in `onEndReached` does not actually throttle `loadMore` (only `initialLoad` toggles
  `loading`). See the cursor-corruption note above for the realtime interaction.

---

## 2. PUSH CONFIGURATION — INVENTORY (configured vs declared-but-inert vs missing)

### Packages (present in `package.json`)
- `expo-notifications` `~0.32.16` ✓
- `expo-device` `~8.0.10` ✓ (gates `Device.isDevice`)
- `expo-constants` `~18.0.13` ✓ (reads `eas.projectId`)
- `@react-native-firebase/app` `^24.0.0` and `@react-native-firebase/analytics` `^24.0.0` ✓
- `firebase` `^12.10.0` (JS SDK — used for Firestore/Auth, i.e. the *in-app* notification source)

### `app.config.ts` — push-relevant declarations (all present)
- `expo-notifications` plugin block (`app.config.ts:108-116`): `icon`
  `./assets/images/logo/notification-icon.png`, `color '#000000'`,
  `defaultChannel 'default'`, `enableBackgroundRemoteNotifications: true`.
- iOS `infoPlist.UIBackgroundModes: ['remote-notification']` (`:52-54`).
- `@react-native-firebase/app` plugin (`:117`) + per-tier `googleServicesFile` for iOS
  (`GoogleService-Info.{prod,preview,development}.plist`) and Android
  (`google-services.{prod,preview,development}.json`) (`:48-72`).
- `expo-build-properties` iOS `useFrameworks: 'static'` (`:118-126`) — the Firebase static-framework requirement.
- `extra.eas.projectId = '382cac59-45fa-420e-ab80-7fc709e6d2f3'` (`:73-76`) — consumed by `getExpoPushTokenAsync`.
- No explicit `aps-environment` entitlement is declared in `app.config.ts`. That is normal —
  EAS injects the APNs entitlement at build time from credentials; it is not a code gap.

### What the push code does today (`PushNotificationsInit.tsx` + `pushNotificationRegister.ts`)
- At module load (`PushNotificationsInit.tsx:12-49`): sets a notification handler (sound +
  badge + banner + list, MAX priority), registers three iOS notification **categories**
  (`SAVED_PRODUCT`, `NAVIGATE`, `MESSAGE` — each with a foreground action button), and an
  Android `default` channel (HIGH importance).
- Permission flow (`:151-183`): on a `user.id` change, if logged out → `unregisterFromPush()`;
  else, only on a real device, a soft-prompt is throttled to once per 5 days
  (`LAST_PROMPT_KEY = 'SPPL'`, `:51,55-64,166-172`). If OS permission is not granted but
  `canAskAgain`, it opens `SOFT_PUSH_PERMISSION_DIALOG`; on accept it sets a flag that triggers
  `registerForPush(user.id)` (`:185-195`). If already granted, it registers immediately.
- `registerForPush` (`pushNotificationRegister.ts:13-43`): re-checks/requests OS permission →
  retrieves **`getExpoPushTokenAsync({ projectId })`** (Expo push token, **not**
  `getDevicePushTokenAsync`/raw FCM) → dedups against `currentToken` → `attachPushTokenToBackend`
  → installs a token-refresh listener and an AppState permission-revoked listener.
- `attachPushTokenToBackend` → `POST /secure/push/token` body `{ token, userId, platform:'EXPO' }`
  (`pushTokenService.ts:3-9`). `unregisterFromPush` → `detachPushTokenFromBackend` →
  `POST /secure/push/token/detach` body `{ pushToken }` (errors swallowed) (`:11-21`).

### Is there a working push path today?
- **Client path is fully coded, not a stub.** Permission prompt, token retrieval, backend
  registration, refresh + revoke listeners, and tap handling are all real implementations.
- **What cannot be confirmed from this repo (the actual gap):** because the app uses the
  **Expo push service** (`getExpoPushTokenAsync`), `getExpoPushTokenAsync` only succeeds on a
  real build if push **credentials are provisioned in EAS** — an APNs key for iOS and FCM (v1)
  credentials for Android uploaded to the Expo project. That provisioning lives in the
  EAS dashboard / Firebase console, **not in the repo**, so it is not verifiable here. The
  backend send-side (does `/secure/push/token` exist; does the backend send via the Expo Push
  API at `exp.host`; does it store the Expo token) is also out-of-repo.
- **Declared-but-inert in code:** the iOS category identifier `NAVIGATE` is registered
  (`PushNotificationsInit.tsx:30`) but never handled — `handleNavigation` branches on
  `NAVIGATION` (`:89`), and `NAVIGATE` is not in the `NotificationCategoryId` enum. No iOS
  category is registered for `PRODUCT_EXPIRATION`, `PRODUCT_EXPIRED`, or `INFO`, though the
  handler branches on the first two. See §4.

### What would be NEEDED to make push work (inventory only, no design)
- EAS push credentials: APNs key (iOS) + FCM v1 credentials (Android) registered against the
  Expo project / bundle ids `com.oglasino[.development|.preview]`. (Out of repo — verify in EAS.)
- A backend that accepts `POST /secure/push/token` (`{token,userId,platform:'EXPO'}`) and
  `POST /secure/push/token/detach` (`{pushToken}`), persists the token, and sends via the Expo
  Push API. (Out of repo — backend audit.)
- Reconcile the category identifier mismatch (`NAVIGATE` vs `NAVIGATION`) and register
  categories for any expiration/info push that should carry action buttons (§4).
- A built binary carrying the `expo-notifications` native module (the 2026-06-01 rebuild is the
  candidate; verify the module is present and a token is actually retrievable on device).

---

## 3. THE NOTIFICATION DATA SHAPE expo expects

### In-app (Firestore document) shape — `firebaseNotifications.ts:32-43`
```ts
interface FirebaseNotification {
  id: string;            // Firestore doc id (injected client-side from doc.id)
  title: string;
  description: string;
  link?: string;         // declared, NOT rendered by the screen (dead field on mobile)
  seen: boolean;         // unread state
  shown: boolean;        // see note — only used by the unused nonShown filter
  createdAt: Timestamp;  // firebase/firestore Timestamp; screen calls .toDate()
  type: NotificationType;        // NORMAL | SUCCESS | DANGER | WARNING  (controls left-border color)
  categoryId: NotificationCategoryId; // MESSAGE | SAVED_PRODUCT | PRODUCT_EXPIRATION | PRODUCT_EXPIRED | NAVIGATION | INFO
  data: any;             // category-specific: { productId } for SAVED_PRODUCT, { navigate } for NAVIGATION
}
```
- Location: `notifications/{firebaseUid}/userNotifications/{id}`.
- `data` is untyped (`any`). The screen reads `data.productId` (SAVED_PRODUCT) and
  `data.navigate` (NAVIGATION). All other categories render title/description with no action.

### Push (tapped-notification) shape — what `handleNavigation` reads
- `notification.request.content.categoryIdentifier` (string) and
  `notification.request.content.data` (`PushNotificationsInit.tsx:125-128`).
- `data` fields consumed: `data.productId` + `data.productName` (SAVED_PRODUCT, → product URL),
  `data.navigate` (NAVIGATION). So the push payload `data` shape expected from the sender is
  `{ productId, productName }` or `{ navigate }` keyed by category.

### Mental compare to web (to reconcile in seam analysis)
- The two enums (`NotificationType`, `NotificationCategoryId`) and the Firestore path are the
  cross-platform contract. Web is the reference platform and presumably renders the same
  Firestore docs. Points to reconcile against web: (a) the enum member sets match exactly
  (esp. whether web emits/handles `INFO`, `PRODUCT_EXPIRATION`, `PRODUCT_EXPIRED`); (b) the
  `data` sub-shape per category (`productId`/`productName`/`navigate`/`link`); (c) whether web
  uses `link` (mobile declares it but ignores it); (d) the `shown` field semantics (mobile's
  `markNotificationAsShown` + `nonShown` filter are dead — see §6).

---

## 4. NOTIFICATION-CLICK / DEEP-LINK HANDLING

- **Foreground/background tap:** `addNotificationResponseReceivedListener` →
  `handleResponse(response, false)` (`PushNotificationsInit.tsx:131-137`).
- **Cold-start tap (`getLastNotificationResponse`):** read once on mount →
  `handleResponse(response, true)` (`:139-149`). This is `expo-notifications`' persisted
  "last response the user ever tapped"; it is never auto-cleared.
- **`handleResponse(response, fromBoot)`** (`:114-129`): reads
  `response.notification.request.identifier`. If `fromBoot`, it compares the identifier
  against the persisted `LAST_HANDLED_RESPONSE_KEY = 'LHNR'` (`:53`) and **returns without
  navigating** if it matches (the dedup). Otherwise it persists the identifier and calls
  `handleNavigation(categoryIdentifier, data)`. In-app taps (`fromBoot=false`) also persist
  the identifier, so a fresh tap is recognized as already-handled on the next cold start.
- **`handleNavigation` branches** (`:75-105`): `MESSAGE` → `/messages`; `SAVED_PRODUCT` →
  product URL via `getNormalizedProductUrl(data.productId, data.productName ?? '')` (guarded on
  `data?.productId`); `NAVIGATION` → `router.push(data.navigate)` (guarded on `data?.navigate`);
  `PRODUCT_EXPIRATION` / `PRODUCT_EXPIRED` → `/notifications`; **`default` → no-op** (explicit
  comment: do not blanket-redirect to `/notifications`, stay on the booted screen / home).

### Current state — recent bug-fix work (matches `issues.md` 2026-06-02 cold-start item)
- The `LHNR` dedup and the narrowed `default` branch are the two fixes from the 2026-06-01
  `bug-batch-3` session for "cold start / reload lands on the notifications screen." The
  symptom was the boot handler re-firing `router.push('/notifications')` off a stale
  `getLastNotificationResponse()` plus a catch-all `default` that blanket-pushed
  `/notifications`. Both are addressed in the current code. On-device confirmation is still
  owed (Ψ) per `issues.md`.

### Observations on the click path (not in scope to fix — flagged)
- **Category id mismatch:** the iOS category registered as `NAVIGATE`
  (`PushNotificationsInit.tsx:30`) is handled nowhere; the handler + in-app render use
  `NAVIGATION`. A `NAVIGATE`-category push would fall to `default` (no-op) and its registered
  action button would do nothing.
- **Two different product-URL helpers:** push-tap uses `getNormalizedProductUrl`
  (`PushNotificationsInit.tsx:84`); the in-app screen button uses
  `getDashboardNormalizedProductUrl` (`notifications.tsx:81`) for the same SAVED_PRODUCT
  category. Verify both land on the intended product surface.
- **Router-readiness:** the cold-start effect calls `router.push` from a mount effect; if the
  navigator is not yet ready at that instant the push can no-op. Worth checking on-device.

---

## 5. THE FOUR EVENTS (favorite / follow / message / ban) — expo side today

Mapping the brief's four events onto the `NotificationCategoryId` enum and the handlers:

| Event | In-app render | Push-tap nav | Notes |
| ----- | ------------- | ------------ | ----- |
| **message** | `MESSAGE` → no action button (title/desc only) | `MESSAGE` → `/messages` | Push category registered (`MESSAGE`) and handled. |
| **favorite** | `SAVED_PRODUCT` → "see product" button → product URL | `SAVED_PRODUCT` → product URL | Closest match to "favorite": notifications about a saved/favorited product. Category registered + handled. |
| **follow** | **nothing** — no `FOLLOW` enum member, no render branch | **nothing** — no handler branch | The app does nothing notification-related for follow today. |
| **ban** | **nothing** in the notifications feature | **nothing** | Ban is handled entirely by the auth lifecycle (`accountBanned` dialog / `USER_BANNED`/`EMAIL_BANNED` in `authStore` + `ForegroundRevalidationInit`), not via a notification category. |

Additional categories the code knows about beyond the four: `PRODUCT_EXPIRATION`,
`PRODUCT_EXPIRED` (push-tap → `/notifications`; no dedicated in-app button), and `INFO`
(no render branch, no handler — renders as a plain card; no push handling).

So today: **message** and **favorite/saved-product** are wired end-to-end (in-app + push-tap);
**follow** and **ban** have no notification-feature surface on expo.

---

## 6. SEAMS — what expo assumes, and what to verify

1. **In-app source is Firestore at `notifications/{firebaseUid}/userNotifications`**, ordered by
   `createdAt` desc. Expo assumes the backend writes docs with exactly
   `{title, description, type, categoryId, data, seen, shown, createdAt}` and that `type` /
   `categoryId` are the enum strings in `firebaseNotifications.ts`. **Verify:** the backend
   notification-write path emits these field names and these enum values (esp. `NAVIGATION`
   not `NAVIGATE`; whether it ever emits `INFO`/`link`).
2. **Firestore security rules** must let the authenticated user read their own
   `notifications/{uid}/userNotifications/*` and `updateDoc({seen:true})` on them
   (`markNotificationsAsSeen`). This lives in `oglasino-firestore-rules` (cross-repo). **Verify**
   read + field-scoped update are allowed and that nothing else (e.g. `shown`) is required.
3. **Push token registration contract.** Backend must expose `POST /secure/push/token`
   (`{token, userId, platform:'EXPO'}`) and `POST /secure/push/token/detach` (`{pushToken}`).
   **Field-name asymmetry to verify:** attach sends `token`, detach sends `pushToken` — confirm
   the backend DTOs match each. Also confirm `userId` is needed/trusted server-side or whether
   the backend derives it from the authenticated identity (Part 11 trust boundary — a
   client-sent `userId` should not be trusted for ownership).
4. **Push send path + payload.** Whoever sends the push (backend) must set
   `categoryIdentifier` to a value the handler knows (`MESSAGE`/`SAVED_PRODUCT`/`NAVIGATION`/
   `PRODUCT_EXPIRATION`/`PRODUCT_EXPIRED`) and a `data` object matching the per-category shape
   (`{productId, productName}` / `{navigate}`). Mobile uses **Expo push tokens**, so the sender
   must use the **Expo Push API** (or Expo's FCM/APNs bridge), not raw FCM/APNs directly.
   **Verify** the send path and the exact category strings.
5. **Push credential provisioning (out of repo).** APNs key + FCM v1 creds in EAS for the
   dev/preview/prod bundle ids. Without these, `getExpoPushTokenAsync` throws on device. This
   is the most likely meaning of the brief's "push believed unconfigured" — confirm in EAS.
6. **Dead/vestigial cross-platform scaffolding to reconcile with web:**
   - `markNotificationAsShown` (`firebaseNotifications.ts:101-107`) has **no consumer**.
   - The `nonShown` parameter of `subscribeToLatestNotifications` (`:48,62-64`) is **never
     passed `true`** by the store, so the `shown` filter is dead; `shown` is written by no
     mobile path.
   - `FirebaseNotification.link` is declared but **never rendered**.
   These three suggest a "shown" / `link` concept that exists on web but was never wired on
   mobile. Reconcile in seam analysis: either mobile should honor `shown`/`link` or these
   should be dropped from the mobile type.

---

## Brief vs reality (one item)

- **Push is more configured than the brief assumes.** Brief says push is "BELIEVED TO BE
  UNCONFIGURED (APNs/FCM setup was deferred to now)." In code, the client path is fully
  implemented (`PushNotificationsInit`, `pushNotificationRegister`, `pushTokenService`) and
  `app.config.ts` declares the `expo-notifications` plugin, the iOS `remote-notification`
  background mode, the RNFirebase plugin, per-tier google-services files, and the EAS
  `projectId`. What is genuinely unconfirmed is **out-of-repo**: EAS/Firebase push credentials
  and the backend send-side. The remaining in-code gaps are small and specific (the
  `NAVIGATE`/`NAVIGATION` category mismatch; missing categories for expiration/info; the dead
  `shown`/`link` scaffolding). Recommend the spec treat the client as "implemented, verify
  credentials + backend" rather than "build from scratch."

---

## Trust-boundary note (conventions Part 11)

`attachPushTokenToBackend` sends a client-supplied `userId` in the `/secure/push/token` body.
Per Part 11 the backend must derive the owning user from the authenticated identity, not trust
the client `userId`, when associating a push token with an account. Flag for the backend
audit / seam analysis. (No expo change implied — this is a contract observation.)
