# Session summary — Audit: user-info projection subqueries & index coverage

**Repo:** oglasino-backend
**Branch:** dev (read-only; no checkout/commit/push)
**Date:** 2026-06-03
**Task:** Read-only audit of the user-info projection queries (the ones backing public profile / auth user loads) and whether their correlated subqueries are covered by database indexes.

Scope was READ-ONLY. No code changed. No spotless/test run (brief waived them). Every claim below was cross-checked with `grep`/`sed`/`cat` against the file it cites (per the repo's tool-fabrication hazard note). Where I could not verify something from source I say so explicitly.

---

## Q1 — The user-info projection queries and their correlated subqueries

Three candidate "user load" queries live in `UserRepository.java`:

| Query | Backs | Correlated subqueries? |
| --- | --- | --- |
| `findUserInfoById` (`UserRepository.java:140–169`) | public profile by id (`getUserInfoForId`) | **2** (see below) |
| `findUserInfoByFirebaseUid` (`UserRepository.java:172–204`) | own/profile by firebase uid (`getUserInfoForFirebaseUid`) | **2** (identical pair) |
| `findAuthDataByFirebaseUid` (`UserRepository.java:45–64`) | `FirebaseAuthFilter` auth load (`redisUserAuth`) | **0** — flat constructor projection, plain `JOIN u.preferredLanguage` + `LEFT JOIN u.baseSite`; **no correlated subquery**, so out of scope for index-coverage. Noted for completeness because the brief mentioned "auth user loads." |

The two `findUserInfo*` queries are **structurally identical** except for their outer `WHERE` (`u.id = :userId AND u.disabled = false` vs. `u.firebaseUid = :firebaseUid`). Each carries the same two correlated subqueries in its SELECT list:

**SQ1 — active-product count** (alias `activeProducts`):
```jpql
SELECT COUNT(p2.id)
FROM Product p2
WHERE p2.owner = u AND p2.productState = :activeState
```

**SQ2 — pending scheduled-deletion timestamp** (alias `scheduledDeletionAt`):
```jpql
SELECT req.scheduledDeletionAt
FROM UserDeletionRequest req
WHERE req.userId = u.id
  AND req.status = com.memento.tech.oglasino.entity.DeletionRequestStatus.PENDING
```

`:activeState` is bound to `ProductState.ACTIVE` at **both** call sites (`DefaultUserService.java:37` and `:47`) — it is never any other value today.

---

## Q2 — Each subquery's table and WHERE columns

| Subquery | Table (physical) | WHERE columns (physical) | Source of mapping |
| --- | --- | --- | --- |
| SQ1 (product count) | `product` | `owner_id` (FK, from `p2.owner = u`), `product_state` (`varchar(255)`) | `entity/Product.java:27-28` (`owner` → `@JoinColumn` `owner_id`), `:19-20` (`productState` → `product_state`); confirmed in schema `V1:354` (`owner_id`), `V1:360` (`product_state`) |
| SQ2 (deletion request) | `user_deletion_requests` | `user_id` (`bigint`), `status` (`varchar(20)`) | `entity/UserDeletionRequest.java` (`@Column(name="user_id")`, `@Column(name="status")`) |

`SELECT`ed (non-filter) columns for completeness: SQ1 returns `count(id)`; SQ2 returns `scheduled_deletion_at`.

---

## Q3 — Index coverage of each subquery's WHERE columns

### `product` indexes in V1 (`V1__init_schema.sql`)
- `idx_product_owner` — `product (owner_id)` — `V1:1153`
- `idx_product_category` — `product (category_id)` — `V1:1139`
- `idx_product_city` — `product (city_id)` — `V1:1146`
- `idx_product_site` — `product (base_site_id)` — `V1:1160`
- `idx_product_deactivated_by_system` — **partial** `product (owner_id, deactivated_by_system) WHERE deactivated_by_system = true` — `V1:1173`
- (+ PK on `id`)

There is **no** index on `product_state` and **no** composite `(owner_id, product_state)` — grep for `owner_id, product_state` / `product_state, owner` returns NONE.

**SQ1 verdict: PARTIALLY COVERED (leading column only).**
- `owner_id` → covered by `idx_product_owner`.
- `product_state` → **uncovered** (no index includes it).
- The planner can seek the user's product rows via `idx_product_owner`, then applies `product_state = 'ACTIVE'` as a residual (non-indexed) filter on those rows. The partial `idx_product_deactivated_by_system` also leads with `owner_id`, but its predicate (`deactivated_by_system = true`) excludes precisely the ACTIVE rows being counted, so it is not usable here.
- Practical blast radius is bounded by per-owner product cardinality (a user's own listings), not by table size — but it is the one subquery that is not fully index-served.

### `user_deletion_requests` indexes/constraints in V1
- `user_deletion_requests_user_id_key` — **UNIQUE (`user_id`)** → implicit unique btree — `V1:1951`
- `idx_udr_pending_due` — **partial** `user_deletion_requests (scheduled_deletion_at) WHERE status = 'PENDING'` — `V1:1982`
- (+ PK on `id`)

**SQ2 verdict: COVERED (single-row seek via the UNIQUE(`user_id`) btree).**
- `user_id` → covered by the implicit unique btree from `user_deletion_requests_user_id_key`. Because `user_id` is **UNIQUE**, this is an at-most-one-row lookup.
- `status` → applied as a residual filter on that single matched row. `idx_udr_pending_due` does **not** serve this lookup (its leading key is `scheduled_deletion_at`, not `user_id`; it indexes the SELECTed column, not the WHERE column), but it does not need to — the unique `user_id` seek already reduces to one row.
- No index work is warranted for SQ2: a composite `(user_id, status)` would only avoid touching the heap for the `status` check on a single row — negligible.

**Net:** SQ1 is the only subquery that is not fully index-covered.

---

## Q4 — Caching: where do these projections live, and how often does the uncovered subquery actually run?

Both `findUserInfo*` queries are wrapped by `@Cacheable(value = "redisUserInfo", …)`:
- `getUserInfoForFirebaseUid` — `key = "#firebaseUid"`, `unless = "#result == null"` (`DefaultUserService.java:34`)
- `getUserInfoForId` — `key = "#userId"`, `unless = "#result == null"` (`DefaultUserService.java:44`)

So `redisUserInfo` holds **two entries per user** (one id-keyed, one firebaseUid-keyed).

**Cache:** `redisUserInfo`, Redis-backed, value type `UserInfoDTO`, **TTL 30 min** ("backstop only" — `RedisConfig.java:44-51`). **Not warmed at startup** — `CacheWarmupService` skips it by design (`CacheWarmupService.java:23,75`), so the first profile view of any given user after a deploy/flush is a guaranteed miss.

**Eviction triggers (the events that force the uncovered SQ1 to re-run on next read):**
1. `DefaultUserService.saveUser` (`@Caching`, `:71-81`): evicts `redisUserInfo[#currentUser.id]`, `redisUserInfo[#currentUser.firebaseUid]`, and `redisUserAuth[#currentUser.firebaseUid]`. This is the main profile-mutation path.
2. `DefaultUserService.evictUserInfoCache(User)` (`@Caching`, `:108-117`): evicts `redisUserInfo[#user.id]` and `redisUserInfo[#user.firebaseUid]` (invalidate-only, no row write).
3. 30-minute TTL expiry (passive backstop).
4. Operator-driven manual eviction — `redisUserInfo` is exposed as `AdminCacheDescriptor.REDIS_USER_INFO` via the cache-admin path.

**Implication:** SQ1 (the uncovered count) runs only on a **cache miss** for a given user key — i.e., once per cache-fill after a profile save / admin action / TTL expiry / cold start — **not on every request**. So the missing `product_state` index is an optimization with a bounded blast radius, not a hot-path fix. (`redisUserAuth`, the auth path, has no correlated subquery and is irrelevant to this concern.)

---

## Q5 — How this repo handles pre-production schema changes — **CONTRADICTION, flagged**

The brief asks me to confirm the convention and quote it. I found **two governing sources that directly contradict each other.** I am not resolving this from the code; I report both verbatim and flag it for Mastermind / Docs/QA.

**Source A — `conventions.md` (the governing rulebook), pre-launch schema section:**
> "During the pre-launch period, schema additions and renames edit `V1__init_schema.sql` in place rather than creating new `V2__*.sql` migration files. This applies to all tables, columns, indexes, and constraints touched by features under active development. … This convention ends at the first production deploy. From that point onward, all schema changes go in new append-only Flyway migrations as normal." (`../oglasino-docs/meta/conventions.md:542-544`)

**Source B — the V1 schema file's own header comment:**
> "User deletion feature schema integrated 2026-05-17 per pre-production decision; **future schema changes go in V{n+1} migrations as normal.**" (`V1__init_schema.sql:9-11`)
> "**Do NOT edit this file after it has been pushed. Any schema change goes in a new V{n}__{desc}.sql file** (Hibernate runs in `validate` mode)." (`V1__init_schema.sql:19-20`)

Source A says: pre-launch index additions **edit V1 in place**. Source B says: **never edit V1**, add `V{n+1}`. Project state is still pre-production (no prod deploy yet per project memory), so under Source A this index would be an in-place V1 edit; under Source B it would be a new `V2__*.sql`. **This needs to be reconciled before anyone adds the index below.** See "For Mastermind."

**Precedent — existing partial index in V1** (matches the shape SQ1 would need):
```sql
CREATE INDEX idx_product_deactivated_by_system
    ON public.product (owner_id, deactivated_by_system)
    WHERE deactivated_by_system = true;
```
(`V1:1173-1176`.) Two more partial precedents using a string-literal predicate: `idx_udr_pending_due … WHERE ((status)::text = 'PENDING'::text)` (`V1:1982-1984`) and `idx_users_deletion_status … WHERE ((deletion_status)::text <> 'ACTIVE'::text)` (`V1:1957-1959`).

**Precedent — existing composite index in V1:** `idx_translation_namespace_key ON public.oglasino_translation (translation_namespace, translation_key)` (`V1:1259`); also `uq_base_site_allowed_languages ON base_site_allowed_languages(base_site_id, language_id)` (`V1:1791`).

---

## Q6 — Smallest index to cover SQ1, and IMMUTABLE-safety

Two viable shapes; both fully serve `WHERE owner_id = ? AND product_state = 'ACTIVE'`:

**(A) Partial — smallest (recommended shape):**
```sql
CREATE INDEX idx_product_owner_active
    ON public.product (owner_id)
    WHERE product_state = 'ACTIVE';
```
Smallest on disk: it indexes **only** ACTIVE rows, and `owner_id` alone is enough to locate-and-count them once the partial predicate pins state. Matches the repo's established partial-index convention (`idx_product_deactivated_by_system`, `idx_udr_pending_due`, `idx_users_deletion_status`).
- **Caveat:** usable **only** while the query always binds `ACTIVE`. Both current call sites pass `ProductState.ACTIVE` (`DefaultUserService.java:37,47`), so this holds today. If a future caller passes `INACTIVE`/`DELETED`, the partial cannot serve it and the planner falls back to `idx_product_owner` + residual filter.

**(B) Composite — general alternative:**
```sql
CREATE INDEX idx_product_owner_state
    ON public.product (owner_id, product_state);
```
Covers any `product_state` value and can satisfy the COUNT as an index-only/index-range scan. Larger (indexes every product row). Choose this only if the count is expected to be parameterised by state in future.

**IMMUTABLE-safety (partial predicate):** `product_state = 'ACTIVE'` compares a column to a **string literal** — no `now()`/`CURRENT_TIMESTAMP`/volatile or stable function. PostgreSQL requires partial-index predicates to reference only IMMUTABLE functions; column-to-constant equality on a `varchar` qualifies. It is the same predicate form already accepted in V1 (`(status)::text = 'PENDING'::text`, `(deletion_status)::text <> 'ACTIVE'::text`). **IMMUTABLE-safe: yes.**

**Priority note:** because SQ1 runs only on a `redisUserInfo` miss (Q4) and is bounded by per-owner product count, this is a low-urgency optimization, not a hot-path fix.

---

## Files inspected (read-only)

- `src/main/java/com/memento/tech/oglasino/repository/UserRepository.java`
- `src/main/java/com/memento/tech/oglasino/repository/projections/UserInfoProjection.java`
- `src/main/java/com/memento/tech/oglasino/entity/Product.java`, `entity/UserDeletionRequest.java`
- `src/main/java/com/memento/tech/oglasino/service/impl/DefaultUserService.java`
- `src/main/java/com/memento/tech/oglasino/config/RedisConfig.java`, `config/CacheWarmupService.java`, `config/RedisCacheEvictConfig.java`
- `src/main/resources/db/migration/V1__init_schema.sql`
- `../oglasino-docs/meta/conventions.md`

## Tests

- Ran: none (read-only audit; brief explicitly waived spotless/test).

## Cleanup performed

- None needed (no code changed).

## Config-file impact

- conventions.md: **no change by me**, but a contradiction with the V1 header is flagged for Docs/QA — see "For Mastermind."
- decisions.md: no change.
- state.md: no change.
- issues.md: no change by me; one candidate entry drafted in "For Mastermind" (the conventions/V1 contradiction) for Docs/QA to author if it agrees.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code changed.
- Part 4a (simplicity): N/A — no code changed; see "For Mastermind" for the no-change evidence block.
- Part 4b (adjacent observations): one adjacent observation surfaced — the conventions.md ↔ V1-header contradiction on pre-launch schema changes (Q5). Flagged, not acted on.
- Part 11 (trust boundaries): N/A — read-only audit, no trust decision touched.
- Other parts touched: none.

## Known gaps / TODOs

- I did not run `EXPLAIN`/`EXPLAIN ANALYZE` against a live DB (read-only audit, no local DB session in scope). Coverage verdicts are derived from the schema DDL and query text, not from a planner trace. If Mastermind wants empirical confirmation, that is a separate follow-up.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (no code changed).
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Pre-launch schema convention contradicts the V1 header (blocker for any index work).** `conventions.md:542-544` says pre-launch index additions **edit `V1__init_schema.sql` in place**; `V1__init_schema.sql:19-20` says **never edit V1, add a new `V{n}__{desc}.sql`**. These cannot both be followed. Whoever implements the SQ1 index needs an authoritative answer first. My read of project state is that prod is not yet deployed, which would put us under the conventions.md "edit V1 in place" regime — but the schema file's own instruction is the opposite, so I will not assume. **Drafted issues.md entry (for Docs/QA to author if it agrees):** *"2026-06-03 — severity: low — status: open — `conventions.md` pre-launch-schema rule (edit V1 in place pre-launch) contradicts the `V1__init_schema.sql` header (never edit V1; add V{n+1}). Reconcile before the next pre-launch schema change."*

- **Audit conclusion in one line:** of the two correlated subqueries in the user-info projection, only **SQ1 (active-product count) is not fully index-covered** — `owner_id` is indexed, `product_state` is not. SQ2 is already served single-row by `UNIQUE(user_id)`. Smallest fix is a partial index `product (owner_id) WHERE product_state = 'ACTIVE'` (IMMUTABLE-safe), but it is low-urgency because the subquery only runs on a `redisUserInfo` cache miss.

- **Suggested next step:** if the index is approved, the implementing brief should (a) state which file the DDL goes in (resolving the Q5 contradiction), and (b) decide partial vs. composite given the current "ACTIVE-only" call pattern. I can implement either on instruction.
