# Audit — Backend Filtering & Search Reference

**Repo:** oglasino-backend
**Branch:** dev (no branch switch)
**Mode:** READ-ONLY — no code changed.
**Date:** 2026-05-31
**Scope:** what the backend product filtering / search path does today, at `file:line` precision, as the contract the mobile alignment work must match.

All `file:line` references are relative to repo root. The code is ground truth; no prior audit or spec was consulted for these answers.

---

## Q1 — State enums

### `ProductState` — **3 constants**

`src/main/java/com/memento/tech/oglasino/entity/ProductState.java:3-7`

```java
public enum ProductState {
  ACTIVE,
  INACTIVE,
  DELETED
}
```

- `ACTIVE` — `ProductState.java:4`
- `INACTIVE` — `ProductState.java:5`
- `DELETED` — `ProductState.java:6`

There is **no** `PENDING` and **no** `REJECTED` constant. (A stale Javadoc in `ProductStateQueryGenerator` references "PENDING / REJECTED / INACTIVE" — see Also-report.)

### `ModerationState` — **2 constants**

`src/main/java/com/memento/tech/oglasino/entity/ModerationState.java:3-6`

```java
public enum ModerationState {
  APPROVED,
  BANNED
}
```

- `APPROVED` — `ModerationState.java:4`
- `BANNED` — `ModerationState.java:5`

---

## Q2 — Translation keys for state values (THE load-bearing answer)

### Headline: per-value keys are **MISSING for all five state values.**

No translation key exists for any individual `ProductState` or `ModerationState` value. A repo-wide search of all four `0001-data-web-translations-*.sql` seed files for any key whose segments encode a state value (`active` / `inactive` / `deleted` / `approved` / `banned` under a `product`/`moderation`/`state` prefix) returned **zero matches**.

| State value | Enum | Per-value translation key | Verdict |
| --- | --- | --- | --- |
| `ACTIVE` | ProductState | none | **MISSING** |
| `INACTIVE` | ProductState | none | **MISSING** |
| `DELETED` | ProductState | none | **MISSING** |
| `APPROVED` | ModerationState | none | **MISSING** |
| `BANNED` | ModerationState | none | **MISSING** |

### What DOES exist (so it isn't mistaken for the above)

Two **filter-dropdown LABEL** keys exist — these label the filter control, not its option values:

| Key | Namespace | EN sample | All four locales? |
| --- | --- | --- | --- |
| `product.state.filter.label` | `DASHBOARD_PAGES` | `'Product State'` | **Yes** — EN `0001-data-web-translations-EN.sql:1204`, RS `…-RS.sql:1201`, CNR `…-CNR.sql:1199`, RU `…-RU.sql:1202` |
| `moderation.state.filter.label` | `DASHBOARD_PAGES` | `'Moderation State'` | **Yes** — EN `…-EN.sql:1205`, RS `…-RS.sql:1202`, CNR `…-CNR.sql:1200`, RU `…-RU.sql:1203` |

Sample row (EN):
```sql
(3845, 3, 'DASHBOARD_PAGES', 'product.state.filter.label', 'Product State', CURRENT_TIMESTAMP),
(3846, 3, 'DASHBOARD_PAGES', 'moderation.state.filter.label', 'Moderation State', CURRENT_TIMESTAMP),
```

Adjacent **lookalikes that are NOT product/moderation state values** (do not reuse for product filtering):
- `user.info.dialog.state.active` / `user.info.dialog.state.banned` (`DIALOG` namespace, e.g. CNR `…-CNR.sql:474-475`) — these are **user account** state labels, not `ProductState`/`ModerationState`.
- `reviews.filter.approved` / `reviews.filter.disapproved` (`ADMIN_PAGES`, e.g. CNR `…-CNR.sql:924,928`) — **review** moderation labels, not product moderation state.

There is **no parent/child collision risk**: the leaf keys are `product.state.filter.label` and `moderation.state.filter.label`; `product.state` / `moderation.state` are not themselves seeded as leaves, so per-value children could nest under a new sub-prefix (Part 6 Rule 2 satisfied as long as you don't seed a bare `product.state` leaf).

### Where a future seed for these would belong (per conventions Part 6)

**This is a report, not an action — no seed was written.**

- **Namespace:** `DASHBOARD_PAGES`. The two existing filter labels already live there; product-state / moderation-state value labels are dashboard-filter UI, so they belong in the same group. (Namespace is fixed per Part 6 Rule 1 — do not invent a new one.)
- **Seed files:** all four — `0001-data-web-translations-EN.sql`, `-RS.sql`, `-CNR.sql`, `-RU.sql`. One row per value per locale (5 values × 4 locales = 20 rows) if all five are labelled; the mobile work may only need the dashboard-reachable subset (`ACTIVE`, `INACTIVE` for product; `APPROVED`, `BANNED` for moderation — `DELETED` is never surfaced to the dashboard, see Q3).
- **Suggested key pattern (decision is the seed brief's, not this audit's):** `product.state.<value_lowercase>` and `moderation.state.<value_lowercase>` (e.g. `product.state.active`, `moderation.state.approved`) — mirrors the existing `product.state.filter.label` stem. Because `product.state.filter.label` already nests under `product.state.*`, adding `product.state.active` is consistent and collision-free (no bare `product.state` leaf exists).
- **Append point & next available ID (per Part 6 Rule 3 — append at end of the namespace group, next free ID, stop on collision):** the `DASHBOARD_PAGES` group ends just before an `--increaseby(100)` gap, so there is ample headroom inside each group's id band:

  | Locale | lang_id | DASHBOARD_PAGES group bounds | Highest id in group | Next free id | Next namespace starts at |
  | --- | --- | --- | --- | --- | --- |
  | EN | 3 | `…-EN.sql:1157`–`1215` | `3854` (`…-EN.sql:1213`) | **3855** | `4000` (COOKIES, `…-EN.sql:1221`) |
  | RS | 1 | `…-RS.sql:1154`–`1212` | `5954` (`…-RS.sql:1210`) | **5955** | (next group after +100) |
  | CNR | 2 | `…-CNR.sql:1152`–`1210` | `1754` (`…-CNR.sql:1208`) | **1755** | (next group after +100) |
  | RU | 4 | `…-RU.sql:1155`–`1213` | `8054` (`…-RU.sql:1211`) | **8055** | (next group after +100) |

  No collision: each locale's id band has ≥100 free ids between the group's current max and the next group's `--increaseby(100)` boundary. The seed brief must re-confirm the next free id at write time (this audit is a snapshot).

---

## Q3 — State-filter handling per search mode

Component: `ProductStateQueryGenerator.wrapWithStateSearchMode(...)` at `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/ProductStateQueryGenerator.java:38-69`. Allowed-set constant at `:18-19`:
```java
private static final Set<ProductState> ALLOWED_DASHBOARD_PRODUCT_STATES =
    Set.of(ProductState.ACTIVE, ProductState.INACTIVE);
```

### PORTAL_SEARCH — body states **ignored**; hard-pinned to `ACTIVE` + `APPROVED`

`ProductStateQueryGenerator.java:52-55`:
```java
case PORTAL_SEARCH ->
    queryBuilder
        .filter(f -> f.term(t -> t.field("productState").value("ACTIVE")))
        .filter(f -> f.term(t -> t.field("moderationState").value("APPROVED")));
```
Client-supplied `productStates` / `moderationStates` are **never consulted** for PORTAL. The two locals are read at `:43-49` but the PORTAL branch does not reference them. **Leftover state filters from a client cannot affect the public surface** — the hard pin to `ACTIVE`+`APPROVED` is absolute.

### DASHBOARD_SEARCH — `productStates` intersected with `{ACTIVE, INACTIVE}`; `moderationStates` passed through

`ProductStateQueryGenerator.java:57-60`:
```java
case DASHBOARD_SEARCH -> {
  addProductStatesFilterTerms(queryBuilder, productStates, ALLOWED_DASHBOARD_PRODUCT_STATES);
  addModerationStatesFilterTerms(queryBuilder, moderationStates);
}
```

- **`productStates`** → `getProductStatesFieldValues(...)` at `:86-107`. With a non-empty `allowed` set:
  - if the body supplies `productStates`, they are **intersected** with `ALLOWED_DASHBOARD_PRODUCT_STATES` (`:96-101`) — `DELETED` is silently dropped; only `ACTIVE`/`INACTIVE` survive.
  - if the body supplies **no** `productStates`, it **defaults to the full allowed set** `{ACTIVE, INACTIVE}` (`:95`, `finalAllowed = allowed`). So a dashboard query always filters productState to a subset of `{ACTIVE, INACTIVE}` and never returns `DELETED`.
- **`moderationStates`** → `addModerationStatesFilterTerms(...)` at `:71-84`: applied verbatim if non-empty, **no restriction**. If empty/null, no moderation filter is added (a dashboard user sees both `APPROVED` and `BANNED` of their own products). This is safe because ownership is pinned to the authenticated user (Q4).

### ADMIN_SEARCH — both passed through unrestricted

`ProductStateQueryGenerator.java:62-65`:
```java
case ADMIN_SEARCH -> {
  addProductStatesFilterTerms(queryBuilder, productStates, null);
  addModerationStatesFilterTerms(queryBuilder, moderationStates);
}
```
- **`productStates`** with `allowed == null` → `getProductStatesFieldValues` takes the `CollectionUtils.isEmpty(allowed)` branch (`:90-93`): if `productStates` non-empty, used verbatim (any of `ACTIVE`/`INACTIVE`/`DELETED`); if empty, **no productState filter** (all states returned).
- **`moderationStates`** passed through verbatim (`:71-84`).

**B6 takeaway:** orphaned state filters after a client "clear-all" are **harmless on PORTAL** (ignored entirely) and **bounded on DASHBOARD** (productState can never escape `{ACTIVE, INACTIVE}`, and ownership is server-pinned so a leftover `moderationStates` only ever re-filters the caller's own products). Only ADMIN honors arbitrary leftover state filters — and ADMIN is role-gated.

---

## Q4 — Ownership / search-mode wrap

Two cooperating components decide `ownerId` scoping:

1. `SearchModeQueryGenerator.wrapWithSearchMode(...)` — `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/SearchModeQueryGenerator.java:15-36`.
2. `UserQueryGenerator.addQuery(...)` — `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/UserQueryGenerator.java:12-20` — applies the **body** `ownerId` as a filter, but is **conditionally skipped** by the builder.
3. The skip logic lives in `DefaultProductsFilterQueryBuilder.buildBaseQuery(...)` `:100-110`.

### PORTAL_SEARCH — `ownerId` taken from the **request body** (client-supplied), unconstrained by auth

- `SearchModeQueryGenerator.java:19`: `case PORTAL_SEARCH -> {}` — no ownership clause added by the search-mode wrap.
- `UserQueryGenerator` **runs** for PORTAL (it is not skipped — the skip at `DefaultProductsFilterQueryBuilder.java:105-108` only fires for `DASHBOARD_SEARCH`). It applies `productsFilter.getOwnerId()` from the body as an `ownerId` term (`UserQueryGenerator.java:16-19`).
- **Verdict:** the public "this seller's listings" view is driven by a **client-supplied** `ownerId` in the body. It is **not** derived from the principal. This is intentional (public profile browsing) and safe **because** PORTAL hard-pins `ACTIVE`+`APPROVED` (Q3), so a client can only ever enumerate another user's already-public listings. See trust-boundary section.

### DASHBOARD_SEARCH — `ownerId` **server-derived from the authenticated principal**; body `ownerId` cannot contribute

- `UserQueryGenerator` is **skipped** for DASHBOARD: `DefaultProductsFilterQueryBuilder.java:105-108`:
  ```java
  if (productsSearchMode == ProductsSearchMode.DASHBOARD_SEARCH
      && generator instanceof UserQueryGenerator) {
    return;
  }
  ```
- Ownership is pinned by `SearchModeQueryGenerator.java:21-26`:
  ```java
  case DASHBOARD_SEARCH -> {
    Long userId = currentUserService.getCurrentUserId().orElseThrow();
    queryBuilder.must(QueryBuilders.term(t -> t.field("ownerId").value(userId)));
  }
  ```
  `getCurrentUserId()` reads from the security context; `.orElseThrow()` fails loud when unauthenticated. **Clean** — body `ownerId` is structurally excluded.

### ADMIN_SEARCH — `ownerId` taken from the **request body**; only a role check is enforced

- `SearchModeQueryGenerator.java:28-32`:
  ```java
  case ADMIN_SEARCH -> {
    if (!currentUserService.isCurrentUserAdmin()) {
      throw new IllegalStateException();
    }
  }
  ```
  No ownership term added here; it asserts the caller is admin.
- `UserQueryGenerator` **runs** for ADMIN (not skipped), so a body `ownerId` filters to that owner — an admin may scope to any user's products. **Clean conditional on the admin role gate** (which is enforced both here and by `@PreAuthorize("hasRole('ADMIN')")` on the controller — see trust-boundary section).

---

## Q5 — `applyRandom` suppression

`DefaultProductsFilterQueryBuilder.applyRandomIfNeeded(...)` — `src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/DefaultProductsFilterQueryBuilder.java:122-145`. Invoked only from `buildQuery(...)` `:80` (the full-search path; the single-id and multi-id query paths never randomize).

```java
private Query applyRandomIfNeeded(Query baseQuery, ProductsFilterDTO productsFilter) {
  if (productsFilter == null || !productsFilter.isApplyRandom()) {   // :123-125
    return baseQuery;
  }
  String searchText = productsFilter.getSearchText();               // :127
  boolean hasSearchText = searchText != null && !searchText.isBlank();
  if (hasSearchText || productsFilter.getOrderBy() != null) {       // :129-131
    return baseQuery;
  }
  return Query.of(/* function_score random_score, seed = randomSeed, field = "id", boost_mode = Replace */); // :133-144
}
```

**Random wrap is APPLIED only when ALL of these hold:**
1. `productsFilter != null` (`:123`)
2. `productsFilter.isApplyRandom() == true` (`:123` — client opt-in flag)
3. `searchText` is null or blank (`:127-130`)
4. `orderBy == null` (`:129`)

**Random wrap is SKIPPED when any of:** filter is null, `applyRandom` is false, **`searchText` is non-blank**, **or `orderBy` is set**.

**Confirmation for the mobile decision:** the backend **already** independently suppresses random scoring when `searchText` is non-blank (`:128-130`) and when `orderBy` is set (`:129`). The suppression does not depend on the client clearing `applyRandom`. So the mobile-side fix is a client correctness/perf change, **not** a behavior change — the backend enforces the same outcome regardless.

(Random seed is `productsFilter.getRandomSeed()`, scored over the `id` field with `boost_mode = Replace`, `:140-144`.)

---

## Trust-boundary pass (conventions Part 11)

Fresh pass on the search/filter request path. Request DTO is `SearchProductsDTO { ProductsFilterDTO productsFilter; PagingDTO paging; }` (`src/main/java/com/memento/tech/oglasino/dto/SearchProductsDTO.java:6-9`). Search mode is **not** a request field — it is hard-coded per endpoint by the controller, so a client cannot choose its own mode:
- PORTAL: `ProductSearchController.java:42,60,73` (path `/api/public/product/search`).
- DASHBOARD: `DashboardProductController.java:78,156` (path `/api/secure/products`).
- ADMIN: `AdminProductSearchController.java:33,47,55` (path `/api/secure/admin/products`, class-level `@PreAuthorize("hasRole('ADMIN')")` at `:24`).

| Input | Source | Drives an authz / moderation / state-transition decision? | Deciding value server-derived or client-supplied? | Verdict |
| --- | --- | --- | --- | --- |
| `ownerId` (body, `ProductsFilterDTO:39`) — **PORTAL** | client body | No. Only narrows results within the already-`ACTIVE`+`APPROVED`-pinned public set. | client-supplied, but only ever enumerates public listings | **clean** — public-by-design; the moderation/state gate is server-pinned (Q3 PORTAL) independent of `ownerId`. |
| `ownerId` (body) — **DASHBOARD** | ignored | Yes (ownership scoping). | server-derived: `currentUserService.getCurrentUserId().orElseThrow()` (`SearchModeQueryGenerator.java:24`); body `ownerId` structurally skipped (`DefaultProductsFilterQueryBuilder.java:105-108`). | **clean** — client cannot scope to another user. |
| `ownerId` (body) — **ADMIN** | client body | Yes (admin chooses which owner). | client-supplied, gated by admin-role check (`SearchModeQueryGenerator.java:29` + `@PreAuthorize` `AdminProductSearchController.java:24`). | **clean-conditional-on:** the `hasRole('ADMIN')` gate. |
| path-segment user IDs | n/a | — | the search endpoints take **no** user id in the URL path (`@RequestBody` only; product-detail GETs take `productId`, not a user id). | **clean** — no path-borne ownership input on the filter path. |
| `productStates` (body, `ProductsFilterDTO:20`) | client body | Visibility/moderation scoping. | **PORTAL:** ignored, server hard-pins `ACTIVE` (`ProductStateQueryGenerator.java:54`). **DASHBOARD:** intersected server-side with `{ACTIVE, INACTIVE}` (`:96-101`). **ADMIN:** trusted, role-gated. | **clean** — never escapes server policy on the public surface; bounded on dashboard; role-gated for admin. |
| `moderationStates` (body, `ProductsFilterDTO:21`) | client body | Moderation-visibility scoping. | **PORTAL:** ignored, hard-pinned `APPROVED` (`:55`). **DASHBOARD:** passed through but combined with server-pinned `ownerId`, so only re-filters the caller's own products. **ADMIN:** trusted, role-gated. | **clean** — a leftover/forged `moderationStates` cannot reveal another user's `BANNED` products on any non-admin surface. |
| `baseSite` / `X-Base-Site` | client header `X-Base-Site`, else `baseSite` query param (`BaseSiteFilter.java:25-28`) | Result partitioning (which marketplace). Not a per-user authz/moderation/state-transition decision. | client supplies a **code**; the server resolves it to a `BaseSiteDTO` via `baseSiteCacheService.getBaseSiteForCode(...)` (`BaseSiteFilter.java:31`) and filters on the **resolved** `getId()` (`BaseSiteQueryGenerator.java:18-23`). Unknown code → null → PORTAL search endpoints reject with `BASE_SITE_MISSING_OR_INVALID` (`ProductSearchController.java:55-57,67-69`). | **clean** — a value the client cannot misrepresent (validated FK against server data); base-site selection is intended public behavior, not an authorization boundary. |

No client-supplied "before/previous" value participates in any change-detection on this path (Part 11's forbidden pattern is absent — the filter path performs no state transitions).

**Overall verdict: clean.** Every input that could scope or gate results is either ignored on the public surface, server-derived, server-bounded, or role-gated. No trust-boundary concern found on the search/filter path.

---

## Also report

### `ProductsFilterDTO` field list (as it exists today)

`src/main/java/com/memento/tech/oglasino/dto/ProductsFilterDTO.java:8-21`:

| Field | Type |
| --- | --- |
| `applyRandom` | `boolean` |
| `randomSeed` | `String` |
| `ownerId` | `Long` |
| `topCategoryId` | `Long` |
| `subCategoryId` | `Long` |
| `finalCategoryId` | `Long` |
| `excludeIds` | `List<String>` |
| `searchText` | `String` |
| `filters` | `List<RequestSelectedFilterDTO>` |
| `orderBy` | `OrderTypeDTO` |
| `priceRange` | `PriceRangeFilterDTO` |
| `selectedRegionAndCityValues` | `SelectedRegionsAndCitiesFilterDTO` |
| `productStates` | `List<ProductState>` |
| `moderationStates` | `List<ModerationState>` |

Wrapper `SearchProductsDTO` adds `paging` (`PagingDTO`) alongside `productsFilter`.

### Default page size & indexing (0-indexed?)

- **Page indexing is 0-based.** `PagingDTO.page` defaults to `0` (`src/main/java/com/memento/tech/oglasino/dto/PagingDTO.java:10`), passed straight to `PageRequest.of(page, …)` (`:30`).
- **Default page size when `paging` is supplied:** `perPage` defaults to **20** (`PagingDTO.java:11`), capped to `[1, 100]` by `cappedPerPage()` (`:37-39`, `MAX_PER_PAGE = 100` at `:8`).
- **Default page size when `paging` is omitted entirely:** **10** — `SearchProductsDTO.getPagingSafe()` returns `Pageable.ofSize(10)` (`SearchProductsDTO.java:27-33`). Note the mismatch: omitting `paging` yields size 10, whereas supplying an empty `PagingDTO` yields size 20. (Low-severity inconsistency — see below.)

### Adjacent observations (Part 4b) — file path + one line + severity + out-of-scope

1. **Stale Javadoc references non-existent states.** `ProductStateQueryGenerator.java:26` says "Public callers must never query PENDING / REJECTED / INACTIVE products," but `ProductState` has no `PENDING` or `REJECTED` (Q1). Misleads a future reader about the enum. **Severity: low.** Out of audit scope.

2. **`getProductStatesFieldValues` ignores its `allowed` parameter in the intersection.** `ProductStateQueryGenerator.java:96-101` intersects against the hard-coded constant `ALLOWED_DASHBOARD_PRODUCT_STATES::contains` rather than the `allowed` parameter it was passed. Behaviorally identical today (the only non-null caller passes exactly that constant), but the parameter is a latent lie — a second caller passing a different `allowed` set would be silently ignored. **Severity: low.** Out of audit scope.

3. **Default page size differs by request shape.** Omitting `paging` → size 10 (`SearchProductsDTO.java:31`); sending an empty `PagingDTO` → size 20 (`PagingDTO.java:11`). Two "defaults" for the same concept; mobile should send an explicit `paging` to be deterministic. **Severity: low.** Out of audit scope.

4. **PORTAL product-detail GET does not require a base site.** `ProductSearchController.getProductDetails` (`:38-49`) has no `BASE_SITE_MISSING_OR_INVALID` guard, unlike the `autocomplete`/`getAllProducts` POSTs (`:55-57,67-69`). A single-product fetch is keyed by `productId` and still PORTAL-pinned to `ACTIVE`+`APPROVED`, so this is not a leak — just an asymmetry. **Severity: low.** Out of audit scope.

Nothing found contradicts Q1–Q5.
