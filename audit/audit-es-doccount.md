# Audit: ES product doc-count discrepancy (read-only)

**Repo:** `oglasino-backend`
**Branch:** `dev` (not switched)
**Date:** 2026-06-04
**Type:** read-only investigation — no code changes, no ES mutations, no reindex, no DB writes. Local Docker ES (`localhost:9200`, no auth) and local Docker Postgres queried read-only.

---

## Verdict (one line)

**There is no product-count discrepancy. The index holds exactly 47 products = 47 root documents. The "1083" is the Lucene-level `docs.count`, which counts every *nested* object as its own hidden Lucene document; the 47 products contain 1036 nested children that sum, with the 47 roots, to exactly 1083.** The prior audit (`audit-es-performance.md`) misread `_cat/indices` `docs.count` (a Lucene segment count) as a product/document count. Nothing is stale, duplicated, or seed-polluted — every doc is a real product or a nested part of one.

The exact accounting:

```
  47  root product documents
+ 188  nameTranslations         nested docs
+ 188  descriptionTranslations  nested docs
+  47  baseSite                 nested docs
+ 205  filterReferences         nested docs (level 1)
+ 205  filterReferences.filter  nested docs (level 2, recursive)
+ 203  filterReferences.options nested docs (level 2, recursive)
= 1083  total Lucene docs   ✓ exact match to docs.count
```

---

## Step 1 — Count vs. expectation

**`_count` (Elasticsearch document count) = 47** — matches Igor's 47 products exactly.

```
$ curl -s 'localhost:9200/products/_count?pretty'
{
  "count" : 47,
  "_shards" : { "total" : 1, "successful" : 1, "skipped" : 0, "failed" : 0 }
}
```

**Exactly one concrete index, aliased once.** `products` is an alias → `products_20260604073907`, and that is the *only* `products_*` concrete index (no orphans from failed reindex jobs):

```
$ curl -s 'localhost:9200/_cat/aliases?v'   (products row only)
alias     index                    filter routing.index routing.search is_write_index
products  products_20260604073907  -      -             -              -

$ curl -s 'localhost:9200/_cat/indices/products*?v'
health status index                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   products_20260604073907   1   1       1083            0    172.9mb        172.9mb
```

Multiple concrete indices are **not** the cause — there is only one. Note `docs.deleted = 0`, so the gap is not tombstones either.

The `docs.count` of **1083** here is Lucene's view (root + nested + all hidden docs). The `_count` API above reports only the 47 root documents. Same index, two different counters — this is the entire discrepancy. `_stats/docs` agrees with `_cat` (and gives the byte figure):

```
$ curl -s 'localhost:9200/products/_stats/docs?pretty'   (primaries)
"docs": { "count": 1083, "deleted": 0, "total_size_in_bytes": 181301943 }
```

181,301,943 bytes = 172.9 MB. (See Step "163 KB/doc" reassessment at the end.)

---

## Step 2 — One-product-many-docs, or many distinct products?

### 2a. The indexer writes exactly one document per product

`ProductIndexer` indexes **one `ProductDocument` per product row, keyed by the product id** — no fan-out per locale, per base-site, or per anything. The doc id is the product id, so re-indexing is an idempotent upsert, never an accumulation.

Bulk reindex path — `ProductIndexer.java:163` maps each product to one query, `:341-344` keys it by product id:

```
163:      List<IndexQuery> queries = products.stream().map(p -> toIndexQuery(p)).toList();
...
341:  private IndexQuery toIndexQuery(Product product) {
342:    ProductDocument doc = modelMapper.map(product, ProductDocument.class);
343:    return new IndexQueryBuilder().withId(String.valueOf(doc.getId())).withObject(doc).build();
344:  }
```

Single-product path — `ProductIndexer.java:373-377`, same id keying:

```
373:    var query =
374:        new IndexQueryBuilder()
375:            .withId(String.valueOf(productDoc.getId()))
376:            .withObject(productDoc)
377:            .build();
```

The reindex builds a brand-new physical index from Postgres and atomically swaps the alias (`doReindex`, `ProductIndexer.java:129-196`), so even repeated reindexes cannot accumulate docs — each run starts from an empty index and ends with exactly `count(product)` root docs.

### 2b. The fan-out is *nested documents*, not extra products

`ProductDocument` declares **four `@Field(type = FieldType.Nested)` fields** — `ProductDocument.java:27-31, 48-49, 84-85`:

```
27:  @Field(type = FieldType.Nested, includeInParent = true)
28:  private List<TranslationReference> nameTranslations;
30:  @Field(type = FieldType.Nested, includeInParent = true)
31:  private List<TranslationReference> descriptionTranslations;
48:  @Field(type = FieldType.Nested, includeInParent = true)
49:  private BaseSiteReference baseSite;
84:  @Field(type = FieldType.Nested, includeInParent = true)
85:  private List<ProductFilterReference> filterReferences;
```

And `filterReferences` nests **recursively** — `ProductFilterReference.java:8-12` puts two more nested objects inside each filter reference:

```
8:  @Field(type = FieldType.Nested, includeInParent = true)
9:  private FilterReference filter;
11:  @Field(type = FieldType.Nested, includeInParent = true)
12:  private List<FilterOptionReference> options;
```

In Lucene, **every nested object is indexed as its own hidden document** sharing the root's block. So one product produces 1 root doc plus one hidden doc per name translation, per description translation, per baseSite, per filter reference, per filter-reference's `filter`, and per filter-reference's `option`. That is the 23× inflation.

### 2c. Distinct product ids = 47 (not 1083)

`size:0` with a `cardinality` agg on the product `id` confirms there are **47 distinct product ids**, and `hits.total` is **47**, not 1083 — the search layer sees 47 documents:

```
$ curl -s 'localhost:9200/products/_search' -d '{"size":0,"aggs":{"distinct_ids":{"cardinality":{"field":"id"}}}}'
hits.total = 47
distinct id cardinality = 47
```

A sample of 5 docs shows real products with multi-language name translations — not test/seed junk (full detail in Step 4):

```
_id 8592  ACTIVE  "Like-new children's leather sneakers ... size 24"   (cnr/sr/ru/en)
_id 8577  ACTIVE  "NVIDIA GeForce RTX 3070 8GB ... 4K Video Editing"   (cnr/ru/en/sr)
_id 8557  ACTIVE  "Bosch GST 90 E Jigsaw _ Excellent Condition!"        (ru/sr/cnr/en)
_id 8579  ACTIVE  "Cargo bike for delivery ..."                         (en/sr/...)
```

### 2d. Nested-doc accounting — the 1036 nested docs, by type

`size:0` nested aggregations on each path give the exact per-type counts:

```
$ curl -s 'localhost:9200/products/_search' -d '{"size":0,"aggs":{
    "names":{"nested":{"path":"nameTranslations"}},
    "descs":{"nested":{"path":"descriptionTranslations"}},
    "bs":{"nested":{"path":"baseSite"}},
    "fref":{"nested":{"path":"filterReferences"}},
    "ffilter":{"nested":{"path":"filterReferences.filter"}},
    "foptions":{"nested":{"path":"filterReferences.options"}}}}'

  "names":   {"doc_count":188},
  "descs":   {"doc_count":188},
  "bs":      {"doc_count":47},
  "fref":    {"doc_count":205},
  "ffilter": {"doc_count":205},
  "foptions":{"doc_count":203}
```

Sum: `47 + 188 + 188 + 47 + 205 + 205 + 203 = 1083`. Exact. Every one of the 1083 Lucene docs is now attributed. (≈4 name + 4 description translations per product, 1 baseSite each, ≈4.4 filter references each, each filter ref carrying 1 filter + ≈1 option.)

---

## Step 3 — Cross-check against Postgres

The DB holds **exactly 47 product rows**, with a state breakdown of 44 ACTIVE / 2 DELETED / 1 INACTIVE (read-only `psql` via the local `oglasino-local-postgres-1` container; the host has no `psql` binary):

```
$ docker exec oglasino-local-postgres-1 psql -U oglasino_user -d oglasino -tAc "SELECT count(*) FROM product;"
47

$ ... "SELECT product_state, count(*) FROM product GROUP BY product_state ORDER BY 2 DESC;"
ACTIVE|44
DELETED|2
INACTIVE|1

$ ... "SELECT moderation_state, count(*) FROM product GROUP BY moderation_state ORDER BY 2 DESC;"
APPROVED|47
```

So "47" is the true product-row count — the DB does **not** secretly hold ~1083 rows. The ES index mirrors the DB row-for-row (see Step 4 — including the 2 DELETED and 1 INACTIVE rows).

---

## Step 4 — Where they came from: staleness vs. seed pollution

**Not stale, not seed pollution. The docs are the live 47 products, and they mirror the DB exactly.**

ES `productState` breakdown is identical to the DB breakdown — 44/2/1:

```
$ curl -s 'localhost:9200/products/_search' -d '{"size":0,"aggs":{"by_state":{"terms":{"field":"productState"}}}}'
ACTIVE 44
DELETED 2
INACTIVE 1
```

The sampled docs are unmistakably real listings (an RTX 3070 GPU, a Bosch GST 90 E jigsaw, children's leather sneakers, a cargo bike), each `moderationState: APPROVED`, with genuine `cnr/sr/ru/en` translations and real `baseSiteId` values (1000, 2000) — none of the `Test123`-style placeholder shape that catalog-gen or the test endpoints produce.

**Every write path into the index is id-keyed and bounded** — there is no boot-time or seed path that could inflate the count:

```
$ grep -rn --include="*.java" "reindexAll|.indexOne(|reindexAllAsync" src/main/java   (callers)
admin/controller/EsIndexerController.java:50    productIndexer.reindexAllAsync(initial.jobId());   # admin-triggered full rebuild
listeners/ProductIndexerEventListener.java:31   productIndexer.indexOne(event.productId());        # per-product on create/update
elasticsearch/service/impl/GlobalIndexerService.java:31  controller.reindexAll();                  # boot-time, @Profile("!prod")
```

- `EsIndexerController` → `reindexAllAsync` — full rebuild, ends with exactly `count(product)` root docs.
- `ProductIndexerEventListener` → `indexOne` — single-product upsert keyed by product id.
- `GlobalIndexerService` (`@Profile({"!prod"})`, `@EventListener(ApplicationReadyEvent.class)`, `GlobalIndexerService.java:27-37`) — re-runs `reindexAll` on every non-prod boot, but each run swaps in a fresh index, so it cannot accumulate.

`TestCreateJSON` and `TestDataImportService` create *product rows* (which then get indexed via the normal event path) — they do not write to ES directly or out-of-band. Whatever they created is already counted in the 47 DB rows. No `@PostConstruct` / `CommandLineRunner` bulk-indexes products.

---

## Does this change the prior audit's "163 KB/doc" reading?

**Yes — it reframes it.** `audit-es-performance.md` (lines 220-260) read 1083 as the document count and computed "172.9 MB / 1083 docs ≈ 163 KB/doc," describing the index as "1083 seeded local docs." That denominator is the **Lucene doc count (root + nested)**, not products. Corrected:

- **Per actual product:** 181,301,943 bytes / **47** products ≈ **3.78 MB/product** (primaries store).
- The "163 KB" figure is "per Lucene doc," which is not a meaningful unit here since most Lucene docs are tiny nested children (a single `langCode`/`translation` pair).

The prior audit's *root-cause analysis* of the size, however, still stands and is in fact reinforced by this audit: it attributed the bloat to (a) `search_as_you_type` shingle/edge-prefix subfields on name+description (`audit-es-performance.md` finding, line 240) and (b) `include_in_parent: true` duplicating every nested field's terms into the parent for no query benefit (line 246). Both act *per nested doc* — and this audit shows there are 1036 nested docs across just 47 products, which is exactly why ~3.78 MB/product is so heavy for text-only listings (no binaries are stored — `imageKeys` are just R2 keys). The size driver is real; the *count* was a misread.

---

## Cleanup needed? (flag only — nothing cleaned)

1. **Doc count itself: nothing to clean.** All 1083 Lucene docs are legitimate — 47 real products and their nested children. No stale docs, no duplicates, no seed pollution, no tombstones (`docs.deleted = 0`), no orphaned `products_*` indices. The 23× number is a counting artifact, not garbage.

2. **DELETED / INACTIVE products are indexed (adjacent observation, flag only).** ES mirrors the DB, so the 2 DELETED and 1 INACTIVE rows are present as searchable docs. This is only correct if the read path filters `productState` at query time. Confirming the search query excludes non-ACTIVE products is worth a separate read-only check; it is **not** a doc-count problem and is out of scope for this brief. *(`findIdsForIndexing` indexes all states — `ProductRepository.java:45`.)*

3. **~3.78 MB/product store size (adjacent observation, flag only).** Large for text-only listings; driven by the nested fan-out × `search_as_you_type` subfields × `include_in_parent` duplication already flagged in `audit-es-performance.md`. This is an index-efficiency follow-up (a write-brief), not a doc-count cleanup. Not actionable here.

**No write is required to resolve the doc-count question — the 1083 is fully explained and correct.**
