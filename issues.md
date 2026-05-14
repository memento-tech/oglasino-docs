# Issues

Append-only log of out-of-scope findings. Newest at the top. Each entry has a date, severity, status, and a short body. Status values: `open`, `fixed`, `wontfix`, `parked`.

---

## 2026-05-14 — `perPage` is uncapped server-side
**Severity:** medium
**Status:** open
**Found in:** `oglasino-backend` — `PagingDTO.getPageable()` → `PageRequest.of(page, perPage)`, called from `DefaultProductsFilterQueryBuilder.buildQuery`
**Detail:** All product-search endpoints (`/api/secure/products`, `/api/secure/admin/products`, `/api/public/product/search`, plus their three autocomplete siblings) pass client-supplied `perPage` straight to Spring Data with no maximum. `RateLimitFilter` mitigates abuse, but a hard cap (e.g. 100) is appropriate hygiene. Discovered in the product-filtering bug sweep (backend audit, Q3).

## 2026-05-14 — `PORTAL_SEARCH` silently drops body `productStates` / `moderationStates`
**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/ProductStateQueryGenerator.java` (`PORTAL_SEARCH` branch)
**Detail:** Public product search hard-pins `productState=ACTIVE AND moderationState=APPROVED` and ignores any client-supplied state filters. This is correct and intentional behavior — a public caller must not be able to request PENDING/REJECTED products — but the shared `ProductsFilterDTO` wire shape doesn't communicate that these fields are inert on the portal path. No code fix wanted; logged so the silent-drop is recorded. Discovered in the product-filtering bug sweep (backend audit, opportunistic flags).

## 2026-05-14 — `baseSite` tenant scoping was never audited
**Severity:** medium
**Status:** open
**Found in:** `oglasino-backend` — `BaseSiteQueryGenerator`, `BaseSiteContext`
**Detail:** `BaseSiteQueryGenerator` and `BaseSiteContext` were referenced but not opened during the product-filtering audit. Whether tenant isolation is header-derived or body-derived is unconfirmed. Needs its own focused audit; will likely become its own documentation page. Discovered in the product-filtering bug sweep (backend audit, out-of-scope note).

## 2026-05-14 — Backend errors are swallowed and rendered as "empty results"
**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/lib/service/nextCalls/productsSearchService.ts` (all three `getXProducts` calls)
**Detail:** Every product-search server call catches errors and returns `EMPTY_OVERVIEWS = { products: [], totalNumberOfProducts: 0 }`. A backend outage is therefore indistinguishable from a genuinely empty result set — the user sees "no products" either way. Pre-launch this is tolerable; for production it warrants at least server-side logging and ideally a distinguishable error state. Discovered in the product-filtering bug sweep (web audit, all surfaces).

## 2026-05-14 — No request cancellation on the pagination path
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/client/product/ProductList.tsx`, `src/lib/service/reactCalls/productsSearchService.ts` (paging branch)
**Detail:** `AbortController` was added to autocomplete during the sweep, but paging requests still have no cancellation — rapid page-button clicks can let a stale response resolve last and win. Deferred deliberately; would need a coordinated pattern across `ProductList` and the service layer. Discovered in the product-filtering bug sweep.

## 2026-05-14 — `parseFiltersFromQueryParams` has a dead `| undefined` return type
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/lib/utils/filtersHelper.ts:44`
**Detail:** The function is typed `ProductsFilterDTO | undefined` but no code path returns `undefined`; the home page's `else` branch handling that case (`app/[locale]/(portal)/(public)/page.tsx:38-43`) is unreachable. Dead code, no behavior impact — a type-tightening cleanup. Discovered in the product-filtering bug sweep (web session summary).

## 2026-05-14 — `/admin/products/[userId]` renders an empty fragment on zero products
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/admin/products/[userId]/page.tsx:34`
**Detail:** When an admin views a user with no products, the page returns `<></>` — no message, no context. Should show a "no products for this user" state. Discovered in the product-filtering bug sweep (web audit, Suspected Bug context).

## 2026-05-14 — `/admin/products/[userId]` filter chips change without refetch or URL sync
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/client/initializers/FilterManager.tsx:52-66` (`isAllowedPath()` excludes `/admin/products/[userId]`)
**Detail:** `/admin/products/[userId]` is excluded from `FilterManager.isAllowedPath()`, so client-side filter mutations on that page change the visible chip row but do not trigger a product refetch and do not sync to the URL. Filtering a single user's product list is not a supported use case (by design — confirmed during the sweep), but the chip-row-changes-without-refetch behavior is a visible inconsistency that will confuse a future reader or engineer. Independent of the empty-fragment entry above; same route. Discovered in the product-filtering bug sweep (web audit, Suspected Bug #2; left out of scope by the web fix brief).

## 2026-05-14 — `conventions.md` says `Auth: Firebase Auth (JWT)` and references "claims from the JWT"
**Severity:** medium
**Status:** open
**Found in:** `oglasino-docs/meta/conventions.md` Part 9 (stack table) and Part 11 (trust-boundary text)
**Detail:** Part 9's stack table and Part 11's trust-boundary text both describe auth as JWT-based. The actual system uses Firebase auth as the source of truth (Firebase-context-derived identity server-side), confirmed by the backend filtering audit (`SearchModeQueryGenerator.DASHBOARD_SEARCH` reads from `SecurityContextHolder` populated by `FirebaseAuthFilter`). Same class of error as the recently-corrected Gradle/Maven mismatch — a convention that contradicts reality and will mislead a future agent. Needs an Igor-and-Mastermind correction pass on Parts 9 and 11. Discovered in the product-filtering bug sweep.

## 2026-05-14 — Product-validation feature doc needs a retro-fit pass against current conventions
**Severity:** low
**Status:** open
**Found in:** `oglasino-docs/features/product-validation.md`
**Detail:** Written before the `### Images` convention (Part 1) and the Part 5 session-naming change existed. It should be brought in line with the current rulebook. Not urgent — tagged for a future docs session, not part of any active work.

## 2026-05-14 — Manual test reminder: pagination overlap across all five surfaces
**Severity:** low
**Status:** open
**Found in:** product-filtering bug sweep — manual QA queue
**Detail:** After the product-filtering web fixes, verify on a live stack that dashboard/admin/public-user pages 2+ show different products than page 1, on a scope with 25+ products. Igor ran an initial check during the sweep and it passed; this entry is the standing regression reminder. **Not a bug** — verification step that should not be lost. Surfaces in scope: home, category, owner dashboard, admin product list (including `[userId]`), public user-products.

## 2026-05-14 — Manual test reminder: random-ordering stability across pages
**Severity:** low
**Status:** open
**Found in:** product-filtering bug sweep — manual QA queue
**Detail:** Verify that paginating home/catalog with random ordering keeps a consistent shuffle across pages within one session, and reshuffles only on refresh (because `randomSeed` is regenerated per SSR call but stable across paging requests within one render). **Not a bug** — verification step.

## 2026-05-14 — Dead `free` field on backend `NewProductRequestDTO`
**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java` (field `private boolean free` plus `isFree()`/`setFree()`)
**Detail:** Web removed `free` from its `NewProductRequestDTO` interface in web session 4 (free-zone derived from `topCategory.freeZone` instead). Backend still carries the field with getter/setter; no production code reads it (grep across `src/main/java` shows zero call sites that consume `request.isFree()` or `requestDTO.isFree()` on a product request). Jackson tolerates the absent field on incoming JSON. Dead field worth deleting in a follow-up chore to match the trust-boundary direction (free-zone is server-derived).

## 2026-05-14 — `regionAndCity` declared in `createProductSchema` but unread by validator
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/lib/validators/productSchemas.ts`
**Detail:** `createProductSchema` still declares `regionAndCity: z.object({...}).nullable().optional()` but `validateProduct` does not consult that field — region and city are user-derived and not user-settable on the create wizard. Harmless as-is; cosmetic cleanup.

## 2026-05-14 — `field-price` scroll target is a no-op on update page
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/owner/products/[productId]/page.tsx`
**Detail:** When a price-only error fires on save, the page calls `document.getElementById('field-price')?.scrollIntoView(...)`, but the two price `<Input>` renders generate their own internal ids and do not accept an external `id`. Inline error still renders correctly next to the price input; only the scroll-into-view is a no-op. Fix is to wrap the price inputs in a `<div id="field-price">`.

## 2026-05-14 — Boot-time moderation config audit logs ERROR but does not fail boot
**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/moderation/ContentValidationConfig.java` (`auditRequiredConfig`)
**Detail:** Missing/blank required keys are surfaced at boot via `log.error(...)` but the listener does not throw — the first runtime moderation call against an absent key throws `IllegalStateException`. Acceptable for dev environments that may run with an un-seeded config table. A `@Profile`-gated boot-fail (or env-toggle) for production would convert the silent-misconfig window into a hard startup failure. Framed as enhancement, not defect.

## 2026-05-14 — Web component-render test coverage gap (testing-library not installed)
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/package.json`
**Detail:** `@testing-library/react`, `@testing-library/user-event`, `happy-dom`, and `jsdom` are all absent. Web tests are pure-function and axios-mock only — dialog-rendering assertions (button-disabled, spinner present, inline-error elements) are covered indirectly via logic-level tests against `validateProduct`, `preValidateProductBasics`, and `productStepMapping`. End-to-end smoke is the first surface that exercises the dialog rendering. Future infra chore: install the testing stack so dialog tests can land in thin follow-ups.

## 2026-05-14 — Keyword-stuffing ratio multipliers not lifted to config
**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/moderation/analyzer/KeywordStuffingAnalyzer.java`, `SpammyDescriptionAnalyzer.java`
**Detail:** Both analyzers pass a `KeywordStuffingDetector.Tuning` record with hardcoded `baseRatio`, `ratioDecayPerWord`, `ratioFloor`, and `minOccurrenceRatio` values. Other thresholds are admin-configurable via `ConfigurationService`; these heuristic ratios are not. Deliberately out of scope for the validation refactor — lifting them is a separate, larger conversation about how much of the heuristic should be admin-tunable.

## 2026-05-14 — Malformed 429 leaves create-wizard silently blocked
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/lib/service/reactCalls/productService.ts`
**Detail:** Web session 4 removed the defensive `ensureSystemErrorKey` helper that synthesised a `__system` rate-limit entry if the 429 response body was malformed. Trade-off was made deliberately (trust the contract, surface upstream bugs instead of papering over them). Consequence: a malformed 429 (no `field:null + translationKey` entry) leaves the create wizard's Next button blocked without showing the inline rate-limit reason — the user is not advanced through, just not told why. If a malformed 429 is ever observed in manual/QA testing, the fix belongs at the backend/transport source, not at the client.

## 2026-05-13 — Legacy unused regex rows in data-configuration.sql
**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/src/main/resources/data/configuration/data-configuration.sql` ids 2–7
**Detail:** Six pre-feature config rows (`validation.regex.banned.words`, `validation.regex.repeated.chars`, `validation.regex.punctuation`, `validation.regex.emojis`, `validation.regex.promo` without language suffix, `validation.regex.spam.description`) remain seeded but have zero call sites — superseded by the per-language `validation.regex.*.{lang}` and `validation.banned_words.{lang}` rows. Backend session 3 flagged these for Mastermind as a follow-up "config-seed garbage collection" PR. Safe to delete in a chore.

## 2026-05-13 — 429 rate-limit response shape diverges from error contract
**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/security/filter/RateLimitFilter.java`
**Detail:** Original report: on HTTP 429, `RateLimitFilter` returned `{"error": "rate_limit_exceeded"}` with a `Retry-After` header, mismatching the project's `{errors: [{field, code, translationKey}]}` contract. **Resolved in backend session 1**: filter now writes `{"errors":[{"field":null,"code":"RATE_LIMITED","translationKey":"product.system.rate_limited"}]}` (see `RATE_LIMITED_BODY` constant). The unified shape applies to all rate-limited endpoints.

## 2026-05-13 — Commented-out `REGION_REQUIRED` annotation in `NewProductRequestDTO`
**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java`
**Detail:** Original report: a commented-out `@NotNull(message = "REGION_REQUIRED")` line sat above the `filters` field. Region is derived from the user, not the request. **Resolved**: the commented line is no longer present in the file (deleted during one of the cleanup passes on the validation-refactor branch).

## 2026-05-13 — Pre-existing spotless violations on validation-refactor branch
**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java`, `oglasino-backend/src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java`
**Detail:** Two files on `feature/validation-refactor` failed `mvn spotless:check` before the pre-validate session began. One needed re-indentation of a commented line; the other had an unused `Region` import. Both fixed by Igor via `mvn spotless:apply` and committed separately under `chore: fix pre-existing spotless violations`.
