# Audit — Admin App Version Control

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-06-06
**Type:** Read-only Phase-2 audit. No code changes.

Reconnaissance for a new WEB admin-panel view that sets the mobile app-update floor
(`minSupportedVersion`) per platform, reachable via the `BACKEND_API` Firebase-Bearer
client pattern. The five questions from the brief, answered against the actual code.

---

## 1. The store

### Entity

`src/main/java/com/memento/tech/oglasino/admin/internal/entity/AppVersionConfig.java`

```java
@Entity
public class AppVersionConfig extends BaseEntity {
  private String platform;
  private String latestVersion;
  private String minSupportedVersion;
  // getters/setters only
}
```

Three business columns: `platform`, `latestVersion` (the ceiling), `minSupportedVersion`
(the floor). Nothing else business-wise.

### Timestamps — YES, both present

`AppVersionConfig extends BaseEntity`
(`src/main/java/com/memento/tech/oglasino/entity/BaseEntity.java`), which supplies:

- `id` — `@GeneratedValue` SEQUENCE `global_id_seq`
- `createdAt` — `@CreationTimestamp`, `@Column(updatable = false)`
- `updatedAt` — `@UpdateTimestamp`

So **every row carries `created_at` and `updated_at`**. `updatedAt` is bumped automatically
by Hibernate on every `save()` of the row. There is no separate `lastWritten`/`writtenBy`
column — `updatedAt` is the only "when last changed" signal, and there is **no record of
WHO changed it**.

### DB table

`src/main/resources/db/migration/V1__init_schema.sql` (lines 50-57):

```sql
CREATE TABLE public.app_version_config (
    created_at timestamp(6) without time zone,
    id bigint NOT NULL,
    updated_at timestamp(6) without time zone,
    latest_version character varying(255),
    min_supported_version character varying(255),
    platform character varying(255)
);
```

Constraints (lines 627-636):

- PK on `id`
- **`UNIQUE (platform)`** (`app_version_config_platform_key`)

The `UNIQUE (platform)` constraint enforces exactly **one live row per platform**.

### History / audit table — NONE

There is **no history/audit table**. The store is the single live row per platform and
nothing else. There is no list of "known versions" to pick from — only the two live
values (`latestVersion`, `minSupportedVersion`) per platform row. The only platforms
seeded are `android` (id=1) and `ios` (id=2), both at `0.0.0`/`0.0.0`
(`src/main/resources/data/configuration/data-app-version-config.sql`, `ON CONFLICT (id) DO NOTHING`).

**Implication for the admin view:** a "pick from known versions" dropdown is not backable
by the DB. The view must accept a free-typed semver (validated client+server-side), or it
can offer the current ceiling (`latestVersion`) as a reference value to type the floor
against. Past floor values are not retained anywhere.

---

## 2. The read

`src/main/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionController.java`

**Exact mapping:** `@RequestMapping("/api/public/app/version")` + `@GetMapping("/{platform}")`,
query param `currentVersion`. (Brief shorthand `/public/app/version/{platform}` = the
`/api/public/...` path here.)

### Force / optional computation — quoted

```java
String floor = config.getMinSupportedVersion();
String ceiling = config.getLatestVersion();

boolean force = isLower(currentVersion, floor);
boolean optional = isLower(currentVersion, ceiling) && !force;
return ResponseEntity.ok(new AppVersionResponseDTO(ceiling, floor, force, optional));
...
private boolean isLower(String current, String target) {
  return Version.parse(current).isLowerThan(Version.parse(target));
}
```

- `forceUpdate` = running `currentVersion` **<** `minSupportedVersion` (floor).
- `optionalUpdate` = running `currentVersion` **<** `latestVersion` (ceiling) **AND NOT force**.
  (Mutually exclusive: a client below the floor gets force-only, never both flags.)

### Fails open — confirmed, quoted

**Missing row / unknown platform:**

```java
AppVersionConfig config = appVersionConfigRepository.findByPlatform(platform).orElse(null);
if (config == null) {
  log.warn("No AppVersionConfig row for platform '{}'; failing open (no update).", platform);
  return ResponseEntity.ok(NO_UPDATE);
}
```

**Unparseable semver (current, floor, or ceiling):**

```java
} catch (RuntimeException ex) {
  log.warn("AppVersion gate semver parse failed ... failing open: {}", ..., ex.getMessage());
  return ResponseEntity.ok(NO_UPDATE);
}
```

`NO_UPDATE` is `new AppVersionResponseDTO("", "", false, false)` — empty version strings,
both booleans false. It returns **HTTP 200 with a no-update DTO, never throws**. The
read path cannot brick the app regardless of garbage stored or sent.

---

## 3. The writes

Both internal writes live in
`src/main/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionAdminController.java`,
class-mapped `@RequestMapping("/internal/app/version")`.

### POST /internal/app/version/floor

- **Path:** `/internal/app/version/floor`
- **Request DTO:** `AppVersionFloorUpdateRequest(String platform, String minSupportedVersion)`
- **Writes:** `config.setMinSupportedVersion(request.minSupportedVersion())` only — comment
  notes "cannot un-set the ceiling."
- **Guard:** `InternalTokenFilter` (X-INTERNAL-TOKEN). The class is under `/internal/`, which
  `SecurityConfig` `permitAll()`s at the Spring-Security layer, then the servlet filter
  `InternalTokenFilter` enforces the shared-secret token (see §4 detail below).
- **UPDATE-ONLY:** yes. `findByPlatform(...).orElse(null)`; if `config == null` →
  `ResponseEntity.notFound().build()` (**404 on missing platform row**). It does **not** upsert
  — a new platform must be seeded into `app_version_config` first.

### POST /internal/app/version/ceiling

- **Path:** `/internal/app/version/ceiling`
- **Request DTO:** `AppVersionCeilingUpdateRequest(String platform, String latestVersion)`
- **Writes:** `config.setLatestVersion(request.latestVersion())` only.
- **Guard:** same `InternalTokenFilter` / X-INTERNAL-TOKEN.
- **UPDATE-ONLY:** yes — identical 404-on-missing pattern, no upsert.
- **EAS build hook:** the in-code comment marks ceiling as the **"EAS post-build hook target"**
  (and floor as the **"Admin panel target"**). Confirmed: ceiling is the one the EAS build hook calls.

### Any /api/secure/admin/** version endpoint today?

**NONE.** Full enumeration of `src/main/java/.../admin/controller/`:
AdminController, AdminProductController, AdminProductSearchController, AdminReportController,
AdminReviewController, AdminTranslationsController, CacheAdminController, ConfigController,
EsIndexerController, EsStateController, FirebaseChatController, MaintenanceAdminController,
StatsController, SuggestionController, UsersController (+ `AdminCacheDescriptor` enum, not a
controller). No version controller. Grep for `app/version` / `AppVersion` under
`admin/controller/` returns nothing. The only version writers are the two `/internal/` endpoints.

---

## 4. The admin-endpoint gap

### Standard admin-gate mechanism

It is **two layers**, both required:

1. **Path-level (SecurityConfig)** — `src/main/java/com/memento/tech/oglasino/security/config/SecurityConfig.java`:

   ```java
   .requestMatchers("/api/secure/**").authenticated()
   ```

   This only requires a *valid Firebase token* (any authenticated user). `FirebaseAuthFilter`
   populates `OglasinoAuthentication` in the `SecurityContextHolder`. There is **no
   `/api/secure/admin/**` → `hasRole('ADMIN')` matcher in SecurityConfig** — the admin
   restriction is NOT done by path matcher.

2. **Method-level (`@PreAuthorize`)** — `@EnableMethodSecurity` is on `SecurityConfig`, and
   each admin controller carries a **class-level** `@PreAuthorize("hasRole('ADMIN')")`. This
   is the gate that actually restricts to admins. Concrete example
   (`CacheAdminController.java`, the cache admin the brief names):

   ```java
   @RestController
   @RequestMapping("/api/secure/admin/cache")
   @PreAuthorize("hasRole('ADMIN')")
   public class CacheAdminController { ... }
   ```

   `MaintenanceAdminController` (`/api/secure/admin/maintenance`) uses the identical
   class-level `@PreAuthorize("hasRole('ADMIN')")`. This is the established pattern a new
   version-admin controller must follow: map under `/api/secure/admin/...` and put
   `@PreAuthorize("hasRole('ADMIN')")` on the class.

### Confirmation of the gap

There is **no admin-namespaced version endpoint today**. To let the web admin view set the
floor over the `BACKEND_API` (Firebase Bearer) pattern, a **new** controller must be added,
e.g. `@RequestMapping("/api/secure/admin/app/version")` + `@PreAuthorize("hasRole('ADMIN')")`,
with a POST that mirrors the floor write. The existing `/internal/app/version/floor` is
M2M-only (token guard) and is not reachable by the Firebase-Bearer browser client.

---

## 5. Semver parsing

- **Library:** `com.github.zafarkhaja:java-semver` **0.10.2** (`pom.xml` line ~178,
  artifactId `java-semver`). This is the semver4j-family library the docs reference.
- **Usage:** `import com.github.zafarkhaja.semver.Version;` →
  `Version.parse(current).isLowerThan(Version.parse(target))` (read path only). The two
  internal writers do **not** parse/validate — they store the string verbatim (see §trust note).
- **Format the admin view must validate against:** standard SemVer 2.0.0 — three-part
  `MAJOR.MINOR.PATCH` (e.g. `1.4.0`). `Version.parse` is strict: a two-part `"1.4"` or empty
  string throws `ParseException`. The seeded baseline is `0.0.0`. The admin view should
  validate input as a parseable three-part semver before submitting.

---

## Trust boundary (Part 11)

- **Admin identity:** must be derived server-side. The correct pattern is exactly what the
  existing admin controllers do — `@PreAuthorize("hasRole('ADMIN')")` reading the role from
  `OglasinoAuthentication` in `SecurityContextHolder`, which `FirebaseAuthFilter` populates
  from the *verified* Firebase ID token (role loaded from `redisUserAuth` / DB, not from a
  client-supplied claim/flag). A new admin endpoint must **not** accept a role/flag from the
  request body or trust an unverified token claim. The floor `platform` + value come from the
  body; the *authorization to write* comes from the security context.

- **Value validation today — NONE on the write path.** This is the key finding for the new
  endpoint. Both `/internal/app/version/{floor,ceiling}` writers store the supplied string
  **verbatim** with no semver-format check and no range/monotonicity check:

  ```java
  config.setMinSupportedVersion(request.minSupportedVersion());
  appVersionConfigRepository.save(config);
  ```

  Validation is implicit and deferred to the *read* path, which fails open on a garbage
  stored value (returns NO_UPDATE). So a malformed floor doesn't crash anything, but it
  also **silently disables the gate** for that platform until corrected — the floor stops
  forcing anyone. The DTO records have no Jakarta constraints (`@NotBlank`, `@Pattern`)
  either.

  The new `/api/secure/admin/**` floor endpoint should **add** server-side validation the
  internal path lacks: (a) `minSupportedVersion` parses as a three-part semver
  (`Version.parse`), rejecting otherwise with the standard error contract (Part 7 — code,
  not message); optionally (b) the floor is not above the current ceiling (`latestVersion`),
  since a floor > ceiling would force-update users to a build that doesn't exist yet. (b) is
  a product decision for Mastermind, not a confirmed requirement.

---

## For Mastermind

- **Defect / hardening flag (medium):** the floor is a state-control value (forces every
  user below it to update), and the existing `/internal/app/version/floor` write does **zero
  server-side validation** of the value — no semver check, no range check, no Jakarta
  constraints on the DTO. A malformed floor silently disables the gate (read path fails
  open). The new admin endpoint should not copy this gap; it should validate the semver
  format server-side and consider rejecting floor > ceiling. File:
  `src/main/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionAdminController.java`.
  Not fixed — out of audit scope.

- **No audit trail of WHO set the floor (low/medium):** the row has `updatedAt` (when) but
  no actor column and no history table. If "which admin raised the floor" ever matters, that
  is net-new schema, not present today. Flagging so the spec can decide whether to add it
  while the admin endpoint is being built (cheap now, pre-prod, per Part 12 V1 fold).

- **No dedicated service layer:** version logic lives directly in the two controllers using
  `AppVersionConfigRepository`. A new admin floor write that needs shared validation +
  monotonicity logic alongside the internal write may warrant extracting a small service so
  the rule isn't duplicated across `/internal` and `/api/secure/admin`. Architecture call
  for the spec.
