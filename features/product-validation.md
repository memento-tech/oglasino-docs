# Product validation

Server-side content moderation and structural validation for product create
and update flows. Backend is the trust boundary for all moderation,
authorization, and state-transition decisions. Backend returns structured
error codes with translation keys; frontend renders translated messages
directly.

**Status:** web-stable
**Branch:** `feature/validation-refactor`

---

## Table of contents

- [Goals and non-goals](#goals-and-non-goals)
- [Trust boundary principles](#trust-boundary-principles)
- [Endpoints](#endpoints)
- [Wire shape](#wire-shape)
- [HTTP status codes](#http-status-codes)
- [Error codes](#error-codes)
- [Request DTOs](#request-dtos)
- [Create flow](#create-flow)
- [Update flow](#update-flow)
- [Content moderation chain](#content-moderation-chain)
- [Configurable thresholds](#configurable-thresholds)
- [Translation requirements](#translation-requirements)
- [Step-to-field mapping](#step-to-field-mapping)
- [Platform adoption](#platform-adoption)
- [Out of scope](#out-of-scope)
- [Known gaps tracked for follow-up](#known-gaps-tracked-for-follow-up)

---

## Goals and non-goals

### Goals

- Server is the single source of truth for content moderation and change
  detection.
- Wire contract is uniform across every product-validation response shape:
  validation failures, auth failures, rate limits, and unhandled errors all
  return the same shape.
- Errors are codes, never message text. Backend ships the translation key;
  frontend translates locally.
- Step 2 of the create wizard catches content problems early via a
  dedicated pre-validate endpoint, so users don't reach step 4 to discover
  moderation rejections.
- Update endpoint short-circuits if nothing changed. No moderation calls,
  no OpenAI translation runs, no Elasticsearch reindex, no DB write.
- All numeric thresholds in the moderation chain are seeded values in
  `data-configuration.sql` and read via `ConfigurationService`. The seed
  is the single source of truth — no hardcoded fallbacks in code.

### Non-goals (in this feature)

- Mobile (Expo) adoption. The backend and web are shipped; mobile adopts
  the frozen contract in a separate Mastermind chat opened after merge.
  See [Platform adoption](#platform-adoption).
- Image ordering. The update endpoint currently treats `imageKeys` as a
  set (order-independent). Ordering support is tracked as a known gap.
- `imageKeys` caller-ownership verification against
  `RedisUploadOwnershipService` on create and update. Tracked as
  follow-up.
- Content moderation v2 (policy scoring engine). Parked in
  `future/content-moderation-v2.md`.

---

## Trust boundary principles

Conventions Part 11 applies in full. Specifically for this feature:

- **Server reads all "previous" state from the database.** Client never
  supplies `oldName`, `oldDescription`, or any other "before" value used in
  change-detection, moderation, or authorization decisions.
- **`freeZone` is derived from `topCategory.isFreeZone()` server-side.**
  Never trusted from client DTO.
- **Image keys are accepted from the client but written to the product
  as-is.** Caller-ownership verification against
  `RedisUploadOwnershipService` is a known gap (out of scope here).
- **Language is resolved server-side** from
  `LanguageContext.getCurrentUserPreferredLanguage()`, which is DB-derived
  via `OglasinoAuthentication.getPreferredLanguage()`. Never from request
  body, never from `X-Lang` header for moderation purposes.
- **Update endpoint short-circuits server-side** if no mutable field has
  changed against the persisted product. The frontend's "did anything
  change" check is a UX optimization; the backend's is a correctness
  guarantee.

For each field on every request DTO, the spec specifies below whether the
value is trusted from client, derived server-side, or read from DB.

---

## Endpoints

Three endpoints are in scope.

### `POST /api/secure/products/create`

- **Request:** `NewProductRequestDTO`
- **Response (2xx):** `NewProductResponseDTO` containing `{id, name}`
- **Rate limit:** `EXPENSIVE_WRITE`
- **Behavior:** validates Jakarta constraints (400 on fail) and runs
  content moderation on name and description (via `@ValidProductName` and
  `@ValidDescription`). Then runs business-rule validation in
  `DefaultProductService.createProduct` (422 on fail). Then persists.

### `POST /api/secure/products/update`

- **Request:** `UpdateProductRequestDTO`
- **Response (2xx):** `NewProductResponseDTO` containing `{id, name}`. In
  the no-op case (no mutable fields changed), still returns 200 with the
  existing product — the response is indistinguishable from a successful
  update from the client's perspective.
- **Rate limit:** `EXPENSIVE_WRITE`
- **Behavior:**
  1. Load the persisted product. If not found, throw
     `ProductValidationException(null, PRODUCT_NOT_FOUND)` → 422.
  2. Owner check: if caller's user id != `product.getOwner().getId()`,
     throw `ProductValidationException(null, NOT_OWNER)` → 403. This is a
     security event, not a UX failure.
  3. No-op short-circuit: if every mutable field on the incoming DTO
     (`name`, `description`, `price`, `currency`, `filters`, `imageKeys`)
     is identical to the persisted value, skip the rest of the method and
     return the existing product as the success response. `imageKeys`
     comparison is set-based (order-independent in this iteration; see
     known gaps).
  4. Run Jakarta validation (already done by `@Valid` before the method
     body runs, but conceptually first).
  5. Run content moderation on name and description (via the annotation
     validators).
  6. Run `MASSIVE_CHANGE` detection (see below).
  7. Run business-rule validation (currency allowed, price ≥ 1 if not free
     zone, filter validity).
  8. Persist, re-translate any changed fields, reindex.

- **`MASSIVE_CHANGE` detection (server-side, post-load):**
  - Compares incoming `name` against persisted `name`. Same logic for
    `description`. Per-field independent checks.
  - The persisted side is read via `resolvePersistedField`: first the
    translation row matching `LanguageContext.getCurrentUserPreferredLanguageCode()`,
    then the row with `autoTranslated == false` (the user's original
    entry), then null (skip).
  - Threshold criteria (all three must hold to flag): Levenshtein
    similarity < `validation.massive_change.similarity_threshold`, no word
    overlap, length delta > `validation.massive_change.length_delta_threshold`.
  - Both thresholds are read on every call via
    `ConfigurationService#getRequiredDoubleConfig` (no cached fallback).
  - On flag: throw `ProductValidationException("name", NAME_MASSIVE_CHANGE)`
    or `ProductValidationException("description", DESCRIPTION_MASSIVE_CHANGE)`
    → 422. If both flag in one call, the exception carries two
    `FieldError`s and the handler picks 422 by severity-precedence
    routing.
  - `MassiveNameChangeValidator` (the prior Jakarta annotation validator)
    and the `@MassiveNameChange` annotation are deleted. The check lives
    in `DefaultProductService.updateProduct` after loading the persisted
    product.

### `POST /api/secure/products/pre-validate`

- **Request:** `PreValidateProductRequestDTO` containing `name` and
  `description` only. No `language` field — language is resolved
  server-side.
- **Response:** always `200 OK` with `ProductErrorResponse`. Body is
  `{"errors": []}` on clean content, `{"errors": [...]}` on violations.
  Returns all violations across both fields (not first-only — frontend
  decides display strategy).
- **Rate limit:** `MODERATION_PROBE` (stricter burst than
  `EXPENSIVE_WRITE` because pre-validate is expected to be called more
  frequently than create/update).
- **Behavior:** runs `ContentModerator.moderate(name, PRODUCT_NAME)` and
  `.moderate(description, PRODUCT_DESCRIPTION)`. No DB writes, no business
  rules, no `MASSIVE_CHANGE` (which requires a stored product). Just
  content moderation on the two text fields.
- **Scope:** create flow only. Update flow does NOT call pre-validate —
  update has single-shot validation at save.

---

## Wire shape

**Every** product-related error response uses this shape:

```json
{
  "errors": [
    {
      "field": "name",
      "code": "NAME_BANNED_WORDS",
      "translationKey": "product.name.banned_words"
    }
  ]
}
```

- `field` — DTO property path. `null` for object-level violations
  (rate limits, auth failures, internal errors, missing user setup).
- `code` — `ProductErrorCode.name()`, the stable enum constant.
- `translationKey` — fully-qualified translation key in the `ERRORS`
  namespace. Each `ProductErrorCode` constant carries its own
  `translationKey` and `httpStatus` fields, populated at construction.
  Response serializer copies `translationKey` onto every field-error in
  the response.

**No message text on the wire.** Ever. The frontend renders the
translation key locally.

**First error per field wins on the client.** Backend may return multiple
violations per field (especially from pre-validate, which returns all);
frontend's parser keeps the first per field.

The reserved client-side key `__system` (exported from
`parseProductValidationErrors` as `SYSTEM_ERROR_KEY`) carries
cross-cutting violations where `field` is `null` (rate limit, auth,
internal error).

---

## HTTP status codes

| Status | When                                                                                  |
| ------ | ------------------------------------------------------------------------------------- |
| 200    | Success, or the pre-validate response (regardless of violations)                      |
| 400    | Jakarta constraint failed (required, length, type, custom annotation validators)      |
| 403    | Auth-related: not authenticated (`NOT_AUTHENTICATED`), access denied (`ACCESS_DENIED`), or attempting to modify another user's product (`NOT_OWNER`). The last case is a security event. |
| 422    | Business-rule failed after Jakarta passed: currency not allowed, price required, filter invalid, massive change, category/product not found, user setup incomplete |
| 429    | Rate limit exceeded (`RATE_LIMITED`). `RateLimitFilter` writes the unified body shape directly. |
| 500    | Unhandled server error (`INTERNAL_ERROR`). If validation logic produces this, it's a bug. |

All status codes return the same wire shape (see above). 403, 429, and
500 emit codes with `field: null`. When a `ProductValidationException`
carries multiple field-errors, `GlobalExceptionHandler#resolveStatus`
picks the highest-severity status across all codes (severity order:
403 → 422 → 400).

---

## Error codes

`ProductErrorCode` enum. Each constant declares both its `translationKey`
and its default `httpStatus`. The handler reads `httpStatus` on the
driving code to pick the response status; `translationKey` is copied onto
every field-error in the serialized response.

The enum carries 42 constants total.

### Cross-cutting (system-level, all endpoints)

| Code                | Status | When                                                  | Translation key                       |
| ------------------- | ------ | ----------------------------------------------------- | ------------------------------------- |
| `RATE_LIMITED`      | 429    | Rate limit exceeded; `Retry-After` header set         | `product.system.rate_limited`         |
| `NOT_AUTHENTICATED` | 403    | Request not authenticated for a secure endpoint       | `product.system.not_authenticated`    |
| `ACCESS_DENIED`     | 403    | Authenticated but lacks required role/permission      | `product.system.access_denied`        |
| `INTERNAL_ERROR`    | 500    | Unhandled exception caught by `GlobalExceptionHandler`| `product.system.internal_error`       |

### Name field

| Code                          | Status | When                                                                                     | Translation key                              |
| ----------------------------- | ------ | ---------------------------------------------------------------------------------------- | -------------------------------------------- |
| `NAME_REQUIRED`               | 400    | `name` null or blank                                                                     | `product.name.required`                      |
| `NAME_TOO_LONG`               | 400    | `name` > 80 chars                                                                        | `product.name.too_long`                      |
| `NAME_REPEATING_CHARS`        | 400    | N+ consecutive identical chars (threshold configurable, name-side)                       | `product.name.repeating_chars`               |
| `NAME_EXCESSIVE_PUNCTUATION`  | 400    | N+ consecutive punctuation chars (threshold configurable)                                | `product.name.excessive_punctuation`         |
| `NAME_ALL_CAPS`               | 400    | All uppercase and length > N (threshold configurable)                                    | `product.name.all_caps`                      |
| `NAME_BANNED_WORDS`           | 400    | Contains a banned word for the detected language                                         | `product.name.banned_words`                  |
| `NAME_LEADING_TRAILING_SPACES`| 400    | Starts or ends with whitespace                                                           | `product.name.leading_trailing_spaces`       |
| `NAME_KEYWORD_STUFFING`       | 400    | A word of length ≥ N appears M+ times (both configurable)                                | `product.name.keyword_stuffing`              |
| `NAME_PROMOTIONS`             | 400    | Contains a promotional keyword for the detected language                                 | `product.name.promotions`                    |
| `NAME_CONTAINS_LINK`          | 400    | Contains URL, domain, email, or HTML tag                                                 | `product.name.contains_link`                 |
| `NAME_CONTAINS_CONTACT`       | 400    | Contains a phone/social/messenger contact for the detected language                      | `product.name.contains_contact`              |
| `NAME_GIBBERISH`              | 400    | Shannon entropy exceeds threshold (length-aware, configurable per language)              | `product.name.gibberish`                     |
| `NAME_MASSIVE_CHANGE`         | 422    | (Update only) new name diverges too far from stored name across three combined criteria  | `product.name.massive_change`                |

### Description field

| Code                                  | Status | When                                                                                | Translation key                                  |
| ------------------------------------- | ------ | ----------------------------------------------------------------------------------- | ------------------------------------------------ |
| `DESCRIPTION_REQUIRED`                | 400    | `description` null or blank                                                         | `product.description.required`                   |
| `DESCRIPTION_TOO_LONG`                | 400    | `description` > 2000 chars                                                          | `product.description.too_long`                   |
| `DESCRIPTION_BANNED_WORDS`            | 400    | Contains a banned word for the detected language                                    | `product.description.banned_words`               |
| `DESCRIPTION_SPAMMY`                  | 400    | Repeating chars or keyword stuffing (description-side thresholds)                   | `product.description.spammy`                     |
| `DESCRIPTION_CONTAINS_LINK`           | 400    | Contains URL, domain, email, or HTML tag                                            | `product.description.contains_link`              |
| `DESCRIPTION_CONTAINS_CONTACT`        | 400    | Contains a phone/social/messenger contact for the detected language                 | `product.description.contains_contact`           |
| `DESCRIPTION_LEADING_TRAILING_SPACES` | 400    | Starts or ends with whitespace                                                      | `product.description.leading_trailing_spaces`    |
| `DESCRIPTION_GIBBERISH`               | 400    | Shannon entropy exceeds threshold (length-aware, configurable per language)         | `product.description.gibberish`                  |
| `DESCRIPTION_MASSIVE_CHANGE`          | 422    | (Update only) new description diverges too far from stored description              | `product.description.massive_change`             |

### Product-level

| Code                              | Status | When                                                                                       | Translation key                            |
| --------------------------------- | ------ | ------------------------------------------------------------------------------------------ | ------------------------------------------ |
| `PRODUCT_ID_REQUIRED`             | 400    | `id` null on update                                                                        | `product.id.required`                      |
| `PRODUCT_NOT_FOUND`               | 422    | Product with given id doesn't exist (replaces bare `orElseThrow` 500s)                     | `product.id.not_found`                     |
| `CATEGORY_REQUIRED`               | 400    | Any of `topCategory`, `subCategory`, `finalCategory` is null on create                     | `product.category.required`                |
| `CATEGORY_NOT_FOUND`              | 422    | Category id doesn't resolve (top/sub/final, replaces bare `orElseThrow` 500s)              | `product.category.not_found`               |
| `PRICE_REQUIRED`                  | 422    | Non-free-zone product with price < 1 (note: name is misleading; renaming deferred)         | `product.price.required`                   |
| `CURRENCY_REQUIRED`               | 400    | `currency` null on create                                                                  | `product.currency.required`                |
| `CURRENCY_NOT_ALLOWED`            | 422    | Currency not in the base-site's allowed list                                               | `product.currency.not_allowed`             |
| `FILTER_NOT_IN_CATEGORY`          | 422    | Filter id not part of selected category tree                                               | `product.filter.not_in_category`           |
| `FILTER_OPTION_NOT_IN_FILTER`     | 422    | Selected option id doesn't belong to the canonical filter                                  | `product.filter.option_not_in_filter`      |
| `FILTER_SINGLE_OPTION_INVALID`    | 422    | Single-option filter has ≠ 1 selected option                                               | `product.filter.single_option_invalid`     |
| `FILTER_MULTI_OPTION_EMPTY`       | 422    | Multi-option filter has no selected options                                                | `product.filter.multi_option_empty`        |
| `FILTER_RANGE_VALUE_REQUIRED`     | 422    | Range filter missing selected range value                                                  | `product.filter.range_value_required`      |
| `FILTER_DATE_VALUE_REQUIRED`      | 422    | Date filter missing year value                                                             | `product.filter.date_value_required`       |
| `FILTER_DATE_VALUE_OUT_OF_RANGE`  | 422    | Date filter year < 1930 or > current year                                                  | `product.filter.date_value_out_of_range`   |
| `NOT_OWNER`                       | 403    | Caller is not the product's owner. Security event.                                         | `product.system.not_owner`                 |
| `USER_SETUP_INCOMPLETE`           | 422    | User lacks base-site, region, or city (create only)                                        | `product.user.setup_incomplete`            |

### Codes removed in this feature

These existed but are deleted:

- `OLD_NAME_REQUIRED`, `OLD_NAME_TOO_LONG`, `OLD_DESCRIPTION_REQUIRED`,
  `OLD_DESCRIPTION_TOO_LONG` — the fields they validated are gone from
  `UpdateProductRequestDTO`.

---

## Request DTOs

### `NewProductRequestDTO`

| Field           | Type                    | Trust              | Jakarta annotations                                  |
| --------------- | ----------------------- | ------------------ | ---------------------------------------------------- |
| `name`          | `String`                | client, moderated  | `@NotBlank("NAME_REQUIRED")`, `@Size(max=80, message="NAME_TOO_LONG")`, `@ValidProductName` |
| `description`   | `String`                | client, moderated  | `@NotBlank("DESCRIPTION_REQUIRED")`, `@Size(max=2000, message="DESCRIPTION_TOO_LONG")`, `@ValidDescription` |
| `price`         | `BigDecimal`            | client, validated server-side (≥ 1 if not free zone) | none |
| `currency`      | `CurrencyDTO`           | client id; validated against `baseSite.allowedCurrencies` | `@Valid`, `@NotNull("CURRENCY_REQUIRED")` |
| `topCategory`   | `CategoryDTO`           | client id; validated via catalog lookup | `@Valid`, `@NotNull("CATEGORY_REQUIRED")` |
| `subCategory`   | `CategoryDTO`           | client id; validated via catalog lookup | `@Valid`, `@NotNull("CATEGORY_REQUIRED")` |
| `finalCategory` | `CategoryDTO`           | client id; validated via catalog lookup | `@Valid`, `@NotNull("CATEGORY_REQUIRED")` |
| `filters`       | `List<SelectedFilterDTO>` | client; structurally validated via `validateFilterData` (filter type and option-set resolved from the canonical catalog, not the client DTO) | none |
| `imageKeys`     | `Set<String>`           | client (ownership check is known gap) | none |

**Contract intent — `free` is not part of the wire DTO.** Web has
dropped `free` from its TypeScript wire shape; backend derives free-zone
from `topCategory.isFreeZone()` server-side. A stale `private boolean
free` field is still present on the backend `NewProductRequestDTO` Java
class (Jackson will tolerate it on incoming JSON, but no production code
reads it). Cleanup tracked in `issues.md`. New clients (mobile) must not
send `free`.

### `UpdateProductRequestDTO`

**No longer extends `NewProductRequestDTO`.** Standalone class.

| Field         | Type                    | Trust                                            | Jakarta annotations                                            |
| ------------- | ----------------------- | ------------------------------------------------ | -------------------------------------------------------------- |
| `id`          | `Long`                  | client; product looked up; ownership re-checked  | `@NotNull("PRODUCT_ID_REQUIRED")`                              |
| `name`        | `String`                | client, moderated; compared to stored for massive-change | `@NotBlank("NAME_REQUIRED")`, `@Size(max=80, message="NAME_TOO_LONG")`, `@ValidProductName` |
| `description` | `String`                | client, moderated; compared to stored for massive-change | `@NotBlank("DESCRIPTION_REQUIRED")`, `@Size(max=2000, message="DESCRIPTION_TOO_LONG")`, `@ValidDescription` |
| `price`       | `BigDecimal`            | client; validated `≥ 1` server-side when present | none                                                           |
| `currency`    | `CurrencyDTO`           | optional; validated when present                 | `@Valid`                                                       |
| `filters`     | `List<SelectedFilterDTO>` | client; validated                              | none                                                           |
| `imageKeys`   | `Set<String>`           | client                                           | none                                                           |

**Fields removed:** `oldName`, `oldDescription` (trust-boundary violation,
now server-derived); `topCategory`, `subCategory`, `finalCategory`,
`regionAndCity` (immutable, read from stored product); `free`,
`productState`, `moderationState` (never read on update path).

The class-level `@MassiveNameChange` annotation is removed. The check
moves to `DefaultProductService.updateProduct` after loading the
persisted product.

### `PreValidateProductRequestDTO`

| Field         | Type     | Trust              | Jakarta annotations                                       |
| ------------- | -------- | ------------------ | --------------------------------------------------------- |
| `name`        | `String` | client, moderated  | `@NotBlank("NAME_REQUIRED")`, `@Size(max=80, message="NAME_TOO_LONG")` |
| `description` | `String` | client, moderated  | `@NotBlank("DESCRIPTION_REQUIRED")`, `@Size(max=2000, message="DESCRIPTION_TOO_LONG")` |

No `language` field. Language is resolved server-side from
`LanguageContext.getCurrentUserPreferredLanguage()`.

---

## Create flow

Four-step wizard in a single popup. Client state local to the wizard
component, not in a Zustand store.

### Step 1 — Images

- Component: `ImageSelectionProductDialog.tsx`
- User selects 0 to 5 images via `ImagesImport`.
- HEIC/HEIF converted to JPEG client-side at picker time.
- Held in memory as `ImageData[]` with `file?: File` and `key?: string`.
  Step 1 only sets `file`.
- **On Next:** client-side validation only:
  - MIME type in allowlist: `image/jpeg`, `image/png`, `image/webp`
    (`product.image.invalid_type`).
  - Each file ≤ 5 MB (`product.image.too_big`).
  - No duplicate files via SHA-256 dedup (`product.image.duplicate`).
  - Max 5 images (enforced by `ImagesImport.maxNumberOfImages={5}`).
  - MIME runs first, then size, then dedup — first-failure-wins on
    ordering.
- If validation fails: errors shown inline next to the image picker.
  Cannot advance.

<!-- Screenshot: the create-product wizard on step 1 (image selection)
     immediately after the user tries to add a file that fails one of
     the client-side image checks. Required state: two or three image
     thumbnails already added in the picker grid (so the grid is
     populated, not empty), and a single inline error message rendered
     in red italic directly under the picker. The error must be one of
     the three product-image keys — the cleanest one to capture is the
     duplicate case ("This image has already been added.", from
     `product.image.duplicate`), but a too-big or invalid-type error
     reads equally well. The Next button is visible and ENABLED (failed
     adds don't block step 1; the user clears the error by adjusting
     the selection, not by retrying Next). Capture the dialog framing
     including the wizard step indicator so the reader sees which step
     they are looking at. -->
![Create wizard step 1, image picker showing a duplicate-image inline error](assets/create-wizard-step-1-image-error.png)

- **No server call at this step.** No R2 upload yet.

### Step 2 — Basic info

- Component: `BasicInfoProductDialog.tsx`
- User fills: top/sub/final category, name, description, price (if
  category is not free-zone), currency (same condition).
- Region and city are read-only, displayed from `user.regionAndCity`.
- **Client structural validation on every field** via Zod schema
  (`createProductSchema`):
  - `name` ≥ 3 and ≤ 80 chars.
  - `description` ≥ 10 and ≤ 2000 chars.
  - `topCategory`, `subCategory`, `finalCategory` all required (single
    inline error at the earliest empty selector — categories collapsed
    to one message).
  - `price` required when `topCategory && !topCategory.freeZone` —
    checked imperatively in `validateProduct` (not Zod), using
    `priceSchema.safeParse` and a blank-after-trim guard. Both
    missing-and-invalid map to the key `product.price.required`.

<!-- Screenshot: the create-product wizard on step 2 (basic info) after
     the user clicked Next on a partially-filled form, with MULTIPLE
     client-side structural errors visible simultaneously. Required
     state, all visible in one frame:
       - The combined category selector showing a single inline error
         rendered against the first empty selector — the collapsed
         "categories required" message, not three identical ones.
         Translated from `product.category.required` ("Please select a
         category for your product.").
       - The name input showing an inline error in red italic next to
         the field — any of `product.name.required` /
         `product.name.too_short` / `product.name.too_long` is fine
         (a 1- or 2-char value triggers too_short cleanly).
       - The description input showing an inline error next to the
         field (e.g. `product.description.required` rendered as
         "Please enter a product description.").
       - A non-free-zone category selected so the price input is
         rendered, with an inline error under it from
         `product.price.required` ("Please enter a price of at least 1.").
       - The italic helper text under the Next/Back action bar
         (`form.incomplete`) visible in red, because at least one of
         name or description has an error.
     Capture the full dialog framing including the wizard step
     indicator. All four errors visible at once is the load-bearing
     image — it shows that errors render inline next to each field,
     not as a banner at the top. -->
![Create wizard step 2 with inline validation errors on category, name, description, and price fields](assets/create-wizard-step-2-inline-errors.png)

- **On Next:**
  1. Client structural validation runs first. If any of `name`,
     `description`, `images`, `topCategory`, `subCategory`,
     `finalCategory`, `price` has an error, errors render inline next to
     the offending field. Cannot advance.
  2. If structural passes, **call `POST /api/secure/products/pre-validate`**
     with `{name, description}` (via `preValidateProductBasics`).
  3. On `{type: 'clean'}` advance to step 3.
  4. On `{type: 'validation'}` render the errors inline next to `name`
     and `description` fields, and if a `__system` entry is present
     (rate limit), show an inline message under the Next button. The
     button stays disabled for a fixed 5-second backoff (Retry-After
     parsing is deferred — see known gaps).
  5. On `{type: 'error'}` (transport, 5xx, unparseable): allow advance
     to step 3 with a warning toast
     (`DIALOG.new.product.pre.validate.warning`). Pre-validate is a UX
     optimization, not a security boundary. Step 4 is the real
     boundary.

<!-- Screenshot: the create-product wizard on step 2 in the rate-limit
     state, captured during the 5-second backoff window after a
     pre-validate call returned a 429 / __system entry. Required
     state:
       - The form has valid-looking content in the name and description
         fields (no inline field errors — the structural check passed,
         which is why pre-validate was called at all).
       - The Next button is DISABLED (visibly greyed/non-interactive,
         no spinner — the request has already returned, the disable is
         just the backoff hold).
       - The Back button is enabled.
       - Directly below the action bar, the inline rate-limit message
         is visible in red and centered, rendered from
         `product.system.rate_limited` ("You're making requests too
         quickly. Please wait a moment and try again.").
     This image is load-bearing because the disabled-Next-with-inline-
     reason pattern is the only place in the feature where a server
     error short-circuits an otherwise valid form, and the visual is
     hard to picture from prose alone. -->
![Create wizard step 2 in the rate-limit backoff state, Next disabled with inline rate-limit message](assets/create-wizard-step-2-rate-limit.png)

### Step 3 — Filters

- Component: `MetaDataProductDialog.tsx`
- User selects filter values. No client-side enforcement of filter
  requirements — user can pick any valid combination.
- **On Next:** reCAPTCHA verification (existing behavior). No content
  validation at this step.

### Step 4 — Submit

- Component: `UploadedProductDialog.tsx`
- On mount (single-shot):
  1. Upload images to R2 via `/secure/images/upload-tokens` and direct
     PUT to the returned URLs. Progressive status: "Uploading images..."
  2. POST `/secure/products/create` with the full DTO. Status:
     "Creating product..."
  3. On success: status "Created" + link to the new product.

- **On failure at step 4:**
  - If `result.type === 'validation'` (400/422): render the errors as a
    list inside the popup, with a header
    (`DIALOG.new.product.create.failed.header`) listing each error's
    translated message. Provide two buttons:
    - **"Go back and fix"** (`DIALOG.new.product.create.failed.fix`) —
      routes to the step containing the earliest unmappable error using
      `resolveTargetStep` (helpers in `productStepMapping.ts`).
      `__system`-only or unresolvable defaults to step 3.
    - **"Exit"** (`DIALOG.new.product.create.failed.exit`) — closes the
      popup.
  - If `result.type === 'error'` (network, 5xx, unexpected): cleanup any
    R2 keys uploaded this session via `cleanupOrphanImages`. Show generic
    failure message + the same two-button UX, with a fixed step-3 jump
    on "Go back and fix."
  - If 429 at any sub-step: the parsed `__system` rate-limit message is
    rendered alongside other errors.

<!-- Screenshot: the create-product wizard on step 4 (submit) AFTER the
     POST /api/secure/products/create returned a validation failure
     (400 or 422). Required state, all visible in one frame:
       - The failure header text at the top of the popup body, rendered
         from `DIALOG.new.product.create.failed.header` ("We couldn't
         create your product. Please fix the following:").
       - Below the header, a bulleted list of two or three translated
         error messages — pick a mix that shows variety:
           * a name banned-words or gibberish error
             (e.g. "Your product name contains words that aren't
             allowed.")
           * a description error (e.g. "Description can't contain
             links.")
           * optionally a category, price, or filter error
         The list is the load-bearing element: it shows that step 4
         consumes `result.errors` instead of discarding them.
       - At the bottom of the popup, the two-button action bar:
         "Go back and fix" on the left (primary action, from
         `DIALOG.new.product.create.failed.fix`) and "Exit" on the
         right (secondary, from `DIALOG.new.product.create.failed.exit`).
       - Capture in the popup overlay context so the dimmed page
         background behind the dialog is visible — this disambiguates
         from a non-modal error surface. -->
![Create wizard step 4 failure UX, error list with Go back and fix and Exit buttons](assets/create-wizard-step-4-failure-list.png)

### Wire payload from step 4

The POST `/secure/products/create` body strips `imagesData` (client-only
field, contains File objects). Sends every other field of
`NewProductRequestDTO`.

---

## Update flow

Single page, all fields visible at once. Page:
`app/[locale]/owner/products/[productId]/page.tsx`.

### Page load

- Fetch `GET /secure/products?productId=...` → returns the current
  product mapped to a `ProductEditState` (editable view-model: mutable
  wire fields plus immutable display fields such as `productState`,
  `topCategory`, `subCategory`, `finalCategory`, `regionAndCity`,
  `moderationState`).
- Frontend stores two copies in state:
  - `productDetails` — the editable working copy.
  - `oldProductDetails` — the unmodified baseline for change detection.
- `productService.toUpdateWirePayload` narrows `ProductEditState` to the
  pure 7-field `UpdateProductRequestDTO` at the trust boundary on save.

### Save flow

1. **Client change-detection:** `deepEqualTest(productDetails,
   oldProductDetails, [])` across all fields. If identical, render
   inline message under the Save button: "No changes to save." Do not
   call backend.
2. **Client structural validation:** run Zod via `validateProduct(name,
   description, price, top, sub, final, imagesData, true)`. Same rules
   as the create wizard's step 2 — categories required, price required
   when non-free-zone, image MIME/size/dedup. Errors render inline next
   to each field. Cannot save.
3. **Image upload (if any new files):** same as create step 4. New keys
   uploaded; failures cleanup uploaded keys.
4. **POST `/secure/products/update`** with the narrowed DTO.
5. **Server-side processing:**
   - Load persisted product. Owner check (`NOT_OWNER` → 403 if
     mismatch).
   - **Server no-op short-circuit:** if every mutable field is identical
     to persisted (set-based comparison for `imageKeys` and filter
     tuples), return existing product as success. No moderation, no
     translation, no reindex, no DB write. Response is HTTP 200 with the
     product DTO, indistinguishable from a real update.
   - If changed: run content moderation, run `MASSIVE_CHANGE` detection
     per-field (against stored values), run business-rule validation,
     persist, re-translate changed text fields, reindex.
6. **On response:**
   - Success: await `getDashboardProductDetails(id)` to refresh the
     snapshot, re-seed both `oldProductDetails` and `productDetails`, and
     show a success toast. On rare refresh failure, fire a warning toast
     (`product.update.refresh.fail`) and leave the stale snapshot in
     place — recoverable.
   - Validation failure (400/422): write errors to `productErrors`
     state. Each field component renders its error inline. Scroll first
     error into view. Save button re-enabled.
   - Generic failure (transport/5xx): single user-facing message via
     `product.update.fail.message` (toast + inline). Save button stays
     enabled for immediate retry.

---

## Content moderation chain

### Architecture

```
ContentModerator (interface)
└── LocalContentModerator (@Component)
    │
    ├── 1. LanguageDetectorAnalyzer  (always first; writes LanguageSignal)
    │
    └── 2. Field-specific analyzer chain (reads LanguageSignal)
```

The interface is the swap surface. To replace with an AI moderator,
implement `ContentModerator` and register the bean as `@Primary`. No
validator or DTO changes needed.

### PRODUCT_NAME chain (in order)

1. `LanguageDetectorAnalyzer` — writes `LanguageSignal` into context
2. `RepeatingCharsAnalyzer` → `NAME_REPEATING_CHARS`
3. `ExcessivePunctuationAnalyzer` → `NAME_EXCESSIVE_PUNCTUATION`
4. `AllCapsAnalyzer` → `NAME_ALL_CAPS`
5. `BannedWordsAnalyzer` → `NAME_BANNED_WORDS` (per-language)
6. `LeadingTrailingSpaceAnalyzer` → `NAME_LEADING_TRAILING_SPACES`
7. `KeywordStuffingAnalyzer` → `NAME_KEYWORD_STUFFING`
8. `PromotionAnalyzer` → `NAME_PROMOTIONS` (per-language)
9. `LinkAnalyzer` → `NAME_CONTAINS_LINK`
10. `ContactAnalyzer` → `NAME_CONTAINS_CONTACT` (per-language)
11. `GibberishAnalyzer` → `NAME_GIBBERISH` (length-aware, per-language)

### PRODUCT_DESCRIPTION chain (in order)

1. `LanguageDetectorAnalyzer`
2. `BannedWordsAnalyzer` → `DESCRIPTION_BANNED_WORDS` (per-language)
3. `SpammyDescriptionAnalyzer` → `DESCRIPTION_SPAMMY` (uses
   description-side repeating-chars threshold and shared
   `KeywordStuffingDetector` with description-side tuning)
4. `LinkAnalyzer` → `DESCRIPTION_CONTAINS_LINK`
5. `ContactAnalyzer` → `DESCRIPTION_CONTAINS_CONTACT` (per-language)
6. `LeadingTrailingSpaceAnalyzer` → `DESCRIPTION_LEADING_TRAILING_SPACES`
7. `GibberishAnalyzer` → `DESCRIPTION_GIBBERISH` (length-aware,
   per-language; threshold tuned for long-form text)

### Behavior

- `LocalContentModerator.moderate()` runs every analyzer in the chain
  and returns all violations.
- `ProductNameValidator` and `ProductDescriptionValidator` (Jakarta layer)
  emit only the first violation per field.
- The pre-validate endpoint returns all violations.
- `KeywordStuffingAnalyzer.KeywordStuffingDetector` is the shared static
  helper that both `KeywordStuffingAnalyzer` (name-side) and
  `SpammyDescriptionAnalyzer` (description-side) call into. Each caller
  supplies its own `Tuning` record (`baseRatio`, `ratioDecayPerWord`,
  `ratioFloor`, `minOccurrenceRatio`). The ratios are hardcoded in the
  call sites — see `issues.md` for the lift-to-config follow-up.

### Language detection

`LanguageDetectorAnalyzer` writes `LanguageSignal(language, confidence)`
to `AnalysisContext`. Detection uses script analysis (Cyrillic vs Latin,
script-distinctive characters), then stopword scoring as a tiebreaker.

The detector never returns a violation. Language mismatch (confidence
≥ 0.7 and detected ≠ expected) is `log.warn`-only. Cross-border listings
are legitimate.

Expected language comes from
`LanguageContext.getCurrentUserPreferredLanguage()` (DB-derived). Used by
the detector as a fallback for short/ambiguous inputs.

---

## Configurable thresholds

Every numeric threshold is read from `ConfigurationService` against the
seeded `data-configuration.sql`. **There are no hardcoded fallbacks in
code.** Reads use `getRequiredConfig` / `getRequiredIntConfig` /
`getRequiredDoubleConfig`, each of which throws `IllegalStateException`
on a missing, blank, or unparseable value. The seed is the single source
of truth; admin edits via the admin panel hot-reload on next moderation
call.

**Boot-time audit.**
`ContentValidationConfig#auditRequiredConfig` listens on
`ApplicationReadyEvent` at `@Order(10)` (after
`DefaultConfigurationService#onAppReady` at `@Order(1)` populates the
cache). It enumerates every required key and ERROR-logs any that are
missing or blank. **It does not throw at boot** — a missing key only
throws on the first moderation call that needs it. Operators see the gap
in the boot log; runtime fails loud. A production boot-fail
(`@Profile`-gated throw) is recorded as an enhancement in `issues.md`.

Each config key's `description` field (visible in admin panel) documents
its purpose and recommended bounds. No code-level clamping in this
iteration — admin is trusted to stay within documented bounds (Igor is
the sole admin).

### Schema (seeded values are the single source of truth)

| Config key                                                  | Type   | Seeded value | Notes                                                                  |
| ----------------------------------------------------------- | ------ | ------------ | ---------------------------------------------------------------------- |
| `validation.repeating_chars.threshold`                      | int    | 4            | Name-side. Minimum consecutive identical characters to flag. Range 3–6 |
| `validation.repeating_chars.description_threshold`          | int    | 5            | Description-side. One notch more permissive than name. Range 4–7       |
| `validation.excessive_punctuation.threshold`                | int    | 4            | Minimum consecutive punctuation characters to flag. Range 3–6          |
| `validation.all_caps.min_length`                            | int    | 3            | Minimum length for all-caps detection. Range 3–6                       |
| `validation.keyword_stuffing.min_word_length`               | int    | 5            | Minimum word length considered for stuffing. Range 4–7                 |
| `validation.keyword_stuffing.frequency_threshold`           | int    | 4            | Repetitions of a long word to flag as stuffing. Range 3–6              |
| `validation.massive_change.similarity_threshold`            | double | 0.5          | Levenshtein similarity below which name/description is "different." Range 0.3–0.7 |
| `validation.massive_change.length_delta_threshold`          | double | 0.4          | Length change ratio above which name/description is "different." Range 0.2–0.6 |
| `validation.gibberish.short_text.length_boundary`           | int    | 40           | Character count below which "short text" entropy threshold applies. Above this, "long text" threshold applies. |
| `validation.gibberish.short_text.entropy_threshold.en`      | double | 4.5          | Shannon entropy ceiling for short EN text. Range 4.0–4.7               |
| `validation.gibberish.short_text.entropy_threshold.sr`      | double | 4.5          | Shannon entropy ceiling for short SR text. Range 4.0–4.7               |
| `validation.gibberish.short_text.entropy_threshold.ru`      | double | 4.2          | Shannon entropy ceiling for short RU text. Range 3.8–4.4               |
| `validation.gibberish.long_text.entropy_threshold.en`       | double | 5.5          | Shannon entropy ceiling for long EN text. Range 5.0–6.5. Tuned up during manual testing — legitimate long-form prose typically needs 5.0+ to avoid false positives. |
| `validation.gibberish.long_text.entropy_threshold.sr`       | double | 5.5          | Shannon entropy ceiling for long SR text. Range 5.0–6.5. Tuned up during manual testing. |
| `validation.gibberish.long_text.entropy_threshold.ru`       | double | 5.0          | Shannon entropy ceiling for long RU text. Range 5.0–6.5. Tuned up during manual testing. |

### Existing dynamic config (unchanged, listed for completeness)

| Config key                                  | Description                                                  |
| ------------------------------------------- | ------------------------------------------------------------ |
| `validation.banned_words.{en,sr,ru}`        | Pipe-separated banned word list per language                 |
| `validation.regex.promo.{en,sr,ru}`         | Promotional keyword regex per language                       |
| `validation.regex.contacts.{en,sr,ru}`      | Phone/contact regex per language                             |
| `validation.stopwords.{en,sr,ru}`           | Stopwords for language detection                             |

### Not configurable (wire contract constants)

- `name` max length: 80
- `description` max length: 2000
- `imageKeys` max count: 5
- `price` minimum: 1 (when not free-zone)
- `price` maximum (client-side hint): 9,999,999

These are wire-contract values. Changing requires coordinated update
across backend, web, and mobile.

---

## Translation requirements

### Storage

All `product.*` translation keys live in
`src/main/resources/data/translations/0001-data-web-translations-{EN,RS,RU,CNR}.sql`
under the `ERRORS` namespace. The `0002-data-translations-*.sql` files
no longer contain product-validation rows.

### Namespace

All `ProductErrorCode` translation keys go into the `ERRORS` namespace.
The `VALIDATION` namespace is frozen per conventions Part 6 Rule 1; the
legacy `product.internal.*` keys that lived there have been deleted from
all four language seeds.

### Requirements

- Every `ProductErrorCode` constant has a translation row in all four
  languages: EN, SR (RS), RU, CNR.
- Translations are user-facing messages, not technical paraphrases of
  the code. Tone: helpful, plain, actionable.

### Examples

| Code                  | EN                                                                                                       |
| --------------------- | -------------------------------------------------------------------------------------------------------- |
| `NAME_BANNED_WORDS`   | Your product name contains words that aren't allowed.                                                    |
| `NAME_MASSIVE_CHANGE` | This change is too large. To list a different product, please create a new listing instead of editing this one. |
| `RATE_LIMITED`        | You're making requests too quickly. Please wait a moment and try again.                                  |
| `NOT_OWNER`           | You don't have permission to modify this product.                                                        |
| `INTERNAL_ERROR`      | Something went wrong on our end. Please try again.                                                       |

### Workflow

- Backend agent drafts EN + SR for every new key when adding it to the
  enum. Same session as the code change.
- Backend agent's session summary lists every key needing RU + CNR with
  English context.
- Igor takes that list, requests RU + CNR translations (via Mastermind
  chat or other means), commits them in a follow-up.

---

## Step-to-field mapping

For the create wizard's step 4 failure routing. Defined in the spec so
mobile can reuse it. Implementation: `productStepMapping.ts` (helpers:
`stepForField`, `resolveTargetStep`, `DEFAULT_FALLBACK_STEP = 3`).
1-indexed; consumers subtract 1 when calling `setCurrentStep`.

| Step | Fields                                                                                           |
| ---- | ------------------------------------------------------------------------------------------------ |
| 1    | `images` (errors from client validation only; server doesn't validate images directly)           |
| 2    | `name`, `description`, `price`, `currency`, `topCategory`, `subCategory`, `finalCategory`        |
| 3    | `filters` (any field starting with `filter.`)                                                    |
| 4    | (none — step 4 is where errors land, not originate)                                              |

`__system` only or unmappable codes fall back to step 3.

---

## Platform adoption

This section is the intake material for the mobile (Expo) adoption chat.
It exists to freeze the contract platform-neutrally and to tell the
mobile chat what is open vs closed before its Phase 2 audit begins.

### Part A — The frozen contract (platform-neutral, copy-exactly)

Mobile conforms to everything below without renegotiation.

**Backend is platform-neutral and complete.** Every endpoint
(`/api/secure/products/create`, `/update`, `/pre-validate`) is reused
as-is. There are no mobile-specific routes. There are no mobile-side
backend gaps to close. Mobile adoption is frontend-only RN work.

**The three endpoints** (see [Endpoints](#endpoints) for paths, methods,
request/response shapes, status codes, rate-limit categories):

- `POST /api/secure/products/create` — `NewProductRequestDTO` →
  `NewProductResponseDTO`, `EXPENSIVE_WRITE`.
- `POST /api/secure/products/update` — `UpdateProductRequestDTO` →
  `NewProductResponseDTO`, `EXPENSIVE_WRITE`, no-op short-circuit on
  server.
- `POST /api/secure/products/pre-validate` — `PreValidateProductRequestDTO`
  → `ProductErrorResponse` (always 200), `MODERATION_PROBE`, create flow
  only.

**The full `ProductErrorCode` list** with `translationKey` and
`httpStatus` per constant — see [Error codes](#error-codes). All 42
constants are seeded in EN/SR/RU/CNR under the `ERRORS` namespace; the
mobile i18n layer reads the same keys.

**The wire shape** is `{errors: [{field, code, translationKey}]}` for
every status (200 with violations on pre-validate, 400, 403, 422, 429,
500). See [Wire shape](#wire-shape) for the `field: null` semantics and
the reserved `__system` cross-cutting key on the client.

**Trust boundaries** are defined per-field in [Request DTOs](#request-dtos).
For every request DTO field, trust is one of: trusted-from-client,
derived-server-side, or read-from-DB. Mobile is bound by the same rules:
no `oldName`/`oldDescription` ever, no `free` on the wire, language
resolved server-side from the authenticated user.

**The validation sequence** (fixed regardless of how the UI presents
steps):

- **Client-side: structural only** — required fields, length limits,
  category presence, conditional price, image MIME/size/dedup. No
  content moderation client-side.
- **Pre-validate (create flow only)** — call between the structural
  check and the final submit. Returns all violations on
  name/description; advance only on `{type: 'clean'}`. Transport failure
  warns and lets the user proceed; the real gate is the submit.
- **Submit (create or update)** — backend runs Jakarta, content
  moderation, business rules, and (update only) MASSIVE_CHANGE and
  no-op short-circuit. Returns errors with the unified wire shape.

**The create-flow logic** (UI-neutral): the user picks images, then
basic info, then filters, then submits. The validation sequence
structural → pre-validate → filter step (no validation) → submit is
fixed. Pre-validate fires at the equivalent of web's step-2-to-step-3
transition.

**The update-flow logic**: load product → edit-in-place → client
change-detection (deep-equal against the baseline snapshot, no `oldX`
fields on the wire) → client structural validation (same rules as
create) → submit → on success refresh the snapshot from
`GET /secure/products?productId=...`.

**Conditional rules:**

- Price required iff `topCategory.freeZone === false`. Single key
  `product.price.required` covers both missing and invalid (≤ 0 or
  > 9,999,999).
- Top, sub, and final category are all required. Hierarchy is validated
  server-side via the catalog; client only enforces presence.
- Image MIME allowlist: `image/jpeg`, `image/png`, `image/webp`. Size
  ceiling 5 MB. SHA-256 dedup. Max 5 images.

### Part B — Platform-specific implementation notes

Written as "web does X; mobile does the RN equivalent." These are
framings, not prescriptions.

- **Structural validation.** Web uses Zod (`createProductSchema`,
  `priceSchema`, `validateProduct`). Mobile uses its own structural
  validation layer — same fields, same rules, same conditional price
  logic. The error keys mobile emits should be the same `product.<field>.*`
  keys web emits (`product.name.required`, `product.name.too_long`,
  `product.name.too_short`, `product.description.*`,
  `product.category.required`, `product.price.required`,
  `product.image.invalid_type`, `product.image.too_big`,
  `product.image.duplicate`) so the same SQL seed and the same i18n
  workflow serve both clients.
- **Error rendering.** Web renders errors inline via
  `tErrors(translationKey)` from a single ERRORS namespace. Mobile
  renders the same `translationKey` through its i18n layer. There is no
  discriminator — every client-origin and server-origin product
  validation key lives in `ERRORS`.
- **Rate-limit UX.** Web's `BasicInfoProductDialog` watches
  `productErrors[SYSTEM_ERROR_KEY]` and disables Next for a fixed 5 s
  backoff. Mobile equivalent: disable submit / next, surface the
  `product.system.rate_limited` message inline. Retry-After parsing is
  deferred on web and may stay deferred on mobile.
- **Step-to-field routing on submit failure.** Web uses
  `productStepMapping.ts` and routes "Go back and fix" to the earliest
  step containing a failed field. Mobile applies the same mapping if its
  UI is multi-step; if mobile presents a single-form UI, the mapping is
  unused but the wire contract is unchanged.
- **Wizard vs single-form is an open question for the mobile chat.**
  Web ships a 4-step popup wizard. The mobile chat decides whether to
  mirror that or to use a single-screen form. The validation *sequence*
  (structural → pre-validate → submit) is fixed regardless of UI
  presentation. If single-form, pre-validate fires at the
  client-structural-pass-but-before-submit point; the user perceives one
  submit even though there are two server round-trips.
- **Update-flow snapshot management.** Web maintains
  `oldProductDetails` and refreshes from `GET` after a successful save.
  Mobile mirrors this: hold the loaded baseline, send no `oldX` fields,
  refresh from `GET` on save success.
- **Image pipeline.** Web uploads to R2 at submit time (step 4). The
  same direct-to-R2 flow applies to mobile per
  `oglasino-docs/features/image-pipeline.md` (planned). Mobile picker is
  RN-specific but the upload contract is the same.

### What this section does not do

- Prescribe mobile pseudocode or component structure.
- Specify the wizard-vs-single-form decision.
- Audit `oglasino-expo`'s current state — that is the mobile chat's
  Phase 2 audit.

---

## Out of scope

- Mobile (Expo) adoption of any of this. See
  [Platform adoption](#platform-adoption) — the contract is frozen,
  mobile work is frontend-only RN, separate Mastermind chat opens after
  merge.
- `imageKeys` caller-ownership verification on create/update.
- Image ordering on update.
- Renaming `PRICE_REQUIRED` to something more accurate (it actually
  means "price too low," not "missing"). Cosmetic; not worth
  wire-contract churn.
- Content moderation v2 (policy scoring engine).
  See `future/content-moderation-v2.md`.

---

## Known gaps tracked for follow-up

The detailed entries with severity and call sites live in
[`../issues.md`](../issues.md). Summary of items still open after the
validation refactor ships:

- **`imageKeys` caller-ownership verification.** Backend accepts any R2
  keys from any caller; should verify against
  `RedisUploadOwnershipService` on create and update.
- **Image ordering on update.** `imageKeys` is a `Set<String>`,
  order-independent. Future work: switch to ordered list, expose order
  in UI.
- **`PRICE_REQUIRED` naming.** Misleading; means "price too low," not
  "missing." Cosmetic.
- **Dead `free` field on backend `NewProductRequestDTO`.** Web has
  dropped it; backend still carries the field with zero readers. Cleanup
  chore.
- **`regionAndCity` unread in `createProductSchema`.** Cosmetic.
- **`field-price` scroll target is a no-op on update page.** Inline
  error renders; only the scroll-into-view fails.
- **Boot audit logs ERROR but does not throw.** `@Profile`-gated
  production boot-fail is a possible enhancement.
- **Web component-render test infrastructure missing**
  (`@testing-library/react` not installed). Logic-level coverage is
  comprehensive; dialog-DOM coverage is a future infra chore.
- **Keyword-stuffing ratio multipliers are inline in code, not in
  config.** Deliberately deferred — larger conversation about how much
  of the heuristic should be admin-tunable.
- **Malformed 429 silently blocks create-wizard.** By design after
  `ensureSystemErrorKey` removal; fix-the-source if observed.
- **Legacy unused regex rows in `data-configuration.sql` ids 2–7.**
  Pre-feature config rows superseded by per-language equivalents.
