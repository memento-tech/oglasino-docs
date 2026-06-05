# Impl: ES description-field mapping change — search_as_you_type → text

**Repo:** `oglasino-backend` · **Date:** 2026-06-04 · **Type:** implementation + local reindex
**Environment:** local Docker ES (`localhost:9200`) + local Docker Postgres (dev profile)

> **Branch note (hard-rule conflict, flagged):** the brief asked for a new feature branch off `dev`.
> My CLAUDE.md hard rules forbid `git checkout` to a different branch. I did **not** create/switch
> branches — all changes are on `dev`, **uncommitted**. Igor creates the feature branch and commits.

---

## Split approach chosen + why

**Chosen: split `TranslationReference` into a shared interface + two concrete reference classes.**

Spring Data ES generates field mappings purely from annotations on the class a field references; there
is no `@Mapping`/`mappingPath` override in this project (confirmed — `grep -rn "@Mapping\|mappingPath"
src/main/java` returns nothing) and no per-field-use override mechanism exists. So name and description
can only carry different mappings if their list elements are genuinely different classes.

- `TranslationReference` is now an **interface** exposing `getLangCode()` / `getTranslation()` — the
  read-side contract. Keeping the existing type name as the interface means the single reader method
  `ProductDocument.getTranslatedValue(...)` stays one method (widened to `List<? extends
  TranslationReference>`); no reader changed.
- `NameTranslationReference implements TranslationReference` — `translation` keeps `@Field(type =
  Search_As_You_Type, analyzer = "name_autocomplete_index", searchAnalyzer =
  "name_autocomplete_search")`. **Byte-for-byte the old mapping.** Name typeahead (`matchBoolPrefix`)
  needs the prefix subfields.
- `DescriptionTranslationReference implements TranslationReference` — `translation` is now `@Field(type
  = FieldType.Text, analyzer = "standard")`. `langCode` stays `@Field(type = FieldType.Keyword)`.

Considered and rejected: an abstract base class holding the shared `langCode`. Rejected for simplicity —
each concrete class is ~10 lines and self-contained; an interface for the read contract plus two flat
classes keeps each class's full ES mapping visible in one file, with no cross-hierarchy field-mapping
surprises. The `langCode`/getter boilerplate duplication is trivial and worth the explicitness.

---

## Blast radius (grep + what changed)

```
$ grep -rn "TranslationReference" src/
src/.../converters/DocumentProductConverter.java:8:  import ...TranslationReference;
src/.../converters/DocumentProductConverter.java:57:   var result = new TranslationReference();   (name)
src/.../converters/DocumentProductConverter.java:68:   var result = new TranslationReference();   (description)
src/.../reference/TranslationReference.java:6:        public class TranslationReference {
src/.../documents/ProductDocument.java:5:             import ...TranslationReference;
src/.../documents/ProductDocument.java:28,31:         private List<TranslationReference> name/descriptionTranslations;
src/.../documents/ProductDocument.java:111-124:       getters/setters
src/.../documents/ProductDocument.java:272,276:       getTranslatedValue(List<TranslationReference> ...)

$ grep -rn "TranslationReference" src/test  →  no test refs
```

Three categories from the brief, each verified:

1. **Populates `descriptionTranslations` (the writer):** `DocumentProductConverter`
   (`Converter<Product, ProductDocument>`) builds the lists by hand. Changed `new
   TranslationReference()` → `new NameTranslationReference()` (name loop) and `new
   DescriptionTranslationReference()` (description loop). The mapper still populates both correctly —
   confirmed by the live reindex (47 docs, description text present, see Check 1). `toIndexQuery` in
   `ProductIndexer` is unaffected (it just `modelMapper.map(product, ProductDocument.class)`).
2. **Reads `descriptionTranslations` back (the readers):** `ProductDetailsConverter`,
   `ProductOverviewConverter`, `SearchProductDataConverter` all call
   `source.getTranslatedValue(source.getDescriptionTranslations()/getNameTranslations(), ...)`. None
   references the `TranslationReference` type by name — they only call getters and pass the result to
   `getTranslatedValue`. `getTranslatedValue` now takes `List<? extends TranslationReference>`, which
   accepts both `List<NameTranslationReference>` and `List<DescriptionTranslationReference>`. The same
   `getTranslation()` getter is exposed → read side unaffected. **No reader file changed.**
3. **Other references:** only `ProductDocument` (field types + getters/setters) and
   `DocumentProductConverter`. No tests, no other source.

**Files changed (this session only):**
- `reference/TranslationReference.java` — class → interface (+8 / -16 net of comment reformat)
- `reference/NameTranslationReference.java` — **new** (39 lines)
- `reference/DescriptionTranslationReference.java` — **new** (38 lines)
- `documents/ProductDocument.java` — field/getter/setter retyped; `getTranslatedValue` widened (+? / -?)
- `converters/DocumentProductConverter.java` — two `new` sites + imports

`./mvnw spotless:apply compile` clean. `./mvnw test` → **934 passed, 0 failures, BUILD SUCCESS.**

---

## New mapping (raw `_mapping` output)

After reindex, `GET products/_mapping` (`descriptionTranslations.translation`, `langCode`, and the
unchanged `nameTranslations.translation`), index `products_20260604155727`:

```json
"nameTranslations.translation":        {"type": "search_as_you_type", "doc_values": false, "max_shingle_size": 3, "analyzer": "name_autocomplete_index", "search_analyzer": "name_autocomplete_search"}
"descriptionTranslations.translation": {"type": "text", "analyzer": "standard"}
"descriptionTranslations.langCode":    {"type": "keyword"}
```

`nameTranslations.translation` is **byte-for-byte identical** to the pre-change mapping (compared
against the captured BEFORE state on `products_20260604073907`).

---

## Reindex method used + post-reindex state

**Method:** the non-prod boot path — `GlobalIndexerService` (`@Profile("!prod")`, fires on
`ApplicationReadyEvent`) calls `ProductIndexer.reindexAll()`, the existing blue/green flow. Booted
locally with `./mvnw spring-boot:run -Dspring-boot.run.profiles=dev` (dev profile imports `.env` →
localhost Postgres/Redis/ES). No hand-written reindex; the admin endpoint was not used (it needs ADMIN
auth, harder locally). App was shut down after the swap (ES queries don't need it running).

Raw reindex log:

```
Starting product reindex: physicalIndex=products_20260604155727 jobId=null
Created physical index products_20260604155727
Reindex batch done: page=0 batchSize=47 indexed=47/47
Atomic alias swap: products products_20260604073907 → products_20260604155727
Deleted previous physical index products_20260604073907
Product reindex done: physicalIndex=products_20260604155727 indexed=47 elapsedMs=1638 jobId=null
```

Post-reindex confirmations:

```
$ GET /_alias/products   → { "products_20260604155727": { "aliases": { "products": {} } } }
$ GET /products/_count   → { "count": 47, ... }     ✓ still 47
$ store size             → 9,210,846 bytes (~8.8 MB)   (was 181,302,611 bytes / ~172.9 MB disk_usage)
```

No orphan accumulation — the old concrete index `products_20260604073907` was deleted by the flow; only
`products_20260604155727` remains.

---

## Correctness check (raw results)

```
CHECK 1 — full-search description clause (term langCode=cnr + plain match, boost 0.2), FULL word "prelijepe"
  hits: [8592, 8580]      ✓ description still searchable via the plain match the code issues

CHECK 2 — autocomplete name prefix "pati" via match_bool_prefix on nameTranslations.translation
  hits: [8592]            ✓ name typeahead intact — search_as_you_type preserved (MUST NOT regress)

CHECK 3 — BEHAVIOR DELTA: PARTIAL word "prelije" on description
  BEFORE (search_as_you_type / edge-ngram): hits [8564, 8592, 8568, 8585, 8587]   (5 hits)
  AFTER  (text / standard):                 hits []                               (0 hits)
```

(Queries run directly against ES mirroring the exact `descriptionNested` / `prefixNameNested` clause
shapes from `TextQueryGenerator`, per the Phase-2 audit's verified query paths.)

---

## Behavior delta (restated)

Description was edge-ngram analyzed, so a partial word matched (`prelije` → `prelijepe`). With the
`standard` analyzer, description matches **whole words only** (`prelijepe` matches; `prelije` does not).
For a boost-0.2 relevance nudge — name carries the real boosts (5/8/4) — this is acceptable and arguably
more correct, but it **is** a behavior change. Demonstrated live by Check 3 above (5 hits → 0). No
change to any query the code actually issues; name search/autocomplete is untouched.

---

Igor to verify final size via `_disk_usage` in console before commit is trusted.
