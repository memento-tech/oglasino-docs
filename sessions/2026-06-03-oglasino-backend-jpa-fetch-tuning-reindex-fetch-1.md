# Audit — JPA fetch tuning: reindex product hydrate

**Repo:** oglasino-backend
**Branch:** dev (read-only; no checkout/commit/push)
**Date:** 2026-06-03
**Task:** Read-only audit of how products are loaded from the database during an Elasticsearch reindex, focused on how many collections are fetched per query and what that costs.

Scope was READ-ONLY. No code changed. No spotless/test run. Every claim below is cited to `file:line` and cross-checked with grep/cat per the repo's tool-fabrication hazard note.

---

## 1. Reindex flow and the repository methods

**Two distinct triggers, one flow:**

- **Admin-triggered, async.** `EsIndexerController.startReindex()` (`admin/controller/EsIndexerController.java:41-52`) is `POST /api/secure/admin/es/reindex`, `@PreAuthorize("hasRole('ADMIN')")`. It reserves a job id, returns `202`, and calls `productIndexer.reindexAllAsync(jobId)` (`:50`).
- **Boot-time, synchronous, non-prod only.** `GlobalIndexerService.onAppReady()` (`elasticsearch/service/impl/GlobalIndexerService.java:26-36`) is `@Profile({"!prod"})`, fires on `ApplicationReadyEvent`, and calls `indexer.reindexAll()` for every `Indexer` bean. So in dev/stage the full reindex runs on every boot; in prod it only runs when an admin triggers it.

Both converge on `ProductIndexer.doReindex(...)` (`elasticsearch/service/impl/ProductIndexer.java:129-191`). `reindexAll()` (`:102-106`) and `reindexAllAsync(...)` (`:113-122`) are both `@Transactional(readOnly = true)`; the async one is also `@Async`.

**Two-stage paginated read** (confirmed `:141-176`):

- **Stage A — page IDs only.** `productRepository.findIdsForIndexing(PageRequest.of(page, batchSize))` (`:142`). The query is `SELECT p.id FROM Product p ORDER BY p.id` (`ProductRepository.java:64-65`) — no collection fetches, so pagination is clean and stable-ordered by id.
- **Stage B — hydrate one page of IDs.** `productRepository.findForIndexingByIds(idsPage.getContent())` (`:157`) loads the heavy fetch graph for that page's IDs (`ProductRepository.java:72-91`).

Per page: map each `Product` → `ProductDocument` via `modelMapper` in `toIndexQuery` (`:336-339`), then one `elasticsearchOperations.bulkIndex(...)` (`:159`). Loop advances on `idsPage.hasNext()` (`:172-175`).

So: **two query stages** — page IDs, then hydrate-by-IDs — repeated per page. Not one query.

---

## 2. The hydrate query and its row multiplication

`findForIndexingByIds` (`ProductRepository.java:72-91`) is `SELECT DISTINCT p ... WHERE p.id IN :ids` with these `LEFT JOIN FETCH`es. Collection type taken from `entity/Product.java`:

| JOIN FETCH | Target | Cardinality | Collection type (Product.java) |
|---|---|---|---|
| `p.translations t` | `ProductTranslation` | **collection** | `Set<ProductTranslation>`, `@OneToMany` (`:64-69`) → PersistentSet |
| `t.language` | `Language` | to-one | — |
| `p.filterValues fv` | `ProductFilterValue` | **collection** | `List<ProductFilterValue>`, `@OneToMany` + `@OrderBy("id ASC")` (`:71-77`) → PersistentBag |
| `fv.filter` | `Filter` | to-one | — |
| `p.city` | `City` | to-one (`@ManyToOne` `:50-52`) | — |
| `p.region` | `Region` | to-one (`:46-48`) | — |
| `p.category` | `Category` | to-one (`:42-44`) | — |
| `p.subCategory` | `Category` | to-one (`:38-40`) | — |
| `p.topCategory` | `Category` | to-one (`:34-36`) | — |
| `p.currency` | `Currency` | to-one (`:54-56`) | — |
| `p.owner` | `User` | to-one (`:26-28`) | — |
| `p.imageKeys` | `String` | **collection** | `Set<String>`, `@ElementCollection` (`:85-88`) → PersistentSet |
| `p.baseSite` | `BaseSite` | to-one (`:30-32`) | — |

**Three collections are fetched simultaneously:** `translations` (Set), `filterValues` (List/bag), `imageKeys` (element-collection Set). The other ten fetches are all to-one and do not multiply rows — each contributes exactly one column-set per row.

**Row multiplication for one product with N translations, M filter values, K images:**

```
rows materialized = N × M × K
```

This is a three-way cartesian product across the three joined collections, hung off the single root row. (`t.language` and `fv.filter` are to-one, so they ride along inside each row and add no factor.) Because these are `LEFT JOIN`s, an empty collection contributes a single null-bearing row rather than zero — i.e. the effective factor is `max(N,1) × max(M,1) × max(K,1)`. Worked example: a product with 3 translations, 4 filter values, 6 images materializes **3 × 4 × 6 = 72** JDBC rows for that one product; the bulk page (`batchSize = 500`) is the sum of each product's own N×M×K.

`SELECT DISTINCT` does **not** shrink that row count at the result-set level: every combination row differs in at least one collection-element column (translation id, filter-value id, image key), so no two rows are byte-identical for DISTINCT to collapse. DISTINCT / Hibernate's entity de-duplication only collapses the materialized rows back into one managed `Product` graph **in memory**, after the DB has already produced and transferred all N×M×K rows. (I did not assert whether the `DISTINCT` keyword is passed through to the SQL on this Hibernate version — that is a Hibernate-6 runtime behavior, not visible in source. Either way the JDBC result set carries N×M×K rows, because the distinct combinations are genuinely distinct rows.)

---

## 3. Is more than one bag fetched? Is `MultipleBagFetchException` possible today?

**Bag inventory among the three fetched collections:**

- `translations` — `Set` → PersistentSet, **not a bag**.
- `imageKeys` — `Set` (`@ElementCollection`) → PersistentSet, **not a bag**.
- `filterValues` — `List` with `@OrderBy("id ASC")` and **no `@OrderColumn`** (`Product.java:71-77`) → PersistentBag. `@OrderBy` only appends an SQL `ORDER BY`; it does not introduce an index column, so Hibernate still classifies this as a bag.

**Exactly one bag is fetched.** `MultipleBagFetchException` is raised only when **two or more bags** are join-fetched in the same query. Fetching one bag alongside any number of Sets is allowed.

**Conclusion: `MultipleBagFetchException` is not possible today** — there is only one List/bag in the fetch graph. The constraint that keeps it impossible is that `translations` and `imageKeys` are `Set`s. If either were ever changed to a `List` (without an `@OrderColumn`), the query would then fetch two bags and would throw at query-compile time. Worth preserving as an invariant if this graph is edited.

---

## 4. Request path or batch job? Batch/page size?

**Batch job, never the request path.** It is invoked only by (a) the admin maintenance endpoint asynchronously on the `taskExecutor`/virtual-thread pool (`@Async`, `ProductIndexer.java:113`; async machinery in `config/AsyncConfig.java`), returning `202` immediately, and (b) the non-prod boot listener (`GlobalIndexerService`). No search-serving or user request path calls `findForIndexingByIds` / `doReindex` (grep §5 confirms the only caller of the hydrate method is `ProductIndexer:157`).

**Batch/page size = 500**, configurable. `elasticsearchProperties.getReindex().getBatchSize()` (`ProductIndexer.java:136`) bound from `app.elasticsearch.reindex.batch-size`; default `500` (`properties/ElasticsearchProperties.java:35`). The same page size bounds both the Stage-B `IN` clause / DB heap and the ES bulk payload (`:159`).

---

## 5. Separate non-paginated full-graph load method — exists? called?

**Yes, it exists:** `findAllForIndexing()` (`ProductRepository.java:35-53`) — a `SELECT DISTINCT p ...` with the **identical** thirteen-join fetch graph as `findForIndexingByIds`, but **no pagination** and no `WHERE` (loads the whole catalog's full graph in one query).

**It is not called anywhere in `src/main` or tests.** grep result:

```
$ grep -rn "findAllForIndexing" src
src/main/java/.../repository/ProductRepository.java:53:  List<Product> findAllForIndexing();          # declaration
src/main/java/.../repository/ProductRepository.java:70:   * Mirrors the fetch graph of {@link #findAllForIndexing()} ...   # javadoc xref
```

Only the declaration and a javadoc cross-reference from `findForIndexingByIds`. **No invocation in main or test code** — it is dead, unreferenced repository code. (Flagged in "For Mastermind" as a Part 4b adjacent observation; not removed, audit is read-only.) Its existence is also a live `MultipleBagFetchException`/cartesian risk if anyone wires it up against a large catalog — same single-bag graph, but full-catalog cartesian with no batch ceiling.

---

## 6. Hypothetical — fetching fewer collections per query and keeping documents complete

The document is built by `modelMapper.map(product, ProductDocument.class)` in `toIndexQuery` (`ProductIndexer.java:337`). `ProductDocument` consumes the collections — `nameTranslations` / `descriptionTranslations` (from `translations`), `imageKeys` (from `imageKeys`), and the product-filter references (from `filterValues`) (`elasticsearch/documents/ProductDocument.java`). So for a document to stay complete, **every collection the mapper reads must be initialized on the managed entity before/while the mapper runs.**

**What initializes collections during indexing today:**

- **Batch path (`doReindex` → `toIndexQuery`): nothing explicit.** There is no `Hibernate.initialize(...)` in the batch path. Completeness rests entirely on the `JOIN FETCH` graph of `findForIndexingByIds`. The mapper runs *inside* the `@Transactional(readOnly = true)` method (`:103-106`/`:114`), so the persistence context is still open during mapping.
- **`indexOne(Long)` path (single-product, event-driven): explicit init.** `indexOne` (`:343-382`, `@Transactional`) does `findById` then an explicit block of `Hibernate.initialize(...)` calls (`:356-363`) for `translations`, `baseSite`, `owner`, `topCategory`, `subCategory`, `category`, `filterValues`, `imageKeys` before mapping. (Note it does **not** explicitly initialize `region`, `city`, `currency`, or `t.language` — those lazy-load on mapper access inside the open transaction.)

**What would have to stay true** if the hydrate query fetched fewer collections per query, for reindexed documents to remain complete:

1. **Same open persistence context at mapping time.** Because mapping already runs inside the open read-only transaction, any collection *removed* from the `JOIN FETCH` would not break completeness outright — the mapper's access would trigger a lazy load instead of a `LazyInitializationException`. The cost is **N+1**: one extra SELECT per un-fetched collection per product (per page, multiplied by batch size). That trades the N×M×K cartesian blow-up for round-trip count.
2. **Or split into multiple fetch queries against the same managed entities.** The blessed way to drop the cartesian without N+1 is to keep at most one bag (and any Sets) per query and issue a second query, for the IDs in the same page and within the same persistence context, that join-fetches the remaining collection(s). Hibernate merges the second query's rows into the already-managed `Product` instances (same first-level cache), so the entity graph ends up complete. (This is the canonical "one collection per query, reuse the session" remedy; not currently implemented here.)
3. **Or explicit `Hibernate.initialize(...)` per un-fetched collection**, mirroring `indexOne` — same N+1 cost as (1) but with the initialization made explicit at the call site rather than implicit on mapper access.

The single line that must "see" initialized collections is `modelMapper.map(product, ProductDocument.class)` (`:337`). Whatever scheme replaces the wide fetch must guarantee that line reads fully-populated collections, either by pre-fetching them in a bounded number of queries against the same context or by accepting bounded N+1 within the still-open transaction.

**Adjacent (out-of-strict-scope) note for completeness:** the hydrate query fetches `fv.filter` (to-one) but does **not** fetch the nested `ProductFilterValue.filterOptions` collection (`Set<FilterOption>`, `entity/ProductFilterValue.java:30-34`). If the mapper reads filter options, that is already a per-filter-value N+1 lazy load inside the open transaction today — independent of any fetch-reduction change. Confirm whether `ProductDocument`'s filter references consume `filterOptions` before relying on this.

---

## Files inspected (read-only)

- `elasticsearch/service/impl/ProductIndexer.java` — reindex flow, two-stage loop, `indexOne`
- `elasticsearch/service/impl/GlobalIndexerService.java` — non-prod boot trigger
- `admin/controller/EsIndexerController.java` — admin async trigger
- `repository/ProductRepository.java` — `findIdsForIndexing`, `findForIndexingByIds`, `findAllForIndexing`
- `entity/Product.java` — collection types and cardinalities
- `entity/ProductFilterValue.java` — nested `filter` / `filterOptions`
- `properties/ElasticsearchProperties.java` — batch size (500)
- `elasticsearch/documents/ProductDocument.java` — mapping target / consumed collections

## Tests

- None run (read-only audit; no code changed). Not required by the brief.

## Cleanup performed

- None needed (read-only).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change required by this session. One dead-code finding is flagged below for Mastermind to triage into issues.md if desired — drafted there, not written here.

## Obsoleted by this session

- Nothing. (Audit only; `findAllForIndexing()` is identified as dead but not removed — read-only scope.)

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched.
- Part 4a (simplicity): N/A — no code added; evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): N/A — internal batch read path, no client-supplied trust-bearing input.
- Other parts touched: Part 10 (feature lifecycle) — this is a Phase-2 read-only audit, output to `.agent/audit-<slug>.md` as specified.

## Known gaps / TODOs

- Did not assert the Hibernate-6 runtime behavior of `DISTINCT` pass-through to SQL (not visible in source). The N×M×K JDBC-row claim holds regardless; the only thing version-dependent is whether the `DISTINCT` keyword reaches the SQL, which does not change the row count for genuinely-distinct combination rows.
- Did not confirm whether `ProductDocument`'s filter references read `ProductFilterValue.filterOptions` (the un-fetched nested collection). Noted as a verification item in §6.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Adjacent observation (Part 4b) — dead repository method.** `ProductRepository.findAllForIndexing()` (`repository/ProductRepository.java:35-53`) has zero callers in `src/main` or tests (grep in §5). Severity: low-medium. Low as pure dead code; medium because it is a full-catalog, non-paginated, single-bag cartesian fetch that would risk heap/`OOM` (and re-introduce the in-memory-pagination degrade the two-stage split was built to avoid) if a future engineer wired it up unaware. I did not remove it — read-only scope. Suggest either deleting it or, if kept as the documented "mirror," annotating it as deliberately-unused. Draft for issues.md if you want it tracked: *"oglasino-backend — `ProductRepository.findAllForIndexing()` is dead (no caller in main/tests); a non-paginated full-graph cartesian fetch that is an OOM footgun if revived. Delete or mark deliberately-unused."*
- **Adjacent observation (Part 4b) — un-fetched nested collection in hydrate.** The hydrate query fetches `fv.filter` but not `ProductFilterValue.filterOptions` (`entity/ProductFilterValue.java:30-34`). If `ProductDocument` consumes filter options, the reindex already does a per-filter-value N+1 inside the open transaction today (independent of any tuning change). Severity: low pending confirmation it is read. I did not trace the mapper internals — flagged for the tuning brief to verify. (See §6 adjacent note.)
- **Bag invariant to preserve if this graph is edited.** Today `MultipleBagFetchException` is impossible only because `translations` and `imageKeys` are `Set`s and `filterValues` is the sole bag (§3). Any future change of `translations`/`imageKeys` to a `List` (without `@OrderColumn`) would make this query throw. Worth stating in the tuning spec as a constraint.
</content>
</invoke>
