# User deletion audit — oglasino-backend

**Date:** 2026-05-17
**Branch:** dev
**Repo:** oglasino-backend
**Scope:** read-only inventory per `.agent/brief.md`. No code changes, no schema changes. No new files outside `.agent/audit-user-deletion.md` and `.agent/last-session.md`.

---

## 1. User entity and schema

### `User` JPA entity (`src/main/java/com/memento/tech/oglasino/entity/User.java`)

`@Entity @Table(name="users")`. Indexes: `idx_user_email` (unique), `idx_user_firebase_uid` (unique), `idx_user_base_site`, `idx_user_region`, `idx_user_city`.

Fields, with their JPA annotations:

| Field | Type | Constraints / annotations |
|---|---|---|
| `id` | `Long` | `BaseEntity` — `@Id`, sequence `global_seq`, not null, unique |
| `createdAt` / `updatedAt` | `LocalDateTime` | `BaseEntity` — `@CreationTimestamp` / `@UpdateTimestamp` |
| `firebaseUid` | `String` | `@Column(unique=true)` — nullable in entity |
| `email` | `String` | `@Column(unique=true, nullable=false)` |
| `displayName` | `String` | `@Column(nullable=false)` |
| `phoneNumber` | `String` | nullable |
| `userRole` | `UserRole` | `@Enumerated(STRING)`, default `ROLE_BASIC` |
| `disabled` | `boolean` | `@Column(nullable=false, columnDefinition="boolean default false")` |
| `lastBaseSiteChange` | `LocalDateTime` | nullable |
| `emailVerifiedExternal` | `boolean` | |
| `verifiedInternally` | `boolean` | |
| `registeredWithProvider` | `String` | nullable (this is the `signInProvider` field — see below) |
| `subscriptionType` | `UserSubscription` | `@Enumerated(STRING)` — `SUBSCRIPTION_FREE` / `SUBSCRIPTION_PREMIUM` |
| `subscriptionActive` | `boolean` | |
| `profileImageKey` | `String` | R2 key, full path with prefix (`public/profiles/{uuid}.{ext}`) |
| `rating` | `double` | |
| `preferredLanguage` | `Language` | `@ManyToOne(optional=false)`, `@JoinColumn(name="preferred_language_id", nullable=false)` — default fetch |
| `translations` | `Set<UserTranslation>` | `@OneToMany(mappedBy="user", cascade=ALL, orphanRemoval=true, fetch=LAZY)` |
| `baseSite` | `BaseSite` | `@ManyToOne(fetch=LAZY)` — nullable |
| `region` | `Region` | `@ManyToOne(fetch=LAZY)` — nullable |
| `city` | `City` | `@ManyToOne(fetch=LAZY)` — nullable |
| `products` | `Set<Product>` | `@OneToMany(mappedBy="owner", cascade=ALL, orphanRemoval=true, fetch=LAZY)` |
| `favoriteProducts` | `Set<Product>` | `@ManyToMany(cascade=PERSIST, fetch=LAZY)` → `user_favorite_products` join table |
| `allowPreferenceCookies` / `allowNotifications` / `allowEmails` / `allowPromoEmails` / `allowPhoneCalling` | `boolean` | the five consent toggles |
| `numOfPenalties` | `int` | `@Column(nullable=false, columnDefinition="integer default 0")` |

### `users` table schema (`V1__init_schema.sql:575-605`)

Columns mirror the entity. Constraints / indexes:

- PK: `users_pkey` on `id`.
- Unique: `users_email_key` (`email`), `users_firebase_uid_key` (`firebase_uid`).
- Check: `users_user_role_check` ∈ {`ROLE_ADMIN`, `ROLE_BASIC`}; `users_subscription_type_check` ∈ {`SUBSCRIPTION_FREE`, `SUBSCRIPTION_PREMIUM`}.
- FKs out of `users`: `preferred_language_id` → `language(id)` (NOT NULL); `base_site_id`, `region_id`, `city_id` → respective tables (nullable).
- FKs into `users`: `product.owner_id`, `review.reviewer_id` (NOT NULL), `review.target_user_id` (NOT NULL), `report.reporter_id` (nullable), `report.reported_user_id` (nullable), `user_follow.follower_id`, `user_follow.following_id`, `push_token.user_id`, `suggestion.user_id`, `user_favorite_products.user_id`, `product_audit.user_id`.

### Existing soft-delete fields

**None found.** A repo-wide grep for `deletion_status`, `deleted`, `is_deleted`, `soft_delete`, `softDelete`, `deletedAt`, `scheduled_for_deletion`, `scheduledForDeletion` returns no matches in `src/main` or `src/test`. The deletion-status, deletion-scheduled-at, ban-related, and lock fields specified in the deletion-feature drafts do not exist yet.

### `signInProvider` equivalent — `registeredWithProvider`

Stored as a free-form `String` (`character varying(255)`, nullable). Written in exactly three places in `src/main`:

- `DefaultFirebaseAuthService.getOrCreateUser:85` — back-fills the value on first authenticated request if it was previously blank.
- `DefaultFirebaseAuthService.createUserSynchronized:106` — sets it on new-user creation.
- `TestUsersImportService:58` — fixture seeding only.

Source value: the raw `firebase` claim from the verified ID token, coerced to string via `Objects::toString`:

```
var claims = token.getClaims();
var providerId = Optional.ofNullable(claims.get("firebase")).map(Object::toString).orElse("");
```

So the value is not a clean enum like `password` / `google.com`; it is `Object::toString` over the entire `firebase` claim object as Firebase Admin deserialises it. Inspect a real row before relying on its shape for re-registration-prevention logic.

---

## 2. Auth filter and session sync

### `FirebaseAuthFilter` (`security/filter/FirebaseAuthFilter.java`)

After successful `verifyIdToken`:

1. Looks up auth data via `firebaseAuthService.getCachedAuthData(uid)` → `Optional<AuthenticatedUserDTO>`.
2. Builds `OglasinoAuthentication(userId, firebaseUid, baseSiteId, preferredLanguage, authorities)` from the DTO.
3. Sets it into `SecurityContextHolder` and stamps `MDC.put("userId", ...)`.
4. Falls through to `filterChain.doFilter(...)`.

**`authData.disabled()` is never consulted.** The DTO carries the field (`AuthenticatedUserDTO.disabled`) and the repository projection populates it (`UserRepository.findAuthDataByFirebaseUid` selects `u.disabled`), but no caller reads it. `OglasinoAuthentication` does not expose `disabled` and the filter has no branch on it. Confirms the state.md backlog entry "Account-disabling & token-revocation enforcement" verbatim.

The filter exempts `firebase-sync` from token verification (path-contains check at line 60) so the first sync-on-login call can pass without the user-row existing yet.

There is also a Postman testing bypass (`validateForPostmanTesting`) that activates only when `spring.profiles.active=dev` AND `X-Postman: true` AND DB config `postman.testing.active=true` — selects a user by `X-User` header (default id 2) and builds a full auth without any token verification. Not a production trust-boundary issue, but worth being aware of.

### `redisUserAuth` cache (`config/RedisConfig.java:51-62`)

- Cache name: `redisUserAuth`.
- Key: `firebaseUid` (from `@Cacheable(value="redisUserAuth", key="#firebaseUid")` on `DefaultFirebaseAuthService.getCachedAuthData`).
- Value: `Optional<AuthenticatedUserDTO>` — flat record with `userId`, `firebaseUid`, `disabled`, `userRole`, `subscriptionType`, `subscriptionActive`, `baseSiteId`, `preferredLanguage` (`LanguageDTO`). Empty Optionals are not cached (`disableCachingNullValues()`).
- TTL: **30 minutes** — `Duration.ofMinutes(30)`. Matches the 2026-05-14 decision. Documented at the call site as a backstop only; freshness is owned by explicit `@CacheEvict`.
- Populated by `FirebaseAuthFilter` on every authenticated request (cache-miss runs the constructor-projection JPQL in `findAuthDataByFirebaseUid`).
- Evicted in `DefaultUserService.saveUser` via `@CacheEvict(value="redisUserAuth", key="#currentUser.firebaseUid")`.

### `redisUserInfo` cache (`config/RedisConfig.java:38-49`, `service/impl/DefaultUserService.java:31-49`)

- Cache name: `redisUserInfo`.
- Dual-keyed: written by both `getUserInfoForFirebaseUid(firebaseUid)` and `getUserInfoForId(userId)` — two separate cache entries per user, one keyed by `firebaseUid` and one keyed by `userId`.
- Value: `UserInfoDTO` (display name, profile image, short bio, rating, verified, active-products count, allow-phone-calling, base-site overview, region/city DTO). Built from `UserInfoProjection` JPQL.
- TTL: **30 minutes**.
- Evictions in `DefaultUserService.saveUser` cover both keys: `@CacheEvict(value="redisUserInfo", key="#currentUser.id")` + `@CacheEvict(value="redisUserInfo", key="#currentUser.firebaseUid")`. Same dual-evict in `evictUserInfoCache(User)`.
- Skipped by `CacheWarmupService` by design (per-user, can't warm up-front).

Relationship between the two caches: independent. `redisUserAuth` is the request-time hot path (every authenticated request); `redisUserInfo` is the per-user public-profile path (loaded when rendering `/user/{id}` or similar). Both live for 30 min, both must be evicted together on any user-row write. `saveUser`'s `@Caching(evict={...})` block does this in one place; `evictUserInfoCache` is a no-write helper for product-side flows that change activeProducts count (called from `DefaultProductService.createProduct` / `changeProductState` / `deleteProduct`).

### `syncUserWithBackend` — `POST /api/auth/firebase-sync` (`controller/AuthController.java:35-46`)

- Accepts `LoginRequest` body (currently has `allowPreferenceCookies` at minimum, plus whatever fields exist on `LoginRequest`).
- Returns `AuthUserDTO` (`dto/AuthUserDTO.java`) on success — 200; 500 only if the service returns null. There is no other status path.
- Called by the frontend on every login to ensure the backend `User` row exists (matches the `getOrCreateUser` shape).

**Does the response body have a banned/disabled signal? No.** `AuthUserDTO` fields are:

```
id, firebaseUid, displayName, email, baseSite, regionAndCity,
profileImageKey, providerId, allowPreferenceCookies, allowNotifications,
allowEmails, allowPromoEmails, allowPhoneCalling
```

There is no `disabled`, `banned`, `scheduledForDeletion`, `numOfPenalties`, or any other state field that the frontend could read to detect that the user has been disabled or scheduled for deletion. **This is the seam.** The same goes for `GET /api/auth/firebase/{firebaseId}` which returns `UserInfoDTO` — also no disabled-flag field.

---

## 3. Review entity

### `Review` JPA entity (`entity/Review.java`)

`@Table(name="review")`. Indexes: `idx_review_reviewer`, `idx_review_target_user`, `idx_review_target_product_id`, `idx_review_approved`.

Fields:

| Field | Type | Annotations |
|---|---|---|
| `reviewer` | `User` | `@ManyToOne(fetch=LAZY, optional=false)` |
| `targetUser` | `User` | `@ManyToOne(fetch=LAZY, optional=false)` |
| `targetProductId` | `Long` | `@Column` — nullable in entity |
| `rating` | `int` | `@Min(1) @Max(5) @Column(nullable=false)` |
| `translations` | `Set<ReviewTranslation>` | `@OneToMany(mappedBy="review", cascade=ALL, orphanRemoval=true, fetch=EAGER)` |
| `imageKeys` | `Set<String>` | `@ElementCollection(fetch=LAZY) @CollectionTable(name="review_images", joinColumns=@JoinColumn(name="review_id")) @Column(name="image_key")` — review images live under `public/products/` per Worker contract §6 (shared prefix with products) |
| `approved` | `Boolean` | nullable (tri-state: null = pending, true = approved, false = rejected) |
| `verified` | `boolean` | `@Column(nullable=false)` |

### `review` table schema (`V1__init_schema.sql:474-485`)

- `reviewer_id bigint NOT NULL` — FK `fk29sgaw0fsbkrgfd8gv15j9vvk` → `users(id)`.
- `target_user_id bigint NOT NULL` — FK `fk2ubqmwkytvonateoxr0554nfe` → `users(id)`.
- `target_product_id bigint` — nullable, no FK (just a Long column).
- `rating integer NOT NULL`, `approved boolean`, `verified boolean NOT NULL`.
- CHECK constraint `review_rating_check` ∈ [1, 5].
- `review_images` table: `(review_id, image_key)` — `image_key` is `varchar(255)`, nullable in DDL.
- `review_translation` table: `(review_id NOT NULL, language_id NOT NULL)` with `unique(review_id, language_id)`; columns `comment text NOT NULL`, `product_capture_name text`, `product_capture_description text`, `disapproval_reason text`, `auto_translated`, `translation_provider`.

**Implication for hard delete.** `review.reviewer_id` and `review.target_user_id` are both `NOT NULL` FKs to `users(id)`. Deleting a user that has either authored reviews (as reviewer) or received reviews (as target) violates these constraints. The spec's "kept (anonymized)" outcome for the user-as-reviewer case requires either replacing the FK with a sentinel "Deleted User" row or making `reviewer_id` nullable. Same for `target_user_id` (user-as-reviewee).

### Reviewer display name — joined live

`ReviewConverter.convert` (`converter/ReviewConverter.java:32`) maps `source.getReviewer()` to `ReviewerInfo` via `modelMapper`. `ReviewerInfo` carries `id`, `displayName`, `profileImageKey` — all pulled live from the joined `User` entity (the queries in `ReviewRepository` use `join fetch r.reviewer rev join fetch rev.preferredLanguage lang`). **Display name is NOT denormalized onto the review row.** A spec that anonymizes a deleted user's reviews by mutating `User.displayName = "Deleted User"` will propagate automatically to every review read — no review-row rewrite needed.

### Review write path (`service/impl/DefaultReviewService.java:39`)

`DefaultReviewService.reviewProduct(ReviewRequestDTO)`:

- `reviewer` ← `entityManager.getReference(User.class, currentUserService.getCurrentUserIdStrict())` — server-derived from auth context.
- `targetUser` ← `entityManager.getReference(User.class, productService.getOwnerIdForProductId(reviewRequest.targetProductId))` — server-derived from the product's owner FK.
- `targetProductId` ← `reviewRequest.targetProductId` (client-supplied, validated by lookup).
- `imageKeys` ← `reviewRequest.imageKeys` (client-supplied; no ownership check visible in this method).
- `verified` ← `!reviewRequest.isSkipTrust()` — client-supplied boolean.
- Optional trust check via `trustReviewService.canReviewProduct(...)` gated by the same client-supplied `skipTrust`.

`ReviewRequestDTO` has `@NotNull targetProductId`, `@NotEmpty @Size(50, 1000) review`, `@Min(1) @Max(5) rating`, plus `skipTrust` and `imageKeys`. The trust validations live in `DefaultTrustReviewService` (Firestore-driven, see §8).

### Review images on R2

Stored in `review_images` join table; keys live under `public/products/` (review images piggyback on the products prefix — confirmed in the `Review.imageKeys` javadoc and `ReviewRepository.existsByImageKey` javadoc). Naming: same `public/products/{uuid}.{ext}` shape as product images — there is no separate prefix or marker that distinguishes review images from product images in R2. Both `ProductRepository.existsByImageKey` and `ReviewRepository.existsByImageKey` are checked in `DefaultImageDeletionFacade.isInUse` for the `public/products/` prefix.

---

## 4. Report entity

### `Report` JPA entity (`entity/Report.java`)

`@Table(name="report")`. Indexes: `idx_report_reporter`, `idx_report_reported_user`, `idx_report_reported_product`, `idx_report_finished`.

Fields:

| Field | Type | Annotations |
|---|---|---|
| `reporter` | `User` | `@ManyToOne(fetch=LAZY)` — **no `optional=false`** |
| `reportedUser` | `User` | `@ManyToOne(fetch=LAZY)` — **no `optional=false`** |
| `reportedProductId` | `Long` | `@Column` — nullable |
| `description` | `String` | `@Column(nullable=false, columnDefinition="TEXT")` |
| `reportOption` | `ReportOption` | `@Enumerated(STRING)` — nullable in entity |
| `reportType` | `ReportType` | `@Enumerated(STRING)` — nullable in entity |
| `finished` | `boolean` | default false |
| `resolution` | `String` | nullable |

### `report` table schema (`V1__init_schema.sql:453-467`)

- `reporter_id bigint` — **nullable**, FK `fkqbhdxqd3ly7fkhly5nrl2j93k` → `users(id)`.
- `reported_user_id bigint` — **nullable**, FK `fk2fm8nu7yscahr6sbhhgw082mp` → `users(id)`.
- `reported_product_id bigint` — nullable, no FK constraint.
- `description text NOT NULL`.
- CHECK `report_report_option_check` ∈ {`FRAUD`, `POOR_SERVICE`, `RULES_VIOLATION`, `VIOLENCE_OR_HARASSMENT`, `INAPPROPRIATE_CONTENT`, `MISLEADING_OR_FAKE_PRODUCT`, `PORTAL_RULES_VIOLATION`, `TECHNICAL_ISSUE`, `OTHER`}.
- CHECK `report_report_type_check` ∈ {`PRODUCT`, `USER`}.

**Implication for hard delete.** Both `reporter_id` and `reported_user_id` are **nullable** at the DB level and **not declared `optional=false`** on the entity. The 2026-05-15 decisions.md entry stated `Report.reporter` and `Review.reviewer` are non-nullable @ManyToOne FKs that will block hard-deletion — that is correct for `Review.reviewer`/`Review.targetUser` but **not** for `Report.reporter`/`Report.reportedUser`. Setting them to null is permitted by both the entity model and the table schema; no FK migration is needed for report-side anonymization.

### Report read path for admin

- `AdminReportController.getReports` → `DefaultAdminReportFacade.getReports` → `ReportRepository` (specification-based filtering) → `modelMapper.map(report, ReportDTO.class)`.
- Per `AdminReportFacade`, the surface returns `PageDTO<ReportDTO>`. The `Report.reporter` / `reportedUser` get mapped through ModelMapper, which navigates the `User` association and pulls the live `displayName` (same propagation pattern as reviewer name).

### Report category enum vs brief

`ReportOption` enum (`entity/ReportOption.java`) and the DB CHECK constraint both enumerate exactly nine values, matching the brief's list one-for-one:

| Brief category | Enum value |
|---|---|
| fraud | `FRAUD` |
| poor service | `POOR_SERVICE` |
| rules violation | `RULES_VIOLATION` |
| violence or harassment | `VIOLENCE_OR_HARASSMENT` |
| inappropriate content | `INAPPROPRIATE_CONTENT` |
| misleading or fake listing | `MISLEADING_OR_FAKE_PRODUCT` |
| portal rules violation | `PORTAL_RULES_VIOLATION` |
| technical issue | `TECHNICAL_ISSUE` |
| other | `OTHER` |

`ReportType` is a separate, smaller enum: `PRODUCT`, `USER` — selects whether the report's target is a product (`reportedProductId`) or a user (`reportedUser`).

---

## 5. Product / listing entity and status

### `ProductStatus` enum

The codebase uses two enums (not one named `ProductStatus`):

- `ProductState` (`entity/ProductState.java`): `ACTIVE`, `INACTIVE`, `DELETED` — **`INACTIVE` exists**.
- `ModerationState` (`entity/ModerationState.java`): `APPROVED`, `BANNED`.

The spec's mention of "set `ProductStatus.INACTIVE`" maps to `Product.productState = INACTIVE`. There is also an explicit `DELETED` state used by the existing soft-delete-then-sweep flow on the product side.

### `products` table schema (`V1__init_schema.sql`)

`product_state varchar NOT NULL`, `moderation_state varchar NOT NULL`, `owner_id bigint NOT NULL` → `users(id)`. Indexes: `idx_product_site`, `idx_product_category`, `idx_product_city`, `idx_product_owner`. No state-column index (the production read path is Elasticsearch — see below — not direct SQL filtering).

### Portal read path — Elasticsearch only

Every "products visible on portal" surface goes through Elasticsearch via `DefaultProductsSearchFacade` / `DefaultProductsFilterQueryBuilder`. The state filter is applied in `ProductStateQueryGenerator.wrapWithStateSearchMode`:

```
PORTAL_SEARCH:
  filter productState == ACTIVE
  filter moderationState == APPROVED
DASHBOARD_SEARCH:
  productStates restricted to ALLOWED_DASHBOARD_PRODUCT_STATES = {ACTIVE, INACTIVE}
ADMIN_SEARCH:
  productStates: client-controllable
```

This wrapping is applied uniformly via:

- `buildSingleProductIdQuery(productId, mode)` — `getProductDetailsForId` (single-product page).
- `buildMultiProductIdQuery(productIds, mode)` — `getProductsForIds` (favorites, batch by id).
- `buildQuery(searchProducts, mode)` — main paginated list / search / autocomplete.

Every portal-facing controller passes `ProductsSearchMode.PORTAL_SEARCH`:

- `ProductSearchController.getProductDetailsForId` (single product) — line 38.
- `ProductSearchController.getProductOverviews` (catalog list) — line 53.
- `ProductSearchController.getAutocomplete*` — line 63.
- `DefaultFavoriteProductFacade.getFavoriteProducts` — line 77.

**There is no SQL-only read path that bypasses ES for portal display.** The DB-level `ProductRepository.findById` is used in admin / write-side flows (`DefaultProductService.changeProductState` for the ownership check, the reindex paths, etc.), not for serving portal reads. **The `Set<Product> products` collection on `User` is never read for display** — it exists only for the JPA-cascade relationship.

There is no in-tree sitemap, "recommendations," or "related products" code in the backend — the sitemap lives in `oglasino-web` (per the 2026-05-16 issues.md entry on `app/sitemap.ts:79-83`) and an `EXTRA_PRODUCTS` translation namespace exists for "related/suggested" but no backend service emits it from a state-unfiltered query.

**Conclusion: setting `productState = INACTIVE` on every product the deleting user owns is sufficient to hide them from every portal-facing read path the backend serves**, provided the ES doc is repopulated/patched for each — which it is, via the event listener (see below).

### ES reindex trigger

`DefaultProductService.changeProductState` (line 633) publishes `ProductStateUpdateEvent(productId, productState)` after the DB save. `ProductIndexerEventListener.handleProductStateUpdated` (`listeners/ProductIndexerEventListener.java:37`) listens with `@TransactionalEventListener(phase=AFTER_COMMIT)` `@Async` `@Retryable`. The listener issues a partial `UpdateQuery` against the `products` index with just the `productState` field. So a per-product state flip propagates to ES asynchronously after commit. Full product updates use `ProductUpdateEvent` → `productIndexer.indexOne(id)` instead.

Note that `changeProductState` already does a `currentUserId.equals(product.getOwner().getId())` ownership check — calling it as the deleted-user-themselves works; an admin-initiated mass-state-change for the deletion sweep cannot reuse this method as-is (the check would reject), and would need a new service path or skip-ownership variant.

### Product images on R2

Naming convention: `public/products/{uuid}.{ext}` (`ImagePaths.productKey` — `images/path/ImagePaths.java:34`). The full key is stored verbatim on `Product.imageKeys` (a `@ElementCollection` set; `product_images` join table).

---

## 6. Translation namespaces

### `TranslationNamespace` enum (`entity/TranslationNamespace.java`)

22 values, in code order: `COMMON`, `COMMON_SYSTEM`, `ERRORS`, `VALIDATION`, `PAGING`, `BUTTONS`, `INPUT`, `DIALOG`, `HEADER`, `FOOTER`, `NAVIGATION`, `INTRO`, `EXTRA_PRODUCTS`, `COOKIES`, `MESSAGES_PAGE`, `DASHBOARD_PAGES`, `ADMIN_PAGES`, `ABOUT_PAGE`, `FREE_ZONE_PAGE`, `PRICING_PAGE`, `METADATA`, `BACKEND_TRANSLATIONS`. Matches `meta/conventions.md` Part 6 Rule 1 verbatim.

### Translation entities on user-content

- `UserTranslation` (`entity/UserTranslation.java`) — table `user_translation`, unique constraint on `(user_id, language_id)`. FK `user_id NOT NULL` to `users(id)`. Field: `shortBio` (TEXT, not null), plus `autoTranslated` and `translationProvider`. Parent side on `User`: `@OneToMany(mappedBy="user", cascade=ALL, orphanRemoval=true, fetch=LAZY)`. **Confirmed: `orphanRemoval = true` on the parent — translations die with the parent row.**
- `ProductTranslation` (`entity/ProductTranslation.java`) — same shape: FK `product_id NOT NULL`, unique `(product_id, language_id)`. Parent side on `Product`: `@OneToMany(mappedBy="product", cascade=ALL, orphanRemoval=true, fetch=LAZY)`. **Confirmed.**
- `ReviewTranslation` (`entity/ReviewTranslation.java`) — same shape: FK `review_id NOT NULL`, unique `(review_id, language_id)`. Parent side on `Review`: `@OneToMany(mappedBy="review", cascade=ALL, orphanRemoval=true, fetch=EAGER)`. **Confirmed.** (Note `fetch=EAGER` here — the other two are LAZY.)

In all three cases the exact annotation form on the parent side is `cascade = CascadeType.ALL, orphanRemoval = true`.

### `COMMON` namespace today

The 0001 SQL seed file has 51 `COMMON` keys; the 0002 file has 0. Existing user-related `COMMON` keys include:

- `user.follow.added`, `user.follow.removed`
- `user.page.not.found.title`, `user.page.not.found.description`, `user.page.no.products.found`
- `user.details.verified.user.label`, `user.details.unverified.user.label`, `user.details.active.products`

**No existing `user.deleted` (or any `user.*.deleted` key) is seeded.** `COMMON` is the consistent home for cross-surface user-facing labels of this shape (reviews, chats, admin views), so adding `user.deleted` there fits the existing pattern. If the literal string is also emitted by the backend directly (e.g., in a push-notification body when the recipient sees a deleted user's name), it would belong in `BACKEND_TRANSLATIONS` instead — flag for the spec to decide.

---

## 7. Firebase Auth integration

### `FirebaseUserService` (`admin/service/FirebaseUserService.java` + `DefaultFirebaseUserService.java`)

Interface has exactly two methods today:

```
void disableUser(String firebaseUid);
void enableUser(String firebaseUid);
```

Both implementations issue a `FirebaseAuth.getInstance().updateUser(new UserRecord.UpdateRequest(uid).setDisabled(true|false))`. **There is no `deleteUser(uid)` method, no existence-check (`getUser(uid)` with `USER_NOT_FOUND` handling), no `revokeRefreshTokens`.** Both methods catch `Exception` and rethrow as `RuntimeException("Failed to ... Firebase user", e)` — no Firebase-specific error code handling.

### Firebase Admin SDK initialization (`security/config/FirebaseConfig.java`)

`@Configuration` class, `@PostConstruct` `init()`:

- Reads credentials path from `app.firebase.credentials-path` (env-var fallback: `FIREBASE_CREDENTIALS_PATH`). Defaults to `./secrets-local/firebase-service-account.json` in dev.
- Asserts the file exists and is readable; throws `IllegalStateException` otherwise.
- Idempotent: skips init if `FirebaseApp.getApps()` is already non-empty (handles tests).
- Calls `FirebaseApp.initializeApp(FirebaseOptions.builder().setCredentials(GoogleCredentials.fromStream(credentialsStream)).build())`.

Subsequent code reaches the SDK via the static `FirebaseAuth.getInstance()` / `FirestoreClient.getFirestore()`.

### `auth_time` claim

**Not read anywhere in `src/main`.** `grep -rn "auth_time\|getAuthTime\|\"auth_time\"" src/main/java` returns no matches. `DefaultFirebaseAuthService.verifyToken` returns `FirebaseToken`, and the only caller reads `token.getUid()`, `token.getEmail()`, `token.getName()`, `token.getPicture()`, `token.isEmailVerified()`, and the `firebase` claim — never `auth_time`. **The reauth-freshness check the deletion request will need does not exist in any form yet.**

---

## 8. Firestore integration

### Existing Firestore-using services

- `admin/service/impl/DefaultFirebaseChatService.java` — admin-side chat browser (paged chats per user, paged messages per chat, chat-membership check). Returns DTOs for the admin UI.
- `admin/service/impl/DefaultFirebaseStatsService.java` — `AggregateQuery.count()` for `notifications` and `chats` collections (dashboard counters).
- `service/impl/DefaultTrustReviewService.java` — `canReviewProduct` check that walks the chat between two users to confirm "they actually messaged about this product."
- `notifications/service/impl/DefaultNotificationsService.java` — writes notification docs.

All four touch Firestore through the static `FirestoreClient.getFirestore()` (no separate Firestore-config bean — initialised as a side effect of `FirebaseApp.initializeApp`).

### Message structure in Firestore (`chats/{chatId}/messages`)

From `DefaultFirebaseChatService` and `DefaultTrustReviewService`:

- Top-level collection: `chats`.
- Chat document fields used by the backend: `users` (array of Firebase UIDs), `lastMessage` (string), `lastUpdated` (Timestamp). Chat document IDs are deterministic: `Stream.of(uid1, uid2).sorted().collect(Collectors.joining("_"))` — produces `{uidA}_{uidB}` with the two UIDs alphabetically ordered.
- Subcollection: `chats/{chatId}/messages`.
- Message document fields read by the backend: `content`, `senderId` (Firebase UID), `receiverId` (Firebase UID), `seen` (boolean), `createdAt` (Timestamp), `productId` (number, optional — used by `DefaultTrustReviewService`).
- **Sender identification = Firebase UID, not display name.** No display name is stored on the message — the admin chat browser re-resolves it from Postgres via `userRepository.findChatUserForFirebaseUid(senderId)`.

### Notification structure in Firestore (`notifications/{firebaseUid}/userNotifications`)

From `DefaultNotificationsService.sendAsyncNotification`:

- Top-level collection: `notifications`.
- Document key per recipient: `firebaseUid` (the doc IS the recipient).
- Subcollection: `userNotifications`.
- Notification document fields: `title`, `description`, `seen`, `createdAt` (`FieldValue.serverTimestamp()`), `type`, `categoryId`, `data`.
- **No sender is stored on the notification document** — the document only encodes who receives it (via the parent doc id) and what it says. The "sender" concept lives outside Firestore; if a deleted user authored content that triggered a notification, the notification text may reference them by display name but the doc itself doesn't carry the sender Firebase UID.

### Update/delete by sender UID

**No existing methods to update or delete messages or notifications by sender UID.** Read paths query by chat (`whereArrayContains("users", uid)`) or by chat document id directly; no method iterates messages-where-senderId-equals across all chats. The orphan-by-sender sweep the deletion feature needs does not exist.

---

## 9. Scheduled jobs

Six `@Scheduled` methods exist:

- `DefaultScheduledRedisFlushService` — `fixedDelayString = "${app.views.flush-delay-ms}"` (20s in dev) — flushes per-product view-count buffer from Redis to DB.
- `ChatImagesRemovalJob` — cron `${app.images.chat.removal}` (Sundays 03:00 in dev) — sweeps `private/chats/` older than 30 days.
- `ProductImagesRemovalJob` — cron `${app.images.sweeper.cron}` (daily 03:00 in dev) — orphan sweeper for `public/products/` + `public/profiles/`. Builds the in-use set from `productRepository.findAllImageKeys` + `reviewRepository.findAllImageKeys` + `userRepository.findAllNonNullProfileImageKeys`; deletes anything in R2 not in that set and older than the grace period.
- `ProductBaseCurrencyUpdater` — `0 0 3 * * *` daily 03:00 — refreshes product base-currency conversions.
- `ProductRemovalJob` — `0 0 2 ? * SUN` Sundays 02:00 — paginates `productRepository.findByProductStateAndUpdatedAtBefore(DELETED, now - daysOld, page)` and hard-deletes each. Batch size and days-old are admin-configurable via `ConfigurationService` keys `product.removal.batch.size`, `product.removal.days.old`. **This is the closest existing precedent for the deletion-feature's hard-delete sweep.**
- `DefaultBaseCurrencyService` — `fixedRate=86400000` (24h) — refreshes base-currency rates from external API.

**No existing job iterates all users for any reason.** No audit-purge or retention-enforcement job exists. The deletion feature will need to add one.

The grace-period-then-hard-delete pattern is established in `ProductRemovalJob` (find rows by state-and-updated-before, paginate, delete one by one with per-row try/catch). The deletion-feature's user sweep can mirror this structure.

---

## 10. Image storage on R2

### Client setup

`DefaultR2Service` (`images/service/impl/DefaultR2Service.java`) holds an injected `S3Client r2Client` (AWS SDK v2). Configuration via `R2Properties` (`properties/R2Properties.java`) bound to `cloudflare.r2.*` yaml — `accountId`, `accessKey`, `secretKey`, `bucket`. **Single bucket** (`props.getBucket()` is one value, no per-prefix bucket split).

### Existing image deletion methods (`R2Service.java`)

- `void delete(String key)` — single object.
- `void deleteBulk(List<String> keys)` — batches at 1000 with `DeleteObjectsRequest.delete(Delete...objects(...).quiet(true))`. Logs per-key errors; swallows top-level exceptions with a logged error and continues.
- `void streamImagesOlderThan(folderName, olderThanCutoff, consumer)` — page-by-page list-and-consume for orphan sweepers, bounded heap (one ListObjectsV2 page in memory at a time).
- `Optional<ImageMetadata> headImage(String key)` — single-object HEAD for age-check / existence test.

There is no `deleteAllUnderPrefix(prefix)` method — the sweepers iterate via `streamImagesOlderThan` and call `deleteBulk` with the keys they've decided to remove.

### Naming conventions (`images/path/ImagePaths.java`)

- Profile pictures: `public/profiles/{uuid}.{ext}` (via `ImagePaths.profileKey(uuid, ext)`).
- Listing images: `public/products/{uuid}.{ext}` (via `ImagePaths.productKey(uuid, ext)`).
- Review images: **same prefix as listing images** — `public/products/{uuid}.{ext}` — per the Review.imageKeys javadoc and the Worker contract §6. There is no review-specific R2 prefix; the storage-key namespace is shared. The in-use check at `DefaultImageDeletionFacade.isInUse` queries both `productRepository.existsByImageKey(key)` and `reviewRepository.existsByImageKey(key)` for any key under `public/products/`.
- Chat attachments: `private/chats/{chatId}/{uuid}.{ext}`. The chat-attachment view JWT carries `keyPrefix=private/chats/{chatId}/`.
- Brand assets: `public/brand/` (manually managed; never stored in Postgres).

---

## 11. Cache eviction patterns

### Where `redisUserInfo` and `redisUserAuth` are evicted today

The single eviction hub is `DefaultUserService.saveUser` (`service/impl/DefaultUserService.java:62-75`):

```java
@Caching(evict = {
    @CacheEvict(value = "redisUserInfo", key = "#currentUser.id"),
    @CacheEvict(value = "redisUserInfo", key = "#currentUser.firebaseUid"),
    @CacheEvict(value = "redisUserAuth", key = "#currentUser.firebaseUid")
})
public User saveUser(User currentUser) { ... }
```

Plus `evictUserInfoCache(User)` for product-side flows that change the user's `activeProducts` count but don't mutate the user row:

```java
@Caching(evict = {
    @CacheEvict(value = "redisUserInfo", key = "#user.id"),
    @CacheEvict(value = "redisUserInfo", key = "#user.firebaseUid")
})
public void evictUserInfoCache(User user) { /* annotations do all the work */ }
```

`evictUserInfoCache` does **not** evict `redisUserAuth` — by design, since the product-side flows don't change anything `AuthenticatedUserDTO` carries.

There is also a `config/RedisCacheEvictConfig.java` that declares CACHES = {"redisUserInfo", "redisUserAuth"} (and probably others) — this is the admin-side cache-evict surface (`CacheAdminController`).

### Every code path that mutates `User.disabled`, `User.userRole`, `User.subscriptionType`, `User.subscriptionActive`

Grep across `src/main/java` (excluding test fixtures and DTO setters):

- `User.disabled`:
  - `DefaultUsersFacade.disableUser` → `user.setDisabled(true); userService.saveUser(user); firebaseUserService.disableUser(...)` ✓ routes through `saveUser`.
  - `DefaultUsersFacade.enableUser` → `user.setDisabled(false); userService.saveUser(user); firebaseUserService.enableUser(...)` ✓ routes through `saveUser`.
  - No other writers.
- `User.userRole`:
  - `DefaultFirebaseAuthService.createUserSynchronized` — sets `ROLE_ADMIN` or `ROLE_BASIC` on new-user creation, then `userService.saveUser(newUser)` ✓.
  - No other writers in main code.
- `User.subscriptionType`, `User.subscriptionActive`:
  - `DefaultFirebaseAuthService.createUserSynchronized` — sets `SUBSCRIPTION_FREE` and `subscriptionActive=true` on creation, then `saveUser` ✓.
  - No other writers in main code (no subscription-upgrade flow yet).

**Result: every mutation of an auth-relevant `User` field routes through `DefaultUserService.saveUser`.** The 2026-05-14 connection-pool decision's constraint holds.

### Redis CacheManager (`config/RedisConfig.java`)

- Backed by `RedisAutoConfiguration`'s `LettuceConnectionFactory` (deliberately not declared manually — see the comment at the top of the file warning against bypassing yaml config).
- `RedisCacheManager` with `defaultCacheConfig().disableCachingNullValues()` plus per-cache overrides for: `redisUserInfo` (30 min TTL, `UserInfoDTO`), `redisUserAuth` (30 min TTL, `AuthenticatedUserDTO`), `redisBaseSites` / `redisBaseSiteOverviews` / `redisLanguage` / `redisLanguages` / `redisBaseCurrency` (no TTL, populated by warmup, evicted via `CacheAdminController`).
- Each cache uses a per-type Jackson serializer (`JacksonJsonRedisSerializer<T>` from `tools.jackson.databind`).
- Eviction strategy: explicit `@CacheEvict`. No `unless`, no per-cache write-through. TTLs on the two user caches are backstops.

---

## 12. Configuration system

Three layers in active use:

1. **`application-{profile}.yaml`** with `@Value("${...}")` injection. The `app:` prefix tree is used for static infrastructure configuration: `app.internal.token`, `app.openai.api-key`, `app.exchange.api-key`, `app.security.recaptcha.*`, `app.security.allowed.cors`, `app.firebase.credentials-path`, `app.elasticsearch.reindex.batch-size`, `app.images.chat.removal`, `app.images.sweeper.enabled`, `app.images.sweeper.grace-period-hours`, `app.images.sweeper.cron`, `app.images.cdn-base-url`, `app.images.upload-token-ttl-ms`, `app.images.view-token-ttl-ms`, `app.images.max-upload-bytes`, `app.images.max-images-per-request`, `app.images.allowed-content-types`, `app.images.jwt.signing-secret`, `app.images.jwt.issuer`, `app.views.flush-delay-ms`. Cloudflare and Spring infra config sit alongside (`cloudflare.r2.*`, `spring.datasource.*`, etc.). These values cannot change at runtime — they're either env-var-resolved at boot or hardcoded into the YAML.

2. **`@ConfigurationProperties`-bound POJOs** like `R2Properties` (`@ConfigurationProperties(prefix="cloudflare.r2")`) — backed by the same YAML, just exposed as a typed bean.

3. **DB-backed `Configuration` entity + `DefaultConfigurationService`** — admin-tunable keys, hot-reloadable. `Configuration` table has `(key, value)` rows seeded by `data/configuration/*.sql`. `DefaultConfigurationService` loads the entire table into a `Map<String, String> configurationCache` at `ApplicationReadyEvent` (`@Order(1)`); `updateConfiguration` updates both the DB row and the in-memory map. Used today for: `product.removal.batch.size`, `product.removal.days.old`, moderation thresholds, banned-word lists, rate-limit configs, `postman.testing.active`.

**Convention for new `app.user-deletion.*` config keys.** The choice is between layers 1 and 3:

- If the value is admin-tunable at runtime without redeploy (`grace-period-days`, `audit-retention-banned-months`, `hard-delete-batch-size`, `sweep-cron-expression`) → DB-backed `Configuration` (layer 3), keyed via `ConfigurationService.getIntConfig` / `getRequiredIntConfig` etc. Mirrors `product.removal.*` precedent. Dot-separated key, not `app.`-prefixed.
- If the value is environment-shape (e.g., a feature-flag that gates a whole subsystem in a profile) → yaml `@Value("${app.user-deletion....}")` (layer 1).

The existing `product.removal.batch.size` (DB-backed, dot-separated, no `app.` prefix) is the closer parallel; the spec's proposed `app.user-deletion.grace-period-days` shape leans yaml. Recommended resolution to flag with Mastermind: pick one home and update the spec accordingly; mixing the two layers for different deletion knobs will fragment the operator's mental model.

---

## Trust boundaries

Applying `meta/conventions.md` Part 11 to every value listed in the brief:

### Ownership-of-product check (for "user can delete their own account" / "user can mass-state-flip their own products")

- The existing product-ownership check at `DefaultProductService.changeProductState:641` is `currentUserId.equals(product.getOwner().getId())` — `currentUserId` is from `currentUserService.getCurrentUserIdStrict()` (auth-derived); `product.getOwner().getId()` is from the DB.
- **No `userId` is accepted from the client body anywhere in this flow.** ✓ Server-derived.

### Banned / disabled state

- Source of truth: `User.disabled` column in Postgres.
- The `AuthenticatedUserDTO.disabled` field is populated from that column at every cache-miss via the JPQL constructor projection. But the value is **never consulted** — the auth filter does not branch on it.
- No client-supplied "disabled" or "banned" value flows into any backend decision.
- ✓ Source of truth is server-side; the gap is enforcement, not trust.

### The user's email

- Source of truth: `User.email` (DB, unique constraint). Originally written from `token.getEmail()` on first sync (DefaultFirebaseAuthService.createUserSynchronized:102).
- The email is never read from the request body anywhere I inspected.
- For the eventual banned-email hashing: the input must be `user.getEmail()` from the DB, not from the request. ✓ Server-derived.

### Firebase UID

- Source of truth: `FirebaseToken.getUid()` from `FirebaseAuth.getInstance().verifyIdToken(idToken)` — Firebase Admin SDK verifies signature against Google's public keys.
- Stored on `User.firebaseUid` (DB unique) after first sync.
- The auth filter uses the verified-token uid (not a body field) to look up the user.
- ✓ Server-derived from a verifiable source.

### `auth_time` claim

- **Not read anywhere** (grep confirms zero matches). The verification path exists (`verifyIdToken`) but the result token's `auth_time` claim is not extracted.
- ✓ Server can derive it from a verified token. The deletion request's "reauth-freshness" gate must read this from `token.getClaims().get("auth_time")` (or the typed Firebase Admin SDK accessor), **not** from a request-body field. The implementation does not exist yet — flag as a seam.

### `Report.reportedUserId` — pre-existing audit gap (out of strict deletion scope, but adjacent)

- `DefaultReportService.createReport:46` sets `report.setReportedUser(entityManager.getReference(User.class, reportRequest.getReportedUserId()))` — **client-supplied** with no check that:
  - the id resolves to a real user (lazy reference; only fails on first access);
  - the reporter is authorised to report this target (e.g., is a participant in a conversation with them or a viewer of their public page).
- This is exactly the open `issues.md` entry "Report-submit endpoint trust-boundary verification unknown" (2026-05-16). **The audit confirms there is no server-side authorization check on the reporter→target relationship.** Severity finalizes from medium-pending-audit to medium (or higher depending on abuse impact).
- Not a user-deletion bug, but flagged here because the deletion feature reads / writes the Report table, and any deletion-time invariant assumed about who can report whom would be wrong.

### `ReviewRequestDTO.skipTrust` — also adjacent

- Client-supplied boolean on the review request. When `true`, the server-side `trustReviewService.canReviewProduct` check is skipped (DefaultReviewService.reviewProduct:50). The `verified` column on the resulting `Review` row is `!skipTrust`, so the client also controls whether the review is marked "verified."
- This means any logged-in user can post a `verified` review with `skipTrust=true` against any user/product without the trust check running. Whether that's intentional (per spec — perhaps `skipTrust` is meant only for admin paths) is unclear from the code alone.
- Out of strict user-deletion scope; flag for Mastermind because deletion logic that anonymizes reviews will inherit this.

**No CRITICAL trust-boundary violation found in code paths strictly required for user deletion.** The two adjacent observations above are pre-existing and worth surfacing.

---

## Seams

- **Banned/disabled signal to the frontend.** Backend does not emit any field on `AuthUserDTO` (or any other response shape I read) that the frontend could use to detect that the authenticated user is banned/disabled. The matching counterpart in `oglasino-web`/`oglasino-expo` audits must confirm what field they expect and on which call. Likely shapes: a top-level `disabled: boolean` on `/api/auth/firebase-sync` and/or `/api/auth/firebase/{uid}`, or a dedicated 4xx code from the auth filter when `disabled=true`. Without one of these, the frontend has no way to know to sign out.

- **Reauth freshness on the delete-account request.** The spec requires a "recently reauthenticated" check on the delete-account endpoint. Backend has all the inputs (verified `FirebaseToken` with `auth_time` claim) but reads none of them. The counterpart audits must confirm that the frontend will resubmit a fresh ID token (signed-out-and-signed-in, or via Firebase `reauthenticateWithCredential`) before the delete call, and that the resulting `auth_time` will be strictly newer than some cutoff (e.g., last 5 min). The backend verification code does not exist; spec must define `auth_time - now <= maxAgeSeconds` and which config key holds the cutoff.

- **Provider-agnostic delete.** The deletion request DTO should not carry a `provider` field. Frontend handles both password-reauth and Google-popup-reauth; both produce a fresh Firebase ID token. Backend has no branching on `registeredWithProvider` in the auth or token-verification path today, so the seam is "stay provider-agnostic on the backend." Cross-repo audits must confirm neither frontend sends a provider field in the delete-account request body.

- **Firestore message-doc shape for sender display name updates.** The spec implies the deletion sweep updates chat-message documents to anonymize the sender's display name in-place. **The backend stores no `senderDisplayName` field on messages today** (only `senderId`, `receiverId`, `content`, `seen`, `createdAt`, optional `productId`). If the frontend renders sender names by re-resolving `senderId` → Postgres `User.displayName` at read time, no Firestore mutation is needed — the rename of `User.displayName = "Deleted User"` propagates automatically. If the frontend has denormalized the sender display name onto each Firestore message doc, the spec must add a sweep operation. The web/mobile audits must confirm which model is in use.

- **Firestore notification-doc shape for sender anonymization.** Notification documents have no sender field today (`title`, `description`, `seen`, `createdAt`, `type`, `categoryId`, `data`). If any notification's `title` or `description` contains a hardcoded display name (e.g., "<NAME> reviewed your product"), those strings are frozen text — anonymization would require a content-rewrite of every existing notification authored by the deleted user. Cross-repo audits and any backend code that *writes* notifications (the senders) need to confirm the template shape.

- **Reindex pipeline on listing hide/show.** Today's trigger is `ProductStateUpdateEvent` published from `DefaultProductService.changeProductState`, picked up by `ProductIndexerEventListener.handleProductStateUpdated` (transactional-after-commit, async, retryable, partial-doc update of just `productState`). The deletion sweep needs to mass-flip every product owned by the user to `INACTIVE`. The existing event-driven path will fire one event per product if reused; for large catalogs this may need batching or a bulk-update path. **The current code path also includes an ownership check (`currentUserId == product.owner.id`) that an admin/system-initiated sweep cannot satisfy.** Either a new service method without the check, or a system-context bypass, is needed.

- **Firebase Admin `deleteUser` + `revokeRefreshTokens`.** No `deleteUser` method exists in `FirebaseUserService`. The hard-delete path needs `FirebaseAuth.getInstance().deleteUser(uid)` (+ idempotent USER_NOT_FOUND handling) and probably `revokeRefreshTokens(uid)` at the start of the soft-delete window. The Firebase Admin SDK exposes both; backend just hasn't called them.

- **Re-registration prevention.** Spec implies hashed-email storage so a banned user can't re-register. No table or service exists for this today (no `banned_email_hash` table, no service that intercepts `getOrCreateUser`). Backend will need a new table, a hash-check on first sync, and a way to expire after the retention window.

- **Connection-pool decision constraint.** Any user-deletion code path that mutates an auth-relevant field on `User` (incl. blanking `displayName` to "Deleted User", flipping `disabled = true` as part of the schedule-for-deletion flow) MUST route through `DefaultUserService.saveUser` so the `@CacheEvict` for `redisUserAuth` fires. Otherwise an evicted user can stay authenticated for up to 30 minutes. This is verbatim from the 2026-05-14 decision and explicitly called out in the state.md user-deletion backlog entry.

---

## For Mastermind

### Out-of-scope adjacent observations (per conventions Part 4b)

1. **`DefaultReportService.createReport` self-assignment bug — medium.** Line 47: `report.setReportedProductId(report.getReportedProductId());` assigns the report's own `reportedProductId` (always null at this point — the entity is fresh) to itself, instead of `reportRequest.getReportedProductId()`. Result: PRODUCT-type reports are persisted with `reported_product_id = NULL`, even when the request body carried a real product id. Visible on every product-report submission. I did not fix this because it is out of scope; the deletion-feature audit only surfaced it. Worth a one-line bug-fix brief.

2. **`DefaultTrustReviewService.canReviewProduct` chat-id self-self bug — high impact on the trust feature.** Line 59: `String chatId = getChatId(currentUserFirebaseUid, currentUserFirebaseUid);` passes the current user's UID twice instead of `(currentUserFirebaseUid, targetUserFirebaseUid)`. Because the chat-id is the sorted-and-joined pair of UIDs, this always produces `{currentUid}_{currentUid}` — a chat document that does not exist between two distinct users, so the message-list query returns empty and `eligible` is always `false`. Effect: the trust-review check returns `FORBIDDEN` for every product where `skipTrust=false`, regardless of whether the two users actually had a relevant chat. If the trust feature is meant to be open ("you must have chatted about this product to review"), it's effectively closed. I did not fix this because it is out of scope. Worth confirming whether anyone has noticed the trust path is dead.

3. **`Report.reporter` / `Report.reportedUser` nullability mismatch with the 2026-05-15 entry — low.** The decisions.md entry "Report.reporter and Review.reviewer are non-nullable @ManyToOne FKs that will block user hard-deletion" overstates the situation for the Report side. Both columns are nullable in the DB schema (`V1__init_schema.sql:457-459`) and on the entity (no `optional=false`). Only the Review side actually blocks hard delete. The decisions.md correction is a docs job for Docs/QA when this lands, not an engineering fix.

4. **`ReviewRequestDTO.skipTrust` is client-controlled — medium-to-high pending the spec.** A client can submit `skipTrust=true` and bypass the trust-review check entirely; the resulting `Review.verified` is also `!skipTrust`, so the client also chooses the verified flag. If this is intended only for admin/system flows, the field should not exist on the request DTO — move it server-side or gate by role at the controller. Not in user-deletion scope, but the deletion feature's anonymization will touch reviews and inherit whatever the trust model is.

5. **Report-submit trust-boundary unknown — open issues.md entry can now resolve to medium.** Audit confirms `DefaultReportService.createReport` does no authorization check on the reporter→target relationship. The issues.md 2026-05-16 entry can be moved from `medium-pending-audit` to `medium` (or higher) and re-routed to fix work.

### Open question for Mastermind to resolve before the spec phase

- **Configuration layer for deletion knobs.** Yaml (`app.user-deletion.*`) vs DB-backed Configuration (dot-separated, e.g., `user.deletion.grace.period.days`). My read of the existing conventions: `app.images.sweeper.*` is yaml, `product.removal.batch.size` / `product.removal.days.old` are DB-backed. The deletion feature looks closer to product-removal than to image-pipeline — both are retention sweepers with knobs the operator may want to tune mid-flight without redeploying. Recommended: DB-backed, dot-separated, matching `product.removal.*`. Confirm before the spec freezes the key shape.

- **Sentinel for anonymized display name.** Two options: (a) mutate `User.displayName = "Deleted User"` (or the user's preferred-language translation), accept that the value is locale-frozen at deletion time; (b) introduce a sentinel value (e.g., empty string, or a new `User.anonymized` flag) and resolve to the translation key `user.deleted` at read time on the frontend. Option (a) is simpler and matches the live-join propagation already in place; option (b) is locale-correct. The spec needs to pick one.

### Conventions check — confirmed

I read `meta/conventions.md` end to end, Part 11 (Trust boundaries) included. The trust-boundaries section above explicitly checks ownership, disabled-state, email, firebaseUid, and `auth_time` against the Part 11 rules — every value is either auth-derived, DB-read, or unforgeable. Two adjacent client-supplied-value findings (`Report.reportedUserId`, `ReviewRequestDTO.skipTrust`) are flagged but are out of strict user-deletion scope.
