# Issues

Append-only log of out-of-scope findings. Newest at the top. Each entry has a date, severity, status, and a short body. Status values: `open`, `fixed`, `wontfix`, `parked`.

---

## 2026-05-15 — Favorites page has brittle positional type-narrowing on initialProductsData
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(protected)/favorites/page.tsx:52,64`
**Detail:** The page accesses `initialProductsData.products?.map(...)` where `initialProductsData` is typed `ProductOverviewsDTO | undefined`. The preceding `if (loading) return <LoadingOverlay />` guard prevents an actual undefined deref at runtime (`getFavorites` always resolves via `EMPTY_OVERVIEWS` on error, and `.then` runs before `.finally`), but the narrowing is purely positional and brittle to any refactor that moves the access or the guard. Type-tightening cleanup, no user-visible bug. Found by Docs/QA during the QA-Preparation Notifications + Favorites batch.

---

## 2026-05-15 — Favorites "recently viewed" carousel is misnamed
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(protected)/favorites/page.tsx:50`
**Detail:** The bottom carousel uses the `recently.viewed.title` translated heading, but its filter is `{ excludeIds: <favorited-ids> }` — no "recently viewed" criterion is sent. The heading promises recently-viewed products; the filter delivers something else. Same family of label/filter mismatch as the `user-page` "Similar products" carousel (documented there as a pitfall of the unfinished extra-products feature). If the extra-products feature is still WIP, this may be a known rough edge rather than a defect to fix — triage accordingly. Found by Docs/QA during the QA-Preparation Notifications + Favorites batch.

---

## 2026-05-14 — Owner-view hydration flash on UserDetails action buttons
**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/client/UserDetails.tsx:51-57`
**Detail:** `iamActive` is computed via `useMemo` over the auth store. On first paint after hydration, a signed-in owner viewing their own `/user/<id>` briefly sees the Follow / Send-Message / Report buttons before they disappear once the auth store resolves. Same pattern as the open row "Owner-view hydration flash on the ProductFunctions bar" (`ProductFunctions.tsx:36-44`) but a separate component — a fix touches `UserDetails` independently. Found by Docs/QA during the QA-Preparation public User page authoring.

---

## [HIGH] ProductErrorCode is a dumping ground for cross-cutting error codes

**Status:** open
**Surfaced:** 2026-05-14, connection-pool BugFixer chat (CurrentLanguageFilter Phase 2 work)
**Severity:** high — not a runtime bug, but a structural problem that actively misleads. The "Cross-cutting (system-level)" group inside `ProductErrorCode` is a precedent that gets cited and copied; every new cross-cutting code added there makes the next misplacement look more justified.

### The problem

`ProductErrorCode` contains error codes that have nothing to do with products. It already has an explicit "Cross-cutting (system-level)" group, and known occupants include:

- the rate-limit code consumed by `RateLimitFilter`
- `LANG_MISSING_OR_INVALID`, added during this chat by `CurrentLanguageFilter` Phase 2 — the engineer correctly flagged the placement felt wrong but followed the existing precedent because no proper home existed

There may be others — a full audit of `ProductErrorCode.values()` is the first step.

### Why it wasn't fixed in the chat that surfaced it

The chat that introduced `LANG_MISSING_OR_INVALID` is a BugFixer chat. Fixing _only_ the one code we added would leave `ProductErrorCode` inconsistently cleaned — some cross-cutting codes moved, others not — which is worse than leaving it whole, because it removes even the bad-but-consistent pattern. The honest fix is the full refactor: create a proper cross-cutting error-code enum and move _all_ non-product cross-cutting codes out. That refactor touches every consumer of the moved codes, the error-contract wiring, the `ProductErrorCodeTest` seed-row enforcement (which iterates `ProductErrorCode.values()`), and the translation seed files. That is feature-sized work, out of scope for a bug-fixer chat — so it is being routed here deliberately, not parked to dodge it.

### What the fix involves (for whoever picks this up)

1. Audit `ProductErrorCode` — list every constant that is not genuinely product-specific.
2. Decide the target enum name and create it (candidates discussed but not settled: `GeneralErrorCode`, `SystemErrorCode`, `RequestErrorCode`). Confirm whether any suitable enum already exists before creating one.
3. Move all non-product cross-cutting codes to the new enum.
4. Update every consumer — `RateLimitFilter`, `CurrentLanguageFilter`, and whatever else references the moved codes.
5. Update `ProductErrorCodeTest` — and check whether an equivalent seed-enforcement test should exist for the new enum.
6. Translation seed rows for the moved codes: confirm whether the `translationKey` values change (if the keys stay `product.system.*` that's itself a smell worth fixing; if they become `system.*` / `general.*` the seed rows move/rename).
7. `./mvnw spotless:check` + `./mvnw test` green.

### Known scope notes

- `LANG_MISSING_OR_INVALID` currently lives in `ProductErrorCode` with translation key `product.system.lang_missing_or_invalid` and seed rows in all four locale files (`0001-data-web-translations-{EN,RS,RU,CNR}.sql`). The `product.system.*` key prefix on a non-product code is part of the same smell.
- This is a refactor, not a behavior change — done right, no error response changes shape or code value from the client's perspective except the codes' _enum home_ (and possibly their `translationKey` if you choose to rename).

---

## 2026-05-14 — NumberOfViews fires a redundant duplicate fetch

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/client/NumberOfViews.tsx:11-13`
**Detail:** The view-count `useEffect` depends on `[numberOfViews]`. It fires once with state at 0, fetches, then `setNumberOfViews(res)` changes the dependency and it fires again — two requests per render for one value, identical responses. No user-visible bug, perf-only. Fix: change the dependency to `[productId]`. Found by Docs/QA during the QA-Preparation Product Detail page authoring.

---

## 2026-05-14 — Hardcoded Serbian tooltips on the product detail page

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/server/ProductDetails.tsx:72`, `oglasino-web/src/components/client/NumberOfViews.tsx:16`
**Detail:** The favorites-count and views-count tooltips use literal Serbian strings (`"Koliko je puta proizvod sacuvan"`, `"Koliko je puta proizvod pogledan"`) instead of next-intl keys — EN / RU / ME visitors see untranslated Serbian. A Part 6 (translations) violation in `oglasino-web` source. Found by Docs/QA during the QA-Preparation Product Detail page authoring.

---

## 2026-05-14 — Owner-view hydration flash on the ProductFunctions bar

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/client/ProductFunctions.tsx:36-44`
**Detail:** The actions bar's visibility is gated on a `useEffect`-driven `allow` flag (`setAllow(user.id === owner.id)`). On first paint after hydration the signed-in owner briefly sees the Call / Message / Favorite / Share / Review / Report bar before it disappears once the auth store resolves. The owner/stranger boundary itself is correct (captured as a topic pitfall); the flash is unintended. Fix: gate the bar with the auth selector synchronously, or render null while auth is loading. Found by Docs/QA during the QA-Preparation Product Detail page authoring.

---

## 2026-05-14 — Messages "Report" dropdown item is inert

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/messages/components/Messages.tsx:158-160`
**Detail:** The conversation-header kebab dropdown renders a "Report" item (`tMessages('func.report')` label) with no `onClick` handler — visible to every user who opens the dropdown, does nothing on click. May fall under the `state.md` backlog item "Chat & messaging cleanup" (`planned`), but no `issues.md` entry recorded it. Found by Docs/QA during the QA-Preparation Messages page authoring.

---

## 2026-05-14 — Messages chat list capped at 15 in the rendered UI

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/messages/components/Chats.tsx`
**Detail:** The conversation list renders only the most-recent Firestore page (limit 15). The chat store implements `loadMoreChats` and tracks `hasMoreChats` / `loadingMoreChats`, but `Chats.tsx` renders no "load more" affordance. A user with more than 15 conversations only ever sees the 15 most-recent by `lastUpdated`; older threads resurface only when the other party sends a new message that bumps `lastUpdated`. Invisible data-loss in the UX — the store supports paging, the UI doesn't expose it. Found by Docs/QA during the QA-Preparation Messages page authoring.

---

## 2026-05-14 — Messages silent text-only send failure

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/messages/store/useChatStore.ts` (`sendMessage` catch branch)
**Detail:** A Firestore write failure on a text-only message send logs `console.error`, runs orphan-image cleanup (a no-op for text), and rolls back the optimistic message by filtering out its tempId. No toast, no inline error, no surfaced state — the sender sees the message simply vanish. Image-upload failures in `MessageInput` do surface as `notify.error` toasts, so this is an asymmetry: a critical action fails silently on the text path. Found by Docs/QA during the QA-Preparation Messages page authoring.

---

## 2026-05-14 — Privacy and Terms render English markdown across all locales

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/privacy/page.tsx:14` (and the terms equivalent)
**Detail:** Both legal pages fetch a hardcoded GitHub raw markdown URL (`memento-tech/oglasino-platform/main/privacy.md` / `terms.md`) with no locale variant — every locale (EN / SR / RU / ME) renders the same English content. `privacy/page.tsx` carries a `// TODO Create locale privacy.md`. Found by Docs/QA during the QA-Preparation simple-pages batch. Surfaced to QA users as a "Known issue" pitfall on the `privacy-page` and `terms-page` QA topics; this entry tracks the underlying code fix.

---

## 2026-05-14 — `markdown.fild.load` translation key is a typo

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/src/components/server/MarkdownViewer.tsx:18,22`
**Detail:** `MarkdownViewer` references `tErrors('markdown.fild.load')` — "fild" is almost certainly meant to be "file". The rendered string is whatever the translation row says, so there is no user-visible break unless the row is also misnamed, but the key should be corrected across its registration and all locales. Found by Docs/QA during the QA-Preparation simple-pages batch.

---

## 2026-05-14 — Terms page does not emit JSON-LD; Privacy page does

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/terms/page.tsx`
**Detail:** `privacy/page.tsx` emits a JSON-LD `<script>` tag for SEO; `terms/page.tsx` does not. The two legal pages are otherwise the same shape. SEO inconsistency — either Terms should emit JSON-LD too, or the difference should be deliberate and documented. Found by Docs/QA during the QA-Preparation simple-pages batch.

---

## 2026-05-14 — Pricing hero renders the same translation key on both subtitle lines

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/pricing/page.tsx:32-34`
**Detail:** The hero H1 calls `tPricing('hero.subtitle.line1')` on both of its two lines — almost certainly meant to be `line1` then `line2`. Result is a visible duplicate line under the hero title in all locales. Found by Docs/QA during the QA-Preparation simple-pages batch.

---

## 2026-05-14 — Bad catalog slug produces two different failure modes depending on position

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web` — catalog route `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx` and its slug-resolution path
**Detail:** A catalog slug that does not resolve appears to produce two different observable outcomes depending on where it sits in the URL: an invalid first slug server-redirects to home, while a partially-broken breadcrumb chain has a client-side fallback to `/error`. Same root cause — a slug that doesn't resolve — two different results. **Suspected, not reproduced** — found by code-reading during the QA-Preparation Catalog page audit, exact locations not pinned. Intended behavior per Igor: messing with catalog routing should show a "category not found" error, consistently. Same root cause should produce the same outcome. Related to the silent-degradation entry below — same root cause, different observable outcome. Next step: confirm against a running stack before scoping a fix.

---

## 2026-05-14 — Invalid catalog sub-slug degrades silently to parent category

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web` — catalog route `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx` and its slug-resolution path
**Detail:** An invalid second or third URL slug on the catalog route appears to degrade silently to the parent category level — `/catalog/cars/bogus` renders identically to `/catalog/cars`, with no error, no redirect, no "category not found". A bad sub-slug produces a result that looks like the parent category, which can read as the parent filter being broken. **Suspected, not reproduced** — found by code-reading during the QA-Preparation Catalog page audit, exact resolution logic not pinned. Intended behavior per Igor: a bad slug at any level should show a "category not found" error, not silently fall back to the parent. Related to the two-failure-modes entry above — same root cause. Next step: confirm against a running stack before scoping a fix.

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
**Status:** fixed
**Found in:** `oglasino-docs/meta/conventions.md` Part 9 (stack table) and Part 11 (trust-boundary text)
**Detail:** Part 9's stack table and Part 11's trust-boundary text both describe auth as JWT-based. The actual system uses Firebase auth as the source of truth (Firebase-context-derived identity server-side), confirmed by the backend filtering audit (`SearchModeQueryGenerator.DASHBOARD_SEARCH` reads from `SecurityContextHolder` populated by `FirebaseAuthFilter`). Same class of error as the recently-corrected Gradle/Maven mismatch — a convention that contradicts reality and will mislead a future agent. Needs an Igor-and-Mastermind correction pass on Parts 9 and 11. Discovered in the product-filtering bug sweep. **Resolved 2026-05-15:** Parts 9 and 11 corrections drafted in `.agent/2026-05-15-oglasino-docs-connection-pool-closeout-1.md` (For Mastermind section) and pending Igor's apply to `conventions.md` — Docs/QA does not edit `conventions.md` per CLAUDE.md hard rule. The drafted change replaces the JWT-mechanism framing with Firebase ID token + `SecurityContextHolder`-populated `OglasinoAuthentication`, using the precise codebase terms (`FirebaseAuthFilter`, `redisUserAuth`).

## 2026-05-14 — Product-validation feature doc needs a retro-fit pass against current conventions

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-docs/features/product-validation.md`
**Detail:** Written before the `### Images` convention (Part 1) and the Part 5 session-naming change existed. It should be brought in line with the current rulebook. Not urgent — tagged for a future docs session, not part of any active work. **Resolved 2026-05-15:** audited against current conventions during the connection-pool close-out session. Part 1 `### Images` is satisfied — 4 image references at lines 417, 465, 505, 566 all use kebab-case lowercase descriptive filenames under `assets/`, all have a preceding HTML comment describing the intended screenshot, and all use standard markdown syntax with descriptive alt text. Part 5 (session-file naming) governs `.agent/` summary files, not feature-spec docs, so it is N/A for this file. No retro-fit edits needed.

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
