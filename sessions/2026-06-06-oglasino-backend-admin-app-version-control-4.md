# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-06
**Task:** Cold-verification (read-only) of the as-built admin app-version surface against `features/admin-app-version-control.md` — verify the four items in the brief; fix nothing.

## Verdict at a glance

1. Admin gate is real — **PASS**
2. GET / POST response shape — **PASS** (one implicit-contract note, low/med — see For Mastermind)
3. The two validations bite — **PASS**
4. Error wire shape — **PASS**

No code changes. No tests added. Nothing fixed.

---

## Item 1 — THE ADMIN GATE IS REAL — PASS

The gate is **genuinely enforced at runtime**. `AdminAppVersionControllerPreAuthorizeTest` is a faithful mirror of the cited precedent `MaintenanceAdminControllerPreAuthorizeTest`, not weaker, and it **would fail if the `@PreAuthorize` annotation were deleted.**

Both tests stand up a real `@EnableMethodSecurity` context that proxies the controller bean, so `@PreAuthorize("hasRole('ADMIN')")` is evaluated at call time (standalone MockMvc never engages method security — confirmed by the class Javadoc and by the second test class, which only reflection-checks the annotation).

`AdminAppVersionControllerPreAuthorizeTest` (lines 40–57):

```java
@Test
@WithMockUser(roles = "ADMIN")
void adminPrincipalPassesGate() {
  when(appVersionService.getAll()).thenReturn(List.of());
  var response = controller.getAll();
  assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
  verify(appVersionService).getAll();
}

@Test
@WithMockUser(roles = "USER")
void nonAdminPrincipalIsDenied() {
  assertThatThrownBy(() -> controller.getAll()).isInstanceOf(AccessDeniedException.class);
  verifyNoInteractions(appVersionService);
}
```

Context config (lines 59–72) — `@EnableMethodSecurity`, real mock service bean, controller bean via constructor:

```java
@EnableMethodSecurity
@Configuration
static class MethodSecurityTestConfig {
  @Bean AppVersionService appVersionService() { return mock(AppVersionService.class); }
  @Bean AdminAppVersionController adminAppVersionController(AppVersionService appVersionService) {
    return new AdminAppVersionController(appVersionService);
  }
}
```

Precedent `MaintenanceAdminControllerPreAuthorizeTest` (lines 49–67) — identical structure:

```java
@Test
@WithMockUser(roles = "ADMIN")
void adminPrincipalPassesGate() {
  when(cloudflareKvService.isMaintenanceActive()).thenReturn(true);
  var response = controller.isActive();
  assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
  ...
}
@Test
@WithMockUser(roles = "USER")
void nonAdminPrincipalIsDenied() {
  assertThatThrownBy(() -> controller.isActive()).isInstanceOf(AccessDeniedException.class);
  verifyNoInteractions(cloudflareKvService);
}
```

**Adversarial check — would it pass with the annotation removed?** No. `@EnableMethodSecurity` only wraps a bean in a security proxy when the bean carries a method-security annotation. Delete the class-level `@PreAuthorize` and the controller bean is no longer proxied → `controller.getAll()` runs the body and returns `200 OK` → `assertThatThrownBy(...).isInstanceOf(AccessDeniedException.class)` finds no exception and **fails**. So `nonAdminPrincipalIsDenied` is a real guard against silent gate removal.

Only difference vs precedent: this test injects the controller via its constructor; the precedent uses `ReflectionTestUtils` field injection because `MaintenanceAdminController` is field-injected. Equivalent — the AdminAppVersion form is cleaner, not weaker.

The annotation it enforces (`AdminAppVersionController.java:27–30`):

```java
@RestController
@RequestMapping("/api/secure/admin/app/version")
@PreAuthorize("hasRole('ADMIN')")
public class AdminAppVersionController {
```

`AdminAppVersionControllerTest` additionally reflection-asserts the annotation value is exactly `hasRole('ADMIN')` and the mapping is exactly `/api/secure/admin/app/version` (lines 100–114) — belt-and-suspenders, catches a wrong/renamed annotation that the proxy test would not distinguish.

---

## Item 2 — GET RESPONSE SHAPE — PASS (capture this verbatim)

### GET `/api/secure/admin/app/version`

**Bare JSON array** — NOT wrapped in an object. Controller returns `ResponseEntity<List<AppVersionAdminDTO>>` (`AdminAppVersionController.java:38–41`):

```java
@GetMapping
public ResponseEntity<List<AppVersionAdminDTO>> getAll() {
  return ResponseEntity.ok(appVersionService.getAll());
}
```

Each element is `AppVersionAdminDTO` (`AppVersionAdminDTO.java:10–11`):

```java
public record AppVersionAdminDTO(
    String platform, String latestVersion, String minSupportedVersion, LocalDateTime updatedAt) {}
```

**Exact element fields + types on the wire:**

| field | Java type | wire type |
| --- | --- | --- |
| `platform` | `String` | string (e.g. `"android"`, `"ios"`) |
| `latestVersion` | `String` | string (e.g. `"1.4.0"`) |
| `minSupportedVersion` | `String` | string (e.g. `"1.0.0"`) |
| `updatedAt` | `LocalDateTime` | **ISO-8601 local date-time string, no zone/offset** (e.g. `"2026-06-06T12:00:00"`) |

**`updatedAt` format — verified empirically, not from memory.** There is no `@JsonFormat` on the field, no `spring.jackson.*` config in any `application-*.yaml`, and no custom/`@Primary` web `ObjectMapper` (the only `ObjectMapper` in `RedisConfig.cacheManager` is a method-local variable scoped to the Redis cache serializers — it does not touch MVC). The web layer therefore uses Spring Boot 4.0.6's auto-configured Jackson 3 (`tools.jackson` 3.1.2) mapper, which leaves `WRITE_DATES_AS_TIMESTAMPS` disabled. I serialized the actual `AppVersionAdminDTO` record on the project's runtime classpath via jshell/javac:

```
{"platform":"android","latestVersion":"1.4.0","minSupportedVersion":"1.0.0","updatedAt":"2026-06-06T12:00:00"}
```

So `updatedAt` is an **ISO-8601 string with NO timezone/offset and NO trailing `Z`** — `"YYYY-MM-DDTHH:MM:SS"`. Note: a real Hibernate `@UpdateTimestamp` value will usually carry sub-second precision, so expect fractional seconds in production, e.g. `"2026-06-06T12:34:56.123456"`. **Web should parse this as a local (zoneless) ISO-8601 timestamp and not assume fixed second/fraction width.** It is NOT epoch millis and NOT a `[2026,6,6,12,0]` array.

### POST `/api/secure/admin/app/version/floor` success response

**Same shape as one GET element** — a single `AppVersionAdminDTO` object (not an array). `AdminAppVersionController.java:43–50`:

```java
@PostMapping("/floor")
public ResponseEntity<AppVersionAdminDTO> updateFloor(@RequestBody AppVersionFloorUpdateRequest request) {
  return appVersionService
      .updateFloor(request.platform(), request.minSupportedVersion())
      .map(ResponseEntity::ok)
      .orElseGet(() -> ResponseEntity.notFound().build());
}
```

So on success: `200` with one object `{ "platform", "latestVersion", "minSupportedVersion", "updatedAt" }` (the post-update row, reflecting the new `minSupportedVersion`). On a platform with no row: **`404` with empty body** (UPDATE-only, matches spec).

---

## Item 3 — THE TWO VALIDATIONS ACTUALLY BITE — PASS

All in `DefaultAppVersionService.updateFloor` (`DefaultAppVersionService.java:38–78`). Order of operations: lookup → `parseFloor` → `rejectFloorAboveCeiling` → `setMinSupportedVersion` → `save`. **Both validations run before any mutation/save** — confirmed below.

```java
public Optional<AppVersionAdminDTO> updateFloor(String platform, String minSupportedVersion) {
  AppVersionConfig config = appVersionConfigRepository.findByPlatform(platform).orElse(null);
  if (config == null) { return Optional.empty(); }
  Version floor = parseFloor(minSupportedVersion);          // throws INVALID_SEMVER before any write
  rejectFloorAboveCeiling(floor, config.getLatestVersion()); // throws ABOVE_CEILING before any write
  config.setMinSupportedVersion(minSupportedVersion);        // mutation only after both pass
  appVersionConfigRepository.save(config);
  ...
}
```

- **Invalid semver** (`"1.4"`, `""`, `"abc"`) → `Version.parse` throws → caught → `throw new AppVersionValidationException(FLOOR_FIELD, APP_VERSION_FLOOR_INVALID_SEMVER)`; thrown before `config` is mutated, nothing saved (lines 55–62). Test `invalidSemver_rejectedWithCodedError_andNotSaved` asserts the code and `verify(...never()).save(any())`.
- **floor > ceiling** uses true semver comparison, not string compare: `parseFloor`/`Version.parse(latestVersion)` then `floor.isHigherThan(ceiling)` (java-semver `Version.isHigherThan`) → `APP_VERSION_FLOOR_ABOVE_CEILING` (lines 64–78). Test `floorAboveCeiling_rejectedWithCodedError_andNotSaved` (`1.5.0` vs ceiling `1.4.0`) asserts the code and no save.
- **floor == ceiling allowed** ("Require latest version" path): `isHigherThan` is strict, so equal is not "higher" → no throw → saves. Test `floorEqualToCeiling_isAllowed` (`2.0.0` floor == `2.0.0` ceiling) asserts present result and `verify(...).save(row)`. **This case passes — the button is safe.**
- **No partial write then throw:** the single `setMinSupportedVersion` + `save` sit after both guards; no field is touched before validation.

Edge note (not a defect, defensible): if the *stored* `latestVersion` is itself unparseable, `rejectFloorAboveCeiling` returns early and skips the ceiling check (lines 64–73, commented rationale: the read gate fails open on a garbage ceiling so no lockout is possible). The floor's own semver check still applies. Matches the spec's fail-open philosophy.

---

## Item 4 — THE ERROR WIRE SHAPE — PASS

422 body for both codes is exactly `{"errors":[{"field","code","translationKey"}]}` with `field="minSupportedVersion"`, `code` = SCREAMING_SNAKE enum name, `translationKey` = dotted key.

Record (`ProductErrorResponse.java:15–17`):

```java
public record ProductErrorResponse(List<FieldError> errors) {
  public record FieldError(String field, String code, String translationKey) {}
```

Handler that builds it for this surface (`GlobalExceptionHandler.java:164–179`):

```java
@ExceptionHandler(AppVersionValidationException.class)
public ResponseEntity<ProductErrorResponse> handleAppVersionValidation(AppVersionValidationException ex) {
  HttpStatus status =
      ex.getErrors().stream()
          .map(e -> e.code().getHttpStatus())
          .max(java.util.Comparator.comparingInt(GlobalExceptionHandler::severity))
          .orElse(HttpStatus.UNPROCESSABLE_ENTITY);
  List<ProductErrorResponse.FieldError> errors =
      ex.getErrors().stream()
          .map(e -> new ProductErrorResponse.FieldError(
                      e.field(), e.code().name(), e.code().getTranslationKey()))
          .toList();
  return ResponseEntity.status(status).body(new ProductErrorResponse(errors));
}
```

- `field` = `e.field()` = `"minSupportedVersion"` (the `FLOOR_FIELD` constant set in the service on both codes).
- `code` = `e.code().name()` = the enum constant name (SCREAMING_SNAKE).
- `translationKey` = `e.code().getTranslationKey()` = the dotted key.
- **HTTP status = 422.** Both codes are `HttpStatus.UNPROCESSABLE_ENTITY` (`AppVersionErrorCode.java:12–16`); `severity` maps `UNPROCESSABLE_ENTITY → 2` (`GlobalExceptionHandler` `severity` switch), so the `max` resolves to 422.

**The two exact wire bodies the web will receive (422):**

```json
{ "errors": [ { "field": "minSupportedVersion", "code": "APP_VERSION_FLOOR_INVALID_SEMVER", "translationKey": "app_version.floor.invalid_semver" } ] }
```

```json
{ "errors": [ { "field": "minSupportedVersion", "code": "APP_VERSION_FLOOR_ABOVE_CEILING", "translationKey": "app_version.floor.above_ceiling" } ] }
```

(`AppVersionErrorCode.java:13–16` defines the two `translationKey`/status pairs above.)

---

## Files touched

- none (read-only verification)

## Tests

- Ran: none added/modified. Read and analyzed `AdminAppVersionControllerPreAuthorizeTest`, `MaintenanceAdminControllerPreAuthorizeTest`, `AdminAppVersionControllerTest`, `DefaultAppVersionServiceTest`.
- One throwaway out-of-repo serialization check (`/tmp/SerCheck.java`, deleted) on the project's `dependency:build-classpath` to confirm `updatedAt`'s wire format under Jackson 3 / Spring Boot 4.0.6. Nothing written into the repo tree.

## Cleanup performed

- none needed (read-only; the only artifact was a `/tmp` file, deleted).

## Config-file impact

- conventions.md: no change
- decisions.md: no change (the spec's owed decisions.md entry is a feature-close item, not produced by this verification)
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no repo artifacts created.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one low/medium observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session (not in verification scope; the two `app_version.floor.*` keys' SQL seeding was not part of the four items and was not audited here).
- Other parts touched: Part 7 (error contract) — confirmed (item 4); Part 11 (trust boundary) — confirmed, admin role read from `@PreAuthorize`/security context, not the request (item 1).

## Known gaps / TODOs

- Translation-seed presence for `app_version.floor.invalid_semver` / `app_version.floor.above_ceiling` was outside the brief's four items and was not verified. Flag for the web/translation pass if not already seeded.
- The 422 wire shape was verified by reading the handler/record/enum; it was not exercised end-to-end via an integration test in this session (none exists asserting the JSON body for these two codes). The standalone-MockMvc controller test does not flow through `GlobalExceptionHandler`. Confidence is high from code, but there is no test pinning the rendered 422 body.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **All four brief items PASS.** The force-update gate (item 1) is genuinely enforced at runtime and the test fails if the annotation is removed — no false-confidence finding.

- **Adjacent observation (low/medium) — `updatedAt` wire format is an implicit, untested contract.** `AppVersionAdminDTO` is the **only** wire-facing DTO in the backend that exposes a `java.time.LocalDateTime` on an HTTP JSON response (grep of `src/main/java/**/dto` for `LocalDateTime` returns only this file). Its serialized form (`"2026-06-06T12:00:00"`, zoneless ISO-8601) rests entirely on Spring Boot's default `WRITE_DATES_AS_TIMESTAMPS=false` — there is no `@JsonFormat`, no `spring.jackson` property, and no test asserting the rendered `updatedAt` string (the standalone-MockMvc test asserts only `platform`/`latestVersion`/`minSupportedVersion`). File: `src/main/java/com/memento/tech/oglasino/admin/dto/AppVersionAdminDTO.java`. Severity: low-to-medium — if anyone later enables date-as-timestamp globally or swaps the mapper, the web "updated X ago" provenance display breaks silently with no failing backend test to catch it. I did not change anything (read-only, and out of scope). Options if Mastermind wants to harden: pin with `@JsonFormat(shape = STRING)` on the field, or add one MockMvc/integration assertion on the `updatedAt` JSON path. Note the brief assumed "an existing endpoint that returns a BaseEntity-derived `updatedAt`" exists to copy from — it does not; this DTO is the first.

- **For the web service layer (verbatim capture, requested):**
  - GET → bare array `[{platform,latestVersion,minSupportedVersion,updatedAt}, ...]`; `updatedAt` = zoneless ISO-8601 string (parse leniently — sub-second precision in prod).
  - POST `/floor` success → single object, same four fields.
  - 422 error → `{"errors":[{"field":"minSupportedVersion","code":"APP_VERSION_FLOOR_INVALID_SEMVER"|"APP_VERSION_FLOOR_ABOVE_CEILING","translationKey":"app_version.floor.invalid_semver"|"app_version.floor.above_ceiling"}]}`.
  - 404 (unknown platform) → empty body.

- Config-file dependency check: none. No `conventions.md`/`decisions.md`/`state.md`/`issues.md` edit is required by this verification.
