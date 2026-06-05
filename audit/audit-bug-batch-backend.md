# Audit — Bug Batch (Backend)

Repo: `oglasino-backend` · Branch: `dev` · **READ-ONLY** — no code changed.
Date: 2026-06-04.

Each finding carries `file:line` evidence. Seed values are from
`src/main/resources/data/configuration/data-configuration.sql`.

---

## Q1 — B1: config default-0 getters

**Safe overload exists.** `ConfigurationService` declares the missing-key-safe
overloads:

- `int getIntConfig(String configKey, int defaultValue)` — `ConfigurationService.java:45`
- `long getLongConfig(String configKey, long defaultValue)` — `ConfigurationService.java:48`
- `double getDoubleConfig(String configKey, double defaultValue)` — `ConfigurationService.java:51`

Implementation: blank → `defaultValue`; unparseable → `defaultValue` + warn; present
value (incl. `0`) returned as-is — `DefaultConfigurationService.java:118-170`.

The **single-arg** getters fall back to `"0"` and parse that, i.e. a missing/blank
key silently yields `0`:

- `getLongConfig(key)` → `Long.parseLong(getConfig(key, "0"))` — `DefaultConfigurationService.java:104-106`
- `getIntConfig(key)` → `Integer.parseInt(getConfig(key, "0"))` — `DefaultConfigurationService.java:114-116`

### Per call

| Caller | Key | Getter used | Default-0 risk | Current seed value |
|---|---|---|---|---|
| `DefaultRedisViewCounterService.java:34` (`incrementViewDeltaBy`) | `redis.product.view.delta.ttl` | `getLongConfig(key)` — single-arg | **yes** | `'86400000'` (24h) — `data-configuration.sql:5` |
| `DefaultRedisViewCounterService.java:75` (`getAndResetDelta`) | `redis.product.view.delta.ttl` | `getLongConfig(key)` — single-arg | **yes** | `'86400000'` — `data-configuration.sql:5` |
| `DefaultProductSeenService.java:43` (`handleProductView`) | `redis.product.view.owner.ttl` | `getLongConfig(key)` — single-arg | **yes** | `'86400000'` (24h) — `data-configuration.sql:4` |
| `DefaultProductSeenService.java:75` (`handleProductViewInternally`) | `redis.product.view.dedup.window.ms` | `getLongConfig(key)` — single-arg | **yes** | `'43200000'` (12h) — `data-configuration.sql:9` |
| `ProductRemovalJob.java:31` | `product.removal.batch.size` | `getIntConfig(key)` — single-arg | **yes** | `'100'` — `data-configuration.sql:22` |
| `ProductRemovalJob.java:32` | `product.removal.days.old` | `getIntConfig(key)` — single-arg | **yes** | `'30'` — `data-configuration.sql:23` |

**Confirmed:** every one of these six reads goes through a single-arg default-0
getter. All five distinct keys **have seed rows** with sane values, so the harm only
surfaces if a seed row is missing/blank in a given environment. Failure modes if a
key were absent: `delta.ttl` 0 → `redis.expire(key, 0ms)` (key dies immediately, delta
lost on next flush); `owner.ttl` 0 → owner cache entry expires instantly (every view
re-queries the DB for the owner); `dedup.window.ms` 0 → dedup key expires instantly
(no view dedup, every request counts); `batch.size` 0 → `PageRequest.of(page, 0)`
throws `IllegalArgumentException` (job dies); `days.old` 0 → removal eligibility
window collapses to "anything not updated today."

---

## Q2 — B5: config cache startup race

**Cache warms on `ApplicationReadyEvent`.** `DefaultConfigurationService.onAppReady`
is `@EventListener(ApplicationReadyEvent.class) @Order(1)`; it assigns
`configurationCache` from `configurationRepository.findAll()` —
`DefaultConfigurationService.java:31-41`. `isReady()` returns
`configurationCache != null` — `DefaultConfigurationService.java:44-46`. The interface
javadoc states the race explicitly: the cache loads on `ApplicationReadyEvent`, which
fires **after** `@Scheduled` tasks start (`ContextRefreshedEvent`) —
`ConfigurationService.java:10-17`.

### Every `@Scheduled` bean and its config exposure

| Bean / method | Schedule | Fires in startup window? | Reads config? | Getter type | Guarded? |
|---|---|---|---|---|---|
| `DatabaseHealthMonitor.poll` (`health/DatabaseHealthMonitor.java:115`) | `fixedRate` 2s (`monitor.poll.interval.seconds:2`) | **Yes** — fixed-rate fires immediately | Yes, direct: `getRequiredDoubleConfig` ×3 (`:160-162`), `getRequiredIntConfig` (`:191`) | **Throwing** required-getters | **Yes** — `if (!configurationService.isReady()) return;` at `:121-123` |
| `DefaultScheduledRedisFlushService.periodicFlush` (`redis/service/impl/DefaultScheduledRedisFlushService.java:26`) | `fixedDelay` (`app.views.flush-delay-ms`) | **Yes** — fixed-delay fires immediately | Yes, **indirect** via `RedisViewCounterService` → `getLongConfig("redis.product.view.delta.ttl")` (`DefaultRedisViewCounterService.java:34,75`) | default-0 (non-throwing) | No — but cannot throw; worst case a 0ms TTL on a delta that exists only if Redis already had tracked ids (none at boot) |
| `ProductRemovalJob.removeOldDeletedProducts` (`jobs/ProductRemovalJob.java:21`) | cron `0 0 2 ? * SUN` | No — Sun 02:00 | Yes, direct: `getIntConfig` ×2 (`:31-32`) | default-0 (non-throwing) | No — cron timing makes a boot collision implausible |
| `UserDeletionScheduledJobs.processScheduledDeletions` (`jobs/UserDeletionScheduledJobs.java:62`) | cron daily 02:00 | No | Yes, direct: `getRequiredIntConfig` (`:65`) | **Throwing** | No guard, but cron-only |
| `UserDeletionScheduledJobs.reconcileFirebaseUsers` (`:187`) | cron weekly SUN 03:00 | No | Yes: `getBooleanConfig` (`:189`), `getRequiredIntConfig` (`:195`) | mixed (one throwing) | No guard, but cron-only |
| `UserDeletionScheduledJobs.sendDeletionReminders` (`:129`) | cron daily 13:00 | No | No config read | — | — |
| `UserDeletionScheduledJobs.purgeExpiredAuditRecords` (`:241`) | cron weekly SUN 04:00 | No | No config read | — | — |
| `ProductImagesRemovalJob` (`images/job/ProductImagesRemovalJob.java:57`) | cron `app.images.sweeper.cron` | No | No config read (R2Service/UploadOwnershipService only) | — | — |
| `ChatImagesRemovalJob` (`images/job/ChatImagesRemovalJob.java:46`) | cron `app.images.chat.removal` | No | No config read (R2Service/ImageService only) | — | — |
| `ProductBaseCurrencyUpdater` (`jobs/ProductBaseCurrencyUpdater.java:39`) | cron `0 0 3 * * *` | No | No config read | — | — |
| `DefaultBaseCurrencyService` (`service/impl/DefaultBaseCurrencyService.java:59`) | `fixedRate` 86400000 | **Yes** (fixed-rate, first tick at boot) | No config read | — | — |
| `DefaultMessagingCleanupService` (`messaging/service/impl/DefaultMessagingCleanupService.java:82`) | cron `messaging.cleanup.cron` | No | No config read | — | — |

> Note: `ConfigurationService.java` appeared in the `@Scheduled` grep only because the
> token occurs in interface javadoc — it is not a scheduled bean.

**Net exposure today.** The only scheduled bean that both (a) fires immediately at
startup and (b) reads config through a **throwing** required-getter is
`DatabaseHealthMonitor.poll`, and it is **already guarded** by the `isReady()` skip
(`:121-123`). `DefaultScheduledRedisFlushService` also fires immediately but reads
only through non-throwing default-0 getters and has no startup data to act on, so it
cannot throw. Every throwing required-getter caller other than the monitor is
cron-scheduled and will not realistically fire inside the boot window. So B5 is, as of
this branch, a latent/structural risk rather than an active crash — the per-bean
`isReady()` skip is the existing mitigation pattern (decisions.md user-deletion-cron
precedent referenced at `DatabaseHealthMonitor.java:37-41`).

### Proposed fix shape

Make warming **eager and deterministic** rather than relying on a per-bean
`isReady()` skip:

- Move warming off `ApplicationReadyEvent` to `@PostConstruct` on
  `DefaultConfigurationService`, annotated
  `@DependsOnDatabaseInitialization` (Spring Boot) so it runs after Flyway/JPA schema
  init and after the seed import but during bean initialization — i.e. **before** the
  `TaskScheduler` starts firing `@Scheduled` tasks. This closes the window at the
  source so no scheduled reader can observe a null cache.
- Keep `isReady()` and the `DatabaseHealthMonitor` skip as belt-and-suspenders (cheap,
  already tested), but they stop being load-bearing.
- Caveat to verify before implementing: confirm the configuration seed
  (`data-configuration.sql`) is applied by the time `@PostConstruct` runs — if the seed
  import is itself an `ApplicationRunner`/ready-event step, `@PostConstruct` would warm
  an empty table and the eager warm would need to depend on the importer instead. (This
  is the kind of ordering check that belongs in the fix brief, not the audit.)

### Test that would assert the ordering

There is already a startup-guard test:
`DatabaseHealthMonitorTest.skipsPollWhileConfigurationCacheIsNotReady()`
(`src/test/java/com/memento/tech/oglasino/health/DatabaseHealthMonitorTest.java:248-249`)
— it asserts the *consumer-side* skip. For the *eager-warm* fix the missing assertion
is a service-level test, e.g. `DefaultConfigurationServiceTest.cacheWarmedAfterPostConstruct()`
(sibling to the existing `DefaultConfigurationServiceTest.warmCache()` setup at
`:31`), asserting `isReady()` is `true` immediately after context init and before any
`@Scheduled` task runs — most robustly an integration/`@SpringBootTest` that asserts
the cache is non-null at `ApplicationReadyEvent`/post-refresh, proving warming
precedes scheduler start.

---

## Q3 — B6: registeredWithProvider

**Stored as the firebase-claim Map's `toString()`.** In `getOrCreateUser`:

```java
var providerId =
    Optional.ofNullable(claims.get("firebase")).map(Object::toString).orElse(StringUtils.EMPTY);
```
— `DefaultFirebaseAuthService.java:110-111`. `claims.get("firebase")` is the nested
claim **Map**; `Object::toString` mangles the whole map into a string (e.g.
`{identities={email=[...]}, sign_in_provider=password}`). That value is what gets
persisted via `setRegisteredWithProvider(providerId)` —
`DefaultFirebaseAuthService.java:124` (back-fill) and `:156` (new user). The class
javadoc confirms this is intentional-but-mangled and contrasts it with the
authoritative `extractSignInProvider` path: *"the persisted `registeredWithProvider`
field … stores the claim map's `toString()`"* — `DefaultFirebaseAuthService.java:185-189`.

### Every reader of the column / `User.registeredWithProvider` / `AuthUserDTO.providerId`

Readers of `User.getRegisteredWithProvider()`:

1. `DefaultFirebaseAuthService.java:122` — `StringUtils.isBlank(...)` set-once guard (internal, not emitted).
2. `DefaultUserFacade.java:170` — `StringUtils.isBlank(...)` gate: only blank-provider (email/password) users may change their email. **Behavioral reader** — uses presence/blankness only, not the value.
3. `AuthUserConverter.java:38` — `destination.setProviderId(source.getRegisteredWithProvider())` → `AuthUserDTO.providerId`.
4. `UpdateUserConverter.java:51` — `destination.setProviderId(source.getRegisteredWithProvider())` → `UpdateUserDTO.providerId` (converter is `Converter<User, UpdateUserDTO>`, `UpdateUserConverter.java:19`).

Writers (not readers, for completeness): `DefaultFirebaseAuthService.java:124,156`;
`TestUsersImportService.java:58` (import path).

`User.java:60,208-213` is the field + accessors. `events/UserRegisteredEvent.java:11`
only documents that the event deliberately does **not** read this field.

### Is `providerId` emitted on the wire?

**Yes — twice, both carrying the mangled `toString()`:**

- `AuthUserDTO` is returned to the client by `AuthController` login/sync:
  `AuthUserDTO dto = modelMapper.map(user, AuthUserDTO.class);` then
  `return new ResponseEntity<>(dto, ...)` — `AuthController.java:163,170`. The
  ModelMapper run invokes `AuthUserConverter`, so `providerId` =
  mangled `getRegisteredWithProvider()`.
- `UpdateUserDTO` is returned by `DefaultUserFacade.getCurrentUserData`:
  `modelMapper.map(user, UpdateUserDTO.class)` — `DefaultUserFacade.java:157` —
  invoking `UpdateUserConverter`, again emitting the mangled `providerId`.

Note `DefaultPasswordResetService.java:93,99` also reads a `providerId`, but that is
Firebase `UserInfo.getProviderId()` (per-provider record off the Firebase user), a
**different** value from our `registeredWithProvider` column — not a reader of this
field.

**Conclusion:** the mangled claim-map `toString()` is exposed to clients on both the
auth/login response and the get-current-user response. (Aside, not in scope: the
authoritative provider string is available via `extractSignInProvider`
(`DefaultFirebaseAuthService.java:191-199`), which reads `firebase.sign_in_provider`.)

---

## Q4 — B7: emailVerifiedExternal

**Set-once at registration, never reconciled.** The only writer in the auth path is
`newUser.setEmailVerifiedExternal(token.isEmailVerified())` inside
`createUserSynchronized` — `DefaultFirebaseAuthService.java:155`. There is **no**
back-fill / reconciliation write anywhere (contrast `registeredWithProvider`, which
has a back-fill at `:122-126`). `FirebaseAuthFilter.java:80` explicitly documents that
per-request verification reads the live token claim
(`decoded.isEmailVerified()`), **never** the stored column — so the stored column is
never refreshed from Firebase after creation. The only other writer is the import path
`TestUsersImportService.java:56`.

### Every reader

1. `EntityUserInfoConverter.java:57` —
   `destination.setVerified(source.isVerifiedInternally() || source.isEmailVerifiedExternal())`
   → `UserInfoDTO.verified`.
2. `AuthController.java:255` — `if (recipient.get().isEmailVerifiedExternal())` in the
   verification-resend endpoint: a no-leak no-op skip when the account is already
   externally verified.

(`User.java:57,192-197` field + accessors; `ImportUserData.java` is the import DTO,
not a runtime reader.)

### Is `UserInfoDTO.verified` the only live reader?

**No — there are two readers**, and this is worth flagging against the brief's premise:

- `EntityUserInfoConverter.java:57` (feeds `UserInfoDTO.verified`) **and**
- `AuthController.java:255` (resend-verification no-op gate).

So `UserInfoDTO.verified` is **not** the only live reader; `AuthController.java:255`
also reads the column directly.

**Is `UserInfoDTO.verified` user-facing display?** Yes. `UserInfoDTO` is returned to
clients by `UserDataController.getUser` (`controller/UserDataController.java:20`) and
`AuthController.getUserForFirebaseId` (`AuthController.java:375`), produced via
`modelMapper.map(user, UserInfoDTO.class)` (`DefaultUserFacade.java:112`). The
`verified` value shown is `verifiedInternally OR emailVerifiedExternal`.

**Path discrepancy worth noting (off the brief's exact ask but relevant to B7):**
there is a *second*, projection-based path producing `UserInfoDTO.verified` that does
**not** consult `emailVerifiedExternal` at all — the JPQL projections map
`u.verifiedInternally AS verified` (`UserRepository.java:145,180`), surfaced via
`UserInfoProjection.isVerified()` (`repository/projections/UserInfoProjection.java:17`)
and `DefaultUserService.java:129` (`userInfo.setVerified(projection.isVerified())`).
So depending on which code path serves a given `UserInfoDTO`, `verified` is either
`verifiedInternally OR emailVerifiedExternal` (converter path) or `verifiedInternally`
only (projection path) — an inconsistency that the B7 fix should reconcile.

---

## Q5 — W2: notification `shown` field

**No.** No backend code writes a `shown` field to the notification document. The
in-app notification doc is assembled in
`DefaultNotificationsService.sendAsyncNotification` and the fields written are exactly:
`title`, `description`, `seen` (=`false`), `createdAt`, `type`, `categoryId`, `data` —
`DefaultNotificationsService.java:54-61`, then `.add(notificationData)` —
`:64-68`. There is no `shown` key, constant, or `.update(... "shown" ...)` anywhere in
the notification write path. The push-only path writes **no** Firestore doc at all
(`DefaultNotificationsService.java:83-91`).

The only `shown` token matches in `src/main` are unrelated:
`CategoryFilter.java:21` (a comment on `displayOrder`),
`MessageSentNotificationDTO.java:12` and `DefaultFirebasePushService.java:73` (both
prose comments about a push banner being "shown"). None write a `shown` field.

**Conclusion:** the notification document field is `seen`, not `shown`; nothing on the
backend writes `shown` meaningfully (or at all).

---

## Part 4a — simplicity

N/A — this was a read-only audit; no code was written, so there is nothing to assess
for unnecessary complexity.
