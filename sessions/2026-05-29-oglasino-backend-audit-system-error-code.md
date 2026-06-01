# Audit — `ProductErrorCode` / System-Code Split + REVIEW-Report Gap

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-29
**Mode:** READ-ONLY (no code changes; this audit file is the only output)
**Scope:** Phase 2 feature audit. Code is ground truth — no prior reports, session files, or issue text were read as authoritative; every claim below is sourced to file:line.

---

## Headline findings (read this first)

1. **`ProductErrorCode` carries 11 cross-cutting constants** out of 47 total. Seven of them live under the `product.system.*` translation-key prefix (`RATE_LIMITED`, `NOT_AUTHENTICATED`, `ACCESS_DENIED`, `INTERNAL_ERROR`, `LANG_MISSING_OR_INVALID`, `BASE_SITE_MISSING_OR_INVALID`, `NOT_OWNER`); the other four are user/profile-level (`USER_SETUP_INCOMPLETE`, `DISPLAY_NAME_REQUIRED/_SIZE/_PATTERN`). These are the natural candidates to migrate into a future `SystemErrorCode` enum.

2. **`ReportErrorCode` is a clean structural template** — identical `(translationKey, httpStatus)` shape to `ProductErrorCode`, no interface, no helpers. A `SystemErrorCode` modelled on it would drop in cleanly.

3. **The lookup extension point is one method, one line.** `GlobalExceptionHandler.resolveTranslationKey(String)` chains `ProductErrorCode.valueOf` → `ReportErrorCode.valueOf`. A third enum needs one more `try/valueOf/catch` block inserted before the final catch (file:line below).

4. **No seed-enforcement test exists for `ReportErrorCode`** — only `ProductErrorCodeTest`. A future `SystemErrorCode` would ship untested for seed coverage unless a per-enum test is added.

5. **The key-rename is low-risk.** Renaming `product.system.*` → `system.*` touches 28 seed rows (7 keys × 4 locales) + 7 enum fields, and the seed-enforcement test self-adjusts (it reads the enum, doesn't hardcode). **Only three hardcoded literal occurrences exist outside the enum + seed SQL** — all in MockMvc test assertions, all for the same `base_site_missing_or_invalid` key.

6. **⚠️ BRIEF vs REALITY (Part C.10):** The brief (via `issues.md` 2026-05-28) asks me to confirm that a `reportType=REVIEW` request runs the PRODUCT existence check (`getOwnerId`) against the review id, producing a misfile-on-collision or `REPORTED_PRODUCT_NOT_FOUND` 422. **The code refutes this.** `ReportType` has only `PRODUCT` and `USER`. With no lenient-enum Jackson config and no `HttpMessageNotReadableException` handler, a `reportType=REVIEW` body **fails Jackson deserialization before any service code runs**, falls to the catch-all, and returns **HTTP 500 `INTERNAL_ERROR` (`product.system.internal_error`)**. The PRODUCT branch is never reached; `getOwnerId` is never called; there is no 422 and no misfile. Full trace and the one wire-dependency caveat are in Part C.10.

---

# Part A — `ProductErrorCode` and the cross-cutting codes

## A.1 — Inventory & classification

**File:** `src/main/java/com/memento/tech/oglasino/exception/ProductErrorCode.java`

All 47 constants, verbatim, in source order, with classification (`P` = product-specific, `X` = cross-cutting):

| # | Constant | translationKey | httpStatus | Class |
|---|----------|----------------|------------|-------|
| 1 | RATE_LIMITED | product.system.rate_limited | TOO_MANY_REQUESTS | **X** — rate limiting is system-level |
| 2 | NOT_AUTHENTICATED | product.system.not_authenticated | FORBIDDEN | **X** — authentication, applies to all endpoints |
| 3 | ACCESS_DENIED | product.system.access_denied | FORBIDDEN | **X** — authorization, not product-specific |
| 4 | INTERNAL_ERROR | product.system.internal_error | INTERNAL_SERVER_ERROR | **X** — generic server error |
| 5 | LANG_MISSING_OR_INVALID | product.system.lang_missing_or_invalid | BAD_REQUEST | **X** — request-level language header |
| 6 | BASE_SITE_MISSING_OR_INVALID | product.system.base_site_missing_or_invalid | BAD_REQUEST | **X** — request-level tenant/site context |
| 7 | NAME_REQUIRED | product.name.required | BAD_REQUEST | P — product name field |
| 8 | NAME_TOO_LONG | product.name.too_long | BAD_REQUEST | P — product name field |
| 9 | NAME_REPEATING_CHARS | product.name.repeating_chars | BAD_REQUEST | P — product name content |
| 10 | NAME_EXCESSIVE_PUNCTUATION | product.name.excessive_punctuation | BAD_REQUEST | P — product name content |
| 11 | NAME_ALL_CAPS | product.name.all_caps | BAD_REQUEST | P — product name content |
| 12 | NAME_BANNED_WORDS | product.name.banned_words | BAD_REQUEST | P — product name moderation |
| 13 | NAME_LEADING_TRAILING_SPACES | product.name.leading_trailing_spaces | BAD_REQUEST | P — product name formatting |
| 14 | NAME_KEYWORD_STUFFING | product.name.keyword_stuffing | BAD_REQUEST | P — product name content |
| 15 | NAME_PROMOTIONS | product.name.promotions | BAD_REQUEST | P — product name moderation |
| 16 | NAME_CONTAINS_LINK | product.name.contains_link | BAD_REQUEST | P — product name moderation |
| 17 | NAME_CONTAINS_CONTACT | product.name.contains_contact | BAD_REQUEST | P — product name moderation |
| 18 | NAME_GIBBERISH | product.name.gibberish | BAD_REQUEST | P — product name content |
| 19 | NAME_MASSIVE_CHANGE | product.name.massive_change | UNPROCESSABLE_ENTITY | P — product name edit workflow |
| 20 | DESCRIPTION_REQUIRED | product.description.required | BAD_REQUEST | P — product description field |
| 21 | DESCRIPTION_TOO_LONG | product.description.too_long | BAD_REQUEST | P — product description field |
| 22 | DESCRIPTION_BANNED_WORDS | product.description.banned_words | BAD_REQUEST | P — product description moderation |
| 23 | DESCRIPTION_SPAMMY | product.description.spammy | BAD_REQUEST | P — product description content |
| 24 | DESCRIPTION_CONTAINS_LINK | product.description.contains_link | BAD_REQUEST | P — product description moderation |
| 25 | DESCRIPTION_CONTAINS_CONTACT | product.description.contains_contact | BAD_REQUEST | P — product description moderation |
| 26 | DESCRIPTION_LEADING_TRAILING_SPACES | product.description.leading_trailing_spaces | BAD_REQUEST | P — product description formatting |
| 27 | DESCRIPTION_GIBBERISH | product.description.gibberish | BAD_REQUEST | P — product description content |
| 28 | DESCRIPTION_MASSIVE_CHANGE | product.description.massive_change | UNPROCESSABLE_ENTITY | P — product description edit workflow |
| 29 | PRODUCT_ID_REQUIRED | product.id.required | BAD_REQUEST | P — product identification |
| 30 | PRODUCT_NOT_FOUND | product.id.not_found | UNPROCESSABLE_ENTITY | P — product lookup |
| 31 | CATEGORY_REQUIRED | product.category.required | BAD_REQUEST | P — product category field |
| 32 | CATEGORY_NOT_FOUND | product.category.not_found | UNPROCESSABLE_ENTITY | P — product category lookup |
| 33 | PRICE_REQUIRED | product.price.required | UNPROCESSABLE_ENTITY | P — product pricing field |
| 34 | CURRENCY_REQUIRED | product.currency.required | BAD_REQUEST | P — product currency field |
| 35 | CURRENCY_NOT_ALLOWED | product.currency.not_allowed | UNPROCESSABLE_ENTITY | P — product currency business rule |
| 36 | FILTER_NOT_IN_CATEGORY | product.filter.not_in_category | UNPROCESSABLE_ENTITY | P — product filter |
| 37 | FILTER_OPTION_NOT_IN_FILTER | product.filter.option_not_in_filter | UNPROCESSABLE_ENTITY | P — product filter |
| 38 | FILTER_SINGLE_OPTION_INVALID | product.filter.single_option_invalid | UNPROCESSABLE_ENTITY | P — product filter |
| 39 | FILTER_MULTI_OPTION_EMPTY | product.filter.multi_option_empty | UNPROCESSABLE_ENTITY | P — product filter |
| 40 | FILTER_RANGE_VALUE_REQUIRED | product.filter.range_value_required | UNPROCESSABLE_ENTITY | P — product filter |
| 41 | FILTER_DATE_VALUE_REQUIRED | product.filter.date_value_required | UNPROCESSABLE_ENTITY | P — product filter |
| 42 | FILTER_DATE_VALUE_OUT_OF_RANGE | product.filter.date_value_out_of_range | UNPROCESSABLE_ENTITY | P — product filter |
| 43 | NOT_OWNER | product.system.not_owner | FORBIDDEN | **X** — authorization (ownership), not content validation |
| 44 | USER_SETUP_INCOMPLETE | product.user.setup_incomplete | UNPROCESSABLE_ENTITY | **X** — user account state, cross-product |
| 45 | DISPLAY_NAME_REQUIRED | displayName.required | BAD_REQUEST | **X** — user profile (shared by LoginRequest + UpdateUserDTO per source comment) |
| 46 | DISPLAY_NAME_SIZE | displayName.size | BAD_REQUEST | **X** — user profile |
| 47 | DISPLAY_NAME_PATTERN | displayName.pattern | BAD_REQUEST | **X** — user profile |

**Total: 47 constants — 36 product-specific, 11 cross-cutting.**

### Cross-cutting sub-list (the migration candidates)

| Constant | translationKey | httpStatus | Sub-group |
|----------|----------------|------------|-----------|
| RATE_LIMITED | product.system.rate_limited | TOO_MANY_REQUESTS | `product.system.*` |
| NOT_AUTHENTICATED | product.system.not_authenticated | FORBIDDEN | `product.system.*` |
| ACCESS_DENIED | product.system.access_denied | FORBIDDEN | `product.system.*` |
| INTERNAL_ERROR | product.system.internal_error | INTERNAL_SERVER_ERROR | `product.system.*` |
| LANG_MISSING_OR_INVALID | product.system.lang_missing_or_invalid | BAD_REQUEST | `product.system.*` |
| BASE_SITE_MISSING_OR_INVALID | product.system.base_site_missing_or_invalid | BAD_REQUEST | `product.system.*` |
| NOT_OWNER | product.system.not_owner | FORBIDDEN | `product.system.*` |
| USER_SETUP_INCOMPLETE | product.user.setup_incomplete | UNPROCESSABLE_ENTITY | `product.user.*` |
| DISPLAY_NAME_REQUIRED | displayName.required | BAD_REQUEST | `displayName.*` |
| DISPLAY_NAME_SIZE | displayName.size | BAD_REQUEST | `displayName.*` |
| DISPLAY_NAME_PATTERN | displayName.pattern | BAD_REQUEST | `displayName.*` |

**Classification note for Mastermind:** the enum's own source comment block (around lines 17–25) explicitly marks constants 1–6 as "Cross-cutting (system-level)". `NOT_OWNER` (line 72) shares the `product.system.*` prefix and is an authorization gate, so it belongs with that group even though it sits lower in the file. `USER_SETUP_INCOMPLETE` and the three `DISPLAY_NAME_*` constants are cross-cutting in the sense of "not product-content validation," but they are **user/profile-level**, not system/request-level — they live under different key prefixes (`product.user.*`, `displayName.*`) and are shared with auth/user DTOs. If the feature's goal is a `SystemErrorCode` enum, the clean 7 are the `product.system.*` set; the user/profile 4 are a separate question (a `UserErrorCode`? leave in place?). **This is a scope decision for Mastermind — flagged, not assumed.**

## A.2 — Consumers of the cross-cutting codes

Every reference found by grepping each constant name across `src/main`. The two brief-named candidates (`RateLimitFilter`, `CurrentLanguageFilter`) are confirmed; all others found independently.

**RATE_LIMITED** — emitted only by the rate-limit filter.
- `RateLimitFilter.java:32,34` — builds a static response body, reading `.name()` and `.getTranslationKey()`.
- `RateLimitFilter.java:77` — writes that body when the bucket is exhausted.
- (Note: `ImageErrorCode.RATE_LIMITED` at `ImageErrorCode.java:31` / `ImageRequestException.java:90` is a **separate enum** in the image subsystem — not `ProductErrorCode`.)

**LANG_MISSING_OR_INVALID** — emitted only by the language filter.
- `CurrentLanguageFilter.java:50–55` — builds static error body (`.name()` + `.getTranslationKey()`).
- `CurrentLanguageFilter.java:85–88` — `if (!languageResolved && requiresLanguage(uri)) { writeLangMissingOrInvalid(response); return; }`.
- `CurrentLanguageFilter.java:106–111` — writes the HTTP 400 body.

**BASE_SITE_MISSING_OR_INVALID** — thrown by three controllers as a null-context guard.
- `ProductSearchController.java:56` and `:68` — `throw new ProductValidationException(ProductErrorCode.BASE_SITE_MISSING_OR_INVALID)` when `baseSiteContext.getCurrentBaseSite() == null`.
- `PublicProductController.java:44` — same guard.

**NOT_AUTHENTICATED / ACCESS_DENIED** — selected by the exception handler.
- `GlobalExceptionHandler.java:42–45` — `code = hasAuthenticatedPrincipal() ? ACCESS_DENIED : NOT_AUTHENTICATED;` then `403`. (Javadoc references at lines 35–36.)

**INTERNAL_ERROR** — the catch-all fallback.
- `GlobalExceptionHandler.java:168` — `ProductErrorResponse.single(null, ProductErrorCode.INTERNAL_ERROR)` at HTTP 500.

**NOT_OWNER** — three ownership gates.
- `DashboardProductController.java:60`, `DefaultProductService.java:228`, `DefaultProductService.java:648` — all `if (!currentUserId.equals(product.getOwner().getId())) throw new ProductValidationException(ProductErrorCode.NOT_OWNER);`.

(The four user/profile codes — `USER_SETUP_INCOMPLETE`, `DISPLAY_NAME_*` — are consumed in the user/product-create and login/update-user paths via Jakarta annotation messages and service throws; they are out of the `product.system.*` migration set, so their full consumer map is deferred unless Mastermind folds them into scope.)

## A.6 — Trust boundaries for the cross-cutting codes

For each code the question is: **does any branch read the code's value to make a moderation / authorization / state-transition decision**, vs. merely emitting it as an error response?

| Code | Decision driver (what actually gates) | Reads the code value to decide? | Verdict |
|------|----------------------------------------|----------------------------------|---------|
| RATE_LIMITED | Bucket4j `ConsumptionProbe.isConsumed()` in `RateLimitFilter` | No — code is written into the body only | **CLEAN** |
| LANG_MISSING_OR_INVALID | `requiresLanguage(uri)` allowlist + language-resolution result | No — code is emitted only | **CLEAN** |
| BASE_SITE_MISSING_OR_INVALID | `baseSiteContext.getCurrentBaseSite() == null` | No — guard reads context state, not the code | **CLEAN** |
| NOT_AUTHENTICATED | Spring Security (no principal) → handler picks code from `hasAuthenticatedPrincipal()` reading `SecurityContextHolder` | No — code is a consequence of the security decision | **CLEAN** |
| ACCESS_DENIED | Spring Security (missing role) → same handler selection | No — consequence, not input | **CLEAN** |
| INTERNAL_ERROR | Catch-all for unhandled exceptions | No | **CLEAN** |
| NOT_OWNER | `currentUserId.equals(product.getOwner().getId())` | No — comparison drives the throw; code is the consequence | **CLEAN** |

**Explicit confirmations requested by the brief:**
- **`LANG_MISSING_OR_INVALID` gates nothing** beyond returning the 400. The gate is `requiresLanguage(uri)` (an allowlist check) plus whether resolution succeeded — neither reads the error code.
- **The rate-limit code (`RATE_LIMITED`) gates nothing.** The throttle decision is the token-bucket probe; the code is only embedded in the response body.

No cross-cutting code is read as input to a moderation/authorization/state-transition decision. **Trust-boundary verdict for the whole cross-cutting set: clean.** This means moving these codes into a `SystemErrorCode` enum is a pure refactor with no trust-boundary implication — no decision logic depends on which enum a code belongs to.

---

# Part A (cont.) — `ReportErrorCode`, the handler, and the seed test

## A.3 — `ReportErrorCode` inventory

**File:** `src/main/java/com/memento/tech/oglasino/exception/ReportErrorCode.java`

All 10 constants, verbatim, source order (lines 13–28):

```
REPORT_TYPE_REQUIRED("report.type.required", HttpStatus.BAD_REQUEST)
REPORT_OPTION_REQUIRED("report.option.required", HttpStatus.BAD_REQUEST)
REPORT_DESCRIPTION_REQUIRED("report.description.required", HttpStatus.BAD_REQUEST)
REPORT_DESCRIPTION_TOO_LONG("report.description.too_long", HttpStatus.BAD_REQUEST)
REPORTED_USER_ID_REQUIRED("report.reported_user_id.required", HttpStatus.UNPROCESSABLE_ENTITY)
REPORTED_PRODUCT_ID_REQUIRED("report.reported_product_id.required", HttpStatus.UNPROCESSABLE_ENTITY)
REPORTED_USER_NOT_FOUND("report.reported_user.not_found", HttpStatus.UNPROCESSABLE_ENTITY)
REPORTED_PRODUCT_NOT_FOUND("report.reported_product.not_found", HttpStatus.UNPROCESSABLE_ENTITY)
REPORT_SELF_NOT_ALLOWED("report.self_not_allowed", HttpStatus.UNPROCESSABLE_ENTITY)
REPORTED_USER_PENDING_DELETION("report.reported_user.pending_deletion", HttpStatus.UNPROCESSABLE_ENTITY)
```

**Total: 10 constants.**

**Shape:** mirrors `ProductErrorCode` exactly — `private final String translationKey; private final HttpStatus httpStatus;` (lines 30–31), a two-arg constructor (33–36), and `getTranslationKey()` / `getHttpStatus()` accessors (38–44). No interface implemented, no helper methods, no structural differences. **This is the clean template for what a `SystemErrorCode` enum should look like.**

## A.4 — `GlobalExceptionHandler.resolveTranslationKey` mechanism

**File:** `src/main/java/com/memento/tech/oglasino/exception/GlobalExceptionHandler.java`
**Method:** `resolveTranslationKey(String message)` — confirmed name, lines 178–193 (verbatim):

```java
private static String resolveTranslationKey(String message) {
  if (message == null) {
    return null;
  }
  try {
    return ProductErrorCode.valueOf(message).getTranslationKey();
  } catch (IllegalArgumentException ignored) {
    // fall through to ReportErrorCode
  }
  try {
    return ReportErrorCode.valueOf(message).getTranslationKey();
  } catch (IllegalArgumentException ignored) {
    log.warn("Jakarta validation message '{}' does not match any known error code", message);
    return null;
  }
}
```

**Mechanism, precisely:**
- **Input:** a Jakarta/Bean-Validation `message` string (the message attribute on the failing constraint, which by convention IS the enum constant name — e.g. `"NAME_BANNED_WORDS"`).
- **Null guard (179–180):** returns `null` immediately.
- **Match strategy:** `EnumType.valueOf(message)` — matches the message string to an enum constant **by `.name()`, case-sensitive**. On match, returns that constant's `getTranslationKey()`.
- **Iteration order:** `ProductErrorCode` first (183); on `IllegalArgumentException` (no match) falls through to `ReportErrorCode` (188).
- **No-match behavior:** logs at WARN and **returns `null`** (190–191). The caller then ships a `FieldError` with a null `translationKey` but a populated `code`.

**Extension point for a third enum (e.g. `SystemErrorCode`):** insert one more `try/valueOf/catch` block **between line 188 and the final catch at line 189** — i.e. after the `ReportErrorCode.valueOf` attempt, before the WARN-and-return-null block:

```java
try {
  return SystemErrorCode.valueOf(message).getTranslationKey();
} catch (IllegalArgumentException ignored) {
  // fall through
}
```

This method is the **only** place in the codebase that maps a validation message string to a translation key by chaining `.valueOf` across these enums. Nothing else needs touching for the lookup to learn about a third enum.

**Adjacent note (low):** this linear `valueOf`-chain grows by one block per enum. With three enums it is still fine; if a fourth/fifth ever joins, a single registry (`List.of(ProductErrorCode.values(), ReportErrorCode.values(), …)` or a static `Map<String,String>` built once) would read better. Not worth doing for the third — flagged only so a future reader sees the trend. Out of scope for this audit.

## A.5 — The seed-enforcement test

**File:** `src/test/java/com/memento/tech/oglasino/exception/ProductErrorCodeTest.java` (confirmed name).

Two tests, both iterating `ProductErrorCode.values()`:

1. **`everyConstantHasNonBlankTranslationKeyAndHttpStatus()` (35–43)** — for each constant asserts `getTranslationKey()` is non-null and non-blank, and `getHttpStatus()` is non-null.
2. **`everyTranslationKeyResolvesInEnglishSeed()` (45–53)** — builds a set of ERRORS-namespace keys by parsing the EN seed file `src/main/resources/data/translations/0001-data-web-translations-EN.sql` with a regex (`(id, lang, 'namespace', 'key', …)` shape), filtering to namespace `ERRORS`, then asserts every `ProductErrorCode` constant's `translationKey` is present in that set. Key assertion: `assertThat(errorKeysInSeed).contains(code.getTranslationKey())`.

**Critically, the test reads `code.getTranslationKey()` — it does not hardcode any literal key string.** So a rename of the enum + seed in lockstep keeps this test green with zero edits.

**No `ReportErrorCodeTest` exists** (zero matches; the only files in that test package are `GlobalExceptionHandlerTest.java` and `ProductErrorCodeTest.java`). That is a real gap: a `ReportErrorCode` constant could today be added without a matching ERRORS seed row and nothing would catch it.

**Should an equivalent per-enum test exist?** Yes. The pattern is per-enum because each test iterates one enum's `values()`. A future `SystemErrorCode` should ship with its own `SystemErrorCodeTest` (clone of `ProductErrorCodeTest`, swap the enum), and the missing `ReportErrorCodeTest` should be authored at the same time so all three enums have seed-coverage enforcement. **Watch-out:** `ProductErrorCodeTest` filters the seed to namespace `ERRORS`. If `SystemErrorCode` keys are seeded under a different namespace, the cloned test's namespace filter must match — otherwise it will report false failures.

---

# Part B — the key-rename question (decides feature scope)

## B.7 — Translation seed rows for the cross-cutting codes

Locale/ID convention (`src/main/resources/data/core/data-languages.sql`): lang **1 = SR (RS)**, **2 = CNR**, **3 = EN**, **4 = RU**. All seed rows live in the four `0001-data-web-translations-<LOCALE>.sql` files under `src/main/resources/data/translations/`, namespace `ERRORS`.

The 7 `product.system.*` keys each have a complete 4-locale set. **No locale row is missing for any of the 7 keys.** IDs and file:line:

| translationKey | EN (id @ line) | RS (id @ line) | RU (id @ line) | CNR (id @ line) |
|----------------|----------------|----------------|----------------|------------------|
| product.system.rate_limited | 3089 @ EN:603 | 5189 @ RS:601 | 7289 @ RU:599 | 989 @ CNR:599 |
| product.system.not_authenticated | 3090 @ EN:604 | 5190 @ RS:602 | 7290 @ RU:600 | 990 @ CNR:600 |
| product.system.access_denied | 3091 @ EN:605 | 5191 @ RS:603 | 7291 @ RU:601 | 991 @ CNR:601 |
| product.system.internal_error | 3092 @ EN:606 | 5192 @ RS:604 | 7292 @ RU:602 | 992 @ CNR:602 |
| product.system.not_owner | 3093 @ EN:607 | 5193 @ RS:605 | 7293 @ RU:603 | 993 @ CNR:603 |
| product.system.lang_missing_or_invalid | 3137 @ EN:654 | 5237 @ RS:652 | 7337 @ RU:650 | 1037 @ CNR:650 |
| product.system.base_site_missing_or_invalid | 3147 @ EN:664 | 5247 @ RS:662 | 7347 @ RU:660 | 1047 @ CNR:660 |

Current EN text values (for reference; per-locale text confirmed present in all four files):
- `rate_limited` → "You're making requests too quickly. Please wait a moment and try again."
- `not_authenticated` → "Please sign in to continue."
- `access_denied` → "You don't have permission to perform this action."
- `internal_error` → "Something went wrong on our end. Please try again."
- `not_owner` → "You don't have permission to modify this product."
- `lang_missing_or_invalid` → "Language is missing or not supported. Please refresh the page and try again."
- `base_site_missing_or_invalid` → "Base site is missing or not supported. Please refresh the page and try again."

(The four user/profile cross-cutting keys — `product.user.setup_incomplete`, `displayName.required/size/pattern` — are NOT under the `product.system.*` prefix and are out of the rename set described below. If Mastermind folds them into a `system.*` migration, they need their own seed-row inventory.)

## B.8 — Rename impact: `product.system.*` → `system.*`

If we rename the 7 keys (drop the `product.` prefix, or any equivalent reshape), the complete change set is:

**1. Seed rows — 28 rows across 4 files** (7 keys × 4 locales). Exact IDs/lines are the table above. Files:
- `src/main/resources/data/translations/0001-data-web-translations-EN.sql` (lines 603–607, 654, 664)
- `…-RS.sql` (lines 601–605, 652, 662)
- `…-RU.sql` (lines 599–603, 650, 660)
- `…-CNR.sql` (lines 599–603, 650, 660)

**2. Enum `translationKey` fields — 7 edits** in `ProductErrorCode.java`: lines 18, 19, 20, 21, 22, 23–24, 72.

**3. Seed-enforcement test — no edit needed.** `ProductErrorCodeTest.everyTranslationKeyResolvesInEnglishSeed()` reads `code.getTranslationKey()` from the enum and searches the seed; rename both in lockstep and it stays green. (Caveat from A.5: if the rename also moves the keys to a non-`ERRORS` namespace, the test's `ERRORS`-filter would need updating — but a prefix-only rename within `ERRORS` is invisible to it.)

**4. Hardcoded literal strings — the critical grep.** Searching the **entire** repo (`src/main` + `src/test`) for `product.system.` literals: **the only hardcoded occurrences outside the enum definition and the seed SQL are three MockMvc test assertions, all for `product.system.base_site_missing_or_invalid`:**
- `src/test/java/com/memento/tech/oglasino/controller/ProductSearchControllerBaseSiteTest.java:56`
- `src/test/java/com/memento/tech/oglasino/controller/ProductSearchControllerBaseSiteTest.java:76`
- `src/test/java/com/memento/tech/oglasino/controller/ProductCountControllerTest.java:88`

These assert the `translationKey` value in the response JSON (`.value("product.system.base_site_missing_or_invalid")`). They do not drive logic; they would need the literal updated on rename (or refactored to read `ProductErrorCode.BASE_SITE_MISSING_OR_INVALID.getTranslationKey()`).

**No other backend code or test hardcodes `product.system.rate_limited` or any other `product.system.*` literal.** `RateLimitFilter` and `CurrentLanguageFilter` build their bodies from `code.getTranslationKey()`, not from a string literal — so they are rename-safe with zero edits.

**Rename verdict:** low-risk, mechanically contained. Total touch list = 28 seed rows + 7 enum fields + 3 test-assertion literals = **38 edits across 6 files**, no logic changes, the enforcement test self-validates. The cleanest path also folds the 3 literal assertions into reading from the enum so the "only the enum + seed name the literal" invariant becomes complete.

---

# Part C — REVIEW-report gap

## C.9 — Report-submit surface inventory

- **Controller:** `ReportController.createReport()` — `src/main/java/com/memento/tech/oglasino/controller/ReportController.java:21`. Maps `POST /api/secure/report/add`, accepts `@Valid @RequestBody ReportRequestDTO`, returns `200 OK` (true) / `406 NOT_ACCEPTABLE` (false, the dedupe path).
- **Facade:** `DefaultReportFacade.createReport()` — `facade/impl/DefaultReportFacade.java:25` — Redis dedupe (24h TTL) before delegating to the service.
- **Service:** `DefaultReportService.createReport()` — `service/impl/DefaultReportService.java:29`.
- **Request DTO:** `dto/ReportRequestDTO.java` — fields (verbatim):
  - `Long reportedUserId` (line 16) — no Jakarta annotation; presence enforced in the service for USER.
  - `Long reportedProductId` (line 22) — no Jakarta annotation; presence enforced in the service for PRODUCT.
  - `String description` (line 26) — `@NotBlank(message="REPORT_DESCRIPTION_REQUIRED")`, `@Size(max=2000, message="REPORT_DESCRIPTION_TOO_LONG")`.
  - `ReportOption reportOption` (line 29) — `@NotNull(message="REPORT_OPTION_REQUIRED")`.
  - `ReportType reportType` (line 32) — `@NotNull(message="REPORT_TYPE_REQUIRED")`.
  - **There is no `reportedReviewId` field.**
- **`ReportType` enum** — `entity/ReportType.java` — verbatim:
  ```java
  public enum ReportType {
    PRODUCT,
    USER
  }
  ```
  **Two values. No `REVIEW`.**
- **Types the service actually branches on:** USER (`DefaultReportService:36–56`) and PRODUCT (`:57–74`). There is **no `else`** — any other type falls straight through to the persist block at `:76–85`.

## C.10 — What happens today on `reportType=REVIEW` (the issue's premise, refuted)

**The brief asks me to confirm the issue's claim** that the service runs `productRepository.getOwnerId(<review-id>)` against the review id, yielding (collision → misfile) or (no-collision → `REPORTED_PRODUCT_NOT_FOUND` 422). **Tracing the code, this is NOT what happens.** The request fails earlier, in deserialization.

**Exact path:**
1. `POST /api/secure/report/add` arrives with JSON `{"reportType":"REVIEW", "reportedProductId":<reviewId>, …}`.
2. Spring MVC deserializes the body into `ReportRequestDTO` via Jackson **before** `@Valid` runs. Jackson must map `"REVIEW"` → `ReportType`. `ReportType` has no `REVIEW` constant.
3. **No leniency is configured:** repo-wide there is no `READ_UNKNOWN_ENUM_VALUES_AS_NULL` / `read-unknown-enum-values-as-null` setting, no `spring.jackson.*` block in `application.yaml` / `-dev` / `-stage` / `-prod`, no custom `ReportType` deserializer (`@JsonCreator`/`@JsonValue` — the enum is bare), and no global `ObjectMapper`/`Jackson2ObjectMapperBuilderCustomizer` MVC override. So Jackson throws `InvalidFormatException`, which Spring MVC wraps as `HttpMessageNotReadableException`.
4. **`GlobalExceptionHandler` does not handle it.** It is a plain `@RestControllerAdvice` (does **not** extend `ResponseEntityExceptionHandler`) and declares no `@ExceptionHandler` for `HttpMessageNotReadableException`. So the exception falls to the catch-all `@ExceptionHandler(Exception.class) handleGeneric` at `GlobalExceptionHandler.java:164–169`.
5. **Result: HTTP 500**, body `{"errors":[{"field":null,"code":"INTERNAL_ERROR","translationKey":"product.system.internal_error"}]}`, logged at ERROR with full stack trace.

**Therefore:**
- The PRODUCT branch is **never entered**; `productRepository.getOwnerId(...)` is **never called** against the review id.
- There is **no collision-misfile outcome** and **no `REPORTED_PRODUCT_NOT_FOUND` 422**. The two outcomes the issue describes do not occur with the current code.
- The actual user-facing result is a generic 500 "something went wrong" — which is arguably worse than a 422 (it logs a stack trace as if it were a server bug, and the web side maps `INTERNAL_ERROR` to a generic failure toast, not a "can't report reviews" message).

**For completeness — the hypothetical fall-through (NOT reachable today):** *if* `REVIEW` were ever accepted by deserialization (e.g. someone adds it to the enum but not to the service), the service's missing `else` means execution reaches the persist block at `DefaultReportService:76–85` with `reportedUser=null` and `reportedProductId=<reviewId>`, saving a row with no target validation. But even that is blocked at the DB layer by the `report_type` CHECK constraint (`V1__init_schema.sql:477`, allows only `'PRODUCT'`,`'USER'`) → `DataIntegrityViolationException` → also a 500. So there is no path today that reaches `getOwnerId` with a review id.

**⚠️ Wire-dependency caveat (seam — cannot confirm from backend alone):** this trace assumes web literally sends the string `"REVIEW"` in the `reportType` field. I cannot read `oglasino-web`. Two branches:
- If web sends `"REVIEW"` (most likely, given a string enum `ReportType.REVIEW = 'REVIEW'`) → **500 `INTERNAL_ERROR`** as traced.
- If web sends a numeric value (e.g. ordinal `2`) → Jackson maps integers to enum ordinals; `2` is out of range for a 2-value enum → still `InvalidFormatException` → **500**.
- The **only** way the issue's "`getOwnerId` runs against the review id → 422" outcome could be real is if web actually sends `reportType:"PRODUCT"` (not `"REVIEW"`) with the review id in `reportedProductId`. The issue text and the brief both say `reportType: ReportType.REVIEW`, so on the stated premise the answer is 500-at-deserialization. **Web must confirm the exact serialized `reportType` string** to close this. See Seam 2.

## C.11 — What a REVIEW branch would need

- **Reviews table & repository — both exist.**
  - Entity: `entity/Review.java` (lines 19–130); table `review` at `V1__init_schema.sql:485–496`.
  - Repository: `repository/ReviewRepository.java` (extends `JpaRepository`, so `findById(Long)` is available; plus custom review queries).
- **Owner/author model — two user associations on `Review`:**
  - `reviewer` (`Review.java:30–31`, `@ManyToOne User`) — the user who **wrote** the review.
  - `targetUser` (`Review.java:33–34`, `@ManyToOne User`, non-optional) — the user the review is **about** (the seller being reviewed).
  - A self-report guard is therefore expressible: reject if `reporterId.equals(review.getReviewer().getId())` (don't report your own review) and/or `reporterId.equals(review.getTargetUser().getId())` — Mastermind decides which identities count as "self" for a review report.
- **Dedupe-key pattern — already extensible.** `DefaultReportFacade.buildCacheKey()` (lines 47–52):
  ```java
  Long targetId = type == ReportType.USER ? request.getReportedUserId() : request.getReportedProductId();
  return String.format("report:user:%d:target:%s:%s", reporterId, type, targetId);
  ```
  The key embeds `type` (the enum name) already, so a `REVIEW` type would produce `report:user:<id>:target:REVIEW:<reviewId>` for free. The only change is the `targetId` extraction — the binary ternary becomes a 3-way `switch` once a `reportedReviewId` field (or a reused slot) exists.

**Minimum coordinated change set to build REVIEW reporting** (for Mastermind's scoping, not implemented here): add `REVIEW` to `ReportType`; widen the `report_type` CHECK constraint at `V1__init_schema.sql:477` (pre-prod V1-fold per conventions Part 12); add a `reportedReviewId` field to `ReportRequestDTO` (or formally repurpose `reportedProductId`); add a REVIEW branch to `DefaultReportService` (existence check via `ReviewRepository.findById` + self-report guard); extend `buildCacheKey`'s `targetId` extraction; add new `ReportErrorCode` constant(s) (e.g. `REPORTED_REVIEW_ID_REQUIRED`, `REPORTED_REVIEW_NOT_FOUND`) + their 4-locale seed rows; and a seed-enforcement test. Cross-repo: web must align the wire contract (Seam 2).

---

# Seams to surface

### Seam 1 — wire `code` vs `translationKey`: which does web key its UI off?

**Backend fact:** the backend emits **both** fields on every validation failure. The wire record is `ProductErrorResponse` → `FieldError(String field, String code, String translationKey)` (`exception/ProductErrorResponse.java:15–17`), serialized as `{"errors":[{"field":"name","code":"NAME_BANNED_WORDS","translationKey":"product.name.banned_words"}]}`. Assembly sites in `GlobalExceptionHandler`: line 76 (Jakarta `MethodArgumentNotValidException`, `code = message`, `translationKey = resolveTranslationKey(message)`), line 100 (Spring 7 `HandlerMethodValidationException`), line 137 (`ReportValidationException`, `code = e.code().name()`, `translationKey = e.code().getTranslationKey()`), line 155 (user-deletion flow), and `ProductErrorResponse.single()` (`code = code.name()`, `translationKey = code.getTranslationKey()`).

**Assumption + what I need from web:** I assume web renders off `translationKey` and treats `code` as the stable machine identifier (consistent with conventions Part 7 "codes, never messages" and the record's own comment). **A key rename (`product.system.*` → `system.*`) only matters to web if web keys translation lookup off `translationKey`.** I need the web engineer to confirm: (a) does the web error parser read `translationKey` or `code` (or both) when selecting the UI string, and (b) does any web code hardcode a `product.system.*` literal? If web keys off `translationKey` and hardcodes any of these strings, the rename becomes a coordinated backend+web change, not backend-only.

### Seam 2 — `ReportType.REVIEW` wire contract

**Backend fact:** there is **no `ReportType.REVIEW`** and **no `reportedReviewId` field**. `ReportType` = `{PRODUCT, USER}`; `ReportRequestDTO` exposes only `reportedUserId` and `reportedProductId`; `Report` (`entity/Report.java:31–43`) persists `reportedUser` + `reportedProductId` only. There is zero backend slot for a review target.

**Assumption + what I need from web:** the issue states web sends `reportType: ReportType.REVIEW` with the review id in `reportedProductId`. Per the C.10 trace, on that premise the backend returns **500**, not the 422 the issue describes — because deserialization of an unknown enum value fails before the service runs. **I need the web engineer to confirm the exact serialized value of `reportType` on a review report** (the literal string on the wire), and which field carries the review id. That single fact determines whether today's behavior is a 500 (web sends `"REVIEW"`) or a mis-filed/422 product report (web actually sends `"PRODUCT"`). This is the load-bearing unknown for whatever resolution Mastermind picks (build REVIEW support / remove the wire / repurpose to PRODUCT).

---

## Audit method note

Findings were produced by a fan-out of read-only reader agents over the three surfaces, then the single highest-impact claim (C.10 — what `reportType=REVIEW` actually returns) was independently re-verified by direct reads of `ReportType.java`, `DefaultReportService.java`, `ReportRequestDTO.java`, `GlobalExceptionHandler.java` (class hierarchy + catch-all + absence of a `HttpMessageNotReadableException` arm), and repo-wide greps confirming no Jackson enum-leniency configuration in any profile. All file:line references in this document were read from the working tree on branch `dev`, 2026-05-29.
