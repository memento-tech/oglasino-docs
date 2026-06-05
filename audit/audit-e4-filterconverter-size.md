# Audit — FilterConverter type-mismatch on the selected-filters path: fix-size assessment

**Repo:** oglasino-backend · **Branch:** dev · **Mode:** READ-ONLY (no code changes)
**Date:** 2026-06-04
**Brief:** decide fix-vs-keep-parked PURELY on fix size for the parked (low/medium)
"FilterConverter type-mismatch on the selected-filters path" item (issues.md 2026-06-01
create/update parity carry-forward).

---

## Mechanism recap (confirmed against code)

`FilterConverter` is registered as `Converter<CategoryFilter, FilterDTO>`
(`converter/FilterConverter.java:18`). It reads the two contested fields from the
**CategoryFilter**, not the Filter:

- `destination.setBasic(source.isBasic());` — `FilterConverter.java:33`
- `destination.setOrder(source.getDisplayOrder());` — `FilterConverter.java:34`

`basic` / `displayOrder` are fields on `CategoryFilter`
(`entity/CategoryFilter.java:19,21`), i.e. the category-specific binding — not on the
shared `Filter`.

On the selected-filters path, `ProductForUpdateConverter` maps a bare `Filter`:

- `productFilter.getFilter()` returns a **`Filter`** (`entity/ProductFilterValue.java:27,46-47`).
- `modelMapper.map(productFilter.getFilter(), FilterDTO.class)`
  (`converter/ProductForUpdateConverter.java:49`) therefore asks ModelMapper to map a
  `Filter` source. `FilterConverter` does not apply (its source type is `CategoryFilter`),
  and no `Converter<Filter, FilterDTO>` exists, so the **default** property mapper runs.
  `Filter` has no `basic`/`order` properties to copy, so `FilterDTO.basic`
  (`dto/FilterDTO.java:10`) lands `false` and `FilterDTO.order` (`dto/FilterDTO.java:11`)
  lands `0`.

This is converter-bypass, matching the parked issue's description. Still harmless today
(no consumer reads `basic`/`order` off selected filters) — the brief is correct that this
is a size decision, not a correctness emergency.

---

## Q1 — The exact selected-filters mapping block

`converter/ProductForUpdateConverter.java:45-62` (verbatim):

```java
    var selectedFilters =
        CollectionUtils.emptyIfNull(source.getFilterValues()).stream()
            .map(
                productFilter -> {
                  var filter = modelMapper.map(productFilter.getFilter(), FilterDTO.class);
                  var options =
                      CollectionUtils.emptyIfNull(productFilter.getFilterOptions()).stream()
                          .map(option -> modelMapper.map(option, FilterOptionDTO.class))
                          .toList();

                  var result = new SelectedFilterDTO();
                  result.setFilter(filter);
                  result.setOptions(options);
                  result.setSelectedRangeValue(productFilter.getSelectedRangeValue());

                  return result;
                })
            .toList();
```

The offending line is `:49` — `modelMapper.map(productFilter.getFilter(), FilterDTO.class)`.
`productFilter.getFilter()` is a `Filter`, so the call bypasses `FilterConverter`.

---

## Q2 — Option A assessment (resolve matching `CategoryFilter` from the chain, map THAT)

### Is the category chain already in scope at that point?

**Yes — already in scope and already materialized in this exact convert, no fetch needed.**

- The three category objects are read on the `source` (the `Product` entity) at
  `ProductForUpdateConverter.java:79-81`:
  `source.getTopCategory()`, `source.getSubCategory()`, `source.getCategory()`.
  These are entity associations on `source`, accessible anywhere inside `convert()` —
  including the selected-filters loop at `:45-62` (which currently just doesn't use them).
- Each `Category` exposes its filters as `Set<CategoryFilter> getFilters()`
  (`entity/Category.java:42,94`).
- Each `CategoryFilter` exposes everything `FilterConverter` needs: `getFilter()`
  (`CategoryFilter.java:35`), `isBasic()` (`:43`), `getDisplayOrder()` (`:52`).
- Critically, the existing code **already traverses all three categories' `getFilters()`
  in the same convert call**: `mapCategory(...)` at `:79-81` routes each `Category` through
  `CategoryConverter`, which iterates `source.getFilters()` and maps each `CategoryFilter`
  via `modelMapper.map(filter, FilterDTO.class)` (`converter/CategoryConverter.java`,
  the `setFilters(...)` block). So the `CategoryFilter` collections are loaded on this path
  regardless. **Option A reuses data already in hand — it introduces no new query and no
  new lazy-load.**

Lookup needed: match `productFilter.getFilter().getId()` against the chain's
`CategoryFilter.getFilter().getId()` to find the right binding for the contested fields.

### How many lines does the fix add? Which files?

**One file: `converter/ProductForUpdateConverter.java`. ~10-15 lines added, 1 changed.**

Shape:
- Build a `Map<Long, CategoryFilter>` keyed by `Filter` id from the (null-safe) union of the
  three categories' `getFilters()` — ~5-7 lines (one helper or an inline `Stream.of(top, sub,
  final).filter(Objects::nonNull).flatMap(c -> getFilters().stream())` collected to a map).
- In the loop, replace `:49` with a lookup: if a matching `CategoryFilter` is found, map THAT
  (`modelMapper.map(matchingCategoryFilter, FilterDTO.class)` → routes through
  `FilterConverter`); else fall back to the current bare-`Filter` map — ~3-5 lines.

No other file changes. No DTO change (`FilterDTO` already has `basic`/`order`). No call-site
or controller change (read path, same response shape — only `basic`/`order` go from
default to correct).

### Does FilterConverter's signature already accept what you'd hand it?

**Yes. No new/overloaded entry point.** `FilterConverter` is
`Converter<CategoryFilter, FilterDTO>` (`FilterConverter.java:18`) and you would hand it a
`CategoryFilter`. `CategoryConverter` already calls `modelMapper.map(categoryFilter,
FilterDTO.class)` against this exact converter, proving the entry point works as-is. The fix
explicitly does **not** alter `FilterConverter` (per the parked issue's Option A).

### One edge to note (not extra scope)

A selected filter whose `Filter` is no longer present in the product's current category chain
yields no match → fall back to the bare-`Filter` map → `basic=false`/`order=0`, i.e. exactly
today's behavior. The fallback is a one-liner, not a new code path's worth of work, and keeps
the change strictly non-regressive.

---

## Q3 — Test impact

`ProductForUpdateConverterTest.java` **exists**
(`src/test/java/com/memento/tech/oglasino/converter/ProductForUpdateConverterTest.java`).

- The harness is already favorable: `setUp()` (`:46-63`) wires **real** `FilterConverter`,
  `CategoryConverter`, and `ProductForUpdateConverter` into a real `ModelMapper` — no mapping
  is mocked, so a new assertion exercises the true converter chain.
- The selected-filters loop is **currently uncovered**: `productWithCategoryChain()` sets
  `product.setFilterValues(null)` (`:105`), so the loop at `:45-62` never runs in any existing
  test. Both existing tests assert only categories + mutable display fields.
- The fixture helper `category(...)` (`:133-152`) already builds a `CategoryFilter` with
  `setBasic(true)` / `setDisplayOrder(0)` and a `Filter`, so the building blocks for a
  selected-filter-with-known-id fixture are already present.

**Estimated test work: modest — one new `@Test` plus a small `ProductFilterValue` fixture
helper (~25-40 lines).** The new test would attach a `ProductFilterValue` whose `Filter` id
matches a chain `CategoryFilter`, run the converter, and assert
`result.getFilters().get(0).getFilter().isBasic()` / `.getOrder()` reflect the chain binding
(and a second case asserting the unmatched-filter fallback, if desired). No existing test
needs to change. (Worth bumping `displayOrder` to a non-zero value in the fixture so the
assertion can't pass on the default `0`.)

---

## Q4 — Verdict

**SMALL.**

≤~15 production lines in a single file (`ProductForUpdateConverter.java`); no signature
change to `FilterConverter` and no new converter; the category chain (and its
`CategoryFilter`s) is already in scope and already materialized on this exact path, so no
fetch/query is added; the existing test harness already wires the real converter chain, so
covering the corrected `basic`/`order` is one new test method plus a small fixture helper.

---

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no code touched.
- Part 4a (simplicity): **N/A (read-only audit, no code written this session).**
- Part 4b (adjacent observations): none new — the finding under assessment is already logged
  (issues.md 2026-06-01, parked). No additional out-of-scope findings surfaced.
- Part 6 (translations): N/A.
- Part 11 (trust boundaries): unaffected — `ProductForUpdateDTO` is read-path only
  (`ProductForUpdateConverter.java:29-31` doc); `basic`/`order` are display metadata, never a
  trust-decision input.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change required by this audit. The 2026-06-01 parked entry already records the
  fix shape (Option A) and the test gap; this audit confirms its SMALL sizing if Mastermind
  elects to unpark. (Any status flip is Mastermind-drafted / Docs/QA-applied, not authored here.)
