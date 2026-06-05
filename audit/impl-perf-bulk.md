# Performance bulk — implementation evidence

**Repo:** oglasino-backend · **Branch:** `dev` (hard rules forbid `git checkout` to a new branch; left uncommitted on `dev`, see Staging note) · **Date:** 2026-06-04

Three independent, code-only changes. No reindex, no config-data changes, no new dependency.
`./mvnw -o spotless:apply compile` clean; `./mvnw -o spotless:check` clean; `./mvnw -o test` → **940 passed, 0 failed, 0 errors** (was 915 baseline + 5 new overload tests + the prior-session uncommitted tree's tests already counted).

---

## Item 1 — `String.intern()` per-UID lock → bounded striped lock (D5)

**File:** `security/service/impl/DefaultFirebaseAuthService.java`

**Guava check:** Guava `33.5.0-jre` is on the classpath transitively via `firebase-admin` (`dependency:tree` confirms `com.google.guava:guava:jar:33.5.0-jre:compile`). `com.google.common.collect.Lists` is already imported in `elasticsearch/facade/impl/DefaultProductsSearchFacade.java`, so relying on Guava is established practice in this repo. Used `com.google.common.util.concurrent.Striped` — **no pom change** (it is already a dependency, as the brief required me to confirm).

**Before:**
```java
/** Synchronized user creation to avoid duplicates. */
private SyncResult createUserSynchronized(
    FirebaseToken token, LoginRequest loginRequest, String providerId) {
  synchronized (token.getUid().intern()) { // synchronize per Firebase UID
    // Check again inside the synchronized block in case another thread already created it
    return getUser(token)
        .map(existing -> new SyncResult(existing, false))
        .orElseGet(
            () -> {
              ... // create new user + registrationTx.execute(...) UNCHANGED
            });
  }
}
```

**After:**
```java
// (field, class level)
private static final Striped<Lock> UID_LOCKS = Striped.lock(64);

/** Synchronized user creation to avoid duplicates. */
private SyncResult createUserSynchronized(
    FirebaseToken token, LoginRequest loginRequest, String providerId) {
  Lock lock = UID_LOCKS.get(token.getUid()); // serialize per Firebase UID
  lock.lock();
  try {
    // Check again inside the locked block in case another thread already created it
    return getUser(token)
        .map(existing -> new SyncResult(existing, false))
        .orElseGet(
            () -> {
              ... // create new user + registrationTx.execute(...) UNCHANGED
            });
  } finally {
    lock.unlock();
  }
}
```

**Preserved exactly:** the in-block re-check (`getUser(token)` inside the lock) and the DB unique constraint on `users.firebase_uid` remain the real correctness guards. `registrationTx.execute(...)` (the commit boundary) is untouched and still runs **inside** the held lock, so the per-UID dedup is intact. The lock is process-local (does NOT prevent cross-instance dupes) — unchanged, and fine, as the DB constraint is the cross-instance guard.

**Why striped instead of the brief's `ConcurrentHashMap` fallback:** the fallback was only specified "if Guava is not a dependency." Guava *is* a dependency, so the brief's preferred path (`Striped.lock(64)`) applies and is strictly better — bounded at 64 stripes, no unbounded map growth, no per-UID object churn.

**Test seam:** no existing concurrency test for this class, and a reliable two-thread same-UID race test would require mocking `FirebaseToken` + repository + `PlatformTransactionManager` + event publisher and coordinating a deterministic race — heavy and flaky for no behavioral delta. Per the brief's stated alternative, confirmed by reasoning: the lock scope is per-UID identical to before (`Striped.get(uid)` maps a UID to a stable stripe; two threads with the same UID contend the same `Lock`), and the re-check + DB constraint (the actual guards) are byte-for-byte preserved. Behavior is identical except for the lock mechanism.

---

## Item 2 — `indexOne` double-refresh (F8)

**File:** `elasticsearch/service/impl/ProductIndexer.java`, `indexOne(Long)`

**Before:**
```java
// Writes go through the alias — at any moment exactly one physical index backs it.
elasticsearchOperations
    .withRefreshPolicy(IMMEDIATE)
    .index(query, IndexCoordinates.of(ALIAS_NAME));
elasticsearchOperations.indexOps(IndexCoordinates.of(ALIAS_NAME)).refresh();
```

**After:**
```java
// Writes go through the alias — at any moment exactly one physical index backs it.
// IMMEDIATE refreshes the alias-backed index as part of the index call, so read-your-write
// holds without a separate refresh.
elasticsearchOperations
    .withRefreshPolicy(IMMEDIATE)
    .index(query, IndexCoordinates.of(ALIAS_NAME));
```

**Confirmation (no caller depended on a side effect beyond IMMEDIATE):** both the `index(...)` call and the removed `refresh()` operated on the **same** target — `IndexCoordinates.of(ALIAS_NAME)` (`"products"`). There is no second index being refreshed; the explicit `refresh()` was redundant with what `RefreshPolicy.IMMEDIATE` already does on the index write. `IMMEDIATE` is left exactly as-is (not changed to `WAIT_UNTIL` or anything else). Read-your-write is unchanged — callers see no difference. `IndexOperations` / `indexOps` / `IndexCoordinates` are still used elsewhere in the class, so no import became unused.

---

## Item 3 — `getIntConfig`/`getLongConfig`/`getDoubleConfig` default-0 footgun — **caller survey + chosen option**

**File surveyed:** `service/impl/DefaultConfigurationService.java:104-115` (the three no-default getters parse `getConfig(key, "0")` → `0` on missing/blank, and throw uncaught `NumberFormatException` on a present non-numeric value).

### Full caller survey (production code; tests excluded)

| Call site | Key | Is `0` safe or a footgun? |
|---|---|---|
| `jobs/ProductRemovalJob.java:31` `getIntConfig` | `product.removal.batch.size` | **Footgun (loud).** `PageRequest.of(page, 0)` throws `IllegalArgumentException("Page size must not be less than one")` → job crashes on a missing key. Not silent, but still wrong. |
| `jobs/ProductRemovalJob.java:32` `getIntConfig` | `product.removal.days.old` | **Footgun (dangerous).** A `0`-day cutoff widens the "old enough to delete" window to *now*, risking premature deletion of soft-deleted products. |
| `redis/service/impl/DefaultRedisViewCounterService.java:34,75` `getLongConfig` | `redis.product.view.delta.ttl` | **Footgun.** A `0ms` Redis TTL breaks delta-expiry semantics. |
| `service/impl/DefaultProductSeenService.java:43` `getLongConfig` | `redis.product.view.owner.ttl` | **Footgun.** TTL `0`. |
| `service/impl/DefaultProductSeenService.java:75` `getLongConfig` | `redis.product.view.dedup.window.ms` | **Footgun-ish.** `0` disables the dedup window. |
| `health/MaintenanceAutoTripService.java:148` `getLongConfig` | `auto.maintenance.locked_until` | **Safe / expected.** This is an epoch-millis timestamp; `0` = "epoch = not locked / lock expired". `0` is the intended absent-key meaning here. |
| `getDoubleConfig` | — | **No production callers** (interface declaration only). |

**The brief's motivating "timeout" consumer** is `openai/service/impl/DefaultOpenAIService.resolveTimeoutMs`. Reality check vs the brief: it does **NOT** use `getIntConfig` — it reads `getConfig(key)` (the String form) and runs its own blank-check + parse + **non-positive guard**. So there was never a `getIntConfig` timeout caller to migrate; the timeout path was already guarded.

### Chosen option: **A (additive overloads)** — with one deviation, flagged

Added to the interface + impl (no change to the existing `0`-returning methods, so **no current caller changes behavior**):

```java
int    getIntConfig(String configKey, int defaultValue);
long   getLongConfig(String configKey, long defaultValue);
double getDoubleConfig(String configKey, double defaultValue);
```
Semantics: return `defaultValue` on **missing / blank / unparseable** (warn-logged on unparseable); a **present** value — including a present `0` — is returned as-is. `0` is deliberately NOT treated as "missing", because `0` is a legitimate value for many configs; rejecting an out-of-range present value (e.g. a non-positive timeout) stays the caller's job.

**Migrated the timeout consumer** (`DefaultOpenAIService.resolveTimeoutMs`) to the new overload:
```java
private int resolveTimeoutMs(String configKey, int fallbackMs) {
  int value = configurationService.getIntConfig(configKey, fallbackMs);
  return value > 0 ? value : fallbackMs;
}
```
This delegates the missing/blank/unparseable→fallback logic to the shared overload (deleting the local blank-check + try/catch parse) while **keeping** the `value > 0` non-positive guard at the call site.

**Deviation from the brief, flagged:** the brief said "remove the local `resolveTimeoutMs` workaround if it's now redundant." It is **NOT fully redundant** — it additionally guards a **present-but-non-positive** value (`0`/negative), which in Apache HttpClient 5 means *infinite timeout* (the exact hang). The generic overload deliberately does not bake in a non-positive guard (it would be wrong for non-timeout configs where `0` is valid). So `resolveTimeoutMs` is kept (slimmed to a 2-liner). Per the brief's own conditional ("*if* it's now redundant"), keeping it is correct.

**Footgun callers NOT migrated (deliberate, reported):** the TTL / batch-size / days-old callers each need a *chosen non-zero default value* — a business decision with no value specified anywhere in the repo. Inventing those numbers would be value-guessing and scope expansion the brief warns against. They are left as-is; the new overload makes each a one-line migration once Igor/Mastermind specify the defaults. Recommend a follow-up brief. `auto.maintenance.locked_until` should stay on the `0`-returning getter (`0` is its correct default).

**Test added:** `DefaultConfigurationServiceTest` (5 tests) — default-on-missing, default-on-blank, default-on-unparseable, present-value-returned, and present-`0`-returned-as-is (pins the non-positive nuance).

---

## Staging note (hard rule: stage only this session's files)

The working tree had **pre-existing uncommitted work** (from prior sessions) that overlaps two of the files I touched:

- `ProductIndexer.java` already carried an uncommitted `productRepository.fetchImageKeysByIds(...)` addition in `doReindex` and an uncommitted removal of `Hibernate.initialize(product.getFilterValues())` in `indexOne`. My only change here is the `.refresh()` removal (Item 2).
- `DefaultOpenAIService.java` already carried the entire uncommitted timeout infrastructure (`resolveTimeoutMs`, the `RequestConfig`/`Timeout` block, the `*_TIMEOUT_MS_KEY` constants — "the last session's workaround" the brief references). My only change here is the body of `resolveTimeoutMs` (Item 3 migration).

Because `git add <file>` cannot split file-level hunks and interactive `git add -p` is unavailable in this environment, I **staged only the four clean-baseline, this-session-only files**:
`DefaultFirebaseAuthService.java`, `ConfigurationService.java`, `DefaultConfigurationService.java`, `DefaultConfigurationServiceTest.java`.

My Item 2 + Item 3 edits remain in the working tree (unstaged) inside `ProductIndexer.java` and `DefaultOpenAIService.java`, mixed with the prior uncommitted work there. **Igor:** stage/commit those two files selectively (or commit the prior work alongside, your call). No `dev` commit was made.
