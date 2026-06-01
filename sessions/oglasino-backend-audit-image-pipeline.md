# Audit — Image Pipeline (Backend, Server Side)

**Repo:** oglasino-backend
**Branch:** dev (unchanged — read-only audit)
**Date:** 2026-05-30
**Type:** READ-ONLY. Code is ground truth; spec (`../oglasino-docs/features/image-pipeline.md`, `web-stable`) treated as a claim to verify.
**Scope:** the backend's half of the image pipeline — the surface mobile (`oglasino-expo`) will call directly.

All file references are relative to `src/main/java/com/memento/tech/oglasino/` unless an absolute path is given. The image surface lives under `images/`.

---

## Divergence rollup (code vs spec)

| # | Section | Severity | Summary |
|---|---|---|---|
| D1 | §3 Scope | low | `REPORT` is accepted as a valid `ImageScope` enum value and passes every named validation; it is only rejected late, in `buildKey()`, re-using `INVALID_SCOPE`. Spec frames `report` as "reserved v2". Functionally rejected (400), but outside the validation fence. |
| D2 | §9 Error contract | informational (by design) | Image endpoints emit `{"error":{code,message,details,retryable}}`; the rest of the backend emits the Part 7 `{"errors":[{field,code,translationKey}]}`. Two distinct envelopes coexist **intentionally** (documented in `GlobalExceptionHandler`). This is correct per spec §8 but is the key fact for mobile's Φ4 `parseServiceError`. |
| D3 | §10 Translations | medium | All 17 keys are seeded ×4 languages (68 rows), but **4 of the 10 active key names differ from the spec**: `converting-heic`→`converting_heic`, `complete`→`complete.label`, `uploading`→`processing.uploading.label`, `uploading.with.size`→`processing.uploading.with.size`. If mobile hard-codes the spec key strings, lookups miss. |

No trust-boundary violations found. Keys are server-constructed; auth is read from `SecurityContextHolder`, never from client input. See §5 and §11.

---

## 1. The three endpoints

### 1.1 `POST /api/secure/images/upload-tokens`

**Controller:** `images/controller/ImageTokensController.issueUploadTokens()` (lines 41–47).

**Request DTO — `images/dto/UploadTokensRequest` (record):**

| Field | Type | Validation annotations |
|---|---|---|
| `scope` | String | `@NotBlank`. Maps to enum {product, profile, chat, report}. |
| `count` | int | `@Min(1)` `@Max(5)` |
| `contentTypes` | List&lt;String&gt; | `@NotEmpty`, `@Size(min=1, max=5)`, elements `@NotBlank` |
| `chatId` | String | (no annotation; cross-field rule: required iff scope=chat, rejected otherwise) |

**Response — `images/dto/UploadTokensResponse`** → `{ "tokens": [UploadTokenEntry] }`, where `UploadTokenEntry` = `{ token, key, uploadUrl, expiresAt }` (e.g. `key="public/products/{uuid}.jpg"`, `uploadUrl="{cdn-base}/public/products/{uuid}.jpg"`, `expiresAt` ISO-8601).

**Validation order as coded — `DefaultImageTokensFacade.issueUploadTokens()` (lines 83–132):**

1. Auth — read `OglasinoAuthentication` from `SecurityContext` (line 84) → `UNAUTHENTICATED` (401) if absent.
2. Scope parse — `ImageScope.from(scope)` (line 86) → `INVALID_SCOPE` (400).
3. Count — `1 ≤ count ≤ maxImagesPerRequest` (line 90) → `INVALID_COUNT` (400).
4. ContentTypes length — `contentTypes.size() == count` (line 91) → `CONTENT_TYPES_MISMATCH` (400).
5. ContentTypes allowlist — each in `allowed-content-types` (line 92) → `CONTENT_TYPE_NOT_ALLOWED` (400).
6. ChatId consistency (line 93) → `CHAT_ID_REQUIRED` / `CHAT_ID_NOT_ALLOWED` (400).
7. Chat membership (if scope=chat) — Firestore `chatService.isUserMemberOfChat()` (lines 95–96) → `NOT_CHAT_MEMBER` (403).
8. **Rate limit** — consume `count` tokens (line 101) → `RATE_LIMITED` (429). (After all validation.)
9. Per content type: UUID, build R2 key, sign HS256 JWT, record Redis ownership (lines 103–123).

Failure statuses: 400 (the six validation codes), 401 `UNAUTHENTICATED`, 403 `NOT_CHAT_MEMBER`, 429 `RATE_LIMITED` (retryable), 500 `INTERNAL` (retryable; JWT-sign or Firestore IO failure). Success: 200.

### 1.2 `POST /api/secure/images/view-tokens`

**Controller:** `ImageTokensController.issueViewToken()` (lines 50–57).

**Request DTO — `images/dto/ViewTokensRequest` (record):** `scope` (String, `@NotBlank`; only `chat` supported in v1), `chatId` (String, `@NotBlank`).

**Response — `images/dto/ViewTokenResponse`** → `{ token, expiresAt, scope, chatId }`.

**Validation order — `DefaultImageTokensFacade.issueViewToken()` (lines 138–171):**

1. Auth (line 139) → `UNAUTHENTICATED` (401).
2. Scope parse (line 141) → `INVALID_SCOPE` (400).
3. Scope must be `CHAT` (lines 142–146) → `INVALID_SCOPE` (400) — v1 only; report deferred to v2.
4. ChatId present (lines 147–149) → `CHAT_ID_REQUIRED` (400).
5. Chat membership — Firestore (line 151) → `NOT_CHAT_MEMBER` (403).
6. **Rate limit** — consume 1 token (line 154) → `RATE_LIMITED` (429). (After validation.)
7. Sign JWT with `keyPrefix = private/chats/{chatId}/` (lines 156–162).

Success: 200. Failures: 400 / 401 / 403 / 429 / 500 as above.

### 1.3 `DELETE /api/secure/images/{*key}`

**Controller:** `images/controller/ImageDeletionController.deleteImage()` (lines 41–46). Spring `{*key}` catch-all preserves `/` in the key; the controller strips the leading `/` before delegating (line 43). No request body.

**Validation chain as coded — `DefaultImageDeletionFacade.deleteOrphan()` (lines 57–113). Order is pinned by tests:**

1. **Auth** (line 58) — `OglasinoAuthentication` from `SecurityContext` → `UNAUTHENTICATED` (401). (`FirebaseAuthFilter` upstream; facade re-checks defensively.)
2. **Format** (line 61) — `ImageKeyValidator.requireValid(key)`. In sequence, each → `INVALID_KEY` (400): non-blank; no leading `/`; no trailing `/`; no `//` (empty segment); no `..` (traversal); no `\` `?` `#`; must start with one of `public/products/`, `public/profiles/`, `public/brand/`, `private/chats/`.
3. **Rate limit** (line 65) — consume 1 token from `IMAGE_TOKEN_ISSUANCE` → `RATE_LIMITED` (429). Note: this is **after format**, before ownership/in-use/age (intentional; see §7).
4. **Ownership** (lines 70–78) — Redis `upload:owner:{key}`; missing or userId mismatch → `NOT_OWNER` (403). Fails closed.
5. **In-use** (lines 83–139) — prefix-conditional entity scan; any reference → `IMAGE_IN_USE` (409). `public/products/` → Product **and** Review; `public/profiles/` → User; `public/brand/` and `private/chats/` → skipped (not in Postgres).
6. **Age** (lines 89–102) — `r2Service.headImage(key)`; missing → `IMAGE_NOT_FOUND` (404); `lastModified` older than 1h → `IMAGE_TOO_OLD` (403).
7. **Delete** (lines 104–109) — `r2Service.delete(key)` then clear Redis ownership. R2-delete failure leaves the Redis record for retry safety; a Redis-clear failure is logged WARN but still returns 204.

Success: **204 No Content**. Failures: 400 `INVALID_KEY`, 401, 403 (`NOT_OWNER` / `IMAGE_TOO_OLD`), 404 `IMAGE_NOT_FOUND`, 409 `IMAGE_IN_USE`, 429, 500 `INTERNAL`.

---

## 2. JWT signing

**Where signed:** `images/token/DefaultImageTokenService` — `signUploadToken()` (lines 72–102) and `signViewToken()` (lines 105–135). JJWT `Jwts.builder()`, HS256, explicit header `typ:"JWT"`. Claim inputs come from the `UploadTokenClaims` / `ViewTokenClaims` records.

**Upload-token claims (signUploadToken):**

| Claim | Source |
|---|---|
| `iss` | `properties.getJwt().getIssuer()` = `oglasino-backend` |
| `iat` | `clock.instant()` |
| `exp` | `iat + uploadTokenTtlMs` |
| `jti` | `ulid.nextULID()` |
| `sub` | `claims.firebaseUid()` |
| `scope` | constant `"upload"` |
| `kind` | `claims.kind()` (product/profile/chat) |
| `key` | full prefixed R2 key |
| `contentType` | MIME from request |
| `maxBytes` | `claims.maxBytes()` |

**View-token claims (signViewToken):** `iss`, `iat`, `exp` (`iat + viewTokenTtlMs`), `jti` (ULID), `sub` (firebaseUid), `scope` = `"view"`, `kind` (`chat`), `keyPrefix` (`private/chats/{chatId}/`), `chatId` (log enrichment, not the auth gate — the Worker enforces `requestedKey.startsWith(keyPrefix)`).

**TTLs (coded vs spec):**
- Upload: `uploadTokenTtlMs` ← `app.images.upload-token-ttl-ms`, default **600000 ms (10 min)** — set in dev/stage/prod yaml. **Matches spec.**
- View: `viewTokenTtlMs` ← `app.images.view-token-ttl-ms`, default **14400000 ms (4 h)** — set in dev/stage/prod yaml. **Matches spec.**

**Secret + startup behavior:** `app.images.jwt.signing-secret` (env `JWT_SIGNING_SECRET`), validated in `@PostConstruct initSigningKey()` (lines 56–69). Blank → `ImageTokenIssuanceException("...signing-secret is missing...")`. `Keys.hmacShaKeyFor()` throws `WeakKeyException` if < 32 bytes, re-thrown as `ImageTokenIssuanceException("JWT_SIGNING_SECRET must be at least 32 bytes for HS256")`. **The app fails to start** if the secret is missing/blank/too short. **Matches spec.**

> Note: `jti` is a ULID (`ulid.nextULID()`), not the UUIDv4 the spec's example shows. Cosmetic — both are opaque unique strings; not flagged as a functional divergence.

---

## 3. Scope / count / content-type validation

**Scope:** `images/facade/ImageScope` enum = {`PRODUCT`, `PROFILE`, `CHAT`, `REPORT`} (lines 11–31). `ImageScope.from(String)` (21–30): null → `invalidScope(null)`; trims + uppercases (ROOT) + `valueOf()`; `IllegalArgumentException` → `INVALID_SCOPE`. Wire form is lowercase via `wire()`. Called at `DefaultImageTokensFacade` line 86 (upload) and 141 (view).

**Count cap:** `@Max(5)` on `UploadTokensRequest.count` (line 23) is the binding constraint; `enforceCount()` (facade 183–188) re-checks `1 ≤ count ≤ properties.getMaxImagesPerRequest()` → `INVALID_COUNT`. The DTO comment notes the facade check is a defensive mirror of `@Max(5)`.

**ContentTypes length == count:** `enforceContentTypesLength()` (facade 190–195) → `CONTENT_TYPES_MISMATCH`.

**Content-type allowlist (config, not constant):** `app.images.allowed-content-types` → `ImageProperties.allowedContentTypes` (`List<String>`, `@ConfigurationProperties(prefix="app.images")`). Default in all profiles: `image/jpeg, image/png, image/webp, image/heic, image/heif`. `enforceContentTypeAllowed()` (facade 197–202) checks membership per type → `CONTENT_TYPE_NOT_ALLOWED`.

**ChatId required-iff-chat:** `enforceChatIdConsistency()` (facade 204–214): scope=CHAT + blank chatId → `CHAT_ID_REQUIRED`; scope≠CHAT + non-blank chatId → `CHAT_ID_NOT_ALLOWED`. (`CHAT_SCOPES = Set.of(ImageScope.CHAT)`, line 59.)

**D1 — DIVERGENCE (low):** `ImageScope.from()` accepts `REPORT` as a valid enum, so a `scope:"report"` request passes scope-parse, count, contentTypes, and chatId-consistency, and is only rejected at `buildKey()` (`DefaultImageTokensFacade` 230–232), which throws `invalidScope(scope.wire())` — re-using `INVALID_SCOPE` (400). Spec frames `report` as "reserved v2". Net effect is still a 400 rejection, but the rejection happens outside the validation fence and re-uses the unrecognized-scope code. View-tokens reject non-`CHAT` scopes earlier (step 3, §1.2), so `report` there fails at the explicit scope check.

---

## 4. Chat membership check (view tokens)

**Where:** `DefaultImageTokensFacade.issueViewToken()` line 151 calls `verifyChatMembership(auth.getFirebaseUid(), request.chatId())` (helper at 216–223), **before** signing (line 162). The same check runs for upload-tokens when scope=chat (§1.1 step 7).

**What it reads:** `admin/service/impl/DefaultFirebaseChatService.isUserMemberOfChat()` (lines 133–160) reads Firestore collection `chats`, document `{chatId}`, field `users` (expected `List<String>`), and tests `users.contains(firebaseUid)` (line 148).

**Produces `NOT_CHAT_MEMBER`:** returns `false` if the chat doc doesn't exist, `users` is missing/not-a-list, or the uid isn't in it → facade throws `ImageRequestException.notChatMember(chatId)` → `ImageErrorCode.NOT_CHAT_MEMBER`, **403**, `retryable=false`, `details={chatId}`.

**IO failures fail safe:** Firestore `ExecutionException` / `InterruptedException` surface as `RuntimeException` (153, 158) → caught by `ImageExceptionHandler` catch-all → **500 `INTERNAL`** (retryable), deliberately not masked as 403.

---

## 5. R2 key construction

**Prefixes — `images/path/ImagePaths` (single source of truth):** `PUBLIC_PRODUCTS = "public/products/"` (15), `PUBLIC_PROFILES = "public/profiles/"` (18), `PUBLIC_BRAND = "public/brand/"` (21), `PRIVATE_CHATS = "private/chats/"` (27). Helpers: `productKey(uuid,ext)`, `profileKey(uuid,ext)`, `chatKey(chatId,uuid,ext)`, `chatPrefix(chatId)` (34–51). All prefixes end in `/`.

**Key generation at token time:** in `DefaultImageTokensFacade.issueUploadTokens()` per content type: `UUID.randomUUID().toString()` (line 108); extension via `ContentTypeExtension.extensionFor(contentType)` (106–107). Map (`ContentTypeExtension` 17–23, case-insensitive lookup, returns `Optional`, unsupported → `contentTypeNotAllowed()`): jpeg→`jpg`, png→`png`, webp→`webp`, heic→`heic`, heif→`heif`. `buildKey(scope, chatId, uuid, ext)` (225–234) dispatches to the `ImagePaths` helper for the scope.

**Server-constructed — evidence:** `UploadTokensRequest` has **no `key` and no `uuid` field** (only scope, count, contentTypes, chatId). The UUID is generated server-side; the full key is embedded in `UploadTokenClaims` → signed into the JWT and returned in `UploadTokenEntry.key`. The only client string that reaches a key path is `chatId`, and it is gated first: non-blank check, then Firestore membership verification, then embedded as `private/chats/{verified-chatId}/...`. The signed `keyPrefix` claim bounds Worker access to that chat. **No trust-boundary violation.**

---

## 6. Ownership record + DELETE authorization

**Write (at token issuance):** `DefaultImageTokensFacade` line 122 → `ownershipService.recordOwnership(auth.getUserId(), key)`. `images/ownership/RedisUploadOwnershipService`: key `upload:owner:{fullKey}` (prefix const line 26), value `String.valueOf(userId)`, **TTL `Duration.ofHours(1)`** (line 29). Best-effort: Redis write failures are logged WARN, not thrown (interface contract, `UploadOwnershipService` 19–22).

**DELETE ownership check:** `DefaultImageDeletionFacade` 70–78: `getOwner(fullKey)` → `Optional<Long>`; compared to `auth.getUserId()` via `.equals()` (autoboxing-safe); empty or mismatch → `NOT_OWNER` (403, non-retryable). **Fails closed** — a Redis read failure returns empty → DELETE rejected (`RedisUploadOwnershipService` 67–75).

**In-use scan (entities):** `DefaultImageDeletionFacade` 83–139, prefix-conditional:
- `public/products/` → `ProductRepository.existsByImageKey()` (`@Query SELECT COUNT(p)>0 FROM Product p JOIN p.imageKeys k WHERE k=:key`, lines 101–102) **and** `ReviewRepository.existsByImageKey()` (57–58).
- `public/profiles/` → `UserRepository.existsByProfileImageKey()` (28, derived query).
- `public/brand/`, `private/chats/` → skipped (not in Postgres).
Any hit → `IMAGE_IN_USE` (409). Rationale (80–81): protects against a re-issued DELETE after entity-create succeeded but the client missed the response.

---

## 7. Rate limiting

**Category:** `security/ratelimit/RateLimitCategory.IMAGE_TOKEN_ISSUANCE` (line 32) = `Bandwidth.simple(60, Duration.ofMinutes(1))` → **60 tokens / minute / user**, Bucket4j-Redis (`rl:user:{userId}:image_token_issuance`). **Matches spec.** Javadoc notes it's enforced in the facade (not a URL filter) because cost depends on the parsed body.

**Implementation:** `Bucket4jImageTokenRateLimiter.consume()` (36–56) → `tryConsumeAndReturnRemaining(tokens)`; failure → `ImageRequestException.rateLimited(retryAfterSec)` (429).

**Cost per operation:** upload-tokens consumes `count` (facade 101); view-tokens consumes 1 (facade 154); DELETE consumes 1 (deletion facade 65).

**Before or after validation:** 
- Upload & view: **after all validation** (facade 101 / 154, following every `enforce*` and membership check). Failed validation throws before the consume call → does **not** burn budget. Confirmed by tests `failedValidationDoesNotConsumeBudget`, `failedMembershipDoesNotConsumeBudget`, `rateLimitIsCheckedAfterMembershipAndBeforeSigning`.
- DELETE: **after format validation, before ownership/in-use/age** (deletion facade 65, comment line 63 — reuses the same per-user pool). A malformed key does **not** burn budget (`invalidPrefixRejectedAsInvalidKey`, `pathTraversalRejectedAsInvalidKey`); but a well-formed key the caller doesn't own **does** consume 1 token before the ownership check runs.

**Conformance:** matches the spec's "failed requests do NOT burn budget" for the validation codes; the DELETE ordering (rate-limit before ownership) is a documented, test-pinned design choice.

---

## 8. Entity field semantics

All three store **full prefixed keys** (no bare UUIDs); all columns `varchar(255)`.

| Field | JPA mapping | DDL (`db/migration/V1__init_schema.sql`) |
|---|---|---|
| `Product.imageKeys` | `@ElementCollection` + `@CollectionTable(name="product_images")`, col `image_keys` varchar(255) (entity 85–88) | `product_images.image_keys varchar(255)` (410–413) |
| `User.profileImageKey` | plain `@Column` varchar(255) (entity 73) | `users.profile_image_key varchar(255)` (612) |
| `Review.imageKeys` | `@ElementCollection(LAZY)` + `@CollectionTable(name="review_images")`, col `image_key` varchar(255) (entity 56–59) | `review_images.image_key varchar(255)` (504–507). Reviews share the products prefix in v1. |

Entity Javadoc on all three states the value is the full path including prefix, returned verbatim by upload-tokens and stored as-is.

**Persistence paths:**
- Create product: `DefaultProductService.createProduct(NewProductRequestDTO)` (line 94) → `product.setImageKeys(...)` filtering blanks into a Set (128–132).
- Update product: `DefaultProductService.updateProduct(UpdateProductRequestDTO)` (line 220) → same set logic (250–254); snapshots old keys (245–248) for orphan cleanup.
- Update profile: `DefaultUserFacade.updateCurrentUserData(UpdateUserDTO)` (line 109) → `user.setProfileImageKey(...)` (133); deletes the old image when the key differs (127–130).
- Review (immutable): `DefaultReviewService.reviewProduct(ReviewRequestDTO)` (line 39) → `review.setImageKeys(...)` (65).
- **Chat images persist to Firestore, not Postgres.** No backend method writes image keys onto chat messages; `ChatMessageDTO` carries no image field. Backend's only chat-image roles are view-token issuance, membership verification, and the `ChatImagesRemovalJob` sweep of `private/chats/`.

---

## 9. Error contract shape

**Image envelope (`images/exception/ImageErrorResponse` 24–28):**

```json
{ "error": { "code": "...", "message": "...", "details": {...}, "retryable": false } }
```

`error.details` is `Map<String,Object>`, omitted when null (`@JsonInclude(NON_NULL)`). `retryable` derives from `ImageErrorCode.retryable()`. **Matches spec §8 exactly.**

**D2 — differs from the project-wide Part 7 contract (by design).** Non-image endpoints (`GlobalExceptionHandler`) emit `{"errors":[{"field","code","translationKey"}]}` (array root). `GlobalExceptionHandler` (27–28) explicitly documents that image endpoints use their own `ImageExceptionHandler` per the pipeline §8 contract. **Two independent envelopes coexist intentionally.**

| Aspect | Image (`ImageExceptionHandler`) | Rest of backend (`GlobalExceptionHandler`) |
|---|---|---|
| Root | `error` (object) | `errors` (array) |
| Code | `error.code` | `errors[].code` |
| Message | `error.message` | — |
| Details | `error.details` (optional map) | — |
| Retryable | `error.retryable` | — |
| Translation key | — (none) | `errors[].translationKey` |
| Field | — | `errors[].field` (null = object-level) |

**Mobile Φ4 implication (flag, clear):** the image surface does **not** match the Part 7 shape `parseServiceError` expects elsewhere. Image errors carry **no `field` and no `translationKey`** — mobile must branch on the envelope (`error` object vs `errors` array) and key its i18n off `error.code` (mapping codes → the `image.*` reserved keys in §10), not off a server-sent `translationKey`.

**`ImageErrorCode` → HTTP (`images/exception/ImageErrorCode` 12–50; mapping in `ImageExceptionHandler` 36–69):**

| Code | HTTP | Retryable |
|---|---|---|
| INVALID_SCOPE | 400 | false |
| INVALID_COUNT | 400 | false |
| CONTENT_TYPES_MISMATCH | 400 | false |
| CONTENT_TYPE_NOT_ALLOWED | 400 | false |
| CHAT_ID_REQUIRED | 400 | false |
| CHAT_ID_NOT_ALLOWED | 400 | false |
| INVALID_KEY | 400 | false |
| UNAUTHENTICATED | 401 | false |
| NOT_CHAT_MEMBER | 403 | false |
| NOT_OWNER | 403 | false |
| IMAGE_TOO_OLD | 403 | false |
| IMAGE_NOT_FOUND | 404 | false |
| IMAGE_IN_USE | 409 | false |
| RATE_LIMITED | 429 | true |
| INTERNAL | 500 | true |

`ImageTokenIssuanceException` → 500 INTERNAL ("image token signing failed"); uncaught `Exception` → 500 INTERNAL ("internal error").

---

## 10. Translation seeds — verdict

**Verdict: SEEDED — all 17 keys, 4 languages, 68 rows — but 4 of the 10 active key names DIVERGE from the spec (D3).**

All 17 keys live in `src/main/resources/data/translations/0001-data-web-translations-{EN,RS,RU,CNR}.sql` (none in 0002/0003). 17 rows per language × 4 = **68 rows**. Language coverage EN/RS/RU/CNR = 17/17 each.

**Active keys (10) — spec vs code:**

| Spec key | Seeded key | Namespace | Match? |
|---|---|---|---|
| image.upload.failed | image.upload.failed | ERRORS | ✓ |
| image.processing.validating | image.processing.validating | INPUT | ✓ |
| image.processing.converting-heic | **image.processing.converting_heic** | INPUT | ✗ hyphen→underscore |
| image.processing.resizing | image.processing.resizing | INPUT | ✓ |
| image.processing.encoding | image.processing.encoding | INPUT | ✓ |
| image.processing.cancelled | image.processing.cancelled | INPUT | ✓ |
| image.processing.error | image.processing.error | INPUT | ✓ |
| image.processing.complete | **image.processing.complete.label** | INPUT | ✗ `.label` suffix |
| image.uploading | **image.processing.uploading.label** | INPUT | ✗ rescoped + `.label` |
| image.uploading.with.size | **image.processing.uploading.with.size** | INPUT | ✗ rescoped under `.processing` |

**Phase-8 reserved keys (7) — all match exactly, namespace `ERRORS`:** image.invalid, image.forbidden, image.bad.format, image.rate.limited, image.server.error, image.token.expired, image.session.expired.

Pre-existing keys confirmed distinct and **not** re-registered (image.max, image.too.big, image.duplicate, image.broke, image.not.good, images.holder.label, images.import, button.close.label, button.cancel.label).

**Mobile dependency:** the backend seed exists, so mobile does **not** need to carry these strings as inline fallbacks — **provided it uses the seeded key names.** The 4 divergent active keys (notably the `.label` suffix on `complete`/`uploading`, and `converting_heic`) likely reflect Part 6 Rule 2 (parent/child collision) renames. If mobile (or web) hard-codes the spec's literal key strings for those four, the lookups will miss. The seeded names are authoritative.

---

## 11. Seams toward mobile

**Auth — client-agnostic (confirmed).** All three endpoints rely solely on Firebase JWT via `FirebaseAuthFilter` (67–130), which populates `OglasinoAuthentication` (userId, firebaseUid, authorities) in `SecurityContextHolder`. Facades read it via `SecurityContextHolder.getContext().getAuthentication()` (tokens facade 176, deletion facade 118). No user-agent / device / client-type gate. **Mobile must satisfy:** send a valid Firebase ID token in the same `Authorization` header web uses — nothing else.

**Content-type / size.** Backend token issuance enforces the **content-type allowlist** (`image/jpeg|png|webp|heic|heif`) and stamps `maxBytes = max-upload-bytes` (10 MB) into the JWT claim, but it does **not** weigh bytes itself — the **Worker** enforces Content-Length ≤ maxBytes and Content-Type == claim on PUT. **Mobile must satisfy:** process/compress client-side so each uploaded file is in the allowlist and ≤ 10 MB, or the Worker rejects the PUT (415 / 413). The content-type sent at token-request time must equal the content-type sent on PUT.

**Key-association shape.** Backend returns **full prefixed keys** from token issuance and stores them verbatim. **Mobile must satisfy:**
- Create/update product → `imageKeys: Set<String>` of full keys (`NewProductRequestDTO`/`UpdateProductRequestDTO`, field name `imageKeys`).
- Update profile → `profileImageKey: String` (singular) (`UpdateUserDTO`).
- Chat → no backend DTO (Firestore-side), but referenced keys must sit under `private/chats/{chatId}/`, since the view-token `keyPrefix` claim binds Worker access to that prefix.
Send keys back exactly as issued — never bare UUIDs, never reconstructed paths.

---

*End of audit. Read-only; no code changed. Divergences D1–D3 summarized at top.*
