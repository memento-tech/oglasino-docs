# Audit — messaging (backend)

**Repo:** `oglasino-backend`
**Branch:** `dev`
**Date:** 2026-05-20
**Scope:** read-only inventory of backend involvement in the messaging surface. Code-only — no spec or prior audit consulted.

---

## 1. Firestore touch surface

All Firestore access goes through the Firebase Admin SDK, which is configured once in `FirebaseConfig` (`src/main/java/com/memento/tech/oglasino/security/config/FirebaseConfig.java:25-40`). Credentials are loaded from the file at `app.firebase.credentials-path` (env var `FIREBASE_CREDENTIALS_PATH`) into `GoogleCredentials.fromStream(...)`. **Every Firestore call below runs as the service account in that JSON file — Admin SDK calls bypass Firestore Security Rules.** There is no per-request actor identity; the backend never uses a Firebase ID token to scope Firestore operations.

Direct `FirestoreClient.getFirestore()` call sites in `src/main/java` (grep result, full surface):

| Class | Method | Firestore path(s) | R/W | Trust-boundary notes (Part 11) |
|---|---|---|---|---|
| `admin/service/impl/DefaultFirebaseChatService.java:30-77` | `getChatsForUserPaged(Long userId, int limit, Instant cursor)` | `chats` (where `users` array-contains the caller's `firebaseUid`, ordered by `lastUpdated` desc) | **Read** | `userId` is a path-variable on the admin controller. `sender.firebaseUid()` is read from Postgres (`userRepository.findChatUserForId`), not from the request. Filter UID is DB-derived ✓. Admin caller identity comes from `OglasinoAuthentication` via `@PreAuthorize("hasRole('ADMIN')")`. |
| `admin/service/impl/DefaultFirebaseChatService.java:79-130` | `getMessagesForChatPaged(String chatId, int limit, Instant lastCreatedAt)` | `chats/{chatId}/messages` (ordered by `createdAt` desc) | **Read** | `chatId` is the path-variable, trusted from the URL. There is **no membership re-check on admin reads** — `hasRole('ADMIN')` alone gates this. `senderId`/`receiverId` per-message are then read back and resolved to `ChatUserDTO` via `userRepository.findChatUserForFirebaseUid`. |
| `admin/service/impl/DefaultFirebaseChatService.java:132-160` | `isUserMemberOfChat(String firebaseUid, String chatId)` | `chats/{chatId}` document, reads the `users` field and tests `contains(firebaseUid)` | **Read** | `firebaseUid` is the caller's identity from `OglasinoAuthentication` (DB-derived ✓). `chatId` is client-supplied on the image-token endpoints. Treats Firestore IO failure as `RuntimeException` (→ 500), not `false` (→ 403 NOT_CHAT_MEMBER) — by design (interface Javadoc lines 12-17). |
| `admin/service/impl/DefaultFirebaseStatsService.java:14-36` | `getNumberOfChats()` / `getNumberOfNotifications()` | aggregate `count()` on `chats` and `notifications` collections | **Read** | Admin stats only. No client-trusted inputs. |
| `service/impl/DefaultTrustReviewService.java:35-112` | `canReviewProduct(Long currentUserId, Long targetUserId, Long targetProductId)` | `chats/{chatId}/messages` (range by `createdAt`, ordered asc) | **Read** | `currentUserId` comes from `OglasinoAuthentication` (see `TrustReviewController` chain — derived ✓). `targetUserId` / `targetProductId` are request inputs, then `userService.getFirebaseUidForUserId` resolves UIDs from the DB. `chatId` is computed locally by `getChatId(uid1, uid2)` — UIDs are DB-derived ✓. |
| `notifications/service/impl/DefaultNotificationsService.java:44-71` | `sendAsyncNotification(NotificationDTO)` | `notifications/{firebaseUid}/userNotifications/{auto-id}` — `.add(notificationData)` | **Write** | `notification.getUserId()` is the **target** user (read from the DB before this call, e.g. `product.getOwner().getId()`). `firebaseUid` is resolved via `userService.getFirebaseUidForUserId(userId)` — DB-derived ✓. Payload fields (`title`, `description`, `type`, `categoryId`, `data`) are constructed by the caller, not pulled from a client request body. |
| `service/impl/DefaultFirestoreUserService.java:23-59` | `deleteUserNotifications(String firebaseUid)` | `notifications/{firebaseUid}/userNotifications/*` (paged batch deletes) then `notifications/{firebaseUid}` document delete | **Write (delete)** | Sole caller: `DefaultUserDeletionService.runHardDelete` (line 413). `firebaseUid` flows through the hard-delete `ctx` object, populated from the DB row of the user being deleted. DB-derived ✓. Best-effort: failures are logged and swallowed by design (lines 50-58). |

**`userchats/**` and `userblocks/**`:** no backend code touches either path. `grep -n 'userchats\|userblocks\|userBlocks\|userChats' --type java src/main/java` returns zero matches. Membership and block state are owned by the web/mobile clients and enforced via Firestore Security Rules — the backend does not consult these collections.

**`chats/**` writes:** none. The backend only reads chat documents.

---

## 2. Image upload-token flow (scope=chat)

Entry point: `images/controller/ImageTokensController.java:41-48` — `POST /api/secure/images/upload-tokens`, body `UploadTokensRequest(scope, count, contentTypes, chatId)`.

Pipeline in `images/facade/impl/DefaultImageTokensFacade.java:82-133`:

1. Resolve `OglasinoAuthentication` from `SecurityContextHolder` (line 84). Unauthenticated → `ImageRequestException.unauthenticated()` (401 UNAUTHENTICATED).
2. Coerce `scope` enum and run Bean-Validation-grade cross-field checks: `enforceCount`, `enforceContentTypesLength`, `enforceContentTypeAllowed`, `enforceChatIdConsistency` (`chatId` required iff `scope=chat`).
3. **For `scope=chat` only:** `verifyChatMembership(auth.getFirebaseUid(), request.chatId())` — calls `FirebaseChatService.isUserMemberOfChat` (Sec 1, row 3). Membership failure → `notChatMember(chatId)` (403 NOT_CHAT_MEMBER). Firestore IO failure → propagates as 500 INTERNAL via the global advice.
4. Consume `count` tokens from the `IMAGE_TOKEN_ISSUANCE` Bucket4j budget (line 101).
5. For each content type: generate a random UUID, build the R2 key via `ImagePaths.chatKey(chatId, uuid, ext)` → **`private/chats/{chatId}/{uuid}.{ext}`** (`images/path/ImagePaths.java:43-46`).
6. Sign one HS256 upload JWT per key via `DefaultImageTokenService.signUploadToken` (`images/token/DefaultImageTokenService.java:71-102`). Claims: `sub=firebaseUid`, `scope=upload`, `kind=chat`, `key=<full key>`, `contentType`, `maxBytes`. Issuer/TTL from `ImageProperties`.
7. Record per-key ownership in Redis (`UploadOwnershipService.recordOwnership(userId, key)`) — best-effort; failures only log WARN. This backs the client-initiated `DELETE /api/secure/images/{key}` endpoint that prunes orphans.

### What the returned token authorizes

The HS256 JWT (signed with `app.images.jwt.signing-secret` / env `JWT_SIGNING_SECRET`) is consumed by the Cloudflare R2 worker, not by this backend. The token's `key` claim is the full R2 key — the worker rejects any PUT to a different key with `TOKEN_KEY_MISMATCH` (per the JJWT-style contract documented at `UploadTokenClaims.java:7-13`). Per-token byte cap is `ImageProperties.getMaxUploadBytes()`.

### Where chat membership is checked

- **What value is checked:** the chat document ID alone (`chatId`) against the caller's Firebase UID.
- **How membership is determined:** **single Firestore read** of `chats/{chatId}.users` array — see `DefaultFirebaseChatService.isUserMemberOfChat`. **No Postgres lookup is involved** — the backend trusts whatever Firestore says about the chat's `users` array.
- **What is *not* checked:** the *other* participant's identity, the chat's `productId`, or that the chatId itself follows the sorted-UID-pair convention. The check is purely "is my UID in the `users` array of the doc with this ID?"

### R2 bucket / path the upload lands in

`private/chats/{chatId}/{uuid}.{ext}` (see `ImagePaths.java:27, 43-46`). Private — every GET requires a view JWT (see Sec 3). The bucket is the same R2 bucket as the other prefixes; access is controlled by key prefix, not by separate buckets.

---

## 3. Image view-token flow (scope=chat)

Entry point: `ImageTokensController.java:50-57` — `POST /api/secure/images/view-tokens`, body `ViewTokensRequest(scope, chatId)`.

Pipeline in `DefaultImageTokensFacade.java:137-171`:

1. Resolve `OglasinoAuthentication` (line 139). Unauthenticated → 401.
2. Enforce `scope == CHAT` — anything else (e.g. `report`) flows out as `INVALID_SCOPE` (line 142-146). v2 will broaden this.
3. `StringUtils.isBlank(chatId)` → `CHAT_ID_REQUIRED` (line 147-149).
4. **`verifyChatMembership(auth.getFirebaseUid(), request.chatId())`** — same Firestore `chats/{chatId}.users` read as upload (Sec 1, row 3). **Membership is re-verified on every view-token mint.** No caching layer between view requests.
5. Consume 1 token from the `IMAGE_TOKEN_ISSUANCE` Bucket4j budget (line 154).
6. Sign HS256 view JWT (`DefaultImageTokenService.signViewToken`, lines 104-135): claims are `sub=firebaseUid`, `scope=view`, `kind=chat`, `keyPrefix=private/chats/{chatId}/`, `chatId=<id>`. The worker enforces `requestedKey.startsWith(keyPrefix)`.

### 401 fallback

The unauthenticated case (no `OglasinoAuthentication` in SecurityContext) returns `UNAUTHENTICATED` mapped to HTTP 401 via `ImageErrorCode.UNAUTHENTICATED` (`images/exception/ImageErrorCode.java:21`). The `ImageExceptionHandler` (`images/exception/ImageExceptionHandler.java:36-45`) wraps it in the §8.1 JSON envelope (`{error: {code, message, details, retryable}}`). Retryable=false. Note: when `FirebaseAuthFilter` rejects a malformed token, it emits a 401 with **no body** (`response.setStatus(SC_UNAUTHORIZED)` at `FirebaseAuthFilter:127`) — so the client sees an empty 401 from the filter, not the JSON envelope. Only requests that *reach* the controller and lack an authenticated principal get the `UNAUTHENTICATED` JSON.

Transient Firestore failure on the membership check throws and is caught by the catch-all `Exception` handler in `ImageExceptionHandler:64-69` → 500 INTERNAL JSON. The client retries 5xx with backoff (contract §C.4 per the inline comment at `DefaultFirebaseChatService:155-158`).

---

## 4. Admin chat endpoints

Controller: `admin/controller/FirebaseChatController.java`. Class-level `@PreAuthorize("hasRole('ADMIN')")` (line 17). Class-level `@RequestMapping("/api/secure/admin/chats")` (line 16). Two endpoints:

### 4a. `GET /api/secure/admin/chats/user/{userId}/chats`

- **Method:** GET.
- **Role:** ROLE_ADMIN (class-level `@PreAuthorize`).
- **Request:** path var `userId: Long`; query params `limit: int (default 10)`, `lastUpdated: Long?` (epoch millis cursor).
- **Response:** `UserChatsPageResponse(List<ChatDTO> chats, Instant nextCursor, boolean hasMore)`. `ChatDTO(chatId, sender: ChatUserDTO, receiver: ChatUserDTO, unreadCount: long, lastMessage: String, lastUpdated: Instant)`. `ChatUserDTO(id: Long, firebaseUid: String, displayName: String)`.
- **What it reads:** the Firestore `chats` collection filtered to documents whose `users` array contains the target user's Firebase UID, sorted by `lastUpdated` desc, paged by cursor. For each chat doc, `lastMessage` and `lastUpdated` are read directly off the chat doc (not from a separate messages query). `unreadCount` is hardcoded to `0L` (`DefaultFirebaseChatService:63`).
- **Write/moderation:** none.

### 4b. `GET /api/secure/admin/chats/{chatId}/messages`

- **Method:** GET.
- **Role:** ROLE_ADMIN.
- **Request:** path var `chatId: String`; query params `limit: int (default 10)`, `lastCreatedAt: Long?` (epoch millis cursor).
- **Response:** `ChatMessagesPageResponse(List<ChatMessageDTO> messages, long nextCursor, boolean hasMore)`. `ChatMessageDTO(messageId, content: Object, sender: ChatUserDTO, receiver: ChatUserDTO, seen: boolean, createdAt: Instant)`.
- **What it reads:** the Firestore `chats/{chatId}/messages` subcollection ordered by `createdAt` desc, paged. The first document's `senderId` and `receiverId` Firebase UIDs are resolved to Postgres `ChatUserDTO` projections; subsequent messages re-use those two records (assumes only two participants).
- **Write/moderation:** none.

### Yes/no on admin write or moderation endpoints

**No.** There is no admin endpoint that posts messages, deletes messages, bans a chat, or otherwise mutates Firestore chat state. Backend admin chat surface is purely read-only.

---

## 5. Notification service

### Method signature and behaviour

`notifications/service/impl/DefaultNotificationsService.java:44-88` — `void sendAsyncNotification(NotificationDTO notification)`, annotated `@Async`. Executed off the request thread.

The DTO (`notifications/dto/NotificationDTO.java`) carries `userId`, `title`, `description`, `notificationCategoryId: NotificationCategoryId`, `notificationType: NotificationType`, `data: Map<String, Object>`. No `Long messageId`, no `firebaseUid` field — the service resolves the target's UID from `userService.getFirebaseUidForUserId(userId)`.

The method does two things:

1. **Firestore write:** `notifications/{firebaseUid}/userNotifications` `.add({title, description, seen=false, createdAt=serverTimestamp, type, categoryId, data})` (lines 51-67). Awaits the future synchronously (`.get()`) inside the `@Async` thread; failure is logged at ERROR and the method returns without dispatching pushes.
2. **Push fan-out:** looks up the user's `PushToken` rows via `pushTokenService.getUserTokens(userId)` and dispatches to the matching `PushService` per platform.

### Firestore path written

`notifications/{firebaseUid}/userNotifications/{auto-id}` (constants at lines 25-26).

### Firebase identity used

The Firebase **Admin SDK service account** (configured in `FirebaseConfig`). No custom token; no caller's Firebase ID token is used to scope this write. Admin SDK writes bypass Firestore Security Rules.

### `NotificationCategoryId` callers

Enum values (`notifications/enums/NotificationCategoryId.java`): `MESSAGE`, `SAVED_PRODUCT`, `PRODUCT_EXPIRATION`, `PRODUCT_EXPIRED`, `NAVIGATION`, `INFO`.

`grep -n 'NotificationCategoryId\.' --type java src/main/java` returns exactly four production assignments:

| Caller file | Line | Category | Notification context |
|---|---|---|---|
| `admin/service/impl/DefaultAdminReviewService.java` | 94 | `NAVIGATION` | review-approve success notification to reviewer (`sendReviewerSuccessNotification`) — `data.navigation=/dashboard/reviews/given?reviewId=<id>` |
| `admin/service/impl/DefaultAdminReviewService.java` | 120 | `NAVIGATION` | review-approve success notification to target user (`sendTargetUserSuccessNotification`) — `data.navigation=/dashboard/reviews/received?reviewId=<id>` |
| `admin/facade/impl/DefaultAdminReportFacade.java` | 108 | `NAVIGATION` | report resolution notification to reporter (`sendReportNotification`) — `data.navigate=/dashboard/reports?reportId=<id>` |
| `facade/impl/DefaultFavoriteProductFacade.java` | 116 | `SAVED_PRODUCT` | someone favorited the owner's product (`notifyUserOnSavedProduct`) — `data={productId, productName}` |

Plus one **non-production** caller:

| Caller file | Line | Category | Notes |
|---|---|---|---|
| `controller/test/NotificationsControllerTest.java` | 28 | *any (from request body)* | Public endpoint `POST /api/public/notification/test` — accepts arbitrary `userId`, type, category, and `data` from a `NotificationTestDTO` and forwards verbatim to `sendAsyncNotification`. See Sec 10. |

### Does `NotificationCategoryId.MESSAGE` have any callers?

**No.** `grep -n 'NotificationCategoryId\.MESSAGE' --type java src/main/java` returns zero matches. The enum constant exists but is unreferenced — `MESSAGE` notifications are written directly by the web/mobile clients to Firestore, not via the backend's notification path.

---

## 6. Trust-review chat ID computation

### `getChatId` signature and algorithm

`service/impl/DefaultTrustReviewService.java:114-116`:

```java
private String getChatId(String uid1, String uid2) {
  return Stream.of(uid1, uid2).sorted().collect(Collectors.joining("_"));
}
```

**Algorithm in plain words:** take the two Firebase UIDs, sort them by natural-string (alphabetical) order, join them with a single underscore. So `getChatId("alice", "bob") == getChatId("bob", "alice") == "alice_bob"`. Deterministic and symmetric in the two arguments. Null behaviour is not defended; both arguments are assumed non-null. Empty strings would produce a string starting/ending with `_` but no validation against that.

### Callers of `getChatId`

**Single caller in `src/main/java`:** the same file, line 59 (`DefaultTrustReviewService.canReviewProduct`):

```java
String chatId = getChatId(currentUserFirebaseUid, targetUserFirebaseUid);
var messagesRef = db.collection("chats").document(chatId).collection("messages");
```

Both UIDs are resolved from Postgres via `userService.getFirebaseUidForUserId(userId)` before this call, so the inputs are DB-derived (trust-boundary clean).

The method is `private` — there is no other code path in this repo that constructs a chatId. Visibility means it cannot be called from outside the class, and `grep` confirms no other call sites.

### Algorithm alignment with the web client

Cannot be audited from this repo per the brief's cross-repo rule (the web audit reports the web side). The backend simply *assumes* the web client mints `chats/{chatId}` documents whose ID is the same sorted-UID-pair-joined-by-underscore — if web ever uses a different separator or unsorted order, trust-review silently degrades to "no messages found → not eligible" (HTTP 403). No log, no exception. Worth a contract entry in the eventual messaging spec.

---

## 7. Translation seed surface for messaging

### Which SQL files contain `MESSAGES_PAGE` rows

`grep -c "'MESSAGES_PAGE'"` across `src/main/resources/data/translations/*.sql`:

| File | Rows |
|---|---|
| `0001-data-web-translations-EN.sql` | **19** |
| `0001-data-web-translations-RS.sql` | **19** |
| `0001-data-web-translations-CNR.sql` | **19** |
| `0001-data-web-translations-RU.sql` | **19** |
| `0002-data-translations-{EN,RS,CNR,RU}.sql` | 0 |
| `0003-data-filter-option-translations-{EN,RS,CNR,RU}.sql` | 0 |

Total: **76 rows across the 4 locales (19 keys × 4 langs)**.

Line ranges (EN file, lines 811-829): IDs 3342-3360 (19 contiguous rows). RS file: 5442-5460. CNR: 1242-1260. RU: 7542-7560.

### Inline-appended or dedicated per-feature

**Inline-appended.** All 19 keys per locale live within the `0001-data-web-translations-{LANG}.sql` general-seed files, in the `MESSAGES_PAGE` namespace group. No dedicated `NNNN-data-messaging-translations-{LANG}.sql` file exists.

`find . -name '*messaging*' -o -name '*messag*-translations*'` returns nothing.

### Existing key inventory (EN values, for orientation only)

| Key | EN value |
|---|---|
| `search.placeholder` | "Search chats..." |
| `chats.empty.list` | "You do not have any chats" |
| `messages.select.placeholder` | "Select a chat to view messages..." |
| `initial.product.message` | "Hello! I am interested in the product {value}" |
| `func.profile` / `func.delete` / `func.report` / `func.block` / `func.unblock` | row-action labels |
| `messages.load.more` | "Load more messages..." |
| `blocking.label` / `blocked.by.label` | block notices |
| `max.images.alert` | "Maximum {value} images." |
| `message.input.placeholder` | "Enter your message..." |
| `max.images.label` | "Import images (max {value})" |
| `submit.message.label` | "Send" |
| `chats.empty.filter.list` | "No chats found" |
| `user.pending.deletion.notice` | "This user has scheduled their account for deletion and cannot receive messages." |
| `user.banned.notice` | "This user has been banned and cannot receive messages." |

Shape implication for future seeds: per conventions Part 6 Rule 3, new keys append at the end of the `MESSAGES_PAGE` group inline. Per the Rule 3 exception, if a single feature adds 10+ keys across multiple namespaces a dedicated `NNNN-data-<feature>-translations-{LANG}.sql` is allowed — none exist for messaging today.

---

## 8. `FirebaseAuthFilter` and identity surface

### Where ID-token verification happens

`security/filter/FirebaseAuthFilter.java`. `doFilterInternal` resolves the token via `firebaseAuthService.resolveFirebaseToken(request)` (line 69) and calls `firebaseAuthService.verifyToken(idToken)` (line 73) which delegates to `FirebaseAuth.getInstance().verifyIdToken(idToken)` (per `DefaultFirebaseAuthService.java:38-40`). The filter runs on every request except `OPTIONS` and the Postman-testing dev path (lines 60-67).

### What's loaded into `OglasinoAuthentication`

After `verifyToken` succeeds, the filter calls `firebaseAuthService.getCachedAuthData(decoded.getUid())` (line 78) — this is `@Cacheable("redisUserAuth", key="#firebaseUid")` returning an `AuthenticatedUserDTO` projected from `users` (cache miss runs a single JOIN-FETCH JPQL projection).

The `AuthenticatedUserDTO` record (`security/auth/AuthenticatedUserDTO.java:20-29`):

```java
record AuthenticatedUserDTO(
    Long userId,
    String firebaseUid,
    boolean disabled,
    UserRole userRole,
    UserSubscription subscriptionType,
    boolean subscriptionActive,
    Long baseSiteId,
    LanguageDTO preferredLanguage,
    DeletionStatus deletionStatus) {}
```

The filter then short-circuits on `disabled=true` (403 USER_BANNED) and on `deletionStatus=PENDING_DELETION` (clears context, passes through anonymously), and constructs `OglasinoAuthentication` on the happy path (lines 99-105):

```java
new OglasinoAuthentication(
    authData.userId(),
    decoded.getUid(),               // firebaseUid
    authData.baseSiteId(),
    authData.preferredLanguage(),
    AuthUtils.subscriptionToAuthorities(authData))
```

`OglasinoAuthentication` (`security/auth/OglasinoAuthentication.java`) carries:

- `Long userId` — **non-null** (`Objects.requireNonNull` line 35)
- `String firebaseUid` — **non-null** (`Objects.requireNonNull` line 36)
- `Long baseSiteId` (may be null)
- `LanguageDTO preferredLanguage` (may be null)
- `Collection<? extends GrantedAuthority> authorities`
- `Object credentials` — the verified `FirebaseToken` is stashed here for downstream controllers (line 109 in the filter; see field Javadoc at `OglasinoAuthentication.java:18-24`).

Authorities are computed by `AuthUtils.subscriptionToAuthorities` (`security/AuthUtils.java:18-33`): if `subscriptionType == null || !subscriptionActive` it returns `List.of()` (empty); otherwise `[ROLE_<subscription>, <userRole>]`. `UserRole` enum names already carry the `ROLE_` prefix (`entity/UserRole.java`: `ROLE_ADMIN`, `ROLE_BASIC`).

### Specifically: is `firebaseUid` available, only `userId`, both?

**Both.** `OglasinoAuthentication.getFirebaseUid()` returns the verified Firebase UID (always non-null on authenticated requests) and `getUserId()` returns the backend Long (also always non-null). The two are linked at filter time through the Redis-cached `AuthenticatedUserDTO`, which is in turn keyed by `firebaseUid` and projected from `users.firebaseUid` (column).

Where each comes from:
- `userId` ← `AuthenticatedUserDTO.userId` (Postgres `users.id`).
- `firebaseUid` ← `decoded.getUid()` from the verified Firebase ID token (Firebase Auth's own UID for the user).

These are both available to any code that reads `SecurityContextHolder.getContext().getAuthentication()` and casts to `OglasinoAuthentication`. Messaging clients use `firebaseUid` for Firestore writes and image-token signing; everything else in the system (Postgres FKs, business logic) uses `userId`.

---

## 9. Cross-repo seams

For each backend involvement in messaging, the assumption the backend makes about its counterpart:

1. **Image upload-token chat membership** — backend assumes the web/mobile client has already created (or is about to create) a Firestore `chats/{chatId}` document whose `users` field is a `List<String>` of exactly the two participants' Firebase UIDs. If the doc doesn't yet exist, `isUserMemberOfChat` returns `false` and the token issuance is rejected with 403 NOT_CHAT_MEMBER. This makes the client-side "create chat doc *before* requesting an upload token" ordering load-bearing.

2. **Image view-token chat membership** — same assumption as upload. Per-view re-check, no caching.

3. **Image worker contract** — backend assumes the Cloudflare R2 worker (`oglasino-router`) enforces the JWT's `key` claim on uploads (rejects mismatched PUTs as `TOKEN_KEY_MISMATCH`) and the `keyPrefix` claim on views (rejects GETs to keys outside the prefix). Backend never validates the worker's response — it just mints tokens. JWT signing secret is the shared trust anchor.

4. **Admin chat list rendering** — backend assumes Firestore `chats/{chatId}` documents carry exactly: `users: List<String>` (Firebase UIDs of participants), `lastMessage: String`, `lastUpdated: Timestamp`. Missing fields → `null` values in the response; no schema version field.

5. **Admin chat messages rendering** — backend assumes Firestore `chats/{chatId}/messages/{msgId}` documents carry: `senderId: String` (Firebase UID), `receiverId: String` (Firebase UID), `content: Object` (passed through as-is), `seen: boolean`, `createdAt: Timestamp`. The first message's `senderId`/`receiverId` set the chat's two-participant projection for the whole page — image messages or any non-string `content` flow through as a Java `Object` in `ChatMessageDTO` and become whatever Jackson serializes.

6. **Trust-review chat-ID convention** — backend assumes web client mints chat doc IDs as `sorted([uid1, uid2]).join("_")`. If web diverges from this convention, every trust-review check silently fails (no messages found → ineligible). There is no shared utility or backend-frontend contract document tying the two; the convention is implicit in code.

7. **Trust-review message schema** — backend assumes each message doc may carry `productId` as a `Number` (Long compatible). Backend uses this to detect "product mention" and gate alternating-message counts. The web/mobile clients must write `productId` as a number, not a string. Trust-review tolerates absent `productId` (skipped, never gated as a mention).

8. **Notification writes via Admin SDK** — backend assumes Firestore Security Rules permit Admin SDK writes to `notifications/{firebaseUid}/userNotifications/**`. Admin SDK bypasses rules, so the rules surface only governs *client*-side reads of these documents. Backend further assumes web/mobile clients read these docs scoped to `auth.uid` so that user A cannot read user B's notifications — rule enforcement lives in `oglasino-firestore-rules`, not here.

9. **Hard-delete notification cleanup** — backend assumes `DefaultFirestoreUserService.deleteUserNotifications(firebaseUid)` can safely orphan partial state on failure (`runHardDelete` is in flight; the user row will be hard-deleted regardless). No `userchats`/`userblocks` cleanup is performed — those collections (if used by the messaging feature) are not part of the hard-delete path and would be orphaned on the Firestore side after a hard delete.

10. **FirebaseAuthFilter clears context on `PENDING_DELETION`** — backend assumes web's SSR or any client carrying a stale post-deletion token will get an anonymous response from `/api/secure/**` (URL rule → 401). Messaging clients must therefore treat 401 on a previously-working request as "session ended, re-auth," not as "chat doesn't exist."

---

## 10. Adjacent observations

Per conventions Part 4b. Flagged here, not fixed; this is a read-only audit.

1. **`getChatsForUserPaged` computes the receiver from the *first* chat doc for every row.** `DefaultFirebaseChatService.java:54-66` — inside `docs.stream().map(doc -> ...)` the receiver is computed by `getReceiverFromChatDoc(docs, sender.firebaseUid())`, and that helper (lines 162-172) does `docs.stream().findFirst().orElseThrow()` — it inspects the *page's first document's* `users` array regardless of which doc is being mapped. So every chat row in the response carries the **same** receiver (whoever the first chat's counterparty is), not the actual receiver of each chat. **Severity: high.** User-facing bug in the admin chat list UI. The fix is to pass `doc` to the helper and read `doc.get("users")` instead of `docs.stream().findFirst()`. I did not fix this because it is out of scope for a read-only audit.

2. **`/api/public/notification/test` is a public endpoint that can write a Firestore notification to *any* user.** `controller/test/NotificationsControllerTest.java:14-35` is registered at `/api/public/notification/test` (which the security config permitAll's) and forwards a body-controlled `NotificationTestDTO` — `userId`, `title`, `description`, `data`, `category`, `type` — verbatim into `sendAsyncNotification`. No auth, no validation that the caller "owns" the target userId. Per the user's saved memory, production deploy is imminent. **Severity: medium.** Out of scope; flagged for routing. Suggest gating behind `@Profile("dev")` or relocating to `/api/secure/**` plus an ADMIN role check before launch.

3. **`AuthUtils.subscriptionToAuthorities` drops every authority — including the user-role — when subscription is inactive or null.** `security/AuthUtils.java:23-33`: `if (!subscriptionActive || subscriptionType == null) return List.of();` — that means an admin user whose paid subscription has lapsed loses `ROLE_ADMIN` authority, and `@PreAuthorize("hasRole('ADMIN')")` on `FirebaseChatController` (and every other admin endpoint) would deny them. Possibly intentional (no admin in prod is ever subscription-less), but it couples role-based access to subscription state in a way that's surprising. **Severity: low.** Pre-existing; out of scope for messaging.

4. **`getMessagesForChatPaged` assumes only-two-participants by hardcoded design.** `DefaultFirebaseChatService.java:103-114` reads `senderId`/`receiverId` off the *first* message in the page and maps every subsequent message to the same two-user pair. If a chat ever supports more than two participants (group chats), this projection collapses incorrectly. Not a bug today (chats are 1:1) but worth recording as the data-shape lock. **Severity: low** (informational).

5. **`ChatMessageDTO.content` is `Object`.** `admin/dto/ChatMessageDTO.java:7`. The admin endpoint passes Firestore's `content` field through Jackson as-is — could be a string for text messages, a `Map` for image messages, anything. The frontend admin chat viewer must defensively handle both. Worth either typing this (e.g. `String text; List<String> imageKeys` discriminated union) or documenting the contract. **Severity: low.**

6. **Trust-review's `getChatId` algorithm is a hidden contract.** See Sec 6. The sort-then-join-by-`_` convention is duplicated across backend (this method) and web client (assumed); no shared source. If the messaging feature ever changes the chatId scheme on the client side, trust-review silently 403s. **Severity: medium** — silent failure mode, not a user-blocker today but exactly the kind of seam Mastermind asked about in Sec 9. Suggest documenting the convention in the eventual messaging spec.

7. **`isUserMemberOfChat` accepts a non-null `chatId` that doesn't exist as "not a member."** `DefaultFirebaseChatService:140-143`: if the chat doc doesn't exist, the method returns `false`, surfacing as 403 NOT_CHAT_MEMBER. A client requesting an upload-token for a chat it's about to create gets 403, not a more specific "chat-not-found." May be by design; worth confirming with the messaging spec. **Severity: low** (informational).

8. **Hard-delete leaves chat side-data orphaned.** `DefaultUserDeletionService.runHardDelete` calls `deleteUserNotifications` but does **not** touch the user's `chats/{*}` membership (would require removing the deleted UID from the `users` array on every chat the user participated in) and does not delete the user's chat *messages* (which reference `senderId=firebaseUid`). For the User Deletion feature this was deemed acceptable (the feature's spec lives at `../oglasino-docs/features/user-deletion.md`); a future messaging feature spec should note that ex-users' UIDs persist in Firestore chat membership arrays and message author IDs after hard-delete. **Severity: low** (informational, downstream of an existing decision).

---

## For Mastermind

1. **The admin chat-list receiver bug (Adjacent observation #1) is user-facing.** Mastermind should decide whether to fold it into the messaging feature (since the admin chat surface is in scope) or queue it as its own bug-chat entry. Either way it shouldn't ship as-is.

2. **`NotificationCategoryId.MESSAGE` is dead-on-backend.** If the messaging feature plans to introduce backend-mediated message notifications (currently web/mobile clients write them directly to Firestore), this enum value is the natural hook; today it's unused. If the plan is to *keep* message-notifications client-driven, the enum value can be deleted as obsolete.

3. **The `/api/public/notification/test` public endpoint (Adjacent observation #2) is a pre-launch blocker.** Likely needs Docs/QA + a small backend brief to gate it behind `@Profile("dev")` before production. Flagging as it intersects messaging (writes notifications via Firestore) and the user's saved-memory production-imminent posture.

4. **Trust-review chat-id convention should be promoted to a documented contract.** Sec 6 + Adjacent observation #6. The web audit will report the web side; if they match, the eventual `features/messaging.md` should pin the algorithm. If they differ, that's a Phase 3 seam to resolve.

5. **No backend code touches `userchats/**` or `userblocks/**`.** Both collections are owned end-to-end by the client + Firestore rules. If the messaging feature relies on either, the rules repo (`oglasino-firestore-rules`) is the only place enforcement happens; backend has no awareness.
