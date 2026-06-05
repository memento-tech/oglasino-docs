# Audit — FilterConverter selected-filters type-mismatch (oglasino-backend, read-only)

**Repo:** oglasino-backend · **Branch:** dev (not switched) · **Mode:** READ ONLY (no code changed)
**Date:** 2026-06-01
**Scope:** Confirm the `FilterDTO.basic`/`order` defect on the product's *selected* filters path in `ProductForUpdateConverter`, its mechanism, its blast radius, and whether the new category-`filters` path shares it.

Every file:line below was confirmed with `cat -n` / `rg`, not Read alone (per the brief's tool-reliability note).

---

## TL;DR

**Defect confirmed.** On the owner product-load response, the *selected* filters are produced by mapping a bare `Filter` entity to `FilterDTO`. No converter exists for `Filter → FilterDTO`, so ModelMapper falls back to reflective property mapping, and `FilterDTO.basic`/`order` have no source field on `Filter` — they come out at Java defaults (`false` / `0`).

**One refinement to the brief's framing:** the `CategoryFilter`-typed converter (`FilterConverter`) is **not invoked** on this path (the source type doesn't match), so it doesn't "drop" `basic`/`order` — it's bypassed, and the default mapper has nowhere to read them from. The *outcome* the brief describes (defaulted values) is exactly right; the *mechanism* is "converter bypassed + missing source field," not "converter runs and loses them."

**Blast radius is one site.** Only `ProductForUpdateConverter` maps a bare `Filter`. The category path, the catalog path, and everything else pass a `CategoryFilter` and are correct. A fix is isolated to `ProductForUpdateConverter` and would **not** perturb the category path (which must not be touched).

**Severity: low / latent** on the backend side. Whether it bites depends on whether web/mobile actually read `basic`/`order` off the *selected* filters — a sibling-repo question I do not audit (see Q4). Not a trust-boundary issue.

---

## Q1 — Locate the selected-filters mapping; converter applied vs. type received

**Site:** `src/main/java/com/memento/tech/oglasino/converter/ProductForUpdateConverter.java:45-62`, specifically **line 49**:

```java
var filter = modelMapper.map(productFilter.getFilter(), FilterDTO.class);
```

Types on this line:
- `source.getFilterValues()` is `List<ProductFilterValue>` — `Product.java:77,199` (`private List<ProductFilterValue> filterValues;`).
- `productFilter` is therefore a `ProductFilterValue`; `productFilter.getFilter()` returns a **`Filter`** entity — `ProductFilterValue.java:27,46` (`private Filter filter; … public Filter getFilter()`).
- So the runtime mapping is **`Filter` → `FilterDTO`**.

**What converter would apply?** The only registered (non-Elasticsearch) converter that produces a `FilterDTO` is `FilterConverter`, and it is typed for a **different source type**:

- `FilterConverter implements Converter<CategoryFilter, FilterDTO>` — `FilterConverter.java:18`.
- Registered at `MapperConfiguration.java:64` (`modelMapper.addConverter(filterConverter)`).
- There is **no** `Converter<Filter, FilterDTO>` anywhere. (`rg "Converter<Filter,"` finds only `FilterReferenceConverter` → `Filter → FilterReference`, an Elasticsearch type, irrelevant here.)

`Filter` is not assignable to `CategoryFilter` (both extend `BaseEntity`; neither is a subtype of the other), so ModelMapper does **not** select `FilterConverter` for a `Filter` source. It falls through to its **default reflective property mapping** for `Filter → FilterDTO`.

**Type expected vs. received:** the converter that knows how to fill `basic`/`order` expects a **`CategoryFilter`**; this path hands it a **`Filter`**, so that converter is skipped entirely.

---

## Q2 — Confirm `basic`/`order` end up default-valued, and show the correct source

**The DTO fields:** `FilterDTO.java:10-11` — `private boolean basic;` / `private int order;` (Java defaults `false` / `0`).

**Why they default on this path:** the `Filter` entity has no field that the default mapper can match to `basic` or `order`. `Filter` carries only — `Filter.java:20-44` — `labelKey`, `iconId`, `filterType`, `filterKey`, `useDefaultOptionsOrder`, `options`, `filterRange`. No `basic`. No `order` / `displayOrder`. With no source property, the default mapping leaves both destination fields at their Java defaults.

**Where they would otherwise be populated:** `basic` and `order` are properties of **`CategoryFilter`**, not `Filter`:
- `CategoryFilter.java:19` `private boolean basic;`
- `CategoryFilter.java:21` `private int displayOrder;`

`FilterConverter` is the code that reads them — `FilterConverter.java:33-34`:

```java
destination.setBasic(source.isBasic());        // CategoryFilter.isBasic()
destination.setOrder(source.getDisplayOrder()); // CategoryFilter.getDisplayOrder()
```

Because that converter never runs on the selected-filters path (Q1), those two setters are never called, and the values stay `false`/`0`.

**Test coverage of this:** none. `ProductForUpdateConverterTest.java:105` sets `product.setFilterValues(null)`, so the selected-filters branch is exercised only with an empty list; no test asserts `basic`/`order` on a *selected* filter. The defect is uncovered.

---

## Q3 — The new category path, and shared code

**Category path is correct.** `ProductForUpdateConverter.java:79-81` maps the three immutable categories via `mapCategory(...)` → `modelMapper.map(category, CategoryDTO.class)` → `CategoryConverter`. Inside `CategoryConverter.java:31-35`:

```java
CollectionUtils.emptyIfNull(source.getFilters()).stream()           // Category.filters : Set<CategoryFilter>
    .sorted(Comparator.comparing(CategoryFilter::getDisplayOrder))
    .map(filter -> modelMapper.map(filter, FilterDTO.class))         // CategoryFilter -> FilterDTO
    .toList();
```

Here the source element type is **`CategoryFilter`**, so ModelMapper **does** select `FilterConverter`, and `basic`/`order` are populated correctly (`FilterConverter.java:33-34`). Confirmed: the category-`filters` path does **not** share the defect.

**Do the two paths share code a fix would touch?** The only common element is `FilterConverter` itself and the generic `modelMapper.map(..., FilterDTO.class)` call. The category path reaches `FilterConverter` (passes `CategoryFilter`); the selected path does not (passes `Filter`). `FilterConverter` is **correct as written** for its `CategoryFilter` contract — it must not change. Therefore a fix to the selected path is confined to `ProductForUpdateConverter` and **cannot perturb the category path**, provided it does not alter `FilterConverter`'s source type.

**Wider blast radius of the bug (other call sites):** `rg "FilterDTO.class"` shows two other production mappers that emit `FilterDTO`, both already correct because they pass `CategoryFilter`:
- `CatalogConverter.java:32` maps `source.getTopFilters()` — `Catalog.topFilters` is `Set<CategoryFilter>` (`Catalog.java:32`). Correct.
- `CategoryConverter.java:34` — `Category.filters` is `Set<CategoryFilter>`. Correct.

`ProductForUpdateConverter.java:49` is the **only** site that maps a bare `Filter` to `FilterDTO`. The defect is unique to the owner product-load *selected*-filters projection.

---

## Q4 — Does anything downstream read `basic`/`order` off the selected filters?

**This is a sibling-repo question and I do not audit `oglasino-web` / `oglasino-expo`** (CLAUDE.md hard rule: no cross-repo edits/reads; standing guidance: refuse sibling-repo audits even when a brief asks, and route them). So I answer only what the backend contract tells me and flag the consumer check for the web and mobile agents.

What the backend contract makes available to the update form (`ProductForUpdateDTO`):
- The three category objects `topCategory`/`subCategory`/`finalCategory`, each with a nested `filters` list whose `FilterDTO.basic`/`order` **are** correct (category path, Q3).
- The product's `filters` list (`List<SelectedFilterDTO>`), each wrapping a nested `FilterDTO` whose `basic`/`order` are **defaulted** (the defect) plus the chosen `options` / `selectedRangeValue`.

**Backend-side hypothesis (not asserted):** an edit form that renders the category's filter set for grouping/ordering would naturally take `basic`/`order` from the *category* filters (correct) and use the *selected* `filters` only to pre-fill chosen values — in which case the defaulted `basic`/`order` on the selected projection are never read and the defect is **latent** (wrong data on the wire, never consumed). If, instead, any client reads `basic`/`order` off the *selected* `filters[].filter`, those clients would see every selected filter as `basic=false, order=0` — visible mis-grouping / mis-ordering.

**Action owed:** web and mobile agents confirm whether their update-product forms read `filters[].filter.basic` / `.order` off the *selected* filters (vs. off the category filters). Their answer sets the severity:
- never read → **latent**, leave as a documented quirk or fix opportunistically;
- read → **wrong data on the wire that is consumed**, fix warranted.

---

## Q5 — Fix shape (plain words) and blast radius

The selected-filters path only has a bare `Filter` in hand; `basic`/`order` live on the `CategoryFilter` that links that `Filter` to one of the product's three categories. Any real fix has to recover that `CategoryFilter`. Options, lowest-risk first:

- **Option A — resolve the `CategoryFilter` and route through `FilterConverter`.** In the selected-filters loop, for each `ProductFilterValue.getFilter()`, find the matching `CategoryFilter` among the product's category chain (`source.getCategory()/getSubCategory()/getTopCategory()`, each exposing `getFilters()` of `CategoryFilter`) by filter id, then `modelMapper.map(thatCategoryFilter, FilterDTO.class)` instead of mapping the bare `Filter`. This reuses `FilterConverter` unchanged and yields correct `basic`/`order` plus the same option-sorting the category path gets. Risk: a selected filter with no matching `CategoryFilter` in the product's current categories (e.g. category re-org) needs a defined fallback.
- **Option B — map the `Filter`, then patch the two fields.** Keep line 49, then look up the same matching `CategoryFilter` and `setBasic`/`setOrder` from it. Functionally equal to A, more imperative, easier to fall back to defaults when no match exists.
- **Option C — accept as latent.** If Q4 says no client reads `basic`/`order` off the selected filters, leave the values defaulted and document that `FilterDTO.basic`/`order` are not meaningful on the *selected*-filters projection (or, cleaner, give the selected projection a slimmer DTO without those fields). Zero data-resolution cost; correctness deferred to the contract note.

**Blast radius of the fix (A or B):**
- **Touched:** `ProductForUpdateConverter.java` (the `selectedFilters` loop only) and `ProductForUpdateConverterTest.java` (add an assertion that a *selected* filter carries the expected `basic`/`order` — currently `filterValues` is `null` in the test, so a new fixture with a populated `ProductFilterValue` + its `CategoryFilter` is needed).
- **Not touched:** `FilterConverter` (changing its source type would break the category and catalog paths), `CategoryConverter`, `CatalogConverter`, `MapperConfiguration`.
- **Risk that fixing the selected path perturbs the category path:** **none**, as long as the fix stays inside `ProductForUpdateConverter`'s selected-filters loop and does not alter `FilterConverter`. The category path runs through `CategoryConverter → FilterConverter` and is independent.
- **New data dependency:** Options A/B require the product's `CategoryFilter` rows (already reachable via `source.get*Category().getFilters()`); confirm those associations are loaded in the same transaction/fetch used by the owner product-load (no extra query if already fetched for the category objects, which the converter already maps).

---

## Trust boundary (Part 11)

Confirmed **not** a trust issue. This is a read / response-shape converter. `ProductForUpdateDTO` is a read-only payload and is never accepted as an update request — the write DTO `UpdateProductRequestDTO` carries no category fields (`ProductForUpdateDTO.java:7-17` documents this; the selected `basic`/`order` are display hints). Nothing here makes a client-supplied value authoritative for a moderation, authorization, or state-transition decision. **Not CRITICAL.**

---

## Summary table

| Question | Answer |
| --- | --- |
| Defect real? | **Yes** — `ProductForUpdateConverter.java:49` maps bare `Filter → FilterDTO`; `basic`/`order` default. |
| Mechanism | `FilterConverter` (typed `CategoryFilter→FilterDTO`) is bypassed; default mapper has no `basic`/`order` source on `Filter`. |
| Correct source | `CategoryFilter.isBasic()` / `.getDisplayOrder()` (`CategoryFilter.java:19,21`), read by `FilterConverter.java:33-34`. |
| Category path affected? | **No** — it passes `CategoryFilter`, so `FilterConverter` runs and fills `basic`/`order` (`CategoryConverter.java:34`). |
| Shared code a fix touches? | None — fix is isolated to `ProductForUpdateConverter`; `FilterConverter` must stay as-is. |
| Other defective sites? | None — `CatalogConverter` and `CategoryConverter` both pass `CategoryFilter`. |
| Downstream consumers read it? | **Unknown — sibling-repo question, routed to web/mobile agents.** Backend hypothesis: likely latent. |
| Test coverage of the bug? | None — `ProductForUpdateConverterTest` nulls `filterValues`. |
| Trust boundary | Not CRITICAL — read-only display hints, never authoritative. |
| Severity | low / latent (pending Q4 answer). |
