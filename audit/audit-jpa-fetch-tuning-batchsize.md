# Audit — JPA fetch tuning, batch-fetch, and request-path lazy-collection N+1

**Repo:** oglasino-backend · **Branch:** dev · **Date:** 2026-06-03 · **Mode:** READ-ONLY (no code changes)

Every claim below is grep/cat-verified against source (per the tool-fabrication hazard). File:line references are exact.

---

## TL;DR

- **No Hibernate read/fetch tuning is configured anywhere.** All JPA `properties` set across dev/stage/prod tune **writes** (`jdbc.batch_size`, `order_inserts`, `order_updates`) or are non-fetch (`lob.non_contextual_creation`, `dialect`). Nothing tunes reads.
- **`hibernate.default_batch_fetch_size` is NOT set** in any YAML. **`@BatchSize` is NOT used** on any entity/association (grep returns only unrelated `batchSize` method names in ES reindex properties).
- **`spring.jpa.open-in-view` is unset → Spring Boot default `true`.** Open-Session-In-View is what lets the facade/controller-layer DTO mappers touch LAZY associations even though the read services are **not** `@Transactional`. It also masks the N+1s below.
- **Product search/listing reads from Elasticsearch, not the DB.** The paged listing path is ES-self-contained — no per-element DB hit. (Product *detail* adds one DB query for the live favorite count; single-product, not a page.)
- **Confirmed request-path N+1s over a page of JPA entities:** Review listing (per-row translation query), Owner-review listing (per-row translation query), Follower/Following listing (per-row baseSite/region/city + products collection + shortBio), Admin Report listing (per-row reporter + reportedUser + their baseSite). **Suggestion listing and admin User listing are clean** (scalars / projection).

---

## 1. Every Hibernate property in the JPA config (dev/stage/prod), write vs read

There is **no base `application.yaml`** — only the three profile files (`src/main/resources/application-{dev,stage,prod}.yaml`). The full set of `spring.jpa.*` keys per environment:

### dev — `application-dev.yaml:31-41`
| Property | Value | Tunes |
|---|---|---|
| `spring.jpa.hibernate.ddl-auto` | `${JPA_DDL_AUTO}` | Schema (neither read nor write — DDL validation) |
| `spring.jpa.show-sql` | `${JPA_SHOW_SQL}` | Logging only |
| `hibernate.jdbc.lob.non_contextual_creation` | `true` | **Write** (LOB creation on insert/update; Postgres workaround) |
| `hibernate.dialect` | `${JPA_HIBERNATE_DIALECT}` | Neither (SQL dialect selection) |

### stage — `application-stage.yaml:38-54`
All of the dev set, **plus**:
| Property | Value | Tunes |
|---|---|---|
| `hibernate.jdbc.batch_size` | `50` | **Write** (JDBC batched inserts/updates) |
| `hibernate.order_inserts` | `true` | **Write** (groups inserts for batching) |
| `hibernate.order_updates` | `true` | **Write** (groups updates for batching) |

### prod — `application-prod.yaml:35-51`
Identical to stage: `lob.non_contextual_creation`, `dialect`, `jdbc.batch_size: 50`, `order_inserts: true`, `order_updates: true`, plus `ddl-auto` / `show-sql`.

**Conclusion:** Every fetch-relevant property is absent. The only performance-oriented Hibernate properties present (`batch_size`, `order_inserts`, `order_updates`) optimize **writes**. **No read/fetch property is configured in any environment.**

---

## 2. Global batch-fetch / `@BatchSize`

- **`hibernate.default_batch_fetch_size`** — `grep -rni "batch_fetch|batch-fetch|default_batch" src/main/resources/` → **NONE**. Not configured in any env.
- **`@BatchSize` annotation** — `grep -rn "@BatchSize" src/main/java/` → **no annotation usages**. The only `BatchSize` hits are unrelated method/field names in `properties/ElasticsearchProperties.java` (`getBatchSize()`) and `elasticsearch/.../ProductIndexer.java` (`elasticsearchProperties.getReindex().getBatchSize()`) — the ES reindex page size, not the Hibernate annotation.

**Conclusion:** There is no batch-fetch mitigation at all — neither global nor per-association. Every LAZY association in a loop fires one query per element.

---

## 3. Request-path page/list iterations that touch LAZY associations

Architecture: `controller → facade → service`. Converters are ModelMapper `Converter<Entity,DTO>` `@Service` beans invoked by `modelMapper.map(entity, Dto.class)`. None of the read services/facades involved are `@Transactional` (see §5) — these lazy reads resolve only because OSIV is on.

### 3a. Review listing (public) — **N+1 on translations**
- Endpoint: `PublicReviewController` → `DefaultReviewFacade.getReviews` — loop at `facade/impl/DefaultReviewFacade.java:58-59` (`pageData.stream().map(review -> modelMapper.map(review, ReviewDTO.class))`).
- Service: `service/impl/DefaultReviewService.java:75-76` / `79-81` → `ReviewRepository.findByTargetProductIdAndApprovedTrue` / `findByTargetUserIdAndApprovedTrue`.
- The repository query **`join fetch r.reviewer rev join fetch rev.preferredLanguage lang`** (`repository/ReviewRepository.java:22-24, 34-36`) — so `Review.reviewer` (LAZY `@ManyToOne`, `entity/Review.java:30`) is **eagerly fetched; NOT N+1**.
- **The actual N+1:** `converter/ReviewConverter.java:28` calls `reviewTranslationsService.getReviewTranslation(source.getId())`, which runs `reviewTranslationRepository.findByReviewAndLanguage(...)` — one DB query per review (`service/impl/DefaultReviewTranslationsService.java:29-34`, not cached).
- **Cost:** ~**1 extra query per review**. Default page size 20 → ~20 extra `select` queries per page (translations), on top of the 1 list query.

### 3b. Owner-review listing (secure, "my reviews") — **N+1 on translations**
- Endpoint: `ReviewController` → `DefaultReviewFacade.getMyReviews` — loop at `facade/impl/DefaultReviewFacade.java:68-69` (`modelMapper.map(review, OwnerReviewDTO.class)`).
- Service: `DefaultReviewService.java:84-95` → `findByReviewerId` / `findByTargetUserIdAndApprovedTrue`, both `join fetch r.reviewer` (`ReviewRepository.java:46-47`). Reviewer access in `converter/OwnerReviewConverter.java:32-36` is therefore **not** N+1.
- **N+1:** `converter/OwnerReviewConverter.java:28` → `getReviewTranslation(id)` per review (same per-row DB query as 3a).
- **Cost:** ~**1 extra query per review**. Page size 20 → ~20 extra per page.

### 3c. Follower / Following listing — **heavy N+1 (worst of the set)**
- Endpoint: `UserController` → `DefaultUserFacade.getMyFollowings` — loop at `facade/impl/DefaultUserFacade.java:104-108` (`modelMapper.map(user, UserInfoDTO.class)` per user).
- Service: `service/impl/DefaultFollowService.java:48-49` → `UserFollowRepository.findAllFollowingUsers` = `SELECT uf.following FROM UserFollow uf WHERE uf.follower.id = :userId` (`repository/UserFollowRepository.java:20-26`). **No join fetch** — returns `User` entities with all associations lazy.
- Per-user LAZY access inside `converter/EntityUserInfoConverter.java`:
  - `:48` `Hibernate.unproxy(source.getBaseSite())` — `User.baseSite` LAZY `@ManyToOne` (`entity/User.java:88`) → 1 query/user
  - `:49` `Hibernate.unproxy(source.getRegion())` — `User.region` LAZY (`entity/User.java:92`) → 1 query/user
  - `:50` `Hibernate.unproxy(source.getCity())` — `User.city` LAZY (`entity/User.java:95`) → 1 query/user
  - `:58` `source.getProducts()` — `User.products` LAZY `@OneToMany` (`entity/User.java:98-100`); `.size()` initializes the collection → 1 query/user
  - `:57` `userTranslationsService.getUserShortBio(id)` → `userTranslationRepository.findByUserAndLanguage(...)` per user, not cached (`service/impl/DefaultUserTranslationsService.java:26-32`) → 1 query/user
- **Cost:** ~**5 extra queries per user**. The `PagingDTO` page bound determines N; at a 20-item page that is ~100 extra queries per page. `BaseSite` is then resolved from `BaseSiteCacheService` (cached, no DB), so the baseSite *overview* lookup isn't additional DB load, but the `getBaseSite()` proxy initialization at `:48` still hits the DB once per user.

### 3d. Admin Report listing — **N+1 on reporter / reportedUser (+ their baseSite)**
- Endpoint: `AdminReportController` → `DefaultAdminReportFacade.getReports` — loop at `admin/facade/impl/DefaultAdminReportFacade.java:53` (`modelMapper.map(report, ReportDTO.class)`).
- Source: `reportRepository.findAll(spec, pageable)` (`:40-43`) — `JpaSpecificationExecutor`, **no join fetch**; `Report.reporter` and `Report.reportedUser` are LAZY `@ManyToOne` (`entity/Report.java:31, 40`).
- Per-report LAZY access in `admin/converter/ReportConverter.java:23-28`: `modelMapper.map(source.getReportedUser(), …)` and `modelMapper.map(source.getReporter(), …)` → up to **2 user loads per report**.
- Each maps via `admin/converter/UserOverviewConverter.java:28` `Hibernate.unproxy(source.getBaseSite())` → **1 baseSite proxy-init per distinct user** (overview itself then comes from `BaseSiteCacheService`, cached).
- **Partial mitigation already present:** the facade pre-batches the deletion-lock lookup (`collectNestedUserIds` → `findActivelyLockedUserIds`, `:47-48, 61-81`) so `UserOverviewConverter.resolveLockedFromDeletion` (`:51-57`) avoids a per-row `existsActiveLockForUser`. Only the deletion-lock query is batched; the user and baseSite loads are **not**.
- **Cost:** up to **~4 extra queries per report** (2 users × (entity load + baseSite init)), minus overlap when reporter/reportedUser/baseSite repeat across rows.

### 3e. Product detail (single, not a page) — DB enrichment on the ES path
Not a page iteration, but noted because it is the one DB read in the product read path: `elasticsearch/converters/ProductDetailsConverter.java:71` calls `productService.getNumberOfFavorites(source.getId())` — **1 DB query per product-detail fetch**. Single product, so not N+1; flagged for completeness.

### Clean paths (named in the brief, verified no N+1)
- **Suggestion listing** — `admin/entity/Suggestion.java` has only scalar fields (`Long userId`, `String suggestion`, `SuggestionType`); **no associations**. `SuggestionController.getSuggestions` (`:35-46`) builds `SuggestionDTO` from scalar getters. No lazy access.
- **Admin User listing** — `admin/service/impl/DefaultUsersService.java:52-58` uses a Criteria `cb.construct(UserOverviewProjection.class, …)` column projection (not entity load), then `proj.toDTO()` + cached baseSite + pre-batched locked-user set (`:82-96`). No entity association touched. Clean by design.

---

## 4. Product read/search path — DB or Elasticsearch?

**Search & listing: Elasticsearch.** Detail: Elasticsearch (+ one DB query for favorites).

Proof:
- `controller/ProductSearchController.java` exposes `GET/POST /api/public/product/search` → `DefaultProductsSearchFacade`.
- `elasticsearch/facade/impl/DefaultProductsSearchFacade.java` maps **`ProductDocument`** (ES document) to DTOs, not the JPA `Product` entity.
- `elasticsearch/service/impl/DefaultProductsSearchService.java` injects **`ElasticsearchOperations`** (`:24`) and calls `elasticsearchOperations.search(..., ProductDocument.class)` for single (`:34-37`), listing (`:47-49`), multi-id (`:59-61`), and `elasticsearchOperations.count(...)` (`:79`). **No JPA `ProductRepository` in this path.**
- DTO mapping is ES-self-contained:
  - **Listing** — `elasticsearch/converters/ProductOverviewConverter.java` reads only `ProductDocument` fields (`getBaseSite`, `getFilterReferences`, `getPrice`, `getImageKeys`, translations). **No DB, no JPA entity, no per-element query.**
  - **Detail** — `elasticsearch/converters/ProductDetailsConverter.java` reads `ProductDocument` fields; the **only** DB touch is `productService.getNumberOfFavorites(id)` at `:71` (live favorite count), plus `BaseSiteCacheService` (Redis-cached) at `:63-64`.

**Classes that prove it:** `DefaultProductsSearchService` (`ElasticsearchOperations`), `ProductDocument`, `ProductOverviewConverter` / `ProductDetailsConverter` (map from `ProductDocument`).

---

## 5. `spring.jpa.open-in-view` and post-transaction lazy access

- **Unset in all three YAMLs** (verified: `grep -rni "open-in-view|open_in_view|OpenEntityManager" src/` → NONE).
- **Spring Boot default is `true`** → Open-Session-In-View is active. The Hibernate `Session`/`EntityManager` stays bound to the request thread through view rendering, so DTO mapping in facades/controllers can initialize LAZY proxies after the (often absent) service transaction.

**Lazy association read after the service transaction / outside any `@Transactional`:** the read services in §3 are **not** `@Transactional` (`grep` shows `@Transactional` only on `DefaultUserService`, `DefaultUserDeletionService`, `DefaultUserAuditService`, `DefaultTranslationService`, `DefaultBaseSiteService`, `DefaultProductService`, `DefaultAdminProductFacade` — none of `DefaultReviewService`, `DefaultFollowService`, `DefaultSuggestionService`, nor `DefaultReviewFacade`/`DefaultUserFacade`/`DefaultAdminReportFacade`). Concrete examples where the LAZY read happens in the mapper after the service call returns, relying entirely on OSIV:

- `facade/impl/DefaultUserFacade.java:104-108` → `EntityUserInfoConverter.java:48-50,57-58` reads `User.baseSite/region/city/products` after `FollowService.getFollowingsForUser` returns (no transaction at all).
- `facade/impl/DefaultReviewFacade.java:58-59` → `ReviewConverter.java:28` reads translations after `ReviewService.getProductReviews` returns.
- `admin/facade/impl/DefaultAdminReportFacade.java:53` → `ReportConverter.java:23-28` / `UserOverviewConverter.java:28` reads `Report.reporter/reportedUser` + `User.baseSite` directly in the (non-transactional) facade.

If OSIV were disabled without first adding `join fetch` / entity graphs / batch-fetch, every one of these would throw `LazyInitializationException`. **This is the key coupling to flag before any "turn off open-in-view" change.**

---

## Files inspected (read-only)

- `src/main/resources/application-{dev,stage,prod}.yaml`
- `entity/Product.java`, `entity/User.java`, `entity/Review.java`, `entity/Report.java`, `admin/entity/Suggestion.java`
- `elasticsearch/service/impl/DefaultProductsSearchService.java`, `elasticsearch/converters/ProductOverviewConverter.java`, `elasticsearch/converters/ProductDetailsConverter.java`
- `facade/impl/DefaultReviewFacade.java`, `facade/impl/DefaultUserFacade.java`, `admin/facade/impl/DefaultAdminReportFacade.java`
- `service/impl/DefaultReviewService.java`, `service/impl/DefaultFollowService.java`, `service/impl/DefaultReviewTranslationsService.java`, `service/impl/DefaultUserTranslationsService.java`, `admin/service/impl/DefaultSuggestionService.java`, `admin/service/impl/DefaultUsersService.java`
- `converter/ReviewConverter.java`, `converter/OwnerReviewConverter.java`, `converter/EntityUserInfoConverter.java`, `admin/converter/ReportConverter.java`, `admin/converter/UserOverviewConverter.java`
- `repository/ReviewRepository.java`, `repository/UserFollowRepository.java`
- `controller/ProductSearchController.java`, `controller/PublicReviewController.java`, `controller/ReviewController.java`, `admin/controller/AdminReportController.java`, `admin/controller/SuggestionController.java`

## Could not verify / out of scope
- I did not enumerate every admin/internal endpoint — focused on the brief's named paths (Review, Report, User/Follow, Suggestion) plus the product path. Other admin list endpoints may exist; none were claimed here without source.
- Exact per-page query counts assume Hibernate de-duplicates already-loaded entities within a session (e.g. a baseSite shared across rows loads once). Real counts depend on data shape; the "~N per page" figures are upper bounds at the stated page sizes.

---

## Conventions check
- **Part 4 (cleanliness):** Read-only audit; no code written, no debug output, no TODO/FIXME introduced. None needed.
- **Part 4a (simplicity):** N/A — no implementation.
- **Part 4b (adjacent observations):** Noted `// TODO pageable and join left with collection is not good...` already present at `ReviewRepository.java:31` (pre-existing, not mine). The OSIV→lazy coupling (§5) is the load-bearing observation for any future fetch-tuning work.
- **Part 11 (trust boundaries):** N/A — no trust-affecting changes.

## Config-file impact
No edit required to `conventions.md`, `decisions.md`, `state.md`, or `issues.md` from this audit itself. **If** Mastermind decides to act on the N+1 findings (§3) or an OSIV change (§5), that would warrant an `issues.md` entry — draft pointer in "For Mastermind" below. This audit does not, on its own, require any config-file write.

## Obsoleted by this session
Nothing.

## For Mastermind
1. **No read-side fetch tuning exists** (§1, §2): no `default_batch_fetch_size`, no `@BatchSize`. The cheapest broad mitigation for the N+1s in §3c/§3d would be a single global `spring.jpa.properties.hibernate.default_batch_fetch_size` (e.g. 25–100), which would collapse the per-row `@ManyToOne`/collection loads into batched `IN (...)` fetches with zero code change. Flagging as a candidate; not implementing without a brief.
2. **The translation-per-row queries** (§3a/§3b — `getReviewTranslation`; §3c — `getUserShortBio`) are *not* association lazy-loads and would **not** be fixed by batch-fetch; they are explicit per-element repository calls. Fixing them needs a batched lookup (load all translations for the page's ids in one query) — a code change per path.
3. **OSIV is load-bearing** (§5): several read paths have no `@Transactional` and rely on `open-in-view=true` to map lazy fields. Any move to disable OSIV must be paired with `join fetch`/entity graphs/batch-fetch on the §3 paths first, or those endpoints will throw `LazyInitializationException`.
4. Suggested `issues.md` candidates (Docs/QA to write, not me): "Request-path N+1: follower/following listing (~5 queries/user)", "Request-path N+1: admin report listing (reporter/reportedUser + baseSite)", "Per-row translation queries in review listings", "No Hibernate batch-fetch configured; OSIV masks N+1s".
