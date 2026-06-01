# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-28
**Task:** /seen view-count re-counts on refresh — dedup path audit (READ-ONLY)

## Implemented

- Read-only audit. No code changes. Read `DefaultProductSeenService`, `RedisRateLimiterService`, `DefaultRedisViewCounterService`, `PublicProductController`, `RateLimitFilter`, `DefaultConfigurationService`, and `data-configuration.sql`.
- Established that the dedup mechanism IS wired (the viewer key gates the increment via `RateLimiterService.tryConsume`), but it is configured as a *rate limiter*, not a *deduper* — capacity=20 tokens, refill=1 token/sec, per viewer key. A normal user refreshing a product page never depletes the bucket, so every refresh returns `allowed=true` and increments the delta.
- Confirmed (no) effect from the just-shipped `skipAuth: true` web change: `viewerKey` has no dependency on the security context or any cookie-derived value.
- Identified a secondary issue: web does not send `X-Device-Id`, so the key falls back to `request.getRemoteAddr()`. Service-level fallback uses raw `getRemoteAddr()` while the adjacent `RateLimitFilter.callerKey` reads `CF-Connecting-IP` first — service path does not see the Cloudflare-resolved client IP. Behind a proxy this *collapses* the key (over-suppress direction) but the catastrophic capacity/refill ratio makes this irrelevant to the observed re-counting.

## Files touched

- (none — read-only audit)

## Tests

- Ran: `./mvnw test`
- Result: 678 passed, 0 failed, 0 errors, 0 skipped — BUILD SUCCESS
- Ran: `./mvnw spotless:check` — clean (no output)
- New tests added: none (read-only)

## Cleanup performed

- No temp logs were added during the audit. `git diff` against `HEAD` shows only the three pre-existing modifications already present at session start (`PublicProductController.java`, `DefaultProductSeenService.java`'s `Long::getLong` → `Long::parseLong`, and `data-configuration.sql` keyword-stuffing config rows) — all unrelated to dedup and out of scope.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (Mastermind decides whether the root cause becomes an issue entry; the recommended draft text lives in "For Mastermind" below)

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, no temp logs added, no debug println, no TODO/FIXME added
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): one flag in "For Mastermind" (service-level `remoteAddr` ignoring `CF-Connecting-IP`)
- Part 6 (translations): N/A this session
- Other parts touched: Part 11 (trust boundaries) — confirmed irrelevant: `viewerKey` derivation does not read from `SecurityContextHolder`, so `skipAuth: true` on the web side cannot have changed it

## Known gaps / TODOs

- The audit does not validate stage Redis state empirically (no `redis-cli` access from this session). All conclusions are derived from code + seeded config values. If Mastermind wants a runtime confirmation of the bucket state on stage, that is a follow-up brief.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing
  - Considered and rejected: nothing
  - Simplified or removed: nothing

---

### Read-only audit — structured findings

#### Point 1 — Is `viewerKey` actually CHECKED before incrementing?

**Yes.** `DefaultProductSeenService.handleProductViewInternally` (lines 72–81):

```java
private void handleProductViewInternally(Long productId, String viewerKey) {
  String capacity = configurationService.getConfig("redis.product.view.rate.limiter.capacity");
  String refillPerSecond =
      configurationService.getConfig("redis.product.view.rate.limiter.refill.per.second");

  boolean allowed = rateLimiterService.tryConsume(viewerKey, capacity, refillPerSecond);
  if (!allowed) return;

  redisViewCounterService.incrementViewDeltaBy(productId, 1);
}
```

The increment is gated on `rateLimiterService.tryConsume(viewerKey, …)`. `RedisRateLimiterService` (lines 13–36) is a token-bucket Lua script: keys `viewerKey:tokens` and `viewerKey:ts`, refills by `(now - last_ts)/1000 * refill_rate` capped at `capacity`, consumes 1 token if `filled >= 1`, returns 1 (allowed) or 0 (denied).

**Verdict:** dedup machinery is wired. The key IS checked before the increment. This rules out classification (A).

#### Point 2 — If wired, what's the dedup window?

The mechanism is not a TTL-based dedup; it is a token-bucket rate limiter. Effective "window" derived from configured values in `src/main/resources/data/configuration/data-configuration.sql`:

```
(14, 'redis.product.view.rate.limiter.capacity',           '20', …)
(15, 'redis.product.view.rate.limiter.refill.per.second',  '1',  …)
(11, 'redis.rate.limiter.ttl',                             '600000', …)  -- 10 min, key TTL only
```

The bucket starts at 20, refills at 1 token/sec, cap 20. A single viewer can refresh up to **20 times back-to-back** before being throttled, and refilling 1 token/sec means **sustained refreshes never trip the limit**. Bucket-key TTL is 10 min, so even idle ≥10 min "resets" the bucket to capacity (allowing another 20 burst).

The Lua script falls back to `capacity` when no key exists (`tokens = tonumber(redis.call("GET", tokens_key)) or capacity`, line 22) — there is no "first visit allowed, all others denied" semantic anywhere.

**Verdict:** the threshold for a *normal user refresh* (1 view, 1-30 sec apart) is roughly never. This is the classic shape of (B) "dedup wired but window too short / TTL unset / misconfigured" — except here the misconfiguration is more fundamental: a rate limiter was wired where a deduper is needed. With these values, dedup never engages.

#### Point 3 — What does `viewerKey` actually resolve to on a normal browser refresh?

Code (`DefaultProductSeenService.handleProductView` lines 51–56):

```java
String deviceId = request.getHeader("X-Device-Id");
String remoteAddr = request.getRemoteAddr();
String viewerKey =
    (deviceId != null && !deviceId.isBlank())
        ? "view:" + productId + ":" + deviceId
        : "view:" + productId + ":" + remoteAddr;
```

- **Does the web `markAsSeen` call send `X-Device-Id`?** Per the brief's cross-repo note, no. Web has no per-installation device-id concept; `X-Device-Id` is sent by mobile (see the same-header use at `RateLimitFilter.java:148`). Web stage traffic therefore hits the `remoteAddr` branch.
- **`skipAuth: true` impact:** none. Neither `deviceId` nor `remoteAddr` reads from cookies, headers other than `X-Device-Id`, or `SecurityContextHolder`. The owner-skip check at line 49 (`currentUserId.equals(productOwnerId)`) reads identity, but that only *short-circuits* the call for the seller — it is not part of `viewerKey`. Conclusion: **stripping cookies + Authorization on the web side did NOT alter any input to `viewerKey`.** This rules out the "did our own fix cause this" hypothesis.
- **`remoteAddr` behind the stack:** on stage, the request lands at the Spring filter chain after passing through the Cloudflare worker. `RateLimitFilter.callerKey` (lines 155–158) reads `CF-Connecting-IP` first and falls back to `request.getRemoteAddr()`. **`DefaultProductSeenService` does NOT do this** — it reads raw `getRemoteAddr()`, which is the Cloudflare worker's upstream connection peer, not the real client IP. So all web stage viewers (X-Device-Id absent) share a single near-constant `viewerKey` per product. This would *over-suppress* (the entire visiting population shares one bucket), not under-suppress. Empirically the count rises on every refresh, so this collapse is not the cause — the rate-limit window is just too lax to throttle even a one-bucket population.

**Verdict:** key shape on a normal web refresh is `view:<productId>:<cloudflareUpstreamIp>` (because `X-Device-Id` is absent). Key-collapse is real but pushes the wrong direction. Classification (C) is *present* as a secondary issue but is not the root cause.

#### Point 4 — Is the dedup store defeated by a parse/serialization bug?

No. The rate limiter's Lua script reads tokens/timestamp via `tonumber(redis.call("GET", …))` (raw Redis GET, no Java-side deserialization) and writes via `SET … PX ttl` (raw string). `StringRedisTemplate` is used for the script execution, which is consistent with the string values written by the script. No `Long.getLong` / boxed-cache misuse here (the prior bug of that flavor was on the *owner* key in `handleProductView` itself — already fixed in the working tree's `Long::getLong → Long::parseLong` change).

`tryConsume`'s reads of `capacity` / `refillPerSecond` go through `ConfigurationService.getConfig` (string-returning, in-memory cache) → passed as ARGV strings → `tonumber` in Lua. The values `"20"` and `"1"` parse cleanly to 20 and 1.

**Verdict:** no parse/serialization defect in the dedup store. Classification (D) is ruled out.

---

### Single named root cause

**(B) — dedup wired but the window is effectively infinite under the configured token-bucket parameters.**

`capacity=20` + `refill=1/sec` is a generous *rate limiter* against rapid scripted abuse; it is **not a deduper**. For a normal user refreshing a product page once every few seconds, `tryConsume` returns `true` indefinitely, the increment runs unconditionally, and `numberOfViews` rises on every refresh. The mechanism is correctly invoked and the keys are correctly composed; the configured parameters do not implement the "count once per viewer per product per window" semantic the feature requires.

A secondary contributor (C) is the X-Device-Id-absent + `remoteAddr`-not-Cloudflare-aware key collapse — every web visitor on stage shares one bucket per product. This does not cause the observed re-counting (it would, if anything, suppress counts further). It does cause a separate latent bug: when one rapid abuser somewhere on the internet burns through 20 tokens on a popular product, all other Cloudflare-upstream visitors are silently denied the count too. Fix shape for (C) is independent and lower priority than (B).

### Fix shape (the actual fix is a separate brief)

The primary fix is to replace token-bucket-as-deduper with a real dedup primitive:

1. **Preferred — Redis `SET key value NX PX <window-ms>`** keyed by `view:<productId>:<viewerId>`. First view per (product, viewer) within `<window-ms>` returns `OK`, increment runs; subsequent attempts return `nil`, skip the increment. Window can be a single config entry like `redis.product.view.dedup.window.ms` (e.g. 30–60 minutes). Drop the token-bucket call from this path entirely.
2. **Alternative — keep token-bucket but reconfigure** to `capacity=1`, `refill_per_second` ≈ `1 / window_seconds`. Functionally equivalent to SETNX but harder to reason about. Not recommended.

Secondary fix (independent, lower priority): align `DefaultProductSeenService`'s `remoteAddr` derivation with `RateLimitFilter.callerKey` — read `CF-Connecting-IP` first, fall back to `request.getRemoteAddr()`. Without this, even after the primary fix lands, every Cloudflare-upstream visitor on a base-site is treated as one viewer per product when `X-Device-Id` is absent. Web may also want to send a stable per-installation/session id (cookie or `localStorage`-backed) as `X-Device-Id` so the key is per-browser rather than per-network — that is a cross-repo concern and belongs in a web brief.

### Adjacent observation (Part 4b)

- `DefaultProductSeenService.handleProductView` line 52 uses `request.getRemoteAddr()` directly, while the adjacent `RateLimitFilter.callerKey` correctly reads `CF-Connecting-IP` first (file `src/main/java/com/memento/tech/oglasino/security/filter/RateLimitFilter.java:155-158`). Severity: medium — it does not cause the *current* observed defect but it does cause every web stage view to share one bucket per product, which both masks legitimate counts and would make any future per-viewer dedup mis-behave the same way. I did not fix this because it is out of scope (read-only audit). Mastermind should fold it into whichever brief covers the primary fix.

### Suggested issues.md draft (Mastermind decides)

If Mastermind wants this captured as an issue while the fix brief is queued:

> **2026-05-28 — Stage: refreshing a product page increments `numberOfViews` on every refresh.** Root cause: `DefaultProductSeenService.handleProductViewInternally` uses `RateLimiterService.tryConsume` (capacity=20, refill=1/sec, per viewer key) as the gate on the increment. With those parameters a normal user refresh never depletes the bucket — the gate always allows and the delta increments every time. Classified (B) per the dedup audit. Fix shape: replace the token-bucket call on this path with a Redis `SET NX PX <window-ms>` keyed by `view:<productId>:<viewerId>`. Secondary: align `getRemoteAddr` derivation with `RateLimitFilter.callerKey` (read `CF-Connecting-IP` first). `skipAuth: true` web change ruled out as a cause — `viewerKey` has no auth-derived input. Audit: `.agent/2026-05-28-oglasino-backend-seen-dedup-audit-1.md`.
