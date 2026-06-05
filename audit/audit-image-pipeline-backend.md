# Audit — Image-Pipeline Backend Reference (READ-ONLY)

**Repo:** oglasino-backend
**Branch:** `dev` (not switched)
**Type:** read-only reference audit — no code changed, nothing staged, no commits.
**Date:** 2026-05-31
**Method:** every "backend does X" claim below is read from the code on disk today and cited by `file:line`. Where the code disagrees with the spec claims quoted in the brief, the disagreement is flagged explicitly. "Not found" is written where a thing genuinely isn't there.

---

## 1. The three `/api/secure/images` endpoints

Both token endpoints live on `ImageTokensController` (`@RequestMapping("/api/secure/images")`, `ImageTokensController.java:31-33`). The DELETE lives on a separate `ImageDeletionController` with the same base mapping (`ImageDeletionController.java:31-33`).

### 1a. `POST /upload-tokens`

- **Controller method:** `ImageTokensController.issueUploadTokens` — `ImageTokensController.java:41-48` (`@PostMapping("/upload-tokens")` at :41).
- **Request DTO:** `UploadTokensRequest` (record) — `UploadTokensRequest.java:21-25`:
  - `scope` — `String`, `@NotBlank` (:22)
  - `count` — `int`, `@Min(1) @Max(5)` (:23)
  - `contentTypes` — `List<@NotBlank String>`, `@NotEmpty @Size(min = 1, max = 5)` (:24)
  - `chatId` — `String`, no annotation (nullable; "required iff `scope=chat`" enforced in the facade, not at the annotation layer) (:25)
- **Response DTO:** `UploadTokensResponse` (record) — `UploadTokensResponse.java:11`: `{ tokens: List<UploadTokenEntry> }`. Each `UploadTokenEntry` (`UploadTokenEntry.java:16`): `{ token: String, key: String, uploadUrl: String, expiresAt: Instant }`. HTTP 200 on success (`ResponseEntity.ok`, controller :47; the class javadoc :22-23 explains why not 201).
- **Validation order:**
  1. Bean Validation (`@Valid` + `BindingResult`) fires first. If `bindingResult.hasErrors()` (:44) the controller maps the **first** field error to a stable code via `translateUploadValidation` (`ImageTokensController.java:66-87`), priority order: `scope` (:68) → `count` (:71) → `contentTypes` (:77) → fallback `INVALID_SCOPE` (:86).
  2. Otherwise `facade.issueUploadTokens` (`DefaultImageTokensFacade.java:82-133`) runs, fail-fast:
     1. `currentAuth()` — `OglasinoAuthentication` pulled from `SecurityContextHolder`; else `UNAUTHENTICATED` (:84, helper :175-181)
     2. `ImageScope.from(request.scope())` — unknown → `INVALID_SCOPE` (:86)
     3. `enforceCount(count)` — `< 1` or `> max` → `INVALID_COUNT` (:90, helper :183-188)
     4. `enforceContentTypesLength` — `size != count` → `CONTENT_TYPES_MISMATCH` (:91, helper :190-195)
     5. `enforceContentTypeAllowed` per entry — not in allowlist → `CONTENT_TYPE_NOT_ALLOWED` (:92, helper :197-202)
     6. `enforceChatIdConsistency` — `CHAT_ID_REQUIRED` / `CHAT_ID_NOT_ALLOWED` (:93, helper :204-214)
     7. if `scope == CHAT`: `verifyChatMembership` (Firestore) — not a member → `NOT_CHAT_MEMBER`; Firestore IO error propagates → 500 (:95-97, helper :216-223)
     8. `rateLimiter.consume(userId, count)` — exhausted → `RATE_LIMITED` (:101)
     9. sign loop (:103-123): content-type→extension, random UUID, `buildKey`, sign upload JWT, record Redis ownership (1h TTL, best-effort)

### 1b. `POST /view-tokens`

- **Controller method:** `ImageTokensController.issueViewToken` — `ImageTokensController.java:50-57` (`@PostMapping("/view-tokens")` at :50).
- **Request DTO:** `ViewTokensRequest` (record) — `ViewTokensRequest.java:11`: `{ scope: String @NotBlank, chatId: String @NotBlank }`.
- **Response DTO:** `ViewTokenResponse` (record) — `ViewTokenResponse.java:17`: `{ token: String, expiresAt: Instant, scope: String, chatId: String }`. `scope`/`chatId` echo the request; HTTP 200 (`ResponseEntity.ok`, controller :56).
- **Validation order:**
  1. Bean Validation; on error `translateViewValidation` (`ImageTokensController.java:89-97`): `scope` (:90) → `chatId` (:93) → fallback `INVALID_SCOPE` (:96).
  2. `facade.issueViewToken` (`DefaultImageTokensFacade.java:137-171`):
     1. `currentAuth()` (:139)
     2. `ImageScope.from(scope)`; if `scope != CHAT` → `INVALID_SCOPE` (:141-146) — **chat is the only accepted view scope in v1**
     3. `StringUtils.isBlank(chatId)` → `CHAT_ID_REQUIRED` (:147-149)
     4. `verifyChatMembership` (:151)
     5. `rateLimiter.consume(userId, 1)` (:154)
     6. sign view JWT bound to `ImagePaths.chatPrefix(chatId)` (:156-162)

### 1c. `DELETE /{*key}`

- **Controller method:** `ImageDeletionController.deleteImage` — `ImageDeletionController.java:41-46` (`@DeleteMapping("/{*key}")` at :41). Catch-all `{*key}` captures the full slash-bearing key; the leading `/` is stripped before the facade call (:43).
- **Request shape:** no body / no DTO. Single `@PathVariable String key`.
- **Response:** `ResponseEntity<Void>` — **204 No Content** on success (:45).
- **Validation order** — `facade.deleteOrphan` (`DefaultImageDeletionFacade.java:56-113`), fail-fast:
  1. `currentAuth()` — else `UNAUTHENTICATED` (:58, helper :117-123)
  2. `ImageKeyValidator.requireValid(fullKey)` — format/path-traversal/prefix gate → `INVALID_KEY` (:61). Rule order in `ImageKeyValidator.requireValid` (`ImageKeyValidator.java:45-70`): non-blank → no leading `/` → no trailing `/` → no `//` → no `..` → no `\ ? #` → must start with one of the four allowed prefixes (`public/products/`, `public/profiles/`, `public/brand/`, `private/chats/`, list at :30-35).
  3. `rateLimiter.consume(userId, 1)` → `RATE_LIMITED` (:65)
  4. Ownership: Redis read, fails **closed** → `NOT_OWNER` if empty or `!recorded.equals(userId)` (:70-78)
  5. `isInUse(fullKey)` — prefix-conditional DB existence check → `IMAGE_IN_USE` (:83-86, helper :130-139)
  6. `r2Service.headImage` — absent → `IMAGE_NOT_FOUND`; age `> MAX_AGE` (1h, :32) → `IMAGE_TOO_OLD` (:89-102)
  7. `r2Service.delete` then `clearOwnership` → 204 (:108-109)

### 1d. The `scope` enum — actual members

**`ImageScope` (`ImageScope.java:11-15`) contains exactly: `PRODUCT, PROFILE, CHAT, REPORT`.** Wire values are lowercase (`wire()`, :17-19); `from()` upper-cases and rejects unknown values with `INVALID_SCOPE` (:21-30). The enum's own javadoc (:8-9) lists only `product, profile, chat, report`.

**⚠ Discrepancy vs the brief's spec claim.** The brief says the spec claims `product | profile | chat | review | report`, with **`review` live** and `report` reserved. Reality:
- **There is NO `REVIEW` member.** A request with `scope="review"` fails `ImageScope.from` → `INVALID_SCOPE`. The spec line (`features/image-pipeline.md:125`) listing `review` as a wire value is contradicted by the enum. The same spec line's prose ("reviews upload via the `product` prefix family") is the accurate part — reviews must send **`scope="product"`**, not `scope="review"`. The wire-table entry is misleading; the enum is the truth.
- **`REPORT` exists in the enum but is not usable.** `buildKey` throws `INVALID_SCOPE` for `REPORT` ("v2 … no scope yet", `DefaultImageTokensFacade.java:230-232`), and `view-tokens` rejects everything except `CHAT`. So `report` is reserved exactly as the brief's spec claim says, but via a runtime throw, not by absence from the enum.
- **Effective accepted scopes today:** upload-tokens → `product`, `profile`, `chat`. view-tokens → `chat` only.

Trust boundary (conventions Part 11): all three endpoints derive caller identity (`userId`, `firebaseUid`) from `OglasinoAuthentication` in the `SecurityContext`, never from the request body (`currentAuth()` in both facades). Chat membership is verified server-side via Firestore; DELETE ownership is read from Redis (server store); in-use from Postgres; age from R2 HEAD. No client-supplied "previous/old" value is trusted. **Clean.**

---

## 2. Translation seed reality

**Seed files** (one per locale, all under `src/main/resources/data/translations/`):
`0001-data-web-translations-EN.sql` (baseSite col `3`), `…-RS.sql` (`1`), `…-RU.sql` (`4`), `…-CNR.sql` (`2`).

> Note: all four files show as modified (`M`) in the working tree at session start. This audit reports the strings as they exist on disk now.

### 2a. What is actually seeded

**29 distinct keys whose name starts with `image.`** are seeded, each present in **all four locales** ⇒ **116 rows** (not the 17 keys / 68 rows the brief's spec claim states). They split across **two** namespaces: 14 in `INPUT`, 15 in `ERRORS`.

**`INPUT` (14 keys)** — EN ids 2988, 3004–3016; RS 5088, 5104–5116; RU 7188, 7204–7216; CNR 888, 904–916:

| Key | EN line | EN value |
| --- | --- | --- |
| `image.max` | EN:513 | `Maximum {value} images.` |
| `image.processing.validating` | EN:529 | `Checking…` |
| `image.processing.converting_heic` | EN:530 | `Converting HEIC…` |
| `image.processing.resizing` | EN:531 | `Resizing…` |
| `image.processing.encoding` | EN:532 | `Compressing…` |
| `image.processing.cancelled` | EN:533 | `Cancelled` |
| `image.processing.error` | EN:534 | `Failed` |
| `image.processing.complete.label` | EN:535 | `Done · {originalSize} → {processedSize}` |
| `image.processing.uploading.label` | EN:536 | `Uploading…` |
| `image.processing.idle` | EN:537 | `Waiting…` |
| `image.processing.default` | EN:538 | `Processing…` |
| `image.processing.uploading.with.size` | EN:539 | `Uploading {size}…` |
| `image.processing.complete.with.sizes` | EN:540 | `Done · {originalSize} → {processedSize}` |
| `image.broke` | EN:542 | `Image is corrupted, it will be removed` |

**`ERRORS` (15 keys)** — EN ids 3073, 3075–3088; RS 5173, 5175–5188; RU 7273, 7275–7288; CNR 973, 975–988:

| Key | EN line |
| --- | --- |
| `image.not.good` | EN:585 |
| `image.upload.failed` | EN:587 |
| `image.invalid` | EN:588 |
| `image.forbidden` | EN:589 |
| `image.bad.format` | EN:590 |
| `image.rate.limited` | EN:591 |
| `image.server.error` | EN:592 |
| `image.token.expired` | EN:593 |
| `image.session.expired` | EN:594 |
| `image.not.owner` | EN:595 |
| `image.in.use` | EN:596 |
| `image.too.old` | EN:597 |
| `image.invalid.key` | EN:598 |
| `image.too.big` | EN:599 |
| `image.duplicate` | EN:600 |

Corresponding lines in the other locales: RS `image.max`:511, `INPUT` block 527–540, `ERRORS` 583–598 · RU `image.max`:510, `INPUT` 526–537, `ERRORS` 582–597 · CNR `image.max`:510, `INPUT` 526–537, `ERRORS` 582–597.

### 2b. The three keys the brief flagged by name — all confirmed

- **`image.processing.converting_heic` (underscore)** — confirmed in all four: EN:530, RS:528, RU:527, CNR:527. (No `converting-heic` hyphen variant is seeded — matching `issues.md` 2026-05-30, which records the mobile-side hyphen-vs-underscore bug now fixed via override.)
- **`image.processing.complete.label`** — EN:535, RS:533, RU:532, CNR:532.
- **`image.processing.uploading.label`** — EN:536, RS:534, RU:533, CNR:533.

### 2c. Spec-vs-seed diff (brief item 2 asks to flag both directions)

- **Spec-listed-active keys NOT seeded: NONE.** All 10 "active" keys (`features/image-pipeline.md:513-522`) and all 7 "Phase 8 reserved" keys (`:524-534`) are seeded ×4. The spec's "already registered" image keys (`:536` — `image.max`, `image.too.big`, `image.duplicate`, `image.broke`, `image.not.good`) are also all present.
- **Seeded keys the spec does NOT mention (7):** `image.processing.idle`, `image.processing.default`, `image.processing.complete.with.sizes` (INPUT), and `image.not.owner`, `image.in.use`, `image.too.old`, `image.invalid.key` (ERRORS). The four ERRORS ones are the DELETE-endpoint codes (`NOT_OWNER`, `IMAGE_IN_USE`, `IMAGE_TOO_OLD`, `INVALID_KEY`) the deletion facade actually throws — live, not dead. The three INPUT ones are real progress-stage strings the mobile pipeline may key against.
- **Net count:** spec says "17 keys / 68 rows confirmed"; the seed files actually carry **29 `image.*` keys / 116 rows**. The spec table enumerates a subset; it is an undercount, not a missing-seed problem.

> Adjacent observation (low): `image.broke` on EN:542 carries a trailing `--to remove let it for now` comment; the key is seeded in all four locales regardless. Reported, not touched.

---

## 3. `UpdateProductRequestDTO` current field set

Class `UpdateProductRequestDTO` — `src/main/java/com/memento/tech/oglasino/dto/UpdateProductRequestDTO.java:20-96`. Every field:

| Field | Type | Line | Annotations |
| --- | --- | --- | --- |
| `id` | `Long` | :22-23 | `@NotNull("PRODUCT_ID_REQUIRED")` |
| `name` | `String` | :25-28 | `@NotBlank("NAME_REQUIRED")`, `@Size(max=80,"NAME_TOO_LONG")`, `@ValidProductName` |
| `description` | `String` | :30-33 | `@NotBlank("DESCRIPTION_REQUIRED")`, `@Size(max=2000,"DESCRIPTION_TOO_LONG")`, `@ValidDescription` |
| `price` | `BigDecimal` | :35 | none (optional) |
| `currency` | `CurrencyDTO` | :36 | `@Valid` (optional) |
| `filters` | `List<SelectedFilterDTO>` | :38 | none |
| `imageKeys` | `Set<String>` | :39 | none |

**`oldName`, `oldDescription`, `productState`, `moderationState`: NONE are present.** Confirmed absent — the DTO has only the seven fields above. This matches the conventions Part 11 trust-boundary precedent (client-supplied "old" values removed; server compares against its own stored entity) and the brief's note about chat A's `toUpdateWirePayload` allow-list. The backend DTO and the client allow-list now agree on shape.

---

## 4. `ProductImagesRemovalJob`

File: `src/main/java/com/memento/tech/oglasino/images/job/ProductImagesRemovalJob.java`.

- **Schedule:** `@Scheduled(cron = "${app.images.sweeper.cron}")` on `sweep()` (:57). The cron resolves to `"0 0 3 * * *"` in all three profiles (`application-dev.yaml:225`, `application-prod.yaml:237`, `application-stage.yaml:248`) and as the `ImageProperties.Sweeper.cron` Java default (`ImageProperties.java:135`) ⇒ **daily at 03:00**.
  - **⚠ Timezone caveat — NOT 03:00 UTC.** The `@Scheduled` has no `zone` attribute, so it fires in the JVM default timezone, and the container sets `ENV TZ=Europe/Belgrade` (`Dockerfile:13`). The job therefore runs at **03:00 Europe/Belgrade** = 02:00 UTC (CET / winter) or 01:00 UTC (CEST / summer), **not 03:00 UTC**. The "03:00 UTC" wording in the code javadoc (`ImageProperties.java:129`, `ProductImagesRemovalJob.java:26`) and in the spec is inaccurate for this deployment. (The grace window below is a `Duration` and is unaffected by timezone.)
- **Grace period:** `Instant.now().minus(Duration.ofHours(cfg.getGracePeriodHours()))` (:66). `gracePeriodHours` default is **24** (Java default `ImageProperties.java:134`; yaml `grace-period-hours: ${IMAGE_SWEEPER_GRACE_HOURS:24}` in dev:224 / prod:236 / stage:247) ⇒ **24h**, confirmed. Only objects older than the cutoff are candidates.
- **Prefixes swept:** `ImagePaths.PUBLIC_PRODUCTS` (`"public/products/"`) and `ImagePaths.PUBLIC_PROFILES` (`"public/profiles/"`) (:69-70; constants `ImagePaths.java:15,18`). Explicitly **out of scope**: `private/chats/` (handled by `ChatImagesRemovalJob` on a 30-day cadence) and `public/brand/` (never auto-deleted) — javadoc :29-36. Reference check (:142-150): products prefix → `productRepository.existsByImageKey OR reviewRepository.existsByImageKey`; profiles prefix → `userRepository.existsByProfileImageKey`.
- **Enable flag:** gated by `cfg.isEnabled()` at the top of `sweep()` — disabled ⇒ logs and returns (:59-63). Bound to **`IMAGE_SWEEPER_ENABLED`** per profile, with **different defaults per environment**:
  - **dev:** `enabled: ${IMAGE_SWEEPER_ENABLED:false}` → default **FALSE** (`application-dev.yaml:223`)
  - **stage:** `enabled: ${IMAGE_SWEEPER_ENABLED:true}` → default **TRUE** (`application-stage.yaml:246`)
  - **prod:** `enabled: ${IMAGE_SWEEPER_ENABLED:true}` → default **TRUE** (`application-prod.yaml:235`)
  - (Java-side default in `ImageProperties.Sweeper.enabled` is `true`, `ImageProperties.java:133`, but the yaml binding wins in every running profile.)
- **Does it actually run in prod?** **Yes** — prod defaults `enabled=true`, sweeping `public/products/` + `public/profiles/` daily at 03:00 Europe/Belgrade with a 24h grace window, unless `IMAGE_SWEEPER_ENABLED=false` is set in the prod env. The mobile-side backstop assumption (24h grace, runs in prod) holds; only the "03:00 UTC" timing detail is off (it's Belgrade-local).

---

## Summary of discrepancies vs the brief's spec claims

1. **`scope` enum has no `REVIEW` member** (item 1d). Spec lists `review` as a live wire scope; the enum is `PRODUCT/PROFILE/CHAT/REPORT`. Reviews must send `scope="product"`; `scope="review"` → `INVALID_SCOPE`. `REPORT` is present but reserved (throws on use). — `ImageScope.java:11-15`.
2. **Translation count is higher than claimed** (item 2). Spec: 17 keys / 68 rows. Actual: **29 `image.*` keys / 116 rows**, in two namespaces (INPUT + ERRORS). All 17 spec keys are seeded; 7 seeded keys are unmentioned by the spec (4 are live DELETE error codes). No spec-active key is missing.
3. **Sweeper fires at 03:00 Europe/Belgrade, not 03:00 UTC** (item 4). `@Scheduled` has no zone; `Dockerfile:13` sets `TZ=Europe/Belgrade`. Grace (24h) and prod-enabled (true) are as the spec claims.

Items with no discrepancy: `UpdateProductRequestDTO` shape (item 3 — `oldName/oldDescription/productState/moderationState` confirmed gone); all endpoint DTO shapes, response shapes, and validation orders (item 1) match a straightforward reading of the contract.

---

## Closure sections (CLAUDE.md / conventions Part 4–5)

**Cleanup performed:** none needed — read-only audit, no code touched.

**Obsoleted by this session:** nothing.

**Config-file impact:**
- `conventions.md`: no change.
- `decisions.md`: no change (the three discrepancies above are seam-analysis input for Mastermind, not decisions made here).
- `state.md`: no change.
- `issues.md`: no change authored here. Three findings are surfaced for Mastermind triage (see "For Mastermind"); whether any becomes an `issues.md` entry is Mastermind's call.

**Conventions check:**
- Part 4 (cleanliness): confirmed — no code added/changed.
- Part 4a (simplicity) / Part 4b (adjacent observations): one low adjacent observation flagged (`image.broke` stale `--to remove` comment, EN:542). Not fixed — read-only audit.
- Part 6 (translations): N/A this session (no keys added; reported only).
- Part 7 (error contract): confirmed — endpoints emit codes (`INVALID_SCOPE`, `INVALID_COUNT`, `NOT_OWNER`, etc.), never messages.
- Part 11 (trust boundaries): confirmed — all three endpoints derive identity from `SecurityContext`; no client "old value" trust.

**Known gaps / TODOs:** none — all four brief items answered with `file:line`.

**For Mastermind (seam-analysis input):**
- **[high relevance] `review` is not a backend scope.** Mobile/web must upload review images with `scope="product"`. If any client sends `scope="review"`, the backend returns `INVALID_SCOPE`. Either correct the spec wire-table at `features/image-pipeline.md:125` (drop `review` as a scope *value*, keep the "reviews use the product prefix" prose), or — if a distinct `review` scope is genuinely wanted — a backend brief is owed to add the enum member + `buildKey` branch. Recommend the former (spec correction); the as-built behavior is internally consistent.
- **[medium] Sweeper timing wording.** The "03:00 UTC" claim (spec + two code javadocs) is wrong for the Belgrade-TZ container; actual is 03:00 local (≈01:00–02:00 UTC). If the mobile backstop reasoning depends on a precise UTC time, note the offset; the 24h grace is the figure that actually matters and it is correct.
- **[low] Translation undercount.** Spec "17 keys / 68 rows" understates the seed (29 keys / 116 rows). Seven live keys are unmentioned by the spec, four of them being the DELETE error codes. Spec table could be reconciled, but nothing is broken.
- **Part 4a simplicity evidence:** Added — nothing (read-only). Considered and rejected — nothing. Simplified or removed — nothing.
