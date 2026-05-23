# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-19
**Task:** Audit `oglasino-backend` for everything that touches the messaging feature — confirm or refute backend involvement in the stage Firestore permission-denied defect, and inventory every backend touchpoint to messaging for the comprehensive reference doc.

## Findings

### 1. HTTP endpoints touching messaging

| # | Method + Path | Controller | Auth | Purpose | In start-message flow? |
|---|---|---|---|---|---|
| 1 | `POST /api/secure/push/token` | `PushTokenController.createOrUpdatePushToken` | authenticated (`/api/secure/**`) | Register or update an FCM/Expo push token for the current user. Request DTO `PushTokenDTO` carries `token`, `userId` (`@NotNull`), `platform` (`EXPO` \| `FIREBASE`). Response is empty 200. | No |
| 2 | `POST /api/secure/push/token/detach` | `PushTokenController.detachUserFromPushToken` | authenticated | Hard-delete a push token row (e.g. browser logout). Request DTO `DetachPushTokenDTO` `{pushToken}`. | No |
| 3 | `POST /api/secure/images/upload-tokens` | `ImageTokensController.issueUploadTokens` | authenticated | Issue R2-upload JWTs. When `scope=chat`, calls `FirebaseChatService.isUserMemberOfChat(uid, chatId)` which reads `chats/{chatId}` in Firestore. Throws `NOT_CHAT_MEMBER` 403 if the chat doc is missing or the caller isn't in `users[]`. | Only if the *first user action* is uploading an image rather than sending text. Possible but not in the captured flow. |
| 4 | `POST /api/secure/images/view-tokens` | `ImageTokensController.issueViewToken` | authenticated | Issue R2-view JWT for `private/chats/{chatId}/`. Same membership check as #3. | No (post-send rendering, not start). |
| 5 | `GET /api/secure/admin/chats/user/{userId}/chats` | `FirebaseChatController.getChats` | `ROLE_ADMIN` | Admin reads `chats` collection paginated by `lastUpdated`. Read-only. | No |
| 6 | `GET /api/secure/admin/chats/{chatId}/messages` | `FirebaseChatController.getMessages` | `ROLE_ADMIN` | Admin reads `chats/{chatId}/messages` paginated by `createdAt`. Read-only. | No |
| 7 | `GET /api/public/user?id={id}` | `UserDataController.getUser` | anonymous | Returns `UserInfoDTO`. Used by messaging client to display the other participant's `displayName`, `profileImageKey`, `firebaseUid` — see item 6. | Yes, pre-send (display participant info). Verified non-failing for the defect (HTTP success would not throw a Firestore permission-denied). |
| 8 | `GET /api/auth/firebase/{firebaseId}` | `AuthController.getUserForFirebaseId` | anonymous | Same `UserInfoDTO` shape, lookup by Firebase UID. Same role. | Plausibly invoked depending on web client. |
| 9 | `POST /api/auth/firebase-sync` | `AuthController.firebaseSync` | anonymous | Sign-in: verify ID token, create/restore local user row, return `AuthUserDTO`. | No — this fires on app load, not on Send. |
| 10 | `POST /api/public/notification/test` | `NotificationsControllerTest.sendNotification` | anonymous (dev-test) | Test endpoint that triggers `NotificationsService.sendAsyncNotification` with arbitrary payload. | No |

**Finding (verdict for scope-item 1):** **No backend HTTP roundtrip is required between the web client clicking Send and the Firestore write attempt for a text message.** A text-only "start chat" flow goes web → Firestore directly; backend is *not* in the wire path. The only adjacent case is item 3 above, fired *if* the user attaches an image as the first action — out of scope for the captured error, which is a Firestore listener + write failure, not an HTTP 403 from backend. (status: `correct`, severity: `critical` — primary verdict.)

### 2. Firebase Admin SDK usage

- **Init:** `security/config/FirebaseConfig.java:25-40`. Single `FirebaseApp.initializeApp(options)` at startup, credentials from `app.firebase.credentials-path` (stage/prod: `/run/secrets/firebase.json`; dev: `${FIREBASE_CREDENTIALS_PATH:./secrets-local/firebase-service-account.json}`). No per-request re-init.
- **Auth service usages:**
  - `security/service/impl/DefaultFirebaseAuthService.java:39` — `FirebaseAuth.getInstance().verifyIdToken(idToken)` (every authenticated request via the filter, plus `AuthController.firebaseSync` for the ban-check email lookup).
  - `admin/service/impl/DefaultFirebaseUserService.java:23,35,44,57,70` — `updateUser` (disable/enable), `deleteUser`, `getUser` (existence check), `revokeRefreshTokens`. Called from admin ban (`DefaultUsersFacade.disableUser` `:86-87`), the `AuthController.firebaseSync` banned-email re-registration cleanup, and `DefaultUserDeletionService` hard-delete (`:163,430`). **Not on the chat-start path.**
- **Firestore usages (all via `FirestoreClient.getFirestore()`):**
  - `notifications/service/impl/DefaultNotificationsService.java:51-67` — **write**: `notifications/{firebaseUid}/userNotifications/{auto}` (in-app notification doc). Triggered from `DefaultAdminReportFacade`, `DefaultAdminReviewService` (×3), `DefaultFavoriteProductFacade`. **Never with `NotificationCategoryId.MESSAGE`** — see item 3.
  - `service/impl/DefaultFirestoreUserService.java:24-58` — **write/delete**: `notifications/{firebaseUid}/userNotifications/*` cleanup on user hard-delete. Best-effort.
  - `admin/service/impl/DefaultFirebaseChatService.java:37-160` — **read only**: `chats` (collection scan), `chats/{chatId}` (membership), `chats/{chatId}/messages` (admin viewer).
  - `admin/service/impl/DefaultFirebaseStatsService.java:14-25` — **read only**: `count()` aggregate on `chats` and `notifications`.
  - `service/impl/DefaultTrustReviewService.java:58-72` — **read only**: `chats/{sorted_uid_a_uid_b}/messages` to score review eligibility.
- **Messaging usage (FCM):** `notifications/service/impl/DefaultFirebasePushService.java:77` — `FirebaseMessaging.getInstance().send(message)` for WebPush. The recipient is the FCM web push token registered via endpoint #1.

**Finding (verdict for scope-item 2):** **Backend does not write to `chats/{chatId}` or `chats/{chatId}/messages/{messageId}` — at all.** It only reads them. There is no "both backend and web write the same chat doc" seam. The Firestore writes backend *does* perform live under `notifications/{firebaseUid}/...`, a different top-level collection. (status: `correct`, severity: `critical` — eliminates one seam class.)

### 3. Push notification flow when a user sends a message

- **Backend webhook listening for Firestore writes:** none. There is no Cloud Function trigger, no Firestore listener, no Eventarc handler in this repo.
- **Backend endpoint invoked by sender's client to dispatch push:** none on the messaging surface. `NotificationsService.sendAsyncNotification` has only four call sites (`DefaultAdminReportFacade.java:111`, `DefaultAdminReviewService.java:106,130,160`, `DefaultFavoriteProductFacade.java:120`, plus the dev-only `NotificationsControllerTest.java:31`). None of them carry `NotificationCategoryId.MESSAGE`; all use `NAVIGATION` or `SAVED_PRODUCT`.
- **`NotificationCategoryId.MESSAGE` enum value:** declared in `notifications/enums/NotificationCategoryId.java:4` but has **zero callers** in `src/main/java`.

**Finding (verdict for scope-item 3):** **Backend is not in the push-notification path for chat messages.** The recipient's notification must therefore be either (a) generated client-side from a snapshot listener on `chats/{chatId}/messages`, or (b) generated by an out-of-repo Cloud Function. Cross-repo seam — see "For Mastermind." (status: `missing`, severity: `medium` — not the defect, but a documented gap relative to the user-visible feature.)

### 4. FCM token management

- **Storage:** entity `PushToken` (`entity/PushToken.java:18-53`) — table with unique `token` (varchar), `user_id` FK NOT NULL (`idx_push_token_user`), `platform` enum (`EXPO` / `FIREBASE`).
- **Registration endpoint:** `POST /api/secure/push/token` → `DefaultPushTokenService.createOrUpdatePushToken` (`:56-72`). Idempotent: if token exists, reassigns its `user_id`.
- **Detach endpoint:** `POST /api/secure/push/token/detach` → `detachUserFromPushToken` (`:33-43`). Hard-delete (no NULL-out — `user_id` is NOT NULL).
- **`firebaseMessaging.send()` site:** `DefaultFirebasePushService.java:77`. WebPush only (sets `WebpushConfig`, not `AndroidConfig` / `ApnsConfig`). On `UNREGISTERED` / `INVALID_ARGUMENT` it deletes the token row.
- **Expo path:** `DefaultExpoPushService.java:70-78`. POSTs to `https://exp.host/--/api/v2/push/send`. Receipt-driven token cleanup at `:148-151`.

**Finding (verdict for scope-item 4):** Token model is complete and functional. **Not in the start-message failing path.** (status: `correct`, severity: `low`.)

### 5. `BACKEND_TRANSLATIONS` for messaging-related strings

Grep across `src/main/resources/data/translations/0001-data-web-translations-EN.sql` (the BACKEND namespace seed):

```
2100  notif.review.see.my.button
2101  notif.review.check.success.title
2102  notif.review.check.success.description
2103  notif.review.see.new.button
2104  notif.review.new.title
2105  notif.review.new.description
2106  notif.review.check.fail.title
2107  notif.review.check.fail.description
2108  notif.product.saved.title
2109  notif.product.saved.description
```

`getBackendTranslation(...)` is invoked only from `DefaultAdminReviewService` (review lifecycle) and `DefaultFavoriteProductFacade` (someone favourited your product). **No `notif.message.*` keys exist and no caller resolves a messaging string from `BACKEND_TRANSLATIONS`.** (status: `missing`, severity: `medium` — confirms scope-item 3 verdict from another angle.)

### 6. User data needed for messaging

Two endpoints carry `UserInfoDTO` and are plausibly relied on by the messaging UI to display the other participant:

- `GET /api/public/user?id={id}` (`UserDataController.java:19-24`).
- `GET /api/auth/firebase/{firebaseId}` (`AuthController.java:113-119`).

`UserInfoDTO` (`dto/UserInfoDTO.java:26-50`) fields visible to messaging clients:

- Identity: `id`, `firebaseUid`, `displayName`, `profileImageKey`, `shortBio`.
- Reputation: `rating`, `isVerified`, `activeProducts`.
- Caller-context (populated post-cache by `DefaultUserFacade.applyCallerContext`): **`iamActive`**, **`isFollowingCurrent`**. If a messaging consumer reads these by calling `UserService` directly instead of going through `UserFacade`, they would get stale defaults (both `false`) from the `redisUserInfo` cache — but this is a documented invariant of the DTO (lines 9-20 of the file), enforced by the routing in both controllers.
- Location: `baseSiteOverview`, `regionAndCity`.
- Deletion state: `state` (default `ACTIVE`), `scheduledDeletionAt`.

**Finding (verdict for scope-item 6):** The messaging client has a well-formed read path for participant data. The caller-context fields are inherently per-request and won't affect chat *delivery* either way — they only colour the participant card. (status: `correct`, severity: `low`.)

### 7. Auth flow for Firestore access

- **Token issuance:** backend never issues Firebase ID tokens — those come from Firebase Auth client SDK on the web side. `grep` for `createCustomToken` / `setCustomClaims` / `setCustomUserClaims` across `src/main/java` returns **zero hits**. There is no custom-token endpoint and no claim-setting code.
- **Token refresh:** backend never refreshes the client's token. The web client's Firebase Auth SDK manages refresh.
- **Token verification:** backend verifies the same ID token on its own endpoints (`FirebaseAuthFilter.java:73`). The verification result is cached as `AuthenticatedUserDTO` in Redis (`@Cacheable("redisUserAuth")`, `DefaultFirebaseAuthService.java:64-69`) keyed by `firebaseUid`. Eviction happens on `DefaultUserService.saveUser` — *not* on every request, *not* on every Firestore write.
- **Session cookies / `setHttpOnly`:** none. `SecurityConfig` is fully stateless: `SessionCreationPolicy.STATELESS` (`SecurityConfig.java:71-72`). No `Set-Cookie` headers issued.
- **Token revocation:** `FirebaseAuth.revokeRefreshTokens(firebaseUid)` is wired but only fires from the admin ban path (`DefaultUsersFacade.disableUser` `:87`) and the user-deletion hard-delete (`DefaultUserDeletionService.java:163`). A user visiting their product page on stage does **not** traverse either path.
- **Banned-user 403 short-circuit:** `FirebaseAuthFilter.java:83-86` returns 403 with `USER_BANNED` *before* the chain continues, but the response is an HTTP body — it does not touch the client's Firestore auth.

**Finding (verdict for scope-item 7):** **The auth-token-issue hypothesis is negative.** Backend does not modify, refresh, revoke, or rewrite the Firebase ID token the web client uses against Firestore on the normal product-page → start-chat path. A stale-token scenario would require either (a) an admin disabling the user mid-session, or (b) a deletion flow firing — neither of which fits the captured symptom. The 5-minute Google JWT clock-skew window is also not in play because the error is `permission-denied`, not `unauthenticated`. (status: `correct`, severity: `critical`.)

### 8. Cross-document data on chats

Backend writes only one Firestore tree: `notifications/{firebaseUid}/userNotifications/{auto}`. It never writes:

- `chats/{chatId}` or any descendant
- `users/{anything}` in Firestore (user data is Postgres-only on backend; backend has no Firestore mirror of the user)
- any document a Firestore Rule on `chats/**` would plausibly `get()` or `exists()` against

**Finding (verdict for scope-item 8):** No backend-written Firestore document can be stale or missing in a way that would cause a `permission-denied` on a `chats` write. If the rules `get()` any document during chat creation, that document is either (a) the chat doc itself (client-written), (b) a user-level Firestore doc that this repo never produces, or (c) external — Mastermind's seam analysis will need the firestore-rules audit to resolve. (status: `correct`, severity: `high` — explicit seam to verify against rules repo.)

### 9. Recent git history on messaging-related backend code

Last 60 days, files matching messaging/chat/notification/Firebase code paths:

| Commit | Date | Subject | Files in scope | Severity |
|---|---|---|---|---|
| `f03ea20` | 2026-05-17 | "Resolve issues" | `DefaultTrustReviewService.java` (1 line) | `suspicious-medium` — fixed a self-reference bug `getChatId(currentUid, currentUid)` → `getChatId(currentUid, targetUid)`. Affects the review-eligibility query, **not** chat write or chat listen. 2 days before the defect timestamp; coincident but mechanistically unrelated to `permission-denied` on chat start. |
| `bf715ae` | 2026-05-11 | "Fixing prod 0 products case and adding admin user display name" | `DefaultFirebaseAuthService.java` (+1 line) | `low` — cosmetic admin display-name change. |
| `3cb87ba` | 2026-05-11 | "Admin user enabled and reindex fixed with admin page" | `DefaultFirebaseAuthService.java` (+7 lines) | `low` — admin-only path. |
| `a5132f2` | 2026-05-08 | "IMAGE_PIPELINE_FINISHED" | `DefaultFirebaseChatService.java` (+31), `FirebaseAuthFilter.java` (+2) | `medium` — added `isUserMemberOfChat` to back the new `/api/secure/images/upload-tokens` membership check. New Firestore *read* path on the chat-image upload flow. **If** a brand-new chat triggers an image-upload-token request before the chat doc exists in Firestore, this returns 403 `NOT_CHAT_MEMBER` — but that surfaces as an HTTP 4xx, not a Firestore listener `permission-denied`. |
| `db2e72f` | 2026-05-06 | "feat: cache admin, request logging, redisUserAuth, translation seed" | `FirebaseAuthFilter.java` (±23), `FirebaseAuthService.java` (+8), `DefaultFirebaseAuthService.java` (+19) | `medium` — introduced the `redisUserAuth` Redis cache for `AuthenticatedUserDTO`. Stale-cache hypothesis: a stale cached entry could mis-route admin-vs-user permission on the *backend*, but cannot affect Firestore directly. |
| `1ffdaea` | 2026-05-05 | "ci(format): introduce Spotless with Google Java Format + sortPom" | `FirebaseConfig.java` (formatting) | `low` — pure formatting. |
| `9a4a3cf` | 2026-05-05 | "Production-readiness: Flyway, rate limiting, catalog sync, CI workflows" | `FirebaseConfig.java` (+52, -14) | `medium` — refactored `FirebaseConfig` to load credentials by path from a profile property (instead of classpath resource). Path-resolution failure here would prevent `FirebaseApp` from initialising at all, which would surface as 500s on every authenticated endpoint — not a `permission-denied` from a Firestore listener. |
| `e0a9617` | 2026-04-29 | "Pre-lunch commit" | Multiple — `DefaultFirebaseChatService` (±56), `NotificationCategoryId` (+3), `DefaultExpoPushService` (±14), `DefaultPushTokenService` (±18), `FirebaseAuthFilter` (+82), `DefaultFirebaseAuthService` (+78), `DefaultTrustReviewService` (±26) | `medium` — broad sweep across the messaging-adjacent surface. Mostly initial introduction of these services per `git log -p` excerpts; not surgical changes. |

**Finding (verdict for scope-item 9):** No commit in the last 60 days touched chat *write* paths (because there are no chat *write* paths on backend). The `f03ea20` "Resolve issues" timing is suspicious purely by date, not by mechanism. (status: `correct`, severity: `high` — flagged for Mastermind anyway.)

### 10. Environment configuration affecting messaging

- **Firebase credentials per environment:**
  - dev (`application-dev.yaml:184-185`): `app.firebase.credentials-path` resolves to `${FIREBASE_CREDENTIALS_PATH:./secrets-local/firebase-service-account.json}` — falls back to in-repo file.
  - stage (`application-stage.yaml:207-208`): `/run/secrets/firebase.json`, container-mounted from `/opt/oglasino/secrets/firebase.json` on the host (`infra/docker-compose-stage.yml`).
  - prod (`application-prod.yaml:196-197`): same path inside the container, same host mount (`infra/docker-compose.yml`).
- **Active profile selection:** `SPRING_PROFILES_ACTIVE: stage` in `infra/docker-compose-stage.yml`; `SPRING_PROFILES_ACTIVE: prod` in `infra/docker-compose.yml`. (Verified.)
- **In-repo credentials inspection:** `./firebase-service-account.json` and `./secrets-local/firebase-service-account.json` both have `"project_id": "oglasino-stage-49abb"` — the stage project. These are the dev fallback. The prod host-mounted credentials file (`/opt/oglasino/secrets/firebase.json`) cannot be read from this audit; per known-issue memory the firebase key being committed to the repo is a separate flagged item.
- **FCM server keys per environment:** none — backend uses Firebase Admin SDK with service-account credentials, not legacy FCM server keys.
- **Messaging-related `application*.yaml` values:** only the chat-image sweep cron (`app.images.chat.removal: "0 0 3 * * SUN"`) — same value across all three profiles. Not related to message delivery.

**Finding (verdict for scope-item 10):** Stage backend is wired to the `oglasino-stage-49abb` Firebase project (per the in-repo fallback credentials, which also live at the prod-config mount path for dev/local — host-mounted secrets on stage/prod hosts authoritative). **If** stage's host-mounted file pointed at prod or another project, every backend Firestore *read* would also fail — that would be observable as 500s on `DefaultTrustReviewService` and the admin chats viewer, not as a web client `permission-denied`. So a host-side project mismatch is unlikely to be the immediate cause but Mastermind should still cross-check with infra. (status: `partial`, severity: `medium` — host secret content not in audit scope.)

## Two key verdicts (consolidated)

1. **Is backend involved in the failing start-message flow?** **No, for the text-send path captured in the error.** No backend HTTP roundtrip happens between Send and the Firestore write attempt. The only backend touchpoint that could deny a chat-related action via Firestore is `DefaultImageTokensFacade.verifyChatMembership` → `isUserMemberOfChat` (Firestore read of `chats/{chatId}`), which fires only if the user attaches an image — and which would surface as HTTP 403 `NOT_CHAT_MEMBER`, not a Firestore listener `permission-denied`.
2. **Auth-token-issue hypothesis?** **Negative.** Backend never issues, refreshes, or revokes the Firebase ID token used by the web client against Firestore on the product-page → start-chat path. Token revocation only fires on admin ban or user hard-delete, neither of which fits the captured symptom.

## Files touched

- none (read-only)

## Tests

- n/a

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed (read-only)
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): see "For Mastermind"
- Part 6 (translations): confirmed — no new keys; verified `BACKEND_TRANSLATIONS` contains no `notif.message.*` entries.
- Part 11 (trust boundaries): confirmed — verified that the membership check (`isUserMemberOfChat`) reads Firestore server-side, not a client-supplied "I'm a member" assertion.

## Known gaps / TODOs

- The audit did not enumerate exhaustive recipient-side per-platform behaviour of `DefaultFirebasePushService` (Webpush-only construction was noted, but iOS/Android paths were not because no caller wires `MESSAGE`).
- Cannot read `/opt/oglasino/secrets/firebase.json` on stage/prod hosts; project ID identity is inferred from the in-repo fallback file, not the live mount.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Primary verdict pair:** see "Two key verdicts" above. Backend is not in the wire path for the failing flow; auth-token-issue is negative.

- **Cross-repo seam #1 — chatId scheme.** Backend (`DefaultTrustReviewService.getChatId`, `service/impl/DefaultTrustReviewService.java:114-116`) computes `chatId = sorted(uid_a, uid_b).join("_")`. The audit *assumes* the web client (and any Firestore rule that derives or asserts a chatId from the participants) uses the same scheme — sort UIDs alphabetically, join with single underscore. If the web client or the rules disagree, `DefaultTrustReviewService` queries the wrong chat (silent semantic bug, not the defect) **and** any `isUserMemberOfChat` lookup would target the wrong doc.

- **Cross-repo seam #2 — chat document `users` field.** Backend's `isUserMemberOfChat` (`admin/service/impl/DefaultFirebaseChatService.java:144-148`) expects `chats/{chatId}.users` to be a `List<String>` of Firebase UIDs, and asserts membership by `users.contains(firebaseUid)`. If the rules require this same array for write authorization, then on a brand-new chat the doc must be `set()` with `users[]` populated atomically with the first message — otherwise the recipient (and possibly the sender, depending on how `request.resource.data.users` is evaluated) would fail rules. **Recommend the firestore-rules audit cross-check this exact shape.**

- **Cross-repo seam #3 — chat message documents.** Backend's `DefaultFirebaseChatService.toChatMessageDTO` (`:174-185`) and `DefaultTrustReviewService` (`:78-103`) both read messages with these field names: `senderId` (Firebase UID string), `receiverId` (Firebase UID string), `content`, `seen` (bool), `createdAt` (Firestore Timestamp), and `productId` (number, optional). If the rules validate per-message writes, these field names and types are the contract.

- **Cross-repo seam #4 — push delivery for messages.** Backend has `NotificationCategoryId.MESSAGE` declared but **no caller**. The user-facing feature (recipient gets a push when sender sends a message) is therefore implemented entirely outside this repo — either web/expo client generates a local notification from a snapshot listener, or there is an unaudited Cloud Function. Worth confirming with the web/mobile audits and asking whether a backend touchpoint was intended but never wired.

- **Suspicious recent commit (timing only, not mechanism).** `f03ea20 "Resolve issues"` (2026-05-17, 2 days pre-defect) fixed a self-reference bug in `DefaultTrustReviewService.getChatId(currentUid, currentUid)` → correct two-arg form. The bug only affected the `canReviewProduct` endpoint, not chat start. Flagging because the date is close and a casual reviewer might fixate on it; the mechanism doesn't fit the symptom.

- **Plausible-but-distinct adjacent failure mode worth ruling out.** `POST /api/secure/images/upload-tokens` with `scope=chat` on a brand-new (not-yet-written) chat throws `NOT_CHAT_MEMBER` 403 (because `chats/{chatId}.exists()` is false → `isUserMemberOfChat` returns false → facade throws). This would manifest as an HTTP 4xx with an `ImageErrorCode`, not a Firestore listener `permission-denied`. **Not the defect**, but if the start-message flow ever uploads an image before the chat doc is created, the user would see a confusing 403. Mastermind should consider whether the web flow's chat-document creation must precede image-upload-token request — sequencing constraint that may want to land in the spec.

- **Known security item already in memory:** the in-repo `firebase-service-account.json` (`oglasino-stage-49abb` project) is a tracked concern; reaffirming it surfaced here but not creating a duplicate flag.

- **Part 4b adjacent observations:**
  - `notifications/enums/NotificationCategoryId.java:4` — `MESSAGE` enum value is declared but has zero callers in `src/main/java`. Either dead code, or a placeholder for an unimplemented "backend pushes for messages" feature. Severity: `low` (cosmetic / decision-pending). Did not delete — out of scope for a read-only audit.
  - `notifications/service/impl/DefaultNotificationsService.java:51-71` does a synchronous `.get()` on the `add` future inside a `@Async` method, then iterates push services. The Firestore write blocks the in-app-notification step, which is fine, but the loop afterwards is sequential; high-fanout users (rare) would notice. Severity: `low`. Did not change — out of scope.
  - `service/impl/DefaultTrustReviewService.java:114-116` — `getChatId` is a static-style helper duplicated across repos by convention. Suggest extracting to a small shared `ChatIdScheme` value type once the messaging feature spec lands. Severity: `low`. Did not change — out of scope.
