# Error Code Domain Split

**Slug:** `system-error-code-split`
**Status:** web-stable
**Repos:** oglasino-backend (primary), oglasino-web (one-line + 2 test fixtures, lockstep)
**Branches:** backend `dev`, web `stage`

## 1. Goal

`ProductErrorCode` is a dumping ground. It holds 47 constants, 11 of which are not product errors. Split the non-product codes into domain enums so each error code lives in the enum that names its domain. Rename the misprefixed translation keys to match. Add seed-coverage enforcement for every error-code enum. No error response changes shape or `code` value from the client's view; only the enum home and the `translationKey` string move.

## 2. End state

Four error-code enums, each with the same shape (`translationKey`, `httpStatus`, two-arg constructor, two accessors — the `ReportErrorCode` template):

- `ProductErrorCode` — 36 genuine product codes. The 11 non-product codes are removed.
- `SystemErrorCode` — 7 system/request-level codes (new enum).
- `UserErrorCode` — 4 user/profile codes (new enum).
- `ReportErrorCode` — 10 existing + REVIEW additions (REVIEW work is the separate `review-reports` spec; this spec does not touch it).

`GlobalExceptionHandler.resolveTranslationKey` resolves against all enums via a single registry built once at class load, not a linear `valueOf` chain.

## 3. The split — exact constants

### Move to `SystemErrorCode` (7)

| Constant | Old key | New key | httpStatus |
|----------|---------|---------|------------|
| RATE_LIMITED | product.system.rate_limited | system.rate_limited | TOO_MANY_REQUESTS |
| NOT_AUTHENTICATED | product.system.not_authenticated | system.not_authenticated | FORBIDDEN |
| ACCESS_DENIED | product.system.access_denied | system.access_denied | FORBIDDEN |
| INTERNAL_ERROR | product.system.internal_error | system.internal_error | INTERNAL_SERVER_ERROR |
| LANG_MISSING_OR_INVALID | product.system.lang_missing_or_invalid | system.lang_missing_or_invalid | BAD_REQUEST |
| BASE_SITE_MISSING_OR_INVALID | product.system.base_site_missing_or_invalid | system.base_site_missing_or_invalid | BAD_REQUEST |
| NOT_OWNER | product.system.not_owner | system.not_owner | FORBIDDEN |

### Move to `UserErrorCode` (4)

| Constant | Old key | New key | httpStatus |
|----------|---------|---------|------------|
| USER_SETUP_INCOMPLETE | product.user.setup_incomplete | user.setup_incomplete | UNPROCESSABLE_ENTITY |
| DISPLAY_NAME_REQUIRED | displayName.required | user.display_name.required | BAD_REQUEST |
| DISPLAY_NAME_SIZE | displayName.size | user.display_name.size | BAD_REQUEST |
| DISPLAY_NAME_PATTERN | displayName.pattern | user.display_name.pattern | BAD_REQUEST |

The `code` value (the enum constant `.name()`, e.g. `"RATE_LIMITED"`) is unchanged on the wire. Only the enum the constant lives in and its `translationKey` string change.

## 4. Constant name vs key — a decision point baked in

The `displayName.*` keys are renamed to `user.display_name.*` for prefix consistency with the rest of `UserErrorCode`. The constant names (`DISPLAY_NAME_REQUIRED` etc.) stay — they are the wire `code` and the Jakarta message string; renaming them would ripple into every `@Pattern(message="DISPLAY_NAME_PATTERN")` annotation and is out of scope. Key-prefix-only.

## 5. Consumer updates (backend)

Every reference to a moved constant changes its enum qualifier (`ProductErrorCode.X` → `SystemErrorCode.X` / `UserErrorCode.X`). Confirmed consumer sites from the audit:

- `RateLimitFilter.java:32,34,77` — `RATE_LIMITED`
- `CurrentLanguageFilter.java:50-55,85-88,106-111` — `LANG_MISSING_OR_INVALID`
- `ProductSearchController.java:56,68`, `PublicProductController.java:44` — `BASE_SITE_MISSING_OR_INVALID`
- `GlobalExceptionHandler.java:42-45` — `NOT_AUTHENTICATED` / `ACCESS_DENIED`
- `GlobalExceptionHandler.java:168` — `INTERNAL_ERROR` (catch-all fallback)
- `DashboardProductController.java:60`, `DefaultProductService.java:228,648` — `NOT_OWNER`
- User/profile code consumers (`USER_SETUP_INCOMPLETE`, `DISPLAY_NAME_*`) in the product-create, login, and update-user paths — the engineer greps each constant name and updates every qualifier.

These line numbers are audit-time hints, not assertions. The engineer greps each constant name and updates every hit; the brief will say so.

## 6. `resolveTranslationKey` rework

Replace the linear `ProductErrorCode.valueOf → ReportErrorCode.valueOf` chain (`GlobalExceptionHandler.java:178-193`) with a single lookup. Build a static `Map<String, String>` once at class load from the `.name() → getTranslationKey()` pairs of all four enums. `resolveTranslationKey(message)` returns `map.get(message)` (or null + the existing WARN on miss).

Guard against name collisions across enums: if two enums ever share a constant name, the map build must fail loudly at class load (not silently last-write-wins). The engineer asserts uniqueness when building the map and throws on duplicate.

## 7. Translation seed changes

28 seed rows for the 7 `system.*` keys (7 × 4 locales) and 16 rows for the 4 `user.*` keys (4 × 4 locales) get their `key` column renamed in place. No text-value change, no ID change, no new rows. Files: the four `0001-data-web-translations-<LOCALE>.sql`. The audit has exact IDs and line numbers per key; the brief restates them.

Locale IDs: 1=SR(RS), 2=CNR, 3=EN, 4=RU.

## 8. Seed-coverage tests

Author three tests cloning `ProductErrorCodeTest`'s two-assertion shape (non-blank key + httpStatus; every key resolves in the EN seed filtered to namespace `ERRORS`):

- `SystemErrorCodeTest`
- `UserErrorCodeTest`
- `ReportErrorCodeTest` (closes the pre-existing gap — `ReportErrorCode` has no seed-coverage test today)

All renamed keys stay in the `ERRORS` namespace, so the namespace filter in each cloned test is unchanged from the `ProductErrorCodeTest` original.

## 9. Web changes (lockstep, on `stage`)

Web keys UI translation off `translationKey`, not `code`. The enum split is invisible to web. The key rename touches web in exactly one production spot plus two test fixtures:

- `src/lib/service/reactCalls/productService.ts:59` — `RATE_LIMITED_TRANSLATION_KEY = 'product.system.rate_limited'` → `'system.rate_limited'`
- `parseProductValidationErrors.test.ts` and `productService.test.ts` — fixture literals `product.system.rate_limited` and `product.system.internal_error` → `system.rate_limited` / `system.internal_error`

No other web change. The wire `code` web branches on (`USER_BANNED`, `RATE_LIMITED`, etc.) is unchanged.

## 10. Ordering constraint

The web `productService.ts:59` change and the backend `system.rate_limited` seed rename must land together, or the 429 synth injects a key the seed no longer has (or vice versa). The brief sequences the backend seed rename and the web literal change to be committed in the same window. The rename is in place (the key never exists under both names at once), so the web brief is sequenced immediately after the backend seed-rename brief, before any deploy.

## 11. Task list / brief order

1. Backend: create `SystemErrorCode` + `UserErrorCode`, move the 11 constants, update all consumers, rework `resolveTranslationKey` to the registry, rename the 44 seed rows, author the three seed-coverage tests. One brief (the moves, consumers, registry, seeds, and tests interlock — splitting them would leave the build red between briefs).
2. Web: the one-line literal + two test fixtures. Separate brief (different repo, Part 3 — no cross-repo briefs).

## 12. Trust boundaries

None affected. All 11 moved codes are error responses, not decision inputs — the audit confirmed every one clean (Part 11). Moving a code between enums has zero trust-boundary implication.

## 13. Definition of done

- Backend: `SystemErrorCode` + `UserErrorCode` exist; `ProductErrorCode` holds only the 36 product codes; all consumers updated; `resolveTranslationKey` is registry-based with a duplicate-name guard; 44 seed rows renamed; three new seed-coverage tests pass. `./mvnw spotless:check` + `./mvnw test` green.
- Web: `productService.ts:59` + two test fixtures renamed; `npm run lint`, `npx tsc --noEmit`, `npm test` green.
- No wire `code` changed; no error response shape changed.

## Factual vs inferred

- **Factual** (from audits): the 11 constants and their consumers; the `ReportErrorCode` template shape; `resolveTranslationKey` mechanism and extension point; the 44 seed rows and their IDs/lines; web keying off `translationKey`; the single web literal + two fixtures; trust-boundary cleanliness.
- **Inferred** (Mastermind framing, confirmed Phase 3/4 by Igor): the two-enum split (vs one `ErrorCode` or 7-only); the `user.display_name.*` key reshape (audit only noted `displayName.*` is clean; the move to a `user.` prefix is this spec's call for `UserErrorCode` consistency, confirmed by Igor); the registry-with-duplicate-guard shape (audit suggested a registry; fail-loud-on-collision is added here).

## Session log

- **2026-05-29** — Code-complete across backend (`dev`) and web (`stage`). Backend (`2026-05-29-oglasino-backend-system-error-code-split-1`): `ErrorCode` interface implemented by all four enums, 11 constants split out (`SystemErrorCode` ×7, `UserErrorCode` ×4), `ProductValidationException`/`ProductErrorResponse` widened, `resolveTranslationKey` reworked to a fail-loud registry, 44 seed rows renamed in place, three new seed-coverage tests (`SystemErrorCodeTest`, `UserErrorCodeTest`, `ReportErrorCodeTest`); full suite green (686 tests). Web (`2026-05-29-oglasino-web-system-error-code-split-1`): the `product.system.rate_limited` literal + four fixtures renamed across three test files (one file + one suffix beyond the brief's enumeration, Igor-authorized), grep-confirmed zero `product.system.*` literals remain; 39 tests across the three touched files green, `tsc`/lint clean. Status flipped `planned` → `web-stable`. No mobile-adoptable surface (backend-internal refactor + web string rename).
