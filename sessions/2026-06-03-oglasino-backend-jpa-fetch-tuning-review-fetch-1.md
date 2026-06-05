# Session summary — Audit: JPA fetch tuning (review-listing fetch path)

**Repo:** oglasino-backend
**Branch:** dev (read-only; no checkout/commit/push)
**Date:** 2026-06-03
**Task:** Read-only audit of the review-listing endpoints (product reviews, user reviews, my reviews) from controller to JSON, with focus on fetch behavior and transaction/session boundaries.

Scope was READ-ONLY. No code changed. No spotless/test run (brief waived them). Every claim below was cross-checked with `grep`/`sed` against the file it cites (per the repo's tool-fabrication hazard note); where I could not verify something from source I say so explicitly.

---

## Endpoints in scope (controller → service → repo)

| Endpoint | Controller | Page size | Service method | Repo query |
| --- | --- | --- | --- | --- |
| Product reviews | `PublicReviewController.getReviews` (`productId` set), `GET /api/public/review` | **20** (`PageRequest.of(page, 20, Sort ASC createdAt)`, `PublicReviewController:33`) | `DefaultReviewService.getProductReviews` | `ReviewRepository.findByTargetProductIdAndApprovedTrue` |
| User reviews | `PublicReviewController.getReviews` (`userId` set) | **20** (same `PageRequest`, `PublicReviewController:33`) | `DefaultReviewService.getUserReviews` | `ReviewRepository.findByTargetUserIdAndApprovedTrue` |
| My reviews | `ReviewController.getMyReviews`, `GET /api/secure/review/product` | **20** (`PageRequest.of(page, 20, Sort ASC createdAt)`, `DefaultReviewService:90`) | `DefaultReviewService.getMyReviews` | `findByReviewerId` (tab `GIVEN`) **or** `findByTargetUserIdAndApprovedTrue` (tab `RECEIVED`/other) |

DTO mapping in all three goes through `DefaultReviewFacade` → `ModelMapper` → a registered `Converter` (`ReviewConverter` for the public path, `OwnerReviewConverter` for my-reviews). Converters registered in `MapperConfiguration:68-69`.

---

## Finding 1 — Review entity associations and fetch types

From `entity/Review.java` (verified by `cat`):

| Field | Mapping | Declared fetch | Effective fetch | Source |
| --- | --- | --- | --- | --- |
| `reviewer` | `@ManyToOne` | `FetchType.LAZY` (explicit) | LAZY | `Review.java:30-31` |
| `targetUser` | `@ManyToOne(optional = false)` | `FetchType.LAZY` (explicit) | LAZY | `Review.java:33-34` |
| `translations` (`Set<ReviewTranslation>`) | `@OneToMany(mappedBy="review", cascade=ALL, orphanRemoval=true)` | `FetchType.EAGER` (explicit) | **EAGER** | `Review.java:43-48` |
| `imageKeys` (`Set<String>`) | `@ElementCollection` `@CollectionTable(review_images)` | `FetchType.LAZY` (explicit) | LAZY | `Review.java:56-59` |
| `targetProductId` | plain `@Column Long` (no association) | — | — | `Review.java:36` |

Notes on JPA defaults (for completeness, though every association here declares its fetch explicitly so no default is in play):
- `@ManyToOne` default is EAGER — here overridden to LAZY on both `reviewer` and `targetUser`.
- `@OneToMany` default is LAZY — here overridden to EAGER on `translations`.
- `@ElementCollection` default is LAZY — `imageKeys` matches the default.

`Review` extends `BaseEntity`; I read `Review.java` end-to-end and it declares no other associations. `BaseEntity` was not separately opened, but the brief scopes "the Review entity" and all four associations above are on `Review` itself.

A relevant transitive association: the listing queries also touch `User.preferredLanguage` (`@ManyToOne(optional=false)` with **no fetch declared → JPA default EAGER**, `User.java:77-79`) and join-fetch it explicitly (see Finding 2).

---

## Finding 2 — Which associations the listing queries JOIN FETCH

All three listing queries are `@Query` JPQL in `ReviewRepository` (verified by `cat`). They are structurally identical on the fetch side:

```
select r from Review r
join fetch r.reviewer rev
join fetch rev.preferredLanguage lang
where ...
```

- `findByTargetProductIdAndApprovedTrue` — `ReviewRepository:20-29`
- `findByTargetUserIdAndApprovedTrue` — `ReviewRepository:32-41`
- `findByReviewerId` — `ReviewRepository:43-50`

**JOIN FETCHed (eagerly, in the main query):**
- `Review.reviewer` (the `User`) — to-one
- `reviewer.preferredLanguage` (the `Language`) — to-one

**NOT join-fetched, therefore loaded by separate SQL after the main query returns:**
- `Review.translations` — EAGER, so Hibernate issues an immediate separate `SELECT` per Review row (N+1). Not lazy — eager-but-not-fetch-joined.
- `Review.imageKeys` — LAZY, initialized later (at converter access / JSON serialization), one `SELECT` per Review row (N+1).
- `Review.targetUser` — LAZY; **not touched at all** by the public/my-reviews converters, so it never loads on this path. (`OwnerReviewConverter`/`ReviewConverter` read only `reviewer`, not `targetUser`. The admin `AdminReviewConverter:29` does read `targetUser`, but admin is a different endpoint, out of this brief's three.)

Cross-check on the join targets: the only to-one associations join-fetched are `reviewer` and `preferredLanguage`; **no collection is fetch-joined** — this matters for Finding 5.

---

## Finding 3 — SQL queries per page (page size 20)

Config confirmed from source: there is **no** `hibernate.default_batch_fetch_size` and **no** `@BatchSize` anywhere (`grep` across `src/main/resources/*.yaml` and `src/main/java` — the only `BatchSize` hits are the unrelated Elasticsearch reindex property). So collection loads are strictly one-row-at-a-time — no batching collapses the N+1.

Counting for one page of the **product-reviews** endpoint (20 approved reviews; the user-reviews and GIVEN/RECEIVED my-reviews paths are identical in shape):

1. **Main SELECT** — reviews + `join fetch reviewer` + `join fetch preferredLanguage`. To-one fetch joins only, so SQL `LIMIT/OFFSET` pagination is applied in the database (no in-memory paging). → **1 query**
2. **COUNT query** — because the return type is `Page<Review>`, Spring Data issues a separate count. → **1 query**
3. **EAGER `Review.translations`** — not fetch-joined, so Hibernate eagerly initializes each review's collection with its own `SELECT`. → **~20 queries** (1 per review)
4. **`getReviewTranslation(reviewId)` in the converter** — `ReviewConverter:28` / `OwnerReviewConverter:28` call `reviewTranslationsService.getReviewTranslation(id)` → `ReviewTranslationRepository.findByReviewAndLanguage(...)`, one query per review to fetch the single language-matched translation row. → **~20 queries** (1 per review)
5. **LAZY `imageKeys`** — `setImageKeys(source.getImageKeys())` stores the lazy `PersistentSet` onto the DTO; it initializes when iterated (at the latest, Jackson serialization), one `SELECT` per review. → **~20 queries** (1 per review)
6. `reviewer` + `preferredLanguage` — already in the main query → **0 extra**. The `ReviewerInfo` projection reads only `id`/`displayName`/`profileImageKey`, all columns on `User`, no further lazy navigation.

**Total ≈ 1 + 1 + 20 + 20 + 20 = ~62 SQL statements to render 20 reviews** — roughly **3 N+1 selects per review** (eager translations, per-language translation lookup, imageKeys) on top of the 2 fixed (main + count).

Caveat: items 3 and 4 are redundant with each other — the EAGER `translations` collection is loaded (item 3) but **never read** by any converter; the converters instead re-query the single language row (item 4). See Finding 6. Eliminating the EAGER load (item 3) would drop ~20 of the ~62 with zero behavior change.

I did not run the app or capture the SQL log to confirm the exact count empirically (read-only audit, no local DB run). The counts above are derived from the mappings and Hibernate's documented eager-collection / lazy-collection behavior, with batching ruled out from config. Treat them as "~", not exact.

---

## Finding 4 — Where Review → DTO conversion happens, relative to the transaction

**Class/method:** `DefaultReviewFacade.getReviews` (`facade/impl/DefaultReviewFacade.java:48-62`) and `DefaultReviewFacade.getMyReviews` (`:64-72`). Both do `pageData.stream().map(review -> modelMapper.map(review, …DTO.class))`, which dispatches to `ReviewConverter` / `OwnerReviewConverter`.

**Inside or outside the transaction:** **Outside** any service/repository transaction.
- `grep` for `Transactional` in `DefaultReviewService`, `DefaultReviewFacade`, and `DefaultReviewTranslationsService` returns **nothing** — none of these classes/methods are `@Transactional`.
- The only transactions on the read path are the implicit per-method transactions that `SimpleJpaRepository` wraps around each repository call. So the page-fetch query commits and its transaction closes **before** the facade's `.map(...)` loop runs. The converter's lazy access (`imageKeys`) and its extra `getReviewTranslation` query therefore execute after the original query's transaction is already gone.

**What keeps the lazy associations loadable at conversion (and serialization) time:** **Open Session In View (OSIV).** There is no `spring.jpa.open-in-view` key in any profile yaml (`grep` across `application-dev/stage/prod.yaml` — no match), so it takes Spring Boot's **default of `true`**. OSIV binds a single `EntityManager`/Hibernate `Session` to the request thread for the whole web request, so:
- `source.getImageKeys()` (LAZY) initializes successfully during the converter / Jackson serialization even though the repository transaction has closed.
- The converter's `getReviewTranslation` call works on its own (it's a fresh repository query, independent of OSIV), but the surrounding lazy navigation that does depend on the open session is `imageKeys`.

If OSIV were disabled, `imageKeys` would throw `LazyInitializationException` at serialization on all three endpoints, because nothing on the read path fetch-joins or otherwise initializes it inside a transaction. (`reviewer`/`preferredLanguage` would survive — they're fetch-joined; `translations` would survive its eager load inside the repository call but is never read anyway.)

---

## Finding 5 — The pagination/collection-fetch TODO

Exact quote (`ReviewRepository.java:31`, immediately above `findByTargetUserIdAndApprovedTrue`):

```
// TODO pageable and join left with collection is not good...
```

**What it warns about, in plain terms:** It flags the tension between `Pageable` and fetch-joining a *collection* association. If one of these queries were "fixed" by adding `join fetch r.translations` (or `r.imageKeys`) to kill the N+1 — the obvious instinct — Hibernate could no longer apply `LIMIT/OFFSET` in SQL, because a collection join multiplies rows (one row per review × per translation), so the database `LIMIT` would slice translation-rows, not reviews. Hibernate's fallback is to pull the **entire** unpaged result set into memory and apply `firstResult/maxResults` **in memory** (the classic `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory` warning). On a large review set that is an unbounded fetch — a memory and latency hazard, and the page's element count can be wrong/duplicated. The TODO is the author noting that the current shape (fetch-join only the to-one `reviewer`/`preferredLanguage`, leave the collections to separate selects) avoids that trap but is itself "not good" because it leaves the N+1 from Finding 3 in place. In short: you can have correct DB pagination **or** a single-query collection fetch, not both via naive `join fetch` — and neither current option is clean.

The same comment applies verbatim to all three listing queries even though it is physically written above only the second one; all three share the structure.

---

## Finding 6 — Hypothetical: `Review.translations` EAGER → LAZY, every consumer

I grepped the whole `src/main` tree for reads of the `Review.translations` entity collection (`getTranslations()` and `.translations`), filtering out the `Product`/`User` translations collections. **Key result: no production consumer navigates `Review.translations` at all.** Every place that needs a review's translation text reads it through a *separate language-filtered query*, not the entity collection:

| Consumer | How it reads translations | Reads the EAGER collection? | After LAZY switch: works? Under what condition? |
| --- | --- | --- | --- |
| `ReviewConverter` (public product/user reviews) `:28` | `reviewTranslationsService.getReviewTranslation(source.getId())` → `findByReviewAndLanguage` | **No** — fresh repo query by `(review, currentLanguage)` | **Works unconditionally.** Independent of the collection fetch type and independent of OSIV for this navigation (it's its own transactional repository call). |
| `OwnerReviewConverter` (my-reviews) `:28` | same `getReviewTranslation(id)` call | **No** | **Works unconditionally**, same reason. |
| `AdminReviewConverter` (admin, out of brief scope) `:25` | same `getReviewTranslation(id)` call | **No** | **Works unconditionally**, same reason. |
| `DefaultReviewTranslationsService.getReviewTranslation` `:29-35` | `reviewTranslationRepository.findByReviewAndLanguage(getReference(Review), getReference(Language))` | **No** — queries the `review_translation` table directly | **Works unconditionally.** |
| `DefaultReviewTranslationsService.generateReviewTranslations` (write path) `:38-69` | builds new `ReviewTranslation` rows and `reviewTranslationRepository.saveAll(...)` | **No** — does not go through the `Review.translations` cascade either | **Works unconditionally.** |
| `TestReviewsImportService:117` | `source.getTranslations()` where `source` is the import DTO `ImportReviewData` (`ImportReviewData:41`), **not** the `Review` entity | **No** (different type) | N/A — not the entity collection. |

**Conclusion for the hypothetical:** Switching `Review.translations` to LAZY would **break nothing** on any current consumer, because nothing reads that collection — the read path deliberately re-queries the single current-language row instead. The switch would simply remove the wasteful eager N+1 (item 3 in Finding 3, ~20 queries/page).

**The OSIV dependency the brief asks me to name:** there is no current consumer that, after the switch, would lazily traverse `Review.translations` and thus depend on `spring.jpa.open-in-view` for that specific collection. So **turning OSIV off would not break any translations reader** — translations are fetched by `findByReviewAndLanguage`, which has its own transaction. (The association that *does* depend on OSIV today, regardless of this hypothetical, is `imageKeys` — see Finding 4. That one would throw `LazyInitializationException` if OSIV were turned off, independent of the translations fetch type.)

One adjacent caveat I can confirm from source but want to flag rather than assert as safe: `Review.translations` carries `cascade = ALL, orphanRemoval = true`. The write path does not rely on this cascade (it saves translations directly), and review deletion uses bulk `DELETE FROM Review` JPQL (`ReviewRepository:74-89`) which bypasses entity-level cascade. So I see no functional break from LAZY. I did **not** exhaustively trace whether any orphan-removal flush relies on the collection being initialized; if a future entity-level `Review` mutation+flush is added, LAZY would change when that collection materializes. That is a hypothetical-future concern, not a current consumer.

---

## Files inspected (read-only)

- `entity/Review.java`, `entity/ReviewTranslation.java`, `entity/User.java` (preferredLanguage + translations region)
- `repository/ReviewRepository.java`, `repository/ReviewTranslationRepository.java`
- `controller/PublicReviewController.java`, `controller/ReviewController.java`
- `facade/ReviewFacade.java`, `facade/impl/DefaultReviewFacade.java`
- `service/impl/DefaultReviewService.java`, `service/impl/DefaultReviewTranslationsService.java`
- `converter/ReviewConverter.java`, `converter/OwnerReviewConverter.java`, `admin/converter/AdminReviewConverter.java`
- `dto/ReviewerInfo.java`
- `MapperConfiguration.java` (converter registration)
- `src/main/resources/application-{dev,stage,prod}.yaml` (OSIV / batch-fetch config absence)

## Tests

- None run. Read-only audit; the brief explicitly waived spotless/test.

## Cleanup performed

- None needed (no code changed).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change required by this audit itself. The N+1 fetch behavior and the EAGER-but-unread `Review.translations` are exactly the subject the parent `jpa-fetch-tuning` feature exists to address, so I am not drafting a standalone issue entry; the findings above are the feature-scoped ground truth for the planned work. If Mastermind wants the "EAGER translations collection is loaded every page and never read" recorded independently of the feature, that draft is in "For Mastermind" below.

## Obsoleted by this session

- Nothing. (Audit only.)

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched, no debug logging, no TODOs added.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session (no translation keys added).
- Other parts touched: Part 11 (trust boundaries) — confirmed N/A: these are read-only listing endpoints; the page query derives `approved = true` and (for my-reviews) the owner id from `currentUserService.getCurrentUserIdStrict()`, not from client input. Part 13 (transactional patterns) — relevant context, not violated.

## Known gaps / TODOs

- Query counts in Finding 3 are reasoned from mappings + Hibernate semantics, not captured from a live SQL log (no local DB run in a read-only audit). Stated as "~".
- I did not open `BaseEntity`; Finding 1 covers only associations declared on `Review` itself, which is what the brief scopes.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Brief vs reality:** No challenge. The brief's six questions all map cleanly onto the code, with one framing nuance worth surfacing: question 6 assumes consumers *read* `Review.translations`; in reality **no consumer navigates that collection** (all read translations via the separate `findByReviewAndLanguage` query). So the EAGER→LAZY switch is functionally free and OSIV is not what protects translations — that's a stronger answer than the question presupposes, not a contradiction of it. Captured fully in Finding 6.

- **Part 4b adjacent observations:**
  1. **Redundant EAGER collection load on every review page** — `Review.translations` is `FetchType.EAGER` (`Review.java:47`) and is loaded once per review on every listing page (~20 selects/page), but **no converter reads it** (they re-query the single current-language row). **Severity: medium** (wasted DB round-trips on hot public endpoints; the exact thing `jpa-fetch-tuning` targets). I did not fix this — out of scope for a read-only audit, and it's the feature's own subject.
  2. **`imageKeys` is silently OSIV-dependent** — on all three endpoints the only thing keeping the response from throwing `LazyInitializationException` is `spring.jpa.open-in-view` defaulting to `true` (no explicit setting anywhere). If a future change disables OSIV (a common hardening move), these endpoints break at serialization unless `imageKeys` is fetched in-tx or mapped inside the transaction. **Severity: medium** (latent; depends on an implicit default). Flagged, not fixed — out of scope.

- **Suggested next step for the feature:** the cheapest high-value change is flipping `Review.translations` to LAZY (removes ~1/3 of the per-page N+1 with zero consumer impact per Finding 6). Killing the remaining N+1 (per-review `getReviewTranslation` and `imageKeys`) needs the collection-vs-pagination care the TODO in Finding 5 describes — e.g. fetch the page of review ids first, then batch-load translations/imageKeys by id set, rather than a naive `join fetch` of a collection alongside `Pageable`.

- Optional `issues.md` draft (only if Mastermind wants it tracked outside the feature): *"`oglasino-backend` — `Review.translations` is EAGER (`Review.java:47`) and loaded on every review-listing page but read by no converter (they re-query the current-language row via `findByReviewAndLanguage`); ~20 redundant selects/page on the public product/user review endpoints. Also: review-listing `imageKeys` serialization depends on `spring.jpa.open-in-view=true` (unset → Spring Boot default); disabling OSIV breaks these endpoints. Both are in scope of the `jpa-fetch-tuning` feature."*
