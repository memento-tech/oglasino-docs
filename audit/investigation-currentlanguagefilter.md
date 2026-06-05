# Investigation — `CurrentLanguageFilter` control flow + X-Lang handling

**Date:** 2026-05-14
**Phase:** 1 (read-only)
**Source file:** `src/main/java/com/memento/tech/oglasino/filter/CurrentLanguageFilter.java`

---

## 1a — The four named language-independent routes

Igor named four routes by shorthand. The exact mappings on disk:

| Shorthand          | Exact path                                          | Method | Controller:line                   |
| ------------------ | --------------------------------------------------- | ------ | --------------------------------- |
| `translations`     | `/api/public/translations`                          | GET    | `TranslationsController.java:17`  |
| `configuration`    | `/api/public/config`                                | GET    | `ConfigurationController.java:14` |
| `baseSiteOverview` | `/api/public/baseSite/overviews`                    | GET    | `BaseSiteController.java:54`      |
| `baseSiteData`     | `/api/public/baseSite/{targetBaseSiteCode}`         | GET    | `BaseSiteController.java:37`      |
| `baseSiteData`     | `/api/public/baseSite/details`                      | GET    | `BaseSiteController.java:46`      |

Notes:

- `configuration` is `/config`, not `/configuration` — the controller's `@RequestMapping` is `"/api/public/config"`. The frontend client name was likely the inspiration for "configuration" in the brief.
- `baseSiteData` is not a single named route. `BaseSiteController` exposes the full `BaseSiteDTO` via two distinct endpoints: `GET /{targetBaseSiteCode}` (one base site) and `GET /details` (all base sites). Both return the same DTO type. Frontend almost certainly hits one of these for "base site data."
- A fourth endpoint exists on the same controller — `GET /api/public/baseSite/regions/{targetBaseSiteCode}` (`BaseSiteController.java:61`) — returning the region list for a base site. It is structurally identical (language-independent reference data, served from `BaseSiteCacheService`). Treating it differently from the other three baseSite endpoints would be arbitrary.

**Implication for the allowlist:** every endpoint under `/api/public/baseSite/*` is language-independent. The clean answer is one prefix entry covering the subtree, not three or four exact entries. See 1c.

---

## 1b — Inventory of every public route

All routes mounted under `/api/public/*`. The "lang-dep?" column is the call I'd make from reading the code:

- `language-independent` = code path under this route does not touch `LanguageContext` and the response payload carries no translated content.
- `language-dependent` = code path consumes `LanguageContext.getCurrentLanguage*()` somewhere downstream of the controller.

| # | Path | Method | Controller:line | Lang-dep? | Why |
|---|------|--------|-----------------|-----------|-----|
| 1 | `/api/public/health/check` | GET | `HealthCheckController.java:12` | language-independent | empty 200, no DB, no services |
| 2 | `/api/public/maintenance/active` | GET | `MaintenancePageController.java:14` | language-independent | reads a Cloudflare KV boolean |
| 3 | `/api/public/maintenance/toggle` | POST | `MaintenancePageController.java:19` | language-independent | flips a Cloudflare KV boolean |
| 4 | `/api/public/verify-recaptcha` | POST | `ReCaptchaController.java:18` | language-independent | reCAPTCHA verification only |
| 5 | `/api/public/config` | GET | `ConfigurationController.java:28` | language-independent | `ConfigurationService.getConfigurations()` returns the global config map; no language read |
| 6 | `/api/public/translations` | GET | `TranslationsController.java:30` | language-independent | language is a `@RequestParam lang` — controller does not consult `LanguageContext` |
| 7 | `/api/public/app/version/{platform}` | GET | `AppVersionController.java:19` | language-independent | currently a no-op `ResponseEntity.ok()` (real logic commented out); even the commented version is language-independent |
| 8 | `/api/public/baseSite/{targetBaseSiteCode}` | GET | `BaseSiteController.java:37` | language-independent | `BaseSiteDTO` carries identifier keys only (`labelKey`, `iconId`, `flagImageKey`); frontend resolves labels client-side |
| 9 | `/api/public/baseSite/details` | GET | `BaseSiteController.java:46` | language-independent | same as #8, all sites |
| 10 | `/api/public/baseSite/overviews` | GET | `BaseSiteController.java:54` | language-independent | `BaseSiteOverviewDTO` is identifier-keys-only too |
| 11 | `/api/public/baseSite/regions/{targetBaseSiteCode}` | GET | `BaseSiteController.java:61` | language-independent | returns the regions list off the same cached `BaseSiteDTO` |
| 12 | `/api/public/product/seen/{productId}` | GET | `PublicProductController.java:19` | language-independent | `ProductSeenService` — no language read |
| 13 | `/api/public/product/views/{productId}` | GET | `PublicProductController.java:28` | language-independent | count read only |
| 14 | `/api/public/product/search` (GET) | GET | `ProductSearchController.java:34` | **language-dependent** | `ProductDetailsConverter` calls `languageContext.getCurrentLanguageCode()` to resolve product name/description |
| 15 | `/api/public/product/search/autocomplete` | POST | `ProductSearchController.java:48` | **language-dependent** | `SearchProductDataConverter` + `TextQueryGenerator` both read `languageContext.getCurrentLanguageCode()` |
| 16 | `/api/public/product/search` (POST) | POST | `ProductSearchController.java:57` | **language-dependent** | `ProductOverviewConverter` + `TextQueryGenerator` read language |
| 17 | `/api/public/user` | GET | `UserDataController.java:19` | **language-dependent** | on cache miss, `DefaultUserService.mapProjectionToUserInfo` calls `userTranslationsService.getUserShortBio(...)` which reads `languageContext.getCurrentLanguageId()` |
| 18 | `/api/public/review` | GET | `PublicReviewController.java:22` | **language-dependent** | `ReviewConverter` → `DefaultReviewTranslationsService.getReviewTranslation` reads `languageContext.getCurrentLanguageId()` |
| 19 | `/api/public/filter_gen/init` | GET | `TestCreateJSON.java:15` | unclear — needs Igor | test/admin utility; reads catalog data; appears language-independent but is also clearly not a production route. Recommend either removing the route or excluding from the design conversation |
| 20 | `/api/public/notification/test` | POST | `NotificationsControllerTest.java:20` | unclear — needs Igor | dev/test endpoint; sends a fixed notification. Appears language-independent. Same recommendation as #19 |

**Note on caching for #17 (`/api/public/user`):** `getUserInfoForId` is `@Cacheable("redisUserInfo", key = "#userId")`. On a *cache hit* the DTO is returned without consulting `LanguageContext`. On a *cache miss* the converter path runs and language is required. Treating the route as `language-dependent` is the safe call — behavior must not depend on whether the cache happens to be warm.

**Routes the filter also runs on, outside `/api/public/*`:** `CurrentLanguageFilter` is a `@Component` extending `OncePerRequestFilter` with no `FilterRegistrationBean` and no `shouldNotFilter` override. It runs on *every* HTTP request — including `/api/secure/**` and `/api/auth/**`. These all sit behind auth, so the X-Lang handling there is also relevant, but they are out of the brief's scope. Flagged in "For Mastermind" in the session summary.

---

## 1c — Allowlist match shape

**Recommendation: a small set of entries mixing prefix and exact matches.** Not pure exact-match. Specifically:

- `/api/public/baseSite/` — **prefix.** Four endpoints under this controller, all language-independent, all returning identifier-key reference data from `BaseSiteCacheService`. Adding the regions endpoint to the allowlist explicitly is the right call too, but the cleaner read-out is "everything under `/baseSite/` tolerates a missing X-Lang."
- `/api/public/config` — **exact** (single endpoint).
- `/api/public/translations` — **exact** (single endpoint).

Optional additions Igor may want to consider for the allowlist based on the inventory in 1b — none of them were in the original four names, so flagging them here rather than assuming:

- `/api/public/health/check`, `/api/public/maintenance/active`, `/api/public/verify-recaptcha`, `/api/public/app/version/...` — all language-independent. Whether they need to be allowlisted depends on whether the web/mobile client ever calls them *before* it has a language to send (e.g. the very first request after a fresh install). If yes, they need to be on the allowlist. If the client always sends X-Lang on every request once it's bootstrapped — even with a fallback default — they don't strictly need to be, but allowlisting them costs nothing.
- `/api/public/product/seen/{id}` and `/api/public/product/views/{id}` are language-independent but called from the product detail page, which already needs X-Lang for the search call — practically X-Lang will always be present when these fire. No need to allowlist.

**Why prefix-or-exact-mixed rather than pure exact:** an exact list of four (`/baseSite/{code}`, `/baseSite/details`, `/baseSite/overviews`, `/baseSite/regions/{code}`) is more brittle than `startsWith("/api/public/baseSite/")`. A future endpoint added to `BaseSiteController` would automatically inherit the allowlist treatment, which is the right default for that subtree (its purpose is reference data, which is language-independent by design). If Igor prefers explicit exact entries for stronger gating, that is also defensible — the trade-off is one line of allowlist code vs. the chance that a future baseSite endpoint accidentally bypasses the allowlist.

---

## 1d — Current control-flow shape

`CurrentLanguageFilter.doFilterInternal` (lines 35–50):

```java
@Override
protected void doFilterInternal(
    HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
    throws ServletException, IOException {

  try {
    resolveRequestLanguage(request).ifPresent(languageContext::setCurrentLanguage);

    resolveAuthenticatedUserPreferredLanguage()
        .ifPresent(languageContext::setCurrentUserPreferredLanguage);

    filterChain.doFilter(request, response);            // <-- INSIDE the try

  } catch (Exception e) {
    log.warn("CurrentLanguageFilter swallowed exception on {}", request.getRequestURI(), e);
  }
}
```

Confirmed: `filterChain.doFilter(request, response)` is inside the try (line 45, between the try opener on line 39 and the catch on line 47). The brief's description of the bug matches the code exactly — every downstream controller exception is swallowed by this catch and turns into an empty 200.

**Is moving `doFilter` outside the try structurally clean?** Yes.

The only state the try block mutates that `doFilter` could depend on is `LanguageContext` (a `@RequestScope` bean). Two scenarios after moving `doFilter` outside the try:

1. **Language resolution succeeds.** `setCurrentLanguage` and/or `setCurrentUserPreferredLanguage` are called. The try completes. `doFilter` runs with `LanguageContext` populated. Same as today.
2. **Language resolution throws** (bad X-Lang) **— and we want to continue.** The catch runs and `LanguageContext.currentLanguage` is never set (the throw happens before `ifPresent` can call the setter). `doFilter` then runs with `LanguageContext.currentLanguage == null`.

   In scenario 2, a *downstream language-dependent* code path calling `languageContext.getCurrentLanguageId()` would NPE — because `LanguageContext.getCurrentLanguageId()` is `return currentLanguage.id();` with no null guard (`LanguageContext.java:21–23`). That NPE would bubble up as a 500.

   This is *exactly* why Phase 2 needs the allowlist split: on an allowlisted route, the downstream code does not consult `LanguageContext`, so a null language is fine and the request succeeds. On a non-allowlisted route, the request must short-circuit to 400 inside the filter *before* the chain runs — otherwise it would hit the NPE-→-500 path downstream. Phase 2's "continue on allowlist / 400 otherwise" is the right shape for this.

So: the structural move (`doFilter` out of the try) is clean. The "continue with language unset" branch is only safe for allowlisted routes, which Phase 2 already enforces.

**`getCurrentUserPreferredLanguageCode`** (`LanguageContext.java:37`) and **`getCurrentLanguageCode`** (`LanguageContext.java:25`) also dereference without null guards. Same pattern as `getCurrentLanguageId`. Same conclusion. Documented here so Phase 2 doesn't accidentally rely on a "language code is null" branch that doesn't exist.

---

## Trust boundary check

`X-Lang` is client-supplied. It is read by `resolveRequestLanguage` (line 52) only to select a `LanguageDTO` from the DB-backed `LanguageService.getLanguageByCode(...)` (line 59) — a value the server validates against its own data. The resulting `LanguageDTO` is set on `LanguageContext.currentLanguage` and consumed downstream solely for:

- ES converters resolving the right translation field (`ProductOverviewConverter`, `ProductDetailsConverter`, `SearchProductDataConverter`)
- `TextQueryGenerator` choosing the right ES analyzer
- `DefaultReviewTranslationsService.getReviewTranslation(...)` joining `ReviewTranslation` by language
- `DefaultUserTranslationsService.getUserShortBio(...)` joining `UserTranslation` by language

None of these are moderation, authorization, or state-transition decisions. The header drives display-text selection only. Trust boundary intact — the 400-vs-continue decision in Phase 2 is based on request *path* (server-derived), not on trusting the X-Lang value for anything security-relevant.

---

## Brief vs reality

Nothing flagged. The brief's description of `CurrentLanguageFilter`'s control flow, the swallowed-exception bug, and the intended Phase 2 behavior all match the code. The only minor brief-vs-reality note is that "baseSiteData" maps to *two* endpoints rather than one — handled in 1a/1c by recommending the `/baseSite/` prefix.

---

## Recommendation for Igor

Confirm the following before Phase 2 runs:

1. **Allowlist match shape:** prefix on `/baseSite/`, exact on `/config` and `/translations`. (Default; please confirm or override.)
2. **Allowlist contents:** `/api/public/baseSite/`, `/api/public/config`, `/api/public/translations`. Whether to also allowlist `/health/check`, `/maintenance/active`, `/verify-recaptcha`, `/app/version/...` — see 1c notes; defaulting to "no" unless Igor says otherwise.
3. **Error code for the 400:** there is no existing code that fits the "missing/unresolvable X-Lang" case. Phase 2 will need a new code (suggested: `LANG_REQUIRED` or `LANG_INVALID`, or a single combined `LANG_MISSING_OR_INVALID`). Flagged for Igor's call rather than invented silently — per the brief's instruction.
4. **Scope of the X-Lang 400 rule:** brief restricts the rule to public routes. The filter also runs on `/api/secure/**` and `/api/auth/**`. Phase 2's allowlist-or-400 decision will therefore also apply to those routes — i.e. a secure endpoint missing X-Lang would 400. Igor should confirm whether that's the intended scope or whether secure routes should always pass through regardless of X-Lang.
