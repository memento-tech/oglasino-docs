# Audit — AdminTranslationsController.refreshTranslationsCache

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-27
**Task:** READ-ONLY AUDIT of `AdminTranslationsController.refreshTranslationsCache` per issues.md 2026-05-27 entry.

---

## 1. Locate the endpoint

**File:** `src/main/java/com/memento/tech/oglasino/admin/controller/AdminTranslationsController.java`
**Lines:** 27–31

```java
@GetMapping("/refresh-cache")
public ResponseEntity<?> refreshTranslationsCache() {
  translationService.indexTranslations();
  return ResponseEntity.ok().build();
}
```

Controller-level annotations (lines 20–23):

```java
@RestController
@RequestMapping("/api/secure/admin/translations")
@PreAuthorize("hasRole('ADMIN')")
public class AdminTranslationsController {
```

---

## 2. Caller inventory inside the backend

Grepped `src/` for any Java method call to `refreshTranslationsCache`.

**Result: zero callers.** The only match is the method definition itself at line 28. No other class calls this method.

---

## 3. Endpoint route

**`GET /api/secure/admin/translations/refresh-cache`**

Composed from the class-level `@RequestMapping("/api/secure/admin/translations")` and the method-level `@GetMapping("/refresh-cache")`.

---

## 4. Web caller inventory

The `issues.md` entry claims zero web callers after Brief 11. Evidence:

- The `decisions.md` 2026-05-27 "version-checksums feature shipped" entry states under "Scope shipped — unified admin cache page (Briefs 10–13)":

  > "Standalone 'Keš prevoda' section deleted; `redisTranslations` joins the unified list. `RefreshTranslationCacheButton` file deleted."
  > "5 old admin cache endpoints (`GET /admin/cache`, `POST /admin/cache/evict/{name}`, `POST /admin/cache/evict`, `POST /admin/cache/warmup`, `POST /admin/cache/refresh`) deleted from `CacheAdminController` after web migration (Brief 13)."

- `AdminTranslationsController.refreshTranslationsCache` is a **separate endpoint** from those 5 deleted endpoints. It lives in `AdminTranslationsController`, not `CacheAdminController`. The 5 deletions from `CacheAdminController` are confirmed — that file now contains only 3 endpoints: `GET /list`, `POST /{cacheName}/clear`, `POST /{cacheName}/warmup`.

- The web's unified admin cache page now uses the per-cache `POST /api/secure/admin/cache/{cacheName}/clear` and `POST /api/secure/admin/cache/{cacheName}/warmup` endpoints for all 8 caches including `redisTranslations`. The `RefreshTranslationCacheButton` (which was the web caller of this endpoint) was deleted.

**Conclusion:** zero web callers exist. The endpoint survived the Brief 11/13 deletion because it is in a different controller.

---

## 5. What the endpoint does

Full method body (lines 28–31):

```java
public ResponseEntity<?> refreshTranslationsCache() {
  translationService.indexTranslations();
  return ResponseEntity.ok().build();
}
```

`TranslationService.indexTranslations()` implementation in `DefaultTranslationService` (lines 44–47):

```java
@Override
public void indexTranslations() {
  indexBackendTranslations();
  self.evictAllTranslationCaches();
}
```

This does two things:
1. **`indexBackendTranslations()`** (lines 140–151) — reloads the `BACKEND_TRANSLATIONS` namespace from DB into a JVM-local `Map<String, Map<String, String>>` used by `getBackendTranslation()`.
2. **`self.evictAllTranslationCaches()`** (lines 137–138) — calls through the `@Lazy` self-injection proxy to hit `@CacheEvict(value = "redisTranslations", allEntries = true)`, evicting all entries from the `redisTranslations` Redis cache.

It does **not** rebuild the caches after eviction. It does not return early on any condition. The response is always `200 OK` with an empty body.

---

## 6. Adjacent endpoints in the same controller

`AdminTranslationsController` has 3 endpoints total (lines 27–49):

```java
@GetMapping("/refresh-cache")
public ResponseEntity<?> refreshTranslationsCache()

@GetMapping("/filtered")
public ResponseEntity<List<TranslationDTO>> getFilteredTranslations(
    @RequestParam TranslationNamespace namespace,
    @RequestParam String lang,
    @RequestParam(required = false) String key,
    @RequestParam(required = false) String value)

@PostMapping("/update")
public ResponseEntity<?> updateTranslation(
    @RequestBody @Valid @NotNull TranslationUpdateRequestDTO translationUpdateRequest)
```

Note: `/update` also returns `ResponseEntity.ok().build()` (empty body). `/filtered` returns `ResponseEntity.ok(List<TranslationDTO>)` (body with data).

---

## 7. Security gating

The controller is annotated at the class level with:

```java
@PreAuthorize("hasRole('ADMIN')")
```

All three endpoints (including `refreshTranslationsCache`) inherit this. The endpoint is under the `/api/secure/admin/**` path, which aligns with the Spring Security configuration's admin-only matcher. Admin-only protection confirmed.

---

## Verdict

**Delete the endpoint.**

Rationale:

1. **Zero callers — Java or HTTP.** No internal Java code calls the method. No web UI calls the HTTP endpoint (the `RefreshTranslationCacheButton` was deleted in Brief 11).

2. **Redundant with the unified admin cache page.** The new `CacheAdminController` exposes `POST /api/secure/admin/cache/redisTranslations/clear` (for eviction) and `POST /api/secure/admin/cache/redisTranslations/warmup` (for rebuild via `VersionChecksumService`). Both are strictly superior:
   - The new clear endpoint evicts the same `redisTranslations` cache.
   - The new warmup endpoint rebuilds the cache after eviction (via `versionChecksumService.rebuildTranslationCachesIfNeeded()`), which the old endpoint does not do — the old endpoint evicts and leaves the cache cold until the next consumer request triggers a cache miss.
   - The new endpoints return structured responses via the unified cache page UX.

3. **`indexTranslations()` is now an orphan.** The method in `TranslationService` / `DefaultTranslationService` exists only to serve this endpoint. `updateTranslation()` uses the newer `versionChecksumService.rebuildTranslationCacheAsync()` path instead. If the endpoint is deleted, `indexTranslations()` and the `TranslationService` interface method should also be deleted.

4. **The endpoint's behavior is actively worse than doing nothing.** It evicts all translation caches without rebuilding, leaving every namespace×language cache slot cold. The next consumer request for each slot pays a DB round-trip. The unified cache page's clear+warmup sequence does not have this gap.

5. **Manual curl/Postman use case is served by the new per-cache endpoints.** An admin who wants to force-refresh translations from the command line can `POST /api/secure/admin/cache/redisTranslations/clear` then `POST /api/secure/admin/cache/redisTranslations/warmup`. These are documented by the `GET /api/secure/admin/cache/list` endpoint.

**Deletion scope:**
- Delete the `refreshTranslationsCache()` method from `AdminTranslationsController.java` (lines 27–31).
- Delete `indexTranslations()` from `DefaultTranslationService.java` (lines 44–47) and its declaration from `TranslationService.java` (line 10).
- Verify `evictAllTranslationCaches()` has no other callers before deciding whether to delete it too. (It is called only from `indexTranslations()` based on the grep, but confirm.)

---

## Cleanup performed

None — read-only audit.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: the existing 2026-05-27 entry for this endpoint should be updated with the audit verdict once the deletion brief ships. Draft for Mastermind below.

## Obsoleted by this session

Nothing — read-only audit, no code changes.

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit
- Part 4a (simplicity): N/A — read-only audit
- Part 4b (adjacent observations): confirmed — see "For Mastermind" below
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit
  - Considered and rejected: nothing — read-only audit
  - Simplified or removed: nothing — read-only audit

- **Verdict: delete the endpoint and its supporting method `indexTranslations()`.** The endpoint is zero-caller, redundant with the unified admin cache page, and actively worse than the replacement (evict without rebuild). A deletion brief should cover the three-item scope listed in the verdict above.

- **Adjacent observation (Part 4b):** The `/update` endpoint in the same controller also returns an empty body (`ResponseEntity.ok().build()` at line 48). This is the same pattern flagged by issues.md for `/refresh-cache`. Severity: low — the update endpoint triggers an async cache rebuild via `versionChecksumService.rebuildTranslationCacheAsync()` so the lack of response body is less operationally harmful (the side effect happens regardless), but a `{ "updated": true }` body would match the pattern expected by the admin translation page's toast-on-success UX. I did not fix this because it is out of scope.

- **Config-file impact draft (issues.md):** Once the deletion brief ships, the 2026-05-27 issues.md entry "AdminTranslationsController.refreshTranslationsCache returns empty body" should be updated to status `fixed` with a note that the endpoint was deleted rather than given a response body.
