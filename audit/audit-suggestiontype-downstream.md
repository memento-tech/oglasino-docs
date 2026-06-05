# Audit — SuggestionType Downstream Safety

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-05
**Mode:** READ-ONLY (no code changes)
**Question:** If `DefaultSuggestionService.saveSuggestion` is fixed to honor the client-supplied
`suggestionType` (instead of hardcoding `CATEGORY_SUGGESTION`), does anything downstream break when
a stored `Suggestion` carries `FEATURE_BUG_SUGGESTION`?

**Verdict up front: SAFE.** Honoring the client type introduces no downstream break. There is no
`switch`/`if` on `SuggestionType` anywhere in the backend; the DB check constraint already allows both
values; the admin read path is type-agnostic; the filter handles both values and null; and inbound
validation already requires a valid non-null enum, so a client can already *send*
`FEATURE_BUG_SUGGESTION` today — the only thing stopping persistence is the hardcode. One frontend
seam to flag (admin UI must render the new type) and one unrelated adjacent bug (`@Max` on a String).

---

## Q1 — `saveSuggestion` verbatim + confirmation

`admin/service/impl/DefaultSuggestionService.java:20-30`:

```java
  @Override
  public void saveSuggestion(SuggestionType suggestionType, String suggestion) {
    var currentUserId = currentUserService.getCurrentUserId().orElse(null);

    var suggestionEntity = new Suggestion();
    suggestionEntity.setSuggestionType(SuggestionType.CATEGORY_SUGGESTION);   // :25 — hardcode
    suggestionEntity.setUserId(currentUserId);
    suggestionEntity.setSuggestion(suggestion);

    suggestionRepository.save(suggestionEntity);
  }
```

**Confirmed:** the method receives `suggestionType` as its first parameter (`:21`) and then never reads
it — line `:25` overwrites with the literal `SuggestionType.CATEGORY_SUGGESTION`. The incoming
parameter is silently discarded. (Brief cited `:21`/`:25`; exact match.)

**Confirmed the request DTO carries a client-supplied type** — `admin/dto/SuggestionRequestDTO.java:13`:

```java
  @NotNull private SuggestionType suggestionType;
```

with public getter/setter (`:23-29`). The controller wires it through —
`admin/controller/SuggestionController.java:22-29`:

```java
  @PostMapping("/api/public/suggestion")
  public ResponseEntity<?> suggestCategory(@RequestBody SuggestionRequestDTO categorySuggestionRequest) {
    suggestionService.saveSuggestion(
        categorySuggestionRequest.getSuggestionType(),       // client value reaches the service…
        categorySuggestionRequest.getSuggestion());
    return ResponseEntity.ok().build();
  }
```

So the client value travels controller → service parameter, then dies at `:25`. The endpoint is
`/api/public/suggestion` (unauthenticated; `getCurrentUserId().orElse(null)` makes `userId` optional).

---

## Q2 — Every reader of `Suggestion.type` / `SuggestionType` across backend src

Full inventory (grep of `SuggestionType` / `suggestionType` / `getSuggestionType` /
`CATEGORY_SUGGESTION` / `FEATURE_BUG_SUGGESTION` over `src/main/java` + `src/test`):

| # | Location | What it does with the value |
|---|----------|------------------------------|
| 1 | `admin/entity/SuggestionType.java:3-6` | Enum definition: `CATEGORY_SUGGESTION`, `FEATURE_BUG_SUGGESTION`. No per-constant behavior. |
| 2 | `admin/entity/Suggestion.java:17-18` | Field, `@Enumerated(EnumType.STRING)` → persisted as the enum *name* string. Getter/setter `:36-42`. Stores whatever it is given. |
| 3 | `admin/dto/SuggestionRequestDTO.java:13` | **Inbound write.** `@NotNull SuggestionType suggestionType`. Deserialized from client JSON. |
| 4 | `admin/dto/SuggestionsFilterRequestDTO.java:14,24-30,40-41` | **Admin filter (read).** Optional filter; `getSuggestionSpec()` adds `cb.equal(root.get("suggestionType"), suggestionType)` **only when non-null** (`:40`). Null = no type predicate (returns all). Value-agnostic — handles both enum values identically. |
| 5 | `admin/dto/SuggestionDTO.java:6` | **Outbound read.** Record field; carries the stored type out to the admin client verbatim. |
| 6 | `admin/controller/SuggestionController.java:26` | Passes `getSuggestionType()` into `saveSuggestion` (currently discarded at the service). |
| 7 | `admin/controller/SuggestionController.java:42` | Admin list path maps `suggestion.getSuggestionType()` into `SuggestionDTO` — straight passthrough, no branching. |
| 8 | `admin/service/SuggestionService.java:9` | Interface signature `saveSuggestion(SuggestionType, String)`. |
| 9 | `admin/service/impl/DefaultSuggestionService.java:21,25` | The parameter (`:21`) + the hardcode (`:25`). |

**Specifically requested checks:**

- **Any `switch`/branch on `SuggestionType` lacking a `FEATURE_BUG_SUGGESTION` case?**
  **None exists.** There is no `switch` and no `if (type == …)` on the value anywhere in `src`. Nothing
  dispatches, routes, notifies, emails, or formats differently by type. There is therefore no
  fall-through / throw / mishandling risk to add a case to.
- **Admin-facing read path (list/filter) — handles both values?**
  **Yes.** `getSuggestions` (`SuggestionController:33-50` → `DefaultSuggestionService:32-36`) runs the
  JPA Specification and maps each row into `SuggestionDTO`. The type is passed through untouched; the
  path does not interpret the value, so both enum values are handled identically.
- **`SuggestionsFilterRequestDTO` — is filtering by type wired, both values handled?**
  **Yes (`:40-41`).** Filtering is wired via the Specification, equality on `suggestionType`, applied
  only when the filter is non-null. A client filtering by either value works; absence of the filter
  returns all types. Both values, and the no-filter case, are handled.

**DB layer (`V1__init_schema.sql:534-542`):** `suggestion_type character varying(255)` (nullable);
CHECK constraint `suggestion_suggestion_type_check` permits **both** `'CATEGORY_SUGGESTION'` and
`'FEATURE_BUG_SUGGESTION'`. Persisting `FEATURE_BUG_SUGGESTION` satisfies the constraint — no DB-level
break.

**Tests:** no test references `SuggestionType` / either constant (grep of `src/test` returns nothing
beyond unrelated `Suggestion`-string matches). So honoring the type breaks no existing assertion.

---

## Q3 — Inbound validation; can a client already SEND `FEATURE_BUG_SUGGESTION`?

**Yes.** `SuggestionRequestDTO.java:13` declares `@NotNull private SuggestionType suggestionType;`.
Jackson deserializes the JSON string `"FEATURE_BUG_SUGGESTION"` to the enum constant successfully (it is
a valid `SuggestionType`), and `@NotNull` is satisfied. There is no allow-list, no
`CATEGORY_SUGGESTION`-only guard, and no rejection of `FEATURE_BUG_SUGGESTION` anywhere upstream of the
service.

**Conclusion:** deserialization + validation of `FEATURE_BUG_SUGGESTION` already succeed today. The
value reaches `saveSuggestion` intact and is discarded *only* at the `:25` hardcode. The hardcode — not
any earlier rejection — is the sole reason `FEATURE_BUG_SUGGESTION` is never persisted. Removing it (and
using the parameter) is sufficient to make the type take effect.

Note: an unknown/garbage type string fails Jackson enum deserialization → 400 before the service, and a
missing type fails `@NotNull` → 400. Both are correct and unaffected by the fix.

---

## Q4 — Verdict

**Honoring the client-supplied type is safe. No downstream break, and no `FEATURE_BUG_SUGGESTION` case
needs to be added anywhere — because nothing in the backend branches on the value.**

Confirmed safe across every layer:
- **Persistence** — `@Enumerated(EnumType.STRING)` + DB CHECK constraint already accept both values.
- **Admin read (list)** — type passed through to `SuggestionDTO`, never interpreted.
- **Admin filter** — equality predicate handles both values and the no-filter (null) case.
- **Validation** — already requires a valid non-null enum; client can already send either value.
- **Branching** — there is none; no `switch`/`if` to update.

**Nothing in this repo requires a `FEATURE_BUG_SUGGESTION` case to be added.**

### Cross-repo seam to flag (not a backend break)
- Once the type is honored, `SuggestionDTO.suggestionType` will carry `FEATURE_BUG_SUGGESTION` to the
  admin UI **for the first time**. The admin frontend (`oglasino-web`) must render that enum value in
  the suggestions table/filter (translation keys `suggestion.table.type` / `suggestion.filter.type.*`
  exist; whether the web admin maps the `FEATURE_BUG_SUGGESTION` label is an `oglasino-web` concern).
  Per the no-cross-repo rule, this is flagged for routing to the web agent, not audited here.

### Adjacent observation (out of scope — Part 4b)
- **`@Max(100)` on a `String` field is a no-op.** `admin/dto/SuggestionRequestDTO.java:10-11`:
  `@Max(100) private String suggestion;`. Jakarta `@Max` validates numeric types only; on a `String` it
  is silently ignored, so there is **no effective length cap** from validation (the DB column caps at
  `varchar(255)` — a 256+ char suggestion would hit a DB error / 500, not a clean 400). Almost certainly
  intended to be `@Size(max = 100)`. **Severity: low/medium.** Unrelated to the type question; not
  fixed here (read-only audit, out of scope).
- **Naming is category-centric** — controller method `suggestCategory`, param `categorySuggestionRequest`
  (`SuggestionController.java:23-24`). Cosmetic only; once `FEATURE_BUG_SUGGESTION` is a real path the
  names read slightly wrong, but this is not a defect. **Severity: low (cosmetic).** Noted, not fixed.
