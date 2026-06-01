# Audit — Expo boot redesign (backend)

Read-only audit of the four boot-time concerns the Expo app gates on. All paths, fields, types are quoted from the actual code on `dev` (HEAD as of 2026-05-28). No code changes performed.

Convention reminder: every public endpoint sits behind a stack of servlet filters. Two filters are load-bearing for the boot question:

- `BaseSiteFilter` (`filter/BaseSiteFilter.java:24`) — reads `X-Base-Site` header (falls back to `?baseSite=` query param). If present, looks up the site via `BaseSiteCacheService` and stows it on a thread-local `BaseSiteContext`. **Never rejects a request for a missing header.** Controllers below can call `baseSiteContext.getCurrentBaseSite()` if they need it.
- `CurrentLanguageFilter` (`filter/CurrentLanguageFilter.java:24`) — reads `X-Lang` header. **For routes under `/api/public/` not on its allowlist, rejects with `400` + JSON body `{"errors":[{"field":null,"code":"LANG_MISSING_OR_INVALID","translationKey":"..."}]}` when `X-Lang` is missing or unresolvable** (`CurrentLanguageFilter.java:80-86`, `:101-106`).

`CurrentLanguageFilter` allowlist (no `X-Lang` required) (`CurrentLanguageFilter.java:35-43`):

- prefixes: `/api/public/baseSite/`, `/api/public/maintenance/`, `/api/public/app/version/`
- exact: `/api/public/config`, `/api/public/translations`, `/api/public/health/check`, `/api/public/verify-recaptcha`

`SecurityConfig.filterChain` (`security/config/SecurityConfig.java:77-82`) marks `/api/public/**`, `/api/auth/**`, `/internal/**` as `permitAll()`. None of the five boot endpoints require auth.

`RateLimitFilter.categorize` (`security/filter/RateLimitFilter.java:88-137`) — none of the five boot endpoints map to any rate-limit category. Boot calls are not rate-limited.

`BotFilter` (`security/filter/BotFilter.java:14-63`) blocks requests whose `User-Agent` contains any of {`Googlebot`, `Bingbot`, `Slurp`, `DuckDuckBot`, `YandexBot`, `SemrushBot`, `AhrefsBot`, `facebot`, `Twitterbot`, `MJ12bot`, `Applebot`, `Baiduspider`, `PostmanRuntime`, `python-requests`} with a `204 No Content` (the comment in the file says "204 or 403" — the code emits 204). Note: `PostmanRuntime` and `python-requests` are blocked too. Bypass: `X-Postman: true` works only on the `dev` Spring profile. The Expo app's native HTTP stack does not match these UAs and is unaffected.

---

## 1. MAINTENANCE — `GET /api/public/maintenance/active`

**Controller:** `controller/MaintenancePageController.java:14-17`

```java
@GetMapping("/active")
public ResponseEntity<Boolean> isActive() {
  return ResponseEntity.ok(cloudflareKvService.getMaintenanceActive());
}
```

`CloudflareKvService.getMaintenanceActive()` → `configurationService.getBooleanConfig("maintenance.active")` (`service/impl/DefaultCloudflareKvService.java:33-35`). This reads the in-memory `configurationCache` populated at `@Order(1)` boot; no DB query per request.

### Wire contract

| Aspect | Value |
| --- | --- |
| Method | `GET` |
| Path | `/api/public/maintenance/active` |
| Auth | none |
| Required headers | none |
| Optional headers | none with effect on this endpoint |
| `X-Lang` required? | **No** — path matches allowlist prefix `/api/public/maintenance/` |
| `X-Base-Site` required? | No — endpoint never reads `BaseSiteContext` |
| Query params | none |
| Body params | none |
| Rate-limited? | No |
| Cache-Control header | not set on this endpoint |

### Response shape

```
HTTP/1.1 200 OK
Content-Type: application/json
```

Body: a raw JSON boolean primitive — literally `true` or `false`. The return type is `ResponseEntity<Boolean>`; Spring serializes it as `true` / `false`, not `{"active":true}` and not `"true"`. Two-byte / four-byte / five-byte payload.

### Status codes

- `200 OK` — always, with `true` or `false` in the body.
- `400 Bad Request` — should not occur for this endpoint (no `X-Lang` requirement, no inputs).
- `500 Internal Server Error` — only on a genuine server bug (e.g. `configurationCache` not initialised).

### Maintenance-on, expressible from this endpoint?

**Partially.** The endpoint answers one question: "is the backend's stored `maintenance.active` flag set to `true`?" That is a flag value, not a deployment-wide maintenance state.

Two things to know:

1. The real maintenance gate lives at the Cloudflare worker. Per `oglasino-docs/meta/conventions.md` Part 8: "The Cloudflare router worker is the edge boundary. Maintenance state, admin bypass, and origin forwarding live there." When maintenance is on, the worker serves the static maintenance page and does not forward to backend at all. So if the app's request to `/api/public/maintenance/active` actually reaches backend and returns `false`, that means: the worker is currently letting traffic through, AND the backend's mirror flag is also off.
2. The backend mirror flag is toggled via `POST /api/public/maintenance/toggle` (`MaintenancePageController.java:19-24`), which also PUTs the new value into the Cloudflare KV namespace (`DefaultCloudflareKvService.toggleMaintenance` → `setMaintenanceCloudflareValue`). Backend and KV stay in sync via that path.

So the app cannot rely on `/maintenance/active` alone:

- **Backend unreachable** (network error, 5xx, timeout, or the worker is serving a maintenance page rather than forwarding) → maintenance is the most likely interpretation, but it's also "server crashed" or "network down." The app needs a fallback.
- **Endpoint returns `true`** → maintenance flag is on but traffic is currently still being forwarded (likely a transitional window where someone toggled but the worker hasn't picked up the KV change yet, or KV propagation lag).
- **Endpoint returns `false`** → not in maintenance.

The body of this endpoint by itself does not give the full "are we in maintenance" answer. The app must combine "endpoint returned true" OR "backend unreachable for long enough / responded with the worker's maintenance page".

### Trust boundary

Endpoint exposes a single server-stored boolean. No client input is read, no auth or moderation decisions hinge on any client value. Clean.

---

## 2. APP VERSION / UPDATE — `GET /api/public/app/version/{platform}`

**Controller:** `admin/internal/controller/AppVersionController.java:13-39`

```java
@RestController
@RequestMapping("/api/public/app/version")
public class AppVersionController {

  @Autowired private AppVersionConfigRepository appVersionConfigRepository;

  @GetMapping("/{platform}")
  public ResponseEntity<?> checkVersion(
      @PathVariable String platform, @RequestParam String currentVersion) {

    return ResponseEntity.ok().build();

    //    var config = appVersionConfigRepository.findByPlatform(platform).orElseThrow();
    //
    //    var force = isLower(currentVersion, config.getMinSupportedVersion());
    //    var optional = isLower(currentVersion, config.getLatestVersion());
    //
    //    return ResponseEntity.ok(
    //        new AppVersionResponseDTO(
    //            config.getLatestVersion(), config.getMinSupportedVersion(), force, optional &&
    // !force));
  }
  ...
}
```

**Critical: the live implementation returns `200 OK` with an EMPTY body.** The decision logic that would populate `AppVersionResponseDTO` is commented out and unreachable. The DTO type, the repository, the admin write endpoint, and the entity all exist on the side of the contract, but the public read endpoint is a stub.

### Wire contract (as currently shipping — the stub)

| Aspect | Value |
| --- | --- |
| Method | `GET` |
| Path | `/api/public/app/version/{platform}` |
| Path variable | `platform` — `String`, free-form. No validation. The intent appears to be `android` or `ios`; commented logic does `appVersionConfigRepository.findByPlatform(platform).orElseThrow()`. |
| `currentVersion` query param | required (`@RequestParam` with no default) — `String`. Currently unused; commented body would have parsed it as semver. |
| `X-Lang` required? | **No** — path matches allowlist prefix `/api/public/app/version/` |
| `X-Base-Site` required? | No |
| Auth | none (`/api/public/**` is `permitAll()`) |
| Rate-limited? | No |
| Cache-Control header | not set |

### Response shape — actual

Body is empty (`ResponseEntity.ok().build()`). No JSON. No content type. The client receives `200 OK` with `Content-Length: 0`.

The app cannot read `latestVersion`, `minSupportedVersion`, `forceUpdate`, or `optionalUpdate` from this endpoint because the endpoint does not return them.

### Response shape — intended (commented body)

`dto/AppVersionResponseDTO.java:1-7`:

```java
public record AppVersionResponseDTO(
    String latestVersion,
    String minSupportedVersion,
    boolean forceUpdate,
    boolean optionalUpdate) {}
```

Backing entity `admin/internal/entity/AppVersionConfig.java:1-35` — `platform`, `latestVersion`, `minSupportedVersion` (all `String`).

The commented-out logic in the controller would have computed:
- `forceUpdate = isLower(currentVersion, config.getMinSupportedVersion())` — true when the client is below the floor (HARD update; block).
- `optionalUpdate = isLower(currentVersion, config.getLatestVersion()) && !forceUpdate` — true when the client is at or above the floor but below the latest (SOFT update; notify, continue).

`isLower` uses `com.github.zafarkhaja.semver.Version.parse` (`AppVersionController.java:36-38`), so versions are full semver.

Soft vs hard decision data (intended):

- HARD update: `forceUpdate == true`. Client should block all use until updated.
- SOFT update: `optionalUpdate == true && forceUpdate == false`. Client should notify, allow continue.
- Neither: client is at or beyond `latestVersion`. No prompt.

`minSupportedVersion` and `latestVersion` are returned alongside the booleans, so a client that wants to make the decision itself can do so.

### Status codes

- Currently: `200 OK` with empty body — always (no inputs validated, no DB read).
- If `currentVersion` query param is missing: Spring returns `400 Bad Request` (defaultRequired `@RequestParam`) before the controller body runs.
- If `platform` path segment is empty (e.g. `/api/public/app/version/`): `404 Not Found` from Spring routing (no handler).
- Commented logic, if restored, would throw `NoSuchElementException` on unknown platform via `.orElseThrow()` → `500 Internal Server Error` from `GlobalExceptionHandler` unless caught earlier. There is no validation that `platform` is `android` or `ios`.

### Trust boundary

The endpoint as-shipped reads no client value with consequence. The intended endpoint reads `currentVersion` from the client and uses it only as input to the comparison — the boolean decisions are computed against server-stored thresholds, so the trust posture would be fine (a client lying about its own version only fools itself).

### Adjacent observations (Part 4b)

- **Severity high.** The endpoint is shipped as a stub that returns `200 OK + empty body`. Any Expo client written against the intended `AppVersionResponseDTO` shape will see `undefined`/null fields and either fail JSON parsing or misinterpret "no update required." This is a hidden no-op gate. Not in audit scope to fix; flagging.
- The admin write endpoint `POST /internal/app/version/update` exists and is wired (`admin/internal/controller/AppVersionAdminController.java`). It writes `AppVersionConfig` rows that the commented-out public read code would have consumed. If the public read is uncommented, the seam works.
- `AppVersionController` lives under package `admin.internal.controller` (the same package as the admin-only POST), but its mapping is `/api/public/app/version` — public. Naming/packaging mismatch worth noting.

---

## 3. BASE SITE LIST (picker)

**Controller:** `controller/BaseSiteController.java:18-56`

Three list-ish endpoints exist plus a per-code single read:

```
GET /api/public/baseSite/{targetBaseSiteCode}    → BaseSiteDTO (single, full+catalog)
GET /api/public/baseSite/details                  → List<BaseSiteDTO> (full+catalog, all sites)
GET /api/public/baseSite/overviews                → List<BaseSiteOverviewDTO> (lightweight, all sites)
GET /api/public/baseSite/regions/{code}           → List<RegionDTO> (regions for one site)
```

Yes — there is a clear **lightweight vs heavy distinction**. Flagging explicitly:

- **Lightweight (use this for the picker):** `GET /api/public/baseSite/overviews` returns `List<BaseSiteOverviewDTO>`. Does NOT include catalog, regions, or currencies. This is what the picker needs.
- **Heavy:** `GET /api/public/baseSite/details` returns `List<BaseSiteDTO>` — the same DTO as the per-code endpoint, including `catalog`, `regions`, and `allowedCurrencies` for every site. Used today by `oglasino-web/sitemap.ts` (per `state.md` "Brief 7" — backend keeps it but web is being migrated to `/overviews`).

### `BaseSiteOverviewDTO` (lightweight) — `dto/BaseSiteOverviewDTO.java:5-78`

| Field | Type |
| --- | --- |
| `id` | `Long` |
| `code` | `String` |
| `domain` | `String` |
| `iconId` | `String` |
| `labelKey` | `String` |
| `flagImageKey` | `String` |
| `defaultLanguage` | `LanguageDTO` |
| `allowedLanguages` | `List<LanguageDTO>` |

No `catalog`. No `regions`. No `allowedCurrencies`. No `defaultCurrency`.

The picker has everything it needs from this: the visible label (`labelKey` is a translation key; the app resolves via the translations namespace map), the flag image (`flagImageKey`), the icon, the domain, the per-site allowed languages list (needed once the user picks a site so the app can pick a `defaultLanguage` if not yet set).

### `BaseSiteDTO` (heavy) — `dto/BaseSiteDTO.java:5-114`

Adds on top of the overview:

| Field | Type |
| --- | --- |
| `catalog` | `CatalogDTO` |
| `defaultCurrency` | `CurrencyDTO` |
| `allowedCurrencies` | `List<CurrencyDTO>` |
| `regions` | `List<RegionDTO>` (each region nests cities) |

### Caching topology

All four endpoints return `Cache-Control: max-age=300, public, stale-while-revalidate=86400` (via `ReferenceDataCacheControl.INSTANCE`, `controller/BaseSiteController.java:28-54`, `config/ReferenceDataCacheControl.java:8-12`).

Server-side:
- `/baseSite/overviews` is cached server-side under Redis cache `redisBaseSiteOverviews` (`@Cacheable` on `DefaultBaseSiteService.getAllBaseSiteOverviews`, `service/impl/DefaultBaseSiteService.java:31-56`). Warmed at boot by `CacheWarmupService`.
- `/baseSite/{code}` is `@Cacheable(value = "redisBaseSite", key = "#code", sync = true)` (`DefaultBaseSiteService.java:66-70`). Rebuilt by `VersionChecksumService` at boot if `catalog.checksum.<code>` mismatches or the Redis key is empty.
- `/baseSite/details` is a composition: `DefaultBaseSiteService.getAllBaseSites` reads `findActiveBaseSiteCodes()` and assembles by calling `self.getBaseSiteByCode(code)` for each (`DefaultBaseSiteService.java:58-63`) — i.e. three `redisBaseSite::<code>` reads, not one cache lookup.

### Wire contract (overviews — the picker endpoint)

| Aspect | Value |
| --- | --- |
| Method | `GET` |
| Path | `/api/public/baseSite/overviews` |
| Auth | none |
| `X-Lang` required? | **No** — path matches allowlist prefix `/api/public/baseSite/` |
| `X-Base-Site` required? | No — endpoint does not read `BaseSiteContext` |
| Params | none |
| Cache-Control header | `max-age=300, public, stale-while-revalidate=86400` |
| Body | JSON array of `BaseSiteOverviewDTO` |

### Status codes

- `200 OK` — always when the cache is warm or DB query succeeds.
- `500 Internal Server Error` — if the DB query fails or the warmup invariant is broken. Genuine server bug.

### Trust boundary

No client input drives any of the four endpoints' output (the `targetBaseSiteCode` path variable on the single-read variant is the lookup key — the server validates it against its own DB via `baseSiteRepository.findCoreByCode(code).orElseThrow(...)`). Clean.

---

## 4. BASE SITE FULL DATA (post-pick + returning user) — `GET /api/public/baseSite/{targetBaseSiteCode}`

Same controller as §3 (`controller/BaseSiteController.java:25-31`):

```java
@GetMapping("/{targetBaseSiteCode}")
public ResponseEntity<BaseSiteDTO> getBaseSiteDTO(
    @PathVariable @NotEmpty @NotNull String targetBaseSiteCode) {
  return ResponseEntity.ok()
      .cacheControl(ReferenceDataCacheControl.INSTANCE)
      .body(baseSiteFacade.getBaseSiteData(targetBaseSiteCode));
}
```

Resolves through `BaseSiteFacade → BaseSiteCacheService.getBaseSiteForCode → DefaultBaseSiteService.getBaseSiteByCode` (`@Cacheable("redisBaseSite", key = "#code", sync = true)`).

### Wire contract

| Aspect | Value |
| --- | --- |
| Method | `GET` |
| Path | `/api/public/baseSite/{targetBaseSiteCode}` |
| Path variable | `targetBaseSiteCode` — `String`, `@NotEmpty @NotNull`. Currently active codes: `rs`, `rsmoto`, `me` (from `data-configuration.sql` checksum seeds and `BaseSiteRepository.findActiveBaseSiteCodes()`). |
| Auth | none |
| `X-Lang` required? | **No** — allowlist prefix `/api/public/baseSite/` |
| `X-Base-Site` required? | No |
| Cache-Control header | `max-age=300, public, stale-while-revalidate=86400` |

### Response shape — `BaseSiteDTO`

`dto/BaseSiteDTO.java:5-114`:

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `Long` | |
| `code` | `String` | e.g. `"rs"`, `"rsmoto"`, `"me"` |
| `domain` | `String` | |
| `iconId` | `String` | |
| `labelKey` | `String` | translation key |
| `flagImageKey` | `String` | |
| `defaultLanguage` | `LanguageDTO` | `{id, code, active, label}` |
| `allowedLanguages` | `List<LanguageDTO>` | |
| `catalog` | `CatalogDTO` | mapped via `ModelMapper` from `BaseSite.catalog` — the heavy nested object that includes categories, filters, options, ranges, all the fields hashed in `VersionChecksumService.computeCatalogChecksum` |
| `defaultCurrency` | `CurrencyDTO` | `{id, code, symbol}` |
| `allowedCurrencies` | `List<CurrencyDTO>` | |
| `regions` | `List<RegionDTO>` | regions with nested cities |

### Status codes

- `200 OK` — happy path.
- `500 Internal Server Error` — on unknown code, `loadBaseSiteByCodeFromDb` throws `new RuntimeException("BaseSite not found")` (`DefaultBaseSiteService.java:77-78`). Not a clean 404. Worth flagging — see "Adjacent observations" below.
- `400 Bad Request` — Jakarta validation on `@NotEmpty @NotNull` path variable; in practice unreachable because routing requires a non-empty segment.

### Size characteristics

There is no single canonical "size" — depends on catalog volume. Indicators:

- All three current sites' catalogs round-trip through `BaseSite.catalog → CatalogDTO`. The catalog DTO includes the full category tree, every filter, every filter option, every filter range and its long-value options.
- The same DTO populates the `redisBaseSite::<code>` Redis cache; the catalog Java entity graph fans out across `Catalog → Categories → CategoryFilters → Filters → FilterOptions, FilterRanges, FilterRanges.options`. Per the version-checksums spec §4.5 the catalog input string is large enough that the spec uses a custom delimiter scheme just to keep it deterministic.
- This is "the heavy payload" per `state.md`'s `expo-performance-foundation` notes: among the four bootstrap calls, `/baseSite/details` (which is just N calls of this endpoint composed) is described as one of the "two large-payload responses."

The per-code endpoint is the right shape for a returning-user fetch: one site, one cache key, soft-replaceable when the catalog changes (`VersionChecksumService.rebuildBaseSiteCache(code)` writes a new `BaseSiteDTO` to `redisBaseSite::<code>` atomically).

### Trust boundary

`targetBaseSiteCode` is looked up server-side. Not trusted for any decision other than "which site's data to return." Clean.

### Adjacent observations (Part 4b)

- **Severity medium.** `DefaultBaseSiteService.loadBaseSiteByCodeFromDb` throws `new RuntimeException("BaseSite not found")` on a missing code (`DefaultBaseSiteService.java:75-78`). Per `conventions.md` Part 7, an unknown lookup should not surface as a 500. Should be a 404 with the standard error envelope, or a typed not-found exception that `GlobalExceptionHandler` maps. Pre-existing; out of scope; flagging.
- The `DefaultBaseSiteFacade.getBaseSiteData` throws `IllegalStateException("Base site code can not be null or empty")` for blank input (`facade/impl/DefaultBaseSiteFacade.java:19-21`). Unreachable in normal flow because the path variable is `@NotEmpty @NotNull`; but if someone calls the facade method outside the controller path with a blank code they'd get a 500. Cosmetic; flagging.

---

## 5. VERSION CHECKSUMS — `GET /api/public/versions`

**Controller:** `controller/VersionController.java:16-52`

```java
@RestController
@RequestMapping("/api/public/versions")
public class VersionController {

  private static final List<TranslationNamespace> ALL_NAMESPACES =
      Arrays.asList(TranslationNamespace.values());

  @GetMapping
  public ResponseEntity<VersionsResponseDTO> getVersions() {
    var translations =
        ALL_NAMESPACES.stream()
            .collect(
                Collectors.toMap(
                    TranslationNamespace::name,
                    ns -> configurationService.getConfig("translations.checksum." + ns.name())));

    var catalog =
        baseSiteRepository.findActiveBaseSiteCodes().stream()
            .collect(
                Collectors.toMap(
                    code -> code,
                    code -> configurationService.getConfig("catalog.checksum." + code)));

    return ResponseEntity.ok()
        .cacheControl(ReferenceDataCacheControl.INSTANCE)
        .body(new VersionsResponseDTO(translations, catalog));
  }
}
```

DTO — `dto/VersionsResponseDTO.java:1-5`:

```java
public record VersionsResponseDTO(Map<String, String> translations, Map<String, String> catalog) {}
```

### Wire contract

| Aspect | Value |
| --- | --- |
| Method | `GET` |
| Path | `/api/public/versions` |
| Auth | none |
| `X-Lang` required? | **YES — flag this loudly.** `/api/public/versions` does NOT match any prefix in `ALLOWLIST_PREFIXES` and is NOT in `ALLOWLIST_EXACT` in `CurrentLanguageFilter.java:35-43`. Per `CurrentLanguageFilter.requiresLanguage` (`CurrentLanguageFilter.java:89-99`), any path starting with `/api/public/` that doesn't match the allowlist requires `X-Lang`. **A request to `/api/public/versions` without a valid `X-Lang` header returns `400` with the `LANG_MISSING_OR_INVALID` envelope.** The endpoint's response is identical regardless of which language is sent — the filter requires `X-Lang` even though the controller never uses it. |
| `X-Base-Site` required? | No — endpoint does not read `BaseSiteContext`; the catalog map is built from `baseSiteRepository.findActiveBaseSiteCodes()`. |
| Params | none |
| Body params | none |
| Rate-limited? | No |
| Cache-Control header | `max-age=300, public, stale-while-revalidate=86400` (`ReferenceDataCacheControl.INSTANCE`) |

### Response shape

```
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=300, public, stale-while-revalidate=86400
```

JSON body — exact shape:

```json
{
  "translations": {
    "COMMON":              "<16-hex>",
    "COMMON_SYSTEM":       "<16-hex>",
    "ERRORS":              "<16-hex>",
    "VALIDATION":          "<16-hex>",
    "PAGING":              "<16-hex>",
    "BUTTONS":             "<16-hex>",
    "INPUT":               "<16-hex>",
    "DIALOG":              "<16-hex>",
    "HEADER":              "<16-hex>",
    "FOOTER":              "<16-hex>",
    "NAVIGATION":          "<16-hex>",
    "INTRO":               "<16-hex>",
    "EXTRA_PRODUCTS":      "<16-hex>",
    "COOKIES":             "<16-hex>",
    "MESSAGES_PAGE":       "<16-hex>",
    "DASHBOARD_PAGES":     "<16-hex>",
    "ADMIN_PAGES":         "<16-hex>",
    "ABOUT_PAGE":          "<16-hex>",
    "FREE_ZONE_PAGE":      "<16-hex>",
    "PRICING_PAGE":        "<16-hex>",
    "METADATA":            "<16-hex>",
    "BACKEND_TRANSLATIONS":"<16-hex>"
  },
  "catalog": {
    "rs":     "<16-hex>",
    "rsmoto": "<16-hex>",
    "me":     "<16-hex>"
  }
}
```

Types: both `translations` and `catalog` are `Map<String, String>` (in JSON: an object). Values are 16-character lowercase hex strings (truncated SHA-256), produced by `VersionChecksumService.sha256Hex16` (`service/impl/VersionChecksumService.java:327-339`). Empty string `""` is possible on a fresh boot before checksums have been computed (defensive — `ResponseEntity` returns whatever is in `configurationCache`, and the seed value is `''`).

### Checksums exposed — exhaustive

**Translations (22 entries — one per `TranslationNamespace` enum value):**

`COMMON`, `COMMON_SYSTEM`, `ERRORS`, `VALIDATION`, `PAGING`, `BUTTONS`, `INPUT`, `DIALOG`, `HEADER`, `FOOTER`, `NAVIGATION`, `INTRO`, `EXTRA_PRODUCTS`, `COOKIES`, `MESSAGES_PAGE`, `DASHBOARD_PAGES`, `ADMIN_PAGES`, `ABOUT_PAGE`, `FREE_ZONE_PAGE`, `PRICING_PAGE`, `METADATA`, `BACKEND_TRANSLATIONS` (verified against `entity/TranslationNamespace.java`).

Per-namespace translation checksum covers: **all translations in that namespace across all languages.** Computed by `VersionChecksumService.computeTranslationChecksum` (`service/impl/VersionChecksumService.java:217-229`): hashes `translationKey|translationValue` rows for the namespace, sorted by `(language_code, translation_key)`. So a single namespace's checksum changes if any key/value in that namespace changes in any language (EN, SR, RU, CNR).

There is NO per-namespace-per-language checksum. The granularity is per-namespace (collapsed across languages).

**Catalog (3 entries — one per active base site code from `BaseSiteRepository.findActiveBaseSiteCodes()`):**

`rs`, `rsmoto`, `me`. The seed in `data-configuration.sql:117-119` pre-creates these three keys. If a new base site is added to the DB and marked active, it will appear in the response automatically (the controller iterates `findActiveBaseSiteCodes`), but the value will be `""` until the corresponding `catalog.checksum.<code>` seed row is added and `VersionChecksumService.onAppReady` runs against it.

Per-base-site catalog checksum covers: **the entire catalog state for that base site.** Per `VersionChecksumService.computeCatalogChecksum` (`service/impl/VersionChecksumService.java:231-311`): walks all `CatalogCategoryAssignment` rows for the base site, sorted by `(category.key, position)`. For each category it hashes the category's `key, labelKey, route, iconId, freeZone, position`, then for each filter on that category (sorted by `filterKey`) it hashes `filterKey, displayOrder, basic, enabled`, then options sorted by `labelKey`, then any `FilterRange` and its options. Field-level: a category renamed, a filter added/removed, a filter option's label key changed, a range option's value changed — any of these change that base site's checksum.

The per-base-site catalog checksum does NOT cover per-language catalog labels (the labels themselves live in the translation namespaces — `COMMON`, `INTRO`, etc. — and are covered by translation checksums). Catalog checksum covers structure + key references; translation checksums cover the strings the keys point to.

### Client usage (intended)

Per `oglasino-docs/features/version-checksums.md` §3 "Read flow":

1. Client (mobile) caches the 25 checksums locally alongside the cached payloads (per-namespace translations, per-base-site `BaseSiteDTO`).
2. On boot / on resume / on any "should I refresh?" trigger, the client calls `GET /api/public/versions`.
3. For each of the 22 translation namespaces, the client compares the returned checksum to its locally-stored value. If different, re-fetch that namespace via the existing `/api/public/translations` (which already accepts a namespace filter — confirmed via the `redisTranslations` cache key shape `redisTranslations::<NAMESPACE>:<langCode>` in `VersionChecksumService.java:204`).
4. For each of the 3 base sites, similarly: if `catalog.<code>` differs from local, re-fetch that base site via `GET /api/public/baseSite/<code>`.
5. If a checksum is the empty string `""`, the spec (`features/version-checksums.md` §4.2) says "Mobile treats empty as 'always re-fetch' or as a sentinel; mobile-side decision." Backend's intent: empty means not-yet-computed, which should not happen post-boot but is theoretically observable in a narrow window.

**Per-base-site single-call check?** Yes — the per-base-site catalog checksum lets the client answer "is my stored catalog for base site X still current" with a single `/versions` call. The client compares `catalog["rs"]` (or whichever code it cached) against its local copy; no other call needed for the comparison.

**Single call for translation freshness AND catalog freshness?** Yes — one `GET /api/public/versions` returns both maps in one response. No separate calls needed for the freshness check.

### Status codes

- `200 OK` — happy path, always (no inputs to validate).
- `400 Bad Request` — **WILL be returned when `X-Lang` is missing or unresolvable** (see `X-Lang required?` row above). Body: `{"errors":[{"field":null,"code":"LANG_MISSING_OR_INVALID","translationKey":"..."}]}`.
- `500 Internal Server Error` — only on genuine server bug (e.g. `configurationCache` not initialised). Per `features/version-checksums.md` §4.3: "A 500 here is a genuine server bug, never used for 'checksums not ready.'"
- Maintenance state: per the Cloudflare worker boundary (`conventions.md` Part 8), if maintenance is on the worker serves a maintenance page and the request never reaches backend. The spec's reference to `REFUSING_TRAFFIC` is the readiness-flip state — if backend is up but not ready, the worker should be handling that too.

### Trust boundary

Endpoint reads only from server state (`configurationCache` populated by `DefaultConfigurationService.onAppReady` at `@Order(1)`, and `baseSiteRepository.findActiveBaseSiteCodes()`). Zero client inputs read or trusted. Checksums leak "this slice changed" not "what changed" — and the content itself is already public via existing endpoints. Per `features/version-checksums.md` §7 and the audit there: clean.

### Adjacent observations (Part 4b)

- **Severity high (for Expo boot specifically).** `/api/public/versions` requires `X-Lang` even though its response is language-independent. If the Expo app calls `/versions` during cold boot before it has resolved a language code, the request returns `400`. This is functionally a contradiction with the spec's intent — `features/version-checksums.md` §4.3 implies the endpoint always returns 200 unless the app is in `REFUSING_TRAFFIC`. The four boot endpoints currently allowlisted for no-X-Lang are `/maintenance/`, `/baseSite/`, `/app/version/`, plus exact `/config` and `/translations`; `/versions` is a peer of these but missing from the allowlist. Recommended resolution: add `/api/public/versions` (exact, not prefix — it's a single endpoint) to `CurrentLanguageFilter.ALLOWLIST_EXACT`. Not in audit scope; flagging.

- **Severity low.** The empty-string-as-not-yet-computed contract relies on the seed value being `''` and `VersionChecksumService.onAppReady` running pre-readiness. If `VersionChecksumService` throws for one namespace, that namespace's checksum stays empty in `configurationCache` for the lifetime of that JVM — clients see `""` forever. The service does catch and log exceptions per-namespace and per-base-site, so a single bad row doesn't take down the rest, but the bad slice is unrecoverable without a restart. Pre-existing, intentional per the spec; flagging in case Mastermind wants a recovery story.

---

## Trust-boundary roll-up

| # | Endpoint | Reads client values? | Trust posture |
| --- | --- | --- | --- |
| 1 | `GET /api/public/maintenance/active` | No | Clean |
| 2 | `GET /api/public/app/version/{platform}` | `platform`, `currentVersion` — but the live shipped code uses neither; the intended code uses them only as inputs to server-side comparison, no trust decision | Clean (intended); N/A (shipped — empty body) |
| 3 | `GET /api/public/baseSite/overviews` | No | Clean |
| 3 | `GET /api/public/baseSite/details` | No | Clean |
| 4 | `GET /api/public/baseSite/{code}` | `code` — server-validated lookup | Clean |
| 5 | `GET /api/public/versions` | No | Clean |

No trust-boundary violations across the five.

## X-Base-Site / X-Lang header roll-up

| # | Endpoint | `X-Base-Site` needed? | `X-Lang` needed? | Behavior if header missing |
| --- | --- | --- | --- | --- |
| 1 | `/api/public/maintenance/active` | No | **No** (allowlist prefix `/api/public/maintenance/`) | n/a |
| 2 | `/api/public/app/version/{platform}` | No | **No** (allowlist prefix `/api/public/app/version/`) | n/a |
| 3 | `/api/public/baseSite/overviews` | No | **No** (allowlist prefix `/api/public/baseSite/`) | n/a |
| 3 | `/api/public/baseSite/details` | No | **No** (allowlist prefix `/api/public/baseSite/`) | n/a |
| 4 | `/api/public/baseSite/{code}` | No | **No** (allowlist prefix `/api/public/baseSite/`) | n/a |
| 5 | `/api/public/versions` | No | **YES** (not on allowlist) | `400` + `LANG_MISSING_OR_INVALID` envelope |

The `X-Lang` requirement on `/versions` is the one item to escalate before the Expo redesign relies on it at cold boot.

---

## For Mastermind

- **Required follow-up — `X-Lang` on `/api/public/versions`.** The endpoint is language-independent in its response but the filter chain still demands a valid `X-Lang`. At cold boot the Expo app does not yet know a language, so `/versions` will 400. Either add `/api/public/versions` to `CurrentLanguageFilter.ALLOWLIST_EXACT`, or the Expo boot redesign must defer the `/versions` call until after a language is selected. Engineer's pick; backend change is one line. Flagging here because the version-checksums feature spec (`features/version-checksums.md`) does not mention this requirement and the four-call cold-start burst diagnosed in `issues.md` (2026-05-28) is studying the wrong problem if this also bites.

- **`AppVersionController` shipped as a stub.** `/api/public/app/version/{platform}` returns an empty `200`. The implementation is commented out. If the Expo boot redesign assumes a working soft/hard-update gate, that gate is currently a no-op. The DTO, entity, repository, and admin write endpoint are all wired — uncommenting the controller body is the resolution, but that's a code change outside this audit's scope. Flagging as high-severity because the next boot-redesign brief may build a client decision flow against a contract that the server isn't actually serving.

- **Adjacent: `BaseSite not found` throws a raw `RuntimeException`.** `DefaultBaseSiteService.loadBaseSiteByCodeFromDb` at `DefaultBaseSiteService.java:77` throws `new RuntimeException("BaseSite not found")`. Surfaces as 500 instead of 404. Pre-existing; severity medium; out of audit scope. Not load-bearing for the Expo boot redesign because the app's selected code always comes from `/overviews` and is valid by construction.

- **No cleanup performed and no code changes; audit is read-only per the brief.**
- **Config-file impact:** none.
