# Audit — Repositories & Services: bottlenecks and improvements

**Repo:** oglasino-backend
**Date:** 2026-06-03
**Scope:** JPA repositories (`repository/`, `admin/**/repository/`) and the service layer that consumes them. Read-only audit — **nothing implemented**. Recommendations are framed to preserve the current workflow (no contract changes, no schema rewrites, no swap of the ES-backed read path).
**Method:** direct read of repositories, entities, the heaviest service impls, JPA/Hikari config, and the cache layer. Sub-agent findings were re-verified against source (CLAUDE.md note on Read fabrication); corrections are called out inline.

---

## TL;DR

The data layer is in good shape: constructor/interface projections are used widely, `@ManyToOne` on the hot entity (`Product`) is explicitly `LAZY`, reference data is cached in Redis at the service layer, the product **read/search path is Elasticsearch** (not DB), and the connection pool is deliberately capped for the DO 1GB Postgres (25-conn) box. So most "classic JPA" advice does **not** apply here — see *Already handled / do not touch* at the bottom before acting on anything.

The findings that actually matter, in priority order:

1. **No global `default_batch_fetch_size`** — the single highest-leverage, lowest-risk change. (Tier 1)
2. **`Review.translations` is `EAGER`** → N+1 on every review-listing page. (Tier 1)
3. **Reindex Stage-B fetches three collections in one query** → cartesian-product row blow-up. (Tier 2)
4. **`Product.imageKeys` `@ElementCollection` defaults to EAGER** → unwanted extra fetch on every single-product DB load. (Tier 2)
5. **Composite index for the `activeProducts` count subquery.** (Tier 2)
6. Per-row patterns in scheduled jobs; minor `@Transactional` / dead-code cleanups. (Tier 3)

---

## Tier 1 — high leverage, low risk

### 1.1 No `hibernate.default_batch_fetch_size` is configured

- **Where:** `application-prod.yaml:46-54` (and `application-stage.yaml:49-54`). Hibernate has `jdbc.batch_size: 50`, `order_inserts`, `order_updates` — those tune **writes**. There is **no** `default_batch_fetch_size`, and **no** `@BatchSize` anywhere in the codebase. Confirmed: `grep default_batch_fetch_size` → none.
- **Cost:** every time code iterates a page/list of entities and touches a `LAZY` association or collection, Hibernate fires one SELECT *per parent row* (N+1). This compounds against the deliberately small 18-connection pool.
- **Direction:** set `hibernate.default_batch_fetch_size` (e.g. 25–50) in the JPA properties block. It batches those N lazy loads into `ceil(N/size)` `IN (...)` queries globally, no code change, no contract change. This alone mitigates finding 1.2 and the lazy-collection cost in the jobs below.
- **Risk:** very low; it only changes how lazy loads are *batched*, never *whether* they happen.

### 1.2 `Review.translations` is `FetchType.EAGER`

- **Where:** `entity/Review.java:43-48` (`@OneToMany(... fetch = FetchType.EAGER)`).
- **Consumers:** `DefaultReviewService.getProductReviews / getUserReviews / getMyReviews` → `ReviewRepository.findByTargetProductIdAndApprovedTrue` / `findByTargetUserIdAndApprovedTrue` / `findByReviewerId` (`ReviewRepository.java:20-50`). These queries `join fetch r.reviewer` (a `@ManyToOne` — pagination-safe) but **do not** fetch `translations`. Because `translations` is EAGER, Hibernate issues one extra SELECT **per review row** to populate it.
- **Cost:** page size 20 ⇒ ~20 extra queries per reviews page, on a user-facing endpoint.
- **Direction (pick one, both preserve behavior):**
  - Make `translations` `LAZY` and rely on finding 1.1 (`default_batch_fetch_size`) to collapse the loads into 1–2 `IN` queries. Cleanest.
  - Or keep current behavior but stop loading reviewer-language data you don't need.
  - Note: a `JOIN FETCH translations` on a *paginated* collection query would trigger Hibernate's "applying in memory" pagination degrade — avoid that route. There is already a stale `// TODO pageable and join left with collection is not good...` at `ReviewRepository.java:31` signalling the author is aware.
- **Risk:** low. Flipping to LAZY changes fetch timing only; all current call sites access translations inside the same transaction.

---

## Tier 2 — real, moderate

### 2.1 Reindex Stage-B fetches three collections in a single query (cartesian product)

- **Where:** `ProductRepository.findForIndexingByIds` (`ProductRepository.java:72-91`), called by `ProductIndexer.doReindex` (`ProductIndexer.java:157`). The two-stage paginated read (IDs first at `:142`, then hydrate) is already a good design and correctly dodges the collection-fetch pagination warning.
- **Issue:** the hydrate query `JOIN FETCH`es `translations` (Set), `filterValues` (List), **and** `imageKeys` (Set element-collection) simultaneously. The result set Hibernate materializes per product is `translations × filterValues × imageKeys` rows — a cartesian product. For a product with 5 translations, 8 filter values and 10 images that's 400 rows for one product, ×`batchSize` products per page.
- **Cost:** inflated heap and CPU during reindex; the larger the per-product collections, the worse. It's a batch job, not request-path, so it's correctness-safe but wasteful and can stress the box during a full reindex.
- **Direction:** fetch at most one collection per query; load the others via a second pass or via `default_batch_fetch_size` (finding 1.1) on the page of IDs. (Only one of the three is a `List`/bag, so there's no `MultipleBagFetchException` today — this is purely a row-multiplication cost, not a correctness bug.)
- **Risk:** medium — touches the reindex path; would need a reindex smoke test.

### 2.2 `Product.imageKeys` `@ElementCollection` defaults to EAGER

- **Where:** `entity/Product.java:85-88` — `@ElementCollection` with no `fetch` ⇒ defaults to `EAGER`. **Correction to sub-agent report:** every `@ManyToOne` on `Product` *is* explicitly `LAZY` (`Product.java:26-56`); `imageKeys` is the only eager association on the entity.
- **Cost:** every single-`Product` DB load pays an extra fetch for `product_images`, even when images aren't needed — e.g. `updateProduct` (`DefaultProductService.java:224`), `changeProductState` (`:645`), `updateProductWithFunc` (`:580`), `deleteProduct`. The user-facing product *read* path is Elasticsearch, so this does **not** hit listing/detail reads — it's the mutation paths that pay.
- **Direction:** make it `@ElementCollection(fetch = FetchType.LAZY)`, then `JOIN FETCH p.imageKeys` (or rely on 1.1) only where images are actually consumed. Note `Review.imageKeys` is *already* LAZY (`Review.java:56`) — this just brings `Product` in line.
- **Risk:** low-medium — `updateProduct` reads `product.getImageKeys()` for the removed-keys diff (`DefaultProductService.java:248`) and `deleteProduct` reads it off a virtual thread (`:605`); both must remain inside an open session or be explicitly fetched. Verify those two sites before flipping.

### 2.3 No composite index for the `activeProducts` count subquery

- **Where:** correlated subquery in `UserRepository.findUserInfoById` / `findUserInfoByFirebaseUid` (`UserRepository.java:146-150, 181-185`): `SELECT COUNT(p2) ... WHERE p2.owner = u AND p2.productState = :activeState`. `Product` is indexed on `owner_id` alone (`Product.java:14`); `product_state` is unindexed.
- **Cost:** the count uses the `owner_id` index then filters `product_state` as a row check. For prolific sellers that's extra heap fetches per public-profile load. Bounded because `UserInfoDTO` is Redis-cached (`DefaultUserService.java:34-44`), so this fires on cache miss / eviction only.
- **Direction:** a composite `(owner_id, product_state)` index (or a partial index on `product_state = 'ACTIVE'`) makes the count index-only. **Do not** add a standalone `product_state` index for "filtering" — filtering is Elasticsearch, not DB.
- **Risk:** low; additive index. Schema is Flyway-owned (`ddl-auto: validate`), so this is a migration for Docs/QA/Mastermind to route, not an entity change — see *Config-file impact*.

---

## Tier 3 — minor / cleanliness

### 3.1 Scheduled-job per-row access (by design, low priority)

- `UserDeletionScheduledJobs.processScheduledDeletions` (`:83-108`): per row, `existsActiveLockForUser` + `findById`. `sendDeletionReminders` (`:145-167`): per row `findById`, then `user.getPreferredLanguage().getCode()` (lazy hop) and a per-row `save`.
- `DefaultMessagingCleanupService.runScheduledCleanup` (`:108-116`): per-row `delete`/`save`.
- **Note:** the per-row try/catch is **intentional fault isolation** (documented at `UserDeletionScheduledJobs.java:124-127`) — one row's failure must not abort the batch. So this is not a bug. The only safe win is batching the *reads* (one `findAllById` over the page instead of N `findById`) while keeping per-row processing/error handling. Daily batch, off request-path ⇒ low priority. `default_batch_fetch_size` (1.1) already removes the `getPreferredLanguage()` lazy hop cost.

### 3.2 `getUserById` opens a read-write transaction

- `DefaultUserService.getUserById` (`:52`) is `@Transactional` (not `readOnly = true`), unlike its neighbours (`getUserByEmail`, `getUserPhoneNumberIfUserAllows`, etc. are all `readOnly`). Minor: it returns a managed entity callers may mutate-and-save, so confirm no caller relies on dirty-checking before changing it. Low.

### 3.3 `findAllForIndexing()` appears unused

- `ProductRepository.findAllForIndexing` (`:35-53`) is the non-paginated full-graph load. `ProductIndexer` uses the two-stage paginated path instead (`:142,157`); `grep` finds no other `src/main` caller. Likely dead code superseded by the paginated reindex. Candidate for deletion (cleanliness, not perf) — verify no test/tooling references it first.

### 3.4 Reference-data EAGER `@ManyToOne` (low, leave unless touched)

- `Filter.filterRange` is explicitly `EAGER` (`Filter.java:42-44`); `Translation.language`, `CategoryFilter.category/filter`, `Catalog.baseSite`, `CatalogCategoryAssignment.baseSite/category` are default-EAGER `@ManyToOne`. These are catalog/reference entities loaded at startup / cache-build, not on the hot path, so the cost is negligible. Listed for completeness; not worth changing on their own.

### 3.5 `findByProductStateAndUpdatedAtBefore` (removal job) is unindexed

- `ProductRepository.findByProductStateAndUpdatedAtBefore` (`:93`) filters `product_state` + `updated_at` with no supporting index ⇒ scan. It's a daily cleanup batch over `DELETED` products, so low concern. The composite index in 2.3 would not cover this (different leading column); only worth an index if the `DELETED` backlog ever grows large.

---

## Already handled / do not touch (workflow-preservation)

Calling these out so they are **not** "fixed" into regressions:

- **Reference-data caching is already done via Redis Spring Cache**, not Hibernate L2C. `redisTranslations`, `redisLanguages`/`redisLanguage`, `redisBaseSite`/`redisBaseSiteOverviews`/`redisActiveBaseSiteCodes`, `redisBaseCurrency`, `redisUserInfo`, `redisUserAuth` (see `DefaultTranslationService:48`, `DefaultLanguageService:20-26`, `DefaultBaseSiteService:31-72`, `DefaultBaseCurrencyService:52`, `DefaultUserService:34-44`, `DefaultFirebaseAuthService:86`). Do **not** add `@Cacheable`/Hibernate second-level cache for Language/Currency/BaseSite/Translation — it would duplicate and fight the existing eviction strategy (`DefaultUserService` `@CacheEvict` chains, `RedisCacheEvictConfig`).
- **Product listing/search is Elasticsearch** (`elasticsearch/**`, `ProductDocument`, `DefaultProductsSearchService`). DB `Product` loads are single-row (mutations) or batched (reindex). So `Product` entity fetch tuning matters only for the mutation paths — don't "optimize" it for a list path that doesn't exist.
- **View counting is already offloaded to Redis** with virtual-thread + dedup window (`DefaultProductSeenService`, `RedisViewCounterService`, `ProductRepository.incrementNumberOfViewsBy`). No per-view DB write on the hot path. Leave as-is.
- **Connection pool is deliberately capped** (`maximum-pool-size: 18`, comment "tuned for DO Basic 1GB Postgres (25 conn limit)"). Do not raise it to "fix" throughput — finding 1.1 reduces query *count*, which is the right lever for this box.
- **Auth/user reads use flat projections** (`AuthenticatedUserDTO`, `UserInfoProjection`) — no entity graph, no lazy proxies. Good; leave.

---

## Suggested sequencing

1. **1.1** (`default_batch_fetch_size`) — config-only, broad benefit, lowest risk. Do first; it partially absorbs 1.2, 2.1, 3.1.
2. **1.2** (`Review.translations` → LAZY) — after 1.1 is in, so the lazy loads batch.
3. **2.2** (`Product.imageKeys` → LAZY) — verify the two read sites (update diff, delete virtual thread) first.
4. **2.1** (reindex collection split) and **2.3** (composite index) — need a reindex smoke / a Flyway migration respectively.
5. Tier 3 as cleanup when convenient.

Each is independently shippable and none changes an HTTP contract or the ES read path.

---

## Config-file impact

- Finding **2.3** (and optionally **3.5**) would need a **Flyway migration** to add an index. Schema is Flyway-owned (`ddl-auto: validate`); the migration is in-repo (not one of the four governed config files), but the *decision* to add an index and its rollout belongs to Mastermind/Docs/QA. Draft for routing: "Add composite index `(owner_id, product_state)` on `product` to make the public-profile active-product count index-only."
- No edits implied to `conventions.md`, `decisions.md`, `state.md`, or `issues.md`. Findings 1.1/1.2/2.x could each become an `issues.md` backlog row if Mastermind wants them tracked — that is Docs/QA's write, drafted here, not made here.

## Conventions check

- Part 4 (cleanliness): audit is read-only; no code touched. Finding 3.3 (`findAllForIndexing`) flagged as a cleanliness candidate, not actioned.
- Part 11 (trust boundaries): no trust-boundary impact; none of the recommendations move a decision off the server.
- Part 7 (error contract): untouched.
- Nothing implemented — no `spotless`/`test` run required this session.

---

# Implementation view (backend engineer — 2026-06-03)

Second pass: every finding above re-read against source line-by-line (no sub-agent; direct
`Read`/`grep`). All findings reproduce. This section records (a) the verification result, (b) the
exact change for each, and (c) **five material refinements** where the source tells a more precise
story than the original write-up. Still read-only — no code changed.

## Verification result (all confirmed)

| # | Claim | Verified at | Verdict |
|---|-------|-------------|---------|
| 1.1 | No `default_batch_fetch_size` / no `@BatchSize` | `application-prod.yaml:44-51`, `application-stage.yaml:47-54`, `application-dev.yaml:38-41`; `grep` clean | ✅ |
| 1.2 | `Review.translations` EAGER; queries don't fetch it | `Review.java:43-48`; `ReviewRepository.java:20-50` (TODO `:31`) | ✅ |
| 2.1 | Reindex hydrate triple-collection fetch | `ProductRepository.java:72-91`; `ProductIndexer.java:142,157` | ✅ |
| 2.2 | `Product.imageKeys` defaults EAGER; all `@ManyToOne` LAZY | `Product.java:85-88`; `Product.java:26-56` | ✅ |
| 2.3 | `activeProducts` count subquery; `product_state` unindexed | `UserRepository.java:147-150,181-185`; `V1__init_schema.sql` (no such index) | ✅ |
| 3.1 | Per-row `findById` in deletion jobs | `UserDeletionScheduledJobs.java:91,147,155,162` | ✅ |
| 3.2 | `getUserById` is RW `@Transactional` | `DefaultUserService.java:51-55` | ✅ |
| 3.3 | `findAllForIndexing()` unused | `grep` → only the declaration + a javadoc cross-ref; zero callers (incl. tests) | ✅ |
| 3.4 | `Filter.filterRange` EAGER | `Filter.java:42-44` | ✅ |
| 3.5 | Removal-job query unindexed | `ProductRepository.java:93`; no `(product_state, updated_at)` index in V1 | ✅ |

## Refinements (where the code is more specific than the write-up)

**R1 — 1.2's "low risk" rests on OSIV, not on a transaction. State that explicitly.**
The write-up says the review call sites "access translations inside the same transaction." They do
not. Neither `DefaultReviewFacade` (the `modelMapper.map(...)` at `:59`/`:69`) nor the
`DefaultReviewService` read methods (`:74-96`) carry `@Transactional`. The entity→DTO mapping runs in
the facade, **after** the repository's own transaction has closed. It works today only because
**Open-Session-In-View is on** — `spring.jpa.open-in-view` is unset in all three YAMLs, so it takes
the Spring Boot default of `true`. Consequences for the flip:
  - Flipping `translations` to LAZY is safe **only while OSIV stays on**. If a future hardening pass
    disables OSIV (a common recommendation, and plausible given the active `backend-security-hardening`
    track), these review endpoints — and any other facade that lazy-touches an association after the
    service returns — throw `LazyInitializationException`. The durable fix is a `@EntityGraph` /
    `JOIN FETCH` of `translations` on the three review queries, but that re-introduces the
    paginated-collection-fetch degrade the `:31` TODO already warns about. So the **correct** durable
    shape is LAZY + `default_batch_fetch_size` (which batches the per-row loads into 1–2 `IN` queries
    *inside* the OSIV window) — i.e. **1.1 must land with or before 1.2**, otherwise 1.2 only relocates
    the N+1 from eager-time to OSIV-lazy-time, it does not remove it.

**R2 — 2.2's `deleteProduct` site is a *guaranteed* break under LAZY, not a "verify," and needs a
2-line fix.** `DefaultProductService.deleteProduct` (`:603-605`) reads `product.getImageKeys()` inside
a `Thread.ofVirtual()` lambda. Its **only** caller is the scheduled `ProductRemovalJob` (`:40`), which
loads the page via `getProductsForRemoval` — a `@Transactional(readOnly = true)` method whose
transaction has **already closed** by the time the loop calls `deleteProduct`. So the entity is
**detached**, and the access is on a **virtual thread with no bound session**. Today that works purely
because the collection is EAGER (already populated at load). Flip to LAZY and it is an unconditional
`LazyInitializationException` on the virtual thread (silently swallowed — the thread just dies, images
never deleted). Required change, same session as the flip:

```java
public void deleteProduct(Product product) {
  // snapshot inside the caller's thread before the collection detaches / the vthread starts
  var keysToDelete = new HashSet<>(CollectionUtils.emptyIfNull(product.getImageKeys()));
  Thread.ofVirtual().start(() -> imageService.deleteImagesAndClearOwnership(keysToDelete));
  ...
}
```
(Note for accuracy: the user-facing `DELETE /dashboard/product` is **not** this method — it is
`changeProductState(DELETED)`, a soft delete, `DashboardProductController.java:82-87`. The hard
`deleteProduct(Product)` is **batch-only**. The write-up listing it among user-facing mutation paths is
slightly off; it doesn't change the fix, but it changes the blast radius — a bad flip breaks the weekly
sweep, not a request.)

**R3 — `updateProductWithFunc` is not a risk; it is a pure beneficiary.** The write-up lists it
(`:580`) among the eager-cost sites. Its only caller passes a func that touches `numberOfFavorites`
only and never reads `imageKeys` (`DefaultFavoriteProductFacade.java:84-94`), and it re-loads the
product via `findById` *inside* the vthread. So under LAZY it simply stops eager-fetching `imageKeys`
on every favorite toggle — a clean win, zero breakage. The in-transaction reads in `updateProduct`
(`:248`) and `isNoOpUpdate` (`:349`) are also safe (lazy-load resolves inside the class-level
`@Transactional`).

**R4 — there is one more `Product.imageKeys` DB-read path the write-up misses, and it's a *read*, not
a mutation.** `ProductForUpdateConverter` (`:74`) maps `Product.imageKeys` → DTO for the owner's
update-form GET (`DashboardProductController.getProductDetails:55` → `DefaultProductFacade
.getProductForUpdate:41` / `toUpdateData:46-47`, mapped after the txn closes → OSIV-served). So
"only the mutation paths pay" is incomplete — this GET reads `imageKeys` too, and **legitimately needs
them** (the form shows existing photos). Implication: flipping to LAZY saves nothing here and adds one
OSIV-lazy query unless the `getProductForUpdate` load fetches `imageKeys`. When implementing 2.2, add a
`@EntityGraph(attributePaths = "imageKeys")` (or `JOIN FETCH`) to that one load and the GET stays
single-query. The ES read converters (`ProductOverviewConverter:62`, `ProductDetailsConverter:70`)
read off `ProductDocument`, **not** the entity — confirmed `Converter<ProductDocument, …>` — so the
audit's "reads are ES" holds for the public path. `DocumentProductConverter` (`Converter<Product,
ProductDocument>`) and `ProductIndexer:363` (`Hibernate.initialize(imageKeys)`) keep the reindex path
correct under LAZY.

**R5 — 2.3 is an in-place V1 edit per Part 12, *not* a new Flyway migration to route through Docs/QA.**
The repo has only `V1__init_schema.sql` (no V2+), and the project is pre-production (state.md target:
prod in ~1 month). Per conventions Part 12 ("Pre-production V1 schema fold"), index additions edit
`V1__init_schema.sql` **in place** until first prod deploy. There is already a direct precedent in that
file — `idx_product_deactivated_by_system` is a **partial composite** `(owner_id, deactivated_by_system)
WHERE deactivated_by_system = true` (`V1__init_schema.sql:1173-1175`). So 2.3's index is the same
pattern and is ordinary in-repo backend work, not a governed-config-file change. This **supersedes the
"migration for Docs/QA/Mastermind to route" framing** in the audit's *Config-file impact* section — the
Mastermind *decision to add the index* still applies, but the mechanism is a V1 edit the backend agent
makes directly. Two index shapes are both valid (Part 12's IMMUTABLE rule is satisfied by either):
  - composite `(owner_id, product_state)` — makes the count index-only for any state; or
  - partial `(owner_id) WHERE product_state = 'ACTIVE'` — smallest, since the subquery only ever counts
    ACTIVE. Predicate is a string-literal compare ⇒ IMMUTABLE-safe.

**R5b — second uncovered subquery in the same projection (minor, additive).** The same
`findUserInfoById` / `findUserInfoByFirebaseUid` projections carry a *second* correlated subquery —
`scheduledDeletionAt` over `UserDeletionRequest WHERE userId = u.id AND status = PENDING`
(`UserRepository.java:157-162,192-197`). `UserDeletionRequest` is indexed only on
`scheduled_deletion_at` (`idx_udr_pending_due`), so this subquery filters `userId` unindexed. Left
unflagged in the original because the table is tiny (one row per pending-deletion user) and the whole
projection is Redis-cached — genuinely low priority. If 2.3's index work is opened anyway, a partial
`(user_id) WHERE status = 'PENDING'` on `user_deletion_request` closes it in the same edit. Not worth
its own change.

## Per-finding implementation plan

**1.1 — `default_batch_fetch_size` (do first).** Add one line to the
`spring.jpa.properties.hibernate` block in all three env YAMLs (keep parity; dev already mirrors the
block at `application-dev.yaml:38-41`):
```yaml
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
        default_batch_fetch_size: 50   # collapse N lazy loads → ceil(N/50) IN-queries
```
50 mirrors the existing `jdbc.batch_size`; 25 is equally defensible. Config-only, no contract/ES
impact. Test: existing `./mvnw test` suite must stay green (it exercises the lazy paths); optionally
assert query count on a review page if a counting harness exists. Partially absorbs 1.2, 2.1, 3.1.

**1.2 — `Review.translations` → LAZY (after/with 1.1).** One annotation: `Review.java:47`
`fetch = FetchType.EAGER` → `FetchType.LAZY`. No query change (the three queries already don't fetch
it; 1.1 batches the now-lazy loads inside OSIV). Carries the R1 caveat: durability depends on OSIV. Add
a test asserting a review page still serializes `translations` end-to-end.

**2.2 — `Product.imageKeys` → LAZY (after 1.1; needs the R2 fix + R4 fetch).**
`Product.java:85` add `(fetch = FetchType.LAZY)`. **Same session, mandatory:** the `deleteProduct`
snapshot (R2). **Recommended:** `@EntityGraph(attributePaths = "imageKeys")` on the
`getProductForUpdate` load (R4) to keep the owner GET single-query. Smoke: a hard-delete sweep over a
product with images (R2 path) + an owner update-form load (R4 path). `updateProductWithFunc` and the
in-txn `updateProduct` reads need nothing (R3).

**2.1 — reindex collection split (own change, reindex smoke).** In `findForIndexingByIds`
(`:72-91`), drop two of the three collection `JOIN FETCH`es (keep at most one; `imageKeys` is already
`Hibernate.initialize`-d at `ProductIndexer.java:363`, and with 1.1 in place the remaining collections
batch-load). Cartesian blow-up is `translations × filterValues × imageKeys` per product × batch (default
500), so this is the biggest heap win during a full reindex. Keep `findAllForIndexing` and
`findForIndexingByIds` in sync per the existing javadoc — or fold 3.3 in and delete the former.
Verify via an admin `/es/reindex` smoke against a product with multi-row collections.

**2.3 — index (in-place V1 edit per R5).** Add the partial-or-composite index to
`V1__init_schema.sql` next to `idx_product_deactivated_by_system`. `ddl-auto: validate` means the index
must exist for boot; since V1 is rebuilt on every dev/stage reset, no separate migration step. Optional
R5b index in the same edit.

**Tier 3.** 3.2: `DefaultUserService.getUserById:52` add `readOnly = true` after confirming no caller
mutates-then-saves the returned managed entity (grep the callers first). 3.3: delete
`findAllForIndexing` once 2.1 stops referencing it as the "mirror." 3.1/3.4/3.5: leave unless a brief
scopes them; 1.1 already removes the `getPreferredLanguage()` lazy-hop cost in 3.1.

## Revised sequencing

1. **1.1** (config) — alone, broad, lowest risk.
2. **1.2** (Review LAZY) — with 1.1 in the same deploy so the N+1 actually collapses (R1).
3. **2.2** (Product imageKeys LAZY) — bundled with the R2 `deleteProduct` snapshot + R4 fetch.
4. **2.1** (reindex split) + **3.3** (delete dead method) together — both touch the reindex graph.
5. **2.3** (V1 index edit) — independent; in-place per R5.
6. Tier 3 cleanups as convenient.

## Config-file impact (supersedes the section above)

- **2.3 / R5b are in-repo V1 schema edits, not governed-config-file changes and not a routed Flyway
  migration** (pre-prod V1 fold, Part 12). No edit to `conventions.md` / `decisions.md` / `state.md` /
  `issues.md` is *required* by any finding. The Mastermind *decision* to add the 2.3 index still
  belongs upstream; the mechanism does not.
- Optional `issues.md` rows (Docs/QA write, drafted not made here): any of 1.1/1.2/2.1/2.2/2.3 if
  Mastermind wants them tracked as backlog rather than executed immediately. **Closure gate:** nothing
  here is "drafted-but-pending" against a config file — all of it is code/schema work for a future
  engineering brief.
