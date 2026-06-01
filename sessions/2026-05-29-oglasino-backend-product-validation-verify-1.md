# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-29
**Task:** Backend product-endpoint verification for the mobile validation rebuild — read-only confirmation that the live backend matches `features/product-validation.md`, flagging any drift.

## Implemented

- Nothing changed on disk. This was a READ-ONLY audit per the brief. No code, no git ops, no runtime, no DB writes.
- Verified the product create/update/pre-validate endpoint surface, the three request DTOs, the GET reload shape, and the error wire/status routing against the spec.
- Found three drifts, all spec-vs-code documentation drift (not backend bugs), detailed in "For Mastermind" as escalation candidates.

## Answers to the brief

### Q1 — Does `POST /secure/product/addUpdate` still exist? → **Gone.**

Searched all of `src/main/java` for: `addUpdate`, `add-update`, `add_update` (case-insensitive), and any singular `/secure/product` (non-plural) mapping. All returned nothing. The only product write/read controller is `DashboardProductController` (`@RequestMapping("/api/secure/products")`, plural), which exposes `/create`, `/update`, `/pre-validate`, `GET`, `POST` (dashboard search), `DELETE`, `/activate`, `/deactivate`, `/suggestion`, `/autocomplete`, `/canCreateProduct`. There is no combined create-or-update `addUpdate` endpoint. Confirmed gone — the expected answer.

### Q2 — Three spec endpoints exist with documented shapes

Path prefix: controller is mapped at `@RequestMapping("/api/secure/products")` (`DashboardProductController.java:43`). **No `context-path` / `servlet.path` is configured** anywhere in `src/main/resources` (grep returned nothing). So the actual served base is `/api/secure/products` — the `/api` is part of the controller mapping, not a context path. `RateLimitFilter.categorize` matches on the same `/api/secure/products` literal paths. The spec's bare `/secure/products/...` occurrences omit the `/api` prefix → see drift #3.

1. `POST /api/secure/products/create` — `createNewProduct(@RequestBody @Valid NewProductRequestDTO)` returns `ResponseEntity<NewProductResponseDTO>` (`:114-120`). `NewProductResponseDTO` = `{id, name}` (`NewProductResponseDTO.java`). **matches spec.**
2. `POST /api/secure/products/update` — `updateProduct(@RequestBody @Valid UpdateProductRequestDTO)` returns `ResponseEntity<NewProductResponseDTO>` (`:122-128`). No-op short-circuit **confirmed**: `DefaultProductService.updateProduct` (`:236-238`) returns the loaded `product` unchanged when `isNoOpUpdate(...)` is true (after owner check at `:228-231`); the facade maps that same `Product` to `NewProductResponseDTO` (`DefaultProductFacade:32-35`), and the controller wraps it in `ResponseEntity.ok(...)` → **HTTP 200 with the existing product, indistinguishable from a real update.** **matches spec.**
3. `POST /api/secure/products/pre-validate` — `preValidate(@RequestBody @Valid PreValidateProductRequestDTO)` returns `ResponseEntity<ProductErrorResponse>`, always `ResponseEntity.ok(...)` → 200 regardless of violations (`:130-149`). Runs `contentModerator.moderate(name, PRODUCT_NAME)` then `(description, PRODUCT_DESCRIPTION)`, collecting all violations into `{errors:[...]}`. Request DTO is `name`+`description` only, no `language`. **matches spec.**

### Q3 — Request DTO field lists (verbatim from disk)

**`NewProductRequestDTO`** (`dto/NewProductRequestDTO.java`):
| Field | Type | Jakarta annotations |
| --- | --- | --- |
| `name` | `String` | `@NotBlank(message="NAME_REQUIRED")`, `@Size(max=80, message="NAME_TOO_LONG")`, `@ValidProductName` |
| `description` | `String` | `@NotBlank(message="DESCRIPTION_REQUIRED")`, `@Size(max=2000, message="DESCRIPTION_TOO_LONG")`, `@ValidDescription` |
| `price` | `BigDecimal` | (none) |
| `currency` | `CurrencyDTO` | `@Valid`, `@NotNull(message="CURRENCY_REQUIRED")` |
| `topCategory` | `CategoryDTO` | `@Valid`, `@NotNull(message="CATEGORY_REQUIRED")` |
| `subCategory` | `CategoryDTO` | `@Valid`, `@NotNull(message="CATEGORY_REQUIRED")` |
| `finalCategory` | `CategoryDTO` | `@Valid`, `@NotNull(message="CATEGORY_REQUIRED")` |
| `filters` | `List<SelectedFilterDTO>` | (none) |
| `imageKeys` | `Set<String>` | (none) |

→ **`private boolean free` is NOT present.** It has been fully removed (no field, no getter/setter; grep for `boolean free`/`isFree`/`getFree`/`setFree` on the DTO returns nothing). The spec (§Request DTOs and §Known gaps) and `issues.md` both still describe it as present-but-tolerated. **Drift #1.**

**`UpdateProductRequestDTO`** (`dto/UpdateProductRequestDTO.java`):
- **Standalone** — does NOT extend `NewProductRequestDTO`. **matches spec.**
| Field | Type | Jakarta annotations |
| --- | --- | --- |
| `id` | `Long` | `@NotNull(message="PRODUCT_ID_REQUIRED")` |
| `name` | `String` | `@NotBlank(message="NAME_REQUIRED")`, `@Size(max=80, message="NAME_TOO_LONG")`, `@ValidProductName` |
| `description` | `String` | `@NotBlank(message="DESCRIPTION_REQUIRED")`, `@Size(max=2000, message="DESCRIPTION_TOO_LONG")`, `@ValidDescription` |
| `price` | `BigDecimal` | (none) |
| `currency` | `CurrencyDTO` | `@Valid` |
| `filters` | `List<SelectedFilterDTO>` | (none) |
| `imageKeys` | `Set<String>` | (none) |

→ `oldName`, `oldDescription`, `productState`, `moderationState`, `topCategory`/`subCategory`/`finalCategory`, `regionAndCity`, `free` — **all absent.** **matches spec.**

**`PreValidateProductRequestDTO`** (`dto/PreValidateProductRequestDTO.java`):
| Field | Type | Jakarta annotations |
| --- | --- | --- |
| `name` | `String` | `@NotBlank(message="NAME_REQUIRED")`, `@Size(max=80, message="NAME_TOO_LONG")` |
| `description` | `String` | `@NotBlank(message="DESCRIPTION_REQUIRED")`, `@Size(max=2000, message="DESCRIPTION_TOO_LONG")` |

→ No `language` field. **matches spec.**

### Q4 — GET shape mobile reloads from on update → exists; does NOT carry `oldName`/`oldDescription`

`GET /api/secure/products?productId=...` is `DashboardProductController.getProductDetails` (`:53-67`): owner-checked, returns `ResponseEntity<UpdateProductRequestDTO>` via `productFacade.toUpdateData(product)`. The response DTO type is **`UpdateProductRequestDTO`** — the same standalone DTO audited in Q3, which has **no `oldName`/`oldDescription`** (and no category/region/state fields). The backend does not send them. (Note the actual path is `/api/secure/products`, not the spec's `/secure/products` — drift #3.) **matches spec** (backend confirmed not to send the fields the mobile audit found mobile typing for).

### Q5 — Error wire shape and status routing

- `ProductErrorResponse` is `record ProductErrorResponse(List<FieldError> errors)` with `record FieldError(String field, String code, String translationKey)` → serializes `{"errors":[{"field","code","translationKey"}]}`. **matches spec.**
- `RateLimitFilter` writes a hardcoded `RATE_LIMITED_BODY` = `{"errors":[{"field":null,"code":"RATE_LIMITED","translationKey":"<key>"}]}` on 429, with `Retry-After` set (`:30-35`, `:72-77`). Same shape. **matches spec.**
- `GlobalExceptionHandler.handleProductValidation` → `resolveStatus` picks the highest-severity status across all carried codes via `severity()`: `FORBIDDEN=3 → UNPROCESSABLE_ENTITY=2 → BAD_REQUEST=1` i.e. **403 → 422 → 400** (`:216-230`). A `ProductValidationException` with multiple field errors resolves to the highest. **matches spec.**

## Files touched

- None (read-only audit).

## Tests

- Ran: none (read-only; no code change). Per brief, no runtime execution.
- Result: N/A
- New tests added: none

## Cleanup performed

- None needed (read-only).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no direct write by me. Drift #1 (the now-removed `free` field) means the `issues.md` "Known gaps" item *"Dead `free` field on backend NewProductRequestDTO"* is stale (the cleanup is done) — drafted as an escalation candidate in "For Mastermind" for Docs/QA to close if Mastermind agrees. I did not edit issues.md.

## Obsoleted by this session

- Nothing in this repo (read-only). Three spec/issues statements are now stale relative to code — see "For Mastermind" drifts; none are mine to edit.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched.
- Part 4a (simplicity): see structured evidence in "For Mastermind" (nothing added/removed — audit only).
- Part 4b (adjacent observations): flagged in "For Mastermind" (drifts #1–#3).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 7 (error contract) — confirmed (wire shape + status table match code); Part 11 (trust boundaries) — confirmed (no `oldName`/`oldDescription` on update or GET; server reads persisted name/description via `resolvePersistedName`/`resolvePersistedDescription`).

## Known gaps / TODOs

- I did not re-derive the moderation chain order or the configurable-threshold reads (out of scope for this verification brief — Q1–Q5 only).
- I did not read sibling enums `SystemErrorCode`/`UserErrorCode`/`ReportErrorCode` in full; I confirmed the 6 non-`ProductErrorCode` codes' home enums from their usages/imports in the controller and `GlobalExceptionHandler` (drift #2).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code added.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Drift #1 — `private boolean free` is fully removed from `NewProductRequestDTO`, but spec + issues.md say it's still present.** (severity: low) File: `src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java`. The spec §"Request DTOs → NewProductRequestDTO" says *"A stale `private boolean free` field is still present on the backend NewProductRequestDTO Java class ... Cleanup tracked in issues.md"*, and §"Known gaps" + `issues.md` carry it as an open cleanup chore. The field, its getter, and its setter no longer exist. **This is good news (cleanup done) and does not affect mobile** (mobile must still not send `free`; Jackson would silently ignore an unknown property on inbound JSON). Escalation candidates for Docs/QA (Mastermind to confirm): (a) amend `features/product-validation.md` §Request DTOs and §Known gaps to state `free` is removed; (b) close the `issues.md` "Dead `free` field" known-gap entry as fixed. I did not edit either file.
  - I did not fix this because it is a docs/spec drift outside this read-only repo's scope and the four config files + the spec are not engineer-writable.

- **Drift #2 — error codes are split across three enums; spec attributes all 42 to `ProductErrorCode`.** (severity: low) File: `src/main/java/com/memento/tech/oglasino/exception/ProductErrorCode.java`. Spec §Error codes says *"`ProductErrorCode` enum ... carries 42 constants total."* The actual `ProductErrorCode` enum carries **36** constants. The other 6 live in sibling enums: `RATE_LIMITED`, `NOT_AUTHENTICATED`, `ACCESS_DENIED`, `INTERNAL_ERROR`, `NOT_OWNER` in `SystemErrorCode`; `USER_SETUP_INCOMPLETE` in `UserErrorCode` (confirmed via the controller imports and `GlobalExceptionHandler.buildTranslationKeyRegistry`, which merges `ProductErrorCode + SystemErrorCode + UserErrorCode + ReportErrorCode`). **The wire contract is unaffected** — all 42 `code` strings exist, are unique across enums (the registry throws on collision), and are seeded; the mobile i18n layer keys off `translationKey`/`code` and never sees the Java enum boundary. Escalation candidate: tighten the spec wording from "the ProductErrorCode enum carries 42" to "the product-validation error contract spans 42 codes across `ProductErrorCode` (36) + `SystemErrorCode` + `UserErrorCode`." Cosmetic precision; not a blocker for the mobile chat.
  - I did not fix this because it is spec wording, not code, and is outside scope.

- **Drift #3 — path prefix: backend serves `/api/secure/products...`; spec writes the GET/flow paths as `/secure/products...` in places.** (severity: low–medium) Files: `DashboardProductController.java:43`, `RateLimitFilter.java:88-117`. There is **no backend context-path**, so the `/api` is mandatory on the wire for all four endpoints, including the update-reload `GET /api/secure/products?productId=...` (spec §Update flow writes `GET /secure/products?productId=...`). The three POST endpoints are written correctly as `/api/secure/products/...` in the spec's §Endpoints and §Platform adoption, so the contract mobile codes against is right there; only the prose GET path and a couple of update-flow references drop `/api`. **Mobile must call `/api/secure/products/...` for all four.** Whether the Cloudflare router rewrites `/api` is an `oglasino-router` concern I cannot and did not inspect. Escalation candidate: normalize the spec's bare `/secure/products` references to `/api/secure/products`.
  - I did not fix this because it is spec wording in a sibling repo's docs, outside scope.

- **Net verdict for the mobile chat:** the frozen contract in `features/product-validation.md` §Endpoints / §Wire shape / §HTTP status codes / §Request DTOs / §Platform adoption Part A **matches the live backend.** The three drifts are documentation precision (a completed cleanup not reflected, an enum-attribution wording nuance, and a `/api` prefix omission in prose) — none change the bytes on the wire. Mobile can build against the spec's §Endpoints + §Request DTOs as written, using the `/api/secure/products/...` paths.

- **Config-file dependency closure:** no engineer-written config-file edits this session. The drift items above are drafted here for Mastermind → Docs/QA; none are "drafted but pending application by me" because I am not the writer. No unstated config dependency remains.
