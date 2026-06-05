# Audit — Notifications (backend current state)

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-02
**Mode:** read-only audit (Phase 2). Code is ground truth; no prior notification reports/specs were read.

**Headline:** A working notification pipeline already exists. It writes an in-app
notification to Firestore *and* fans out push to two platforms (Firebase Web Push +
Expo). It is wired to three live triggers today (favorite, admin review
approve/disapprove, admin report resolve). Of the brief's four target events, **only #2
(product favorited) is wired**; #1 (new message) and #4 (admin bans product) have **no
server-side hook at all** (neither event is processed server-side today), and #3 (new
follower) has a clean hook point but fires nothing. There are **two real defects** in the
Firebase Web Push path (sections 1 and 6) and **one Part 11 trust-boundary violation** on
the push-token endpoint (section 2).

---

## 1. Notification data model & pipes

### Package `com.memento.tech.oglasino.notifications`

| File | Role |
| --- | --- |
| `dto/NotificationDTO.java` | The in-process payload. Fields: `Long userId`, `String title`, `String description`, `NotificationCategoryId notificationCategoryId`, `NotificationType notificationType`, `Map<String,Object> data`. Internal DTO — never deserialized from a client request. |
| `dto/PushTokenDTO.java` | Request body for `POST /api/secure/push/token`. `@NotEmpty token`, `@NotNull userId`, `@NotNull platform`. **See section 2 — `userId` is client-supplied.** |
| `dto/DetachPushTokenDTO.java` | Request body for `POST /api/secure/push/token/detach`. `@NotEmpty pushToken`. |
| `enums/NotificationCategoryId.java` | `MESSAGE, SAVED_PRODUCT, PRODUCT_EXPIRATION, PRODUCT_EXPIRED, NAVIGATION, INFO` |
| `enums/NotificationType.java` | `NORMAL, SUCCESS, DANGER, WARNING` |
| `service/NotificationsService.java` (+ `impl/DefaultNotificationsService.java`) | The single notification-creation service. One method: `@Async void sendAsyncNotification(NotificationDTO)`. |
| `service/PushService.java` | Push-platform SPI: `getPushServicePlatform()` + `sendPush(PushToken, NotificationDTO)`. Two impls, selected per token platform. |
| `service/impl/DefaultFirebasePushService.java` | `FIREBASE` platform → Firebase Admin SDK **Web Push** (`FirebaseMessaging.getInstance().send(...)` with `WebpushConfig`). |
| `service/impl/DefaultExpoPushService.java` | `EXPO` platform → HTTP POST to `https://exp.host/--/api/v2/push/send`, then receipt check at `.../getReceipts`. |
| `service/PushTokenService.java` (+ `impl/DefaultPushTokenService.java`) | CRUD over the Postgres `push_token` table. |
| `service/NotificationConstants.java` | Shared param-name string constants (`title`, `data`, `type`, `categoryId`, `body`). |

`NotificationCategoryId` and `NotificationType` are **enums, not** entities — they are not the
DB-table-with-id "NotificationCategoryId" the brief's wording anticipated. There is no
notification-category table.

### `NotificationCategoryId` value usage (live vs. stub)

| Value | Referenced by live code? |
| --- | --- |
| `SAVED_PRODUCT` | **Live** — `DefaultFavoriteProductFacade:116` |
| `NAVIGATION` | **Live** — `DefaultAdminReportFacade:108`, `DefaultAdminReviewService:94` and `:120` |
| `MESSAGE` | **Stub** — zero callers |
| `PRODUCT_EXPIRATION` | **Stub** — zero callers |
| `PRODUCT_EXPIRED` | **Stub** — zero callers |
| `INFO` | **Stub** — zero callers |

`NotificationType`: `SUCCESS`, `NORMAL`, `WARNING` are used; `DANGER` is a zero-caller stub.

### Is there a notification-creation service? What does it write to?

Yes — `DefaultNotificationsService.sendAsyncNotification` (`@Async`, enabled via
`config/AsyncConfig.java @EnableAsync`). It writes to **both** stores:

1. **Firestore** (in-app notification): document added at
   `notifications/{firebaseUid}/userNotifications/{autoId}` with fields `title`,
   `description`, `seen=false`, `createdAt=serverTimestamp`, `type`, `categoryId`, `data`.
   `firebaseUid` is resolved server-side from `userId` via
   `UserService.getFirebaseUidForUserId` (`DefaultUserService:98` →
   `userRepository.findFirebaseUidForUserId`). Clients read this collection **directly from
   Firestore** — there is **no backend GET endpoint** for the notification list.
2. **Push** (fan-out): loads all `PushToken` rows for the user
   (`pushTokenService.getUserTokens(userId)`), then for each token picks the matching
   `PushService` by `platform` and calls `sendPush`. Unknown platform → logged error, skipped.

### Trace of one notification (favorite → delivery)

`POST /api/secure/favorite` (handled by `DefaultFavoriteProductFacade.addRemoveFromFavorites`)
→ on an *add* it calls `notifyUserOnSavedProduct(product, lang)` (`:59`) → builds a
`NotificationDTO` with `userId = product.getOwner().getId()` (`:107`), title/description from
`BACKEND_TRANSLATIONS` seeds (`notif.product.saved.*`), `SAVED_PRODUCT` /`SUCCESS`, and
`data = {productId, productName}` → `notificationsService.sendAsyncNotification(...)` (`:120`)
→ writes the Firestore doc under the owner's firebaseUid → fans out to the owner's push
tokens (Web Push and/or Expo).

### FCM / push-sending code

Yes, and it is wired and live:

- **`DefaultFirebasePushService`** uses `com.google.firebase.messaging.FirebaseMessaging`
  with `WebpushConfig` + `WebpushNotification` — i.e. **Web Push only** (browser), not native
  FCM data/notification messages for Android. It is the `FIREBASE` platform branch. On
  `UNREGISTERED`/`INVALID_ARGUMENT` it self-heals by deleting the bad token. **Defect:** see
  section 6 — it throws on any notification with `data == null`.
- **`DefaultExpoPushService`** is the `EXPO` platform branch (Expo push service over HTTP,
  with receipt polling and `DeviceNotRegistered` token cleanup).

Both are invoked by `DefaultNotificationsService`. The only `FirebaseMessaging` usage in the
repo is in `DefaultFirebasePushService`.

---

## 2. Push tokens

### Endpoint `POST /api/secure/push/token` — `PushTokenController.createOrUpdatePushToken`

- **Storage:** Postgres. Entity `entity/PushToken.java` → table `push_token` (default JPA
  naming; `BaseEntity` id). Columns: `token` (`unique=true, nullable=false`), `user_id`
  (`@ManyToOne`, `nullable=false`, FK to `users`), `platform` (`@Enumerated(STRING)`,
  `EXPO`/`FIREBASE`). Index `idx_push_token_user` on `user_id`. Repo
  `PushTokenRepository` (`findAllByUserId`, `findByToken`, `deleteByUserId`).
- **Keyed by:** the `token` string is the natural upsert key
  (`createOrUpdatePushToken` looks up `findByToken`, inserts if absent else rewrites
  `user`+`token`). Read path for sending is `findAllByUserId`.
- **`userId` validated against the authenticated principal? NO — Part 11 VIOLATION.**
  `PushTokenDTO.userId` is taken straight from the request body and used as the FK
  (`DefaultPushTokenService.createNewPushToken:76` /
  `updateExistingPushToken:88` → `entityManager.getReference(User.class, dto.getUserId())`).
  Nothing reads `SecurityContextHolder` / `CurrentUserService`. The controller has no
  ownership check.
  **Impact:** a caller can POST `{token: <their device token>, userId: <victim id>}` and have
  the victim's notifications (which route by `getUserTokens(victimId)`) delivered to the
  attacker's device — an information-disclosure path. Conversely they can attach an arbitrary
  token string to their own id. The recipient routing key is therefore client-controllable.
  **Recommended resolution:** derive `userId` from
  `currentUserService.getCurrentUserIdStrict()` and drop `userId` from the request DTO (or
  reject when it != authenticated id).

### `POST /api/secure/push/token/detach`

`detachUserFromPushToken(token)` hard-deletes the row by token
(`DefaultPushTokenService.detachUserFromPushToken:42`). No ownership check — any
authenticated caller who supplies a valid token string deletes that row. Low impact
(self-healing on next login; tokens aren't enumerable), but it is the same
trust-boundary class as the create path. Flag, not block.

### Push-token rows in Postgres / code that reads them to send

Yes — schema above. The only reader for sending is
`DefaultNotificationsService.sendAsyncNotification` via `pushTokenService.getUserTokens`.

### Cross-check: `fcmToken` on a Firestore `users/{userId}` doc?

**No such field exists in this repo.** Repo-wide grep for `fcmToken` / `fcm_token` /
`FcmToken` returns nothing. The backend writes Firestore only under the `notifications`
collection (write: `DefaultNotificationsService`; delete-on-account-deletion:
`DefaultFirestoreUserService.deleteUserNotifications`) and reads the `chats`/userchats
collections for messaging cleanup. **The suspected push-token duplication into a users doc
is not present on the backend side.** (Whether a client writes `fcmToken` into Firestore is
a web/expo question — out of this repo's scope.)

---

## 3. The four events — where each fires server-side

### (1) New message — **no server-side hook**
Messaging is Firestore-first. The backend's only messaging code is
`messaging/service/impl/DefaultMessagingCleanupService.java` (a `@Scheduled` job that
anonymizes/cleans chat docs on user hard-delete) and the **read-only** admin endpoints in
`admin/controller/FirebaseChatController` (`GET .../chats`, `GET .../messages`). There is **no
message-send/persist endpoint** server-side — the backend never observes a message being
sent. A "new message" notification cannot be fired from existing server code; the event lives
entirely client/Firestore-side. (A backend hook would require either a Firestore trigger
the backend can't host today, or a new client→backend ping on send.)

### (2) Product favorited — **wired and live**
`POST /api/secure/favorite` → `DefaultFavoriteProductFacade.addRemoveFromFavorites` → fires
`SAVED_PRODUCT` notification to the product owner on *add only* (`:54-60`). This is the one
target event already implemented end-to-end. (Find the exact favorite route mapping in the
favorite controller; the facade is the hook.)

### (3) New follower — **clean hook point, fires nothing**
`POST /api/secure/follow/{targetUserId}` (`FollowController`) →
`DefaultUserFacade.toggleFollowForUser` → `DefaultFollowService.toggleFollow`
(`service/impl/DefaultFollowService.java:25`). `toggleFollow` returns `true` exactly when a
new follow row is created (`:38-44`), which is the natural firing point. **No
`NotificationsService` call exists there today.** Adding one is a one-call insertion; recipient
is `targetUserId` (the followed user), actor is `currentUserId`.

### (4) Admin bans product → notify owner — **no ban write-path at all**
There is **no code that bans a product.** `ModerationState` has only `APPROVED` and `BANNED`
(`entity/ModerationState.java`); `setModerationState(...)` is called only with `APPROVED`
(`DefaultProductService:107`) and in test import. **`ModerationState.BANNED` is never
assigned anywhere in `src/main`.** The admin product controller
(`AdminProductSearchController`, `/api/secure/admin/products`) is **search-only** (get / list /
autocomplete) — no ban, no moderation-state mutation. Admin product *removal* exists only as
deletion (`DashboardProductController.deleteProduct` for owners; `ProductRemovalJob` for
expiry; both → `DefaultProductService.deleteProduct`), not a ban-and-notify. So event #4
cannot be fired from existing server code because **the ban operation itself does not exist** —
it must be built (an admin endpoint that transitions a product to `BANNED`) before a
notification can hang off it.

> Note: existing admin notification triggers that are *not* in the brief's four but exercise
> the pipe today: admin **review approve/disapprove** (`DefaultAdminReviewService`, 3 sends)
> and admin **report resolve** (`DefaultAdminReportFacade.resolveReport`, opt-in via
> `notifyUser`). They are the closest precedent for an admin-action→owner-notification (#4).

---

## 4. Trust boundaries — recipient identification per event

The recipient `userId` on `NotificationDTO` must be server-derived. Current state:

| Event | Recipient set from | Verdict |
| --- | --- | --- |
| Favorite (#2) | `product.getOwner().getId()` — DB | ✅ server-derived |
| Review approve (existing) | `review.getReviewer().getId()`, `review.getTargetUser().getId()` — DB | ✅ server-derived |
| Review disapprove (existing) | `review.getReviewer().getId()` — DB | ✅ server-derived |
| Report resolve (existing) | `report.getReporter().getId()` — DB | ✅ server-derived |
| New follower (#3, if built) | should be `targetUserId` (path var, validated as FK) | ✅ acceptable — server validates the FK |
| New message (#1) | n/a — no server hook | — |
| Ban product (#4, if built) | must be the product's owner from the DB (`product.getOwner().getId()`), never client-supplied | ⚠️ to enforce when built |

**The one live trust-boundary break is not on a recipient but on the push-token write path**
(section 2): the *routing key* (`PushTokenDTO.userId`) is client-supplied, so even though every
recipient above is server-derived, an attacker can still redirect a victim's push stream to
their own device by registering under the victim's id. Fixing recipient derivation does not
close this; the token endpoint must derive `userId` from auth.

---

## 5. Seams (what the backend assumes; what to verify with counterparts)

- **In-app notification list (web + expo):** backend assumes clients **read Firestore
  directly** at `notifications/{firebaseUid}/userNotifications` and own the `seen` flag /
  ordering by `createdAt`. Verify with web/expo: that they read that exact path and that the
  Firestore rules allow a user to read/update only their own subtree (Firestore-rules repo).
- **Push tokens (web = `FIREBASE` Web Push; expo = `EXPO`):** backend assumes each client
  registers its token via `POST /secure/push/token` with the correct `platform`, and that web
  uses the Firebase Web Push (VAPID/service-worker) token while expo uses an Expo push token.
  Verify: web sends `platform=FIREBASE` + a Web Push registration token (not a native FCM
  token — `DefaultFirebasePushService` builds `WebpushConfig`, so a native FCM token would
  misbehave); expo sends `platform=EXPO` + an `ExponentPushToken`. **Also confirm whether any
  client writes `fcmToken` into a Firestore users doc** (the brief's suspected duplication) —
  the backend does not, so any such write would be client-only and redundant with `push_token`.
- **`data` payload contract:** backend ships `data` maps with keys like `productId`,
  `productName`, `navigation`/`navigate`, `label`. Verify clients read these exact keys for
  deep-linking (note the existing inconsistency in section 6: report uses `navigate`, reviews
  use `navigation`).
- **New message (#1):** backend assumes messaging is fully Firestore-side and never sees a
  send. To notify on new messages, the counterpart needed is either a client→backend "message
  sent" signal or a Firestore-triggered function — neither exists; this is a design decision,
  not a wiring gap.
- **Ban product (#4):** backend has no ban capability. Verify with web admin what "ban a
  product" should mean (set `BANNED` vs delete) so the new admin endpoint + the owner
  notification can be specified together.

---

## 6. Defects found in the existing pipe (current-state facts; flagged for Mastermind)

1. **`DefaultFirebasePushService` throws on any notification with `data == null` (medium).**
   `DefaultFirebasePushService.java:51-57`: when `notification.getData()` is null, `dataMap`
   is initialized to `Map.of()` (immutable), then `dataMap.put(TYPE_PARAM_NAME, ...)` is called
   → `UnsupportedOperationException`, swallowed by the generic `catch (Exception)` at `:95`. So
   **the Web Push path silently sends nothing whenever `data` is null.** Today the only such
   caller is review-disapprove (item 2), but it is a latent trap for every future
   null-`data` notification.
2. **Review-disapprove notification has a null `notificationCategoryId` (medium).**
   `DefaultAdminReviewService.sendFailNotification` (`:149-160`) sets type/title/description
   but never sets `notificationCategoryId` or `data`. On the Web Push path,
   `DefaultFirebasePushService:58` calls `notification.getNotificationCategoryId().toString()`
   → NPE (also swallowed). Combined with defect 1, a Web-Push (web) recipient of a
   review-disapproval gets **no push at all**; the Firestore in-app doc is still written (with
   `categoryId=null`). Expo recipients are unaffected (null tolerated).
3. **Hard-coded recipient language `"rs"` in several admin notifications (low).**
   `DefaultAdminReviewService.sendTargetUserSuccessNotification` and `sendFailNotification`,
   and the button labels in `sendReviewerSuccessNotification`, pass the literal `"rs"` to
   `getBackendTranslation` instead of the recipient's `preferredLanguage` (which
   `sendReviewerSuccessNotification` *does* read for title/description). Non-RS recipients get
   Serbian text for those strings. The favorite and report paths localize correctly.
4. **`data`-key inconsistency (low).** Report uses `"navigate"`
   (`DefaultAdminReportFacade:109`); reviews use `"navigation"`
   (`DefaultAdminReviewService:86,112`). Clients must handle both, or one deep-link breaks.

`BACKEND_TRANSLATIONS` seeds for the existing pipe (`notif.review.*`, `notif.product.saved.*`)
are present in all four locale files (EN/RS/RU/CNR) — 10 keys each — so no seed gap for what's
wired. New events (#1/#3/#4) will need new `BACKEND_TRANSLATIONS` keys.

---

## Summary table — readiness of the four target events

| Event | In-app+push pipe exists? | Server-side trigger exists? | Hook point | Work to wire |
| --- | --- | --- | --- | --- |
| #1 new message | yes | **no** | none (Firestore-side only) | needs a server signal/trigger — design decision |
| #2 favorited | yes | **yes (live)** | `DefaultFavoriteProductFacade:54-60` | already done |
| #3 new follower | yes | no | `DefaultFollowService.toggleFollow:38-44` | one `sendAsyncNotification` call + seeds |
| #4 admin bans product | yes | **no** | no ban write-path; admin product controller is search-only | build the ban endpoint first, then notify owner |
