# Audit: ES read-path verification + mapping/settings (read-only)

**Repo:** `oglasino-backend`
**Branch:** `dev` (not switched)
**Date:** 2026-06-04
**Type:** read-only audit — no code changes, no config changes, no live ES mutations. Local Docker ES (9.0.1, `localhost:9200`, no auth) queried read-only via `curl`.

This is a Phase-2 independent re-verification of a prior exploratory performance audit's claims. Every Part 1 claim is confirmed against raw `grep`/`cat` output (tool-fabrication hazard). Part 2 inspects the ES config layer the prior audit never touched.

---

## Part 1 — Source-fact confirmations

### 1. No source filtering anywhere on ES queries — **CONFIRMED**

```
$ grep -rn "withSourceFilter" src/      → NO MATCH
$ grep -rn "FetchSourceFilter" src/     → NO MATCH
```

Neither `withSourceFilter` nor `FetchSourceFilter` appears anywhere in `src/`. (A broad `grep` for `SourceFilter|sourceFilter|_source|setFetchSource|fetchSource` returned only unrelated hits — catalog `"filter.power_source"` keys and translation rows.) Every ES `search(...)` therefore ships the **full `_source`** of each `ProductDocument` hit.

Query execution path confirmed at `elasticsearch/service/impl/DefaultProductsSearchService.java:45-48`:

```
45:  public SearchHits<ProductDocument> getProductsFor(
...
47:    return elasticsearchOperations.search(
48:        productsFilterQueryBuilder.buildQuery(searchProducts, productsSearchMode),
```

No `.withSourceFilter(...)` on the `NativeQuery.builder()` in `DefaultProductsFilterQueryBuilder` (file read in full, lines 1-174).

### 2. `descriptionTranslations` shipped on every listing hit, never read by the overview converter — **CONFIRMED**

- It **is** a field on `ProductDocument` (so it is in `_source` and shipped ES→app):
  `elasticsearch/documents/ProductDocument.java:30-31`
  ```
  30:  @Field(type = FieldType.Nested, includeInParent = true)
  31:  private List<TranslationReference> descriptionTranslations;
  ```
- The listing/overview converter **does not read it.** `elasticsearch/converters/ProductOverviewConverter.java` (read in full, lines 1-67) reads `getFilterReferences`, `getBaseSite`, `getProductState`, `getModerationState`, `getId`, `getOwnerId`, `getPrice`, `getCurrencyCode`, `isFree`, `getCityLabelKey`, `getNameTranslations` (line 56-58), `getImageKeys` (line 61-62) — **never `getDescriptionTranslations`.**
- It **is** read elsewhere — by the **search query builder**, not the converter: `elasticsearch/generator/impl/TextQueryGenerator.java:94-112` adds a low-boost (`0.2f`) nested `match` on `descriptionTranslations.translation` as a `should` clause.

**Net:** `descriptionTranslations` is shipped in `_source` on every listing hit and discarded by the overview converter, but it is a genuine *search* dependency (see Part 2.1 for why excluding it from a source filter would still be free — search reads the inverted index, not `_source`).

### 3. `track_total_hits(true)` at three sites — **CONFIRMED (exact lines 45, 72, 88)**

```
$ grep -rn "TrackTotalHits|trackTotalHits" src/main/
DefaultProductsFilterQueryBuilder.java:45:        .withTrackTotalHits(true)
DefaultProductsFilterQueryBuilder.java:72:        .withTrackTotalHits(true)
DefaultProductsFilterQueryBuilder.java:88:            .withTrackTotalHits(true);
```

Three sites, all literal `true`, exactly at lines 45 / 72 / 88 — matching the claim. No `withTrackTotalHitsUpTo` anywhere. The three builders:
- `buildSingleProductIdQuery` (line 45)
- `buildMultiProductIdQuery` (line 72)
- `buildQuery` — the listing **and** autocomplete path (line 88)

Each forces an exact total-hit count rather than the 10k-capped default.

### 4. Autocomplete uses the same full `buildQuery` path as listing — **CONFIRMED**

`controller/ProductSearchController.java:52-61` (`POST /api/public/product/search/autocomplete`) calls `productsSearchFacade.getAutocompleteProductsSearch(...)`.

`elasticsearch/facade/impl/DefaultProductsSearchFacade.java:60-72`:
```
61:  public List<SearchProductData> getAutocompleteProductsSearch(
...
65:            productsSearchService
66:                .getProductsFor(searchProducts, productsSearchMode)   // ← same method as listing
```
The full-listing endpoint (`getProductOverviews`, line 52-58) calls the **same** `productsSearchService.getProductsFor(...)`, which routes through `buildQuery` (Part 1.1 evidence above). So autocomplete pays:
- full bool construction (every `QueryGenerator` runs),
- full `_source` (no source filter — Part 1.1),
- exact total (`withTrackTotalHits(true)` — line 88),

…only to map each hit down to **3 fields** — `SearchProductData` has exactly `id`, `name`, `catalogRoute` (`dto/SearchProductData.java`, read in full). Confirmed as claimed.

### 5. Four external clients with no timeout — **ALL FOUR CONFIRMED (no timeout)**

The bounded reference bean (for contrast): `ApplicationConfig.java:58-62`
```
58:  public RestTemplate restTemplate() {
59:    var factory = new SimpleClientHttpRequestFactory();
60:    factory.setConnectTimeout(3000);
61:    factory.setReadTimeout(3000);
62:    return new RestTemplate(factory);
```

- **reCAPTCHA — no timeout; does NOT use the bounded bean.** `service/impl/DefaultReCaptchaService.java:13`
  ```
  13:  private final RestTemplate rest = new RestTemplate();
  ```
  Field-initialized `new RestTemplate()` (default `SimpleClientHttpRequestFactory`, no timeouts), not the `ApplicationConfig.restTemplate()` bean.

- **SMTP — no `connectiontimeout` / `timeout` / `writetimeout` in any of the three env yamls — CONFIRMED.** All three `mail.properties.mail.smtp.*` blocks contain only `auth: true` + `starttls`:
  - `application-prod.yaml:84-90` (`smtp: auth/starttls` only)
  - `application-stage.yaml:87-91` (same)
  - `application-dev.yaml:70-76` (same)
  A `grep -ni "smtp|connectiontimeout|writetimeout|mail\."` across all three returned only host/port/username/password/auth/starttls — no timeout keys. (Spring's JavaMail default for these is infinite.)

- **OpenAI — no timeout — CONFIRMED.** `openai/service/impl/DefaultOpenAIService.java:42`
  ```
  42:    try (CloseableHttpClient client = HttpClients.createDefault()) {
  ```
  `HttpClients.createDefault()` — no `RequestConfig` connect/socket/response timeout applied (no `RequestConfig`/`setTimeout` anywhere in the file).

- **R2/S3 — no `apiCallTimeout` / `socketTimeout` / `connectionTimeout` override — CONFIRMED.** `config/R2Config.java` (read in full):
  ```
  .httpClient(UrlConnectionHttpClient.builder().build())
  ```
  `UrlConnectionHttpClient.builder().build()` with no `.socketTimeout(...)`/`.connectionTimeout(...)`, and the `S3Client.builder()` has no `.overrideConfiguration(... apiCallTimeout/apiCallAttemptTimeout ...)`. SDK defaults apply (no app-level bound).

> Adjacent (already in `issues.md`, not re-litigated here): a **fifth** no-timeout client exists — `DefaultCloudflareKvService.java:23` `new RestTemplate()` — logged 2026-06-03. Not in this brief's list of four; noting it is already tracked.

---

## Part 2 — ES mapping + settings findings (live local ES, read-only)

Index resolution: `@Document(indexName = "products")` on `ProductDocument.java:15`. Live, `products` is an **alias** onto a timestamped concrete index created by the rolling indexer:

```
$ curl -s "http://localhost:9200/products/_alias?pretty"
{ "products_20260604073907" : { "aliases" : { "products" : { } } } }
```

### 2.1 `GET products/_mapping` — `descriptionTranslations` mapping (the decision-critical fact)

`descriptionTranslations` — **`nested`, with `translation` mapped as `search_as_you_type` (analyzed/searchable), shipped in `_source`**. Verbatim:

```json
"descriptionTranslations" : {
  "type" : "nested",
  "include_in_parent" : true,
  "properties" : {
    "_class" : { "type" : "keyword", "index" : false, "doc_values" : false },
    "langCode" : { "type" : "keyword" },
    "translation" : {
      "type" : "search_as_you_type",
      "doc_values" : false,
      "max_shingle_size" : 3,
      "analyzer" : "name_autocomplete_index",
      "search_analyzer" : "name_autocomplete_search"
    }
  }
}
```

`nameTranslations` — **identical structure** to `descriptionTranslations` (same `search_as_you_type` + `name_autocomplete_*` analyzers):

```json
"nameTranslations" : {
  "type" : "nested",
  "include_in_parent" : true,
  "properties" : {
    "_class" : { "type" : "keyword", "index" : false, "doc_values" : false },
    "langCode" : { "type" : "keyword" },
    "translation" : {
      "type" : "search_as_you_type",
      "doc_values" : false,
      "max_shingle_size" : 3,
      "analyzer" : "name_autocomplete_index",
      "search_analyzer" : "name_autocomplete_search"
    }
  }
}
```

`filterReferences` (for context) — **doubly-nested** (`filter` and `options` are each `nested` inside the `nested` parent), all `include_in_parent: true`:

```json
"filterReferences" : {
  "type" : "nested",
  "include_in_parent" : true,
  "properties" : {
    "_class" : { "type" : "keyword", "index" : false, "doc_values" : false },
    "filter" : {
      "type" : "nested", "include_in_parent" : true,
      "properties" : {
        "_class" : { "type": "keyword", "index": false, "doc_values": false },
        "filterType" : { "type" : "keyword" },
        "iconId" : { "type" : "keyword" },
        "id" : { "type" : "long" },
        "key" : { "type" : "keyword" },
        "labelKey" : { "type" : "keyword" },
        "rangePrefix" : { "type" : "keyword" },
        "rangeSuffix" : { "type" : "keyword" }
      }
    },
    "options" : {
      "type" : "nested", "include_in_parent" : true,
      "properties" : {
        "_class" : { "type": "keyword", "index": false, "doc_values": false },
        "id" : { "type" : "long" },
        "labelKey" : { "type" : "keyword" }
      }
    },
    "selectedRangeValue" : { "type" : "long" }
  }
}
```

**Decision answer — is excluding `descriptionTranslations` from a source filter free, or does it break a search dependency?**
It is **free for search.** Search on `descriptionTranslations` goes through a nested query (`TextQueryGenerator.java:94-112`, `QueryBuilders.nested().path("descriptionTranslations")`) which reads the **inverted index / `search_as_you_type` subfields**, not `_source`. A `_source` filter changes only what is *returned and deserialized into `ProductDocument`* — it does not touch queryability. Since the overview converter never reads `getDescriptionTranslations()` (Part 1.2), a source filter that includes everything **except** `descriptionTranslations` would drop the unused payload from listing/autocomplete hits with **zero** search impact. (The detail path maps to `ProductDetailsDTO`; confirm that DTO also doesn't read description before applying a filter to the *single-id* queries — the listing/autocomplete builders are the safe targets. This is a fix-shape note for a future write-brief; **no change made here.**)

### 2.2 `GET products/_settings`

- `number_of_shards`: **1**
- `number_of_replicas`: **1**
- `refresh_interval`: **1s** (default — not overridden; only visible with `include_defaults=true`)
- `max_result_window`: **not overridden → 10000 default** (confirmed via `include_defaults=true`) — matches the claim.

Custom analyzers present in `index.analysis` (from `@Setting(settingPath="es-config/elastic-analyzer.json")`): `name_autocomplete_index/_search`, `description_autocomplete_index/_search`, filters `name_autocomplete_filter` (edge_ngram 1–20), `description_autocomplete_filter` (edge_ngram 3–20), tokenizer `description_autocomplete_tokenizer`. **See 2.5 finding #2 — the `description_autocomplete_*` analyzers + tokenizer are defined but unused by any field.**

### 2.3 `GET _cat/indices?v` and `GET _cat/shards/products?v`

```
health status index                    pri rep docs.count store.size
yellow open   products_20260604073907  1   1   1083       172.9mb
```
```
index                   shard prirep state      docs   store
products_20260604073907 0     p      STARTED    1083 172.9mb
products_20260604073907 0     r      UNASSIGNED
```

- **Index count:** 1 product index (`products_20260604073907`, aliased to `products`); the remaining ~12 indices are Kibana/Elastic internal `.internal.alerts-*` system indices (0 docs each), not application data.
- **Doc count — NOT empty locally:** **1083 docs, 172.9 MB** in the product index. This contradicts the brief's framing ("indices are empty everywhere pre-launch"). This is **local seed data**, not production — it does not change the profiling-deferral conclusion (1083 seeded local docs are not representative of production scale or traffic), but the empty-index assumption is false on this machine and is the reason a `store.size` number is even available. **172.9 MB / 1083 docs ≈ 163 KB/doc** is notably heavy for documents this shape — see 2.5 findings #1 and #3 for the likely drivers.
- **Shard topology:** 1 primary (`STARTED`), 1 replica (`UNASSIGNED`) → index health **yellow** on this single-node cluster (replica can't allocate). See 2.5 finding #4.

### 2.4 Slow-log state — **OFF (not configured)**

`index.search.slowlog.threshold.*` is **not set** on the product index. With `include_defaults=true` the `slowlog` block carries only the default `include.user: false` scaffold — **no thresholds**, so the search slow-log emits nothing. Same for `index.indexing.slowlog`. **Not enabled (and not enabled by me — read-only).** Implication for the deferred post-launch profiling: there is **no slow-log to lean on**; profiling will need explicit `profile: true` runs (or the slow-log thresholds set first, which is a write — out of scope here).

### 2.5 Part 4b — adjacent ES-layer observations

1. **`search_as_you_type` on `descriptionTranslations.translation` is over-specified for how description is queried — `medium`.**
   `TextQueryGenerator.java:107` queries description with a plain `.match()` at boost `0.2f` — **not** `matchBoolPrefix` and **not** any of the `._2gram/._3gram/._index_prefix` subfields that `search_as_you_type` exists to provide. So the description field pays the full `search_as_you_type` indexing cost (shingle subfields + edge-prefix subfield, `max_shingle_size: 3`) while the query uses only the root analyzed field. A plain `text` field would serve the existing query identically at a fraction of the index footprint. Likely a primary driver of the 163 KB/doc size. `reference/TranslationReference.java:9-13` (shared by name + description). *Out of scope — read-only audit; flagged for a write-brief.*

2. **`description_autocomplete_index` / `description_autocomplete_search` analyzers + `description_autocomplete_tokenizer` are defined but unused — `low`.**
   `TranslationReference.translation` (used by **both** name and description) is hardcoded to `analyzer = "name_autocomplete_index"`, `searchAnalyzer = "name_autocomplete_search"` (`reference/TranslationReference.java:11-12`). The `description_autocomplete_*` analyzers in `es-config/elastic-analyzer.json` are dead config. This is also a correctness smell: descriptions are intended (by the existence of `description_autocomplete_filter`, edge_ngram **min 3**) to ngram differently from names (**min 1**), but in reality descriptions are analyzed with the *name* analyzer (min_gram 1). Misleading to a future reader. *Out of scope; flagged.*

3. **`include_in_parent: true` on every nested field appears unused by any query — `medium`.**
   All translation/filter searching goes through `QueryBuilders.nested().path(...)` (`TextQueryGenerator` for name/description; `SpecFilterQueryGenerator.java:50-72,93+` for `filterReferences`). I found **no** query that reads the flattened (parent-promoted) copies. `include_in_parent: true` therefore duplicates every nested field's terms into the parent document for no query benefit — pure index bloat (the converters read `_source`, which is unaffected by `include_in_parent`). Combined with #1, the most likely explanation for 163 KB/doc. Verifying there is truly no flattened-field consumer (e.g. a sort, an aggregation, a non-nested term) is a prerequisite before removing it. *Out of scope; flagged.*

4. **`number_of_replicas: 1` yields a permanently-yellow index on a single-node cluster — `low/medium`.**
   Locally the replica is `UNASSIGNED` (single node) → yellow forever, one wasted unallocated shard slot. Harmless locally, but if the same index settings ship to a single-node production ES, the prod product index will sit yellow indefinitely with an unallocatable replica (and any health gate that treats yellow as degraded would misfire). Confirm prod ES node count; if single-node, set `number_of_replicas: 0` for that environment. *Out of scope; flagged for prod sizing.*

5. **Rolling index + alias topology (informational, not a defect).**
   `ProductDocument` targets the `products` **alias**, fronting a timestamped concrete index (`products_20260604073907`) — `ProductIndexer` rebuilds into a fresh timestamped index and repoints the alias (blue/green reindex). This is sound; noting it so a future reader doesn't expect a literal `products` index. The `applyRandomIfNeeded` `function_score` random ordering (`DefaultProductsFilterQueryBuilder.java:124-147`) uses `random_score(seed, field:"id")` — deterministic per seed, gated off when search text or an explicit `orderBy` is present; nothing anomalous.

---

## Deferral statement (verbatim, on record)

**Absolute ES query latency is not measurable pre-launch (empty indices); query profiling deferred to a post-launch brief once production listings accumulate.**

(Caveat for accuracy: the *local* product index is **not** empty — it holds 1083 seed docs, see 2.3 — but 1083 seeded local documents are not representative of production scale or concurrent traffic, so the deferral stands: meaningful latency profiling waits for production listings.)
