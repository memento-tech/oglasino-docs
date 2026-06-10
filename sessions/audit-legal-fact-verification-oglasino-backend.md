# Audit — Legal-Document Fact Verification (oglasino-backend)

**Date:** 2026-06-10
**Type:** Read-only audit. No code changed.
**Method:** Every cited claim verified twice — a direct `Read` of the cited region **and** an `rg` confirmation. Verdicts are relative to the quoted Privacy Policy / Terms of Use draft claims.

Verdict vocabulary: **VERIFIED-TRUE / VERIFIED-FALSE / PARTIAL / NOT FOUND**.

---

## Q1 — Ban flow

### (a) Account states in the model

There is **no single account-state enum.** State is carried by two independent dimensions plus derived view-enums:

| Concept | Where | Values |
| --- | --- | --- |
| Ban flag | `entity/User.java:45-46` (`disabled`), `:48-49` (`banReason`) | `disabled` boolean + free-text `banReason` |
| Deletion lifecycle | `entity/User.java:51-53` (`deletionStatus`); enum `entity/DeletionStatus.java:3-6` | `ACTIVE`, `PENDING_DELETION` |
| Deletion-request status (not the account) | `entity/DeletionRequestStatus.java` | `PENDING`, `CANCELLED`, `COMPLETED` |
| Public-facing composite | `entity/UserState.java:11-28` | `ACTIVE`, `PENDING_DELETION`, `BANNED` (BANNED wins) |
| Admin-facing composite | `admin/dto/UserStateInfoDTO.java:23`; resolved in `admin/facade/impl/DefaultUsersFacade.java:201-210` | `ACTIVE`, `BANNED`, `LOCKED`, `PENDING_DELETION` |

> `entity/DeletionStatus.java:3-6`
> ```java
> public enum DeletionStatus { ACTIVE, PENDING_DELETION }
> ```
> `entity/UserState.java:20-27` — `resolve(disabled, deletionStatus)`: `disabled → BANNED`; else `PENDING_DELETION → PENDING_DELETION`; else `ACTIVE`.

The `LOCKED` admin state derives from a separate `UserDeletionLock` row (deletion hold), not from the user row.

**Plain answer:** A ban is the boolean `disabled=true` (+ `banReason`), distinct from `deletionStatus` (`ACTIVE`/`PENDING_DELETION`). Public view collapses to `ACTIVE`/`PENDING_DELETION`/`BANNED`; admin view adds `LOCKED`.

### (b) What admin ban actually does — **VERIFIED-FALSE** (vs ToU §10.2 "deletes all user data immediately and permanently")

`admin/facade/impl/DefaultUsersFacade.disableUser:70-115`:

> ```java
> user.setDisabled(true);
> user.setBanReason(trimmed);
> userService.saveUser(user);                       // :78-80
> eventPublisher.publishEvent(new UserStateChangedEvent(userId));   // :85
> eventPublisher.publishEvent(new UserBannedEvent(userId));         // :89  (suspended email)
> userAuditService.recordBannedUser(user.getEmail(), trimmed, adminId);  // :91 (ban hash)
> firebaseUserService.disableUser(user.getFirebaseUid());           // :93
> firebaseUserService.revokeRefreshTokens(user.getFirebaseUid());   // :94
> ...
> productService.changeProductStateAsSystem(productId, ProductState.INACTIVE);  // :105 (each ACTIVE product)
> ```

Ban **sets a status and retains all data.** Concretely:
- **Visibility:** public state → `BANNED` (`UserState.resolve`, disabled wins). Public profile-by-id 404s — `repository/UserRepository.findUserInfoById:155` filters `WHERE u.id = :userId AND u.disabled = false`. Phone suppressed — `UserRepository.getUserPhoneNumberAllowed:73-79` requires `disabled = false`.
- **Auth:** Firebase account disabled + all refresh tokens revoked (`:93-94`).
- **Push tokens:** **not** cleared at ban (only at hard delete — see Q2b).
- **Listings:** every `ACTIVE` product flipped to `INACTIVE` with `deactivated_by_system=TRUE` (`:102-106`), reversible on unban (`enableUser:148-154`).
- No row, image, review, report, or Firestore data is deleted.

**Plain answer:** Admin ban sets `disabled=true`, hides profile/phone/listings, disables Firebase auth and revokes tokens, and records a ban-hash audit — it does **not** delete data, contradicting ToU §10.2.

### (c) Is automatic deletion ever scheduled for a banned account? — **VERIFIED (owner intent satisfied)**

`disableUser` does not touch `deletionStatus` and creates no `UserDeletionRequest`. There is no timer that deletes banned accounts. The only deletion paths are user-requested (Q2/Q3), admin force-delete, or the weekly Firebase-orphan cascade (`jobs/UserDeletionScheduledJobs.reconcileFirebaseUsers:187-229`) which fires only when the Firebase account itself is gone — not because of a ban.

**Plain answer:** No automatic deletion is scheduled for banned accounts; the ban hash is retained 12 months (Q2d) but the account data persists until a separate deletion path runs.

---

## Q2 — Hard deletion contents and timers

### (a) Grace period — **VERIFIED-TRUE** (doc: 7 days)

`data/configuration/data-configuration.sql:86` — `(49, 'user.deletion.grace.period.days', '7', …)`. Read at `service/impl/DefaultUserDeletionService.requestDeletion:115` and added to `now` at `:120`.

### (b) What hard delete deletes vs retains (per store)

All in `service/impl/DefaultUserDeletionService.runHardDelete:303-455` unless noted.

| Store | Action at hard delete | Evidence |
| --- | --- | --- |
| **Postgres — favorites** | join rows cleared | `:343` |
| **Postgres — reports** | anonymized-if-banned, else deleted | `:347` → `repository/ReportRepository.java:21-45` |
| **Postgres — reviews** | approved → `reviewer_id` NULL (kept, anonymized); pending → delete; disapproved → delete; reviews *about* the user → delete | `:355-358` |
| **Postgres — follows** | both directions deleted | `:361` (`deleteByFollowerIdOrFollowingId`) |
| **Postgres — push tokens** | deleted | `:362` (`pushTokenRepository.deleteByUserId`) |
| **Postgres — suggestions / product_audit** | deleted | `:363-364` |
| **Postgres — user (+ cascades)** | user deleted → JPA cascade deletes products (+ translations, product_images join, product_filter_value), `user_translation` (orphanRemoval); FK `ON DELETE CASCADE` removes `user_deletion_requests` + `user_deletion_locks` | `:381-384` |
| **Firestore — notifications** | user notifications doc deleted (post-commit) | `:434-436` |
| **Firestore — messages** | **anonymized, not deleted**, by the Sunday cron: `senderId`/`receiverId`/chat `users[]` rewritten to `deleted:<uid>` sentinel; `userchats`/`userblocks`/`userblocksReverse` sidecars deleted. **Message text is retained.** | queued `:413-416`; `messaging/service/impl/DefaultMessagingCleanupService.java:51, 147-224, 277-299` |
| **R2 objects** | product images bulk-deleted; profile image deleted | `:438-447` |
| **R2 — CDN cache purge** | **none issued** — no purge/invalidate code exists anywhere | `rg purgeCache\|cdn.*purge\|invalidate` → no R2/CDN purge |
| **Firebase Auth** | user record deleted | `:448-454` |
| **Elasticsearch** | **not deleted** — `runHardDelete` has zero ES references; product ES docs orphan until the next full reindex (drift tracked by `EsStateResponse`) | `rg elasticsearch\|ProductIndexer` over the file → none; see Adjacent finding 2 |
| **Redis** | only `@CacheEvict` of `redisUserAuth` + `redisUserInfo` via `saveUser`; transient keys (`view:…`, `product:owner:…`, report dedup) not purged — expire by TTL | `:344` etc.; see Adjacent finding 4 |

**Plain answer:** Hard delete removes the Postgres user (and cascaded products/translations/follows/push-tokens), Firestore notifications, R2 images, and the Firebase Auth record; it **anonymizes** chat messages (rewrites IDs, keeps text) and approved reviews; it does **not** delete Elasticsearch docs, does **not** purge the CDN, and does **not** clear transient Redis keys.

### (c) Reports filed BY and ABOUT the user

`service/impl/DefaultUserDeletionService.linkReportsToBanIfBanned:462-480` + `repository/ReportRepository.java:21-45`:
- **If banned:** `reporter`→NULL and `reportedUser`→NULL, both linked to the `banned_user_audit` row (`bannedUserAuditId`). Kept (anonymized) for the 12-month ban window, then FK-CASCADE-deleted when the ban-audit row is purged.
  > `ReportRepository.java:23-25` — `UPDATE Report r SET r.reporter = NULL, r.bannedUserAuditId = :auditRowId WHERE r.reporter.id = :userId`
- **If not banned:** reports deleted outright (`:39-45`).

Raw user-id identifiers (`reporter_id`, `reportedUser_id`) are **never retained** post-delete (nulled when kept, gone when deleted). The free-text `description` and `reportedProductId`/`reportedReviewId` remain on anonymized rows (the deleting user's own product/review IDs are themselves deleted).

**Plain answer:** Reports by/about a deleted user are either deleted (unbanned) or kept with the user FKs nulled and linked to the ban audit (banned) — no raw user IDs/emails survive in report rows.

### (d) Audit retention + purge jobs + hashing

- **Deletion audit log retention = 30 days:** `data-configuration.sql:88` `(51, 'user.deletion.audit.retention.normal.days', '30', …)`; applied `requestDeletion:144-145` / `ensureAuditLogRow:508`.
- **Ban hash retention = 12 months:** `data-configuration.sql:89` `(52, 'user.deletion.audit.retention.banned.months', '12', …)`; applied `service/impl/DefaultUserAuditService.recordBannedUser:64`.
- **Both purge jobs exist and run on a schedule:** `jobs/UserDeletionScheduledJobs.purgeExpiredAuditRecords:241-252`, `@Scheduled(cron = "…:0 0 4 * * SUN")` (weekly Sun 04:00 UTC):
  > ```java
  > int deletedAudit = auditLogRepository.deleteExpired(now);              // :244 (30-day rows)
  > int deletedBans = bannedUserAuditRepository.deleteExpired(now);        // :245 (12-month rows)
  > int deletedAdminActions = adminActionAuditRepository.deleteByRetentionUntilLessThanEqual(now); // :246
  > ```
- **SHA-256 is UNSALTED:** `service/impl/DefaultUserAuditService.java:113-125` (`sha256Hex` = plain SHA-256, hex). `hashEmail = sha256(email.trim().toLowerCase())` (`:26-28`); `hashUserId = sha256(userId.toString())` (`:31-33`).

**Plain answer:** Retention constants (30 days / 12 months) and both weekly purge jobs are implemented as the doc states; the email/user-id hashes are **unsalted** SHA-256.

---

## Q3 — Soft-deletion behavior (inside the grace window)

### (a) Profile publicly visible with a "scheduled for deletion" flag — **VERIFIED-TRUE**

`repository/UserRepository.findUserInfoById:125-158` returns the row while `disabled = false` (so a `PENDING_DELETION` user is still served) and projects both `deletionStatus` and `scheduledDeletionAt`. `dto/UserInfoDTO.java:47-50` exposes `state` + `scheduledDeletionAt`; `converter/EntityUserInfoConverter.java:59` sets `state = UserState.resolve(...)`.

### (b) Phone number hidden — **VERIFIED-TRUE**

`repository/UserRepository.getUserPhoneNumberAllowed:73-79`:
> ```sql
> WHERE u.id = :userId AND u.allowPhoneCalling = true AND u.disabled = false
>   AND u.deletionStatus = …DeletionStatus.ACTIVE
> ```
Pending-deletion users return empty → phone suppressed.

### (c) Listings hidden from public view — **VERIFIED-TRUE**

`service/impl/DefaultUserDeletionService.requestDeletion:157-161` flips every `ACTIVE` product to `INACTIVE` (system context); the state change propagates to ES via `ProductStateUpdateEvent` (`listeners/ProductIndexerEventListener.java:37-43`).

### (d) Reviews about the user remain visible — **VERIFIED-TRUE**

The soft-delete path (`requestDeletion`) touches no reviews. Reviews are only anonymized/deleted at hard delete (`runHardDelete:355-358`). During the grace window they are untouched and remain visible.

### (e) Messaging blocked in BOTH directions — **PARTIAL**

- **Outbound (the deleting user) — verified backend-side:** refresh tokens are revoked at request time (`requestDeletion:168-170`), and `security/filter/FirebaseAuthFilter.java:102-106` drops the security context for `PENDING_DELETION`, so every `/api/secure/**` call (incl. the message-send push) returns 401. Without a valid Firebase token the user also cannot read/write Firestore directly.
- **Inbound (others messaging the deleting user):** enforced — if at all — by Firestore Security Rules in the **`oglasino-firestore-rules`** repo. This backend contains no inbound-messaging gate keyed on the recipient's deletion state.

**Plain answer:** The backend blocks the deleting user's own messaging (token revoke + auth-filter short-circuit); the inbound block is a Firestore-rules concern outside this repo and cannot be confirmed here.

### (f) Display name → "Deleted User" at SOFT delete — **VERIFIED-FALSE**

Soft delete changes nothing in messaging. Anonymization happens at **hard** delete, via the Sunday cron, and is a **sentinel-UID rewrite**, not a stored "Deleted User" name: `messaging/service/impl/DefaultMessagingCleanupService.java:51` (`SENTINEL_PREFIX = "deleted:"`), `:240-251` (rewrites `senderId`/`receiverId`), `:216-223` (rewrites chat `users[]`). Any "Deleted User" label is a client-side rendering of that sentinel.

**Plain answer:** The name does not change at soft delete; counterpart anonymization is a hard-delete-time `deleted:<uid>` sentinel rewrite, not a soft-delete "Deleted User" display name.

### (g) Sign-in during the window restores everything — **VERIFIED-TRUE**

`service/impl/DefaultUserDeletionService.cancelDeletionOnLogin:187-240` sets `deletionStatus = ACTIVE` (`:222`) and flips system-deactivated products back to `ACTIVE` (`:229-233`); phone visibility and secured access auto-restore because they gate on `deletionStatus = ACTIVE` / a valid token. Triggered exclusively by the sign-in path (`FirebaseAuthFilter:97-98` comment; `controller/AuthController.java:155`).

---

## Q4 — Notifications, favorites, follows, reports, reviews

### (a) Per-event notification + push contents

Firestore notification doc fields (`notifications/service/impl/DefaultNotificationsService.java:54-61`): `title`, `description`, `seen`, `createdAt`, `type`, `categoryId`, `data`. Push payload (`DefaultFirebasePushService` / `DefaultExpoPushService`): `title`, `body` (= description), `data` (`type`, `categoryId`, + custom).

- **Favorite (SAVED_PRODUCT)** — `facade/impl/DefaultFavoriteProductFacade.notifyUserOnSavedProduct:96-121`. Recipient = product owner; title/description from backend translation, description formatted with **product name**; `data = {productId, productName}`. **Does NOT identify the favoriting user** (no display name, no user ID).
- **Follow** — `facade/impl/DefaultUserFacade.notifyUserOnNewFollower:67-91`. Description formatted with `follower.getDisplayName()` (`:79`); `data.navigate = "/user/" + followerId` (`:84-85`). Includes follower **display name and ID**.
- **Chat message** — `notifications/service/impl/DefaultMessageNotificationService.buildNotification:126-138`. Push-only (no Firestore doc). `title = sender.getDisplayName()` (`:130`); `body =` client-supplied **message text** (or localized "[sent a photo]" for image-only, `:113-124`); `data.navigate = "/messages"`. **Contains sender display name + full message text.**
- **Admin/report-resolution** — admin-supplied title/description, no PII (per `admin/.../DefaultAdminReportFacade`, `DefaultAdminReviewService`).

**Plain answer (the two doc-flagged cases):** the favorite notification does **not** identify the favoriting user; the chat push **does** contain the sender's display name and the full message text.

### (b) Follow feature

- **Storage:** `entity/UserFollow.java:11-44` — table `user_follows`, columns `follower_id`, `following_id`, unique `(follower_id, following_id)`, indexed both ways.
- **Endpoints:** `POST /api/secure/follow/{targetUserId}` toggle (`controller/FollowController.java:18-22`); `POST /api/secure/user/follows` returns the **caller's own** followings only (`controller/UserController.java:41-44` → `getMyFollowings`, `currentUserId`). No endpoint exposes another user's follower/following list; only the boolean `isFollowingCurrent` (caller-follows-queried) on `UserInfoDTO`. No public follower/following counts.
- **Hard delete:** erased in both directions — `service/impl/DefaultUserDeletionService.java:361`.

### (c) Report target types — `PRODUCT`, `USER`, `REVIEW`

`entity/ReportType.java:3-7`. **No conversation/message report type.**

### (d) Delete-own endpoints

- **Delete-own-listing: exists** — `DELETE /api/secure/products?productId` → `changeProductState(productId, ProductState.DELETED)` (`controller/DashboardProductController.java:82-87`; facade `DefaultProductFacade.changeProductState:65-66`, which enforces owner-scoping in the service layer).
- **Delete-own-review: NOT FOUND** — `controller/ReviewController.java` (`/api/secure/review/product`) exposes only `GET /{targetProductId}`, `POST`, `GET` — no `DELETE`. Users cannot self-delete a review; only admin moderation removes reviews.

---

## Q5 — Translation payload

### (a) Everything in the OpenAI request — **VERIFIED-TRUE** (doc: only the text is sent)

`openai/service/impl/DefaultOpenAIService.sendRequest:52-114`:
- POST to `apiUrl`; headers: `Authorization: Bearer <key>` and `Content-Type: application/json` only (`:66-67`). **No custom/metadata headers.**
- Body = `ResponseRequest` (`:139-158`): `{ model, input, reasoning.effort, store=false, top_p=1.0, presence_penalty=0.0, frequency_penalty=0.0 }`. **No `user` field.**
- `input` = formatted prompt only: configured template + language labels + language codes + the text — `openai/facade/impl/DefaultOpenAIFacade.getFormattedPrompt:118-122`. **No user ID, email, listing/review ID, or other metadata.**
- `store=false` (`:153`) — OpenAI is asked not to retain the request.

### (b) Content types translated; chat status — **VERIFIED-TRUE**

Callers of `openAIFacade.translateTextWithPrompt`: product title/description (`service/impl/DefaultProductService.updateTranslations`), profile bio (`service/impl/DefaultUserTranslationsService.generateUserTranslations`), review comments (`service/impl/DefaultReviewTranslationsService.generateReviewTranslations`); plus a product-description AI suggestion (product name only, `DefaultOpenAIFacade.getOpenAIDescriptionSuggestion:47-69`). The `messaging/` package contains no OpenAI call.

**Plain answer:** Only the text (wrapped in the language-instruction prompt) is sent — no identifiers, no `user` field, no custom headers; listing title/description, profile bio, and review comments are translated, and **chat messages are never sent to OpenAI.**

---

## Q6 — Automated enforcement and view counting

### (a) Fully-automated enforcement paths — **PARTIAL** (vs PP §9 "all enforcement decisions … made by a human administrator")

- **Content moderation on create/edit is validation-only:** `validator/ProductNameValidator` / `ProductDescriptionValidator` return `false` → 4xx; nothing changes listing or account state. `DefaultProductService.createProduct` always persists `ACTIVE` + `APPROVED`. Ban-product and ban-user require a human (`admin/.../DefaultAdminProductFacade`, `DefaultUsersFacade.disableUser`). The `numOfPenalties` counter is incremented only by an admin review action (`admin/service/impl/DefaultAdminReviewService.java:91`) and is never read to auto-act.
- **One fully-automated enforcement path exists:** the Firebase-orphan cascade `service/impl/DefaultUserDeletionService.markForImmediateDeletion:258-285` runs a §3.12 abuse check (`:262-277`) that, when an orphaned user has open reports above threshold, **auto-inserts a ban hash** (`recordBannedUser(..., null)` — `adminId=null`, system actor) and then hard-deletes — no human in the loop. It is triggered by the Firebase account disappearing, not by content.

**Plain answer:** Content validation never auto-blocks/bans/locks, but the firebase-cascade abuse check is a fully automated ban+delete path with no human action, so PP §9's absolute wording is not strictly accurate.

### (b) View/favorite counters — **PARTIAL** (vs PP §2.4 "anonymous totals only")

`service/impl/DefaultProductSeenService.java`:
- Per-view **dedup key stored in Redis**: `view:{productId}:{deviceId|ip}` (`:61`), where the id is the `X-Device-Id` header else `CF-Connecting-IP`/`getRemoteAddr` (`:48-60`), written via `setIfAbsent(... , Duration.ofMillis(windowMs))` with TTL `redis.product.view.dedup.window.ms` default **43,200,000 ms = 12 h** (`:74-79`).
- Owner-skip cache key `product:owner:{productId}`, TTL `redis.product.view.owner.ttl` default 24 h (`:37-43`).
- The persisted **count is an anonymous total**: `Product.numberOfViews` + Redis delta `product:views:delta:{productId}`.
- Favorites: tracked per-user via the `user_favorite_products` join (`entity/User.java:105-110`) and an anonymous `Product.numberOfFavorites`; **no IP/device dedup key** (`facade/impl/DefaultFavoriteProductFacade.java:44-94`).

**Plain answer:** Stored counters are anonymous totals, but view counting **does** keep a short-lived per-device/per-IP dedup record in Redis (`view:{productId}:{deviceId|ip}`, 12 h TTL) — so PP §2.4's "anonymous totals only" understates a transient per-IP/device key.

---

## Q7 — Ops and infrastructure

### (a) Logs — IPs/user IDs, destination, 90-day purge — **partly VERIFIED-FALSE**

- Logs **include client IP and user ID**: `logging/RequestLoggingFilter.java:55-99` puts `clientIp` (X-Forwarded-For first hop, else `getRemoteAddr`, `:92-99`) and reads `userId` from MDC (set by `security/filter/FirebaseAuthFilter.java:122`); console pattern `application-prod.yaml:171` = `[req=%X{requestId} usr=%X{userId} ip=%X{clientIp}]`. One `ACCESS` line per request.
- **Destination: stdout/console only.** No logback XML, no `RollingFileAppender`/`FileAppender`/`maxHistory` anywhere (`rg` over `src/main/resources` → none). Captured by the container log driver.
- **90-day purge (PP §8.2): NOT FOUND** in code — no in-app retention/rotation. Any retention depends on the external Docker/host log driver.

### (b) Alerts / incident_log PII + retention — **no PII; no incident_log purge**

`health/AlertService.buildBaseBody:139-158` / `buildCriticalBody:161-177`: body = pressure level, time, pool metrics (active/idle/total/max/threads-awaiting), incident-row id, auto-trip outcome, and `pg_stat_statements` normalized query text. **No IPs, emails, or user IDs.** `incident_log`: `repository/IncidentLogRepository.java` is insert/read only (no delete); no purge job → rows **persist indefinitely** (operator psql cleanup only).

### (c) Where Redis and Elasticsearch run — both **self-hosted Docker** (correction)

- **Redis:** self-hosted container `redis:7-alpine` on the app host — `infra/docker-compose.yml:107-108`.
- **Elasticsearch:** self-hosted container `elasticsearch:9.0.1` — `infra/docker-compose.es.yml` (`name: oglasino-es`), bound to the **private VPC IP** `10.114.0.4:9200`, data volume `/opt/oglasino-es/data`. **Not** a DigitalOcean-managed or third-party-SaaS Elasticsearch. The app connects via `ELASTICSEARCH_URIS` (`application-prod.yaml:69-74`).

**Plain answer:** Both Redis and Elasticsearch are first-party self-hosted Docker services (Redis on the app host; ES as a separate `oglasino-es` compose on a VPC-private droplet) — so PP §4 listing no third-party processor for either is consistent.

### (d) Postgres managed? Admin-access audit log? — **DO-managed Postgres; PARTIAL on admin audit**

- **Postgres = DigitalOcean Managed DB:** `DATASOURCE_URL` env-injected; `do-postgres-ca.crt` mounted (`infra/docker-compose.yml:87`); Hikari tuned "for DO Basic 1GB Postgres (25 conn limit)" (`application-prod.yaml:14`). At-rest encryption is a DO-platform property, not in code.
- **Admin-access audit log:** there is **no dedicated admin login/session audit table.** `entity/UserAdminActionAudit.java` records admin **actions** (UNBAN; bans go through `AdminActionLog.message` → `log.warn` at `DefaultUsersFacade:108-114`); admin HTTP requests are captured in the application ACCESS log with `userId`. So PP §10 "logging and monitoring of administrative access" is only **partially** backed by code.

### (e) Email verification / password reset sender — **VERIFIED-TRUE (Brevo sends)**

Firebase Admin SDK **mints the OOB link** — `service/impl/VerificationEmailSender.java:75` (`generateEmailVerificationLink`), `service/impl/PasswordResetEmailSender.java:81` (`generatePasswordResetLink`) — the backend extracts the `oobCode`, builds its own `<web>/<locale>/verify|reset?oobCode=…` URL, and **sends via Brevo SMTP** (`emailService.sendHtml` → `application-prod.yaml` mail block `:76-95`, `smtp-relay.brevo.com`). Firebase's built-in mailer is **not** used for delivery.

---

## Adjacent findings (contradict or qualify doc claims, not directly asked)

1. **Banned-user profile leak via firebaseUid lookup (Medium).** `repository/UserRepository.findUserInfoById:155` filters `u.disabled = false` (banned → 404), but `findUserInfoByFirebaseUid:160-193` has **no such filter** — a banned user's public info is still returned when fetched by firebaseUid. Inconsistent with the "banned users are indistinguishable from non-existent" posture documented at `:119-124`.
2. **Elasticsearch docs survive hard delete (Medium).** `runHardDelete` issues no ES delete (no per-product event on the cascade delete); product docs orphan in ES until the next full reindex / drift reconciliation. Any PP statement implying complete removal of listing data at deletion overstates ES behavior.
3. **Chat message text is never deleted/anonymized.** The Sunday cleanup only rewrites `senderId`/`receiverId`/`users[]` to a sentinel; the **message body content of a deleted user remains** in Firestore indefinitely. Relevant to any PP claim about message-content erasure.
4. **Transient Redis keys not purged at hard delete (Low).** View dedup (`view:…`), owner cache (`product:owner:…`), and report dedup keys are left to expire by TTL; they may briefly outlive the deleted user.
5. **`numOfPenalties` is write-only (Low).** Incremented by admin review (`DefaultAdminReviewService:91`) but never read for enforcement — effectively dead as an enforcement signal.
6. **`user.deletion.report.window.days` (id 50, =7) defined but not independently enforced (Low).** Report handling at hard delete keys off ban status, not this separate window; the knob exists (`data-configuration.sql:87`) for future divergence from the grace period. PP text relying on a distinct "report window" should note it currently mirrors the grace period and has no separate enforcement.
