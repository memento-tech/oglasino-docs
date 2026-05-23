# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-20
**Task:** Audit brief — perPage server-side cap: current state + recommended fix

This is a read-only audit per the brief. No code edits. Output is the audit summary below.

## Audit

### 1. Current state — `PagingDTO.getPageable()`

`src/main/java/com/memento/tech/oglasino/dto/PagingDTO.java` in full:

```java
package com.memento.tech.oglasino.dto;

import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;

public class PagingDTO {
  private int page = 0;
  private int perPage = 20;

  public int getPage() { return page; }
  public void setPage(int page) { this.page = page; }
  public int getPerPage() { return perPage; }
  public void setPerPage(int perPage) { this.perPage = perPage; }

  public Pageable getPageable() {
    return PageRequest.of(page, perPage);
  }

  public Pageable getPageable(Sort sort) {
    return PageRequest.of(page, perPage, sort);
  }
}
```

- **Field types and nullability.** `page` and `perPage` are `int` primitives — non-nullable by language. Defaults: `page = 0`, `perPage = 20`. Jackson omits unsupplied fields, leaving the defaults.
- **Existing validation.** None. No Jakarta annotations (`@Min`, `@Max`, `@PositiveOrZero`), no manual checks, no clamping. The setters are plain assignment.
- **Behaviour on null / zero / negative `perPage`.**
  - Null is impossible — primitive `int`.
  - `perPage = 0` or `< 0` is not pre-handled; `PageRequest.of(page, size)` itself throws `IllegalArgumentException("Page size must not be less than one")`. That bubbles up; under the project's `GlobalExceptionHandler` this surfaces as an HTTP 500 (no specific mapping for `IllegalArgumentException`).
  - `perPage = Integer.MAX_VALUE` succeeds at `PageRequest.of` but blows up downstream in `PageImpl`/ES allocation — the abuse vector the brief is closing.

### 2. Callers of `PagingDTO.getPageable()`

Every callsite in `src/main/java`. None do their own `perPage` clamping or validation before calling.

| # | File | Method / line | Sort overload? | Endpoint / surface | Pre-call clamp |
|---|------|---------------|----------------|--------------------|----------------|
| 1 | `dto/SearchProductsDTO.java` | `getPagingSafe():32` | no-arg | wrapper used by all product-search / autocomplete (see §3a) | none |
| 2 | `admin/facade/impl/DefaultAdminReportFacade.java` | `:44` (`reportFilterRequest.getPageable(Sort.by(ASC,"createdAt"))`) | Sort | `AdminReportController` → admin reports list | none |
| 3 | `admin/service/impl/DefaultSuggestionService.java` | `:35` (`suggestionsFilterRequest.getPageable(Sort.by(DESC,"createdAt"))`) | Sort | admin suggestions list | none |
| 4 | `admin/service/impl/DefaultUsersService.java` | `:98` (`filterRequest.getPageable(Sort.by("id"))`) | Sort | admin users list (`UsersFilterRequestDTO extends PagingDTO`) | none |
| 5 | `admin/service/impl/DefaultAdminReviewService.java` | `:43` (`reviewFilterRequest.getPageable(Sort.by(ASC,"createdAt"))`) | Sort | admin reviews list | none |
| 6 | `controller/FavoriteProductsController.java` | `:34` (`paging.getPageable()`) | no-arg | `POST /api/secure/favorites/overviews` | none |
| 7 | `service/impl/DefaultConfigurationService.java` | `:62` (`filterRequest.getPageable()`, `ConfigFilterRequestDTO extends PagingDTO`) | no-arg | admin configuration list | none |
| 8 | `service/impl/DefaultFollowService.java` | `:49` (`paging.getPageable()`) | no-arg | `POST /api/secure/user/follows` (via `UserController`) | none |

All eight inherit a central cap installed in `PagingDTO.getPageable()` / `getPageable(Sort)`.

Subclasses of `PagingDTO` discovered while tracing callers (no overrides of `getPageable` in any): `ConfigFilterRequestDTO`, `ReportFilterRequestDTO`, `SuggestionsFilterRequestDTO`, `ReviewFilterRequestDTO`, `UsersFilterRequestDTO`. All inherit the future cap automatically.

### 3. Direct `PageRequest.of(...)` call sites elsewhere in `src/main/java`

These do **not** route through `getPageable()`. Each is classified.

#### 3a. Inside `PagingDTO` and its safe wrapper

- `dto/PagingDTO.java:28` and `:32` — the two methods being changed. Size argument is `this.perPage`.
- `dto/SearchProductsDTO.java:29` — `return Pageable.ofSize(10);` — fallback used only when `paging == null` on a `SearchProductsDTO` request. Size is a hardcoded constant `10`. Not client-supplied. Not a perPage path. (Note: it's `Pageable.ofSize`, not `PageRequest.of`, but worth listing for completeness.)
- `dto/SearchProductsDTO.java` `getPagingSafe()` delegates to `paging.getPageable()` when `paging` is non-null, so the central cap covers all product-search / autocomplete endpoints (see brief-vs-reality note below).

#### 3b. Public / secure endpoints that bypass `getPageable()`

| File | Line | Surface | Size source | Page source | Notes |
|------|------|---------|-------------|-------------|-------|
| `controller/PublicReviewController.java` | `:33` | `GET /api/public/review` | hardcoded `20` | `@RequestParam int page` (client-supplied) | size is not client-supplied → cap doesn't apply by intent. Page index is unbounded but that's a deep-paging concern, out of scope here. |
| `service/impl/DefaultReviewService.java` | `:90` (`getMyReviews(int page, ...)`) | `GET /api/secure/review/product` (via `ReviewController:42-44`) | hardcoded `20` | client-supplied `page` | same shape as above. |

#### 3c. Internal / batch use (no client surface)

| File | Line | Size source | Trigger |
|------|------|-------------|---------|
| `jobs/ProductRemovalJob.java` | `:31` | `configurationService.getIntConfig("product.removal.batch.size")` | `@Scheduled` Sunday 02:00 |
| `jobs/UserDeletionScheduledJobs.java` | `:58` | `configurationService.getRequiredIntConfig("user.deletion.hard.delete.batch.size")` | `@Scheduled` daily 02:00 |
| `jobs/UserDeletionScheduledJobs.java` | `:134` | `configurationService.getRequiredIntConfig("user.deletion.firebase.reconciliation.page.size")` | `@Scheduled` Sunday 03:00 |
| `jobs/ProductBaseCurrencyUpdater.java` | `:61` | `BATCH_SIZE` constant | `@Scheduled` |
| `elasticsearch/service/impl/ProductIndexer.java` | `:142` | `elasticsearchProperties.getReindex().getBatchSize()` | internal reindex |

None of these read a size value from a client request body or query param. Cap is unnecessary for them.

### 4. Existing tests

- No tests live on `PagingDTO.getPageable()` directly. `grep -rn "PagingDTO\|getPageable" src/test/` returns one file: `admin/controller/UsersControllerAdminExtensionTest.java`, lines 252 and 270, which send `perPage=20` in the request body. Both are well under any sensible cap; behaviour unchanged.
- No test asserts behaviour on a large `perPage` value.

### 5. Brief vs reality

One discrepancy worth noting; it does **not** change the fix or block work — recording for accuracy.

1. **`DefaultProductsFilterQueryBuilder.buildQuery` does not call `paging.getPageable()` directly.**
   - Brief says: "`PagingDTO.getPageable()` → `PageRequest.of(page, perPage)`, called from `DefaultProductsFilterQueryBuilder.buildQuery`."
   - Code says: `DefaultProductsFilterQueryBuilder.java:85` calls `searchProducts.getPagingSafe()`. `SearchProductsDTO.getPagingSafe()` (lines 27–33) delegates to `paging.getPageable()` when `paging != null`, else returns `Pageable.ofSize(10)`.
   - Why this matters: only a callout. The central-cap assumption in the brief still holds — `getPagingSafe()` routes through `getPageable()`, and the null-fallback uses a hardcoded `10`, not a client value. The cap inherits transparently.

2. **The brief's "every paginated endpoint inherits" line is not literally true.** `PublicReviewController` and `ReviewController.getMyReviews` build `PageRequest.of` directly with a hardcoded `size=20`. They don't need the cap (size isn't client-supplied), but it's worth recording so the issues.md entry doesn't over-promise.

Both observations were captured here rather than as a blocking challenge, because neither alters the cap design.

## Proposed fix

### Fix shape

In `dto/PagingDTO.java`, add a private constant and clamp the upper bound inside both `getPageable()` overloads:

```java
private static final int MAX_PER_PAGE = 100;

public Pageable getPageable() {
  return PageRequest.of(page, Math.min(perPage, MAX_PER_PAGE));
}

public Pageable getPageable(Sort sort) {
  return PageRequest.of(page, Math.min(perPage, MAX_PER_PAGE), sort);
}
```

- **What the clamp does.** Silently caps any `perPage > 100` at `100`. Wire contract unchanged.
- **Zero / negative.** Recommend **leaving the lower-bound behaviour alone** — `PageRequest.of` already throws `IllegalArgumentException` on `size < 1`. The brief's stated intent is hygiene against client-driven page-size blow-ups, not malformed-request normalisation; a silent coercion of `0` or `-5` to `1` would mask a clearly bogus client request and was not part of the decision. If Igor wants those normalised, the simplest extension is `Math.max(1, Math.min(perPage, MAX_PER_PAGE))` — flagged here, not assumed.
- **No-args overload.** The two methods share the same expression; a tiny private helper (`private int cappedSize() { return Math.min(perPage, MAX_PER_PAGE); }`) would dedupe one line and one fewer place to drift on a future change. Optional — happy to inline both call sites instead. Bias is toward Part 4a's "match surrounding style"; the current file is plain getters with no helpers.

### Cap value handling

- Cap = 100 per Igor's decision. Default to **`private static final int MAX_PER_PAGE = 100;` inside `PagingDTO`**.
- No other layer needs to reference the constant — there is no existing `PagingConstants` file, no `application.yaml` knob, no caller that branches on the cap value. Keeping it private avoids inventing a new shared symbol for a single-use number.
- If Igor wants it environment-configurable later, the move to `@Value` is a one-line edit and doesn't change the call sites. Not done now.

### Internal-call exposure

Items in §3b and §3c above that bypass `getPageable()`. None require capping under the brief's scope:

- `PublicReviewController:33` and `DefaultReviewService:90` — `size` is the constant `20`, not client-supplied. Flag only.
- `ProductRemovalJob`, `UserDeletionScheduledJobs`, `ProductBaseCurrencyUpdater`, `ProductIndexer` — internal cron / batch, sizes from config or constants. Not exposed to clients. Flag only.

If Igor wants these unified through a central cap as future hygiene, that's a separate brief and would likely live as a `PagingConstants.MAX_PER_PAGE` referenced by both `PagingDTO` and the review controllers/services. Not proposed now.

### Risks and regressions

- **Web / mobile clients.** Web product-search pages today use page sizes that fit standard product grids (e.g. 20, 50). No survey of every web/mobile caller was done — the audit is backend-only. If any UI today legitimately requests `perPage > 100`, it would silently degrade. Worth a single check by Igor (or a routed brief to the web agent) before applying. Most likely candidates to verify: admin lists (users, reviews, suggestions, reports), which can fetch larger batches than portal grids.
- **Tests.** Two existing test fixtures send `perPage=20`. Unaffected. No test exists today that asserts a specific large `perPage` value.
- **`PageRequest.of` already throws on `perPage < 1`** — that error path stays as it is under the recommended clamp. If a client today is sending `perPage = 0` and getting a 500, this change does not improve or worsen that.
- **`SearchProductsDTO.getPagingSafe()` null-fallback** (`Pageable.ofSize(10)`) is untouched.

### Test plan

A focused unit test on `PagingDTO` covers the cap end-to-end because every caller routes through it.

- `getPageable()` returns size `perPage` when `1 <= perPage <= 100`.
- `getPageable()` returns size `100` when `perPage` is `101`, `500`, `Integer.MAX_VALUE`.
- `getPageable(Sort)` matches the no-arg behaviour and preserves the Sort.
- Boundary case: `perPage = 100` returns `100` (cap is inclusive).
- Existing-default case: with no setter call, `getPageable()` returns size `20`.
- (Optional, only if Igor opts for lower-bound coercion) `perPage = 0` and `perPage = -1` cases.

No new integration tests required — controller tests already exercise `perPage=20` and remain green.

## Files touched

- (none — read-only audit)

## Tests

- Ran: (none — read-only audit)
- Result: n/a
- New tests added: none

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change (Igor's "cap at 100, silent clamp, central in `getPageable()`" decision is already captured in the brief; if Mastermind wants it formalised, that's a draft for them to produce — not for this audit)
- state.md: no change
- issues.md: no change. The 2026-05-14 entry "`perPage` is uncapped server-side" stays open until the implementation session closes; the entry's body could optionally be tightened — it names `DefaultProductsFilterQueryBuilder.buildQuery` but the actual chain goes through `SearchProductsDTO.getPagingSafe()` first. Surface in "For Mastermind."

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): N/A — read-only
- Part 4a (simplicity): N/A — brief explicitly waived the structured evidence for read-only audits
- Part 4b (adjacent observations): two observations recorded in "Brief vs reality" and in "For Mastermind" below
- Part 11 (trust boundaries): touched lightly — `perPage` is client-supplied and the cap is the trust-boundary enforcement; confirmed
- Other parts touched: Part 7 — confirmed the brief's no-400-no-error-code-silent-clamp choice is consistent with the wire-shape rules (silent clamp keeps the response shape stable)

## Known gaps / TODOs

- Web / mobile audit of "does any UI today send `perPage > 100`" — out of scope for this backend audit. Flagged for Igor / Mastermind to route if desired before the implementation brief.

## For Mastermind

- **Two brief-vs-reality observations** (do not block the fix):
  - `DefaultProductsFilterQueryBuilder.buildQuery` calls `SearchProductsDTO.getPagingSafe()`, not `paging.getPageable()` directly. The cap still inherits through `getPagingSafe()` → `paging.getPageable()` on the non-null path. Severity: low. The brief and the issues.md entry both name `buildQuery` as the direct caller — accurate enough for hygiene purposes, but worth tightening when the issue entry is closed.
  - The brief states "every paginated endpoint inherits" — verified incorrect for `PublicReviewController:33` and `ReviewController.getMyReviews` (via `DefaultReviewService:90`). Both hardcode `size=20`, so they don't need the cap, but the inheritance claim is not literal. Severity: low.
- **Suggested issue-entry tightening (optional).** When the 2026-05-14 `issues.md` entry flips to `fixed`, the "called from `DefaultProductsFilterQueryBuilder.buildQuery`" line could be amended to "called from `DefaultProductsFilterQueryBuilder.buildQuery` via `SearchProductsDTO.getPagingSafe()`." Routing this through Docs/QA on the close-out.
- **Open question for Igor before implementation.**
  - Lower-bound coercion: should the clamp also normalise `perPage <= 0` to `1` (silent), or leave the existing `IllegalArgumentException` path alone? Audit's recommendation is "leave alone" — the brief is scoped to the upper-bound abuse vector — but flagging explicitly so the implementation brief can confirm.
  - Helper extraction: introduce `private int cappedSize()` to dedupe the `Math.min` in both overloads, or inline both? Audit's bias is "inline both" — Part 4a, matching the file's current no-helpers style — but trivial either way.
- **Adjacent observations** (none warranting their own issues.md entry, all low):
  - `PublicReviewController:23-33` and `ReviewController.getMyReviews` use `@RequestParam int page` with no upper bound on page index. Not a `perPage` problem; not in scope for this entry; deep-paging cost is bounded by Elasticsearch defaults rather than by the controller. Severity: low. Not flagging as a new entry — recording here in case it surfaces again.
  - `FavoriteProductsController` has its endpoint methods declared `private` on a `@RestController` (`:25`, `:31`, `:38`). Spring still wires them because the proxy is generated from the bean, but the visibility is unusual and easy to misread as "not exposed." Not a perPage concern. Severity: low (style). Not fixing.
- **(No "For Mastermind" Part 4a structured evidence — read-only audit per brief.)**
