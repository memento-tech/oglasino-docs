# Performance Overview Audit — oglasino-backend

**Type:** exploratory, read-only. Not part of a feature, not scoped to a fix.
**Branch:** dev · **Date:** 2026-06-04 (v2 — re-verified end-to-end and extended with deeper
findings + implementation suggestions; supersedes the v1 overview of the same name)
**Question:** for the endpoints real users hit most, where does wall-clock time go on a request, and
what concrete changes would move the needle?

**Method:** static reading of controllers → facades → services → ES/DB/Redis/external, every claim
cross-checked against source with grep/cat (per the fabrication-hazard note). Where the truth needs a
profiler/APM trace to know, this says so rather than inventing a number — **there are no latency
figures in this document that were measured.** All "cost" language is structural (round-trips,
blocking vs offloaded, payload shape), not timed.

**What changed since v1 (read this first):**

- **Two v1 "needs verification" items are now resolved.**
  - **reCAPTCHA is _definitively_ a no-timeout `new RestTemplate()`** (`DefaultReCaptchaService.java:13`)
    — not the bounded bean. v1 left this ambiguous; it is now confirmed and **upgraded to HIGH**
    because it sits on a public, captcha-gated path.
  - **The "R2 delete inside an open transaction" speculation is FALSE.** Every delete flow keeps R2
    out of the JPA connection-holding window (user-delete via `TransactionTemplate` post-commit;
    product-delete on a virtual thread; image-orphan and profile-image paths carry no surrounding
    transaction). v1's speculative finding #6 is **closed, not a risk.** Details in §6.
- **New findings v1 missed**, all verified to source: ES fetches the full `_source` per hit with no
  source filtering (the single biggest read-path win, §2/F1); `track_total_hits=true` forces an exact
  full count on every search (§2/F3); **SMTP has no timeout in any env yaml** (§3/HIGH); the
  view-counter flush does an N+1 sequential Redis sweep (§4/D4); `String.intern()` per Firebase UID is
  a latent contention/memory risk (§3a/D5).
- **One v1 worry is retracted:** the Redis cache uses **Jackson JSON** serialization, not JDK
  serialization, so there is no slow-serializer concern (§4/D2).

**Out of scope (owned by other planned features — do not re-litigate here):**

- **JPA fetch tuning / batch-fetch / the listing-and-report N+1s** → owned by `jpa-fetch-tuning`
  (`features/jpa-fetch-tuning.md`, status `planned`). Batch 1 (`hibernate.default_batch_fetch_size: 50`)
  has already landed in all three env yamls. Anything DB-fetch-shaped below is named only to draw the
  backend boundary, not to propose a fix.
- **The structural connection-pool fix** (per-key single-flight on cache misses, removing
  `@Transactional` from `DefaultBaseSiteService.getAllBaseSites`, the `redisBaseSites` cold-window) →
  owned by `connection-pool-hardening` (`features/connection-pool-hardening.md`, status `planned`).
- **Graduated DB-pressure throttling / auto-maintenance-trip** → owned by `db-overload-protection`
  (status `planned`).

This overview owns the **ES query shape, the Redis-call shape, the external-call timeout posture, and
payload size** — the things none of those three features cover.

---

## 0. TL;DR

- **The hot read path is genuinely ES-dominated and structurally clean.** Product search, listing,
  autocomplete, count, product detail, and favorites-overview each make **one Elasticsearch query**
  (favorites adds **one** Postgres id-page read first) and then map hits **purely in memory** — no
  per-hit DB/Redis/ES call. That part of v1 holds and is the good news.
- **But the ES queries are not _tuned_, only _shaped_ right.** Three concrete, low-risk wins sit on
  the hottest path: (1) the query pulls the **entire `_source`** for every hit including the unused
  `descriptionTranslations` blob — add source filtering (F1, highest-value read win); (2)
  `track_total_hits=true` forces ES to **exactly count every match** on every search (F3); (3)
  autocomplete runs the **full bool search + full `_source`** to return three fields (F4). None of
  these are visible without reading the query builder, which is why v1 missed them.
- **Auth on every request is cheap and local** — local RSA verify + one Redis GET (Jackson-JSON
  serialized), confirmed. The one wrinkle: the auth filter runs that full verify+GET **even on
  `/api/public/**`** when a logged-in user browses (D6), and the per-UID first-login lock uses
  `String.intern()` (D5, latent contention).
- **The real latency tail is unbounded synchronous external calls on action/auth endpoints** — and
  the re-check found **more missing timeouts than v1**: reCAPTCHA (HIGH, public), SMTP (HIGH, no
  timeout in any yaml), OpenAI (HIGH), R2/S3 SDK (MED), legacy Cloudflare-KV (LOW, already logged).
  None are on the browse path; all are robustness fixes, but four of the five are genuinely unbounded.

**If you act on a short list later:** (1) add the four missing client timeouts — reCAPTCHA, SMTP,
OpenAI, R2 — they're config/one-line changes that remove unbounded hangs; (2) add ES source filtering
+ bound `track_total_hits` on the listing/autocomplete path — the only app-layer browse-latency wins
that exist. Everything else is small.

---

## 1. Hot endpoint → backend map (the centerpiece)

"Round-trips" = calls made **on the request thread, before the response is written**. Async/virtual-
thread/scheduled work is listed separately as "offloaded." `redisUserAuth` (the per-request auth cache
read) is omitted from every secured row below — assume +1 Redis GET (cache hit) or +1 Postgres
JOIN-FETCH (cache miss) on **every** `/api/secure/**` request (and, per D6, on public requests that
carry a token), described once in §3a.

| #   | Endpoint                                             | Controller                          | ES                   | Postgres                                               | Redis                    | External                               | Offloaded                                        |
| --- | ---------------------------------------------------- | ----------------------------------- | -------------------- | ------------------------------------------------------ | ------------------------ | -------------------------------------- | ------------------------------------------------ |
| 1   | `POST /api/public/product/search` (listing)          | ProductSearchController              | **1** (search, full `_source`, exact total) | 0                        | 0                        | 0                                      | —                                                |
| 2   | `GET /api/public/product/search?productId=` (detail) | ProductSearchController              | **1** (search-by-id) | **2** (favorites counter + views counter, both by PK)  | 1 GET (view delta); base-site cached | 0                          | —                                                |
| 3   | `POST /api/public/product/search/autocomplete`       | ProductSearchController              | **1** (full search!) | 0                                                      | 0                        | 0                                      | —                                                |
| 4   | `GET /api/public/product/count`                      | PublicProductController              | **1** (`_count` API) | 0                                                      | 0                        | 0                                      | —                                                |
| 5   | `GET /api/public/product/seen/{id}` (view ping)      | PublicProductController              | 0                    | 0–1 (owner-id on cache miss)                           | 1 GET (owner) + 0–1 SET  | 0                                      | dedup SET + delta increment → **virtual thread** |
| 6   | `GET /api/public/product/views/{id}`                 | PublicProductController              | 0                    | **1** (views counter, by PK)                           | 1 GET (delta)            | 0                                      | —                                                |
| 7   | `GET /api/public/review?productId/userId` (page=20)  | PublicReviewController               | 0                    | **≥1** (paged reviews; per-row lazy loads = JPA scope) | 0                        | 0                                      | —                                                |
| 8   | `POST /api/auth/firebase-sync` (login)               | AuthController                      | 0                    | 1–N (find user; insert + event on first login)         | auth-cache evict on save | `verifyIdToken` ×2 **(local, see §3a)** | registration email → `@Async` AFTER_COMMIT       |
| 9   | `GET /api/secure/user/...` profile reads             | UserController / UserDataController | 0                    | 0–1 (`redisUserInfo` miss)                             | 1 GET auth + 1 GET `redisUserInfo` (D3) | 0                       | —                                                |
| 10  | `POST /api/secure/favorites/overviews`               | FavoriteProductsController          | **1** (multi-id)     | **1** (favorite-id page)                               | —                        | 0                                      | —                                                |
| 11  | `GET /api/secure/favorites` (ids only)               | FavoriteProductsController          | 0                    | **1** (id list)                                        | —                        | 0                                      | —                                                |
| 12  | `GET /api/secure/favorites/addRemove` (write)        | FavoriteProductsController          | 0                    | several (load user+favorites, update counter, save)    | —                        | 0                                      | "saved product" notification → `@Async`          |
| 13  | `POST /api/secure/follow/{id}` (write)               | FollowController                    | 0                    | several (toggle)                                       | —                        | 0                                      | notification → `@Async`                          |

**Reading of the map.** Rows 1–4 and 10 — the browse/search surface — are **one ES query each (plus,
for favorites, one Postgres id-page), no per-hit DB/Redis, no external call.** That is the whole
ballgame for read latency and the shape is correct. The remaining work on these rows is _inside_ the
single ES query (F1/F3/F4), not in the app's call graph.

**Correction vs v1:** row 2 (detail) does **two** PK reads, not one — `numberOfFavorites` **and**
`numberOfViews` are each read live from Postgres on a detail view (D8), plus a Redis GET for the view
delta. Still all PK-scoped, still single-row, but it is two DB touches, not one.

---

## 2. The Elasticsearch path in detail + deeper findings (the read-path wins live here)

**Query shape — confirmed.** `DefaultProductsSearchService` is the only ES caller; it issues a single
`elasticsearchOperations.search(...)` per request (listing/autocomplete) or `elasticsearchOperations
.count(...)` (count). The query is assembled by `DefaultProductsFilterQueryBuilder` from
filter/category/price/region/text/base-site generators — pure in-memory construction, no IO. Listing
and detail differ only in the query; both return `SearchHits<ProductDocument>`.

**Per-result mapping — confirmed pure in-memory.** `DefaultProductsSearchFacade.getResponseData` maps
each hit via a reused singleton `ModelMapper` (`ProductOverviewConverter`). Name resolves from
`nameTranslations` already embedded in the doc; the `condition` filter label resolves via
`DefaultReferenceProductFilterService` — a **pure in-memory stream** over `filterReferences` already in
the doc; the top image is the first `imageKey` in memory. **No DB, no Redis, no ES per hit.** The
listing path does **zero per-hit backend round-trips.** `ProductOverviewDTO.isFavorite` exists but is
**never populated** on this path (client derives favorites from row 11 and marks locally — adjacent
note, not a perf issue).

**Detail path** (`ProductDetailsConverter`) does its enrichments by PK / from cache:
`productService.getNumberOfFavorites(id)` is `SELECT p.numberOfFavorites FROM Product p WHERE p.id = ?`
(`ProductRepository:30`) — a denormalized counter by PK; base-site overview is served from the no-TTL
`redisBaseSiteOverviews` cache. (Plus the `numberOfViews` read, D8.)

So the **structure** is as clean as v1 said. The **tuning**, which v1 did not inspect, is where the
wins are:

### F1 — ES fetches the full `_source` for every hit; no source filtering. **Severity: HIGH (the top read-path win)**

`buildQuery` builds a `NativeQuery` with **no** `withSourceFilter` / `withFields`
(`DefaultProductsFilterQueryBuilder.java:84-92`; grep confirms `withSourceFilter` appears **nowhere**
in the codebase). Every search returns the entire `ProductDocument` `_source`, which is fat: two
`List<TranslationReference>` — `nameTranslations` **and** `descriptionTranslations` (the latter is the
full multi-language long-description text), the full `imageKeys` set, and the entire nested
`filterReferences` list. The listing converter reads only one name translation, the first image key,
price/currency/city-key, the condition filter, and the base site. **`descriptionTranslations` is
shipped ES→app on every listing hit and never used.**

**Implementation suggestion.** Add a source filter to the **listing and autocomplete** query only
(the detail path legitimately needs description + all images, so leave it on full `_source`):

```java
// in buildQuery (and the autocomplete query), include only fields the overview converter reads:
.withSourceFilter(FetchSourceFilter.of(
    new String[] { "id", "catalogRoute", "nameTranslations", "imageKeys",
                   "price", "currencyCode", "free", "cityLabelKey",
                   "filterReferences", "baseSite", "productState" /* + state fields actually read */ },
    null))
```

or, lower-risk, an **exclude** of just the heavy unused field:
`FetchSourceFilter.of(null, new String[] { "descriptionTranslations" })`. Either cuts the ES→app
payload and the per-hit Jackson deserialization on the hottest path. **Verify the include-list against
`ProductOverviewConverter` field-by-field before shipping** so nothing read goes missing — the
exclude-only variant is the safe first step.

### F3 — `track_total_hits=true` forces an exact full count on every search. **Severity: MED**

All three query builders set `.withTrackTotalHits(true)` (`DefaultProductsFilterQueryBuilder.java:45,
72,88` — verified). This makes ES count **every** matching document exactly for `getTotalHits()`,
which is the most expensive part of a broad match-all on a large index. The grid rarely needs an exact
total past the first few pages.

**Implementation suggestion.** Bound it on the listing path
(`withTrackTotalHitsUpTo(10000)` / the int overload), and **disable it for autocomplete**
(`withTrackTotalHits(false)`) since autocomplete discards the total entirely. Confirm the web grid can
render "10,000+" rather than an exact count past the cap — a one-line frontend contract check (flag to
Mastermind for the web seam, do not change web here).

### F4 — Autocomplete runs the full bool search + full `_source` to return three fields. **Severity: MED**

Autocomplete uses the **identical** `getProductsFor`/`buildQuery` path as the full listing, so it pays
full bool-query construction, full `_source` fetch (F1), and exact total counting (F3), only to map
~10 docs into `{id, name, catalogRoute}`.

**Implementation suggestion.** Give autocomplete a dedicated lightweight query:
`withSourceFilter` limited to `id`/`catalogRoute`/`nameTranslations`, `withTrackTotalHits(false)`, a
small fixed size. Longer term a `search_as_you_type` or `completion` field on the per-language name
keyword is markedly cheaper than the analyzed bool query — but that's an index-mapping change (reindex
required), so stage it as a follow-up, not a first cut.

### F2 — Deep-pagination `from+size` blowup is unguarded. **Severity: MED**

`buildQuery` uses `withPageable(getPagingSafe())` — plain from+size paging. `PagingDTO` caps `perPage`
at 100 but **does not cap `page`**. A client requesting `page=100000&perPage=100` asks ES for
`from=10,000,000`, which hits the default `index.max_result_window` (10,000) — there is **no**
`max_result_window` override in the index settings (grep found none), so today this returns an ES
**error**, wasting an expensive request and giving a broken UX rather than a graceful empty page.

**Implementation suggestion.** Clamp `from + size` against a `MAX_RESULT_WINDOW` constant in
`buildQuery` (return the last valid page or empty when `page * perPage >= 10000`), or add a `page`
upper bound in `PagingDTO`. If genuine deep paging is ever needed, move that path to `search_after`
keyed on the sort field + `id`. The clamp is the cheap, correct first step.

### F5 / F9 — Per-hit `ModelMapper` reflection on nested types. **Severity: LOW-MED**

The `documentMapper` is built once and reused (good — not rebuilt per request), and the top-level
`ProductDocument → DTO` mappings use hand-written `Converter`s. But two nested mappings still go
through ModelMapper's reflection **per hit**: `documentMapper.map(source.getBaseSite(),
BaseSiteOverviewDTO.class)` in `ProductOverviewConverter` (per row) and
`documentMapper.map(ref, ProductFilterDTO.class)` per filter ref in
`DefaultReferenceProductFilterService`. The matching strategy is left at the default STANDARD
(fuzzy) matcher.

**Implementation suggestion.** Two options, either acceptable: (a) hand-map
`BaseSiteReference → BaseSiteOverviewDTO` and `ProductFilterReference → ProductFilterDTO` directly in
the converters (removes reflection from the hot loop); or (b) set
`mapper.getConfiguration().setMatchingStrategy(MatchingStrategies.STRICT)` to make TypeMap resolution
cheaper and unambiguous. Since a listing page is almost always **one** base site (filtered by
`baseSiteId`), F9's specific win is to resolve the `BaseSiteOverviewDTO` **once per page** and reuse it
across hits rather than re-mapping per row. Measure first — this is per-hit CPU, unmeasured, and only
matters at large page sizes; it is the lowest-priority ES item.

### F6 / F7 — Confirmed clean, no action.

Sort is keyword/numeric-field based (validated against the `OrderType` allowlist; random ordering uses
`function_score` `random_score` on the `id` long) — **no fielddata loading, no expensive script
sort.** Count uses the dedicated ES `_count` API (not a `size=0` search) and is HTTP-cached for the
edge. Keep the `isSortableField` allowlist tight to keyword/numeric fields.

### F8 — `indexOne` double-refreshes per single-product write. **Severity: LOW (write path, read-impacting)**

`indexOne` uses `withRefreshPolicy(IMMEDIATE).index(...)` **and then** an explicit
`indexOps(...).refresh()` — a redundant second refresh. Frequent immediate refreshes churn Lucene
segments and degrade query-side performance under write load.

**Implementation suggestion.** Drop the explicit `.refresh()` (IMMEDIATE already refreshes); consider
`WAIT_UNTIL` instead of `IMMEDIATE` if the caller doesn't need read-your-write in the same request.

**Verdict for §2:** the call graph is clean; the ES query is **not yet tuned**. F1 (source filtering)
is the single highest-value read-path change available anywhere in the backend, and F3/F4 are cheap
companions on the same path.

---

## 3. External / network calls on request paths — the timeout audit (revised, more findings than v1)

Ranked by how much a real user can wait on them. **The critical re-check finding:** four of the five
synchronous external clients have **no timeout**, two more than v1 confirmed. None is on the browse
path; all are robustness fixes; but "unbounded hang pinning a request thread" is a real production
failure mode for each.

| Rank     | Call                    | File:line (verified)                                          | Endpoint that blocks                                                  | Timeout                          | Note                                                                                                   |
| -------- | ----------------------- | ------------------------------------------------------------- | -------------------------------------------------------------------- | -------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **High** | reCAPTCHA verify        | `DefaultReCaptchaService.java:13` (`new RestTemplate()`)      | captcha-gated public path (registration-adjacent)                    | **NONE** (verified)              | **Upgraded from v1's "ambiguous."** Field-init bare `RestTemplate`, default infinite connect+read. Public, unauthenticated. A hang ≠ the caught `RestClientException`. |
| **High** | SMTP send               | `DefaultEmailService.java:69` + **no yaml timeout** (verified) | `resend-verification`, `request-password-reset`, banned-reblock      | **NONE** (verified)              | **New vs v1.** No `mail.smtp.connectiontimeout/timeout/writetimeout` in prod/stage/dev yaml → JavaMail defaults to infinite. Synchronous on resend/reset. |
| **High** | OpenAI completion       | `DefaultOpenAIService.java:42` (`HttpClients.createDefault()`) | `POST /api/secure/openai` (AI description)                            | **NONE** (verified)              | Unbounded hang holds a Tomcat worker. Authenticated, one feature.                                     |
| **High** | Firebase email-link gen | `VerificationEmailSender.java:75` / `PasswordResetEmailSender.java:81` | `resend-verification`, `request-password-reset`              | Firebase default                 | A real Firebase network round-trip (link minting), on the same synchronous path as SMTP above.        |
| **Med**  | R2 / S3 op              | `R2Config.java:29` (`UrlConnectionHttpClient.builder().build()`, no override) | product/profile/orphan-image delete                       | **NONE** (SDK default)           | No `apiCallTimeout`/`socketTimeout`. **But NOT inside any transaction** (see §6) — risk is a pinned thread, not a pinned DB connection. |
| **Med**  | Cloudflare KV toggle    | `DefaultCloudflareKvService.java:28` (legacy `new RestTemplate()`) | admin maintenance toggle only                                    | **NONE**                         | Already logged in `issues.md` (2026-06-03). Admin-only. Auto-trip path already uses the bounded bean. |
| **Low**  | Web revalidation        | `DefaultWebRevalidationService.java:52`                       | product/user mutations (AFTER_COMMIT)                                | 3s/3s (bounded bean)             | Best-effort, errors swallowed, post-commit. Fine.                                                     |

**Implementation suggestions (the four real fixes, in ROI order):**

1. **reCAPTCHA (one line, highest ROI).** Delete the field-initialized `new RestTemplate()` and inject
   the existing bounded bean: `@Qualifier("restTemplate") RestTemplate rest` (3s/3s, from
   `ApplicationConfig.java`). Removes an unbounded hang on a public endpoint.
2. **SMTP (config-only, all three yamls).** Add under `spring.mail.properties.mail.smtp`:
   `connectiontimeout: 5000`, `timeout: 5000`, `writetimeout: 5000`. This also bounds the **async**
   senders, which otherwise occupy a virtual thread indefinitely on a hung relay.
3. **OpenAI.** Replace `HttpClients.createDefault()` with a shared client carrying
   `RequestConfig` connect ≈3s / response ≈20s (a sane budget for the description model), built once as
   a bean rather than per call. Consider moving the suggestion off the request thread later; bounding
   it is the immediate fix.
4. **R2/S3.** Add `.overrideConfiguration(o -> o.apiCallTimeout(Duration.ofSeconds(30))
   .apiCallAttemptTimeout(Duration.ofSeconds(10)))` and give `UrlConnectionHttpClient` explicit
   `connectionTimeout`/`socketTimeout`. Bounds the hang on the image-orphan and profile-image request
   threads (and the offloaded delete threads).

These four share one shape — **a missing client timeout** — and could be one small "timeout hardening"
brief. The legacy KV one is already logged and admin-only; fold it in or leave it per Mastermind.

### 3a. Firebase `verifyIdToken` is LOCAL — confirmed, plus two new auth-path findings

`DefaultFirebaseAuthService.java:62` calls the **single-arg** `verifyIdToken(idToken)` — local RSA
verification against the Admin SDK's cached Google public keys, **not** a per-request network call (the
`(token, true)` check-revoked overload is used nowhere). Confirmed. Every authenticated request pays:
RSA verify (CPU, sub-ms) + **one** Redis GET (`redisUserAuth` hit; read exactly once per request at
`FirebaseAuthFilter.java:68`, no double-read). Miss = one Postgres JOIN-FETCH constructor projection
(`UserRepository.findAuthDataByFirebaseUid`), 30-min TTL, `@CacheEvict` on `saveUser`. Login
(`firebase-sync`) verifies the token **twice** (filter + `getOrCreateUser`) — still two **local**
verifies, not two network calls; login latency is DB-bound. **Auth is cheap and local — do not add a
"cache the verification" layer; there is nothing remote to cache.**

Two new auth-path findings the re-check surfaced:

- **D5 — `String.intern()` per Firebase UID (latent contention/memory). Severity: MED.**
  `createUserSynchronized` does `synchronized (token.getUid().intern())`
  (`DefaultFirebaseAuthService.java:124`) to serialize concurrent first-logins of the **same** UID.
  The lock scope is correct (per-UID), but `intern()` pushes every distinct UID into the JVM's shared
  string-pool table — which grows with unique-user volume and adds global lock contention on the
  intern table, all for a per-user mutex. **Implementation suggestion:** replace with a bounded striped
  lock — Guava `Striped<Lock> locks = Striped.lock(64)` then `locks.get(uid).lock()`, or a
  `ConcurrentHashMap<String,Object>` keyed mutex, or `Interners.newWeakInterner()` so entries are
  GC-eligible. The in-block re-check + DB unique constraint is the real correctness guard (and the lock
  is process-local anyway, so it does not prevent cross-instance duplicates) — a bounded lock loses
  nothing.
- **D6 — the auth filter runs the full verify+GET on `/api/public/**` too. Severity: LOW-MED.**
  `FirebaseAuthFilter` has **no `shouldNotFilter` override** (contrast `InternalTokenFilter`, which
  does); its only skips are OPTIONS and the `firebase-sync` substring. A logged-in user browsing public
  listings sends their Bearer token, and the filter pays a full RSA verify + Redis GET the permit-all
  rule doesn't require. **This may be intentional** — public endpoints like `DefaultProductSeenService`
  read `getCurrentUserId` to personalize. **Implementation suggestion:** confirm whether any public
  endpoint actually consumes the auth context; if yes, keep it; if no, gate the expensive path so
  token-bearing public requests skip verify. This is a product decision, not a clear-cut fix — flag,
  don't change blind.

### 3b. Correctly offloaded (not on any request thread) — confirmed good

Push (Firebase/Expo) `@Async`; Firestore notification writes `@Async`; registration/lifecycle emails
`@Async` AFTER_COMMIT listeners (`@Retryable`); currency-rate fetch `@Scheduled` daily **with the
bounded RestTemplate**; product-view dedup+increment on a **virtual thread**
(`DefaultProductSeenService:63`). The `@Async` executor is `newVirtualThreadPerTaskExecutor()`
(`AsyncConfig`) — uncapped virtual threads, the right call for IO-bound work (no unbounded-queue
failure mode), **but** because the email/push clients lack timeouts (above), a provider outage grows
pinned virtual threads without bound — another reason the SMTP/Firebase timeout fix matters. **No
`@Async` + `@Transactional` proxy self-invocation traps were found** (all such pairs are reached
through the Spring proxy / event dispatcher).

**Verdict for §3:** the external-call latency tail is **off the hot read path**, but the timeout
posture is worse than v1 reported — four unbounded clients, two of them (reCAPTCHA, SMTP) on
unauthenticated user-facing endpoints. The fixes are small and share one shape.

---

## 4. Caching reality + deeper findings

**Reference data — no-TTL, warmed at boot** (`CacheWarmupService`, `ApplicationReadyEvent`):
`redisBaseSite*`, `redisActiveBaseSiteCodes`, `redisLanguages`/`redisLanguage`, `redisBaseCurrency`,
`redisTranslations`. Warmup gates Actuator readiness REFUSING→ACCEPTING in a `finally`, so the app
doesn't take traffic until these are populated. Rebuilt on checksum change by `VersionChecksumService`.
Per-user caches (`redisUserAuth`, `redisUserInfo`) are not warmed (can't be) — they populate on first
hit, 30-min TTL, evicted on save. **Confirmed mature.**

**`redisUserAuth` — the one cache on the truly hot path.** Read once on every secured (and
token-bearing public) request; miss = one JOIN-FETCH projection; freshness owned by `@CacheEvict`,
TTL a backstop. Correctly shaped.

**View counter / dedup** writes Redis deltas (24h TTL) flushed to Postgres on a ~20s schedule; reads
sum the PK counter + Redis delta. Keeps per-view writes off Postgres. Correct design — with one
hot-spot in the **flush** (D4 below).

New findings from the re-check:

- **D2 — serializer is Jackson JSON, not JDK (v1 worry retracted). Severity: none.**
  `RedisConfig.java:109` uses `JacksonJsonRedisSerializer`. No JDK `SerializationPair` anywhere. There
  is **no** slow-serializer concern. (Micro-note: a fresh `ObjectMapper` per cache config at boot —
  negligible, boot-time only.)
- **D4 — the view-counter flush does an N+1 sequential Redis sweep. Severity: MED (scales with
  traffic).** Lettuce runs a single shared connection (no `commons-pool2`, no pool config), so multi-key
  ops are sequential blocking round-trips. `DefaultRedisViewCounterService.getTrackedProductIds()` does
  one `SMEMBERS` then a **per-member `EXISTS`** in a stream (N+1); `incrementViewDeltaBy` issues 3
  commands per view (`INCRBY`+`EXPIRE`+`SADD`); `getAndResetDelta` up to 4 per product inside the flush
  loop. **Implementation suggestion:** (a) replace the EXISTS-sweep with a single `SMEMBERS` + `MGET`
  of the delta keys (a `null` from MGET ⇒ expired ⇒ remove), or a small Lua script; (b) pipeline the
  per-view increment via `executePipelined`; (c) set the delta-key TTL only on creation (when `INCRBY`
  returns == delta) instead of on every increment, dropping one command per view. This is the biggest
  scaling win on the view hot path.
- **D3 — profile-self read hits Redis twice. Severity: LOW.** `redisUserInfo` (profile fields) is a
  separate cache from `redisUserAuth` (auth fields); a request to the user's own profile pays the
  filter's `redisUserAuth` GET **plus** a `redisUserInfo` GET, and `redisUserInfo` is double-keyed
  (id + firebaseUid) so a save evicts two keys. The split is intentional (auth vs profile fields) and
  probably correct. **Suggestion:** only if profile-self reads turn out hot, serve the identity-
  overlapping fields from the already-loaded `OglasinoAuthentication` instead of a second Redis fetch.
- **D7 — cache stampede on a cold `redisUserAuth` key. Severity: LOW-MED for a viral user.** Spring's
  `RedisCache` does not lock misses, so concurrent requests for the same just-evicted active user each
  run the JOIN-FETCH projection until the first write lands. Reference caches are immune (warmed before
  traffic). **Suggestion:** if it shows in metrics (`enableStatistics()`), add request-coalescing /
  cache-writer locking for the user caches. Note the **structural** version of this (reference-data
  concurrent-miss pool exhaustion) is explicitly owned by `connection-pool-hardening` — do not solve it
  here.
- **D8 — product-detail view does a second live DB read for the base view count. Severity: LOW-MED
  (hot path).** `DefaultProductSeenService.getNumberOfViews` reads the DB base count + a Redis delta on
  every detail view; combined with the `numberOfFavorites` read this is **two** PK DB reads on the
  detail path. **Suggestion:** fold the view base count into whatever product projection the detail
  facade already loads, so a detail view is 1 product query + 1 Redis GET rather than 2 separate DB
  reads + 1 Redis GET. Confirm against the detail facade before acting; cross-references the JPA-fetch
  boundary, so coordinate with `jpa-fetch-tuning` rather than duplicating.

**Not worth caching:** `getNumberOfFavorites` on detail (single PK read; caching saves little, risks
staleness — leave it). The `count` endpoint hits ES `_count` live with a `cacheControl` header so the
edge absorbs it — fine.

**Verdict for §4:** caching is mature. The only real win is D4 (pipeline/Lua the view-counter flush);
D8 is a coordinate-with-JPA item; the rest are leave-as-is unless metrics say otherwise.

---

## 5. Payload size

- **Listing (`ProductOverviewDTO`)** is trimmed to card fields: single resolved `name`, single
  `topImageKey`, price/currency/city-key, condition. The fat source fields are collapsed during
  mapping and **not serialized to the client.** Good. **But note F1:** the trimming happens in the app
  _after_ ES has already shipped the full `_source` over the wire — the client payload is small, the
  **ES→app** payload is not. Source filtering (F1) closes that gap.
- **Detail (`ProductDetailsDTO`)** adds description (single locale), full `imageKeys` (keys, not bytes
  — direct-to-R2, no base64 blobs), category ids, filter lists. Reasonable for a detail page.
- **Reviews** page size fixed at 20 — bounded.
- No unbounded collection, no base64 image payload, no entity dump on any hot endpoint.

**Verdict for §5:** client response shapes are appropriately narrow. The only payload issue is the
unfiltered ES→app `_source` (F1), not the client wire shape.

---

## 6. Blocking / contention outside JPA — including the resolved R2-in-transaction question

- **No N+1 ES queries** — every multi-product path issues **one** ES query (listing `buildQuery`,
  favorites `buildMultiProductIdQuery`); detail is one query. Confirmed.
- **No N+1 Redis gets on listing** — mapping is in-memory. (The N+1 Redis sweep is on the view-counter
  **flush**, D4, not on a request path.)
- **No N+1 external calls** on read paths.
- **R2 delete inside an open DB transaction? — RESOLVED: NO (v1's speculative finding closed).** Traced
  every delete flow:
  - **User hard-delete** — R2 is explicitly **outside** the tx: `DefaultUserDeletionService` uses a
    `TransactionTemplate` that commits the DB work first, then runs R2 post-commit, `bestEffort`-wrapped.
  - **Product delete** — `DefaultProductService.deleteProduct` offloads R2 to a **virtual thread**
    (`Thread.ofVirtual().start(...)`) before `productRepository.delete`. Not in the tx.
  - **Image-orphan DELETE** — `DefaultImageDeletionFacade` is **not** `@Transactional`; R2 delete runs
    on the request thread with no DB tx held.
  - **Profile-image change** — `DefaultUserFacade.updateCurrentUserData` is **not** `@Transactional`;
    the R2 delete runs (best-effort) **before** the short `saveUser` tx opens.
  **No R2 op runs inside a Hikari-connection-holding window.** The residual risk is purely the **missing
  SDK timeout** (§3, MED) — a hung R2 pins a Tomcat or virtual thread, not a DB connection. Emails,
  push, and web-revalidation are all AFTER_COMMIT/async, never inside a tx, confirmed.
- **`synchronized (uid.intern())`** serializes same-UID first-logins only — per-user, not global; benign
  beyond the `intern()` concern (D5).

**Verdict for §6:** no contention or fan-out outside the JPA bucket. The one genuinely-open v1 question
is now answered and **closed**.

---

## 7. Pools / connection setup (verified numbers)

| Env   | Hikari max-pool                          | Tomcat max-threads                      | Ratio |
| ----- | ---------------------------------------- | --------------------------------------- | ----- |
| stage | **8** (`application-stage.yaml:18`)      | **50** (`:113`)                         | ~6:1  |
| prod  | **18** (`application-prod.yaml:15`)      | **100** (`:107`)                        | ~5.5:1 |
| dev   | **20** (`application-dev.yaml:18`)       | **200** (implicit Boot default, not set) | ~10:1 |

The stage/prod ratios are deliberate (comments cite small-droplet + Postgres connection-limit
headroom — prod yaml notes a "25 conn limit"). **Correction vs v1:** dev's 200 Tomcat threads is the
framework **default**, not an explicit setting. Because the hot path is ES-served, Postgres pressure
comes mainly from auth-cache misses, reviews, favorites, and the view flush — not browse — so the tight
Hikari pool is unlikely to be the browse bottleneck, but it **is** the pool most likely to saturate
first under a write/auth burst, and the relevant shape is precisely where the synchronous external
calls (§3) back up: if OpenAI/SMTP/reCAPTCHA hang, request threads pile up against a small connection
pool. **This is why the timeout fixes (§3) are also pool-protection.** Redis (Lettuce, single
multiplexed connection, no pool) and the ES client use env-driven timeouts not fully visible in
committed yaml — confirm those against the deployed env. Virtual threads are **not** enabled globally;
only the view-counter path and the `@Async` executor use them explicitly. **The structural pool-
exhaustion fix (single-flight on cache miss) is owned by `connection-pool-hardening` — out of scope
here.**

---

## 8. Ranked findings (by likely impact on a user's wait, with the fix)

1. **(Informational, highest value) The hot read path's call graph is already optimal** — one ES query,
   in-memory mapping, no per-hit backend call. Browse latency lives in ES query time + the unfiltered
   `_source` payload, not in the app's call graph. The wins below are _inside_ the ES query.
2. **ES fetches the full `_source` per hit (F1, HIGH).** Add a source filter / `descriptionTranslations`
   exclude to the listing + autocomplete queries. The only app-layer browse-latency win that exists.
3. **Four unbounded external-client timeouts (§3, HIGH×3 + MED).** reCAPTCHA (public, one-line bean
   swap), SMTP (config in 3 yamls), OpenAI (shared client + RequestConfig), R2/S3 (SDK override). One
   "timeout hardening" brief. Robustness + pool protection, not browse latency.
4. **`track_total_hits=true` exact count on every search (F3/F4, MED).** Bound it on listing; disable
   on autocomplete; give autocomplete a dedicated light query.
5. **View-counter flush N+1 Redis sweep (D4, MED).** Pipeline / Lua the flush; set delta TTL on
   creation only. Scales with traffic.
6. **Deep-pagination `from+size` blowup unguarded (F2, MED).** Clamp `from+size` to the result window.
7. **`String.intern()` per Firebase UID (D5, MED, latent).** Replace with a striped lock.
8. **Product-detail does two PK DB reads (D8, LOW-MED).** Fold the view base count into the detail
   projection — coordinate with `jpa-fetch-tuning`.
9. **Auth filter verifies on public token-bearing requests (D6, LOW-MED).** Product decision: gate it
   if public endpoints don't consume the auth context.
10. **Per-hit nested ModelMapper + `indexOne` double-refresh (F5/F9/F8, LOW).** Hand-map nested types
    / resolve base site once per page; drop the redundant explicit refresh.

---

## 9. Probably already fine — don't "optimize" these

- **The ES listing call graph & in-memory mapping.** Clean. (The tuning wins are query settings, not
  the mapping layer — don't "batch" the mapping.)
- **`verifyIdToken` on every request.** Local crypto + one Redis GET. Not a Firebase round-trip. No
  "cache the verification" layer.
- **Redis serializer.** Jackson JSON, not JDK. No serializer change needed (v1 worry retracted).
- **`numberOfFavorites` live read on detail.** Single PK read; caching risks staleness for little gain.
- **Reference-data caches.** No-TTL, boot-warmed, readiness-gated, checksum-rebuilt. Mature.
- **The view-counter Redis-delta-then-flush _design_.** Correct (the win is in the flush _mechanics_,
  D4, not the design).
- **The `count` endpoint hitting ES `_count` live.** Edge-cached. Fine.
- **Sort fields.** Allowlisted keyword/numeric — no fielddata/script-sort risk.
- **`@Async` virtual-thread executor.** Right choice for IO work; no unbounded-queue trap.
- **Adjacent (not perf): `ProductOverviewDTO.isFavorite` never set on listing.** Client derives from
  the ids endpoint. Vestigial, not a perf issue.

---

## 10. What static reading CANNOT tell us here (be honest)

- **Actual ES query time** for listing/detail under real index size and filter complexity, and how much
  F1/F3 actually save — needs an ES `profile`/slow-log or APM span. The structure is known; the
  duration is not.
- **`redisUserAuth` hit-rate and miss cost under load** — depends on traffic shape and `saveUser`
  eviction churn. Needs a metric (`enableStatistics()`), not a read.
- **Redis and Elasticsearch client timeouts** — env-var driven, not in committed yaml. Unknown without
  the deployed env.
- **ModelMapper per-hit CPU at large page sizes** (F5) — plausibly non-trivial (reflection) but
  unmeasured; would show in a CPU profile of the search controller.
- **Whether any public endpoint actually consumes the auth context** (D6) — a runtime/usage question
  the static read narrows but does not settle; decides whether D6 is a fix or a no-op.

No numbers were invented anywhere above. Every "cheap"/"trivial" is a structural claim (PK read,
in-memory stream, cache hit, one ES query), not a timing.

---

## 11. Consolidated implementation suggestions (prioritized; nothing implemented here)

Grouped so each could become a self-contained brief. Effort/risk are rough.

| Group | Items | Effort | Risk | Owner / boundary |
| ----- | ----- | ------ | ---- | ---------------- |
| **A. Timeout hardening** | reCAPTCHA bean swap, SMTP yaml timeouts (×3), OpenAI shared client, R2/S3 SDK override; (optionally fold the already-logged KV one) | Low (config + one-line each) | Low | This overview — none claimed elsewhere except KV (logged). **OpenAI + R2-in-tx had no prior issue entry — genuinely new.** |
| **B. ES read-path tuning** | F1 source filter (start with `descriptionTranslations` exclude), F3 bound `track_total_hits`, F4 dedicated autocomplete query, F2 clamp deep paging | Low–Med | Low–Med (F3 needs a web-grid contract check; F1 needs an include-list review) | This overview. F4's `search_as_you_type` mapping is a reindex → stage as follow-up. |
| **C. Redis view-counter flush** | D4 pipeline/Lua the SMEMBERS+EXISTS sweep; TTL-on-create | Med | Low–Med | This overview. |
| **D. Auth-path** | D5 striped lock replacing `intern()`; D6 gate auth filter on public (decision first) | Low (D5) / decision (D6) | Low (D5) | This overview; D6 needs a product call. |
| **E. Coordinate-with-other-features** | D8 fold view base count into detail projection; D7 user-cache stampede; F5/F9 per-hit mapping | Low–Med | Low | D8/F-mapping touch JPA fetch → coordinate with `jpa-fetch-tuning`; D7's structural sibling is `connection-pool-hardening`. |

**Sequencing suggestion.** A (timeout hardening) is the cleanest standalone first brief — low risk,
real production-robustness value, and it doubles as pool protection. B (ES tuning) is the only group
that touches browse latency and is worth a measured before/after (ES slow-log). C and D are
independent and small. E should be folded into the features that already own the adjacent code, not
done here.

---

## For Mastermind (issues.md candidates — drafts only, not written by me)

Per conventions Part 3, draft-only; Docs/QA is the sole writer of `issues.md`. The context re-check
confirmed which of these are **already logged** and which are **genuinely new**:

1. **(candidate, HIGH — NEW) reCAPTCHA verify uses a no-timeout `new RestTemplate()` on a public
   endpoint.** `DefaultReCaptchaService.java:13`. Field-initialized bare `RestTemplate`, default
   infinite connect+read; a slow/unreachable Google siteverify hangs the request thread (a hang is not
   the caught `RestClientException`). One-line fix: inject the bounded bean. _Same class as the logged
   2026-06-03 KV no-timeout entry; not yet logged._

2. **(candidate, HIGH — NEW) SMTP has no timeout in any env yaml.** No
   `mail.smtp.connectiontimeout/timeout/writetimeout` in prod/stage/dev → JavaMail defaults to
   infinite. Synchronous on `resend-verification` / `request-password-reset`; also grows pinned virtual
   threads on the async senders during a relay outage. Config-only fix in three files.

3. **(candidate, HIGH — NEW) OpenAI HTTP client has no timeout.** `DefaultOpenAIService.java:42`
   (`HttpClients.createDefault()`); `POST /api/secure/openai` can hang the request thread (and a Tomcat
   slot) indefinitely. No prior `issues.md` entry exists for this. Mirror the bounded-timeout pattern.

4. **(candidate, MED — NEW) R2/S3 SDK client has no timeout override.** `R2Config.java:29`
   (`UrlConnectionHttpClient.builder().build()`, no `apiCallTimeout`/`socketTimeout`). **Note for the
   record: the v1 "R2 inside an open transaction" speculation is FALSE — every delete flow keeps R2 out
   of the tx (verified, §6); this is a thread-pin risk, not a DB-connection-pin risk.** No prior entry.

5. **(candidate, MED — NEW) ES listing/autocomplete fetch the full `_source` and force
   `track_total_hits=true`.** `DefaultProductsFilterQueryBuilder.java:45,72,88` + no `withSourceFilter`
   anywhere. The only app-layer browse-latency wins available. No prior entry.

6. **(candidate, MED — NEW) View-counter flush does an N+1 sequential Redis sweep.**
   `DefaultRedisViewCounterService.getTrackedProductIds()` (SMEMBERS + per-member EXISTS) and the 3–4
   commands-per-product flush loop. Pipeline/Lua candidate.

7. **(candidate, MED — NEW) `String.intern()` per Firebase UID in `createUserSynchronized`.**
   `DefaultFirebaseAuthService.java:124`. Latent string-pool growth + intern-table contention; replace
   with a striped lock.

8. **(already logged — no new entry) Cloudflare-KV legacy `new RestTemplate()` no-timeout.**
   `DefaultCloudflareKvService.java:28`. Logged 2026-06-03. Admin-only. Listed only so Mastermind sees
   it grouped with the timeout-hardening set.

**Cross-feature flags (Part 4b):**
- D8 (fold the detail view-count read into the product projection) and the F5/F9 per-hit mapping items
  touch JPA fetch / detail-projection code — recommend routing into `jpa-fetch-tuning` rather than a
  separate brief.
- D7 (user-cache concurrent-miss stampede) is the per-user sibling of the structural reference-data
  pool-exhaustion already owned by `connection-pool-hardening`.
- D6 (gate the auth filter on public token-bearing requests) needs a product decision — does any
  public endpoint consume `OglasinoAuthentication`? — before it's a fix rather than a no-op.

**Config-file impact:** none written. The four config files are untouched (read-only audit). The only
candidate edits are the `issues.md` drafts above, for Docs/QA to apply if Mastermind agrees. Items 1–7
have **no existing `issues.md` entry** (confirmed by full-file grep); item 8 is already logged.
