# Audit: ES description-field search-usage verification (read-only)

**Repo:** `oglasino-backend` · **Branch:** `dev` · **Type:** read-only (no writes, no ES/DB mutations, no reindex)
**Date:** 2026-06-04

## Verdict

**Changing `descriptionTranslations.translation` from `search_as_you_type` to plain `text` would affect: NOTHING in query behavior** — the only thing that queries description anywhere in the codebase is a single plain `.match()` on the root field `descriptionTranslations.translation` (boost 0.2), in both full search and autocomplete (they share one query path). A plain `.match()` reads the analyzed root field, never the `._index_prefix` / `._2gram` / `._3gram` prefix subfields, so dropping those subfields changes no result. **Name is different** — name uses `.matchBoolPrefix()`, which DOES rely on the prefix machinery, so name's `search_as_you_type` mapping must be preserved.

**One blocking caveat for the later mapping-change brief (Q1):** name and description share a single Java class (`TranslationReference`) whose one `translation` property carries the `@Field(type = Search_As_You_Type, …)`. Field mappings are generated purely from annotations (no `@Mapping`/`mappingPath` override exists — confirmed). So the annotation cannot be changed for description without also changing it for name. **The shared class must be split first** (or the mapping otherwise overridden per-field). And whatever replaces description's mapping must keep `descriptionTranslations.translation` an analyzed `text` field (the plain `.match()` must keep working) and should preserve an equivalent analyzer.

---

## Q1 — Is the `TranslationReference` field mapping actually shared by both name and description?

**Yes — fully shared, single annotation.** Description's mapping cannot be changed independently of name via the annotation; the shared class must be split first.

`src/main/java/com/memento/tech/oglasino/elasticsearch/reference/TranslationReference.java` (verbatim, lines 6–14):

```java
public class TranslationReference {
  @Field(type = FieldType.Keyword)
  private String langCode;

  @Field(
      type = FieldType.Search_As_You_Type,
      analyzer = "name_autocomplete_index",
      searchAnalyzer = "name_autocomplete_search")
  private String translation;
```

Both name and description are typed `List<TranslationReference>` in `ProductDocument`
(`src/main/java/com/memento/tech/oglasino/elasticsearch/documents/ProductDocument.java`, lines 27–31):

```java
  @Field(type = FieldType.Nested, includeInParent = true)
  private List<TranslationReference> nameTranslations;

  @Field(type = FieldType.Nested, includeInParent = true)
  private List<TranslationReference> descriptionTranslations;
```

Confirmation the mapping is annotation-derived (no override file):

```
$ grep -rn "@Mapping\|mappingPath" src/main/java
none
```

`ProductDocument` carries only `@Setting(settingPath = "es-config/elastic-analyzer.json")` (ProductDocument.java:16) — that is index **settings/analyzers**, not field mappings. The only es-config resource is `src/main/resources/es-config/elastic-analyzer.json` (analyzer definitions). Therefore `nameTranslations.translation` and `descriptionTranslations.translation` receive an **identical** `search_as_you_type` mapping generated from the one `@Field` on the shared `translation` property. Changing that annotation changes BOTH name and description.

---

## Q2 — What does the FULL-SEARCH query do with description (and name)?

All clauses are built in `TextQueryGenerator.addQuery`
(`src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/TextQueryGenerator.java`).

### Description — one clause, plain `.match()`, root field, boost 0.2 (confirms the brief's claim)

Lines 94–112:

```java
      var descriptionNested =
          QueryBuilders.nested()
              .path("descriptionTranslations")
              .query(
                  q ->
                      q.bool(
                          b ->
                              b.must(
                                      QueryBuilders.term()
                                          .field("descriptionTranslations.langCode")
                                          .value(lang)
                                          .build())
                                  .must(
                                      QueryBuilders.match()
                                          .field("descriptionTranslations.translation")
                                          .query(text)
                                          .boost(0.2f)
                                          .build())))
              .build();
```

- Clause type: `QueryBuilders.match()` (plain match).
- Field: `descriptionTranslations.translation` — the **root** field only. No `._index_prefix`, no `._2gram`, no `._3gram`, no `.*`.
- Boost: `0.2f`. Language-filtered by a `term` on `descriptionTranslations.langCode`.
- Combined as a `.should(descriptionNested)` (line 121).

A plain `match` on a `search_as_you_type` root field queries the analyzed root text, not the prefix subfields. So this clause behaves identically whether the field is `search_as_you_type` or plain `text`.

### Name — three clauses; one of them (`matchBoolPrefix`) relies on the prefix subfields

1. **`fuzzyNameNested`** (lines 28–49) — `QueryBuilders.match()` on `nameTranslations.translation`, `fuzziness("AUTO")`, `prefixLength(1)`, `operator(And)`, boost `5f`. Root field. (`.must`, line 118)
2. **`exactNameNested`** (lines 51–69) — `QueryBuilders.match()` on `nameTranslations.translation`, boost `8f`. Root field. (`.should`, line 119)
3. **`prefixNameNested`** (lines 71–89) — **`QueryBuilders.matchBoolPrefix()`** on `nameTranslations.translation`, boost `4f`. (`.should`, line 120)

All three target the root field `nameTranslations.translation` by name. Clause 3 is `matchBoolPrefix`, which against a `search_as_you_type` field uses the `_2gram` / `_3gram` / `_index_prefix` subfields internally to do prefix/typeahead matching. **This is the only consumer of the prefix machinery, and it is on NAME, not description.**

Combination (lines 117–122):

```java
      queryBuilder
          .must(fuzzyNameNested)
          .should(exactNameNested)
          .should(prefixNameNested)
          .should(descriptionNested)
          .minimumShouldMatch("0");
```

---

## Q3 — Does AUTOCOMPLETE build a different query?

**No. Autocomplete routes through the exact same `buildQuery` / `TextQueryGenerator` as full search.** Therefore the name/description clauses are identical to Q2.

Trace:

`DefaultProductsSearchFacade.getAutocompleteProductsSearch`
(`src/main/java/com/memento/tech/oglasino/elasticsearch/facade/impl/DefaultProductsSearchFacade.java`, lines 60–72) calls the same service method as full search (`getProductOverviews`, line 57):

```java
  public List<SearchProductData> getAutocompleteProductsSearch(
      SearchProductsDTO searchProducts, ProductsSearchMode productsSearchMode) {

    return CollectionUtils.emptyIfNull(
            productsSearchService
                .getProductsFor(searchProducts, productsSearchMode)   // <-- same call as full search
                .getSearchHits())
        .stream()
        .map(SearchHit::getContent)
        .map(data -> documentMapper.map(data, SearchProductData.class))
        .toList();
```

`DefaultProductsSearchService.getProductsFor`
(`src/main/java/com/memento/tech/oglasino/elasticsearch/service/impl/DefaultProductsSearchService.java`, lines 45–50):

```java
  public SearchHits<ProductDocument> getProductsFor(
      SearchProductsDTO searchProducts, ProductsSearchMode productsSearchMode) {
    return elasticsearchOperations.search(
        productsFilterQueryBuilder.buildQuery(searchProducts, productsSearchMode),
        ProductDocument.class);
  }
```

`DefaultProductsFilterQueryBuilder.buildBaseQuery`
(`src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/DefaultProductsFilterQueryBuilder.java`, lines 95–122) iterates every `QueryGenerator` bean — `TextQueryGenerator` among them — for every search:

```java
      generators.forEach(
          generator -> {
            if (productsSearchMode == ProductsSearchMode.DASHBOARD_SEARCH
                && generator instanceof UserQueryGenerator) {
              return;
            }
            generator.addQuery(boolBuilder, productsFilter);
          });
```

There is **no separate autocomplete query construction**. The three public `/autocomplete` endpoints (`ProductSearchController`, `DashboardProductController`, `admin/AdminProductSearchController`) all delegate to `getAutocompleteProductsSearch`, which delegates to `getProductsFor` → `buildQuery`. The only autocomplete-vs-full-search difference is response mapping (`SearchProductData` vs `ProductOverviewDTO`), not the query.

**Answer to the key question:** autocomplete prefix-matches against **NAME only** (via `prefixNameNested` / `matchBoolPrefix`). Against **description** it does only a plain `.match()` (boost 0.2) — autocomplete does **not** prefix-match description. So the description prefix subfields are dead weight for autocomplete too.

---

## Q4 — Does ANYTHING else query the description prefix subfields?

**No.** The only place `descriptionTranslations` appears in query construction is the single `TextQueryGenerator` clause from Q2.

Every `descriptionTranslations` reference in `src/`:

```
$ grep -rn "descriptionTranslations" src/
src/.../generator/impl/TextQueryGenerator.java:96:              .path("descriptionTranslations")
src/.../generator/impl/TextQueryGenerator.java:103:                  .field("descriptionTranslations.langCode")
src/.../generator/impl/TextQueryGenerator.java:108:                  .field("descriptionTranslations.translation")
src/.../documents/ProductDocument.java:31:  private List<TranslationReference> descriptionTranslations;
src/.../documents/ProductDocument.java:120:    return descriptionTranslations;
src/.../documents/ProductDocument.java:123:  public void setDescriptionTranslations(List<TranslationReference> descriptionTranslations) {
src/.../documents/ProductDocument.java:124:    this.descriptionTranslations = descriptionTranslations;
```

Only TextQueryGenerator (the Q2 clause: `path`, `langCode` term, root-field `match`) plus the document field/getter/setter. **No sort, no aggregation, no `multiMatch`, no wildcard `fields("*")`, no `matchBoolPrefix`, no subfield literal touches description.**

No `search_as_you_type` subfield is referenced by literal string anywhere — the only hit for the family is the `@Field` annotation itself:

```
$ grep -rn "_index_prefix\|_2gram\|_3gram\|search_as_you_type\|Search_As_You_Type" src/
src/.../reference/TranslationReference.java:11:      type = FieldType.Search_As_You_Type,
```

Sort path note: `DefaultProductsFilterQueryBuilder.addSortToQuery` (lines 149–170) honors only client sort fields that pass the seeded `OrderType` whitelist (`orderTypeService.isSortableField`, line 159), defaulting to `id DESC`. No translation/description field participates in sorting.

---

## Preservation checklist for the later mapping-change brief

1. **Split the shared mapping first.** `TranslationReference.translation` is one annotation feeding both name and description. To make description plain `text` while keeping name `search_as_you_type`, the two must stop sharing that annotation (e.g., split `TranslationReference` into a name-typed and a description-typed reference class, or otherwise give description its own `@Field`). Mappings are annotation-derived — no `@Mapping` JSON to edit instead.
2. **Keep `descriptionTranslations.translation` an analyzed `text` field.** The full-search/autocomplete `descriptionNested` clause is a plain `.match()` on this root field; it must keep matching. Preserve an equivalent analyzer (today it analyzes via the `name_autocomplete_index` / `name_autocomplete_search` analyzers from `elastic-analyzer.json`).
3. **Do not touch name's mapping.** Name's `prefixNameNested` (`matchBoolPrefix`) depends on the `search_as_you_type` prefix subfields. Leave `nameTranslations.translation` as `search_as_you_type`.
4. **A reindex will be required** to apply the new description mapping (mapping type changes are not in-place). Out of scope for this read-only audit — flagged only.

_No writes performed. No ES/DB mutations, no reindex, no branch switch._
