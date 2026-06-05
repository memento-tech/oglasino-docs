# Issues

Append-only log of out-of-scope findings. Newest at the top. Each entry has a date, severity, status, and a short body. Status values: `open`, `fixed`, `wontfix`, `parked`.

---

## 2026-06-04 — Password (email/password) users are wrongly told to change their email "with their provider"; in-app email change is blocked for them

**Repo:** `oglasino-backend` · **Severity:** medium · **Status:** fixed
**Found in:** `DefaultUserFacade.java:170` (the gate); `DefaultFirebaseAuthService` `extractSignInProvider` (the provider value source).
**Detail:** The gate at `DefaultUserFacade.java:170` allows an in-app email change only when the stored `registeredWithProvider` is **blank** (`StringUtils.isBlank`). Apparent intent: email/password users (no external provider) may change their email; federated users (Google etc.) may not, since the provider owns the address. Reality: email/password users do **not** get a blank provider — they get `"password"` (via `extractSignInProvider`, as of the 2026-06-04 provider-string fix; before that, the mangled claim-map `toString()` — also non-blank). So the `isBlank` branch is never true for them, and the gate blocks email changes for **all** users whose token carried a firebase claim (i.e. everyone). User-facing symptom: a password user attempting to change their email is shown a message to the effect of "change your email with your provider" — wrong/confusing, because for a password user there is no external provider; the app itself is the provider. Pre-existing; **not** caused by the 2026-06-04 provider-string fix (which preserved this behavior exactly). Flagged as a Part 4b adjacent observation in that session's backend summary. Related to the persisted-field entry [2026-06-03 — `registeredWithProvider` stored as the firebase-claim Map's `toString()`](#2026-06-03--registeredwithprovider-stored-as-the-firebase-claim-maps-tostring).

**Open questions to resolve before fixing (Igor to decide):**
- Should email/password users be able to change their email in-app at all?
- If yes: where does the change happen — through the backend (then the gate must compare against the literal `"password"` provider rather than blank), or directly via the Firebase client SDK (then the backend gate is moot and the client flow + the misleading message are what need work)? Depends on Firebase project settings (e.g. verify-before-update) and which client owns the flow — not yet audited.
- For federated (Google etc.) users, email genuinely is provider-owned; any message for them should say that accurately, distinct from the password-user case.

> **Fixed 2026-06-05 (Igor).** Two-part resolution. (1) Product decision: password
> (email/password) users do NOT change their email in-app — the gate blocking them is
> CORRECT behavior, not a defect, so the entry's framing of the block as a bug is retracted.
> (2) Code fix: the misleading "change your email with your provider" message has been
> REMOVED for password users, closing the real user-facing flaw. The three open product
> questions are answered: password users can't change email in-app (by design); no in-app
> email-change flow exists for them; federated (google.com) users remain provider-owned as
> before.

---

## 2026-06-04 — Web: `/notifications` page `markNotificationsAsSeen` exhaustive-deps warning

**Repo:** `oglasino-web` · **Severity:** low · **Status:** fixed
**Detail:** Found and resolved in the same prod-bug-sweep session (`notifications/page.tsx:23`). The only missing dep is the unstable `markNotificationsAsSeen` (recreated every render); adding it would cause redundant Firestore reads + an `onSnapshot` re-fire loop. Kept the `[user]` deps and added `eslint-disable-next-line react-hooks/exhaustive-deps` + a rationale, matching in-repo precedent.

> **Fixed 2026-06-04 (prod-bug-sweep, `dev`).** Suppress-with-rationale; lint 143 → 142. See `sessions/2026-06-04-oglasino-web-prod-bug-sweep-2.md`.

---

## 2026-06-04 — ES-performance + external-client-timeout thread: carry-forward items (all resolved)

**Repo:** `oglasino-backend` · **Severity:** mixed (per-item below) · **Status:** fixed (all four items resolved — config-getters, guava, and getTranslatedValue fixed 2026-06-04 bug-batch-fix; OpenAI-throttle bullet wontfix per Igor 2026-06-04)
**Detail:** Part 4b adjacent observations surfaced across the 2026-06-04 ES-performance / timeout-hardening / perf-bulk sessions and deliberately left out of scope. Each is independent.

- **(medium) Config default-`0` getters — unmigrated footgun callers.** `DefaultConfigurationService.getIntConfig`/`getLongConfig`/`getDoubleConfig` return `0` on a missing/blank/unparseable key. The perf-bulk session added safe `get*Config(key, default)` overloads but migrated only the OpenAI timeout consumer. Still reading via the `0`-returning getters: `DefaultRedisViewCounterService` (`redis.product.view.delta.ttl`), `DefaultProductSeenService` (`redis.product.view.owner.ttl`, `redis.product.view.dedup.window.ms`), and `ProductRemovalJob` (`product.removal.batch.size` → loud crash on 0; `product.removal.days.old` → **risks premature deletion / latent data loss**). Each is now a one-liner via the new overload but needs a business-chosen non-zero default (unspecified in the repo). **Follow-up brief candidate.** Surfaced by `perf-bulk-1`.
  > **Fixed 2026-06-04 (bug-batch-fix, `dev`).** All six remaining single-arg default-0 call sites (`DefaultRedisViewCounterService` ×2, `DefaultProductSeenService` ×2, `ProductRemovalJob` ×2) migrated to the 2-arg safe overload with the seeded value as default (86400000L / 43200000L / 100 / 30); a missing/blank row now falls back to seeded intent, not 0. Single-arg getters unchanged (other callers). 943 tests green.
- **(low) OpenAI failure Telegram alert has no throttle.** A sustained OpenAI outage fires one Telegram alert per failed request. Deliberate — no existing dedup/cooldown to reuse, and the brief said not to invent one. Throttling is a follow-up. Surfaced by `timeout-hardening-1`.
  > **Wontfix (Igor, 2026-06-04).** The un-throttled alerting is intended: sustained hammering during an OpenAI outage is the desired signal so it gets attention immediately. Not a defect.
- **(low) Guava is a transitive dependency only.** `Striped` (added this thread) and the existing `Lists` usage rely on `com.google.guava:guava` arriving transitively via `firebase-admin`; it is not declared directly in `pom.xml`. Would break silently if firebase-admin ever dropped it. Consider declaring it directly. Surfaced by `perf-bulk-1`.
  > **Fixed 2026-06-04 (bug-batch-fix, `dev`).** `com.google.guava:guava` declared directly in `pom.xml`, pinned to `33.5.0-jre` (the exact version firebase-admin already resolves — `dependency:tree` confirms same version, now top-level). `Striped` + `Lists` usages compile; 940 tests green.
- **(low, cosmetic) `ProductDocument.getTranslatedValue` is a pseudo-static instance method.** It does list filtering and is really a static utility, yet lives as a public instance method on the document; called by three converters. Predates this thread. `elasticsearch/documents/ProductDocument.java:271`. Surfaced by `es-description-text-mapping-1`.
  > **Fixed 2026-06-04 (bug-batch-fix, `dev`).** Made `public static`; all 4 call sites across the 3 converters (`SearchProductDataConverter`, `ProductOverviewConverter`, `ProductDetailsConverter` ×2) updated to `ProductDocument.getTranslatedValue(...)`. No instance state was read. 940 tests green.

---

## 2026-06-04 — Router: prod `assetlinks.json` carries a placeholder SHA-256 (Android App Links unverified until Play Console setup)

**Repo:** `oglasino-router` · **Severity:** medium · **Status:** open

`src/index.ts` `ASSETLINKS_PROD` ships `sha256_cert_fingerprints: ["REPLACE_AFTER_PLAY_CONSOLE_SETUP"]` because the app is not yet in the Play Store. Android App Links verification on `oglasino.com` will fail until the real production signing-cert SHA-256 replaces the placeholder. With Play App Signing, the correct value is Google's app-signing key SHA-256 from Play Console → Setup → App signing (list both upload + Play-signing keys; the field is an array). iOS AASA is final. **Fix:** swap once Play Console app signing is set up (one-line router edit + redeploy). See [features/deep-linking.md](features/deep-linking.md) §5.

---

## 2026-06-04 — Router: stage `assetlinks.json` carries a placeholder preview SHA-256

**Repo:** `oglasino-router` · **Severity:** medium · **Status:** open

`src/index.ts` `ASSETLINKS_STAGE` ships `sha256_cert_fingerprints: ["REPLACE_AFTER_PREVIEW_KEYSTORE_SETUP"]`. The real preview fingerprint (`eas credentials` → Android → preview keystore → SHA-256; no Play Console dependency) was not available at implementation time. Android App Links verification on `stage.oglasino.com` for `com.oglasino.preview` will fail until swapped in. **Fix:** generate the preview keystore, pull its SHA-256, swap (pairs with the prod-SHA entry above). See [features/deep-linking.md](features/deep-linking.md) §5.

---

## 2026-06-04 — In-app base-site switch leaves the product feed stale (latent, pre-existing)

**Repo:** `oglasino-expo` · **Severity:** medium · **Status:** fixed (pending on-device Ψ)

Surfaced by the deep-link base-site-switch audit. The in-app `PortalConfigDialog` base-site switch (`pickBaseSite` while the portal is mounted) does **not** refresh the home/catalog product feed unless the switch also changes the language: the feed's only post-mount refetch triggers are `fetchPage` identity and `selectedLanguage.code` (`ProductList.tsx:154,162`) — **never `selectedBaseSite`**. So switching between two base-sites that share the user's active language likely leaves the feed showing the **old** site's products until a manual refresh or navigation. The `X-Base-Site` header is correct on the next request (interceptor reads the store live) — the gap is nothing *triggers* that next request. Not introduced by the deep-link feature (the deep-link path switches before the portal mounts, so it's unaffected). **Fix candidate:** add a `selectedBaseSite.code` dependency to the feed's refresh effect, mirroring the existing language trigger. **Verify on-device first.**

> **Fixed 2026-06-04 (prod-bug-sweep, `new-expo-dev`) — pending on-device Ψ.** Added one dedicated `useEffect` in `ProductList.tsx` keyed on `selectedBaseSite?.code` (the string, not the object) with an initial-capture ref guard, calling `onRefreshCurrent()`. The home feed was the real gap; catalog mostly self-heals. 515 tests pass. **Requires device Ψ before final close:** (i) switching between two base-sites sharing the active language refreshes the home feed with no manual pull, (ii) no double-fetch / flicker, (iii) the language-also-changes case still works. See `sessions/2026-06-04-oglasino-expo-prod-bug-sweep-1.md`.

---

## 2026-06-04 — Web: app-store/play-store footer badges are dead, and no "open in app" surface exists yet

**Repo:** `oglasino-web` · **Severity:** low · **Status:** open (deferred — blocked on store listings)

Web's footer app-store/play-store badges (`Footer.tsx:80-86`) are icons only — no `<a>`/`href`/`onClick`. They're the natural consumer surface for store-listing links and a future "open in app" affordance, but both are blocked on the app being published (store-listing URLs don't exist yet). The dormant `appDeepLink.ts` scaffold (`APP_DEEP_LINK_ENABLED=false`, custom-scheme based) is the intended single home to extend for an https "open in app" link. **Deferred** until the app is in the stores; no v1 web work. See [features/deep-linking.md](features/deep-linking.md) §5.

> **Note (2026-06-04):** overlaps with the 2026-05-31 issue "Mobile-app store badges on web are dead/icon-only" — that entry tracks wiring the badges to the **store-listing** URLs; this entry adds the **"open in app" https affordance** as the second, app-publish-gated surface on the same badges. Resolve together when the app is in the stores.

---

## 2026-06-03 — Mobile: `getNormalizedProductUrl` hardcodes the production host on all build tiers

**Repo:** `oglasino-expo` · **Severity:** medium · **Status:** fixed

`getNormalizedProductUrl` (`src/lib/utils/utils.ts:136`) hardcodes `https://oglasino.com` regardless of build tier, so **preview/dev builds emit product *share* links pointing at production** — a tester sharing a product from a non-prod build sends recipients to the prod site (wrong-environment links; not a crash). Surfaced while adding the tier-aware `getForgotPasswordUrl` (same file), which selects the host from `extra.env` (`production → oglasino.com`, else `stage.oglasino.com`). **Fix direction:** make the share-URL host tier-aware the same way — the two URL builders could be unified onto a single tier→host helper. Left unfixed deliberately: changing share-link host is a behavior change wanting its own decision, and it's outside the password-reset scope. Low effort, own touch.

> **Fixed 2026-06-04 (prod-bug-sweep, `new-expo-dev`).** Extracted a single `getWebBase(env)` host helper in `utils.ts` (`production` → `oglasino.com`, else `stage.oglasino.com`); the prefixed `getNormalizedProductUrl` and `getForgotPasswordUrl` both resolve host through it, and `env` was threaded into the two outbound share-URL callers (`ShareProductButton.tsx`, `UploadedProductDialog.tsx`). Preview/dev builds now emit stage share links, not prod. `utils.test.ts` updated; 515 tests pass. URL-string logic, unit-tested — no device Ψ strictly required. See `sessions/2026-06-04-oglasino-expo-prod-bug-sweep-1.md`.

---

## 2026-06-03 — Web: `Input` interpolated Tailwind width class is invisible to JIT

**Repo:** `oglasino-web` · **Severity:** low · **Status:** fixed

`src/components/server/Input.tsx:63-65` builds a width class by interpolation — `` max-w-[${maxWidth}px] `` — which Tailwind's JIT compiler cannot see at build time, so any non-default `maxWidth` silently produces **no** width class (the prop appears to work but does nothing). Surfaced during the password-reset web build (which passes no `maxWidth`, so unaffected). **Fix direction:** use a static class map or an inline `style={{ maxWidth }}` rather than an interpolated arbitrary value. Low severity (no current caller relies on it) but a real footgun for a future caller who sets `maxWidth` and sees it ignored. Left unfixed: out of scope.

> **Fixed 2026-06-04 (prod-bug-sweep, `dev`).** Both `Input.tsx` and `Textarea.tsx` switched from the `max-w-[${maxWidth}px]` interpolation to inline `style={maxWidth > 0 ? { maxWidth } : undefined}`. The live caller (`owner/products/[productId]/page.tsx:489`, `maxWidth={100}`) now gets its cap — audit corrected the "purely latent" premise: Item had one live caller. The `Textarea.tsx` twin was not in the original report; found in audit and fixed in the same session. tsc/test green. See `sessions/2026-06-04-oglasino-web-prod-bug-sweep-2.md`.

---

## 2026-06-03 — `DefaultConfigurationService` cache warms after @Scheduled tasks start (startup race)

**Repo:** `oglasino-backend` · **Severity:** medium · **Status:** fixed
**Found in:** `service/impl/DefaultConfigurationService.java` (cache populated on `ApplicationReadyEvent`);
surfaces in any early `@Scheduled` config consumer.
**Surfaced by:** DB Overload Protection stage verification (2026-06-03), via `DatabaseHealthMonitor`.

**Detail.** `DefaultConfigurationService` populates its `configurationCache` on `ApplicationReadyEvent`.
Spring starts `@Scheduled` tasks earlier, at `ContextRefreshedEvent` (which fires before
`ApplicationReadyEvent`). So there is a window at every boot where a scheduled task that reads
configuration sees an unwarmed cache: `getConfig(...)` returns empty, and the required-accessors
(`getRequiredConfig` / `getRequiredDoubleConfig`, etc.) throw
`IllegalStateException: Required configuration key is missing or blank` rather than returning a value.
The condition self-corrects once `ApplicationReadyEvent` warms the cache (≈1s into boot), so steady-state
operation is unaffected — but the first poll(s) of any early scheduled consumer fail at every cold start.

**How it surfaced.** `DatabaseHealthMonitor` polls every 2s with no initial delay and uses the throwing
required-accessors, so its first cold-boot poll threw the `IllegalStateException` (for
`threshold.yellow.ratio`) and the monitor sat at GREEN until the cache warmed. The seed data was present
and correct — purely a load-ordering race, not missing config.

**Blast radius (other early consumers in the same window).** Any `@Scheduled` task reading config during
the startup window is exposed, e.g. `DefaultScheduledRedisFlushService` and the image-removal jobs. These
have not visibly broken (they do not use the throwing accessors, tolerate the empty return, or do not run
that early), but the latent race applies to them too. The monitor is simply the consumer that surfaced it.

**Already mitigated for the monitor.** The DB-overload `DatabaseHealthMonitor` was made immune during stage
verification (early-return / unwarmed-cache skip on the poll, confirmed by a clean cold boot). This issue
covers the REMAINING, central fix for all other early consumers — the monitor's local guard does not fix them.

**Proposed fix (deliberate, not a feature rider).** Warm the cache eagerly and ordered before scheduled
tasks — e.g. `@PostConstruct` + `@DependsOnDatabaseInitialization` on the cache-init method, so it runs
after `spring.sql.init` seeding but before `@Scheduled` tasks start. This is a change to a CORE service that
the whole app depends on, so it must be done on a clean tree with its own test pass (assert the cache is
populated before scheduled tasks run) and a quick check that the other early consumers
(`DefaultScheduledRedisFlushService`, image-removal jobs) are not surprised by the new ordering. Not bundled
into feature work; not urgent (benign, self-correcting, predates the DB-overload feature).

> **Fixed 2026-06-04 (bug-batch-fix, `dev`).** Warm moved to `@PostConstruct` + `@DependsOnDatabaseInitialization`, running after Flyway + `spring.sql.init` seed but before the scheduler. `isReady()` + `DatabaseHealthMonitor` skip kept as non-load-bearing backup. Seed-ordering verified. Unit test added; `@SpringBootTest` ordering assertion still owed (no harness in project). 943 green.

---

## 2026-06-03 — Web: JSON-LD `<script>` injection — backend strings via `JSON.stringify` not escaped for script context

**Repo:** `oglasino-web` · **Severity:** high · **Status:** fixed
**Found in:** `src/components/server/seo/JsonLd.tsx:14` (`dangerouslySetInnerHTML={{ __html: JSON.stringify(item) }}`); fed by `generateProductPageStructuredData.ts:61-62,116`, `generateUserPageStructuredData.ts:19,46`, `generateCatalogPageStructuredData.ts:44`.
**Detail:** Raw `product.name` / `product.description` / `user.displayName` / `product.title` are injected into a `<script type="application/ld+json">` via `JSON.stringify`, which does not escape `</script>`/`<`/`>`. A value containing `</script><script>…</script>` breaks out and executes — stored XSS on public, unauthenticated pages. Currently masked by the backend's input HTML-encoding (`InputSanitizationFilter`); will be exposed when the `backend-security-hardening` feature removes that filter (M1). Fix web-side: escape serialized JSON for the script context (`.replace(/</g,'\\u003c')`) or change the delivery off a raw `<script>`. Closing this is Brief 3 of `backend-security-hardening`, and it **gates** the backend filter removal (Brief 4). Surfaced by the backend-security-hardening web output-encoding audit (`sessions/audit-web-output-encoding.md`).

> **Fixed 2026-06-04 (backend-security-hardening Brief 3).** `oglasino-web` `src/components/server/seo/JsonLd.tsx` escapes serialized JSON-LD for the HTML script context as its single output path (`serializeJsonLd`: `<`/`>`/`&` → `\uXXXX`); a `</script>` payload in any backend/user field can no longer break out. Output remains valid, parseable JSON-LD. Verified by `JsonLd.test.ts`.

---

## 2026-06-03 — Account-method collision: social user's email+password register/login UX is unhandled

**Repo:** `oglasino-web` + `oglasino-expo` + `oglasino-backend` (Firebase config) · **Severity:** medium · **Status:** open

Surfaced during password-reset Phase 1 intake. A user who created their account via a social provider (Google) has a Firebase user with provider `google.com`. The Firebase project is set to **"Link accounts that use the same email"** (one user per email; providers linkable). Two unhandled UX consequences: (1) `signInWithEmailAndPassword` on that email fails with the generic `auth/invalid-credential` (email-enumeration protection on) — the user is told "Invalid email or password" with no path forward; (2) under account-linking, getting a password onto that account requires a credential-linking (re-auth) flow rather than a clean "add a password" UX — neither client surfaces a usable linking flow today, and `mapAuthError` shows generic copy. **Not worked out:** the product behavior — (a) detect and guide ("this email uses Google — sign in with Google", squared against the no-leak posture used elsewhere), (b) build an explicit account-linking UX (let the user add a password to a social account via re-auth), or (c) friendlier copy on the raw errors. **Needs** its own Mastermind feature chat. **NOT a password-reset blocker** — reset is correct regardless (social-only accounts get the provider-gated "use social" email, never a reset link). Logged so it isn't lost.

---

## 2026-06-03 — `registeredWithProvider` stored as the firebase-claim Map's `toString()`

**Repo:** `oglasino-backend` · **Severity:** medium · **Status:** fixed
**Found in:** `service/impl/DefaultFirebaseAuthService.java` `getOrCreateUser` (~:96-97; provider-derive path also flagged ~:78-79).
**Detail:** `getOrCreateUser` stores `User.registeredWithProvider` as the `firebase` claim **Map's `toString()`** (e.g. `{sign_in_provider=password, identities=…}`) rather than the provider string. That stored value feeds `AuthUserDTO.providerId`, which web/mobile may read. The email-notifications feature's verification gate and registration listener read the **nested `firebase.sign_in_provider` claim** off the live token correctly and are unaffected, but the persisted field is likely garbage. Confirmed by multiple email-notifications backend sessions (`-4` Part 4b, `-5` Brief-vs-reality §3); deliberately not fixed there (out of scope — the feature reads the claim, not the column). Fix: store the actual provider string (`sign_in_provider`), not the map's `toString()`. Route to a backend chat.

> **Fixed 2026-06-04 (bug-batch-fix, `dev`).** `getOrCreateUser` now derives `providerId` via `extractSignInProvider` (`firebase.sign_in_provider`) instead of the claim map's `toString()`; both write sites and both wire emitters (`AuthUserDTO`/`UpdateUserDTO`.`providerId`) carry the clean value. No migration (no existing users). Email-edit gate behavior unchanged. 943 tests green.

---

## 2026-06-03 — `emailVerifiedExternal` is stale-by-design — must not be read as authoritative

**Repo:** `oglasino-backend` · **Severity:** low/medium · **Status:** open
**Found in:** `service/impl/DefaultFirebaseAuthService.java` (~:143, set once at registration); column `users.email_verified_external`.
**Detail:** `emailVerifiedExternal` is captured **once** at registration from the token and **never reconciled** — it goes stale the moment a user verifies after first login. It must **not** be used as an authoritative verification source anywhere: the **live Firebase token** is authoritative (the `FirebaseAuthFilter` gate reads it). The email-notifications resend path's former reliance on this column was **removed** (`-10`, Igor-confirmed: the unauthenticated resend must not trust it — no live token there to read true state, and the rate-limits + no-leak property bound the harmless re-send). Logged as a trap so nothing reads it as truth later. Flagged across email-notifications backend sessions (`-1` audit, `-4`, `-6`, `-10`). Today its only live reader is display (`UserInfoDTO.verified`).

> **2026-06-04:** projection path reconciled to OR in `emailVerifiedExternal`, matching the converter — both display paths now agree. Column remains stale-by-design; live Firebase token stays source of truth.

---

## 2026-06-03 — `DefaultCloudflareKvService` uses a no-timeout `RestTemplate`

**Repo:** `oglasino-backend` · **Severity:** medium · **Status:** fixed
**Found in:** `service/impl/DefaultCloudflareKvService.java:23` (`private RestTemplate
restTemplate = new RestTemplate();`).
**Detail:** The Cloudflare KV client uses a `RestTemplate` with no connect/read timeout,
unlike the 3s-timeout `RestTemplate` in `ApplicationConfig`. A hung Cloudflare KV call blocks
the calling thread indefinitely. Surfaced by the DB-overload-protection Phase-2 audit. The
auto-trip path in that feature explicitly avoids this by using a timeout-bounded client for
its own KV write, but the EXISTING `toggleMaintenance()` (the admin maintenance wrench) still
uses the no-timeout client. Fix: give `DefaultCloudflareKvService` a timeout-bounded
`RestTemplate` (mirror `ApplicationConfig.restTemplate()`). Out of scope for the audit;
logged for a backend chat.

> **Fixed 2026-06-04 (external-client timeout hardening).** Legacy field-init `new RestTemplate()` deleted; `DefaultCloudflareKvService` now uses the single bounded `ApplicationConfig.restTemplate()` bean (3s/3s) for both the admin toggle and the auto-trip assert path. The two read helpers stay distinct (degrade-to-off vs propagate-on-failure); only the client is shared. Full suite 935 green.

---

## 2026-06-03 — `FirebaseAuthFilter` and `InternalTokenFilter` are double-registered

**Repo:** `oglasino-backend` · **Severity:** medium · **Status:** fixed
**Found in:** `ApplicationConfig:22` (`FirebaseAuthFilter` `@Bean`), `ApplicationConfig:33`
(`InternalTokenFilter` `@Bean`); both added to the Security chain in `SecurityConfig:83-84`.
**Detail:** Both are `@Bean` `OncePerRequestFilter`s with no `FilterRegistrationBean`
disabling servlet auto-registration, so Spring Boot also auto-registers them as standalone
servlet filters at `LOWEST_PRECEDENCE`, in addition to their place in the Security chain.
`OncePerRequestFilter`'s dedupe guard means each body runs once per request, but the wiring is
duplicated. `RateLimitConfig:61-67` shows the correct disable pattern
(`reg.setEnabled(false)`). Surfaced by the DB-overload-protection Phase-2 audit. Pre-existing;
out of scope. Fix: add `FilterRegistrationBean`s disabling servlet auto-registration for both,
matching `RateLimitConfig`.

> **Fixed 2026-06-04 (backend-security-hardening Brief 6 / M2).** Servlet auto-registration disabled via `FilterRegistrationBean.setEnabled(false)` for both filters in `ApplicationConfig`, mirroring `RateLimitConfig`. (Note: both extend `OncePerRequestFilter`, so the body already ran once — this was redundant wiring, not a perf doubling, correcting the original security audit's claim.) Full suite 915 green.

---

## 2026-06-02 — RESOLVED — New-message push did not fire (emitter never built)

**Repo:** `oglasino-expo` + `oglasino-web` (client emitter) · **Severity:** medium · **Status:** fixed
**Detail:** (2026-06-02) RESOLVED — new-message push did not fire on either client. Root cause: the
message-ping was built **receiver-first** (backend B4 endpoint `/notifications/message-sent`) but
the client **emitter** that calls it on message-send was never built — caught during on-device
verification, when no network call left the device on send. Fixed: expo E3 + web W4 added the
post-commit fire-and-forget ping to `/notifications/message-sent` (`{chatId, messageText}` only,
Part 11 — recipient/participant-check server-side). Confirmed working on device + web 2026-06-02.
Process note: the `mobile-stable` hold did its job — device verification surfaced the gap before the
feature was marked stable. See [decisions.md](decisions.md) 2026-06-02.

---

## 2026-06-02 — RESOLVED — Image-only messages produced no push

**Repo:** `oglasino-backend` (B6) + `oglasino-expo` (E5) + `oglasino-web` (W4) · **Severity:** medium · **Status:** fixed
**Detail:** (2026-06-02) RESOLVED — image-only (no-text) messages produced no push (the emitter
skipped them; the client must not fabricate a body). Fixed across the stack: backend B6 relaxed the
message-ping DTO to accept empty `messageText` and supplies a localized "photo" body
(`notif.message.photo.body`) in the recipient's language; expo E5 + web W4 now ping with empty
`messageText` for image-only and let the backend frame the body. Both clients send `messageText=''`
(empty string, key present) for image-only — a consistent wire shape. Confirmed on device
2026-06-02. See [decisions.md](decisions.md) 2026-06-02.

---

## 2026-06-02 — RESOLVED — iOS launch crash from ungated ATT call (pre-existing, non-notifications)

**Repo:** `oglasino-expo` · **Severity:** high (crash) · **Status:** fixed
**Detail:** (2026-06-02) RESOLVED — the iOS dev build crashed instantly on launch (SIGABRT, TCC)
because `AnalyticsInit.tsx` called `requestTrackingPermissionsAsync()` (App Tracking Transparency)
at launch without `NSUserTrackingUsageDescription` in `Info.plist`. Android unaffected (no ATT).
Oglasino uses first-party analytics only, not ad-tracking, so ATT is not needed: the ATT call was
**removed** (not legitimized with a usage string), and the analytics init gate is now consent-only
(`isAnalyticsConsentGranted()`), with the `attAllowed` operand removed (not defaulted false, which
would have silently disabled analytics). Two `src/` files, no native change, 47 tests. Pre-existing
issue surfaced by the fresh dev build; unrelated to the notifications feature (surfaced and fixed
alongside the notifications device-testing round).

---

## 2026-06-02 — NEW (low) — unused dependency `expo-tracking-transparency`

**Repo:** `oglasino-expo` · **Severity:** low · **Status:** wontfix
**Detail:** `expo-tracking-transparency` is now an unused dependency in `oglasino-expo`
`package.json` (its only use, the ATT call, was removed — see the ATT-crash fix above). Removing it
is a dependency change (npm + pod drop) requiring a prebuild/rebuild, so it was deferred. Remove at
the next expo prebuild. No runtime impact while unused.

> **2026-06-04 (prod-bug-sweep) — stays open, recategorized "remove at next prebuild."** Audit confirmed the JS usage and the ATT call are gone (launch-crash fix), but the dep is still in `package.json:63` + lockfile + `node_modules` + the prebuilt iOS Pod (`Podfile.lock` ExpoTrackingTransparency 6.0.8). Removal = `npm uninstall` + prebuild, both forbidden in a code session. Not closeable as not-an-issue — there is a concrete pending action, and an unused ATT native module is App Store reviewer-bait. Fold into the next prebuild.

> **Wontfix 2026-06-04 (Igor).** Removed at next prebuild as a build step; not separately tracked — a rebuild is owed regardless of this dep, so the drop rides it. Supersedes the "stays open" prod-bug-sweep note above.

---

## 2026-06-02 — NEW (low) — analytics GA4 DebugView eyeball owed

**Repo:** `oglasino-expo` · **Severity:** low · **Status:** fixed
**Detail:** After the ATT-call removal, the analytics init path is structurally confirmed
(consent-only gate, tsc/eslint/47 tests green) but events were not personally watched landing in
GA4 DebugView. Owed: toggle consent on, trigger an event, confirm it reports. Expected to work;
flagged for completeness.

> **Closed 2026-06-04 (prod-bug-sweep).** Verification item — considered done.

---

## 2026-06-02 — NEW (low, cosmetic) — web/mobile message-body trim divergence

**Repo:** `oglasino-expo` + `oglasino-web` · **Severity:** low · **Status:** fixed
**Detail:** For text messages, `oglasino-expo` trims the message text before sending the ping
(`messageText.trim()`) while `oglasino-web` sends it raw. Both send a non-blank string for text and
`''` for image-only, so the backend behaves identically; the only effect is a recipient could see
leading/trailing whitespace in a web-originated push body that mobile would have trimmed. Trivial;
no session warranted. Flagged for consistency.

> **Fixed 2026-06-04 (prod-bug-sweep).** Web side: `useChatStore.ts:536` now sends `textBlock?.text?.trim() ?? ''`, preserving the image-only `''` contract; stored Firestore content unaffected (`dev`). Expo side: no change — expo already trims correctly (`useActiveChatStore.ts:452`); the divergence was web-only. Both halves now consistent. See `sessions/2026-06-04-oglasino-web-prod-bug-sweep-2.md`.

---

## 2026-06-02 — RESOLVED — Push-token sink ambiguity (fcmToken Firestore-vs-Postgres)

**Repo:** `oglasino-web` (+ `oglasino-backend` send-side, `oglasino-expo`) · **Severity:** medium · **Status:** fixed
**Detail:** (2026-06-02) RESOLVED — the long-standing "`fcmToken` lives in the Firestore `users`
doc vs the Postgres `push_token` table" question. The backend send-side reads ONLY the Postgres
`push_token` table; the Firestore `users.fcmToken` write was dead weight. `oglasino-web` (W1)
deleted the Firestore path entirely (`fcmClient.getFcmToken`, `setUserFcmToken`, the
`ensureUserInFirestore` token write, the owner-settings toggle's Firestore write) and consolidated
on the backend Postgres path. Postgres `push_token` is now the sole web-side sink. The push-token
attach body also no longer carries `userId` (web W1 + expo E2; backend B1 derives the owner from
auth). Closed by the notifications feature — see [decisions.md](decisions.md) 2026-06-02 and
[features/notifications.md](features/notifications.md) §8.1.

---

## 2026-06-02 — RESOLVED — Ban/unban notification deep-link route-shape mismatch

**Repo:** `oglasino-backend` (+ web/expo navigate contract) · **Severity:** medium · **Status:** fixed
**Detail:** (2026-06-02) RESOLVED — the ban/unban notification deep-link did not resolve (raised by
web W2). The backend originally emitted the mobile-shaped `/dashboard/products?productId=`, which
resolved on neither client after the owner route tree was flattened to `/owner/*` on both. Backend
B5 standardized all notification navigate paths to `/owner/*`: ban/unban →
`/owner/products?productId=`, reviews → `/owner/reviews` (flat), report → INFO category with no
navigate. Verified resolving on web (W3) and expo (E2): `/owner/products` and `/owner/reviews` both
resolve, no 404. The web↔expo↔backend navigate contract now agrees.

---

## 2026-06-02 — Notifications feature: carry-forward items (flagged, not fixed)

**Repo:** `oglasino-web` + `oglasino-backend` · **Severity:** mixed (per-item below) · **Status:** fixed (all 5 bullets resolved — 4 fixed 2026-06-04 prod-bug-sweep; the backend `shown`-field bullet closed 2026-06-04, residual dead expo type-member cleanup deferred to an expo structural-sweep session)
**Detail:** Low-priority items surfaced during the notifications feature (2026-06-02) and
deliberately flagged-not-fixed. Each is independent. **Per-bullet status is marked inline below.**

- **(low)** ~~Web `/notifications` in-app row click uses the plain `next/navigation` `useRouter`
  (`notifications/page.tsx`) passed into `resolveNotificationAction`, which `stripRoutingLocale()`s
  the path but does not re-add the locale (unlike the foreground/SW handlers, which use the
  next-intl-wrapped router). Same destination route, different locale-resolution path; works via
  edge resolution. Pre-existing (predates the notifications feature). Flagged by web W2/W3.~~
  **→ Fixed 2026-06-04 (prod-bug-sweep, `dev`).** `/notifications/page.tsx` switched from the plain
  `next/navigation` `useRouter` to the next-intl-wrapped `@/src/i18n/navigation` router, matching the
  other handlers. (Audit correction: the SW/foreground handlers do **not** call
  `resolveNotificationAction` — they inline their own switch; the only callers are this page and the
  bell-driven toast.)
- **(low/medium)** ~~Web service-worker focus-match never matches (opens a new tab each tap). In
  `public/firebase-messaging-sw.template.js` the `notificationclick` focus-match compares an
  absolute `client.url` to a relative url, so it never matches — every background notification tap
  opens a new window/tab instead of focusing an already-open one. Pre-existing. UX (extra tabs).
  Flagged by web W2 (outside the W2 locale scope).~~
  **→ Fixed 2026-06-04 (prod-bug-sweep, `dev`).** Focus-match now compares pathnames via `new URL(...)`
  instead of absolute-vs-relative `===`. Edited the `.template.js`; the generated
  `firebase-messaging-sw.js` is not git-tracked — Igor's build regenerates it, and testing requires a
  hard refresh / SW update.
- **(low)** Backend `shown` field possibly dead across the stack. Expo E2 removed its only mobile
  consumer (the `nonShown` filter) and writer (`markNotificationAsShown`). The field is now
  unreferenced by mobile code and was already unrendered on web. It remains on the type as a
  documented backend-written wire field. ACTION: confirm whether the backend writes `shown`
  meaningfully; if not, drop it from the notification doc shape and both clients' types. Flagged by
  expo E2.
  > **Closed 2026-06-04.** Backend writes `seen`, never `shown` (audit confirmed:
  > `DefaultNotificationsService` writes no `shown` key); the web type never declared `shown`. Both
  > are no-ops. The only live action is removing the dead `shown` member from the expo notification
  > type, deferred to a dedicated expo structural-sweep session.
- **(low)** ~~Web `resolveNotificationAction` (`src/notifications/lib/notificationActions.ts`) has no
  unit test. The W3 guard-hoist behavior (NAVIGATION / SAVED_PRODUCT resolve to `undefined` when
  their data field is absent) would be cheap to lock with a small vitest spec. Filed per Mastermind
  (will not get a dedicated session). Flagged by web W3.~~
  **→ Fixed 2026-06-04 (prod-bug-sweep, `dev`).** Added `notificationActions.test.ts` (vitest, 10 cases
  across the 7 scenarios: SAVED_PRODUCT / NAVIGATION / PRODUCT_EXPIRATION / PRODUCT_EXPIRED / MESSAGE /
  unknown).
- **(low)** ~~Web `router: any` parameter type in `notificationActions.ts:7` — could be typed to the
  next-intl / next-navigation router union; intersects the pre-existing plain-vs-wrapped router
  inconsistency on the notifications page (first bullet above), so not a one-liner. Filed per
  Mastermind (will not get a dedicated session). Flagged by web W3.~~
  **→ Fixed 2026-06-04 (prod-bug-sweep, `dev`).** Three `router: any` (`notificationActions.ts`,
  `notificationManager.ts`, `useNotifications.ts`) typed to `WrappedRouter`, sequenced after the locale
  fix that unified both callers onto the wrapped router.

---

## 2026-06-01 — Orphaned Facebook sign-in scaffolding (cosmetic-hide leftover)

**Repo:** `oglasino-expo` · **Severity:** low · **Status:** wontfix
**Detail:** The 2026-06-01 Facebook-button hide removed only the button + icon import from `LoginOptionsDialog.tsx`. Left in place, now caller-less, for a future Ω teardown: `FacebookIcon.tsx` (no consumer); `authStore.ts` `loginWithFacebook` type :62 + impl :143 + the `loginWithFacebookFirebase` import :9; `authService.ts:215` `loginWithFacebookFirebase` stub over its commented-out FB SDK block :216–240 (the commented block is a standing Part 4 violation); `authStore.test.ts:34` stale `loginWithFacebookFirebase` mock; `DeleteAccountConfirmationDialog` 'facebook.com' provider branch (structurally unreachable). Deliberate scope decision, not an oversight.

> **Wontfix 2026-06-04.** Scaffolding intentionally retained for future Facebook login; not a defect.

---

## 2026-06-01 — Mobile: Facebook login button shown though Facebook is not an allowed sign-in option

**Repo:** `oglasino-expo` · **Severity:** medium · **Status:** fixed
**Detail:** The Expo login screen renders a Facebook sign-in button, but Facebook is not an allowed log-in option for the platform. Reported by Igor 2026-06-01. No mobile code was read here, so no `file:line` is asserted. Fix: remove the Facebook button from the login UI so only the supported sign-in providers are offered. Confirm against web's login provider set during the fix to keep parity.

> **Fixed (code, cosmetic hide) 2026-06-01 (session bug-batch-3).** Facebook provider button + its `FacebookIcon` import removed from `LoginOptionsDialog.tsx`. Deliberate cosmetic-hide scope — the broader FB scaffolding (FacebookIcon.tsx, `loginWithFacebook` store action, `loginWithFacebookFirebase` stub + commented SDK body, test mock, `DeleteAccountConfirmationDialog` 'facebook.com' branch) is left in place and logged for a future Ω teardown (see the 2026-06-01 "Orphaned Facebook sign-in scaffolding" entry directly above). On-device confirmation owed (Ψ).

---

## 2026-06-01 — Create/update product-parity carry-forward items (out of scope, logged open)

**Repo:** `oglasino-expo` + `oglasino-backend` + `oglasino-web` · **Severity:** mixed (per-item below) · **Status:** fixed (all 5 items resolved — the `FilterConverter` selected-filters bullet, the last open item, fixed 2026-06-04)
**Detail:** Findings surfaced during the create + update product-parity feature (2026-06-01) and deliberately left out of scope. Logged so they are not lost. Each is independent.

- **(low)** ~~`ProductCard` renders raw enum as state badge text — `oglasino-expo` `ProductCard.tsx:84`/`:88` prints the literal `ProductState`/`ModerationState` enum string (e.g. "INACTIVE", "BANNED") rather than a translated label. Parity item if real product cards are expected to show localized state labels.~~ **Closed as not-an-issue 2026-06-04 (prod-bug-sweep).** Audit confirmed web renders the same raw enums (`UniversalProductCard.tsx:47,52`) — this is platform **parity, not a defect**. The badge shows correct status (INACTIVE/DELETED/BANNED) on the owner-facing dashboard only; localizing would need new backend seed rows + a two-platform change + translator debt for low-value owner-facing polish. Not deferred — closed.
- **(low)** ~~Preview Details tab reads `imageKeys` directly, not the hydrated `imagesData` — `oglasino-expo` `PreviewProductDialog.tsx` (details mode). Guarded (`?.map ?? []`) so it cannot crash, but not re-pointed to the single-`imagesData` model. Follow-up if the details preview should mirror the update screen's image model.~~ **Fixed (code) 2026-06-01 (session bug-batch-3).** Details-mode carousel re-pointed from `imageKeys` to the hydrated `imagesData` (key → `publicImageUrl(key,'hero')`, else `file?.uri`), `?? []` guard kept. Live product page unaffected (correctly reads backend-hydrated `imageKeys`). On-device confirmation owed (Ψ).
- **(low)** ~~Unused `getTranslation` helper in `oglasino-expo` `PreviewProductDialog.tsx` — pre-existing dead code, lint-flagged. Sweep on next touch.~~ **Fixed (code) 2026-06-01 (session bug-batch-3).** Dead `getTranslation` helper deleted from `PreviewProductDialog.tsx`; the orphaned `t`/COMMON_SYSTEM binding it solely fed was removed too (the audit's 'used elsewhere' claim was wrong — engineer caught it).
- **(low)** `FilterConverter` type-mismatch on the selected-filters path — `oglasino-backend` `ProductForUpdateConverter` (~line 46 / the carried-over mapping). Maps a `Filter` entity through a `CategoryFilter`-typed converter, leaving `FilterDTO.basic`/`order` at defaults on the product's *selected* filters. Pre-existing, deliberately fenced off during the backend fix. NOTE: the NEW category-`filters` path does NOT have this problem (it maps through `CategoryConverter` correctly). **Parked 2026-06-01.** Confirmed real by the backend audit (`audit-backend-filterconverter.md`): `ProductForUpdateConverter` maps a bare `Filter` to `FilterDTO` on the selected-filters path; no `Converter<Filter,FilterDTO>` exists, so `FilterConverter` (typed `CategoryFilter→FilterDTO`) is bypassed and the default mapper leaves `basic`/`order` at `false`/`0`. Mechanism is converter-bypassed (not run-and-dropped). Blast radius is one site — category and catalog paths pass `CategoryFilter` and are correct. Confirmed HARMLESS: both the web and the mobile update-product forms source filter layout/grouping from the CATEGORY filters and read the SELECTED filters only for pre-selected values — neither reads `basic`/`order` off the selected filters (web: `MetaDataProduct.tsx`; mobile: `MetaDataProduct.tsx:33–56`, grep for basic/order/sort returns nothing). Parked under Part 4a — no consumer reads the defaulted fields, so fixing wrong-but-unread data isn't worth the test churn. Fix shape if a future consumer ever reads them (Option A from the audit): in `ProductForUpdateConverter`'s selected-filters loop, resolve the matching `CategoryFilter` from the product's category chain and map THAT through `FilterConverter`; do not alter `FilterConverter`. Test gap noted: `ProductForUpdateConverterTest` nulls `filterValues`.
  > **Fixed 2026-06-04 (`e4-filterconverter-fix`, `dev`).** Option A landed in `ProductForUpdateConverter` — selected filters now resolve their `CategoryFilter` from the product's category chain and map through `FilterConverter`; falls back to prior behavior when a selected filter is absent from the chain. 2 new tests, full suite 942 green. Correction to the original note above: the bypassed default left `order` at `useDefaultOptionsOrder ? 1 : 0`, not strictly `0` (non-deterministic) — the matched path now bypasses this entirely.
- **(low/medium)** ~~Web latent price-gate coupling — `oglasino-web` `page.tsx:402`/`:481`. The price/currency render block is gated on `productDetails.topCategory`; if category ever stops arriving, the price field silently disappears. Not biting now (categories arrive post-`ProductForUpdateDTO` fix), but worth a guard.~~ **Fixed 2026-06-01 (oglasino-web `dev`, session price-gate-guard-1).** Price/currency block decoupled from category *presence* — gate changed from `topCategory && !topCategory.freeZone` to `!topCategory?.freeZone` at page.tsx:402/:481; price now renders whether or not `topCategory` arrived, free-zone exclusion preserved.

---

## 2026-06-01 — Theme toggle segments use raw untranslated value as accessibility label (web + mobile)

**Repo:** `oglasino-web` + `oglasino-expo` · **Severity:** low · **Status:** open
**Found in:** `oglasino-web/src/components/client/buttons/ToggleButton.tsx:23` (`aria-label={opt.value}`); `oglasino-expo/src/components/dialog/dialogs/PortalConfigDialog.tsx` (segment `accessibilityLabel={option.value}`).
**Detail:** The tri-state theme control is icon-only on both platforms (Sun/Monitor/Moon). The per-segment accessibility label is the raw untranslated theme value string — `"light"`/`"system"`/`"dark"` — so a screen reader announces English value strings regardless of locale. Web has carried this since the theme-cookie-persistence feature (2026-05-27). Mobile deliberately matched web's behavior in the expo-system-theme feature (2026-06-01) rather than diverging — matching the reference platform was the call, and a per-segment label translation key was explicitly rejected because web has none and parity required none. Fixing it is a shared decision: introduce localized accessibility labels on both platforms (new translation keys in both repos), or accept the gap. Surfaced during the expo-system-theme web reference check (2026-06-01). Out of scope for that feature, which matched rather than fixed.

---

## 2026-06-01 — Review deletion has no backend route; dead "Delete" button removed from both UIs

**Repo:** oglasino-backend (missing capability) · oglasino-web + oglasino-expo (dead UI, now removed)
**Severity:** low · **Status:** open
**Detail:** The dashboard given-reviews card carried a "Delete" button on both web and mobile with no onClick/onPress, no backing service, and no backend route — review deletion was never implemented. reviewService exposes only canReviewProduct / reviewProduct / getReviews / getMyReviews on both platforms; repo-wide grep for deleteReview / delete-review found nothing. The dead button (and its empty action-row wrapper) was removed from oglasino-expo GivenReviewCard and oglasino-web GivenReviewCard on 2026-06-01 (sessions dashboard-reviews-buttons + given-review-delete-removal). Wiring real review deletion — letting a user retract a review they authored — is deferred feature work: it needs a backend endpoint (DELETE on the review, owner-only, server-derived reviewer identity per Part 11), a service on both clients, and the UI button restored. Also for that work: the given-card approval gate was inconsistent across platforms before removal (mobile `!review.approved` vs web `review.approved`) — reconcile when the feature is built. Not scoped now; logged so the missing capability survives.

---

## 2026-06-01 — Mobile on-device UI/UX findings (batch)

**Repo:** `oglasino-expo` · **Severity:** mixed (per-item below) · **Status:** open
**Detail:** UI/UX and correctness issues Igor observed during an on-device pass on iOS and Android (2026-06-01). Reported in user terms — no mobile code was read, so no `file:line` locations are asserted. Platform is iOS + Android per Igor; per-item platform was not separately specified. Each item carries its own severity and can be resolved independently.

- [x] **(medium)** ~~Owner product page — a **Follow** button appears in the user-info section when viewing your *own* product, and the same Follow button appears on your own user page. Follow should not render on your own profile/products (self-follow). Hide it when the viewed owner is the current user. (Distinct from the stray **Send** button in the same user-info section logged in the 2026-05-31 Ψ batch below.)~~ **Fixed (code) 2026-06-01** (`oglasino-expo` `new-expo-dev`, session `qa-batch-1`). `ProductUserDetails.tsx`: Follow render gated `!userDetails.iamActive` at the call site (single caller; same owner signal as the adjacent Send/Share buttons). Hides on both surfaces. On-device confirmation owed.
- [x] **(medium)** ~~Featured / "more products" section — the section title renders the raw interpolation placeholder: `Jos proizvoda is {value} (10)`. The `{value}` token is not being substituted (the count appears separately as `(10)`). Wire the count into the placeholder, or fix the key/interpolation, so the literal `{value}` is not shown.~~ **Fixed (code) 2026-06-01** (`oglasino-backend` `dev`). Root cause was a backend translation-seed defect, not the mobile call site (mobile passes `{value}` correctly). The `category.products` (EXTRA_PRODUCTS) seed wrapped the placeholder in ICU-escaping single quotes (`'{value}'`) in all four locales, plus a stray extra `}` in EN/RU. Session `category-products-icu-1` stripped the quotes (Igor's call: bare category name, matching the majority no-quote convention) and removed the stray `}`. Verified via intl-messageformat. Takes effect on next DB reset. On-device confirmation owed against a freshly-seeded build.
- [x] **(medium)** ~~Dashboard update-product page — product images are not displayed.~~ **Fixed (code) 2026-06-01** (`oglasino-expo` `new-expo-dev`, product-update-parity). Root cause: `ImagesImport` sized its `expo-image` via NativeWind `className`, but `expo-image` is not `cssInterop`-registered in the project, so dimensions collapsed to 0×0 (NOT a backend/keys problem — keys arrive fine; confirmed because the same images render on the portal product page in the same app). Fixed via inline `style` dimensions. The earlier `imagesData` hydration fix in the same feature also resolved an untouched-save image-wipe data-loss bug (display and submit now share one `imagesData` source; an untouched save no longer sends `imageKeys: []` and wipes all images). On-device confirmation owed.
- [x] **(medium)** ~~Dashboard update-product page — some filters and the category are missing from the form.~~ **Fixed 2026-06-01** via the backend `ProductForUpdateDTO` (owner product-load now returns the three category objects with `labelKey` + nested `filters`; see [decisions.md](decisions.md) 2026-06-01). Web zero-code, verified live by Igor; mobile zero-code (reads fields it already expected) — category labels + the full filter set now populate. On-device confirmation owed for mobile.
- [x] **(medium)** ~~Dashboard update-product page — the whole screen should be compared field-by-field against the web update-product form to find everything missing (the two items above may be a subset). Action: a parity audit of mobile update-product vs. web.~~ **Closed as answered 2026-06-01** (Igor's call). The Phase-2 expo create + update audits and the web scaffold/update audits, plus the create + update parity fixes (images, category + filters, Preview crash / false-banned / scroll, Delete button, three-section layout), substantially answered the field-by-field comparison. On-device Ψ on the individual fixes is still owed and tracked per-item above and in the `state.md` Risk Watch.
- [x] **(low)** ~~User settings — the Danger Zone block should be visually distinct: a red border around the block, and the (delete) button rendered as an outlined button.~~ **Fixed (code) 2026-06-01** (`oglasino-expo` `new-expo-dev`, session `cosmetic-qa-batch-1`). Block wrapped in `rounded-md border border-red-600 p-4`; delete button switched to `variant="outline"` + `border-red-600` + red label. Visual only. On-device confirmation owed.
- [x] **(medium)** ~~Reviews — the reviews UI is rough; needs a UI/UX pass for layout and readability.~~ **Fixed (code) 2026-06-01** (`oglasino-expo` `new-expo-dev`). Three sessions: `cosmetic-qa-batch-1` added inter-card spacing; `dashboard-reviews-buttons` removed the dead given-card action row (a no-op Delete + a blank-label self-report no-op); the reviews-layout session fixed the clipped-last-card (screen root `items-center`→`flex-1`, FlatList `flex:1`) and made cards full-width (`w-full` on both card wrappers; shared `ProductReview` untouched). Web parity: dead given-card Delete also removed from `oglasino-web` (`GivenReviewCard.tsx`, session `given-review-delete-removal-1`). On-device confirmation owed. (Note: re-rated low→medium during the chat — the "rough UI" turned out to include two non-functional buttons and a layout clip, not just spacing.)
- [x] **(medium)** ~~Dashboard product list — activate/deactivate gives no immediate feedback: after tapping deactivate/activate, the product's state in the list does not update until the user pull-to-refreshes. The toggle should reflect immediately (optimistic update or local refetch of the affected row) rather than forcing a manual scroll-to-refresh.~~ **Fixed (code) 2026-06-01 (session bug-batch-3).** Added optimistic row update — on a successful toggle the row's `productState` flips immediately in local list state (ACTIVE/INACTIVE per action), additive to the existing `onRequestProductRefresh` refetch which remains the eventual-consistency reconciler. No flip on failure. On-device confirmation owed (Ψ).
- [x] **(low)** ~~AI generate-description (product name → description) — while generating, the loading overlay is see-through: the content behind it is fully visible, so it doesn't read as a blocking/busy state. The overlay needs an opaque (or dimmed) backdrop so the screen behind is obscured during generation.~~ **Fixed (code) 2026-06-01** (session `cosmetic-qa-batch-1`). `src/components/LoadingOverlay.tsx`: added `bg-black/50` to the transparent backdrop layer. On-device confirmation owed.
- [x] **(medium)** ~~Product creation dialog — region/city is asked even though the user already has a region and city set on their profile; the creation flow lets you pick a new region/city for the product instead of defaulting from the user's saved values. Action: a parity audit of the product **creation** dialog (mobile vs. web) — expo needs to be adjusted to match web. (Companion to the update-product parity item above, which covers the **update** screen.)~~ **Fixed (code) 2026-06-01** (`oglasino-expo` `new-expo-dev`, session `product-create-parity-1`). Region/city is now display-only, sourced from `useAuthStore().user.regionAndCity` (matching web), guarded to render nothing when absent — not collected, not on the wire (was: an editable picker the user filled and the wizard then discarded). On-device confirmation owed.
- [x] **(medium)** ~~Product creation dialog — the dialog title is not visible on step 2 and higher; it renders only on the first step. Across the multi-step flow the user loses the title (and with it the orientation cue for which dialog/step they're on) past step 1. Show the title on every step. (Distinct from the region/city defaulting item directly above — same `ProductCreationDialog`, separate defect.)~~ **Fixed (code) 2026-06-01** (same session `product-create-parity-1`). The dialog title now shows on every non-terminal step; reused the existing `currentStep !== steps.length - 1` predicate (was: step 0 only). On-device confirmation owed.
- [x] **(low)** ~~Empty category — when a category has no products, the empty-state text is not centered.~~ **Fixed (code) 2026-06-01** (session `cosmetic-qa-batch-1`). `catalog/[...categories].tsx`: `text-center` on the empty-state `<Text>` + `px-4`. On-device confirmation owed.
- [x] **(medium)** ~~Catalog filters — the mobile filters are not right and have filters missing versus web. Action: re-validate the expo filters against the web filter set and bring them to parity (which filter types/options render, and which are absent). (Distinct from the update-product-form "some filters and the category are missing" item above, which is about the **update-product** form; this is the **catalog/browse** filtering surface. See also the resolved 2026-05-31 "SELECT-type catalog filter has no dispatch branch" entry below — confirm against the current backend filter contract during the audit.)~~ **Fixed (code) 2026-06-01** — catalog-filters parity pass (`oglasino-expo` `new-expo-dev`, sessions `catalog-filters-2`/`-3`/`-4`, behind a web reference audit + expo gap audit + backend price/random audit + web region/city wire audit). The "filters missing / not right" symptom was primarily the RANGE/DATE serialization bug: catalog range and date filters rendered but dropped `rangeFrom`/`rangeTo` from the request, so they filtered nothing, showed no chip, and added 0 to the count (Finding 1, fixed end-to-end). Also: owner dashboard narrowed to web's set — dynamic per-category filters + show-more gated off, product-state restricted to ACTIVE/INACTIVE, DELETED dropped (Finding 2); active-filter count aligned to web (1 per selected-filter entry) and single-sourced to `getActiveFilterCount` (Finding 3); region/city auto-collapse to region when all cities selected one-by-one (Finding 5). Price-string bounds, random suppression, and the region/city wire shape were all confirmed already correct (no code) — the region/city wire "mismatch" was a retracted false alarm; mobile already sends web's exact `selectedRegionsAndCities: {regions,cities}` object shape. On-device Ψ owed before adopted/mobile-stable. See decisions.md 2026-06-01.

- [x] **(low)** ~~Catalog category page — text-search placeholder shows the full category path. On a category page, search scopes to the current category and its child categories (correct behavior). The placeholder currently reads `Search in {topCategory}/{subCategory}/{finalCategory}`, which overflows and gets cut off — and when the user is in a final (leaf) category the path is clipped mid-string. Change it to show only the current category — `Search in {currentCategory}` — and if even that is too long, truncate with an ellipsis (`…`). Placeholder-text fix only: the search scope is unchanged (still searches the current category plus its children).~~ **Fixed (code) 2026-06-01 (session bug-batch-3).** Placeholder now shows the deepest/current category via the existing `navigation.search.label.one` key instead of the full Top/Sub/Final path; search scope (category IDs) untouched. On-device confirmation owed (Ψ).
- [x] **(medium)** ~~Cold start / reload lands on the notifications screen — reloading the Expo app, or opening it from a killed/dead state, lands on the notifications screen instead of the home/product feed. It should land on the home page where the user sees products. Fix the initial route so a cold launch / reload restores home, not notifications.~~ **Fixed (code) 2026-06-01 (session bug-batch-3).** Not an initial-route issue — the boot push handler (`PushNotificationsInit`) re-fired `router.push('/notifications')` off a stale `getLastNotificationResponse()` and its catch-all `default` branch. Fixed both: dedupe the handled-response identifier (persisted in AsyncStorage, key `LHNR`) so a stale response no longer navigates on cold start/reload, and narrowed the `default` branch so unknown categories no longer blanket-push notifications. Genuine fresh taps still navigate. On-device confirmation owed (Ψ).
- [x] **(low)** ~~Notifications are unstable — visible at one moment, gone the next (intermittent visibility). Per Igor, this is expected to be resolved by the notifications feature, so it is logged here for tracking rather than scoped as standalone bug-fix work.~~ **Fixed (code) 2026-06-02 — notifications feature, expo E1** (flicker root cause: listener re-subscribe on user-object identity + wholesale array replacement; fix subscribes on `firebaseUid`, single live listener, reconciles updates/deletes, cursor/first-page fix); **on-device confirmation done 2026-06-02 (Igor)** — fully closed.

> Related: the open 2026-05-31 update-product crash (`regionAndCity` undefined) entry directly below is a separate defect on the same dashboard update-product screen. Open batch — Igor may append more items.

> **Note 2026-06-05.** All line items in this batch are fixed-in-code (each marked [x]).
> What remains is on-device (Ψ) confirmation, tracked in state.md Risk Watch — NOT open
> bug-fix work. This entry stays open only as a standing container for new on-device finds.

---

## 2026-05-31 — Mobile: update-product screen crashes — `regionAndCity` undefined on `productDetails`

**Repo:** `oglasino-expo` · **Severity:** medium · **Status:** fixed
**Detail:** Opening the dashboard update-product screen throws `TypeError: Cannot read property 'region' of undefined`, blocking the screen from rendering. The render reads `productDetails.regionAndCity.region.labelKey` (and `.city.labelKey` on the next line), but `productDetails.regionAndCity` is `undefined` at render time. Reported by Igor 2026-05-31 from an on-device run; the stack trace he pasted points at `app/owner/dashboard/products/[productId].tsx:333` (`UpdateProductScreen`), propagating up through `app/owner/dashboard/_layout.tsx:11` → `app/owner/_layout.tsx:42` → `app/_layout.tsx:100`. No mobile code was read here — the `file:line` is from the pasted stack trace, not a Docs/QA code read. Likely either `productDetails` arrives without a `regionAndCity` object for some products, or the screen renders before that slot is populated and needs a guard / loading gate before dereferencing `regionAndCity.region`/`.city`. Carrier: a mobile chat — not Docs/QA, and not yet traced to a root cause.

> Resolved (2026-06-01, `oglasino-expo-update-product-region-crash-2`, `new-expo-dev`, code-complete / Ψ-pending). Mobile-only bug — the screen read immutable region/city from `productDetails`, which the backend's `UpdateProductRequestDTO` omits by design (region/city are immutable post-create). Backend confirmed correct as designed; web confirmed sourcing region/city from the authenticated user, display-only. Fixed by sourcing from `useAuthStore().user.regionAndCity` (matching web), display-only, guarded for absent `regionAndCity` / null region / null city (renders nothing when absent). Adjacent `productDetails.imageKeys.map` hardened with `?? []`. tsc/lint/test green; lint baseline held. No backend change. On-device confirmation rides the product-validation Ψ smoke — verify against a product whose owner has, and has not, a region/city set.

---

## 2026-05-31 — Web: `ReportDialog` renders error via unguarded `tErrors()` with no missing-key fallback

**Repo:** `oglasino-web` · **Severity:** low · **Status:** fixed
**Detail:** `ReportDialog.tsx:115` renders the surfaced error via `tErrors(result.errorTranslationKey)` with no missing-key guard — if a backend code ever ships a `translationKey` not seeded web-side, `next-intl` throws / renders the raw key rather than degrading (contrast the `safeT` pattern in `errorMapping.ts`). Latent today: all twelve current `report.*` keys are seeded in all four locales. The existing REVIEW path used the same unguarded render, so guarding one path only would create two patterns. Out of scope for the bug that surfaced it. Surfaced as a Part 4b observation by the web lane (2026-05-31 bug batch).

> Fixed (2026-06-01, `oglasino-web-report-dialog-safet-2`, branch `dev`). `ReportDialog.tsx:114` now gates the dynamic backend-key render on `tErrors.has(result.errorTranslationKey)`; a present-but-unseeded key falls through to the existing generic `tErrors('report.send.fail')`. Uses next-intl's own `t.has`, not `safeT` (whose `=== key` check does not match a namespaced translator's `"ERRORS.<key>"` fallback). One guard covers USER/PRODUCT/REVIEW (shared dialog, identical line). The issue-entry premise that the unguarded call "throws" was corrected by the audit — next-intl logs `console.error` and renders the namespace-prefixed key; no crash.

---

## 2026-05-31 — Mobile Ψ on-device UI findings (batch)

**Repo:** `oglasino-expo` (`new-expo-dev`) · **Severity:** mixed (per-item below) · **Status:** open
**Detail:** UI/UX issues Igor observed during on-device Ψ smoke on iOS and Android (2026-05-31). The surfaces span several features, so this is routed here for triage into the relevant chats rather than bound to one spec. Reported in user terms — no mobile code was read, so no `file:line` locations are asserted. Each item carries its own platform and severity and can be resolved independently.

- [x] **(iOS · low)** ~~Text-search placeholder sits at the bottom of the input box~~ **Fixed 2026-06-01** (session `search-input-ios-clip-1`; device-confirmed).
- [x] **(iOS · medium)** ~~Text-search typed text is not visible while typing~~ **Fixed 2026-06-01** by the same change (device-confirmed): removed the `h-10` + `lineHeight:40` centering hack in `SearchInput.tsx`, replaced with padding-based vertical centering (no fixed-height crop). Was a clip, not a color issue.
- [x] **(iOS · low)** ~~Login (email + password) inputs — the field label renders below the input line~~ **Fixed 2026-06-01** (session `input-label-resting-offset-1`; device-confirmed). `basic/Input.tsx` resting label offset changed `top-7` → `Platform.OS === 'ios' ? 'top-4' : 'top-7'`; `basic/Textarea.tsx` brought to match (`top-7` → `top-5`).
- [x] **(iOS + Android · low)** ~~Base-site selector flag in PortalConfigDialog is not round — clipped top and bottom.~~ **Fixed 2026-06-01** (session `portal-config-flag-round-1`; device-confirmed). Removed `contentFit="contain"` from the flag <Image> in `PortalConfigDialog.tsx` so it uses expo-image's default `cover`, matching the round render in `BaseSiteSelector.tsx`.
- [x] **(iOS + Android · low)** ~~No system-theme (follow-OS) option on mobile — parity gap vs. web's system-theme support. (Handed to a separate Mastermind chat 2026-06-01 — tri-state light/system/dark is a build, not a bug-chat tweak. Tracked there, not here.)~~ **Fixed in code 2026-06-01** (`oglasino-expo-expo-system-theme-2`/`-3`, `new-expo-dev`). Tri-state light/system/dark theme shipped to parity with web; icon-only 3-segment control in PortalConfigDialog, sidebar toggle replaced with a config opener. Code-complete / Ψ-pending — on-device confirmation rides the iOS+Android rebuild. See decisions.md 2026-06-01, features/expo-system-theme.md. No backend work.
- [x] **(iOS + Android · medium)** ~~A stray "Send" button appears in the user-info section of the product page.~~ **Fixed 2026-06-01** (`oglasino-expo`, `new-expo-dev`, session `seller-share-gate-1`; device-confirmed iOS + Android). Diagnosis corrected: a web/mobile parity gap, not a stray element — the seller-block ShareProductButton was gated `!isOnUserPage` (all visitors) where web gates it `iamActive` (owner-only). Re-gated to `userDetails.iamActive` in `ProductUserDetails.tsx`. Visitors see one share button (Actions row); owner retains the seller-block share.
- [x] **(iOS + Android · medium)** ~~Categories page has no back button~~ **Fixed 2026-06-01** (`oglasino-expo`, `new-expo-dev`, sessions `categories-back-button-1` + patch; device-confirmed). Added a back affordance to CatalogScreen via FilteredProductList's HeaderComponent, then extracted a shared `BackButton` component (router.back(), go.back.one.step) reused by the product page, user page (UserProductsList), and categories.
- [x] **(iOS + Android · medium)** ~~Privacy Policy and Terms of Use open old / outdated links~~ **Fixed 2026-06-01** (`oglasino-expo`, `new-expo-dev`, session `legal-link-targets-1`; device-confirmed). The two in-app MarkdownViewer screens fetched stale GitHub raw files (privacy.md / terms.md); re-pointed to the current targets privacy-policy.en.md / terms-of-use.en.md under the same memento-tech/oglasino-platform/refs/heads/main/ path. English-only content is the platform-wide state (blocked on lawyer review, 2026-05-27) — not a mobile bug.

> Open batch — Igor may append more items as the Ψ smoke continues.

> **Note 2026-06-05.** All line items fixed-in-code; most device-confirmed inline. This
> entry stays open only as a standing container for further Ψ-smoke finds, not as open
> bug-fix work.

---

## 2026-05-31 — Mobile: active base sites fetched on every request, uncached

**Repo:** `oglasino-expo` · **Severity:** medium · **Status:** fixed
**Detail:** The active-base-sites lookup is called a lot on mobile — apparently on every request — with no caching. Needs both a cache and an investigation into why it fires that often (which call sites trigger it, and whether the result can be memoized/persisted like the other AsyncStorage-backed boot data). Reported by Igor 2026-05-31; not yet traced to specific `file:line`s — a Phase 2 audit should confirm the call count and the trigger paths before scoping a fix. Likely lands in or alongside chat **I** (backend-calls optimization), which is the natural carrier for redundant-call reduction.

> **Resolved 2026-05-31 (chat I, `backend-calls-reduction-mobile`).** Two parts. (1) Backend: the per-request uncached `findActiveBaseSiteCodes` Postgres hit (the real cost, paid by all clients via `BaseSiteFilter`) is closed — now served from a no-TTL Redis cache `redisActiveBaseSiteCodes`, warmed at boot by `CacheWarmupService`, evictable via `CacheAdminController` (session `oglasino-backend-backend-calls-reduction-2`). (2) Mobile: the "fetched on every request" symptom was not reproduced on `new-expo-dev` — the request interceptor reads `selectedBaseSite` from the cached bootStore slot and only sets the `X-Base-Site` header; no `GET /public/baseSite/*` fires per request, and the legacy `fetchBaseSites` list call is dead code. The original observation was likely Next-cache behavior or an older build. `VersionController` / `VersionChecksumService` keep reading the codes uncached on purpose (checksum-build paths need fresh DB state).

---

## 2026-05-31 — Web: app-store / play-store badges are dead links (icons only)

**Repo:** `oglasino-web` · **Severity:** medium · **Status:** open
**Detail:** The mobile-app links on web render only their icons and do nothing on click — the user is not taken to the App Store or Google Play. Either the anchors carry no (or a broken) store URL, or the badge images are not wrapped in working links. Reported by Igor 2026-05-31. Fix: wire the badges to the real App Store / Play Store listing URLs (and confirm they open correctly on desktop and mobile web). Carrier: a web chat — not Docs/QA, and not in the Expo queue.

---

## 2026-05-31 — Spec §15 (user-deletion translations) drifted from the as-built backend seed

**Repo:** `oglasino-docs` (`features/user-deletion.md` §15)
**Severity:** low (docs-only; mobile + web were built against the seed, so behavior is correct)
**Status:** resolved 2026-05-31 (reconciled in the user-deletion close-out Docs/QA pass)

§15 diverged from the backend translation seed (the source frontends fetch at boot) in five ways: post-deletion dialog keys listed under DASHBOARD_PAGES but seeded under DIALOG; `buttons.account.deleted.go.home.label` listed but the seeded button is `account.deleted.acknowledge.label` ("OK"); `user.banned.label` (COMMON) and `user.banned.notice` (MESSAGES_PAGE) seeded but omitted from §15; and systemically, §15 wrote keys with namespace prefixes while the seed stores them leaf-only. Same staleness class as the corrected sessionStorage and ERRORS-prefix drift. Confirmed against all four locale seed files during user-deletion mobile Briefs 3–4.

One key remains pending verification: `errors.user.not.pending.deletion` (`USER_NOT_PENDING_DELETION`, §15.6) was deliberately left prefixed rather than stripped on assumption — Docs/QA does not read the backend seed, and the brief directed not to change it without confirmation. It has zero mobile impact (the defensive 400 on a non-pending restore; mobile's restoration is the `firebase-sync` header path that never surfaces it). Strip to the seeded leaf once a backend read confirms the form. Noted inline in §15.6.

---

## 2026-05-31 — RESOLVED — Banned-dialog close-button key carried a stray `buttons.` prefix

**Repo:** `oglasino-expo` (`new-expo-dev`)
**Severity:** low (latent; cosmetic raw-key render on a never-device-verified dialog)
**Found:** during user-deletion mobile adoption Brief 3 (seed verification).
**Fixed:** user-deletion mobile adoption Brief 4 (one-line).

`AccountStateDialogsInit.tsx` (the Φ1 `accountBanned` slot) passed `tButtons('buttons.banned.go.home.label')`; the seeded BUTTONS key is un-prefixed `banned.go.home.label` (confirmed 4/4 locales). On-device this would have rendered a raw key / fallen through to the InfoDialog default close label. Corrected to the seeded key. No new key; no seed change. Pre-existing from Φ1, never device-verified (Φ1 is Ψ-pending). Caught before user-deletion Ψ.

---

## 2026-05-31 — Mobile: new-expo-dev carries a large uncommitted change set

**Repo:** oglasino-expo (new-expo-dev) · **Severity:** medium · **Status:** wontfix
**Detail:** All Expo work since the branch point (admin removal, Φ chats, chats A–D,
etc.) is uncommitted on `new-expo-dev` (~220 files in `git diff --stat HEAD`). With no
commit boundary, per-session integrity checks cannot diff one session's changes against
a baseline — a chat-D verification session confirmed it could not bind ~217 files to
specific sessions. Recommend committing/staging the work as reviewable units soon; the
risk compounds as the branch grows.

> **Closed 2026-06-04 (prod-bug-sweep).** Process / commit-boundary item, owned by Igor — not a tracked bug. Closed.

---

## 2026-05-31 — Mobile: SELECT-type catalog filter has no dispatch branch

**Repo:** oglasino-expo (new-expo-dev) · **Severity:** low · **Status:** resolved (deferred — no SELECT in contract)
**Found in:** `src/components/dialog/dialogs/FiltersDialog.tsx` — `getAppropriateFilter`
handles MULTI_OPTION / SINGLE_OPTION / RANGE / DATE, but not SELECT; a SELECT-typed
filter falls through to the "filter not found" fallback (now a safe `<View>`/`<Text>`
after the B5 fix, no longer a crash). `SelectFilter` is imported in the file but used
only for the dashboard product-state dropdown, not wired into `getAppropriateFilter`.
Building the SELECT branch is feature work, deferred. **Unknown whether the backend
catalog actually emits a SELECT filter type today** — if it does not, this is latent
and harmless; if it does, those filters render no usable control. Verify against the
backend catalog before prioritizing.

> **Resolved — deferred, no `SELECT` in the contract (2026-05-31 bug batch, expo lane, verification only, branch `new-expo-dev`).** The open question is now answered: there is no `SELECT` member in either the app `FilterType.ts` or the backend `FilterType.java` enum — both carry only `SINGLE_OPTION` / `MULTI_OPTION` / `RANGE` / `DATE`. A SELECT dispatch branch would wire to a contract value that cannot be emitted today, so it is deferred until the contract adds `SELECT`; the `getAppropriateFilter` fallback is harmless and unreachable. Zero code change.

---

## 2026-05-31 — Latent crash: `FilteredProductList` calls `.trim()` on a possibly-undefined `searchText`

**Repo:** `oglasino-expo` (`new-expo-dev`)
**Severity:** medium (latent; masked today by `tsconfig` `strict: false`)
**Scope:** product filtering — NOT user deletion. Out of scope for the user-deletion mobile adoption; logged here so it survives.
**Surfaced by:** product-filtering verification (`oglasino-expo/.agent/verify-product-filtering-session3.md`); re-confirmed incidentally during the user-deletion current-state audit.

**Location:** `src/components/product/FilteredProductList.tsx:104` —
`const suppressRandom = searchText.trim() !== '' || selectedOrder !== undefined;`
`searchText` is destructured from `useFilterStore` with no default; the store default is `undefined` (`src/lib/store/useFilterStore.ts:44`; type `searchText?: string`).

**Symptom:** on any first render where no search term has been typed (`searchText === undefined`, the store default), `fetchPageInternal` would throw `TypeError: undefined is not an object (evaluating 'searchText.trim')`. This affects the catalog screen and every other `FilteredProductList` consumer on cold load.

**Why `tsc` misses it:** `tsconfig.json` has `strict: false`, so `strictNullChecks` is off and `(string | undefined).trim()` is not a compile error. The repo's `tsc --noEmit` passes clean.

**Recommended fix (own session):** `(searchText ?? '').trim()` or `searchText?.trim()` in the `suppressRandom` expression. Confirm on-device whether the catalog actually crashes on cold load (an upstream default not visible in the audit may mask it).

**Status:** fixed (verified already-guarded — see resolution note)

> **Fixed — already guarded (2026-05-31 bug batch, expo lane, verification only, branch `new-expo-dev`).** The `(searchText ?? '')` guard is present at `FilteredProductList.tsx:104`; the unguarded form described in the report above was never committed — verified against HEAD, the suppression block was authored with the guard already in place. Closed as already-guarded; no code change.

---

## 2026-05-31 — Messaging spec §3.6 admin view-token override is not implemented in backend

**Severity:** medium
**Status:** open
**Found in:** `oglasino-backend` `DefaultImageTokensFacade.issueViewToken` (`images/facade/impl/DefaultImageTokensFacade.java:138-171`); surface is backend + web admin (NOT mobile).
**Detail:** [`features/messaging.md`](features/messaging.md) §3.6 states admins can mint view-tokens for arbitrary chats. The backend does not implement this. `DefaultImageTokensFacade.issueViewToken` unconditionally calls `verifyChatMembership(auth.getFirebaseUid(), request.chatId())` — there is no role/admin branch, and `isUserMemberOfChat` (`DefaultFirebaseChatService.java`) has no admin short-circuit. A sweep of the `admin/` package found no view-token / `signViewToken` / `chatPrefix` usage. Result: an admin who is not a participant of `chats/{chatId}.users` is denied with 403 `NOT_CHAT_MEMBER`, so admins cannot view chat images for chats they are not party to.

The surface is **backend + web admin only** — mobile is consumer-only here and carries no admin code (admin was removed from `oglasino-expo` in chat α), so the mobile messaging adoption does not touch this. Surfaced by the messaging backend spec-validation (`oglasino-backend/.agent/validate-messaging-backend.md`, claim 5, 2026-05-30).

**Decision needed (separate Mastermind triage):** either the spec is aspirational and §3.6 should be corrected to match the as-built no-admin-override reality, or admin chat-image viewing is a required surface and a backend brief is owed to add the admin path. Not resolved here — §3.6 is deliberately left in the spec untouched pending that triage.

---

## 2026-05-30 — Backend `allowPhoneCalling` write-path ghost

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-backend` `DefaultUserFacade.updateCurrentUserData`; `/secure/user/update` request DTO + GET response.
**Detail:** `allowPhoneCalling` is accepted on the `/secure/user/update` request DTO and returned on the user GET, but `DefaultUserFacade.updateCurrentUserData` never calls `setAllowPhoneCalling` — the value is silently dropped on save, so the toggle never persists. Route to a backend chat. Surfaced by the consent-mode-mobile backend audit (2026-05-30). Does not block consent-mode-mobile (that chat does not touch this toggle).

> **Fixed (2026-05-31 bug batch, backend lane `oglasino-backend-allowphonecalling-admin-preauth-1`, branch `dev`).** `setAllowPhoneCalling` wired into `DefaultUserFacade.updateCurrentUserData`; the toggle now persists on save.

---

## 2026-05-30 — Web `allowPreferenceCookies` dead type fields

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-web` `src/lib/types/.../AuthUserDTO.ts`, `UpdateUserDTO.ts`.
**Detail:** `oglasino-web` still declares `allowPreferenceCookies?: boolean` on `AuthUserDTO` and `UpdateUserDTO`, although the [consent-mode-v2](features/consent-mode-v2.md) spec says the field was removed end-to-end and the firebase-sync wire never sends it. Dead type members; delete after confirming the backend no longer emits/accepts the field (the consent-mode-mobile backend audit confirmed backend removed it end-to-end). Surfaced by the consent-mode-mobile web audit (2026-05-30).

> **Fixed (2026-05-31 bug batch, web lane, branch `dev`).** Both `allowPreferenceCookies?: boolean` members deleted from `AuthUserDTO` and `UpdateUserDTO`; grep-confirmed dead (no remaining references).

---

## 2026-05-30 — Web consent spec-vs-code drifts (docs)

**Severity:** low
**Status:** fixed
**Found in:** `features/consent-mode-v2.md` (vs the as-built `oglasino-web` code).
**Detail:** Minor inaccuracies in [`features/consent-mode-v2.md`](features/consent-mode-v2.md) against the as-built web code, surfaced by the consent-mode-mobile web audit (2026-05-30):
- the client `gtag('consent','update',...)` snippet is documented as listing `preference`, but the code sends only the four Google signals (`analytics_storage`, `ad_storage`, `ad_user_data`, `ad_personalization`) — no `preference`;
- the `globalCookie.cookieConsent → og_consent` migration shim is documented but was never built (superseded by cookies-closing);
- the settings route is `/owner/cookies`, not `/owner/user`, in two places.
A Docs/QA correction pass on the web spec — not the consent-mode-mobile chat's scope.

> Fixed (2026-06-01, Docs/QA bug-chat close-out). All three drifts re-confirmed against current `oglasino-web` code and corrected in `features/consent-mode-v2.md`: (1) the `gtag('consent','update',...)` snippet now lists only the four Google signals, no `preference`; (2) the `globalCookie.cookieConsent → og_consent` migration-shim documentation removed — it was never built (superseded by cookies-closing); (3) cookie-settings route corrected to `/owner/cookies`, not `/owner/user`, in both places. The bug-chat draft also listed a *fourth* drift — that the spec's implementation-file pointers (`ConsentMode.tsx` / `consentSignals.ts` / `consentCookie.ts` / `consentState.ts`) were stale — but Docs/QA found those filenames are **not** in the spec and never were (`grep` + `git log -S` confirm); the spec already references the correct `src/lib/consent/*` paths. That "drift" was phantom `Read` content (see the `state.md` Risk Watch tool-reliability row), so there was nothing to correct.

---

## 2026-05-30 — Mobile HEIC stage label was English-only for non-EN locales (image-pipeline V6)

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-expo/src/lib/images/errorMapping.ts`; stage emitted at `oglasino-expo/src/components/.../AvatarUpload.tsx:71`.
**Detail:** `errorMapping.ts` built the translation key `image.processing.converting-heic` (hyphen) for the HEIC-conversion stage, but the seeded key is `image.processing.converting_heic` (underscore). The lookup missed and fell back to a hard-coded English string, so SR/RU/CNR users saw "Converting HEIC…" in English even though a localized seed exists. The `converting-heic` stage is actively emitted at `AvatarUpload.tsx:71`.

> **Fixed** in `oglasino-expo-image-pipeline-4` (2026-05-30) via a `STAGE_KEY_OVERRIDES` entry (`'converting-heic' → 'converting_heic'`, mirroring the existing `uploading`/`complete` overrides) + a dedicated override-resolution test. Tests 109 → 110. See [decisions.md](decisions.md) 2026-05-30 entry.

---

## 2026-05-30 — Mobile orphan cleanup missing on avatar + chat save-failure (image-pipeline V9)

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-expo/app/owner/dashboard/user.tsx` (avatar); `oglasino-expo/src/lib/store/useActiveChatStore.ts` (`sendMessage`, chat).
**Detail:** The Layer-1 client-initiated `cleanupOrphanImages` DELETE was wired for the product and review surfaces but not for the avatar or chat surfaces. On an avatar `updateUser` failure (throw or falsy result) or a chat send failure, the already-uploaded R2 key was never DELETE'd — the orphan lingered until the backend sweeper backstop (which for `private/chats/` is only Sundays, ≥30 days old).

> **Fixed** in `oglasino-expo-image-pipeline-4` (2026-05-30): `cleanupOrphanImages` wired into both the avatar save-failure path (throw + falsy-result, guarded so an unchanged existing avatar key is never deleted) and the chat send-failure catch, matching the product/review fire-and-forget pattern. See [decisions.md](decisions.md) 2026-05-30 entry.

---

## 2026-05-30 — Web likely has the same HEIC stage-label localization miss (cross-repo)

**Severity:** medium
**Status:** resolved (no bug — verified)
**Found in:** `oglasino-web` `errorMapping.ts` (not verified — out of the mobile chat's scope).
**Detail:** The mobile image-pipeline validation flagged that `oglasino-web` appears to use the same hyphenated `converting-heic` stage value, so web may have the identical silent-English-fallback bug that the mobile V6 fix (above) closed. This was NOT verified in web — the mobile validation chat does not read web code. A future web chat (or bug chat) should check `oglasino-web`'s `errorMapping.ts` (and the stage emitter) for the same hyphen-vs-underscore mismatch against the seeded `image.processing.converting_heic` key, and apply the equivalent override fix if confirmed.

> **Resolved — no bug (2026-05-31 bug batch, web lane, branch `dev`).** Verified `oglasino-web` uses the underscore `converting_heic` form end-to-end (stage value and seeded key match); no hyphen-vs-underscore mismatch exists, so the silent-English-fallback class the mobile V6 fix closed does not occur web-side. No fix needed.

---

## 2026-05-30 — Mobile dead code: `ProductReviewImageImport.tsx` + `app/__smoke__/upload.tsx`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-expo/src/components/dialog/components/ProductReviewImageImport.tsx`; `oglasino-expo/app/__smoke__/upload.tsx`.
**Detail:** The image-pipeline current-state audit found `ProductReviewImageImport.tsx` is never imported — the live review dialog uses `ImagesImport` — and it carries a `//TODO TODO` plus hardcoded Serbian strings. `app/__smoke__/upload.tsx` is a self-marked-for-deletion smoke harness. Both are deletion candidates for the next Expo chat that touches image surfaces (likely the Ω structural sweep). Not deleted in the conformance-fix session — out of its scope.

> Fixed in chat H session `oglasino-expo-image-pipeline-5` (2026-05-31, `new-expo-dev`). Both files deleted; the two by-name comment references (`ImageStatusOverlay.tsx`, `uploadProgress.ts`) and the smoke-harness comment pointer (`uploadPrimitive.ts:5`) updated. No dangling references remain (grep-confirmed: `ProductReviewImageImport`, `__smoke__`, `UploadSmokeScreen` all zero hits across `src/` and `app/`).

---

## 2026-05-30 — Duplicate `isPngInput` in mobile `processImage.ts` and `uploadImages.ts`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-expo/src/lib/images/processImage.ts`; `oglasino-expo/src/lib/images/uploadImages.ts`.
**Detail:** `isPngInput` is duplicated byte-for-byte across the two files. The token-time `Content-Type` ↔ PUT-time `Content-Type` equality (Worker-enforced) silently depends on the two copies staying in sync; a future edit to one only could desync the pairing and cause Worker rejections. Consolidation candidate for the Ω structural sweep. Surfaced by the image-pipeline current-state audit / deep validation.

> Fixed in chat H session `oglasino-expo-image-pipeline-5` (2026-05-31, `new-expo-dev`). `isPngInput` now exported from `processImage.ts` and imported into `uploadImages.ts`; the duplicate deleted. No import cycle (uploadImages already depended on processImage). Both call sites resolve to the single definition; the `uploadImages.test.ts` processImage mock updated to provide it. tsc clean, 325 tests pass, lint baseline (80 warnings) held.

---

## 2026-05-30 — No method-security behavioral test coverage for admin `@PreAuthorize` gates in `oglasino-backend`

**Severity:** low
**Status:** fixed (representative — scope caveat below)
**Found in:** `oglasino-backend` security configuration / admin endpoints (surfaced by the expo-maintenance-split engineer sessions when the maintenance toggle moved to `POST /api/secure/admin/maintenance/toggle` with `@PreAuthorize("hasRole('ADMIN')")`).
**Detail:** Admin `@PreAuthorize` gates are verified only by annotation presence plus manual check — there is no method-security behavioral test (e.g. `@WithMockUser` / Spring Security test) asserting that a non-admin caller actually receives 403 and an admin caller passes. Annotation-only verification means a future refactor that disables global method security, or a typo in a role string, would not be caught by the suite. Pre-existing gap, broader than maintenance; logged for a future backend hardening pass.

> **Fixed — representative coverage (2026-05-31 bug batch, backend lane `oglasino-backend-allowphonecalling-admin-preauth-1`, branch `dev`).** `MaintenanceAdminControllerPreAuthorizeTest` added — a method-security behavioral test asserting a non-admin caller receives 403 and an admin caller passes on the maintenance toggle endpoint. **Scope caveat preserved:** one representative admin endpoint is now behaviorally covered; the ~13 other admin controllers remain annotation-presence-verified only. The broader hardening pass (behavioral coverage across all admin gates) is not closed by this representative test.

---

## 2026-05-30 — Backend docs 01/02/04/06/11 still reference the single `maintenance.active` KV key

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/docs/` (files 01, 02, 04, 06, 11).
**Detail:** After the two-flag split (`maintenance.web.active` + `maintenance.backend.active`), several `oglasino-backend/docs/` pages still describe the single `maintenance.active` KV key and the old maintenance model. Per conventions Part 1, no new docs are added under `<repo>/docs/`, but existing ones may be edited; these five are stale and should be corrected or pointed at the live runbook [`infra/cloudflare/maintenance.md`](infra/cloudflare/maintenance.md) when the backend repo is next touched. Cross-repo — not a Docs/QA edit (Docs/QA may not write backend source/docs); routed here for the backend engineer agent.

> FIXED 2026-06-01 (`oglasino-backend-docs-maintenance-staleness-2`). All stale `maintenance.active` refs in 01/02/04/06/11 corrected to the two-flag model and traced to `docs/12-deployment.md` (already migrated) + [`infra/cloudflare/maintenance.md`](infra/cloudflare/maintenance.md). 02 additionally corrected: :154 key inventory (now four keys) and the restore model (Phase 7 rehearsal + Phase 10 reflect the 3-step manual restore). 04 was the single :445 fallback curl. `grep -rn maintenance.active docs/` → none. Separately, the `06-architecture.md` KV-role sentence was corrected (2026-06-01) to state the backend both reads AND writes KV (admin maintenance toggle); the bug-4 hypothesis that `CLOUDFLARE_KV_CONFIG_TOKEN`'s write scope was stale was checked and found FALSE — 11:136 "reads/writes" is accurate (backend runs a live KV-write path via `DefaultCloudflareKvService.toggleMaintenance`), left unchanged.

---

## 2026-05-30 — Router `maintenanceResponse` HTML (frontend branch) does not set `X-Robots-Tag: noindex` on stage

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-router` worker `maintenanceResponse` (frontend/HTML branch).
**Detail:** The worker's frontend maintenance branch does not set `X-Robots-Tag: noindex` on the stage 503 response, so a crawler hitting stage during maintenance could index the maintenance page. Pre-existing and explicitly out of scope for expo-maintenance-split (which touches the API/mobile composition, not the HTML branch's headers). Related to the `MAINTENANCE_ORIGIN` 503 missing-`X-Robots-Tag` observation in [`features/seo-foundation.md`](features/seo-foundation.md). Logged for a future router/SEO pass.

> **Fixed (2026-05-31 bug batch, router lane, branch `stage`).** The stage HTML 503 now carries the full `NOINDEX_HEADER`, gated by the same `isStage` mechanism as `forwardToOrigin`; prod responses unchanged. Test coverage added.

---

## 2026-05-29 — Mobile's backend `/public/maintenance/active` check is redundant with the edge worker

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-expo` cold-start boot path (the `GET /public/maintenance/active` boot request, one of the four documented in the 2026-05-28 Φ3 cold-start entry below); `oglasino-router` (the Cloudflare Worker that owns maintenance state).
**Detail:** Mobile polls the backend's `/public/maintenance/active` endpoint at boot to decide whether to show the maintenance screen. Per conventions Part 8, the Cloudflare router worker is the edge boundary — "Maintenance state, admin bypass, and origin forwarding live there." Every mobile API call already passes through that worker. A separate backend maintenance check is therefore architecturally redundant: maintenance is an edge concern, not a backend concern.

The check also can't do what it appears to do. Mobile is fully backend-dependent — if the backend is down, the app doesn't function regardless of the maintenance flag. So a backend-served maintenance endpoint cannot signal "we are in maintenance" in the one scenario (origin down) where the edge would. The worker is the only layer positioned to gate maintenance independently of origin liveness.

**Direction (not a decision):** let the Cloudflare router worker be the single source of maintenance truth for mobile, the same way it is the edge boundary for web, and drop or rework the dedicated backend `/maintenance/active` poll on the mobile boot path. This is cross-repo (router + mobile) and touches the boot sequence, so it is a design decision for a Mastermind / mobile-adoption chat, not a docs change.

Surfaced by Igor directly (2026-05-29). Recorded here for triage; no engineering scoped yet.

> Resolved by expo-maintenance-split — the dedicated backend `/public/maintenance/active` poll is removed; mobile maintenance is composed at the edge worker (`maintenance.backend.active`, or a failed `/actuator/health/readiness` probe) and surfaced via the `X-Oglasino-Maintenance` 503 header.

> **Status corrected 2026-06-05.** Missed flip: the body's own resolution sub-note
> (expo-maintenance-split removed the dedicated backend /maintenance/active poll; mobile
> maintenance now composed at the edge worker) means this is resolved. Flipping open → fixed
> to match the body. No new work.

---

## 2026-05-28 — Review-reporting wire is an unfinished feature; web sends review id in product-id slot

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-web/src/components/owner/reviews/ReceivedReviewCard.tsx:18` (web wire); `oglasino-backend` report-submit service (no REVIEW branch).
**Detail:** The owner's received-reviews list mounts a `ReportButton` with `reportType: ReportType.REVIEW` and passes the review id into the `reportedProductId` request slot. The backend's report-submit endpoint (`POST /api/secure/report/add`), hardened on 2026-05-28 (see `decisions.md` 2026-05-28 "Report-submit trust-boundary closed"), only handles USER and PRODUCT branches. There is no `reportedReviewId` field on the request DTO, no `REPORTED_REVIEW_ID_REQUIRED` error code, no REVIEW branch in `DefaultReportService`, and no REVIEW handling in the self-report or existence-check logic.

**Runtime behaviour (corrected 2026-05-29):** `ReportType` had only `PRODUCT` and `USER` — no `REVIEW`. Web sent the literal `"REVIEW"`; with no lenient-enum Jackson config and no `HttpMessageNotReadableException` handler, the request failed deserialization → catch-all → HTTP 500 `INTERNAL_ERROR`. The PRODUCT existence check was never reached; there was neither a misfile nor a `REPORTED_PRODUCT_NOT_FOUND` 422. The earlier misfile/422 description assumed a code path that did not exist.

Surfaced by the W6 audit (`oglasino-web-report-submit-mapping-audit-1`, 2026-05-28) and confirmed against the backend `ReportErrorCode` enum and `DefaultReportService`. Pre-existing — the wire was added when review-reporting was scoped but the backend half was never built.

**Why parked (not fixed in W6):** the honest fix is a feature, not a bug. It requires a backend design decision (do we want review-reporting at all? what does "report a review" mean — abusive content, fake review, retaliatory review?), a new `reportedReviewId` field on the DTO, a REVIEW branch in the service with its own existence check against the reviews table, a new error code (`REPORTED_REVIEW_ID_REQUIRED` or similar), self-report logic, and per-target dedupe key extension. Cross-repo work, design surface, feature-sized — explicitly outside bug-chat scope per `meta/bug-chat-bootstrap.md` Section 10. Igor's call (2026-05-28 W6 chat): unfinished feature, not bug-chat work.

**Three possibilities for the eventual resolution** (decision deferred to Mastermind):

- (a) **Build it.** Mastermind feature chat: design REVIEW report semantics, scope backend + web work, ship. The web wire stays in place until backend lands.
- (b) **Remove the wire.** Web-only deletion of the `REVIEW` `ReportButton` mount in `ReceivedReviewCard.tsx`. Decision: review-reporting is not a launch feature.
- (c) **Repurpose the wire** to report the product the review is about (`reportType: PRODUCT`, `reportedProductId: review.product.id`). Web-only one-line fix. Possible if the original intent was "this review is on a sketchy product" rather than "this review itself is sketchy."

Picking between (a), (b), and (c) is a product decision, not an engineering call.

**Reachability:** the Report button is visible to any logged-in user viewing their received reviews. A real user can press it today. Severity is medium rather than high because the outcome is either a silent mis-file (data quality issue, not user harm) or a confusing error message (UX issue, not blocking).

> **Resolved 2026-05-29** by the `review-reports` feature (option (a) — build it). Backend gained `ReportType.REVIEW`, a validated `reportedReviewId`, a REVIEW service branch (existence + reviewer-only self-report, server-side), dedupe extension, and two new `ReportErrorCode` constants + seeds. Web swapped the `ReceivedReviewCard` mount to `reportedReviewId` and surfaces the new codes as translated messages. Code-complete on backend (`dev`) + web (`stage`); see `decisions.md` 2026-05-29 and `features/review-reports.md`.

---

## 2026-05-28 — Φ3 cold-start boot-loading hang (`config` and `baseSite/details` never reach backend)

**Severity:** high (blocks Φ3 ship)
**Status:** fixed
**Found in:** `oglasino-expo` cold-start path; `src/lib/config/api.ts`'s request interceptor; the three bootstrap service files (`configurationService.tsx`, `baseSiteService.tsx`, `maintenanceService.tsx`).
**Detail:** On first run / cleared storage, the app hangs on the loading spinner: bootstrap's `Promise.all([checkIfMaintenance, getAppConfiguration, fetchBaseSites])` does not resolve because two of the four boot-time requests — `GET /public/config` and `GET /public/baseSite/details` — never reach the backend, while `GET /public/maintenance/active` and `GET /public/app/version/android` consistently do.

**Established facts:** (a) both failing endpoints return 200 quickly via curl from a laptop AND via the phone's own browser, ruling out backend, payload size, and network reachability; (b) the app's axios request interceptor passes all four requests through correctly (`_bootstrap: true` honored, barrier not blocking — confirmed via `[BOOT]` instrumentation); (c) backend is not under load; (d) the two failing endpoints are the two large-payload responses, the two succeeding are tiny; (e) the distinguishing variable is that the app fires four requests concurrently at cold start whereas the browser fetches one at a time.

**Leading hypothesis at handoff:** a client-side concurrency / transport behavior specific to the four-parallel cold-start burst, NOT app bootstrap logic and NOT the boot refactor (the refactor's barrier, render gate, and cycle break are all verified functioning).

**Owner:** resolved (see fix note below). See [decisions.md](decisions.md) 2026-05-28 Entry C.

> See expo-maintenance-split (the dedicated `GET /public/maintenance/active` poll is removed; the worker's `X-Oglasino-Maintenance` signal replaces it on the mobile boot path).

> **Fixed — missed status flip (recorded 2026-05-31 bug batch).** Resolved 2026-05-29 by the `expo-boot-redesign` feature: the `bootStore` gate state machine replaced `AppContext.bootstrap`, eliminating the four-parallel cold-start request burst that triggered the hang; real-device regression passed. This was a missed flip — the fix landed under a separate feature, not as new bug-chat work. Cross-reference the [decisions.md](decisions.md) 2026-05-29 `expo-boot-redesign` entry and the `state.md` Risk Watch row already marked RESOLVED 2026-05-29. The related 2026-05-28 "[BOOT] diagnostic instrumentation" entry was already `fixed` by the same redesign close (no action there).

---

## 2026-05-28 — Live `[BOOT]` diagnostic instrumentation in `api.ts` request interceptor

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-expo/src/lib/config/api.ts` request interceptor.
**Detail:** `console.log('[BOOT]', ...)` lines were added during Φ3 Briefs 14–19 diagnostic exploration of the cold-start boot-loading issue and intentionally left in place at chat close so the resolving Mastermind chat retains the diagnostic signal. The resolving chat must remove these `[BOOT]` log lines as part of its fix. Flagged here so the cleanup is not forgotten when the boot-loading hang is resolved.

> **Fix (2026-05-29, `expo-boot-redesign` feature close):** The `[BOOT]` interceptor log was removed in step 7a.1, and all remaining boot diagnostics (`[BOOT-START]`, `[MOUNT-EFFECT]`, `[OVERLAY]`, and the three `fetchNamespace` debug logs caught as an adjacent observation in 7b) were removed across steps 7b and 7c. The boot/HTTP layer was replaced wholesale by the `bootStore` gate state machine. See [decisions.md](decisions.md) 2026-05-29 entry.

---

## 2026-05-28 — Chat-store require-cycle triangle (Cycle B) deferred to chat B

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-expo` chat-store module triangle (surfaced during Φ3 Brief 17 boot-refactor design audit).
**Detail:** A require cycle in the chat-store module triangle was identified during Φ3's boot-refactor design audit. Deliberately deferred to chat B (mobile messaging adoption), which already owns the chat-store reshape per `features/expo-performance-foundation.md` §2 (D9.x messaging divergences). Benign at runtime — all cross-module calls go through dynamic `getState()` lookups, not module-evaluation-time imports. No load-order failure today; structural untidiness only. Φ3 closed Cycle A (`api.ts ↔ authStore ↔ authService`) via Brief 18; Cycle B was left in place to avoid touching code chat B will reshape anyway.

> **Fixed** in `oglasino-expo-messaging-adoption-4` (2026-05-31, branch `new-expo-dev`). On `new-expo-dev` the cycle was a 2-node dyad (`useActiveChatStore ↔ useChatNavStore`), **not** a triangle — `useChatListStore`/`useChatBlockStore` are leaves (each imports only `./authStore`). Broken by removing the top-level static `import { useActiveChatStore }` from `useChatNavStore.ts` and resolving the store lazily on the narrower edge — a function-scoped `require('./useActiveChatStore') as typeof import('./useActiveChatStore')`, with a scoped `// eslint-disable-next-line @typescript-eslint/no-require-imports` — inside `setTempReceiver`. Zero behavior change: every runtime `getState()` crossing is unchanged. `tsc` exit 0; 325 tests pass; lint baseline (80 warnings / 0 errors) held via the scoped suppression. The surviving `useActiveChatStore.ts → useChatNavStore` static import is non-cyclic.

---

## 2026-05-28 — Always-mounted `<Stack>` exposes pre-bootstrap API calls (root cause)

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-expo` `app/_layout.tsx` (always-mounted `<Stack>` per the 2026-05-27 Φ2 decision); `AppContext` bootstrap cascade in `src/components/context/AppContext.tsx`.
**Detail:** Φ2 (2026-05-27) adopted the always-mounted `<Stack>` pattern because expo-router cannot conditionally render a navigator. The side effect: child routes (notably the product feed) can mount before `AppContext` bootstrap finishes setting the base-site/language codes, firing API calls with null `X-Base-Site`/`X-Lang` headers. Φ3 mitigated this by (1) adding a codes-derived `globalThis`-pinned bootstrap barrier in `apiStore.ts` (Brief 18) and (2) the `<RequireBaseSite>` per-screen render gate (Briefs 19, 19a). The underlying architecture remains vulnerable: any future boot-time mount-phase API caller is exposed unless it either gates its own render or routes through the barrier. Residual structural risk; not closed by Φ3.

> **Resolved (2026-05-29, `expo-boot-redesign` feature close):** The two Φ3 mitigations this entry described no longer exist — the `apiStore` `globalThis` barrier and `<RequireBaseSite>` were both deleted by the boot redesign. The structural risk is now governed differently: a single Stack-level mount gate at `app/_layout.tsx` mounts the portal only when `bootStatus ∈ {ready, updating}` (amendment #6 in the [decisions.md](decisions.md) 2026-05-29 entry), and `bootStore` populates the base-site/language/config slots before `ready`. Child routes therefore cannot mount before the codes exist, so the pre-bootstrap null-header API call is structurally precluded rather than mitigated after the fact. The one residual mount→status-write path (the product deep-link base-site auto-switch) is `ready`-guarded and idempotent. Closed by the redesign.

---

## 2026-05-28 — `setState`-during-render in `Filters.tsx` and `FiltersDialog.tsx`

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-expo/src/components/navigation/Filters.tsx:86-90`, `oglasino-expo/src/components/navigation/FiltersDialog.tsx:87-91`. Surfaced during Φ3 Brief 2 (F8 React.memo work) as an adjacent observation.
**Detail:** Both files run a `setState` call inside the render body (not wrapped in an effect or callback). React tolerates this but produces a runtime warning and can cause extra render passes / lost updates under StrictMode. Pre-existing — not introduced by Φ3. Out of scope for Φ3 (which targeted memoization, not render-time control-flow). Tracked for Ω structural sweep.

> **Fixed (2026-05-31, oglasino-expo-product-filtering-3).** Both render-time setState
> patterns (`setIsFreeCategory` inside a `useMemo` body, and the render-body `forEach`
> store-prune) removed from the live filter dialog. Path correction: the cited paths are
> stale — `src/components/navigation/Filters.tsx` has since been deleted (dead code, zero
> importers), and `navigation/FiltersDialog.tsx` never existed. The live filter dialog is
> `src/components/dialog/dialogs/FiltersDialog.tsx`, where the fix landed.

---

## 2026-05-28 — `console.error` calls in `ProductList.tsx` should use the project logger

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-expo/src/components/product/ProductList.tsx:93`, `:186`. Surfaced during Φ3 Brief 16 (boot-loading diagnostic exploration) as an adjacent observation.
**Detail:** Two `console.error` call sites violate the codebase's logger conventions (project standard is `logServiceError` / `logServiceWarn`). Pre-existing and harmless at runtime. Tracked for Ω structural cleanup.

> **Fixed (2026-05-31, oglasino-expo filters-deadcode-console-cleanup).** Both
> `ProductList.tsx` sites (`loadNextPage` and `onRefreshCurrent` catch blocks) and the
> sibling `HorizontalExtraProductsListView.tsx:37` site now call `logServiceError`
> (`src/lib/utils/serviceLog.ts`) instead of `console.error`. tsc/lint/tests green.

---

## 2026-05-28 — Unused `ONE_MIN` constant in `AppVersionConfigInit.tsx`

**Severity:** low
**Status:** fixed (obsolete — host file deleted)
**Found in:** `oglasino-expo/src/components/init/AppVersionConfigInit.tsx:27`. Surfaced during Φ3 Brief 16 as an adjacent observation.
**Detail:** `ONE_MIN` is declared but never referenced anywhere in the file or imported elsewhere. Dead constant; safe to delete. Tracked for Ω structural cleanup.

> **Fixed — host file deleted (2026-05-31 bug batch, expo lane, verification only).** The host file `AppVersionConfigInit.tsx` was deleted by the `expo-boot-redesign` feature (2026-05-29, see the decisions.md "Deleted from the old path" list); `ONE_MIN` no longer exists anywhere in the tracked tree. The issue was real when filed (2026-05-28, Φ3 Brief 16) and was incidentally resolved by a separate feature, not by this batch. Cross-reference the expo-boot-redesign deletion.

---

## 2026-05-28 — Product view count always 0 across the portal (increment never fires)

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`, `oglasino-web/src/lib/service/nextCalls/productService.ts` (`markAsSeen`), `oglasino-web/src/components/client/NumberOfViews.tsx`.
**Detail:** The product-page view count shows 0 on every product. Two cooperating causes were diagnosed, one fixed, one NOT yet resolved:

1. **Display guard hid real zeros — FIXED.** `NumberOfViews` rendered nothing on `<= 0`, masking new products and failure paths. Fixed in session `oglasino-web-isfollowing-and-views-fix-1`: guard relaxed to `< 0`, `hasFetched` flag added (no flash), `getNumberOfViews` returns `null` on failure to distinguish a real 0 from a fetch failure.

2. **Increment (`/public/product/seen/<id>`) never reaches the backend — NOT FIXED.** `markAsSeen` was an un-awaited async call inside the product RSC render (`page.tsx:95`); Next 15/React 19 do not guarantee fire-and-forget side effects flush before the request lifecycle ends. Fix attempt in session `oglasino-web-markasseen-after-fix-1` wrapped the call in `after()` from `next/server` and added a local log-on-failure to `markAsSeen`. **This fix FAILED manual smoke (2026-05-28): `/seen` is still not observable on the backend (no request, no `Product.numberOfViews` movement) nor producing any web-side log line.** The `after()` callback's delivery is unconfirmed.

**Backend confirmed clean:** read-only check `oglasino-backend-view-counter-check-1` verified `iamActive` defaults to `false` for anonymous (gate fires), `/seen` and `/views` touch the same counter (DB base + Redis delta), increment persists, owner-exclusion is server-side and correct. The backend works *if `/seen` arrives*. It is not arriving.

**Open hypotheses for the next Mastermind (in priority order):**
- (a) `after()` callback is not executing at all (verify with diagnostic logging at the top of the `after` callback and top of `markAsSeen`).
- (b) The `after()` callback runs but the fetch inside is dropped/failing silently — the log-on-failure added this session should now catch this; check for a `logServiceWarn('product.next.markAsSeen', ...)` line.
- (c) **The gate is now suppressing the call.** After the #8 fix (session `oglasino-web-isfollowing-and-views-fix-1`) dropped `skipAuth` on `getUserForId`, `owner.iamActive` became authoritative. If it now resolves `true` for the viewing population (e.g. owner lookup wrong, or auth-forwarding makes the backend mis-set it), the gate `if (!owner.iamActive)` is false and `markAsSeen` is correctly skipped — for everyone. This is a NEW possibility introduced by the #8 fix landing and is the most likely regression to check first.
- (d) `NEXT_PUBLIC_API_URL` / base-URL resolution issue specific to this call (unlikely — other `FETCH_BACKEND_API` calls on the page work).

**Recommended first step:** a read-only diagnostic-instrumentation pass (temp logs at: before the gate showing `owner.iamActive`'s value; inside the `after()` callback; at the top of `markAsSeen`; and confirm on the backend whether `/seen` arrives). Read the output, pin the break point, revert the logs. Precedent: the Φ2 root-cause method (decisions.md 2026-05-27 — "code-reading + runtime instrumentation beat theorizing").

**Related sessions (archived in `sessions/`):** `oglasino-web-views-not-displaying-audit-1`, `oglasino-web-markasseen-never-fires-audit-1`, `oglasino-backend-view-counter-check-1`, `oglasino-web-isfollowing-and-views-fix-1`, `oglasino-web-markasseen-after-fix-1`.

> **Fixed (2026-05-28, stage-confirmed).** Root cause was NOT hypothesis (c). The `owner.iamActive` gate is correct — the backend audit (`oglasino-backend-iamactive-auth-forward-audit-1`) proved `iamActive` is `currentUserId.equals(dto.getId())`, which is `false` for the normal viewing population (anonymous + logged-in non-owners) both before and after the #8 auth-forward change; the deep web audit confirmed this at runtime. Two real walls caused the symptom:
>
> 1. **`cookies()` inside `after()` (web).** The `after()`-wrapped `markAsSeen` chained through `request()`'s `await cookies()` at `fetchApi.ts:39`, which Next 15+ forbids inside an `after()` callback — it threw before `fetch()`, the throw was swallowed by `markAsSeen`'s try/catch, so `/seen` never left the Next process. Fixed by adding `skipAuth: true` to the `markAsSeen` call (`src/lib/service/nextCalls/productService.ts`), bypassing the `cookies()` hop. Owner exclusion does not depend on auth on this call — enforced by the web gate and independently by the backend at `DefaultProductSeenService` against `product.owner.id` (see decisions.md 2026-05-28 skipAuth-reversal entry).
> 2. **`LANG_MISSING_OR_INVALID` 400 (backend).** Once `/seen` dispatched, it hit the `CurrentLanguageFilter` 400 rule (2026-05-14) because `/api/public/product/seen/*` was not in the language-independent allowlist and `markAsSeen` sends no `X-Lang`. Fixed by adding the route to `CurrentLanguageFilter`'s allowlist prefixes — a view-count ping has no language dependence, so allowlisting is the correct fix.
>
> Also fixed on the same path: the owner-id Redis cache read used `Long.getLong` (reads JVM system properties) instead of `Long.parseLong`, silently defeating the cache; corrected so the owner lookup is cached as intended.
>
> A second, related defect — `/seen` counting on every refresh — was found and fixed in the same chat (see decisions.md 2026-05-28 view-count dedup entry). Stage smoke confirmed: single viewer refreshing counts once; a second viewer on a different network counts again.
>
> Related sessions (archived in `sessions/`): the iamActive auth-forward audit, the deep web audit, the allowlist validate-then-fix, the markAsSeen skipAuth fix, the Long::parseLong owner-cache fix, the dedup audit, the dedup fix, the ttl-keys audit, and the cleanup session.

---

## 2026-05-27 — Per-locale legal markdown content (privacy + terms)

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/privacy/page.tsx`, `oglasino-web/app/[locale]/(portal)/(public)/terms/page.tsx`.
**Detail:** Both pages hardcode the English-only URLs (`privacy-policy.en.md`, `terms-of-use.en.md`) from `memento-tech/oglasino-platform`. Per-locale legal content is blocked on lawyer review of Serbian translations.

**Swap shape (when Serbian files exist):**
```tsx
const locale = await getRoutingLocale();
const { oglasinoLocale: lang } = getTenantLocale(locale);
const effectiveLang = lang === 'sr' || lang === 'cnr' ? 'sr' : 'en';
const url = `https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.${effectiveLang}.md`;
```

**Required files in `memento-tech/oglasino-platform`:**
- `privacy-policy.sr.md`
- `terms-of-use.sr.md`

**CNR decision:** Conventions Part 9 says Montenegrin (cnr) aliases to Serbian. If that extends to legal content, `cnr` maps to `sr` and no separate CNR files are needed.

**RU decision:** Russian (`ru`) currently falls back to English — no Russian legal drafts exist. If Russian legal content is desired, add `privacy-policy.ru.md` and `terms-of-use.ru.md` and extend the ternary.

**Dependency:** lawyer review of legal drafts (tracked in `state.md` under "Privacy Policy + Terms (drafts)" — status `drafted`).

---

## 2026-05-27 — `AdminTranslationsController.refreshTranslationsCache` returns empty body

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/.../AdminTranslationsController.java`.
**Detail:** The endpoint evicts all translation caches successfully but provides no response body. After Brief 11's web migration, the endpoint has zero web callers but is retained as backend ground-truth — accessible via curl or similar manual admin tooling. If retained, consider returning a small status response (e.g., `{ "evicted": true, "cachesCleared": N }`) for operational visibility. Surfaced in version-checksums Brief 4.

> **Fix:** The endpoint was deleted in session `oglasino-backend-admin-translations-controller-cleanup-1` (2026-05-27) rather than given a response body. It had zero callers after Brief 11's web migration to the unified admin cache page. Its supporting methods (`indexTranslations`, `evictAllTranslationCaches`) were also deleted. The `/update` endpoint in the same controller was changed from returning `204 No Content` to returning `200 OK` with body `{ "updated": true }` in the same session.

---

## 2026-05-27 — Circular dependency `DefaultTranslationService` ↔ `VersionChecksumService`

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend/.../DefaultTranslationService.java`, `oglasino-backend/.../VersionChecksumService.java`.
**Detail:** `DefaultTranslationService` uses `@Autowired` field injection for `VersionChecksumService` with `@Lazy` after a runtime boot failure during Brief 6 integration. `VersionChecksumService` uses constructor injection for `TranslationService`. The cycle is broken by `@Lazy`'s deferred resolution. If `DefaultTranslationService` ever migrates to full constructor injection (the modern Spring pattern), the cycle would need a different resolution (e.g., `@Lazy` on the `TranslationService` constructor parameter in `VersionChecksumService`, or restructuring the service boundaries). Surfaced and resolved post-Brief 6.

> **Documentation note (2026-05-28, session `oglasino-backend-report-trust-boundary-fix-1`):** A `// NOTE:` comment was added at `DefaultTranslationService:35` (the `@Lazy VersionChecksumService` field) documenting why the field is `@Lazy` and the caveat that a future constructor-injection migration must re-resolve the cycle (e.g. via `@Lazy` on the constructor parameter on the other side). Code change is zero-behavior; the note exists so a refactor doesn't accidentally break the bean graph.

---

## 2026-05-27 — No DOM test environment in `oglasino-web`

**Severity:** low
**Status:** parked
**Found in:** `oglasino-web` project configuration.
**Detail:** The project has no `@testing-library/react`, `jsdom`, or equivalent. Component-level tests (render, click, toast assertions) are not possible; service-layer tests cover backend interaction logic only. Adding DOM infrastructure would expand test coverage but is its own scope decision. Surfaced in version-checksums Brief 8.

> **Parked 2026-06-04.** Own infra decision, not a bug — adding a DOM test stack is a scope choice, not a defect.

---

## 2026-05-27 — `params` as `useEffect` dep in `TranslationsTable`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/components/admin/translations/TranslationsTable.tsx:48`.
**Detail:** `params` is a new object each render, so the effect re-fires on every render. The server-component boundary in `page.tsx` mitigates this in practice but the structural fragility remains. Surfaced in version-checksums Brief 8.

> **Fix:** Replaced `params` object dep with individual primitive fields (`params.namespace`, `params.lang`, `params.key`, `params.value`, `params.tenant`) in session `oglasino-web-admin-translations-cleanups-1` (2026-05-27). Closes the structural fragility that was masked by the server-component boundary.

---

## 2026-05-27 — `PagingDTO` inheritance dead weight on `TranslationsFiltersRequest`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/translations/types/TranslationsFiltersRequest.ts`.
**Detail:** `TranslationsFiltersRequest` extends `PagingDTO` with `page` and `perPage` fields, but `getFilteredTranslations()` never passes them to the backend. The backend returns all matching rows. No runtime impact; misleading type. Surfaced in version-checksums Brief 8.

> **Fix:** Removed `extends PagingDTO` from `TranslationsFiltersRequest` and dropped `page`/`perPage` from the construction site in `page.tsx` plus the test fixture in session `oglasino-web-admin-translations-cleanups-1` (2026-05-27). The `PER_PAGE` constant was also removed. No consumer read `page` or `perPage`; backend never accepted them.

---

## 2026-05-27 — Dead `base-site:details` cache tag button in `CacheEvictionPanel`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/components/admin/cache/CacheEvictionPanel.tsx:103`.
**Detail:** The button still calls `evictCacheTag('base-site:details')` after Brief 7 removed the last producer of that tag. Clicking the button succeeds (no-op eviction) but misleads the admin. Surfaced in version-checksums Brief 11.

> **Fix:** Button deleted in session `oglasino-web-admin-translations-cleanups-1` (2026-05-27). Zero producers of the `base-site:details` cache tag existed in the repo.

---

## 2026-05-27 — Nested `tAdmin(tAdmin('svi korisnici'))` double-translation bug in `CacheEvictionPanel`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/components/admin/cache/CacheEvictionPanel.tsx:182`.
**Detail:** The inner call translates the literal string `'svi korisnici'` (not a translation key — it's a Serbian phrase), then the outer call tries to translate the result. Likely produces incorrect output in non-Serbian locales. Surfaced in version-checksums Brief 11.

> **Fix:** Replaced with `tAdmin('cache.label.all_users')` in session `oglasino-web-admin-translations-cleanups-1` (2026-05-27), matching the pattern of the four sibling buttons. The translation key was already seeded across all four locales.

---

## 2026-05-27 — `AppContext.Provider` value not memoized

**Severity:** low
**Status:** fixed (2026-05-27, Φ3 Brief 5 — F10 AppContext split)
**Found in:** `oglasino-expo/src/components/context/AppContext.tsx:247-255`.
**Detail:** `AppContext.Provider` value creates a new object on every render because `setBaseSiteForCode`, `setLanguageForCode`, and `getConfiguration` are regular functions (not `useCallback`). Every `useAppContext()` consumer re-renders on every AppContextProvider re-render. Structural audit finding F10. Φ3 scope — fix lands with the AppContext value memoization work. Surfaced as adjacent observation during the Φ2 refetch-loop investigation session (2026-05-27).

> **Fix:** Φ3 Brief 5 (F10) split `AppContext` into `AppStateContext` and `AppActionsContext` per spec S3, and memoized the actions value via `useMemo`. State-only consumers no longer re-render when actions identity changes; action-only consumers no longer re-render when state changes. Closes the audit's F10 finding. See archived session `sessions/2026-05-27-oglasino-expo-phi3-5.md`.

---

## 2026-05-27 — `configurationService.tsx` return type contract misleading

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-expo/src/lib/services/configurationService.tsx:6`.
**Detail:** Declares return type `Promise<ConfigMap>` but returns `null` on failure (lines 15, 17). The `.catch(() => undefined)` wrapper in `bootstrap()` doesn't catch non-thrown `null` returns. The `!configuration` check treats both `null` and `undefined` as failure, which is correct at runtime, but the type contract is misleading — callers see `Promise<ConfigMap>` when the actual return type is `Promise<ConfigMap | null>`. Surfaced as adjacent observation during the Φ2 refetch-loop investigation session (2026-05-27).

> **Fixed (code) 2026-06-01 (session bug-batch-3).** Annotation corrected to `Promise<ConfigMap | null>`; caller already `?? {}`-guards. Type-only.

---

## 2026-05-25 — Redundant per-tenant translation cache entries

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/translations/lib/translationsCache.ts:40` (sends `X-Base-Site`), `:85-92` (caches per `(namespace, lang, tenant)`); `oglasino-backend/.../controller/TranslationsController.java` (endpoint ignores the header).
**Detail:** Web caches translations per `(namespace, lang, tenant)`; backend translation endpoint takes only `(namespace, lang)` and ignores `X-Base-Site`. Three cache entries hold identical data for the three tenants. Memory/storage cost only, no correctness impact. The redundancy is an artifact of the `X-Base-Site` header being sent uniformly by all API calls. Surfaced by the 2026-05-25 image-alt-translation base-site audit.

> **Fix:** Removed `tenant` parameter from `fetchNamespaceTranslations`, `getNamespaceTranslations`, `loadAllNamespaces`, and caller `request.ts` in session `oglasino-web-tier2-batch-1` (2026-05-28). Removed the `X-Base-Site` header from the translations fetch call (backend `TranslationsController` ignores it). Cache key reduced from `(namespace, lang, tenant)` to `(namespace, lang)`, eliminating 132 redundant entries (220 → 88). Updated stale comment in `revalidate/route.ts`.

---

## 2026-05-25 — About page content duplication across base sites

**Severity:** low
**Status:** wontfix
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/about/page.tsx`, `oglasino-web/src/metadata/generateAboutMetadata.ts`.
**Detail:** `/rs-sr/about` and `/rsmoto-sr/about` render byte-for-byte identical content (same language, same translations, same hero image). Google may flag these as near-duplicate pages despite separate hreflang clusters. Accepted by the SEO foundation feature ([decisions.md](decisions.md) 2026-05-24 entry on `base-site-scoped` hreflang mode). If it causes crawl-budget or ranking problems post-launch, consider consolidating to a shared about page with `all-locales` hreflang. Surfaced by the 2026-05-25 image-alt-translation base-site audit.

> **Wontfix (Igor, 2026-06-04)** — only the language differs and the content is correct as-is; no change is possible without altering SEO behavior, and the SEO-foundation feature already accepted this (decisions 2026-05-24). Closing as a recorded SEO decision, revisit via Search Console post-launch if it ever bites.

---

## 2026-05-25 — Hardcoded `rs-sr` in intro page SearchAction urlTemplate

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/metadata/generateIntroPageStructuredData.ts:35`.
**Detail:** Schema.org SearchAction `urlTemplate` is `${basicMetadata.baseUrl}/rs-sr/catalog?searchText={search_term_string}` — always points at the Serbian variant of the catalog page regardless of which base site or locale the user is on. The intro page is locale-agnostic (locale-selector), so there's no "current locale" to use, but if Google generates a sitelinks search box from this SearchAction, users land on `rs-sr` even after picking a different locale. Surfaced by the 2026-05-25 image-alt-translation audit 1.

> **Fix:** Explanatory comments added in session `oglasino-web-round-2-fixes-1` (2026-05-27) to both `generateIntroPageStructuredData.ts` (above the urlTemplate) and `generateIntroPageMetadata.ts` (above the matching OG locale hardcoding). The intro page is locale-agnostic (locale-selector landing page) so the SearchAction urlTemplate and OG locale must commit to a single default; Serbian is >90% of traffic per the SEO foundation decision (decisions.md 2026-05-24). Comments cross-reference each other so future audits don't re-surface either site separately.

---

## 2026-05-25 — Intro page `'Oglasino'` title and `© Memento Tech` copyright footer remain hardcoded English

**Severity:** low
**Status:** `wontfix`
**Found in:** `oglasino-web/src/metadata/generateIntroPageMetadata.ts:16` (title), `oglasino-web/app/page.tsx:81` (copyright footer).
**Detail:** Both intentionally hardcoded. The `'Oglasino'` title is the brand name used as the locale-selector page's `<title>`, `openGraph.title`, and `twitter.title` — an intentional editorial choice. The `© {year} Memento Tech` copyright footer is a universally non-translated pattern. Recorded as `wontfix` so future audits don't re-surface them. Surfaced by the 2026-05-25 image-alt-translation audit 1.

---

## 2026-05-25 — Auth listener null-path doesn't run full store cleanup

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-expo/src/lib/store/authStore.ts` (the `initAuthListener` null-path branch).
**Detail:** Φ1 added two code paths that call `auth.signOut()` outside of `authStore.logout()` (F5's axios 403 interceptor and F4's foreground re-validation listener). When sign-out is triggered from these paths, the `onIdTokenChanged(null)` listener fires and sets `user: null` but does NOT run the full user-scoped store cleanup (chat, favorites, notifications, portal filter, dashboard filter) that `authStore.logout()` performs. Stores retain stale data until the next `authStore.logout()` call or app restart.

Severity is low because: (1) the user is immediately navigated to the public surface on sign-out, so stale store data is invisible; (2) any subsequent logout-the-action call (manual user logout) clears the stores; (3) the next sign-in's `syncUserToBackend` cascade overwrites stale user state.

Surfaced by the Φ1 Brief 7 engineer (F5 — 401/403 interceptor) as a Part 4b adjacent observation. Fix path: either add the full store cleanup to the listener's null-path branch (mirroring `authStore.logout()`'s cleanup sequence), or route interceptor sign-outs through `authStore.logout()` instead of calling `auth.signOut()` directly. Engineer audits and picks at Ω-chat time.

> **Fixed 2026-06-04 (prod-bug-sweep, `new-expo-dev`).** Extracted a module-level `clearUserScopedStores()` helper (store-clear steps only — NOT `set({user:null})`, NOT `logoutFirebase()`, which would re-fire `onIdTokenChanged(null)` and recurse), called from both `logout()` and the `onIdTokenChanged(null)` listener branch. Interceptor / foreground sign-outs now clear stale chat / favorites / notifications / filter state. 515 tests pass. See `sessions/2026-06-04-oglasino-expo-prod-bug-sweep-1.md`.

## 2026-05-25 — `oglasino-expo` has two Firebase auth listeners on overlapping events

**Severity:** low
**Status:** parked
**Found in:** `oglasino-expo/src/lib/store/authStore.ts` (`initAuthListener` uses `onIdTokenChanged`) and `oglasino-expo/src/components/init/InitFavoritesStore.ts` (uses `listenAuthState`, which wraps `onAuthStateChanged`).
**Detail:** Φ1 Brief 4A switched the main auth listener from `onAuthStateChanged` to `onIdTokenChanged` to match web's pattern (fires on token rotation in addition to sign-in/out). `InitFavoritesStore` was correctly left untouched in Brief 3 (F22 — hydration race) because its Firebase-native listener doesn't have the Zustand hydration race. Result: two Firebase auth listeners run simultaneously, firing on overlapping events with different timing for token rotations.

Benign today — the favorites listener doesn't call `syncUserToBackend`, so the duplication produces no redundant backend traffic. Structurally untidy: a single listener with both responsibilities (auth-state monitoring for `syncUserToBackend` + favorites init) would be cleaner.

Surfaced by the Φ1 Brief 4A engineer as a Part 4b adjacent observation. Fix path: consolidate to one listener — either move favorites init into the main `initAuthListener` callback, or have favorites subscribe to `useAuthStore` after `_hasHydrated` instead of running its own Firebase listener. Engineer audits and picks at Ω-chat time.

> **Parked 2026-06-04 (prod-bug-sweep, Ω / post-prod).** Benign today — the favorites listener doesn't call `syncUserToBackend`; the events touch disjoint state. Consolidating now risks a load-order regression pre-launch. Recategorized as tidiness / Ω cleanup, not a launch blocker.

## 2026-05-25 — Circular module dependency in oglasino-expo auth wiring

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-expo/src/lib/config/api.ts` → `src/lib/store/authStore.ts` → `src/lib/services/authService.ts` → `src/lib/config/api.ts`.
**Detail:** Φ1 Brief 4A introduced this cycle when adding `useAuthStore.getState().setRestored(true)` to the axios response interceptor in `api.ts`. All cross-module accesses are inside callbacks (runtime only, not module-evaluation time), so the cycle works without runtime issue today. Tests confirm; 109 passing throughout Φ1.

The cycle is fragile to future refactors. If a future engineer adds a module-evaluation-time access (e.g., `const cached = useAuthStore.getState()` at module scope inside `authStore.ts`), the cycle becomes load-order-dependent and could fail at app boot.

Surfaced by the Φ1 Brief 4A engineer as a known structural issue with explicit "dynamic only, no current runtime issue" framing. Fix path: extract the axios instance to a leaf module that neither `authStore` nor `authService` imports from. Engineer audits and picks at Ω-chat time.

> **Closed 2026-06-04 (prod-bug-sweep) — already resolved.** The reported cycle no longer exists: `api.ts` no longer imports `authStore`; the DI-hooks pattern (`authInterceptors.ts`) already broke it. The issue description was stale against current code. No action needed.

## 2026-05-23 — Create path missing `PRICE_REQUIRED` enforcement

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java:125` (verbatim store), `:686` (`populateCategories` free-zone-only overwrite), `:825` (`updatePriceCurrency` throw, update-path only). Surfaced by the 2026-05-23 read-only backend audit ([`sessions/2026-05-23-oglasino-backend-price-required-create-gap-1.md`](sessions/2026-05-23-oglasino-backend-price-required-create-gap-1.md)).
**Detail:** `DefaultProductService.createProduct` stores `request.getPrice()` verbatim at line 125. `populateCategories` (line 686) overwrites to `BigDecimal.ZERO` for free-zone categories but does not validate price for non-free-zone categories. The `PRICE_REQUIRED` throw at line 825 lives in `updatePriceCurrency`, which is called only from the update path. A client that bypasses the web Zod check at `oglasino-web/.../productValidator.ts:76–82` (mobile pre-adoption, curl, Postman, future flows) can create a product with `price = null, free = false`.

Today's web wizard's Zod check is the only gate; nothing broken has reached the server through that surface. Whether any product was created with `price = null, free = false` historically is unverified — would require a DB query.

**Trust-boundary verdict:** clean. The gap is a validation gap, not a trust-boundary violation. `price` is not used in moderation, authorization, or state-transition decisions.

**Recommended fix (from the audit):** Option B — guard in `createProduct` between the `topCategory` resolve and the `populateCategories` call. ≤6 lines, one new test, no DTO change, no wire change, no translation change. Matches the update path's enforcement shape and the spec's 422 status code for `PRICE_REQUIRED`. Likely lands on `feature/validation-refactor` where the surrounding work has shipped.

**Two adjacent observations recorded in the audit**, both downstream of this gap and both moot for new products once Option B lands; only relevant for historical malformed products if any exist:

- `DefaultProductService.updatePriceCurrency:823` silently accepts null price on non-free-zone updates (the `if (source.getPrice() != null)` guard is intentional for partial updates, but means a malformed product cannot be repaired through the update path).
- `DefaultProductService.isNoOpUpdate:327` treats two nulls as equal — a product persisted in the gap state remains persisted in the gap state on update with no validation surface.

> **Fix (initial guard, 2026-05-25, commit `4c2c5e4`):** Guard added at `DefaultProductService.java:159-162`. Non-free-zone products now require `price != null` on the create path, throwing `PRICE_REQUIRED` on failure.
>
> **Threshold alignment (2026-05-27, session `oglasino-backend-create-path-test-coverage-1`):** Comparison style changed from `doubleValue() <= 0` to `compareTo(BigDecimal.ONE) < 0`, matching the update path's threshold exactly. Both paths now enforce `price != null && price >= 1` for non-free-zone products. See `decisions.md` 2026-05-27 entry for the alignment rationale. 16 new unit tests added in `DefaultProductServiceCreateTest.java` covering PRICE_REQUIRED, USER_SETUP_INCOMPLETE, CATEGORY_NOT_FOUND, CURRENCY_NOT_ALLOWED, free-zone behavior, image-key filtering, and state/moderation defaults. Backend test suite: 641 passing.

---

## 2026-05-22 — Theme switcher should support 'system' option

**Severity:** low
**Status:** fixed (2026-05-27, session oglasino-web-theme-cookie-persistence-2)
**Found in:** `oglasino-web/src/components/client/buttons/ToggleButton.tsx`. Surfaced by the cookies-closing Mastermind chat closing pass (2026-05-22).
**Detail:** Today the toggle in PortalConfigDialog is bi-state (light/dark); the underlying `next-themes` provider supports tri-state (light/dark/system). Adding 'system' to the UI is a separate UI decision unrelated to the cookie persistence work in cookies-closing Item 4. The future theme handoff (Path A, see [`.agent/handoffs/theme-path-a.md`](.agent/handoffs/theme-path-a.md)) replaces `next-themes` entirely — at that point the 'system' option can be folded in or kept out as a UI decision separate from the persistence work.

> **Fix:** the theme-cookie-persistence feature (2026-05-27) shipped a tri-state segmented toggle (Sun/Monitor/Moon) in `PortalConfigDialog`. The `'system'` option resolves to the current OS `prefers-color-scheme` via `window.matchMedia` and updates live when the OS preference changes via the `SyncThemeFromSystem` component. `GlobalCookie.theme` was widened to `'light' | 'dark' | 'system'` to support persisting the choice when preference consent is granted.

---

## 2026-05-22 — Product page `getBaseSiteServer()` null-guard latent issue

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`. Surfaced during cookies-closing Item 4 Brief 3b.
**Detail:** `getBaseSiteServer()` returns `BaseSiteDTO | null`, but the product page accesses `baseSite.code` without a null check. Would throw `TypeError` if `getBaseSiteServer()` returns null (backend down, fetch error). Rare in practice. Pre-existing; not introduced by the cookies-closing work.

> **Fix:** `if (!baseSite) return {};` added at top of `generateMetadata` and `if (!baseSite) notFound();` added at top of `ProductPage` in session `oglasino-web-tier1-batch-1` (2026-05-27). Guards match the parent `[locale]/layout.tsx` pattern.

---

## 2026-05-22 — `generatePrivacyPageMetadata.ts` and `generateTermsPageMetadata.ts` pass compound routing locale as JSON-LD `inLanguage`

**Severity:** low
**Status:** fixed (2026-05-24, SEO foundation feature — brief 1 already used `localeToBcp47` for inLanguage; brief 5g deleted these JSON-LD documents entirely, making the issue moot)
**Found in:** `oglasino-web/src/metadata/generatePrivacyPageMetadata.ts`, `oglasino-web/src/metadata/generateTermsPageMetadata.ts`. Surfaced during cookies-closing Item 4 Brief 3d.
**Detail:** Both metadata generators pass the compound routing locale (`me-cnr`) as the JSON-LD `inLanguage` value. JSON-LD `inLanguage` expects BCP-47; `me-cnr` is not BCP-47. SEO consumers may ignore or fall back. Pre-existing. Fix: use `getTenantLocale(locale).locale` to produce `cnr-ME` (or equivalent BCP-47 tag).

---

## 2026-05-22 — `ForegroundPushInit.tsx` notification URL handling assumes unprefixed paths

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/components/client/ForegroundPushInit.tsx`. Surfaced during cookies-closing Item 4 Brief 3i.
**Detail:** Migrated to the wrapped `useRouter` from `@/src/i18n/navigation` based on backend payload contract inspection. If the backend ever emits prefixed `data.navigate` URLs (e.g., already locale-prefixed), the wrapper will double-prefix and produce a broken navigation. Fix lives at the backend payload shape, not in the web UI.

> **Fix:** New `stripRoutingLocale(href)` helper at `src/lib/utils/stripRoutingLocale.ts` strips any leading routing-locale segment before `router.push`. Applied at both `data.navigate` call sites (`ForegroundPushInit.tsx` and `notificationActions.ts`) in session `oglasino-web-tier1-batch-1` (2026-05-27). Locally-constructed URLs (`/notifications`, `/messages`, `/product/...`) are unaffected — they don't start with a locale segment.

---

## 2026-05-22 — `pathnameWithoutLocale(locale)` dead-check in protected layout

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/app/[locale]/(portal)/(protected)/layout.tsx`. Surfaced during cookies-closing Item 4 Brief 3d.
**Detail:** The layout calls `pathnameWithoutLocale(locale)` expecting a pathname; passes a locale string instead. The check never matches, so the favicon-skip branch is dead code. Pre-existing; not introduced by the cookies-closing work.

> **Fix:** Dead favicon-skip branch deleted in session `oglasino-web-tier1-batch-1` (2026-05-27). `pathnameWithoutLocale` import removed. Function definition in `utils.ts` retained (zero callers remain; separate cleanup decision).

---

## 2026-05-22 — `PortalConfigDialog.navigate` is declared async but never awaits

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/components/popups/dialogs/PortalConfigDialog.tsx`. Surfaced during cookies-closing Item 4.
**Detail:** The `navigate` helper is declared `async` but contains no `await` and is never awaited at call sites. No-op keyword; misleading to readers. Could be removed in any future PortalConfigDialog work.

> **Fix:** `async` keyword removed in session `oglasino-web-round-2-fixes-1` (2026-05-27). No callers depended on Promise return; body had no `await`.

---

## 2026-05-22 — `getPathname` exported from `src/i18n/navigation.ts` but unused

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/i18n/navigation.ts`. Surfaced during cookies-closing Item 4 navigation rewrite.
**Detail:** Was unused before the navigation wrapper rewrite; remains unused after. Could be removed in a future cleanup. Kept export-symmetric with `createNavigation`'s shape for now.

> **Fix:** `getPathname` deleted in session `oglasino-web-round-2-fixes-1` (2026-05-27). `redirect` was deleted in the same session as a fold-in — it had zero external callers and was the sole consumer of `getPathname`. The `Link`, `useRouter`, and `usePathname` re-exports stay. The forward-looking-API-completeness comment block was deleted alongside.

---

## 2026-05-22 — `AppInit`'s `setLocale(locale)` uses empty deps array

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/components/client/AppInit.tsx`. Surfaced during cookies-closing Item 4 Brief 3d.
**Detail:** `AppInit` invokes `setLocale(locale)` to populate `storedLocale` in `api.ts` but the effect's deps array is empty. If tenant or language changes without unmounting `AppInit`, `storedLocale` stays at the original value. Pre-existing; the typical mount lifecycle (AppInit lives at the app root) means it usually only runs once per navigation entry, masking the gap.

> **Fix:** `locale` added to deps array in session `oglasino-web-round-2-fixes-1` (2026-05-27). `setLocale` mutates a module-scope variable only, no infinite loop risk. The bug was masked by `[locale]` layout remount; fix closes the structural gap.

---

## 2026-05-22 — Two-redirect chain when cookie's lang is valid and URL's lang is invalid for tenant

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/proxy.ts`. Surfaced during cookies-closing Item 4 Brief 3e-bis.
**Detail:** Visiting `/rs-cnr/...` with `cookie.lang='ru'` produces a two-hop redirect: `proxy.ts`'s invalid-locale branch redirects to `/rs-sr/...` first (tenant default), then cookie-wins redirects to `/rs-ru/...`. Each redirect is correct in isolation. Could be optimized to a single redirect by making the invalid-locale fallback cookie-aware (prefer the cookie's language if the tenant supports it, else fall back to tenant default).

> **Fix:** Hoisted the cookie parse into a single `cookieLangIfValidForTenant` computation at the top of `proxy.ts`'s `if (match)` block in session `oglasino-web-batch1-1` (2026-05-28). The invalid-locale fallback now uses `cookieLangIfValidForTenant ?? [...allowed][0]` as the redirect target, and the cookie-wins branch fires only when `cookieLangIfValidForTenant && cookieLangIfValidForTenant !== urlLang`. The previous duplicate `JSON.parse` / `try/catch` in the cookie-wins branch is consolidated into the single hoisted parse. `/rs-cnr/...` with `cookie.lang='ru'` now collapses to a single redirect to `/rs-ru/...`. The four regression cases (URL-invalid no-cookie → tenant default; URL-invalid + valid cookie → direct to cookie lang; URL-invalid + invalid cookie → tenant default; URL-valid + differing cookie → cookie-wins fires) all hold by reasoning. No `proxy.ts` test surface exists; manual smoke per the brief.

---

## 2026-05-22 — `spring-boot-starter-mail` was an orphan dependency in `pom.xml` until the Brevo SMTP integration wired it up

**Severity:** low
**Status:** fixed (2026-05-22, session oglasino-backend-brevo-smtp-1)
**Found in:** `oglasino-backend/pom.xml` lines 67-70 (managed by Spring Boot 4.0.6 parent, no explicit version).
**Detail:** `spring-boot-starter-mail` had been declared in `pom.xml` with zero consumers across `src/`. Grep for `JavaMailSender`, `MimeMessage`, `SimpleMailMessage`, `spring.mail`, `smtp`, `brevo`, and `sendinblue` returned zero email-sending-related matches. Either added speculatively and never wired, or wired and reverted leaving the starter behind. The 2026-05-22 Brevo SMTP integration session made it load-bearing — the dependency was the right one to keep, no version change needed. Logged here for the record so a future engineer reading `pom.xml` history doesn't re-investigate the gap.

---

## 2026-05-21 — Portal home: `onIdTokenChanged` listener has no single-flight guard, causing duplicate `firebase-sync` + `push/token` POSTs on hard refresh

**Severity:** medium
**Status:** fixed (2026-05-21, session oglasino-web-usetokenrefresh-single-flight-1; manual verification by Igor)
**Found in:** `oglasino-web/src/components/client/initializers/UseTokenRefresh.tsx`.
**Detail:** Firebase's `onIdTokenChanged` listener emits twice on cold session restoration (persisted token, then refreshed token ~8 ms later). The listener callback fired `syncUserToBackend` + `initPushForAuthenticatedUser` independently for each emission, producing 2× `POST /api/auth/firebase-sync` and 2× `POST /api/secure/push/token` per hard refresh. Normal refresh fired one of each because the SDK did not re-emit on the warmer path. The "sole hydrator" contract from the Consent Mode v2 closing entry (`decisions.md` 2026-05-21) was intact; the gap was idempotency at the listener.

> **Fix:** `useRef`-held `{ inFlightUid, lastSyncedUid, lastSyncedAt }` with a 2-second drop window for same-uid back-to-back emissions. `writeFirebaseTokenCookie(token)` hoisted into the listener so the cookie mirror runs on every emission (covers the rotated-token case even when the second backend sync is dropped). Guard resets on sign-out so sign-out → sign-in re-fires a fresh cascade. `lastSyncedAt` only set on success — failed syncs retryable immediately. `refreshUser` (the only non-listener caller of `syncUserToBackend`) untouched. Hard refresh now fires 1× firebase-sync + 1× push/token; normal refresh unchanged. Full context in `decisions.md` 2026-05-21 "Portal home cold-load cascade" entry.

---

## 2026-05-21 — Portal home: `randomSeed: Date.now()` in the `FilteredProductList` React key causes product-list remount and visible "jerk" on every SSR re-render

**Severity:** medium
**Status:** fixed (2026-05-21, session oglasino-web-filteredproductlist-random-seed-key-1)
**Found in:** `oglasino-web/src/components/client/product/FilteredProductList.tsx`, `oglasino-web/src/lib/utils/filtersHelper.ts:45-48`.
**Detail:** The no-filters home synthesizes `{ applyRandom: true, randomSeed: Date.now() }` on every SSR render. The inner `ProductList` was keyed on `JSON.stringify(filtersData)`, so a new seed → new key → React unmounts the previous list and mounts a fresh one with the new ordering. Visible product-order swap during load. Combined with the multiple-RSC-re-render bug (separate `issues.md` entry), each render produced its own visible swap.

> **Fix:** destructure `randomSeed` out of the filters object before stringifying for the key calculation; the seed still reaches the backend via the unmodified `filtersData` on the request body. After the fix, seed changes do not cause remount; only real filter changes do. Locks the jerk out unconditionally regardless of how many SSR re-renders happen. Full context in `decisions.md` 2026-05-21 "Portal home cold-load cascade" entry.

---

## 2026-05-21 — Portal home: FilterManager SYNC TO URL trailing-slash false-positive + redundant `setTimeout(router.refresh)` causes 2 extra RSC re-renders per refresh

**Severity:** medium
**Status:** fixed (2026-05-21, session oglasino-web-filtermanager-refresh-fix-1)
**Found in:** `oglasino-web/src/components/client/initializers/FilterManager.tsx`.
**Detail:** Two independent root causes inside the SYNC TO URL effect. (1) The URL-canonicalization gate compared `newUrl` (built with a trailing slash for `pathname === '/'`) to `current` (browser URL with no trailing slash), so the gate fired on every refresh of `/rs-sr` even when no filters had changed. Only the home was affected — every other FilterManager-mounted route produces a non-`/` `pathname` and never had the bad trailing slash. (2) The gate's body called `router.replace(newUrl, { scroll: false })` followed by `setTimeout(() => router.refresh(), 0)`. In Next 16, `router.replace` to a new pathname already refetches RSC; the `setTimeout(refresh)` was leftover from an older Next version where `replace` did not consistently refetch. One firing of the gate → two RSC re-renders → two extra `POST /api/public/product/search` calls per visit.

> **Fix:** Lever A — normalize the trailing slash in the `newUrl` construction so the gate evaluates `false` on the canonical no-filters home (no spurious `router.replace`). Lever B — delete the redundant `setTimeout(() => router.refresh(), 0)` line so legitimate filter-change triggers fire one RSC refetch instead of two. Both applied in one session. Hard refresh of `/rs-sr` dropped from 4 → 2 product-search POSTs; normal refresh dropped from 3 → 1. Other FilterManager-mounted routes scanned and confirmed unaffected by Lever A and improved by Lever B. Full context in `decisions.md` 2026-05-21 "Portal home cold-load cascade" entry.

---

## 2026-05-21 — `GlobalError` is the default-export function name in both `app/error.tsx` and `app/global-error.tsx`

**Severity:** low
**Status:** fixed (2026-05-23, session oglasino-web-google-analytics-v1-4)
**Found in:** `oglasino-web/app/error.tsx:6`, `oglasino-web/app/global-error.tsx:6`. Surfaced by Phase 2 audit of `features/google-analytics-v1.md` (2026-05-21).
**Detail:** Both files default-export a function literally named `GlobalError`. Next.js routes the two boundaries by filename (`error.tsx` is the route-level boundary, `global-error.tsx` is the app-level boundary), so the function name is cosmetic — runtime behaviour is unaffected. A future reader grepping for the global error boundary will hit two matches and have to disambiguate by file. The cleaner naming would be `RouteError` in `app/error.tsx` and `GlobalError` in `app/global-error.tsx`. Engineer flagged this under Part 4b at audit time as low severity, out of scope. Fold into the GA4 v1 brief that wires `trackError` into both boundaries — engineer renames `RouteError` while editing the file.

> **Fix:** GA4 v1 Brief 8 renamed `app/error.tsx`'s default-export function from `GlobalError` to `RouteError` while wiring `trackError`. `app/global-error.tsx`'s function name stays `GlobalError` (intended end-state per this entry). Future readers grepping for the global error boundary now hit exactly one match in `app/global-error.tsx`.

---

## 2026-05-21 — Mastermind drafted RU translation copy that diverged from established codebase convention

**Severity:** low (process error, caught by the engineer)
**Status:** fixed (engineer aligned to codebase convention)
**Found in:** Consent Mode v2 brief A.
**Detail:** Brief A's RU translation drafts dropped the `kh` digraph and apostrophe-soft-sign rendering that brief 8 explicitly established as the codebase's RU transliteration convention. Engineer caught the contradiction in pre-flight Brief vs reality and aligned the new rows to existing `category.necessary.*` values.

**Lesson:** when drafting translation copy that should "mirror" existing seeded copy, read the existing rows first and quote them verbatim rather than re-drafting and hoping the drafts match.

---

## 2026-05-21 — Mastermind misread conventions Part 6 Rule 3 dedicated-file exception

**Severity:** low (process error, caught mid-session by the engineer)
**Status:** fixed (process correction)
**Found in:** Consent Mode v2 brief 8.
**Detail:** Brief 8 framed Rule 3's dedicated-file exception as applicable to "20 keys in one namespace" on the reasoning that practical pain applied equally. The exception is strictly multi-namespace; single-namespace seeds default to inline-append regardless of key count. The engineer initially followed Mastermind's recommendation, then reversed after Igor's mid-session correction.

**Lesson:** when invoking a conventions exception, quote the rule verbatim before applying it. Paraphrasing the rule and arguing the paraphrase fits is the same smell as the cross-repo "scoped one-time exception" error.

---

## 2026-05-21 — Mastermind drafted a cross-repo brief that violated conventions Part 3

**Severity:** low (process error, caught at the engineer level; no code shipped)
**Status:** fixed (process correction)
**Found in:** Consent Mode v2 brief 8.
**Detail:** Brief 8 was drafted to cover both backend SQL seeds and a one-line web translation-key swap. Conventions Part 3 has no scoped-one-time exception for cross-repo edits. The backend engineer correctly refused the web portion. Brief 8 ran as backend-only; the web swap was routed as brief 8b.

**Lesson:** Phase 5 briefs that touch more than one repo must split. "Convince yourself the rule is OK to break this once" is a smell — conventions only change via decisions.md.

---

## 2026-05-20 — `TestCreateJSON.java` is an anonymous public endpoint triggering catalog-JSON regeneration

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-backend` — `src/main/java/com/memento/tech/oglasino/controller/test/TestCreateJSON.java`. Surfaced by the messaging feature's Brief 4 backend engineer as an adjacent observation.
**Detail:** `TestCreateJSON.java` lives in `src/main/java`, registers `GET /api/public/filter_gen/init` (anonymous, permitAll), and triggers `CatalogToJsonService.createJSONForAllBaseSites()` — a side-effecting catalog-JSON regeneration job. Not body-controlled like the `/api/public/notification/test` endpoint that the messaging feature deleted (Brief 4 B2), but functionally similar: an anonymous trigger for an expensive backend job.

Pre-launch posture per Igor: production deploy is imminent. The endpoint as-is is an unauthenticated trigger for catalog regeneration; an attacker hitting it repeatedly could cost CPU and storage I/O on the backend.

**Fix path:** either (a) gate behind `@Profile("dev")` if the endpoint is only useful in dev workflows, (b) move to `/api/secure/admin/**` with `@PreAuthorize("hasRole('ADMIN')")` if admins need it in production, or (c) delete entirely if no workflow uses it. Determine actual usage before deciding.

Separate one-shot bug-fix brief, post-messaging-launch.

> **Fix:** `@Profile("dev")` added to the class in session `oglasino-backend-round-2-fixes-1` (2026-05-27), matching the existing precedent at `MissingExtraTranslationsService.java`. On non-dev profiles the bean is not registered and the endpoint 404s. The underlying `CatalogToJsonService.createJSONForAllBaseSites()` method is preserved.

---

## 2026-05-20 — Brief 5 cleanup cron smoke not run on dev; first production Sunday is the live test

**Severity:** low
**Status:** open
**Found in:** `oglasino-backend` — `src/main/java/com/memento/tech/oglasino/messaging/service/impl/DefaultMessagingCleanupService.java`; messaging feature Brief 5 (2026-05-20).
**Detail:** The Sunday cleanup cron from the messaging feature was implemented and unit-tested (11 tests, all green, Mockito-mocked Firestore). The end-to-end path — Postgres hard-delete writes a `pending_chat_cleanup` row → `@Scheduled` cron fires → row is read → Firestore Admin SDK scans `chats` where `users` array-contains the deleted uid → chat-root `users[]` and per-message `senderId`/`receiverId` are anonymised to `"deleted:<uid>"` → `userchats/`, `userblocks/`, `userblocksReverse/` are deleted → queue row is removed — was not exercised against live Firestore on dev before the feature closed.

Options considered for pre-launch live verification: (A) temporarily set `messaging.cleanup.cron` to a high-frequency expression in `application-dev.yaml`, hard-delete a test user, observe the full path, revert. (B) defer to the first production Sunday at 03:00 UTC. (C) add a dev-only manual-trigger endpoint. Igor chose (B) — unit-test coverage is thorough enough that the integration risk is bounded, and the first production Sunday is itself the live test.

**Risk:** if the live Admin SDK behaviour differs from the Mockito stubs (e.g. `whereArrayContains` returning a `QuerySnapshot` shape the code doesn't anticipate, or a `batch.update(docRef, field, value)` overload mismatch), the cron will fail on its first Sunday production run. Per-user failure mode is logged, the queue row stays in place, and `attempt_count` increments, so the failure is recoverable (the next Sunday retries after the fix lands). The blast radius is bounded by the cron's per-user try/catch and the singleton job-lock.

**What to do:** on the first Sunday at 03:00 UTC after production deployment of the messaging feature, monitor backend logs for the `DefaultMessagingCleanupService` cron-start log line. If the first run produces ERROR-level entries with per-user firebaseUid identifiers, retrieve a sample of those identifiers and check whether the corresponding chat documents in Firestore have their `users[]` arrays anonymised. If the cron ran clean (INFO summary log with successCount > 0 or "no pending rows"), close this entry as `fixed`. If the cron produced ERRORs, open a bug-chat to debug against the live Admin SDK shape.

If the launch produces no hard-deletions before the first Sunday (likely for the launch period), the cron's first real exercise is whenever the first hard-deleted user reaches the queue. Adjust the watch window accordingly.

---

## 2026-05-20 — Edit-profile `displayName` change is silently dropped server-side

**Severity:** medium
**Status:** fixed (2026-05-20, session oglasino-backend-registration-displayname-1)
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/facade/impl/DefaultUserFacade.java` (`updateCurrentUserData`); `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/UpdateUserDTO.java`.
**Detail:** `UpdateUserDTO` declared `@NotBlank private String displayName;` and the controller `POST /api/secure/user/update` validated it via `@Valid`, but `DefaultUserFacade.updateCurrentUserData` never copied the value onto the loaded `User` entity — the field was parsed, validated, then silently discarded. Any user editing their displayName via the profile flow got a 200 OK while the server kept the original. Discovered during the 2026-05-20 backend audit for the registration `displayName` persistence fix.

> **Fix:** added `user.setDisplayName(updateData.getDisplayName())` to `DefaultUserFacade.updateCurrentUserData` alongside the existing setEmail/setProfileImageKey/etc. block. `UpdateUserDTO.displayName` validation tightened from bare `@NotBlank` to `@NotBlank + @Size(2, 60) + @Pattern("^[\\p{L} ]+$")` to match the registration contract. Cache plumbing inherits from `userService.saveUser`'s existing `@CacheEvict` set. Same session as the registration fix.

---

## 2026-05-19 — Registration `displayName` never persists to backend

**Severity:** medium
**Status:** fixed (2026-05-20, sessions oglasino-backend-registration-displayname-1 + oglasino-web-registration-displayname-1)
**Found in:** `oglasino-web/src/lib/store/useAuthStore.ts` (register action); `oglasino-web/src/lib/service/reactCalls/authService.ts` (registerUserFirebase); `oglasino-backend` firebase-sync handler.
**Detail:** Registration form collects `displayName`, but it never reaches Firebase Auth, Firestore, or the backend's `users` row. The pre-existing flow created a Firebase user via `createUserWithEmailAndPassword(email, password)` (no displayName), POSTed `{ allowPreferenceCookies }` to firebase-sync (no displayName), then patched the in-memory DTO with the form value before storing. The patching gave the UI a few-second illusion of "looks right" before the next sync overwrote it.

Frontend Observation-1 patch (2026-05-19) removed the patching as part of the listener-is-sole-hydrator design. The underlying gap is now visible: post-register, the user sees whatever the backend defaulted to.

**Proposed fix (chosen design):** extend the `firebase-sync` POST body to carry `displayName`. Backend `firebase-sync` handler reads it from the body and persists to the `users` row's `displayName` column. The backend owns the canonical store of `displayName`; Firebase Auth carries identity only.

Architecture: backend as source of truth aligns with how `displayName` changes are handled later in the user's lifecycle (the edit-profile flow writes only to the backend). Symmetric pattern across register + update means one place to maintain.

Out of scope for this feature (registration is its own surface). Schedule as a separate brief post-launch or whenever a registration-pass is scheduled.

> **Fix:** Backend — `LoginRequest` gained a `displayName` field. Validation is `@Size(min = 2, max = 60)` + `@Pattern("^[\\p{L} ]+$")` (Unicode letters and spaces only). `@NotBlank` was deliberately NOT added because `/auth/firebase-sync` is called on every login and token refresh, not only on register; the existing `resolveDisplayName(token)` chain handles the no-displayName case (OAuth providers populate `token.getName()`; email+password edge case falls back to email local-part). `DefaultFirebaseAuthService.createUserSynchronized` prefers `loginRequest.getDisplayName()` when non-blank, else falls back. Column folded `varchar(255)` → `varchar(60)`. Three new `ERRORS` keys seeded across 4 locales. Web — `useAuthStore.register` passes `displayName` through `registerUserFirebase` to a module-scoped one-shot cell consumed once by the listener's next `syncUserToBackend` call. Listener-is-sole-hydrator contract preserved. Register and edit-profile forms now mirror the backend validation (trim + size + pattern). Dead `displayName` field removed from the Firestore `users/<uid>` write and from the `FirestoreUser` type. 538 backend tests + 154 web tests green.

---

## 2026-05-19 — Admin Users detail page indicators stale after in-page ban/unban toggle

**Severity:** low (cosmetic)
**Status:** fixed (2026-05-20, session oglasino-web-admin-user-detail-disabled-lift-1)
**Found in:** `oglasino-web/app/[locale]/admin/users/[userId]/page.tsx`; `oglasino-web/src/components/admin/users/EnableDisableButton.tsx`.
**Detail:** The detail page renders state indicators from initial-fetch `userData.disabled`. When an admin toggles ban/unban via the `EnableDisableButton` on the same page, the button's local `disabled` state flips and its icon swaps, but the page's `userData.disabled` is not lifted to a shared source. The new "Banned" badge does not appear until the page is refetched or re-navigated.

**Fix path:** mirror the list-page `UserRow` pattern — lift `disabled` (and `lockedFromDeletion` if a lock toggle is added here later) to page state, add an optional `onDisabledChange` callback to `EnableDisableButton` matching `EnableDisableIcon`'s controlled-mode contract.

Out of scope for the indicator-only brief that surfaced it.

> **Fix:** `app/[locale]/admin/users/[userId]/page.tsx` now owns a `disabled` `useState`, seeded inside the existing fetch effect from `data.disabled` at the same time as `setUserData`. A `displayUser: UserDetailsDTO = { ...userData, disabled }` is passed to `UserStateIndicators` and `EnableDisableButton`, and `hasStateIndicator` is derived from the lifted value. `EnableDisableButton` gained an optional `onDisabledChange?: (next: boolean) => void` prop matching the `EnableDisableIcon` controlled-mode contract; the callback fires inside `handleSuccess` (success branch only) immediately after the internal `setDisabled`. Backwards-compatible: existing callers without the prop are unchanged. `lockedFromDeletion` was deliberately not lifted because no toggle for it exists on this page today; if a lock toggle is added later, the same lifting shape applies. `tsc`/`lint`/`test`/`prettier` clean.

---

## 2026-05-19 — Backend → web cache revalidation may not work in production via oglasino.com domain

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/service/impl/DefaultWebRevalidationService.java`; Cloudflare router worker (`oglasino-router`); backend prod/stage env var `WEB_REVALIDATE_URL`.
**Detail:** Group 2/3's backend → web cache-revalidation call POSTs to a URL provided via the `WEB_REVALIDATE_URL` env var. If that URL points at the `oglasino.com` domain (e.g., `https://oglasino.com/api/revalidate`), the Cloudflare router worker routes `/api/*` paths to the backend — meaning the backend POSTs to itself, hits no `/api/revalidate` endpoint, returns 404, and the web-side cache is never invalidated.

**Mitigation chosen:** Igor will point `WEB_REVALIDATE_URL` directly at the Vercel deployment URL (e.g., `https://oglasino-web.vercel.app/api/revalidate` or whatever the prod Vercel hostname is) to bypass the Cloudflare worker entirely.

**Verification needed:** confirm direct-Vercel-URL works in production. Two failure modes possible: (a) Vercel deployment URL changes per environment (preview vs prod) and breaks if not maintained; (b) backend's HTTP client treats the direct hostname differently (TLS, redirect handling, etc.) than the worker-fronted hostname.

**Alternative architectural fixes** (if direct-URL proves fragile):

- Add `/api/revalidate` as a Cloudflare worker exception that routes to Vercel
- Rename the frontend endpoint to a non-`/api/*` path (`/_revalidate` or `/internal/revalidate`)
- Use a separate internal domain

Verification first. If direct-URL works, close as "verified working" post-launch. If not, follow-up brief ships one of the alternatives.

> **Fixed 2026-06-04.** Verified working in prod via direct-Vercel-URL (Igor, 2026-06-04).

---

## 2026-05-19 — Home page only loads first page; possibly affects other paginated pages

**Severity:** medium (not blocking User Deletion launch)
**Status:** fixed
**Found in:** home page; possibly other paginated views (not yet audited).
**Detail:** Surfaced during User Deletion manual smoke testing. Out of scope for User Deletion feature — log here for follow-up. Pagination loads only the first page; subsequent pages may not be fetchable.

Investigation needed. Schedule as its own brief.

> **Status:** `fixed` (2026-05-28)
> Verified resolved on `stage` by Igor during the Tier 3 batch (2026-05-28). The read-only audit (`sessions/2026-05-28-oglasino-web-tier3-batch-audit-1.md` Section 3) found the web pagination code path structurally sound and offered three hypotheses (backend random-pagination, silent client-side failure, `initialProductsData` reference reset). Live reproduction on stage no longer shows the symptom — the home page paginates correctly. Root cause not isolated; likely closed incidentally by a prior fix (FilterManager trailing-slash / random-seed-key work, 2026-05-21, or Tier 2 AbortController work). No further code change required.

---

## 2026-05-19 — `/api/revalidate` used wrong cache-invalidation profile (silent no-op)

**Severity:** medium (resolved)
**Status:** fixed (2026-05-19)
**Found in:** `oglasino-web/app/api/revalidate/route.ts:82` (now corrected).
**Detail:** Route handler called `revalidateTag(tag, 'default')`, which under Next.js 16 is the long-cache profile (renews tag lifetime under the default profile), not immediate invalidation. Backend revalidation calls succeeded at HTTP level but did not actually invalidate the SSR cache. Users saw stale `UserInfoDTO` for up to 5 minutes after state transitions.

**Resolved by:** changing the call to `revalidateTag(tag, { expire: 0 })` — `CacheLifeConfig` with zero expiry forces immediate invalidation. Manual smoke confirmed: ban → unban → normal refresh now shows correct state (no hard refresh needed).

Round-3 handoff flagged this as a project-wide pattern. The `/api/revalidate` site is now fixed; other call sites (`oglasino-web/app/actions/cacheActions.ts` and possibly others) still need audit/fix. Separate task.

---

## 2026-05-19 — Manual testing of messaging gating for deleted and banned users not yet performed

**Severity:** medium
**Status:** open
**Found in:** `oglasino-web/src/messages/components/Messages.tsx`; `oglasino-backend` ban + deletion lifecycle.
**Detail:** The User Deletion feature's messaging-gating logic is shipped in code on both repos but has not been end-to-end manually verified. Two surfaces are affected:

- **Pending-deletion gating** (round 3): per spec §3.2 / §14.8, when a user is in `PENDING_DELETION`, counterparties should see the "Scheduled for deletion" badge in chat header, the message input should be disabled, and the notice "This user has scheduled their account for deletion and cannot receive messages." should render. The deleting user themselves should be signed out and unable to send/receive.
- **Banned-user gating** (B2, 2026-05-19): per the same spec section extended for ban handling, when a user is banned, counterparties should see the "Banned" badge in chat header, the message input should be disabled, and the notice "This user has been banned and cannot receive messages." should render. Banned users cannot sign in at all so the counterparty-side gating is the only visible surface.

The smoke test documented in the frontend B2 session summary (`.agent/2026-05-19-oglasino-web-user-deletion-<n>.md`, see "Tests" section) cannot run today because messaging is broken in current local stack. A separate fix is queued. Once messaging is working, the full smoke sequence must run:

1. User A messages User B; admin bans User B; counterparty (User A) sees badge / disabled input / notice in chat with User B.
2. Admin unbans User B; counterparty's chat returns to normal.
3. Third user (User C) put in pending-deletion via danger zone; counterparty sees pending-deletion badge / disabled input / notice.
4. User C cancels deletion by signing back in; counterparty's chat returns to normal.
5. Edge case: pending-deletion user is then admin-banned; counterparty sees "Banned" badge (not "Scheduled for deletion") — banned wins over pending-deletion in the UI per backend's `UserState.resolve()` composition.

Until this manual verification completes, the User Deletion feature's messaging-gating surface is considered not-fully-tested. Code review and unit tests on both repos passed (backend 487, web 154 green); behavior under live local stack remains unverified.

Fix scope: zero code work pending. This entry tracks the verification gap; close it after the manual smoke runs successfully.

---

## 2026-05-17 — `UserInfoDTO.isFollowingCurrent` seed is non-authoritative because `getUserForId` uses `skipAuth: true`

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-web/src/lib/service/nextCalls/userService.ts:13-19` (the `getUserForId` call), `oglasino-web/src/lib/config/fetchApi.ts:15-19, 38-45` (the `skipAuth` semantics), and the consumer at `oglasino-web/src/components/client/UserDetails.tsx:119` (`isFollowing={userDetails.isFollowingCurrent}` passed to `FollowUserButton`).
**Detail:** Server-side `getUserForId` fetches `/public/user?id=<id>` with `skipAuth: true`. Per the `fetchApi.ts` documentation comment, `skipAuth: true` explicitly bypasses the `cookies()` call AND the Firebase Bearer header — so the backend cannot identify the requesting viewer on this code path. Yet the returned `UserInfoDTO` carries two viewer-dependent boolean fields, `iamActive` and `isFollowingCurrent`, both of which require knowing who is asking to populate correctly. `iamActive`'s consequence is mitigated in `UserDetails.tsx:53-59` by an internal client-side recomputation against `useAuthStore.user.id`; `isFollowingCurrent` has no such mitigation — it is passed straight into `<FollowUserButton isFollowing={userDetails.isFollowingCurrent} />` (`UserDetails.tsx:119`), which uses it as the seed for its local `following` state. Observable consequence: a signed-in viewer who already follows User X opens `/user/<X-id>` (or any `/product/<…>` owned by X) and sees the Follow icon (UserPlus) instead of the Unfollow icon (UserMinus). One click then _unfollows_ them, contrary to the user's mental model. Compounding: there is no `revalidateTag('user:${userId}')` call anywhere downstream of `markFollowUser`, so the cached `UserInfoDTO` (Next.js Data Cache, 5-minute `revalidate` with tag `user:${userId}` per `userService.ts:17-18`) does not refresh after a follow action either. Fix scope: either split `getUserForId` into a public-skipAuth path that omits the viewer-dependent fields and an authenticated path that does forward auth (matching the `/public/*` vs `/secure/*` split the backend already has), or drop `isFollowingCurrent` from this public endpoint and seed the button via a separate small auth-required call, or compute the field client-side post-mount via a Zustand follows-store. Project-wide implications similar to `useAuthResolved`-style hydration handling — likely its own focused chat. Found by Docs/QA during the QA-Preparation `follow-flow` topic authoring.

> **Fix (2026-05-28, session `oglasino-web-isfollowing-and-views-fix-1`):** Fix A applied — `skipAuth: true` removed from `getUserForId` (`src/lib/service/nextCalls/userService.ts`) and from the `portal` branch of `getProductDetails` (`src/lib/service/nextCalls/productsSearchService.ts`, the latent twin). Auth is now forwarded on both fetches, so the backend populates viewer-dependent fields (`isFollowingCurrent`, `iamActive`) authoritatively. `markFollowUser` (`src/lib/service/reactCalls/followService.ts`) now calls `await revalidateUserCache(userId)` on success to bust the `user:${userId}` Data Cache tag, closing the 5-minute staleness window after follow/unfollow. Two adjacent items folded into the same session: product-page owner null-guard (`if (!owner) notFound();` mirroring the `baseSite` pattern at `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`), and `FollowUserButton` click/a11y (click handler moved from inner icons to the `<button>` element, `type="button"` + `aria-label` added; keyboard Enter/Space activation now works). Fix A was chosen over Fix B (drop the field + small auth-required call) and Fix C (`useFollowsStore`) per the `skipauth-footprint-audit-1` finding that both consumer routes are already dynamic for unrelated reasons, so the route-level Full Route Cache cost of dropping `skipAuth` is zero; only per-viewer Data Cache fragmentation increases, which the 5-minute / 60-second revalidate windows mitigate. B and C remain principled fallbacks if per-viewer fragmentation becomes a measured problem. Knock-on (intended): `owner.iamActive` is now authoritative on the product page, so the `if (!owner.iamActive) markAsSeen(...)` gate correctly skips for the owner-as-viewer case for the first time — see the open 2026-05-28 view-count entry for the increment-side investigation this surfaced.

---

## 2026-05-17 — `/owner/follows` `UserCard` calls `notify.error()` for both success and failure paths

**Severity:** low
**Status:** fixed (2026-05-20, session oglasino-web-follow-result-shape-1)
**Found in:** `oglasino-web/src/components/owner/follows/UserCard.tsx:19-33`.
**Detail:** The `onUnfollow` handler in `UserCard` has two branches based on the `markFollowUser` result. The `result === undefined` branch (line 21-25) calls `notify.error()` and is dead code — `markFollowUser` always returns a `boolean` (`followService.ts:8,12,15`), never `undefined`. The other branch (line 26-31), explicitly meant for the success case (`tCommon(result ? 'user.follow.added' : 'user.follow.removed')`), also calls `notify.error()` instead of `notify.success()`. Result: every unfollow action from the owner dashboard toasts in the error/red style even when the operation committed successfully. Trivial wording fix — switch the success branch to `notify.success()` and drop the dead `undefined` branch (or repurpose it to call `notify.error` for a real error path returned by a future revision of `markFollowUser`). Found by Docs/QA during the QA-Preparation `follow-flow` topic authoring.

> **Fix:** `markFollowUser` widened to `Promise<{ ok: true; following: boolean } | { ok: false }>`; both consumers (`UserCard.tsx`, `FollowUserButton.tsx`) now branch on `result.ok` and surface `notify.error` on HTTP failure. No new translation seed rows authored — both call sites use `review.system.1` as the generic-system-error fallback. A follow-up session (`oglasino-web-follow-button-failure-unblock-1`, 2026-05-20) also released the 2-second `setBlocked` debounce synchronously on the failure branch so the user can retry immediately without waiting on a server-state-unchanged debounce; the 2-second debounce on success is unchanged.

---

## 2026-05-17 — `/owner/follows` `UserCard` does not remove the unfollowed row from the list

**Severity:** low
**Status:** fixed (2026-05-20, session oglasino-web-follows-row-removal-1)
**Found in:** `oglasino-web/src/components/owner/follows/UserCard.tsx:19-33` (the handler) and `oglasino-web/app/[locale]/owner/follows/page.tsx:75-91` (the consumer that owns the list state).
**Detail:** `UserCard.onUnfollow` calls `markFollowUser` and fires a toast but does not update the parent `followsUsers` state. The card stays visible in the list with its "Unfollow" button still active and labelled — clicking it again _re-follows_ the user (because the underlying endpoint is a toggle: first call adds, second call removes, etc.). The list only re-aligns with reality after a manual page reload or a `loadMore` call against the backend. Owner sees a stale list with confusing toggle semantics. Fix scope: pass a parent callback into `UserCard` (or call a parent-supplied filter) that removes the row from `followsUsers.users` on a successful unfollow response. Related to the dead-row-state class of bug; same root pattern as the `/owner/follows` UI not auto-refreshing. Found by Docs/QA during the QA-Preparation `follow-flow` topic authoring.

> **Fix:** added a required `onUnfollowSuccess: (userId: number) => void` prop on `UserCard`. The callback fires only on `result.ok === true && result.following === false` (the genuine unfollow case). Parent (`app/[locale]/owner/follows/page.tsx`) implements `handleUnfollowSuccess` which filters the unfollowed user out of `followsUsers.users` and decrements `followsUsers.totalNumberOfUsers`, mirroring the spread-into-new-object setState shape that `loadMore` already uses. `hasMore = users.length < totalNumberOfUsers` continues to compute correctly post-removal because both sides decrement by 1. Toast still fires unchanged. Lint/tsc/test clean.

---

## 2026-05-17 — Start-Message product context (`tempProductReason`) cleared on MessageInput mount, drops `productId` from the first message

**Severity:** medium
**Status:** fixed (2026-05-20, messaging feature Brief 2, sessions oglasino-web-messaging-1 + smoke confirmed)
**Found in:** `oglasino-web/src/messages/components/MessageInput.tsx:57-68` (consumer) and `oglasino-web/src/messages/store/useChatStore.ts:499-501, 516-518, 575-578` (producer + intended cleanup site).
**Detail:** When a user starts a conversation from a product page via `StartMessageButton`, the store action `setTempProductReason(product)` records the originating product so that the first `MessageRequest` written to Firestore can carry its `productId`. `MessageInput`'s mount `useEffect` reads `tempProductReason` to compose the `initial.product.message` suggestion text, then immediately calls `setTempProductReason(undefined)` — clearing the store field _before_ the user types and sends. By the time `useChatStore.sendMessage` runs and checks `if (state.tempProductReason) { messageRequest.productId = state.tempProductReason.id; }`, the field is already `undefined`, so `productId` is silently dropped from the first message of every product-path conversation. The store's own end-of-send cleanup (`set({ tempReceiver: null, tempProductReason: null })` at `useChatStore.ts:575-578`) further confirms the intent was for the clear to happen on send, not on mount. Visible symptom: every conversation initiated via a product page lacks the originating-product link in its first message — any consumer of `MessageRequest.productId` (audit, "this chat started about Product X" surfacing, analytics) sees no data. The text prefill itself still works because `MessageInput`'s `suggestion` is stored in local state. Fix scope: move the `setTempProductReason(undefined)` call out of `MessageInput`'s mount effect; rely on the store's existing post-send cleanup in `sendMessage` instead. Found by Docs/QA during the QA-Preparation `start-message-flow` topic authoring.

> **Fix:** Brief 2 W1 renamed `tempProductReason` → `tempProductContext` across `useChatStore`, `ChatStore` type, and consumers; W2 removed the premature `setTempProductContext(undefined)` mount effect in `MessageInput.tsx`; W3 reads the captured context synchronously at the top of `sendMessage` and includes `productId` on the message write. Smoke-verified end-to-end on 2026-05-20 — first message of a new product-page-initiated conversation lands with `productId` set in the Firestore message document.

---

## 2026-05-17 — Hardcoded Serbian "Kategorije iz <X>" breadcrumb button label

**Severity:** low
**Status:** fixed (2026-05-20, session oglasino-backend-breadcrumb-translation-seed-1 + Igor web swap)
**Found in:** `oglasino-web/src/components/server/OglasinoBreadcrumbs.tsx:81`
**Detail:** The "jump into subcategories" button rendered at the tail of a breadcrumb chain (when the chain has ≤ 2 levels) uses the literal Serbian string `Kategorije iz {t(category.labelKey)}` instead of a next-intl key — EN / RU / ME visitors see Serbian. A Part 6 (translations) violation in `oglasino-web` source, same family as the existing 2026-05-14 _Hardcoded Serbian tooltips on the product detail page_ entry. Found by Docs/QA during the QA-Preparation `category-navigation` topic authoring.

> **Fix:** `breadcrumb.categories_from.label` seeded in the `NAVIGATION` namespace across all four locale files (EN id 2543, RS 4643, RU 6743, CNR 443). Web swap (`OglasinoBreadcrumbs.tsx:81` literal → `useTranslations` lookup) applied by Igor directly.

---

## 2026-05-16 — SEO JSON-LD delivered as `<meta>` tags instead of `<script>`, project-wide

**Severity:** medium
**Status:** fixed (2026-05-24, SEO foundation feature — brief 1 converted 7 pages to `<script>` tags; brief 5g deleted JSON-LD from privacy and terms entirely, shrinking scope from 9 to 7 pages)
**Found in:** `oglasino-web` — nine `src/metadata/generate*Metadata.ts` files: Home, About, Pricing, FreeZone, Catalog, Product, User, Privacy, Terms. Plus the inline `<script>` at `app/[locale]/(portal)/(public)/privacy/page.tsx:21-26`.
**Detail:** Every public page emits its Schema.org document into `metadata.other['application/ld+json']`. Next.js renders `metadata.other` entries as `<meta name="application/ld+json" content="...">` tags; search engines parse JSON-LD only from `<script type="application/ld+json">` blocks. Result: every public page in the codebase has a structured-data block that no crawler will read. Real SEO impact pre-launch. Fix scope: export the structured-data helpers from each metadata file and emit inline `<script>` tags on each page. Nine-page touch plus helper-export plumbing. Privacy additionally has a malformed inline `<script>` that stringifies `generatePrivacyPageMetadata`'s return value (a Next.js `Metadata` object — `{ title, description, alternates, openGraph, twitter, other }`) instead of the dedicated `generatePrivacyPageStructuredData` helper output; a clean fix on Privacy is part of this work. Feature-sized. Subsumes the 2026-05-14 entry "Terms page does not emit JSON-LD; Privacy page does" — see that entry's amended body. Surfaced by the terms-json-ld-1 session.

---

## 2026-05-16 — Report-submit endpoint trust-boundary verification unknown

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-web/src/lib/service/reactCalls/reportService.ts:12` (client submit call site); backend home is `oglasino-backend` (controller + service for `POST /secure/report/add`).
**Detail:** `reportedUserId` is client-supplied to `POST /secure/report/add` from both `ReportButton.tsx` (public user page) and the new `Messages.tsx` dropdown wiring. Whether the backend verifies (a) the id resolves to a real user, and (b) the authenticated reporter is authorized to report this target (participant in a conversation, viewer of a public user page, etc.) cannot be determined from the web repo alone. If the backend doesn't validate, any logged-in user can file false reports against any other user by manipulating the request payload — an abuse vector regardless of how the UI surfaces the action. Pre-existing if it's a gap (the `ReportButton` call site has the same shape since before this chat's work). Needs a one-shot backend audit of the `report/add` controller and service to confirm; severity finalizes after the audit. Surfaced by the messages-report-wiring-1 session.

> **Audit (2026-05-28, session `oglasino-backend-trust-boundary-audit-1`):** Verdict — **violation** per conventions Part 11. Reporter identity is server-derived (`currentUserService.getCurrentUserIdStrict()` from `SecurityContextHolder`) ✓. Target identity (`reportedUserId` / `reportedProductId`) is taken verbatim from the client body with no application-level existence check, no self-report block, no participation check. `reported_user_id` is FK-protected at the DB layer (`fk2fm8nu7yscahr6sbhhgw082mp`), so a totally fake id surfaces as an uncaught `DataIntegrityViolationException` (HTTP 500). `reported_product_id` has no FK at all — any client-supplied long is persisted verbatim. The 24-hour Redis dedupe key is `report:user:<reporterId>` (reporter-only, not per-target), which bounds one account's rate but not the population's — an attacker can spin up N Firebase accounts and from each file one report against the same target. Recommended fix shape: application-level `userRepository.findById` / `productRepository.findById` lookups, self-report block, deletion-state guard, per-target dedupe key, full DTO `@Valid` + typed error codes (delete dead `if` at `DefaultReportService:34-37`). One backend session + one web brief. Audit body has the full blast-radius treatment.

> **Backend half fixed (2026-05-28, session `oglasino-backend-report-trust-boundary-fix-1`):** All six audit-recommended changes shipped — existence checks via `findById` / `getOwnerId`, self-report block, deletion-state guard, per-target dedupe key, DTO `@Valid` with new `ReportErrorCode` family replacing every `IllegalStateException` path (responses now coded 400/422/406, never 500), dead `if` at `DefaultReportService:34-37` deleted. New tests: `DefaultReportServiceTest` (9 tests) + `DefaultReportFacadeTest` (4 tests including the per-target dedupe verification: same reporter + two different targets both succeed within 24h; same reporter+target twice → 406). Trust-boundary verdict: clean (Part 11). Backend half fixed 2026-05-28; web error-code mapping pending — next Mastermind.

> **Web error-code mapping complete (2026-05-31 bug batch, web lane, branch `dev`).** The full `ReportErrorCode` enum is now surfaced as translated UI copy web-side, closing the "web error-code mapping pending" follow-up the backend-half note left open. Both halves of this entry are now complete (backend trust-boundary hardening 2026-05-28; web error-code mapping 2026-05-31), so the overall status was flipped `open` → `fixed` (Igor-confirmed, 2026-05-31 bug-batch follow-up).

---

## 2026-05-16 — `useAuthResolved` adoption pending across the app

**Severity:** low
**Status:** fixed (partial)
**Found in:** `oglasino-web` — `src/lib/hooks/useAuthResolved.ts` (new), `src/components/client/HeaderNavButtons.tsx`, `src/components/client/SessionGuard.tsx`, `src/lib/store/useAuthStore.ts`.
**Detail:** The `useAuthResolved` hook introduced by the hydration-flashes-1 session is the right shared primitive, but adoption is partial. `HeaderNavButtons` still inlines the same Firebase-ready + store-user-loaded gate that the hook now formalizes (the precedent the hook mirrored). `SessionGuard` uses `auth.authStateReady()` for whole-route gating, a related-but-different technique. `useAuthStore` exposes no synchronous "initialized" flag, which is why every consumer that cares about the flash window has to bolt a Firebase listener onto the side. Scope: audit every auth-gated component across the app, consolidate around `useAuthResolved` where in-component gating is needed and `SessionGuard`-style for route-level, and consider adding an `initialized: boolean` flag to `useAuthStore` so the side-bolt becomes unnecessary. Likely its own focused session, possibly a Mastermind feature chat. Surfaced by the hydration-flashes-1 session.

> **Fix:** `HeaderNavButtons` and `MobileFooterNavigation` adopted `useAuthResolved` in session `oglasino-web-useAuthResolved-adoption-1` (2026-05-28). The two inlining components identified by the audit are now using the hook. Full adoption (including adding `initialized` to `useAuthStore` and any remaining auth-gating components) is a future Mastermind decision — see audit Candidate B at `.agent/2026-05-28-oglasino-web-tier3-batch-audit-1.md` Section 1.

---

## 2026-05-16 — `isAllowedPath()` allowlist anti-pattern

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/components/client/initializers/FilterManager.tsx:52-66`.
**Detail:** `isAllowedPath()` is an opt-in-by-string-matching list of paths embedded inside `FilterManager`. It grows one path at a time and forgets dynamic segments (the `[userId]` exclusion was its second documented failure). The mechanism has produced two real bugs so far: the original `/admin/products/[userId]` chip-no-op, and the in-app-nav broken state on `/admin/products` and `/owner/products` (the second was latent until the admin-filter-pipeline audit surfaced it). A different model — a per-route `enableFilterSync` opt-in prop, or moving `FilterManager` from layouts to page-level mounts — would remove the footgun. Feature-sized refactor; not bug-fix scope. Surfaced by the admin-filter-pipeline audit and fix sessions.

> **Fixed 2026-06-05 (filter-batch).** The anti-pattern this entry describes was removed.
> FilterManager was relocated from the 3 layouts to 5 page-level mounts and isAllowedPath()
> was deleted entirely (the W7 mount relocation). A store-rehydrate regression that the
> relocation introduced (filters cleared on product→logo→home) was then fixed via a
> store-aware fresh-mount guard in FilterManager (keeping per-page mounting, NOT reverting
> to the layout+allowlist arrangement). The opt-in-by-string-matching mechanism no longer
> exists. Verified live by Igor (logo→home preserves filters; catalog category-switch and
> hard-refresh both still work). Branch: web dev (staged, uncommitted per process).

---

## 2026-05-16 — Sitemap generation fetches full product payload to read a count

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/app/sitemap.ts:79-83` — `getProductCountForBaseSite`.
**Detail:** `getProductCountForBaseSite` calls `getProductsPage(baseSite, 0, 1)` purely to read `totalNumberOfProducts`, paying for one full product record's payload (and any backend hydration cost on a 1-row page) when only a count is needed. A count-only backend endpoint (`GET /api/public/product/count?baseSite=...` or equivalent) would be cheaper at sitemap-generation time. Cross-repo — needs a backend addition before the sitemap can switch over. Surfaced by the empty-overviews-consolidation-2 session.

> **Fix:** `getProductCountForBaseSite` in `oglasino-web/app/sitemap.ts` now calls `GET /api/public/product/count` with `X-Base-Site` header instead of `getProductsPage(baseSite, 0, 1)`. The backend endpoint (shipped 2026-05-27 in session `oglasino-backend-product-count-endpoint-2`) uses Elasticsearch's `_count` API directly — no document fetch, no deserialization, no mapping. Web swap shipped 2026-05-27 in session `oglasino-web-sitemap-count-swap-1`.

---

## 2026-05-16 — `/admin/products` filter chips don't take effect after in-app nav from disallowed admin route

**Severity:** medium
**Status:** fixed (2026-05-16, session oglasino-web-admin-filter-pipeline-fix-1)
**Found in:** `oglasino-web/src/components/client/initializers/FilterManager.tsx`.
**Detail:** Surfaced and fixed in the same chat. `FilterManager`'s HYDRATE FROM URL effect had deps `[baseSite]` and ignored `pathname` changes; in-app nav from a disallowed admin path (e.g., `/admin/users` → click "Products") into `/admin/products` left `hydratedRef.current === false` because the effect's deps never re-fired, and the SYNC TO URL effect silently early-returned on every subsequent chip click. URL never updated, list never refetched. Hard reload was the only way to escape the gated state. Fixed by replacing `hydratedRef` boolean with a per-path `lastHydratedPathRef`, adding `pathname` to the HYDRATE effect deps, and updating the SYNC effect gate accordingly. Same root cause behind the existing 2026-05-14 entry on `/admin/products/[userId]` chip changes; both entries close on the same fix. The same deps-array bug had a latent twin on `/owner/products` (in-app nav from disallowed owner sidebar routes like `/owner/balance`, `/owner/reviews`, `/owner/follows`, `/owner/analytics`) — incidentally fixed by the same change.

---

## 2026-05-16 — Global header search input has no Enter-key submission handler

**Severity:** medium
**Status:** fixed (2026-05-20, session oglasino-web-header-search-enter-1)
**Found in:** `oglasino-web/src/components/client/SearchInput.tsx:134-141`
**Detail:** The `<Input>` rendering the global header search has no `onKeyDown` handler and is not wrapped in a `<form>`, so pressing Enter while focused in the input does nothing — no submit, no popup open, no navigation. The only path that persists a typed term into the destination listing page's `searchText` chip and URL is clicking the submit button at the bottom of the autocomplete dropdown (same file, lines 164-194), and that button only renders when the autocomplete returned at least one product. Users almost universally try Enter first; the dead key reads as "search is broken" in a way that's hard to diagnose without reading the component. Affects all three scopes (portal, owner dashboard, admin) — the same `SearchInput` is reused via `SelectableSearchInputWrapper`. Cross-referenced from the `global-header-search` QA topic as a "Known issue" pitfall; the topic's `qaChecklist` includes a standing check that pins current behaviour and will need flipping when this is fixed. Found by Docs/QA during the QA-Preparation `global-header-search` topic authoring (2026-05-16).

> **Fix:** `SearchInput.tsx` extracts a `commitSearch(rawTerm)` helper and adds an `onKeyDown` handler on the global `<Input>` that, on `Enter`, trims `term`, no-ops on whitespace-only input, and otherwise calls `commitSearch(trimmed)`. The autocomplete submit button now also routes through `commitSearch(debouncedTerm)`. The fix applies uniformly to portal, owner dashboard, and admin scopes (shared component via `SelectableSearchInputWrapper`). Note: the standing "Known issue" pitfall and `qaChecklist` item on the `global-header-search` QA topic in `oglasino-docs/features/qa-preparation.md`-derived content (`topics.ts`) now describe stale behavior — the pitfall and the standing-check both need flipping in a follow-up Docs/QA session. Not in scope for this brief.

---

## 2026-05-16 — Five loose `^N` major-only version constraints remain in `package.json`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/package.json` — `@tailwindcss/postcss`, `@types/node`, `@types/react-dom`, `tailwindcss`, `typescript`
**Detail:** Five direct dependencies are declared with major-only `^N` constraints (e.g. `"typescript": "^5"`, `"tailwindcss": "^4"`). Under these constraints, the `package.json` floor doesn't enforce a meaningful minimum — `npm install` drifts forward freely as soon as a new minor lands within the major. The actual installed version sits in `package-lock.json` only. Down from six since the 2026-05-16 dependency-upgrade brief tightened `@types/node ^20` to `^22`. A future small cleanup pass should normalize these to `^MAJOR.MINOR.PATCH` floors reflecting what's actually installed. Found by Docs/QA in the web dependency audit (2026-05-16) and confirmed in the upgrade session summary.

> **Fix:** All five tightened to `^MAJOR.MINOR.PATCH` against lockfile-pinned versions in session `oglasino-web-package-json-tightening-1` (2026-05-27): `@tailwindcss/postcss` `^4.3.0`, `@types/node` `^22.19.19`, `@types/react-dom` `^19.2.3`, `tailwindcss` `^4.3.0`, `typescript` `^5.9.3`. Metadata change only — only the floor rose; upper bounds and lockfile pins unchanged.

---

## 2026-05-16 — 211 pre-existing ESLint warnings in `oglasino-web`

**Severity:** low
**Status:** parked
**Found in:** `oglasino-web` — repo-wide
**Detail:** `npm run lint` reports `✖ 211 problems (0 errors, 211 warnings)` on `dev`. All 211 are pre-existing — none introduced by the 2026-05-16 dependency-upgrade work that surfaced them. Categories: `react-hooks/exhaustive-deps`, `@typescript-eslint/no-explicit-any`, `@next/next/no-img-element`. Pre-launch this is tolerable; the count is the kind of slow drift that becomes invisible. Worth one focused lint-cleanup brief at some point, but not blocking anything. Found by the web dependency-upgrade engineer (2026-05-16).

> **Parked 2026-06-04 (prod-bug-sweep).** Optional cleanup pass; 0 errors throughout — not a launch blocker. **Baseline corrected: the count is now 142, not 211** (the six-fix prod-bug-sweep batch removed three `any`s landing at 143, then the exhaustive-deps suppression below took it to 142). The stale 211 figure is reconciled in `state.md`.

---

## 2026-05-16 — Three `-Xlint` compile warnings in `oglasino-backend`, pre-existence unverified

**Severity:** low
**Status:** fixed (2026-05-20, session oglasino-backend-xlint-warnings-1)
**Found in:** `oglasino-backend` — `src/main/java/com/memento/tech/oglasino/admin/internal/controller/AppVersionController.java`, `src/main/java/com/memento/tech/oglasino/elasticsearch/service/impl/ProductIndexer.java`, `src/test/java/com/memento/tech/oglasino/exception/GlobalExceptionHandlerTest.java`
**Detail:** `./mvnw clean compile` and `./mvnw test` emit three `-Xlint` notices: deprecated-API use in `AppVersionController` and `GlobalExceptionHandlerTest`, unchecked/unsafe operations in `ProductIndexer`. Surfaced during the 2026-05-16 dependency-upgrade session (`firebase-admin 9.8.0 → 9.9.0`, AWS SDK 2.42 → 2.44, `postgresql` 42.7.10 → 42.7.11). The session did not capture a pre-upgrade baseline, so it cannot prove these are pre-existing — but none of the three files reference Firebase Admin or AWS SDK classes by name, suggesting they are pre-existing codebase notices unrelated to the upgrade. **Recommended quick check:** a `-Xlint:deprecation,unchecked` compile on a fresh checkout of the pre-upgrade `pom.xml` would confirm pre-existence cheaply. If pre-existing, normal cleanup; if newly-surfaced, one of the upgraded SDKs is implicated.

> **Fix:** verified all three warnings pre-existed the 2026-05-16 dependency upgrade by checking out the pre-upgrade pom (commit `5df54c9`) via `cp`, running `clean compile` and `clean test-compile`, grepping each named file in the logs, then restoring the pom (verified byte-identical via `diff`). `AppVersionController.java`: `Version.valueOf` / `lessThan` swapped to `Version.parse` / `isLowerThan` (non-deprecated semver4j API). `ProductIndexer.java`: raw `Map.class` deserialisation tightened to `TypeReference<Map<String, Object>>()` so the parameterised type is preserved end-to-end through `IndexOperations.create(Map<String, Object>)`. `GlobalExceptionHandlerTest.java`: three method-level `@SuppressWarnings("deprecation")` annotations added with explanatory comment — narrow-scope suppression chosen because swapping the test to `HttpStatus.UNPROCESSABLE_CONTENT` while production `ProductErrorCode` still constructs `UNPROCESSABLE_ENTITY` would compare two distinct enum instances and fail. The suppressions should come out as part of the future coordinated `UNPROCESSABLE_ENTITY → UNPROCESSABLE_CONTENT` sweep. Post-fix `./mvnw clean compile` and `./mvnw clean test-compile` show zero matches for the three named files. `./mvnw spotless:check` and `./mvnw test` (502 passing) green.

---

## 2026-05-16 — Verify whether `opentelemetry-semconv` pin remains necessary post-Firebase-upgrade

**Severity:** low
**Status:** fixed (2026-05-20, verified by Igor directly)
**Found in:** `oglasino-backend/pom.xml` — `<dependencyManagement>` `io.opentelemetry.semconv:opentelemetry-semconv 1.41.1`
**Detail:** The pin was added on 2026-05-15 because `firebase-admin → google-cloud-storage` was pulling in `opentelemetry-semconv 1.29.0-alpha`, which lacked `DbAttributes` and caused Elasticsearch client startup failure (see 2026-05-15 entry, status `fixed`). On 2026-05-16, `firebase-admin` was upgraded 9.8.0 → 9.9.0 (transitively pulling `google-cloud-storage` 2.63.0 → 2.64.0). `dependency:tree` on both before and after resolves `opentelemetry-semconv` to `1.41.1` — but that's the effective post-mediation version, dominated by the pin. The real question — whether Maven would still mediate to `1.29.0-alpha` without the pin — needs `./mvnw dependency:tree -Dverbose=true` on the post-upgrade pom to surface omitted candidates. **If the verbose output no longer shows `1.29.0-alpha` in the candidate set, the pin is redundant and can be removed.** If `1.29.0-alpha` is still in the set, the pin remains load-bearing. Cheap to run in one read-only session.

> **Fix:** verified by Igor directly. Pin disposition recorded as resolved.

---

## 2026-05-15 — `oglasino-web` tsconfig has `strict: false`

**Severity:** low
**Status:** parked
**Found in:** `oglasino-web/tsconfig.json`
**Detail:** Strict null-checks are not enforced. The filtersHelper-return-type-1 session surfaced the consequence: declared-optional fields can be de-`?.`-chained without tsc rejection, even when the field's type declaration says they may be undefined. This means the typechecker is silent on a class of real bugs — anywhere a `Type | undefined` value is used without a null guard, tsc will not complain. Moving to `"strict": true` (or at minimum `"strictNullChecks": true`) would surface every such site as a tsc error; some will be true bugs, most will be type-tightening work where the producer always emits but the declaration says otherwise. Scope is project-wide, not a single-file fix. Surfaced by the filtersHelper-return-type-1 session addendum.

> **Parked (Igor, 2026-06-04)** — measured 214 errors under full strict (~73% null-safety family, spread over 72 files). Not a bug-batch item; deferred to a dedicated strict-mode cleanup chat. `strict` stays false until then.

---

## 2026-05-15 — `printStackTrace()` instead of logger in GlobalIndexerService

**Severity:** low
**Status:** fixed (2026-05-20, session oglasino-backend-global-indexer-logger-1)
**Found in:** `oglasino-backend` — `src/main/java/com/memento/tech/oglasino/elasticsearch/service/impl/GlobalIndexerService.java:29`
**Detail:** Per-indexer failures during boot reindex are printed via `ex.printStackTrace()` instead of using the project's logger. Conflicts with conventions Part 4 ("No `System.out.println`... Logger calls fitting the existing logging strategy are fine"). Diagnostic-output style only; no user-visible impact.

> **Fix:** replaced `ex.printStackTrace()` at `GlobalIndexerService.java:29` with `log.error("Boot reindex failed for {}", controller.getClass().getSimpleName(), ex)`. Added the `private static final Logger log = LoggerFactory.getLogger(...)` field plus the two SLF4J imports, matching the dominant vanilla-SLF4J pattern used by sibling files in the same package (`ProductIndexer`, `DefaultEsStateService`). Catch-block logic untouched; the loop still continues past per-indexer failures. `./mvnw spotless:check` and `./mvnw test` (502 passing) green.

---

## 2026-05-15 — Main method uses Java 21 preview signature, blocks `mvn spring-boot:run`

**Severity:** high
**Status:** fixed
**Found in:** `oglasino-backend` — `src/main/java/com/memento/tech/oglasino/OglasinoApplication.java:11` and `pom.xml` `spring-boot-maven-plugin` configuration
**Detail:** `OglasinoApplication.main` is declared `static void main(String[] args)` — package-private. This signature is only valid as a JVM entry point under Java 21 preview feature JEP 463 (Instance Main Methods). The Maven compile and surefire plugins pass `--enable-preview`, but `spring-boot-maven-plugin` does not, so `mvn spring-boot:run` fails immediately with `Error: Main method not found in class com.memento.tech.oglasino.OglasinoApplication`. IDE runs likely add the preview flag automatically, which is why this has not been noticed. Effect: every brief that mandates a `mvn spring-boot:run` verification step is currently un-verifiable from the command line. Fixed by changing `main` to `public static void main`. Verified end-to-end via Docker dev environment.

---

## 2026-05-15 — Backend fails to start: missing OpenTelemetry semconv class

**Severity:** high
**Status:** fixed
**Found in:** `oglasino-backend` — startup fails during `GlobalIndexerService.onAppReady()` → `ProductIndexer.reindexAll()` → Elasticsearch client index creation
**Detail:** Spring Boot startup fails with `java.lang.NoClassDefFoundError: io/opentelemetry/semconv/DbAttributes`. The Elasticsearch Java client (9.2.6) has built-in OpenTelemetry instrumentation that requires the `opentelemetry-semconv` library. Either it is not on the classpath or a transitive version is too old to contain `DbAttributes` (the class lives in `opentelemetry-semconv` 1.27+). Suspected cause: a recent dependency upgrade (Spring Boot 4.0.5, Spring 7.0.6, Elasticsearch Java 9.2.6) bumped the Elasticsearch client to a version that needs a newer semconv than what's resolved. App does not start locally. Fixed by pinning `opentelemetry-semconv` to 1.41.1 in pom.xml dependencyManagement. See commit 5df54c9cde9507109f0e78bf662b27c994a16be2.

---

## 2026-05-15 — `use.backend.check` read sits outside the worker's fail-open try/catch

**Severity:** medium
**Status:** fixed (2026-05-20, session oglasino-router-use-backend-check-fail-open-1)
**Found in:** `oglasino-router/src/index.ts:91-93`
**Detail:** The fail-open `try`/`catch` in the router worker wraps the reads for `maintenance.active` and `admin.bypass.disabled`, but the `CONFIG.get("use.backend.check")` call sits outside it. If that specific KV read throws (transient KV outage), the exception propagates and the request fails — the opposite of the worker's documented fail-open behavior. Trivial fix: wrap or move the read inside the existing `try`. Found by Mastermind during the maintenance-split-propagation audit (2026-05-15). Linked from [`infra/cloudflare/maintenance.md`](infra/cloudflare/maintenance.md) Known limitations.

> **Fix:** the `CONFIG.get("use.backend.check")` read is now inside the existing fail-open `Promise.all` alongside the other two KV reads. A throw on the `use.backend.check` read now resets all three flags to `false` (full fail-open), matching the existing semantics for the other two. The previously-nested `if (!maintenanceActive) { const useBackendCheck = await ... }` block was collapsed into a single combined gate (`if (!maintenanceActive && useBackendCheck) { ... }`). Lint and tests pass (32/32).

---

## 2026-05-15 — Worker matrix comment doesn't mention `use.backend.check`

**Severity:** low
**Status:** fixed (2026-05-20, session oglasino-router-use-backend-check-fail-open-1)
**Found in:** `oglasino-router/src/index.ts:1-13`
**Detail:** The matrix-comment block at the top of the worker is the closest thing to an in-source spec for the gate logic, but it documents only the two split flags (`maintenance.active` and `admin.bypass.disabled`). `use.backend.check` is a third input to `maintenanceActive` that can independently force the gate on; a reader treating the header comment as the spec misses a real input. Documentation-only fix — extend the comment to cover all three inputs. Found by Mastermind during the maintenance-split-propagation audit (2026-05-15). Linked from [`infra/cloudflare/maintenance.md`](infra/cloudflare/maintenance.md) Known limitations.

> **Fix:** the top-of-file matrix-comment block in `src/index.ts` now names `CONFIG.use.backend.check` as the third input to `maintenanceActive`, with a footnote on the `maintenance.active=false` row explaining the probe-escalation interaction. The catch-block comment now states explicitly that all three flags fall back to `false`. Same session as the fail-open fix above.

---

## 2026-05-15 — Favorites page has brittle positional type-narrowing on initialProductsData

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-favorites-narrowing-1)
**Found in:** `oglasino-web/app/[locale]/(portal)/(protected)/favorites/page.tsx:52,64`
**Detail:** The page accesses `initialProductsData.products?.map(...)` where `initialProductsData` is typed `ProductOverviewsDTO | undefined`. The preceding `if (loading) return <LoadingOverlay />` guard prevents an actual undefined deref at runtime (`getFavorites` always resolves via `EMPTY_OVERVIEWS` on error, and `.then` runs before `.finally`), but the narrowing is purely positional and brittle to any refactor that moves the access or the guard. Type-tightening cleanup, no user-visible bug. Found by Docs/QA during the QA-Preparation Notifications + Favorites batch.

**Fix:** `useState` retyped from `ProductOverviewsDTO | undefined` to `ProductOverviewsDTO` with explicit empty initial value; positional `?.` chains on `.products` at both `excludeIds` sites removed; `if (loading)` UX guard retained.

---

## 2026-05-15 — Favorites "recently viewed" carousel is misnamed

**Severity:** low
**Status:** parked
**Found in:** `oglasino-web/app/[locale]/(portal)/(protected)/favorites/page.tsx:50`
**Detail:** The bottom carousel uses the `recently.viewed.title` translated heading, but its filter is `{ excludeIds: <favorited-ids> }` — no "recently viewed" criterion is sent. The heading promises recently-viewed products; the filter delivers something else. Same family of label/filter mismatch as the `user-page` "Similar products" carousel (documented there as a pitfall of the unfinished extra-products feature). If the extra-products feature is still WIP, this may be a known rough edge rather than a defect to fix — triage accordingly. Found by Docs/QA during the QA-Preparation Notifications + Favorites batch.

> **Parked (2026-05-27):** This finding is tied to the unfinished extra-products feature. The favorites page renders two near-identical carousels (one with `applyRandom: true`, the other without) and the "recently viewed" heading is at odds with the actual `{ excludeIds }` filter. There is no "recently viewed" tracking mechanism in the codebase. Resolving the carousel naming requires a product decision on the extra-products feature itself: rename to match what the filter does, delete the redundant carousel, or build the missing tracking. Re-surfaced when the extra-products feature reaches active work.

---

## 2026-05-14 — Owner-view hydration flash on UserDetails action buttons

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-hydration-flashes-1)
**Found in:** `oglasino-web/src/components/client/UserDetails.tsx:51-57`
**Detail:** `iamActive` is computed via `useMemo` over the auth store. On first paint after hydration, a signed-in owner viewing their own `/user/<id>` briefly sees the Follow / Send-Message / Report buttons before they disappear once the auth store resolves. Same pattern as the open row "Owner-view hydration flash on the ProductFunctions bar" (`ProductFunctions.tsx:36-44`) but a separate component — a fix touches `UserDetails` independently. Found by Docs/QA during the QA-Preparation public User page authoring.

**Fix:** auth-derived button visibility now gated on a new shared hook `useAuthResolved` that composes Firebase `onAuthStateChanged` ready-state with `useAuthStore` user-loaded state, mirroring the existing `HeaderNavButtons` precedent. Replaces the `useEffect`/`useMemo` patterns that flashed because they updated state after first paint. The hook is at `src/lib/hooks/useAuthResolved.ts`.

---

## [HIGH] ProductErrorCode is a dumping ground for cross-cutting error codes

**Status:** fixed
**Surfaced:** 2026-05-14, connection-pool BugFixer chat (CurrentLanguageFilter Phase 2 work)
**Severity:** high — not a runtime bug, but a structural problem that actively misleads. The "Cross-cutting (system-level)" group inside `ProductErrorCode` is a precedent that gets cited and copied; every new cross-cutting code added there makes the next misplacement look more justified.

### The problem

`ProductErrorCode` contains error codes that have nothing to do with products. It already has an explicit "Cross-cutting (system-level)" group, and known occupants include:

- the rate-limit code consumed by `RateLimitFilter`
- `LANG_MISSING_OR_INVALID`, added during this chat by `CurrentLanguageFilter` Phase 2 — the engineer correctly flagged the placement felt wrong but followed the existing precedent because no proper home existed

There may be others — a full audit of `ProductErrorCode.values()` is the first step.

### Why it wasn't fixed in the chat that surfaced it

The chat that introduced `LANG_MISSING_OR_INVALID` is a BugFixer chat. Fixing _only_ the one code we added would leave `ProductErrorCode` inconsistently cleaned — some cross-cutting codes moved, others not — which is worse than leaving it whole, because it removes even the bad-but-consistent pattern. The honest fix is the full refactor: create a proper cross-cutting error-code enum and move _all_ non-product cross-cutting codes out. That refactor touches every consumer of the moved codes, the error-contract wiring, the `ProductErrorCodeTest` seed-row enforcement (which iterates `ProductErrorCode.values()`), and the translation seed files. That is feature-sized work, out of scope for a bug-fixer chat — so it is being routed here deliberately, not parked to dodge it.

### What the fix involves (for whoever picks this up)

1. Audit `ProductErrorCode` — list every constant that is not genuinely product-specific.
2. Decide the target enum name and create it (candidates discussed but not settled: `GeneralErrorCode`, `SystemErrorCode`, `RequestErrorCode`). Confirm whether any suitable enum already exists before creating one.
3. Move all non-product cross-cutting codes to the new enum.
4. Update every consumer — `RateLimitFilter`, `CurrentLanguageFilter`, and whatever else references the moved codes.
5. Update `ProductErrorCodeTest` — and check whether an equivalent seed-enforcement test should exist for the new enum.
6. Translation seed rows for the moved codes: confirm whether the `translationKey` values change (if the keys stay `product.system.*` that's itself a smell worth fixing; if they become `system.*` / `general.*` the seed rows move/rename).
7. `./mvnw spotless:check` + `./mvnw test` green.

### Known scope notes

- `LANG_MISSING_OR_INVALID` currently lives in `ProductErrorCode` with translation key `product.system.lang_missing_or_invalid` and seed rows in all four locale files (`0001-data-web-translations-{EN,RS,RU,CNR}.sql`). The `product.system.*` key prefix on a non-product code is part of the same smell.
- This is a refactor, not a behavior change — done right, no error response changes shape or code value from the client's perspective except the codes' _enum home_ (and possibly their `translationKey` if you choose to rename).

> **Resolved 2026-05-29** by the `system-error-code-split` feature (`web-stable`; backend `dev` + web `stage`). `ProductErrorCode`'s 11 non-product codes were split into domain enums: `SystemErrorCode` (7), `UserErrorCode` (4), with `ReportErrorCode` already separate. `ProductErrorCode` retains 36 genuine product codes. The misprefixed `product.system.*` / `product.user.*` translation keys were renamed to `system.*` / `user.*`; the `code` value on the wire is unchanged. `GlobalExceptionHandler.resolveTranslationKey` now resolves against all enums via a single registry built at class load, and seed-coverage enforcement was added per enum. See `features/system-error-code-split.md` and `decisions.md` 2026-05-29.

---

## 2026-05-14 — NumberOfViews fires a redundant duplicate fetch

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-numberofviews-redundant-fetch-1)
**Found in:** `oglasino-web/src/components/client/NumberOfViews.tsx:11-13`
**Detail:** The view-count `useEffect` depends on `[numberOfViews]`. It fires once with state at 0, fetches, then `setNumberOfViews(res)` changes the dependency and it fires again — two requests per render for one value, identical responses. No user-visible bug, perf-only. Fix: change the dependency to `[productId]`. Found by Docs/QA during the QA-Preparation Product Detail page authoring.

**Fix:** `useEffect` dependency changed from `[numberOfViews]` to `[productId]`. Same session also added a pre-fetch render guard (`if (numberOfViews <= 0) return null`) to hide the placeholder flash, and a `.catch(() => setNumberOfViews(0))` on the promise chain to swallow rejection cleanly (service layer continues to log via `logServiceError`).

---

## 2026-05-14 — Hardcoded Serbian tooltips on the product detail page

**Severity:** low
**Status:** fixed (2026-05-20, sessions oglasino-backend-product-tooltip-seed-1 + oglasino-web-product-tooltip-swap-1)
**Found in:** `oglasino-web/src/components/server/ProductDetails.tsx:72`, `oglasino-web/src/components/client/NumberOfViews.tsx:16`
**Detail:** The favorites-count and views-count tooltips use literal Serbian strings (`"Koliko je puta proizvod sacuvan"`, `"Koliko je puta proizvod pogledan"`) instead of next-intl keys — EN / RU / ME visitors see untranslated Serbian. A Part 6 (translations) violation in `oglasino-web` source. Found by Docs/QA during the QA-Preparation Product Detail page authoring.

> **Fix:** two new keys seeded in the `COMMON` namespace — `product.tooltip.favorites_count.label` (EN 2374, RS 4474, RU 6574, CNR 274) and `product.tooltip.views_count.label` (EN 2375, RS 4475, RU 6575, CNR 275). Web swap replaced the literal `"Koliko je puta proizvod sacuvan"` (corrected to `sačuvan` in the Serbian seed) at `ProductDetails.tsx:72` and the literal `"Koliko je puta proizvod pogledan"` at `NumberOfViews.tsx:16` with `tCommon('product.tooltip.favorites_count.label')` and `tCommon('product.tooltip.views_count.label')` respectively. Backend tests 501 passing, web tests 154 passing, lint/tsc clean.

---

## 2026-05-14 — Owner-view hydration flash on the ProductFunctions bar

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-hydration-flashes-1)
**Found in:** `oglasino-web/src/components/client/ProductFunctions.tsx:36-44`
**Detail:** The actions bar's visibility is gated on a `useEffect`-driven `allow` flag (`setAllow(user.id === owner.id)`). On first paint after hydration the signed-in owner briefly sees the Call / Message / Favorite / Share / Review / Report bar before it disappears once the auth store resolves. The owner/stranger boundary itself is correct (captured as a topic pitfall); the flash is unintended. Fix: gate the bar with the auth selector synchronously, or render null while auth is loading. Found by Docs/QA during the QA-Preparation Product Detail page authoring.

**Fix:** auth-derived button visibility now gated on a new shared hook `useAuthResolved` that composes Firebase `onAuthStateChanged` ready-state with `useAuthStore` user-loaded state, mirroring the existing `HeaderNavButtons` precedent. Replaces the `useEffect`/`useMemo` patterns that flashed because they updated state after first paint. The hook is at `src/lib/hooks/useAuthResolved.ts`.

---

## 2026-05-14 — Messages "Report" dropdown item is inert

**Severity:** medium (escalated from low during 2026-05-16 triage)
**Status:** fixed (2026-05-16, session oglasino-web-messages-report-wiring-1)
**Found in:** `oglasino-web/src/messages/components/Messages.tsx:158-160`
**Detail:** The conversation-header kebab dropdown renders a "Report" item (`tMessages('func.report')` label) with no `onClick` handler — visible to every user who opens the dropdown, does nothing on click. May fall under the `state.md` backlog item "Chat & messaging cleanup" (`planned`), but no `issues.md` entry recorded it. Found by Docs/QA during the QA-Preparation Messages page authoring.

**Fix:** wired the inert "Report" dropdown item in `Messages.tsx` to open the existing `REPORT_DIALOG` via `useDialogStore.openDialog(DialogId.REPORT_DIALOG, ...)`, mirroring the payload shape used by `ReportButton.tsx` on the public user page. `reportedUserId` is derived from `activeChat.withUser.id` (the same state the conversation header uses for the avatar/name). Addendum also removed the dead `onSendReport` field from `ReportDialogProps` (zero code consumers; `grep` confirmed) and the corresponding `as ReportDialogProps` cast plus unused import in both `ReportButton.tsx` and `Messages.tsx`.

---

## 2026-05-14 — Messages chat list capped at 15 in the rendered UI

**Severity:** medium
**Status:** fixed (2026-05-20, messaging feature Brief 2 W8 + Brief 6a)
**Found in:** `oglasino-web/src/messages/components/Chats.tsx`
**Detail:** The conversation list renders only the most-recent Firestore page (limit 15). The chat store implements `loadMoreChats` and tracks `hasMoreChats` / `loadingMoreChats`, but `Chats.tsx` renders no "load more" affordance. A user with more than 15 conversations only ever sees the 15 most-recent by `lastUpdated`; older threads resurface only when the other party sends a new message that bumps `lastUpdated`. Invisible data-loss in the UX — the store supports paging, the UI doesn't expose it. Found by Docs/QA during the QA-Preparation Messages page authoring.

> **Fix:** Brief 2 W8 added a "Load more" button at the bottom of `Chats.tsx`, wired to `useChatStore.loadMoreChats`, visible when `hasMoreChats` is true. Brief 6a swapped the placeholder reuse of `MESSAGES_PAGE.messages.load.more` to a dedicated `MESSAGES_PAGE.chats.load.more` key — seeded in all four locales (EN/RS/RU/CNR) and consumed by the swapped call site in `Chats.tsx`.

---

## 2026-05-14 — Messages silent text-only send failure

**Severity:** medium
**Status:** fixed (2026-05-20, messaging feature Brief 2 W5)
**Found in:** `oglasino-web/src/messages/store/useChatStore.ts` (`sendMessage` catch branch)
**Detail:** A Firestore write failure on a text-only message send logs `console.error`, runs orphan-image cleanup (a no-op for text), and rolls back the optimistic message by filtering out its tempId. No toast, no inline error, no surfaced state — the sender sees the message simply vanish. Image-upload failures in `MessageInput` do surface as `notify.error` toasts, so this is an asymmetry: a critical action fails silently on the text path. Found by Docs/QA during the QA-Preparation Messages page authoring.

> **Fix:** Brief 2 W5 lifted the send-failure path to the React boundary. `useChatStore.sendMessage` now rethrows after orphan-image cleanup and optimistic-message rollback; the React caller in `Messages.tsx` awaits and calls `notify.error({ id: 'message-send-failed', title: tMessages('messages.send.failed.toast') })` on rejection. Same pattern as `MessageInput.tsx`'s image-upload-failure toast. `tempReceiver` and `tempProductContext` are preserved on failure so retry from the same context works. Translation key seeded by Brief 4 in all four locales.

---

## 2026-05-14 — Privacy and Terms render English markdown across all locales

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/privacy/page.tsx:14` (and the terms equivalent)
**Detail:** Both legal pages fetch a hardcoded GitHub raw markdown URL (`memento-tech/oglasino-platform/main/privacy.md` / `terms.md`) with no locale variant — every locale (EN / SR / RU / ME) renders the same English content. `privacy/page.tsx` carries a `// TODO Create locale privacy.md`. Found by Docs/QA during the QA-Preparation simple-pages batch. Surfaced to QA users as a "Known issue" pitfall on the `privacy-page` and `terms-page` QA topics; this entry tracks the underlying code fix.

> **Fix:** Interim-state comments added to both `privacy/page.tsx` and `terms/page.tsx` in session `oglasino-web-tier2-batch-1` (2026-05-28). URLs remain hardcoded at English `.en.md` files. Per-locale content is blocked on lawyer review of Serbian translations. The swap to locale-aware URLs is tracked as a new `issues.md` entry (2026-05-27 — "Per-locale legal markdown content (privacy + terms)").

---

## 2026-05-14 — `markdown.fild.load` translation key is a typo

**Severity:** low
**Status:** fixed (2026-05-20, by Igor directly)
**Found in:** `oglasino-web/src/components/server/MarkdownViewer.tsx:18,22`
**Detail:** `MarkdownViewer` references `tErrors('markdown.fild.load')` — "fild" is almost certainly meant to be "file". The rendered string is whatever the translation row says, so there is no user-visible break unless the row is also misnamed, but the key should be corrected across its registration and all locales. Found by Docs/QA during the QA-Preparation simple-pages batch.

> **Fix:** key renamed `markdown.fild.load` → `markdown.field.load` in `MarkdownViewer.tsx` and the four locale seed files. Applied by Igor directly.

---

## 2026-05-14 — Terms page does not emit JSON-LD; Privacy page does

**Severity:** low
**Status:** fixed (2026-05-24, SEO foundation feature — brief 1 normalized both pages; brief 1b unified them as `WebPage + about`; brief 5g deleted JSON-LD from both entirely. Resolved by retraction.)
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/terms/page.tsx`
**Detail:** `privacy/page.tsx` emits a JSON-LD `<script>` tag for SEO; `terms/page.tsx` does not. The two legal pages are otherwise the same shape. SEO inconsistency — either Terms should emit JSON-LD too, or the difference should be deliberate and documented. Found by Docs/QA during the QA-Preparation simple-pages batch.

**Investigated 2026-05-16 (terms-json-ld-1 session).** The visible asymmetry is a symptom of a broader SEO delivery bug — both pages already emit JSON-LD via `metadata.other['application/ld+json']`, but Next.js renders that as `<meta>` tags which search engines do not parse as structured data. Privacy additionally has a malformed inline `<script>` that stringifies the Next.js `Metadata` object instead of the Schema.org helper. Both findings rolled into the 2026-05-16 entry "SEO JSON-LD delivered as `<meta>` tags instead of `<script>`, project-wide."

---

## 2026-05-14 — Pricing hero renders the same translation key on both subtitle lines

**Severity:** low
**Status:** fixed (2026-05-20, by Igor directly)
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/pricing/page.tsx:32-34`
**Detail:** The hero H1 calls `tPricing('hero.subtitle.line1')` on both of its two lines — almost certainly meant to be `line1` then `line2`. Result is a visible duplicate line under the hero title in all locales. Found by Docs/QA during the QA-Preparation simple-pages batch.

> **Fix:** `pricing/page.tsx` second subtitle line now calls `tPricing('hero.subtitle.line2')` instead of duplicating `line1`. Applied by Igor directly.

---

## 2026-05-14 — Bad catalog slug produces two different failure modes depending on position

**Severity:** medium
**Status:** fixed (2026-05-20, session oglasino-web-catalog-slug-failure-unify-1)
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:92-94` (server-side first-slug redirect to home); `oglasino-web/src/components/client/product/ProductBreadcrumbs.tsx:55-61` (client-side fallback to `window.location.href = '/error'` when the breadcrumb chain resolves empty on `/catalog`).
**Detail:** A catalog slug that does not resolve produces two different observable outcomes depending on where it sits in the URL: an invalid first slug server-redirects to home, while a partially-broken breadcrumb chain has a client-side fallback to `/error`. Same root cause — a slug that doesn't resolve — two different results. Intended behavior per Igor: messing with catalog routing should show a "category not found" error, consistently. Same root cause should produce the same outcome. Related to the silent-degradation entry below — same root cause, different observable outcome. Cross-referenced from the `category-navigation` and `catalog-page` QA topics as Known-issue pitfalls; when this bug is fixed, the topic pitfalls must flip to reflect the new behaviour.

> **Fix:** `getCategoriesFromPathSlugs` in `[[...slugs]]/page.tsx` is now strict — sub-slug and final-slug misses return `undefined` like first-slug misses. The page guard swapped `redirect()` for `notFound()`. An explicit `if (slugs.length === 0)` redirect-to-home guard sits above the resolver so the bare `/catalog` URL keeps today's 307 behavior. New `app/[locale]/not-found.tsx` added so locale-layout providers (`NextIntlClientProvider`, `setRequestLocale`) wrap the 404 render; exports `metadata` with hardcoded `title: 'Not Found'` and `robots: { index: false, follow: false }`. `ProductBreadcrumbs.tsx`'s `useEffect` redirecting to `/error` was deleted (unreachable post-fix). All three failure modes now produce HTTP 404 at the original URL. `tsc`/`lint`/`test`/`prettier` clean.

---

## 2026-05-14 — Invalid catalog sub-slug degrades silently to parent category

**Severity:** medium
**Status:** fixed (2026-05-20, session oglasino-web-catalog-slug-failure-unify-1)
**Found in:** `oglasino-web/app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:38-44` (sub/final slug resolution — `subCategory` and `finalCategory` stay `undefined` on a bogus slug, and the page renders against the deepest level that did resolve with no error and no redirect).
**Detail:** An invalid second or third URL slug on the catalog route degrades silently to the parent category level — `/catalog/cars/bogus` renders identically to `/catalog/cars`, with no error, no redirect, no "category not found". A bad sub-slug produces a result that looks like the parent category, which can read as the parent filter being broken. Intended behavior per Igor: a bad slug at any level should show a "category not found" error, not silently fall back to the parent. Related to the two-failure-modes entry above — same root cause. Cross-referenced from the `category-navigation` and `catalog-page` QA topics as Known-issue pitfalls; when this bug is fixed, the topic pitfalls must flip to reflect the new behaviour.

> **Fix:** same root cause as the paired 2026-05-14 entry above; closed by the same session. See that entry's fix block for details.

---

## 2026-05-14 — `perPage` is uncapped server-side

**Severity:** medium
**Status:** fixed (2026-05-20, session oglasino-backend-perpage-cap-1)
**Found in:** `oglasino-backend` — `PagingDTO.getPageable()` → `PageRequest.of(page, perPage)`, called from `DefaultProductsFilterQueryBuilder.buildQuery` via `SearchProductsDTO.getPagingSafe()`
**Detail:** All product-search endpoints (`/api/secure/products`, `/api/secure/admin/products`, `/api/public/product/search`, plus their three autocomplete siblings) pass client-supplied `perPage` straight to Spring Data with no maximum. `RateLimitFilter` mitigates abuse, but a hard cap (e.g. 100) is appropriate hygiene. Discovered in the product-filtering bug sweep (backend audit, Q3).

> **Fix:** `PagingDTO.java` gained a private `MAX_PER_PAGE = 100` constant and a private `cappedPerPage()` helper returning `Math.max(1, Math.min(perPage, MAX_PER_PAGE))`. Both `getPageable()` overloads (no-arg and `Sort`-variant) now call `cappedPerPage()` for the `PageRequest.of` size argument. Cap inherits transparently through all 8 `getPageable()` callers including `SearchProductsDTO.getPagingSafe()` on the non-null path. Direct `PageRequest.of` call sites that bypass `getPageable()` (PublicReviewController, DefaultReviewService.getMyReviews, scheduled jobs) use hardcoded or config-driven sizes and were deliberately out of scope. Lower bound clamps `perPage <= 0` to 1, changing observable behavior on `perPage = 0` from HTTP 500 to HTTP 200 size-1. New `PagingDTOTest` covers 9 cases. `spotless` + 511 tests green.

## 2026-05-14 — `PORTAL_SEARCH` silently drops body `productStates` / `moderationStates`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/elasticsearch/generator/impl/ProductStateQueryGenerator.java` (`PORTAL_SEARCH` branch)
**Detail:** Public product search hard-pins `productState=ACTIVE AND moderationState=APPROVED` and ignores any client-supplied state filters. This is correct and intentional behavior — a public caller must not be able to request PENDING/REJECTED products — but the shared `ProductsFilterDTO` wire shape doesn't communicate that these fields are inert on the portal path. No code fix wanted; logged so the silent-drop is recorded. Discovered in the product-filtering bug sweep (backend audit, opportunistic flags).

> **Fix:** Method-level Javadoc added to `ProductStateQueryGenerator.wrapWithStateSearchMode` in session `oglasino-backend-round-2-fixes-1` (2026-05-27) documenting the PORTAL_SEARCH trust-boundary hard-pin, the DASHBOARD_SEARCH / ADMIN_SEARCH behavior, and the null-`productsFilter` propagation from two of three call sites. Security mechanism (hard-pin to ACTIVE+APPROVED) was already correct; only documentation was missing.

## 2026-05-14 — `baseSite` tenant scoping was never audited

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-backend` — `BaseSiteQueryGenerator`, `BaseSiteContext`
**Detail:** `BaseSiteQueryGenerator` and `BaseSiteContext` were referenced but not opened during the product-filtering audit. Whether tenant isolation is header-derived or body-derived is unconfirmed. Needs its own focused audit; will likely become its own documentation page. Discovered in the product-filtering bug sweep (backend audit, out-of-scope note).

> **Audit (2026-05-28, session `oglasino-backend-trust-boundary-audit-1`):** Verdict — **clean** per conventions Part 11. The `X-Base-Site` header (with `?baseSite=` fallback) is validated against the DB via `DefaultBaseSiteCacheService.getBaseSiteForCode`, which sources from `baseSiteRepository.findActiveBaseSiteCodes()` — a client cannot inject a fake or inactive tenant code; an unknown code throws. The header is consumed only as an Elasticsearch query-scoping filter on public catalog data (`BaseSiteQueryGenerator`); it never feeds an authorization, moderation, or state-transition decision. The persisted `Product.baseSite` at create time is `user.baseSite` (server-derived from the loaded `User` entity, `DefaultProductService:108`), not the header — a client cannot move products between tenants. The server-trusted Part 11 `OglasinoAuthentication.baseSiteId` has exactly one downstream consumer (`DashboardProductController.canUserCreateProduct:158`), and it uses the value as a "user setup complete" gate, not a tenant-isolation gate. Two minor adjacent observations recorded in the audit body but not promoted to separate issues: `getBaseSiteForCode` returns a 500 on unknown codes (Part 7 — codes-only) and `DefaultProductService.getBaseSite()` prefers the header when domains match (fragile shape, no current bug since base sites are unique per domain).

## 2026-05-14 — Backend errors are swallowed and rendered as "empty results"

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-web/src/lib/service/nextCalls/productsSearchService.ts` (all three `getXProducts` calls)
**Detail:** Every product-search server call catches errors and returns `EMPTY_OVERVIEWS = { products: [], totalNumberOfProducts: 0 }`. A backend outage is therefore indistinguishable from a genuinely empty result set — the user sees "no products" either way. Pre-launch this is tolerable; for production it warrants at least server-side logging and ideally a distinguishable error state. Discovered in the product-filtering bug sweep (web audit, all surfaces).

> **Fix:** Two-part resolution. (1) Structured logging shipped in session `oglasino-web-service-log-structured-1` (2026-05-28) — backend outages now observable in server-side logs. (2) Error-state propagation shipped in session `oglasino-web-product-search-error-state-1` (2026-05-28) — `getProducts` in both `nextCalls/` and `reactCalls/` returns `ServiceResult<ProductOverviewsDTO>` discriminated union. Six SSR consumer pages render an error state distinguishable from empty results. Three client-side pagination handlers surface failures via toast. Scope is product search only; other services retain sentinel-return behavior.

## 2026-05-14 — No request cancellation on the pagination path

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/components/client/product/ProductList.tsx`, `src/lib/service/reactCalls/productsSearchService.ts` (paging branch)
**Detail:** `AbortController` was added to autocomplete during the sweep, but paging requests still have no cancellation — rapid page-button clicks can let a stale response resolve last and win. Deferred deliberately; would need a coordinated pattern across `ProductList` and the service layer. Discovered in the product-filtering bug sweep.

> **Fix:** `AbortController` ref added to `ProductList` in session `oglasino-web-tier2-batch-1` (2026-05-28). Both `onDisplayPageChange` and the favorites refetch effect abort the previous in-flight request before issuing a new one and guard state updates on `!signal.aborted`. Signal plumbed through the full chain to `BACKEND_API.post({ signal })` in `getProducts` and `getFavorites`, matching the existing autocomplete AbortController pattern. All seven `ProductList` surfaces inherit the fix uniformly.

## 2026-05-14 — `parseFiltersFromQueryParams` has a dead `| undefined` return type

**Severity:** low
**Status:** fixed (2026-05-15, session oglasino-web-filtersHelper-return-type-1)
**Found in:** `oglasino-web/src/lib/utils/filtersHelper.ts:44`
**Detail:** The function is typed `ProductsFilterDTO | undefined` but no code path returns `undefined`; the home page's `else` branch handling that case (`app/[locale]/(portal)/(public)/page.tsx:38-43`) is unreachable. Dead code, no behavior impact — a type-tightening cleanup. Discovered in the product-filtering bug sweep (web session summary).

**Fix:** return type narrowed; two consumer-side dead branches removed (home `if/else`, catalog `filtersData &&` guard); three other callers confirmed already-correct and untouched.

## 2026-05-14 — `/admin/products/[userId]` renders an empty fragment on zero products

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-admin-filter-pipeline-fix-1)
**Found in:** `oglasino-web/app/[locale]/admin/products/[userId]/page.tsx:34`
**Detail:** When an admin views a user with no products, the page returns `<></>` — no message, no context. Should show a "no products for this user" state. Discovered in the product-filtering bug sweep (web audit, Suspected Bug context).

**Fix:** replaced `<></>` with the same conditional empty-state-vs-list pattern used by the sibling `/admin/products` page. Renders `tDash(filtersApplied ? 'products.filters.empty.list' : 'products.empty.list')` in a centred italic empty-state box, with the filter chip row and side panel still visible. Reused existing `DASHBOARD_PAGES` keys; no new translation key needed.

## 2026-05-14 — `/admin/products/[userId]` filter chips change without refetch or URL sync

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-admin-filter-pipeline-fix-1)
**Found in:** `oglasino-web/src/components/client/initializers/FilterManager.tsx:52-66` (`isAllowedPath()` excludes `/admin/products/[userId]`)
**Detail:** Two root causes, both addressed in the fix session: (1) `FilterManager.tsx`'s HYDRATE FROM URL effect had deps `[baseSite]` and ignored `pathname` changes, so in-app nav from a disallowed admin route left `hydratedRef.current === false` and the SYNC effect silently early-returned on chip clicks — the dominant root cause shared with the 2026-05-16 entry on `/admin/products`. (2) `/admin/products/[userId]` was excluded from `FilterManager.isAllowedPath()` entirely; even with the deps-array bug fixed, the path was still gated off. Fix: replaced `hydratedRef` boolean with per-path `lastHydratedPathRef` and added `pathname` to deps; extended `isAllowedPath()` to use `pathname.startsWith('/admin/products')` instead of exact match; added defensive `setTimeout(() => router.refresh(), 0)` after `router.replace` to match the established admin pattern at `src/components/admin/FiltersPanel.tsx:65-69`. The same deps-array bug had a latent twin on `/owner/products`; incidentally fixed by the same change.

## 2026-05-14 — `conventions.md` says `Auth: Firebase Auth (JWT)` and references "claims from the JWT"

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-docs/meta/conventions.md` Part 9 (stack table) and Part 11 (trust-boundary text)
**Detail:** Part 9's stack table and Part 11's trust-boundary text both describe auth as JWT-based. The actual system uses Firebase auth as the source of truth (Firebase-context-derived identity server-side), confirmed by the backend filtering audit (`SearchModeQueryGenerator.DASHBOARD_SEARCH` reads from `SecurityContextHolder` populated by `FirebaseAuthFilter`). Same class of error as the recently-corrected Gradle/Maven mismatch — a convention that contradicts reality and will mislead a future agent. Needs an Igor-and-Mastermind correction pass on Parts 9 and 11. Discovered in the product-filtering bug sweep. **Resolved 2026-05-15:** Parts 9 and 11 corrections drafted in `.agent/2026-05-15-oglasino-docs-connection-pool-closeout-1.md` (For Mastermind section) and pending Igor's apply to `conventions.md` — Docs/QA does not edit `conventions.md` per CLAUDE.md hard rule. The drafted change replaces the JWT-mechanism framing with Firebase ID token + `SecurityContextHolder`-populated `OglasinoAuthentication`, using the precise codebase terms (`FirebaseAuthFilter`, `redisUserAuth`).

## 2026-05-14 — Product-validation feature doc needs a retro-fit pass against current conventions

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-docs/features/product-validation.md`
**Detail:** Written before the `### Images` convention (Part 1) and the Part 5 session-naming change existed. It should be brought in line with the current rulebook. Not urgent — tagged for a future docs session, not part of any active work. **Resolved 2026-05-15:** audited against current conventions during the connection-pool close-out session. Part 1 `### Images` is satisfied — 4 image references at lines 417, 465, 505, 566 all use kebab-case lowercase descriptive filenames under `assets/`, all have a preceding HTML comment describing the intended screenshot, and all use standard markdown syntax with descriptive alt text. Part 5 (session-file naming) governs `.agent/` summary files, not feature-spec docs, so it is N/A for this file. No retro-fit edits needed.

## 2026-05-14 — Manual test reminder: pagination overlap across all five surfaces

**Severity:** low
**Status:** open
**Found in:** product-filtering bug sweep — manual QA queue
**Detail:** After the product-filtering web fixes, verify on a live stack that dashboard/admin/public-user pages 2+ show different products than page 1, on a scope with 25+ products. Igor ran an initial check during the sweep and it passed; this entry is the standing regression reminder. **Not a bug** — verification step that should not be lost. Surfaces in scope: home, category, owner dashboard, admin product list (including `[userId]`), public user-products.

## 2026-05-14 — Manual test reminder: random-ordering stability across pages

**Severity:** low
**Status:** open
**Found in:** product-filtering bug sweep — manual QA queue
**Detail:** Verify that paginating home/catalog with random ordering keeps a consistent shuffle across pages within one session, and reshuffles only on refresh (because `randomSeed` is regenerated per SSR call but stable across paging requests within one render). **Not a bug** — verification step.

## 2026-05-14 — Dead `free` field on backend `NewProductRequestDTO`

**Severity:** low
**Status:** fixed (2026-05-20, session oglasino-backend-newproductdto-free-field-delete-1)
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java` (field `private boolean free` plus `isFree()`/`setFree()`)
**Detail:** Web removed `free` from its `NewProductRequestDTO` interface in web session 4 (free-zone derived from `topCategory.freeZone` instead). Backend still carries the field with getter/setter; no production code reads it (grep across `src/main/java` shows zero call sites that consume `request.isFree()` or `requestDTO.isFree()` on a product request). Jackson tolerates the absent field on incoming JSON. Dead field worth deleting in a follow-up chore to match the trust-boundary direction (free-zone is server-derived).

> **Fix:** `private boolean free`, `isFree()`, and `setFree()` removed from `NewProductRequestDTO.java`. Pre-edit grep confirmed zero call sites in `src/main/java` and `src/test/java`. Verified Jackson tolerates the absent field on incoming JSON: no `@JsonIgnoreProperties` on the DTO, no `spring.jackson.*` override flipping `FAIL_ON_UNKNOWN_PROPERTIES` to `true`. Legacy clients still sending `"free": true` are silently ignored per Spring Boot's default. `./mvnw spotless:check` and `./mvnw test` (501 passing) green.

## 2026-05-14 — `regionAndCity` declared in `createProductSchema` but unread by validator

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-regionAndCity-cleanup-1)
**Found in:** `oglasino-web/src/lib/validators/productSchemas.ts`
**Detail:** `createProductSchema` still declares `regionAndCity: z.object({...}).nullable().optional()` but `validateProduct` does not consult that field — region and city are user-derived and not user-settable on the create wizard. Harmless as-is; cosmetic cleanup.

**Fix:** one-line deletion in `productSchemas.ts`; grep confirmed zero consumers and zero type derivations. `priceSchema` `export` investigated as adjacent and kept (legitimate cross-module consumer in `productValidator.ts`).

## 2026-05-14 — `field-price` scroll target is a no-op on update page

**Severity:** low
**Status:** fixed (2026-05-16, session oglasino-web-field-price-scroll-target-2)
**Found in:** `oglasino-web/app/[locale]/owner/products/[productId]/page.tsx`
**Detail:** When a price-only error fires on save, the page calls `document.getElementById('field-price')?.scrollIntoView(...)`, but the two price `<Input>` renders generate their own internal ids and do not accept an external `id`. Inline error still renders correctly next to the price input; only the scroll-into-view is a no-op. Fix is to wrap the price inputs in a `<div id="field-price">`.

**Fix:** rewrote to viewport-aware `data-field="price"` lookup on both responsive wrappers (mobile and desktop). Both client-validation and server-error scroll pickers updated to pick the visible element via `querySelectorAll` + `offsetParent`. Same session also moved `field-images` id from the conditional error `<p>` to the unconditional outer wrapper for pattern consistency, and added a system-error scroll case (`SYSTEM_ERROR_KEY` branch wrapping the system-error `<p>` in `<div id="field-system-error">`) so rate-limit-style errors auto-scroll like per-field errors.

## 2026-05-14 — Boot-time moderation config audit logs ERROR but does not fail boot

**Severity:** low
**Status:** fixed (2026-05-20, session oglasino-backend-moderation-audit-fail-fast-1)
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/moderation/ContentValidationConfig.java` (`auditRequiredConfig`)
**Detail:** Missing/blank required keys are surfaced at boot via `log.error(...)` but the listener does not throw — the first runtime moderation call against an absent key throws `IllegalStateException`. Acceptable for dev environments that may run with an un-seeded config table. A `@Profile`-gated boot-fail (or env-toggle) for production would convert the silent-misconfig window into a hard startup failure. Framed as enhancement, not defect.

> **Fix:** `auditRequiredConfig` now throws `IllegalStateException` on the first missing or blank required key encountered, aborting the Spring application context at boot on every environment (no profile gate, no env toggle). Exception message names the offending key and instructs the operator to seed it. Existing positive seed test plus one new negative test (`auditRequiredConfig_throwsOnFirstMissingOrBlankRequiredKey`) cover the behavior. Seed file verified to carry every required key — boot passes on a fresh checkout. `./mvnw spotless:check` and `./mvnw test` (502 passing, +1 new) green.

## 2026-05-14 — Web component-render test coverage gap (testing-library not installed)

**Severity:** low
**Status:** open
**Found in:** `oglasino-web/package.json`
**Detail:** `@testing-library/react`, `@testing-library/user-event`, `happy-dom`, and `jsdom` are all absent. Web tests are pure-function and axios-mock only — dialog-rendering assertions (button-disabled, spinner present, inline-error elements) are covered indirectly via logic-level tests against `validateProduct`, `preValidateProductBasics`, and `productStepMapping`. End-to-end smoke is the first surface that exercises the dialog rendering. Future infra chore: install the testing stack so dialog tests can land in thin follow-ups.

## 2026-05-14 — Keyword-stuffing ratio multipliers not lifted to config

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/moderation/analyzer/KeywordStuffingAnalyzer.java`, `SpammyDescriptionAnalyzer.java`
**Detail:** Both analyzers pass a `KeywordStuffingDetector.Tuning` record with hardcoded `baseRatio`, `ratioDecayPerWord`, `ratioFloor`, and `minOccurrenceRatio` values. Other thresholds are admin-configurable via `ConfigurationService`; these heuristic ratios are not. Deliberately out of scope for the validation refactor — lifting them is a separate, larger conversation about how much of the heuristic should be admin-tunable.

> **Fix:** All ten hardcoded ratio values (five per analyzer) lifted to `ConfigurationService` via `getRequiredIntConfig` / `getRequiredDoubleConfig` in session `oglasino-backend-keyword-stuffing-config-lift-2` (2026-05-27). Ten seed rows added to `data-configuration.sql`. Boot-time audit (`GLOBAL_REQUIRED_KEYS`) and seed-coverage test (`ConfigurationSeedTest.REQUIRED_KEYS`) both extended. Four placeholder rows (IDs 23–26) deleted. 647 tests passing.

## 2026-05-14 — Malformed 429 leaves create-wizard silently blocked

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-web/src/lib/service/reactCalls/productService.ts`
**Detail:** Web session 4 removed the defensive `ensureSystemErrorKey` helper that synthesised a `__system` rate-limit entry if the 429 response body was malformed. Trade-off was made deliberately (trust the contract, surface upstream bugs instead of papering over them). Consequence: a malformed 429 (no `field:null + translationKey` entry) leaves the create wizard's Next button blocked without showing the inline rate-limit reason — the user is not advanced through, just not told why. If a malformed 429 is ever observed in manual/QA testing, the fix belongs at the backend/transport source, not at the client.

> **Fix:** Introduced a narrowly scoped 429-only `parseProductErrorsForStatus(status, response)` normalizer in `src/lib/service/reactCalls/productService.ts` in session `oglasino-web-batch1-1` (2026-05-28). When the response status is 429 and the parsed validation result has no `SYSTEM_ERROR_KEY` entry, the helper synthesizes one with translation key `product.system.rate_limited` (existing backend seed; no new keys). Applied at all six 429-handling sites: `createNewProduct` (resolved-non-2xx + thrown), `updateProductData` (resolved-non-2xx + thrown), and `preValidateProductBasics` (resolved + thrown). Non-429 statuses (400/422/403/500) still demand a contract-shaped body — synth is 429-only by design. The existing `if (result.errors.byField[SYSTEM_ERROR_KEY])` gate in `BasicInfoProductDialog.onNextInternal` (`:128`) now fires on every malformed 429, engaging the cooldown timer and the `disabled={preValidating || isRateLimited}` Next-button gate. `UploadedProductDialog.tsx`'s step-4 failure UX (`:199`) renders the synthesized rate-limit message in its bullet list. Four new tests in `productService.test.ts` cover the malformed-429 synth (empty `errors: []`, field-keyed-only entries) and confirm 429-only scope (malformed-400 does not synth). A `// NOTE:` block on the helper documents the rationale — a 429 from the Cloudflare edge router worker (conventions Part 8) may not pass through Spring's `RateLimitFilter` and so may lack the unified body shape. `parseProductValidationErrors` itself is unchanged.

## 2026-05-13 — Legacy unused regex rows in data-configuration.sql

**Severity:** low
**Status:** fixed (2026-05-20, session oglasino-backend-legacy-regex-seed-cleanup-1)
**Found in:** `oglasino-backend/src/main/resources/data/configuration/data-configuration.sql` ids 2–7
**Detail:** Six pre-feature config rows (`validation.regex.banned.words`, `validation.regex.repeated.chars`, `validation.regex.punctuation`, `validation.regex.emojis`, `validation.regex.promo` without language suffix, `validation.regex.spam.description`) remain seeded but have zero call sites — superseded by the per-language `validation.regex.*.{lang}` and `validation.banned_words.{lang}` rows. Backend session 3 flagged these for Mastermind as a follow-up "config-seed garbage collection" PR. Safe to delete in a chore.

> **Fix:** six rows for keys `validation.regex.banned.words`, `validation.regex.repeated.chars`, `validation.regex.punctuation`, `validation.regex.emojis`, `validation.regex.promo` (unsuffixed), `validation.regex.spam.description` deleted from `data-configuration.sql`. Pre-edit grep confirmed zero call sites in `src/main/java` and `src/test/java` for each exact unsuffixed key. Per-language `validation.regex.promo.{lang}`, `validation.banned_words.{lang}`, and `validation.regex.contacts.{lang}` rows untouched (live consumers). `ConfigurationSeedTest` and the full 501-test suite pass post-deletion.

## 2026-05-13 — 429 rate-limit response shape diverges from error contract

**Severity:** medium
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/security/filter/RateLimitFilter.java`
**Detail:** Original report: on HTTP 429, `RateLimitFilter` returned `{"error": "rate_limit_exceeded"}` with a `Retry-After` header, mismatching the project's `{errors: [{field, code, translationKey}]}` contract. **Resolved in backend session 1**: filter now writes `{"errors":[{"field":null,"code":"RATE_LIMITED","translationKey":"product.system.rate_limited"}]}` (see `RATE_LIMITED_BODY` constant). The unified shape applies to all rate-limited endpoints.

## 2026-05-13 — Commented-out `REGION_REQUIRED` annotation in `NewProductRequestDTO`

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java`
**Detail:** Original report: a commented-out `@NotNull(message = "REGION_REQUIRED")` line sat above the `filters` field. Region is derived from the user, not the request. **Resolved**: the commented line is no longer present in the file (deleted during one of the cleanup passes on the validation-refactor branch).

## 2026-05-13 — Pre-existing spotless violations on validation-refactor branch

**Severity:** low
**Status:** fixed
**Found in:** `oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/NewProductRequestDTO.java`, `oglasino-backend/src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java`
**Detail:** Two files on `feature/validation-refactor` failed `mvn spotless:check` before the pre-validate session began. One needed re-indentation of a commented line; the other had an unused `Region` import. Both fixed by Igor via `mvn spotless:apply` and committed separately under `chore: fix pre-existing spotless violations`.
