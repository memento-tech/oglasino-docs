# Decisions

Append-only log of decisions made across planning sessions. Igor commits entries. Mastermind drafts them. Engineers read but don't write.

Format: each decision has a date, a one-line summary, the reasoning, and the alternatives we considered. Newest at the top.

---

## 2026-06-11 — Contact form: exact-duplicate notice + server-side message minimum

Contact form exact-duplicate notice: a Redis marker on (lowercased effective
email + SHA-256 of the exact message), 6h TTL, stamped only after a successful
send and checked ahead of the per-email throttle so a duplicate never consumes a
cooldown/daily slot. Returned as 422 `CONTACT_DUPLICATE` / `contact.duplicate`
(ERRORS namespace), hand-built in the controller like `CONTACT_SEND_FAILED` (not a
`ContactErrorCode` constant). Server-side message minimum raised to 50, carried as a
distinct 400 `CONTACT_MESSAGE_TOO_SHORT` / `contact.message.too_short`.

---

## 2026-06-11 — Contact validation keys: accepted VALIDATION-freeze exception

The contact feature's validation keys (`email.empty`, `email.bad`,
`contact.message.empty` / `.too_short` / `.too_long`) live in the VALIDATION
namespace, which [conventions](meta/conventions.md) Part 6 Rule 1 marks frozen.
This is an accepted, documented exception: the keys were seeded there alongside
the pre-existing generic `email.*` keys for consistency, are shipped and working,
and will not be migrated. Future new validation keys still go to ERRORS per the
rule — this exception is specific to the already-seeded contact keys, not a
relaxation of the freeze.

---

## 2026-06-10 — Legal docs localized by reader language (Serbian default)

Privacy Policy and Terms of Use now serve per-language on both web and mobile,
reading the lawyer-reviewed files on memento-tech/oglasino-platform main:
`privacy-policy.{en,sr}.md` and `terms-of-use.{en,sr}.md`. The .en.md files the
apps already fetched are unchanged; this added the .sr.md-selection branch.

**The rule (Serbian is the default).** `en` or `ru` → `.en.md`; everything else
(`sr`, `cnr`, null, unknown) → `.sr.md`. The rule tests the en-set, not `sr`,
because `cnr` (Montenegrin) is a distinct live language code on both platforms and
a naive `=== 'sr'` check would wrongly route Montenegrin readers to English. Serbian
covers Serbian + Montenegrin readers (Part 9 — Montenegrin aliases to SR) and is the
fallback for any unexpected or absent code.

**Web** (`oglasino-web`): both server pages derive the bare language via
`getTenantLocale(await getRoutingLocale())?.oglasinoLocale ?? 'sr'` and pass the URL
from a new `src/lib/utils/legalDocUrl.ts` helper to MarkdownViewer. The
`TenantLocale | undefined` return is narrowed with `?.` + `'sr'` default (required
for tsc). The hardcoded-English "pending lawyer review" comment block was deleted
from both pages. Per-URL Data Cache keying (force-cache + revalidate:3600) gives each
language its own entry — no cache change.

**Mobile** (`oglasino-expo`, branch `dev`): both screens read
`useBootStore((s) => s.language)?.code` and pass the URL from a new
`src/lib/utils/legalDocUrl.ts` helper to MarkdownViewer. MarkdownViewer's fetch is
keyed on [url], so a language change re-fetches automatically — no cache change.

**One helper per repo** (not shared — the two repos share no code). Each owns the
doc→stem map and the language→file-token rule in one place so the privacy and terms
callers cannot drift.

**No spec, no trust boundary, no cache work.** Public static content, language is
display-only. Resolves the issues.md 2026-05-27 "Per-locale legal markdown content"
item and the SEO-foundation deferred "per-locale legal content" item.

**Rejected:** a canonical feature spec (too small for the lifecycle); a 404/file-missing
fallback (the .sr.md files exist; the only fallback is the language ternary); a
configurable org/repo/ref (one value, no foreseeable second — Part 4a); a narrower
`lang` type (the helper is deliberately permissive so unknown codes fall to the sr
default).

---

## 2026-06-09 — iOS image-picker permission strings + deny-path toast

Closes two HIGH iOS crash blockers and wires the previously-silent permission-deny path.
The plugin add, the two backend seeds, the toast wiring, and the audit findings below are
**factual** (three archived session summaries — `oglasino-expo-permissions-2`,
`oglasino-backend-media-permission-denied-1`, `oglasino-backend-open-settings-label-1` — plus
the read-only `audit-permissions.md`). The reviewer-facing rationale for choosing English is
Igor's stated reason.

- **The two HIGH crash blockers (factual — found by the read-only 2026-06-09 permissions
  audit):** `expo-image-picker` was installed and used, but **no** `NSCameraUsageDescription`
  / `NSPhotoLibraryUsageDescription` was declared or plugin-injected — the first take-photo /
  choose-from-gallery tap SIGABRTs on iOS. Reached by user action (picker sheet / avatar
  upload), so it crashes on first tap, not at launch.
- **Fix (factual):** added the `expo-image-picker` **config plugin** to `app.config.ts`'s
  `plugins` array (not hand-written `ios.infoPlist` keys) with English single-string props
  (`cameraPermission` / `photosPermission`). The plugin owns both iOS strings at prebuild and
  declares the picker's Android permissions via manifest merge.
- **Deny-path UX (factual):** the camera/gallery deny branches were silent dead-ends; now a
  shared `useMediaPermissionDeniedToast()` hook shows a danger toast (`permission.media.denied`)
  - an Open Settings action (`open.settings.label`, `Linking.openSettings()`) across the three
    picker surfaces. `MessageInput`'s PHPicker call is left ungated (needs no permission).
- **English (not Serbian) chosen for the iOS strings — Igor's call (his stated rationale):**
  optimizes for App Store reviewers.
- **Rejected:** hand-writing `NS*` keys into `ios.infoPlist` (the plugin is the Expo-idiomatic
  owner and co-locates the strings with the module); relying on the plugin's generic English
  default strings (chose listing-specific copy); per-locale `InfoPlist.strings` (English
  single-string now, per-locale deferred — iOS does not pull these from i18n); per-surface deny
  keys (one shared key — same Settings destination either way).
- **Materialization (factual):** strings/permissions land only at **prebuild** (not run this
  session); on-device verification owed before the pre-build blocker clears (see
  [state.md](state.md) header + Risk Watch).

---

## 2026-06-09 — Versioning for first production build; OTA does not yet exist (force-update gate is a store-redirect)

Two coupled decisions taken while preparing the first production build: the cross-repo
versioning scheme, and the finding that there is no OTA pipeline today.

**Versioning scheme.**

- `backend`, `web`, `expo` use **SemVer**; first production release is **`1.0.0`**. Each
  repo versions **independently** — no lockstep across repos.
- `router`, `firestore-rules`, `image-router` use a **bare incrementing integer marker**
  (`v1`, `v2`, …), **not** SemVer. Stored in `package.json` `"version"` (npm accepted a
  bare `"1"` with no SemVer warning in all three) plus a root `CHANGELOG.md` carrying the
  human-facing `vN` entry. First production marker is **`v1`**.
- Applied this session (engineer work, recorded here): backend `pom.xml`
  `0.0.1-SNAPSHOT` → `1.0.0` (single artifact confirmed; 969 tests green); web already
  `1.0.0` since first commit (no change needed); `router` / `firestore-rules` /
  `image-router` `0.1.0` → `"1"` + `CHANGELOG.md` (tests green 54 / 70 / 62); expo
  marketing version already `1.0.0` (single shared string across all three `APP_ENV`
  tiers).
- **Rejected:** lockstep versioning across repos (each repo versions on its own cadence
  instead); SemVer for the three infra repos (a bare integer is simpler and matches their
  deploy model); rewriting web `package.json` to an identical `1.0.0` (no-op churn).

**OTA does not exist yet; the force-update gate is NOT OTA.**

- Finding (verified in code, expo Phase A): `expo-updates` is **not installed** (absent
  from `package.json` and `node_modules`). No `updates.url`, no `updates.enabled`, no
  `runtimeVersion`, no channel on any `eas.json` profile. There is no OTA pipeline.
- The existing `/internal/app/version/ceiling` + `minSupportedVersion` force-update gate
  (`HardUpdateScreen` / `SoftUpdateModal`) is a **store-redirect** mechanism, not
  over-the-air delivery. It pushes users to the store; it cannot deliver a fix without
  store review. Verified: the gate compares the **marketing** version
  (`Application.nativeApplicationVersion` = `expo.version`), not the build number.
- **Decision (Igor, 2026-06-09):** OTA will be wired **before** the first production
  build. `runtimeVersion: { policy: "appVersion" }` is deferred **into** that OTA track
  and applied there as one unit — **not** applied now in isolation.
- **Rejected:** applying `runtimeVersion: { policy: "appVersion" }` now (inert without an
  OTA system, and would imply a capability the app lacks — Part 4a simplicity);
  `runtimeVersion` policy `nativeVersion` (incompatible with build-number
  `autoIncrement`); policy `fingerprint` (experimental, churn-prone).

See [conventions.md](meta/conventions.md) Part 9 (Versioning) for the durable rule, and
[state.md](state.md) (Versions section + the OTA / store-redirect Risk Watch rows).

## 2026-06-09 — Correction: mobile GA4 init is consent-only, not ATT-gated

The 2026-06-02 "GA4 mobile v1 shipped" entry describes the mobile analytics init
as "consent+ATT-gated." That phrasing is stale. App Tracking Transparency was
removed (issues.md 2026-06-02, iOS ATT-crash fix): Oglasino does not use ATT, does
not collect an advertising identifier (IDFA), and does not track across apps. The
mobile analytics init gate is consent-only — `isAnalyticsConsentGranted()`, with the
`attAllowed` operand removed. This matches the privacy-policy draft (§2.16, §7), which
states the app does not use App Tracking Transparency. No code change — the code already
removed the ATT call; this corrects the decisions.md description to match.

## 2026-06-06 — Timestamp Zone — UTC: container TZ flipped to `Etc/UTC`

**Feature:** [features/timestamp-zone-utc.md](features/timestamp-zone-utc.md). Shipped
(code) on backend `dev` 2026-06-06; `verifying` pending Igor's stage boot spot-check.

**The problem.** `BaseEntity.createdAt`/`updatedAt` are `@CreationTimestamp`/
`@UpdateTimestamp` `LocalDateTime`, stored in `timestamp without time zone` columns. The
container ran `TZ=Europe/Belgrade` (`Dockerfile`), so Hibernate wrote Belgrade wall-clock,
and one reader (`DocumentProductConverter:76`, `toInstant(ZoneOffset.UTC)`) re-labelled that
value as UTC — producing an Elasticsearch `Instant` skewed by the Belgrade offset
(+1h winter / +2h summer), with a paired display inverse at `ProductDetailsConverter:78`.

**The fix — Option A.** A single `Dockerfile` ENV flip, `TZ=Europe/Belgrade` → `Etc/UTC`.
No `hibernate.jdbc.time_zone` was pinned anywhere (audit-confirmed), so the flip alone makes
`@CreationTimestamp`/`@UpdateTimestamp` write UTC wall-clock; the two UTC-reading sites become
correct as-written. **No reader code changed.** Pre-production (schema rebuilds from `V1` on
every reset, no stored rows), so the re-base is migration-free. Backend flip + own-repo
doc-sync landed 2026-06-06, `spotless` clean, 969 tests green.

**Corrected blast radius.** Exactly **one** genuine skew site — `DocumentProductConverter:76`
(ES `Instant`) plus its display inverse `ProductDetailsConverter:78`. `EmailDateFormatter` and
`ProductIndexer` operate on `Instant`s, not `BaseEntity` `LocalDateTime`s, and were always
flip-immune — the [issues.md](issues.md) 2026-06-06 entry's "four call sites" framing was
wrong (the "skewed email dates" symptom never existed). `AppVersionAdminDTO.updatedAt`
converts via `ZoneId.systemDefault()` (`DefaultAppVersionService:99`) and is correct under any
container zone — it was **not** double-corrected; left alone.

**Accepted cron-timing shift.** No `@Scheduled` cron sets a `zone=`, so each wall-clock
expression is now interpreted as UTC. Operator-accepted (no zone pinning). The deletion
reminder moves to 13:00 UTC = 15:00 Belgrade summer / 14:00 winter; the other eight crons
shift likewise. No cron's correctness depends on zone — each recomputes its cutoffs from
`Instant.now()`.

**Alternatives considered.** **Option B — fix the four readers to `systemDefault()`** instead
of flipping the container — rejected. Clean-hands pre-prod made the TZ flip the durable
root-cause fix; B would bake Belgrade into the data's semantics and re-arm the `ZoneOffset.UTC`
trap for any future reader, leaving the underlying mismatch in place.

## 2026-06-06 — notifications-toggle-removal shipped (close-out)

The toggle + `allowNotifications` flag removal (decision earlier this date) is
code-complete across backend, web, and mobile — all staged, all DoD-green, all
grep-zero. The flag no longer exists in any repo. Final shape matches the spec
exactly: no dispatch gate added (delivery was and remains token-only), web/mobile
push registration on the auth/boot path untouched, OS/browser permission is the
off-switch, logout detaches the device token.

**Two audit-completeness gaps surfaced during execution** (both caught by the
grep-zero DoD, neither shipped a defect): (1) the backend Phase-2 audit enumerated
the `testUsers.json` _consumer_ (`ImportUserData` + `TestUsersImportService`) but
not the JSON _source file_ — the backend engineer folded in the 3-entry removal;
(2) the web audit did not surface that the COOKIES seed keys
`notifications.label`/`.description`/`.warning` would be orphaned by the removal.
Lesson for future audits: enumerate fixture/seed _source files_, not just their
consumers. Logged for the audit-discipline track.

**Now-dead seed keys deleted directly.** The three COOKIES seed keys
(`notifications.label`/`.description`/`.warning`) were dead across both clients
(web + mobile grep-zero). Because the Ω teardown pass had already run before this
feature, they were deleted directly from the backend seed rather than parked for Ω:
12 rows (3 keys × 4 locales EN/RS/CNR/RU) removed from
`src/main/resources/data/translations/0001-data-web-translations-*.sql`, 969 tests
green against the fresh seed, zero code references. The live BUTTONS key
`button.notifications.label` (mobile BottomBar) was left untouched — different
namespace. Disposable-ID gaps left in the COOKIES sequence per convention, no renumber.

## 2026-06-06 — Admin App Version Control: backend + web feature for the per-platform update floor

**Feature:** an admin-panel view to set the mobile app-update **floor**
(`minSupportedVersion`) per platform — the value the existing version gate uses to
force a hard-update. Styled like `/admin/cache`. See
[features/admin-app-version-control.md](features/admin-app-version-control.md).

1. **Backend + web, not web-only.** The floor had no admin-reachable write path: it was
   writable only via the M2M `POST /internal/app/version/floor` (guarded by
   `X-INTERNAL-TOKEN`), which the Firebase-Bearer web admin client cannot reach. A new
   admin-gated endpoint was required, so this is a backend feature as much as a web one.
2. **New admin surface.** `GET /api/secure/admin/app/version` (both platforms:
   `platform`, `latestVersion`, `minSupportedVersion`, `updatedAt`) +
   `POST /api/secure/admin/app/version/floor`, on a controller with class-level
   `@PreAuthorize("hasRole('ADMIN')")`, mirroring `CacheAdminController` /
   `MaintenanceAdminController`. Admin identity is derived server-side from
   `SecurityContextHolder` (Part 11) — never a client-supplied role/flag.
3. **Floor > ceiling rejected server-side** (coded `422 APP_VERSION_FLOOR_ABOVE_CEILING`)
   — a floor above the ceiling would force users to a version that does not exist yet
   (total lockout). Plus strict 3-part-semver validation (coded
   `422 APP_VERSION_FLOOR_INVALID_SEMVER`). Both live in a new `AppVersionService`; the
   existing `/internal` write paths were left **byte-identical** (the EAS ceiling hook
   depends on them).
4. **No actor / "who-set-it" column.** Single-operator system; `updatedAt` provides the
   "when" provenance. "In case we need it" rejected (Part 4a). Revisit if a second admin
   appears.
5. **EAS-integration rejected for v1.** The view drives off the two live backend values;
   `latestVersion` already flows in via the EAS post-build hook; no version-history table
   exists so a picker isn't backable. No live EAS call. Revisit on real demand.
6. **Web view mirrors `/admin/cache`.** A "Require latest version" button (fills the
   floor field with the current `latestVersion` — the safe 99% path, can't typo a
   lockout) + a manual semver escape-hatch field + Save behind a consequence-naming
   confirm dialog. The floor input is button-or-type, not a picker (no history to pick
   from).
7. **Dedicated `AppVersionValidationException` + handler** (mirroring the Report domain)
   rather than reusing the generic `ProductValidationException` — the reuse was
   semantically wrong (an app-version write throwing a product exception). Mastermind
   override of the implementer's initial reuse.
8. **`updatedAt` wire format pinned.** `AppVersionAdminDTO.updatedAt` changed
   `LocalDateTime` → `Instant`, emitting an explicit-zone (trailing `Z`) value via
   `ZoneId.systemDefault()` conversion, plus a serialization test — so the web
   provenance display can't be silently broken by a global Jackson change and the web
   client needn't assume a zone. This fix is what surfaced the separate, feature-
   independent timestamp-zone bug now logged in [issues.md](issues.md) 2026-06-06
   (`BaseEntity` LocalDateTime timestamps interpreted as UTC). Any system-wide timestamp
   fix must not double-correct this DTO — it is already right.

**Mobile:** untouched. `oglasino-expo` already reads the version gate and obeys the
backend-computed booleans; no adoption expected.

**Rejected:** web-only scoping (the floor had no admin-reachable write path — #1);
reusing the internal floor endpoint from web (unreachable by the Firebase-Bearer client,
and it does zero validation — #2/#3); an actor/audit column "in case we need it" (#4);
EAS live-integration / a historical version picker for v1 (not backable, not needed —
#5); reusing `ProductValidationException` for app-version writes (semantically wrong —
#7); leaving `updatedAt` as a zoneless `LocalDateTime` (forces the web client to assume
a zone — #8).

## 2026-06-06 — "Allow notifications" toggle removed; OS/browser permission is the off-switch

The user-settings "Allow notifications" toggle is removed from both clients and the
account-wide `allowNotifications` flag is removed end-to-end (entity, V1 column,
`UpdateUserDTO`, `AuthUserDTO`, `ImportUserData`, the four converters/readers, the
three admin seeds). Legal adviser confirmed (2026-06-06) the toggle is not legally
required. Spec at [features/notifications-toggle-removal.md](features/notifications-toggle-removal.md).

**Why remove rather than fix.** The flag gated nothing. Three read-only audits
(2026-06-06) confirmed: backend `fanOutPush` (`DefaultNotificationsService`) sends to
every token that exists and never reads the flag; web reads it only to render the
toggle; mobile never reads it (its toggle was already dead per the 2026-05-30
consent-mode-mobile decision). The toggle also drove two divergent half-built models
at once — an account-wide column AND a per-device token attach/detach — so flipping it
on one of two devices left the account-wide flag `false` while the other device kept
receiving push. Fixing required picking account-wide-vs-per-device and reconciling
three repos; removing was less work and deletes the drift permanently.

**The off-switch model after removal.** OS (mobile) / browser (web) notification
permission is the user's opt-out; logout detaches the device token. There is no
server-stored preference flag and no durable in-app "logged-in-and-silent" state on
web (web auto-registers at boot when permission is `granted`, so a stale in-app
"off" would re-register on next login — permission is the single source of truth).

**Registration unchanged — this is a deletion, not a relocation.** The audit's
load-bearing question was whether the toggle was the only path attaching a web push
token. It is not: web attaches on the auth/boot path
(`UseTokenRefresh.tsx:103` → `initPushForAuthenticatedUser`, on every
sign-in/token rotation), mirroring mobile's `PushNotificationsInit`. The toggle was a
redundant second trigger. Permission-prompt timing is also unchanged — the prompt
fires on the boot path for a `default`-permission user, never driven by the toggle.

**No ordering gate.** Both client DTO fields are optional (`?:`), so the field-absent
payload and the client read-removals are order-independent. Recommended order
backend → web → mobile → docs is for cleanliness, not correctness.

**Rejected:** fixing the flag as an account-wide master switch (gate dispatch on it) —
more work across three repos for a control legal says isn't required; per-device
subscription model (drop the column, keep per-device tokens) — closer to the shipped
dispatch but leaves a per-device toggle most users misread as account-wide; UI-only
removal leaving the column and detach plumbing in place — leaves a dead column and an
orphaned detach path for the next reader to puzzle over.

## 2026-06-04 — Deep Linking (Universal/App Links): product & architecture decisions

**Feature:** shareable `https://oglasino.com/...` links open the mobile app (iOS Universal Links / Android App Links). Custom-scheme `oglasino://` links already worked; this adds the verified-https kind. Spans `oglasino-router`, `oglasino-expo` (backend N/A, web deferred). See [features/deep-linking.md](features/deep-linking.md).

1. **Openable paths: public only.** Product, user, catalog + static pages (about/pricing/privacy/terms/blog/free-zone). Secured routes (`/messages`, `/owner/*`) deferred — they'd need post-login return-path work not in v1.
2. **Apex-only associated domains.** `oglasino.com` (prod) / `stage.oglasino.com` (preview); no `www`. The router 301s www→apex and verifiers don't follow redirects, so associated domains must point at the host serving the files directly.
3. **Two-tier rollout (preview + production).** Every artifact (AASA, assetlinks, native config) tier-parameterized: production → `oglasino.com` + `44PHQVN8PB.com.oglasino`; preview → `stage.oglasino.com` + `44PHQVN8PB.com.oglasino.preview`. Apple Team ID `44PHQVN8PB` is constant across tiers (identifies the Apple account; public, not a secret); only the bundle ID differs. Preview is verifiable ahead of production (preview Android keystore has no Play Console dependency).
4. **`.well-known` association files served directly by the router worker, not the web origin.** `/.well-known/apple-app-site-association` and `/.well-known/assetlinks.json` are served inline by `oglasino-router` (`src/index.ts`), short-circuited before the maintenance gate, origin forward, and KV reads, tier-correct per env. **Rationale:** origin-forwarding to Vercel let a maintenance window 503 them (can de-verify the domain association on OS re-verification), passed origin-emitted redirects through (verifiers don't follow them), and left Next's dotfile/extensionless serving uncertain. **Shifts ownership of these two paths web→router** and supersedes any prior assumption (incl. `seo-foundation.md` §11) that web serves them.
5. **The compound locale segment is NOT discarded — it drives a strict-validated base-site switch with link-language filter resolution. (Supersedes the earlier "strip locale and discard" position.)** The locale in `https://oglasino.com/{baseSite}-{language}/...` is base-site + language, both load-bearing:
   - `+native-intent` strips the segment for routing (the app route tree is locale-less) but parses and stashes `(baseSite, language)` via a read-and-clear side-channel; the locale travels beside the path, never in it.
   - Boot Gate 3 validates **strictly**: base-site must exist AND language must be in that base-site's `allowedLanguages`. **If either is invalid the whole link is untrusted → normal boot on the stored site** (no switch, no filters, no crash). A bad locale poisons trust in which language the filter words are in (e.g. `rs-cnr` is a near-miss between Serbian and Montenegrin with different filter labels), so it is rejected rather than guessed.
   - If valid and the base-site differs, the app **switches base-site** (the data context — catalog/regions/cities/currency — is one `BaseSiteDTO`). Filters are resolved against the **link's language** (a fixed link-language translator over a transient in-memory label load), because web emits filter slugs in the link's language and they only match if resolved in it.
   - **The user's preferred display language is never changed or persisted-over.** Resolved filters are language-independent objects, so they survive with the display in the preferred language. A loading curtain holds (`status` stays `'booting'`) across the switch so there is no flash; the portal mounts once — switched, in the preferred language, filters applied, single feed fetch.
6. **No display-language auto-switch.** Consequence of #5: a shared `rs-en` link does not change what language the user reads the app in; the link language is used only to resolve filters, then dropped.
7. **Filter deep-linking (home + catalog).** A shared URL's filter query string populates and applies filters on open. **Strict on the locale** (it establishes trust), **lenient on individual filters** — within a valid link, filter values that don't resolve are skipped, not errors; apply what resolves. Mobile mirrors web's SSR `parseFiltersFromQueryParams` (slug-vs-slug, accent-robust), not web's lossy client-store hydrate.

**iOS:** fully shippable now (Team ID + bundle known). **Android:** verification deferred per-tier on the signing-cert SHA-256 (see [issues.md](issues.md) 2026-06-04). **Verification:** on-device only (Ψ).

**Rejected:** serving the well-known files from the web origin (the three failure modes in #4); discarding the locale after stripping it (loses the base-site + filter-language signal — superseded); auto-switching the display language to the link's language (changes what the user reads for a shared link — rejected for #6); treating an invalid locale leniently / guessing the base-site or language (poisons filter-word trust — strict-reject instead).

---

## 2026-06-04 — Review listings tolerate a missing translation row at read time (jpa-fetch-tuning Batch 4)

Review listings (public, owner, admin) now render gracefully when a review has no translation row for the reader's current language — falling back to the review's **original** (human-written, non-auto-translated) row, then any available row, then empty fields — instead of throwing. The read path fetches **all** language rows for the page's reviews in **one** batched query and resolves in memory, so the single-query-per-page property is preserved (no per-row query reintroduced).

The write-side cause was **deliberately not fixed** in this feature: review creation generates the non-author-language rows inside an OpenAI call whose exceptions are swallowed, so an OpenAI outage at creation time persists a review with only the author's-language row.

**Rationale:** the previous throw was a reachable production 500 — a single translation-less review 500'd the entire listing page. Making the read path resilient is the safe, contained fix and ships with jpa-fetch-tuning. Fixing the write-side (OpenAI retry / backfill) is separate translation-pipeline reliability work, out of scope for fetch tuning, and no longer launch-blocking now that the read path cannot crash. Recorded so a future reader understands why the read path tolerates missing translations and where the underlying write-side gap still lives.

**Rejected:** keeping the throw (a reachable 500 on a user-facing endpoint); a base-site-default-language fallback (not plumbed to the converter and can itself be absent — strictly worse than the original-row fallback); a second batched query for default-language rows (the single all-languages `IN` query is cheap and page-bounded); fixing the write-side within this feature (out of scope — separate translation-pipeline reliability work).

---

## 2026-06-04 — Backend authorization is default-deny (H1)

`SecurityConfig.authorizeHttpRequests` now terminates in `anyRequest().authenticated()` (was `permitAll()`). The public surface is explicit and small: `OPTIONS /**`, `/actuator/health/**` (unauthenticated — the docker healthcheck and the edge mobile-liveness probe carry no token), `/error`, `/health`, `/api/public/**` + `/api/auth/**` + `/internal/**`; `/actuator/prometheus` + `/actuator/info` are `denyAll`. Adding a public endpoint now requires an explicit matcher — the safe default is "denied," not "open."

**Note:** with `formLogin`/`httpBasic` disabled and no custom `AuthenticationEntryPoint`, all unauthenticated denials return **403** (`Http403ForbiddenEntryPoint`), not 401. This corrects the original 2026-06-03 audit's "401" expectation; the security invariant (denied-is-denied) is unchanged.

**Rejected:** blanket-permitting `/actuator/**` (would re-expose prometheus/info); leaving `anyRequest().permitAll()` with per-path opt-out (the H1 finding itself — allow-by-default is the wrong posture for a server).

---

## 2026-06-04 — Admin is provisioned out-of-band, not by code or email (H2)

Admin is now a per-environment seeded DB row (`data/admin/data-admin-{dev,stage,prod}.sql`) keyed on that env's Firebase UID, idempotent via `ON CONFLICT (firebase_uid) DO NOTHING`; registration is unconditionally `ROLE_BASIC`. The hardcoded `admin@oglasino.com` email-literal grant and the JSON test-data admin seed were both removed.

Per-env SQL selection is achieved by each env's own yaml listing an explicit admin file — there is **no** `data-${profile}.sql` convention; all envs otherwise load identical globs. Future per-env seeds follow this explicit-per-yaml pattern.

**Rejected:** keeping the email-literal grant (re-arming trap on a fresh-env seed or delete-and-recreate); a `data-${profile}.sql` auto-selection convention (doesn't exist today; explicit per-yaml listing is clearer and matches the as-built mechanism).

---

## 2026-06-04 — XSS defense is render-time output-encoding, not backend input-encoding (M1)

The `InputSanitizationFilter`/`Sanitizer.sanitize` HTML-entity-encoding at the input layer was removed. It was the wrong layer (it corrupted non-HTML consumers — mobile/JSON/ES — and double-encoded on re-render) and half-broken (it never overrode `getInputStream`, so the dominant JSON `@RequestBody` surface bypassed it anyway). XSS encoding now lives at the actual render points: web clients (React/RN escape by default; the JsonLd `<script>`-context escape landed web-side, Brief 3) and backend email bodies (`HtmlUtils.htmlEscape` at the interpolation point). The OWASP encoder dependency was dropped. `Sanitizer.stripNewlines` was retained as a live log-injection guard.

**Hard sequencing (M1 gate):** the web JsonLd escape must reach an environment before or with the backend filter removal, or stored XSS reopens on public pages for the gap. Mobile needs no action (no DOM).

**Rejected:** keeping input-layer HTML-encoding (wrong layer, already bypassed); replacing it with input-layer semantic hygiene only (trim/normalize) as a security control (output-encoding at the render point is the correct boundary).

---

## 2026-06-04 — Identity authority decoupled from subscription state (M3)

`AuthUtils.subscriptionToAuthorities` now **always** emits the identity role (`ROLE_ADMIN`/`ROLE_BASIC`) and gates only the subscription-tier authority (`ROLE_<TIER>`) on an active subscription. Previously it returned zero authorities — including the identity role — when the subscription was inactive or null, so a lapsed-subscription admin lost `ROLE_ADMIN` and was locked out of every `@PreAuthorize` admin endpoint. Source values are server-derived (Part 11 clean); the bug was the coupling. The fix removes the dependency on the admin row's subscription columns entirely (the Brief 5 read-only check of that row is therefore no longer load-bearing).

**Rejected:** leaving identity coupled to subscription (a real lockout risk for any lapsed/null-subscription admin; new users only worked by accident via their active free subscription).

---

## 2026-06-03 — Password reset: web-owned flow, email-channel provider help, no-leak request endpoint

**[DECISION]**

- **Web owns the entire reset flow; both platforms funnel to web.** `/forgot-password` (request entry) and `/reset` (oobCode confirm). Password change is pure client-side Firebase (`confirmPasswordReset`); the backend never sees the password and there is **no schema change**.
- **Branded email via Admin SDK `generatePasswordResetLink` → Brevo** (oobCode extract-and-rewrite to `<webBase>/<locale>/reset`), reusing the email-notifications building blocks. The request endpoint `POST /api/auth/request-password-reset` is a sibling of `/auth/resend-verification`: unauthenticated, email-only, 60s + 4/day per-email Redis throttle on **separate keys**, no account-existence leak, coded responses.
- **Provider gate:** send the reset link **iff the account's linked providers contain `password`**. The Firebase project is confirmed set to **"Link accounts that use the same email"** (one user per email; providers linkable), so the rule is _contains-password_, not _is-social_. Provider detected via Admin SDK `getUserByEmail().getProviderData()` — **not** the garbage `registeredWithProvider` column.
- **"Option 4" (login-screen "this account uses Google" hint) is DROPPED.** Rationale: Firebase email-enumeration protection (on) makes a social-account email+password failure return the generic `auth/invalid-credential`, indistinguishable from a wrong password client-side; `fetchSignInMethodsForEmail` is neutered. The only client-side detections were (a) a backend login pre-check (a new enumeration oracle + latency) or (b) disabling enumeration protection (reopens the existence leak) — both rejected. All provider help moves to the **email channel**: a social-only account that requests a reset receives a branded "use your social sign-in" email instead of a reset link.
- **No leak, no lie:** the endpoint returns an identical generic response and consumes limits identically across all four states (banned / unknown / social-only / has-password). Every **real** account receives an email (reset link or "use social"); only a **non-existent** email receives nothing. On-screen copy is conditional: "If an account exists for this email, you'll receive an email shortly." **Banned** emails receive no reset link and no email (a banned user must not reset back in; ban comms are owned by the reblock flow).
- **Success-step app-deep-link is SCAFFOLDED but DEFERRED** to a later session (inbound browser→app handling does not exist in the app today). The web success page wires a dormant, TODO'd "Open in app" affordance with a single-constant scheme target so the later session drops in; the live behavior is the web "log in here" fallback.

---

## 2026-06-03 — DB Overload Protection spec authored (graduated throttle + auto-maintenance-trip)

Backend-only graduated DB-pressure response: **YELLOW** sleeps each request, **RED** sheds with
HTTP 503 + `Retry-After`, **CRITICAL** auto-flips the Cloudflare maintenance gate. Operator-alerted
(email on RED, email + Telegram on CRITICAL) and forensic-logged to a new `incident_log` table.
Spec at [features/db-overload-protection.md](features/db-overload-protection.md). Status `planned`;
Phase 5 (engineering) not started. Authored at Phase-4 close behind a completed Phase-2 backend
audit.

The eight Phase-3 seam resolutions (the load-bearing decisions; substance from the spec's "Spec
decisions" section):

1. **Thresholds as fractions of the live pool size, not absolute counts.** The audit found
   `maximum-pool-size` differs per env (dev 20 / stage 8 / prod 18). Absolute 13/14/17 was
   prod-only and made CRITICAL unreachable on stage's pool of 8. The monitor reads
   `getMaximumPoolSize()` live and resolves `ceil(ratio × size)` per poll. _Rejected:_ absolute
   counts as Configuration rows — every pool-size change becomes two coordinated edits, and a
   missed one silently mis-calibrates protection.

2. **In-process `HikariDataSource` bean read for the pressure signal; no Prometheus.** There is no
   `micrometer-registry-prometheus` on the classpath and the throttle runs in the same JVM as the
   pool, so it reads the pool MXBean directly. _Rejected:_ a Prometheus HTTP scrape of its own
   process — strictly worse than a direct bean read. Prometheus/Grafana operator-observability is
   queued as a separate future chat.

3. **Active-count gradient + `threadsAwaiting > 0` as the CRITICAL discriminator.** Active-count
   saturates at pool max and cannot distinguish "busy" from "starving"; threads-awaiting is the
   leading starvation signal (and `leak-detection-threshold` is disabled everywhere). CRITICAL gates
   on count AND `threadsAwaiting > 0`. _Rejected:_ threads-awaiting as the sole signal — no gradient
   below saturation.

4. **Two-layer enable: per-env YAML master switch + live Configuration kill-switch.** The audit
   found **no per-env Configuration seed exists** (all envs load the identical SQL), so a
   Configuration row alone cannot express "off on dev / on in stage+prod." YAML carries the per-env
   stance (`${ENV_VAR:default}`, the `app.images.sweeper.enabled` precedent); the Configuration
   `throttle.runtime.enabled` row is the live override; both must permit. _Rejected:_
   Configuration-only flags — same value everywhere, can't vary per env.

5. **Poll interval in YAML, not Configuration.** `@Scheduled` resolves its interval from the Spring
   Environment at bean construction and cannot read the runtime Configuration cache (user-deletion
   crons precedent, decisions.md 2026-05-19). Thresholds and windows stay in Configuration
   (read per-poll, not at construction).

6. **`SERVICE_DEGRADED` in `SystemErrorCode` (post-`system-error-code-split`); 503 body carries
   `translationKey`.** The split shipped, so `SystemErrorCode` is the home, not the old
   `ProductErrorCode` dumping ground. The 503 body mirrors the as-built `RateLimitFilter` 429 body
   (which already carries `translationKey`). Noted as a filter-layer extension of Part 7's
   `{field, code}` envelope, **not** a Part 7 amendment.

7. **Filter placement via the Security chain.** No filter declares `@Order`; only the Security chain
   trio is deterministic. Registered via `SecurityConfig.addFilterAfter(requestThrottleFilter,
RateLimitFilter.class)`, mirroring how `RateLimitFilter` itself is placed — sidesteps the
   undefined `@Component` ordering bucket rather than depending on it.

8. **Auto-trip asserts maintenance-on (no-op if already on), writes BOTH web+backend flags, on a
   timeout-bounded KV client, off the poll thread.** `toggleMaintenance()` is a flip — used raw it
   would turn maintenance _off_ if it were already on when CRITICAL fired. The trip asserts on
   instead. Writes both `maintenance.web.active` and `maintenance.backend.active` (operator decision
   2026-06-03: backend overloaded ⇒ web unusable anyway). The KV client must be timeout-bounded — the
   trip calls Cloudflare precisely when degraded, and the existing no-timeout `RestTemplate` could
   hang the monitor thread on the very API the trip depends on; the write also runs off the poll
   thread so it can't stall polling. _Rejected:_ raw `toggleMaintenance()`.

**Factual vs inferred.** The audit findings (per-env pool sizes, no Prometheus on classpath, no
per-env Configuration seed, the no-timeout KV `RestTemplate`, `@Scheduled` resolving at
construction) and the operator decisions (write both flags, Prometheus out, threads-awaiting in,
timeout-bounded KV in scope) are **factual** — the Phase-2 audit deliverable plus Igor's
confirmations in the Mastermind chat. The threshold ratio defaults (`0.72` / `0.80` / `0.94`) are
**inferred** starting guesses chosen to reproduce the original 13/14/17 intent against prod's pool of
18; they are Configuration-tunable and flagged as such in the spec (§ 3.2, § 12).

**Implementation note (2026-06-03, Phase-5 build).** Two implementation facts worth preserving for a
future audit:

- **`SlowQueryService` reads `pg_stat_statements` via `EntityManager.createNativeQuery`** — the only
  such call in backend `src/main`. The house native-read idiom is Spring Data
  `@Query(nativeQuery = true)`, which requires a domain `@Entity`; a system view is not an entity, so
  the `EntityManager` native query is the idiomatic JPA read here. Justified, not a silent divergence.
- **`IncidentLog` is a standalone `@Entity`** (BIGSERIAL / IDENTITY id, semantic `occurred_at`), **not**
  extending the shared `BaseEntity` — a deliberate divergence for a forensic / append-only table,
  approved in Session 1.

---

## 2026-06-02 — Message notifications are push-only with OS-collapse anti-spam (no in-app doc, no server unread-state)

New-message notifications are delivered as **push only** — no in-app Firestore notification
document is written for messages (the user already has the messages page; a bell-list entry would
duplicate it). The push banner shows the actual message text (sender name as title, message body
as body) — an accepted privacy tradeoff, standard messenger behavior.

Anti-spam ("10 messages → one banner, not ten") is delivered by a **per-chat OS collapse key**
(Android `collapseKey` / `tag`, iOS `apns-collapse-id`, Expo `collapseId`; on Firebase Web Push
both the RFC-8030 `Topic` header AND the `Notification` tag), **not** by a server-side
unread-state check. The backend always sends on every message-ping; the OS replaces the prior
banner for that chat. This is best-effort (collapse is not guaranteed on every device/OS), and
there is no in-app fallback surface for messages by design.

- **Why not server unread-state.** The chat root doc carries no server-readable per-recipient
  unread state (only a per-message `seen` field in a subcollection and a client-maintained
  `unreadCount` sidecar that is already incremented by the time the ping fires). A
  server-authoritative "first-unread" gate is therefore not possible without new presence
  infrastructure, which was out of scope. Active-viewing suppression (don't surface a banner for
  a chat the user currently has open) is a client-side concern, deferred (no clean signal existed
  on either client at build time).
- **Trigger.** Client→backend message-ping endpoint
  (`POST /api/secure/notifications/message-sent`). The backend verifies via Admin SDK that the
  authenticated caller is a participant of the chat and derives the recipient from the chat doc's
  `users[]` — never from a client-supplied recipient field. Message text is client-supplied
  **display content only** (the participant-verified sender's own message; not used in any
  authorization / moderation / state decision). No Cloud Functions; fits the existing Spring +
  Admin SDK architecture.
- **Emitter (client → backend), confirmed on device 2026-06-02.** The trigger is a client
  **emitter** that fires the ping post-commit, fire-and-forget: after the chat store commits the
  message to Firestore, the client POSTs `{ chatId, messageText }` only (no recipient, no userId,
  no anti-spam flag — the backend owns all of that). It never awaits into the send path and never
  fires on a failed write (web W4, expo E3). **Image-only (no-text) messages:** the client sends
  `messageText: ''` (empty string, key present — a consistent wire shape across both clients) and
  the backend (B6) supplies a localized photo body (`notif.message.photo.body`, EN "📷 Photo") in
  the recipient's `preferredLanguage`; the client never fabricates a body (expo E5, web W4).
- **Alternatives considered.** A server-side first-unread gate (rejected — needs presence
  infrastructure, out of scope); an in-app message-notification document (rejected — duplicates
  the messages page); client active-viewing suppression (deferred — no clean signal at build
  time).

This supersedes the spec's earlier §2.1 "first-unread-per-conversation" server-unread-state
wording, corrected to the as-built OS-collapse mechanism in the same close-out (see
[features/notifications.md](features/notifications.md) §2.1).

**Process note (recorded once).** The message-ping was built **receiver-first** (backend B4 endpoint)
and the client **emitter** that calls it on message-send was never built — caught only during
on-device verification, when no network call left the device on send (fixed by expo E3 + web W4,
then B6 + E5 for image-only). The `mobile-stable` hold did its job: device verification surfaced
the gap before the feature was marked done. Lesson for future features with a client→backend
trigger: **brief the emitter and the receiver together**, not receiver-first.

---

## 2026-06-02 — Prefer `affectedKeys().hasOnly([...])` over per-field pinning when a Firestore doc schema is cross-repo-unverifiable

For a Firestore security-rule `update` field-lock on a document whose full field set is owned by
another repo (e.g. the backend Admin SDK writes the doc and the rules repo cannot verify the
complete schema), prefer
`request.resource.data.diff(resource.data).affectedKeys().hasOnly(['<allowed>'])` (plus a type
guard such as `... .seen is bool`) over per-field equality pins.

- **Why.** `hasOnly` enforces "only these keys may change" correctly for ANY schema and
  additionally rejects added fields, whereas per-field pinning requires enumerating every other
  field (a guess when the schema is cross-repo) and does not block newly-added fields.
- **Scope / caveat.** The `messages`-update rule retains per-field pinning — its schema is
  repo-local and fixed, so enumeration is safe and the existing idiom is kept for consistency
  there. The two sibling rules therefore use different idioms **by design** (notifications:
  `hasOnly`; messages: per-field), each chosen for its schema's verifiability.
- **Applied in.** The notifications update rule
  (`notifications/{userId}/userNotifications/{id}`) — the recipient may change only `seen`.

---

## 2026-06-02 — Docs-sync is a cleanliness rule: revalidate README/docs on every change

Every repo now carries a root `README.md` (written this session across backend, web, expo,
router, firestore-rules, image-worker, landing, maintenance, and docs; `oglasino-private`
excluded by Igor). To keep them from drifting, a new **Part 4 (Cleanliness)** bullet binds all
agents: when a change makes a `README` or any other doc stale (file/folder layout, commands,
scripts, env vars, endpoints, status, cross-links), the doc is updated in the **same session** —
the doc that describes a change is part of the change. The mirror rule is also in the Docs/QA
`CLAUDE.md` hard rules.

- **Ownership boundary.** Each engineer agent owns its own repo's `README` + `<repo>/docs/`;
  Docs/QA owns `oglasino-docs`. The no-cross-repo-edits rule is unchanged — if a change in one
  repo invalidates a doc in another, the agent flags it per Part 4b rather than reaching across.
- **Why.** Docs/QA does not read code in the sibling repos, so a Docs/QA-only rule could not keep
  engineer-repo READMEs fresh; the teeth have to live in the shared rulebook.
- **Alternatives considered.** Keeping the rule Docs/QA-only (rejected — leaves the new READMEs
  to rot); editing each engineer repo's `CLAUDE.md` (rejected — cross-repo writes are forbidden
  and out of scope; conventions Part 4 is the shared lever).

---

## 2026-06-02 — GA4 mobile v1 shipped — strict mirror of web's catalog, mobile-stable

All 13 events mirroring web's catalog (same names, param keys, PII rules) are now shipped on
`oglasino-expo` (`new-expo-dev`). A single `track`/`trackError` wrapper over Firebase `logEvent`;
consent+ATT-gated init; `user_id` set on auth resolve / cleared on sign-out. Verified on-device via
Firebase DebugView on **both iOS and Android** (2026-06-02): all 13 events land with correct params,
and `page_view` fires with no parallel auto `screen_view`, confirming the auto-screen-reporting
disable took.

- **Decisions that shaped it.** `page_view`, not native `screen_view`, reporting into the **same GA4
  property as web** (`oglasino-stage-49abb`, ID 536855998) via the pre-existing mobile data streams —
  auto-`screen_view` collection disabled in `firebase.json` so views don't double-count (confirmed
  clean in the smoke). `sign_up`/`login` fire from the explicit auth methods, never the
  `onIdTokenChanged` listener (no spurious cold-start `login`); `wasRegister` surfaced via a type-only
  `AuthUserDTO` change, zero backend work. `exception` built RN-native (route `ErrorBoundary` +
  ungated `ErrorUtils` global handler chaining to the prior handler + promise-rejection tracking).
  `form_submit_failed` two shapes: the real Part-7 `code` on structured backend paths (product
  create/update), synthetic codes (`DISPLAY_NAME_EMPTY`, `EMAIL_EMPTY`, `EMAIL_FORMAT`,
  `PASSWORD_EMPTY`, `PASSWORD_INVALID`) on client-validated paths (register/login/profile).
  `filter_change` debounced on a `searchText`-excluded projection of `FilteredProductList.filtersData`.
  `sort_order`/`currency` read the string code off their DTO objects; `price` coerced to number.
- **GA4 console.** Verified done — mobile data streams already existed in the shared property
  (per-tier Firebase apps from the 2026-05-26 cloud-setup), App IDs match the GoogleService build
  files, custom dimensions registered (mobile inherits them via identical param keys). Web's
  `G-P0LEVEJ0V9`/`G-GNKB4WBNC0` are web-stream Measurement IDs, not used by mobile (which routes via
  Firebase App ID).
- **Process note (recorded honestly).** The eight transactional events were specified early but their
  implementation session was initially missed; a premature closure attempt claimed all 13 shipped on
  evidence for 5. Docs/QA caught the discrepancy and refused the closure; session
  `oglasino-expo-google-analytics-v1-4` then implemented and verified the eight. Recorded so the audit
  trail shows the catch and the correction.

This closes the GA4-mobile Mastermind chat; the feature is `mobile-stable` on `new-expo-dev`. See
[features/google-analytics-v1.md](features/google-analytics-v1.md) Platform adoption — Mobile (Expo).

---

## 2026-06-02 — iOS build requires `expo-build-properties.ios.buildReactNativeFromSource: true`

`oglasino-expo` sets `expo-build-properties.ios.buildReactNativeFromSource: true` (alongside
`useFrameworks: 'static'`) to build the iOS app. Required on Expo SDK 54 / RN 0.81 / Xcode 26: the
precompiled `React.xcframework` + static frameworks (needed by Firebase + Google Sign-In) + explicit
modules produce hard `-Werror` non-modular / cross-module failures in the react-native-firebase pods;
building RN from source removes the precompiled-framework boundary and is the only approach that built
green (verified locally on Xcode 26.2 → `Build Succeeded`). Intermediate workarounds — a
`CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES` / `$RNFirebaseAsStaticFramework` config plugin,
and `forceStaticLinking` — were tried and rejected as insufficient and removed from disk.

- **Cost.** Longer iOS build times (no precompiled RN binaries). Tracked in `state.md` Risk Watch;
  revisit on the next Expo SDK / RNFirebase bump — if upstream fixes the prebuilt-core path, the flag
  can be dropped to restore faster builds.
- **Refs.** expo/expo#39233, expo/expo#39607, invertase/react-native-firebase#8657.

---

## 2026-06-01 — Image pipeline closed — mobile `mobile-stable`; on-device smoke deferred

The mobile image-pipeline is closed at `mobile-stable` on `new-expo-dev`. All four upload
surfaces (product, profile, chat, review), `expo-image` display, raw-byte PUT via
`expo-file-system`, the retry policy, orphan cleanup, and the per-stage progress text are
implemented and validated; the iOS+Android rebuild carrying the V6 (HEIC label localization)
and V9 (avatar + chat orphan cleanup) conformance fixes plus the `@react-native-community/netinfo`
native module has landed.

- **Smoke deferred, not skipped.** The 14-case on-device smoke
  ([features/image-pipeline-mobile-test-cases.md](features/image-pipeline-mobile-test-cases.md))
  is deferred — Igor's explicit call. Status was flipped to `mobile-stable` on Igor's confirmation
  rather than on a completed device pass. Any failures surfaced when the smoke is eventually run
  will be reported and resolved as ordinary follow-ups, not as a re-opening of the feature.
- **Backend deferrals accepted, not re-dispositioned.** The backend-side deferred items are
  carried as accepted deferrals, pre-production: no automated `deleteProduct`→image-cleanup e2e
  test; the fire-and-forget JVM-kill cleanup window in `deleteProduct`; unverified `updateProduct`
  cache invalidation; and R2 orphans from a partially-failed batch upload — the last already
  resolved-by-decision via the 2026-05-31 sweeper entry. Considered and accepted at close, not
  re-opened.
- **Translations.** All `image.*` keys and the B15 picker-sheet keys (`image.source.title` /
  `.camera` / `.gallery`) are seeded in all four locales. EN/RS are final; RU/CNR remain
  placeholder pending native-translator review — the standing project-wide review debt, not an
  image-pipeline blocker.
- **Factual vs inferred (close-out convention).** Factual on Igor's 2026-06-01 confirmation:
  code-complete, validated, rebuild landed, smoke deferred, backend deferrals accepted. Not yet
  observed (inferred-pending): the on-device smoke result itself — deferred, so no device pass is
  asserted by this entry.

---

## 2026-06-01 — Owner product-load response widened with display-only category objects (`ProductForUpdateDTO`)

The owner product-load endpoint `GET /api/secure/products?productId=` now returns a dedicated
read-only `ProductForUpdateDTO`, distinct from the update-request DTO. It carries the three category
objects (`topCategory`/`subCategory`/`finalCategory`) as full `CategoryDTO` (with `labelKey` and nested
`filters`), in addition to the mutable edit fields (`id, name, description, price, currency, filters,
imageKeys`).

- **Purpose.** Clients render the category read-only AND derive the category-specific filter set from the
  nested `filters`. A label-only or ID-only response would not fill the filter form — the full object with
  `.filters` is required. The change unblocked three latent symptoms on the consumers with no client code:
  web's previously-blank category fields, the short filter list (was ~4 base-site filters; now the full
  ~9–10 for the category chain), and the latent hidden-price gate — all opened by the contract change alone
  (web verified live by Igor). Mobile reads the same fields it already expected.
- **Trust boundary (Part 11).** Category remains server-immutable on the update WRITE path. The write path
  still accepts the separate, unchanged `UpdateProductRequestDTO`, which has no category fields and is still
  guarded against client-sent category fields. The widened response is read-only; it does not make category
  bindable on update.
- **Implementation note.** Reused the existing `CategoryConverter` (the create/catalog mapping path); no new
  category converter was invented. The old `UpdateProductRequestConverter` was replaced by a new
  `ProductForUpdateConverter` and deleted. Backend on `dev`. See [issues.md](issues.md) 2026-06-01 update-surface
  resolutions and the carried-over `FilterConverter` selected-filters type-mismatch logged there.

---

## 2026-06-01 — `DialogWrapper` now uses the gesture-handler `ScrollView` (shared-component change, `oglasino-expo`)

`DialogWrapper` (used by every dialog in `oglasino-expo`) now uses the `react-native-gesture-handler`
`ScrollView` instead of the React Native one. The app is already wrapped in `GestureHandlerRootView`.

- **Reason.** Dialogs with tall and/or interactive content (nested horizontal carousels, touchable rows) had a
  scroll/gesture-responder dead-zone; the GH `ScrollView` arbitrates gestures through the gesture tree so the
  whole dialog body scrolls.
- **Consequence for future work.** Nested scrollables and touchables inside dialogs now coordinate correctly.
  The earlier `FiltersDialog` workaround (its own inner `ScrollView` + `nestedScrollEnabled`) is no longer the
  only pattern for tall dialog content.
- **Verification.** Verified on device on the preview path (Igor, 2026-06-01) — the only item in the
  create/update product-parity feature confirmed on a device so far. Because this is a shared-component change
  affecting every dialog, the main scrolling dialogs (`FiltersDialog`, `AddUpdateProductDialog`, and any other
  tall dialog) are to be confirmed still scrolling at/before commit; a regression there would be a separate
  follow-up, not part of this feature. Branch `new-expo-dev` (Ψ-pending with the rest of the branch).

---

## 2026-06-01 — Expo catalog filters brought to web parity (code); five findings, two real bugs fixed

The mobile (`oglasino-expo`) catalog/browse + owner-dashboard filter surface was audited
against web (the parity reference) and brought to parity. Code-complete on `new-expo-dev`
across three engineer briefs (`catalog-filters-2`/`-3`/`-4`) behind four read-only audits: web
catalog-filters reference, web region/city wire-shape, expo current-state+gap, and a backend
price/random contract check. On-device Ψ owed (shares the pending iOS+Android rebuild). Web
does not change.

**The two real bugs.**

- **RANGE/DATE filters filtered nothing (the headline).** Mobile's request build
  (`FilteredProductList.filtersData`) mapped only `optionIds` per selected filter and dropped
  the `rangeFrom`/`rangeTo` that `addRemoveRangeFilter` stored on the same
  `SearchSelectedFilter`. Range and date filters rendered, were selectable, but POSTed an empty
  option filter — they filtered nothing, showed no removable chip, and added 0 to the
  active-filter count. This is the substance behind Igor's on-device "filters are missing / not
  right" report. Fixed end-to-end in `catalog-filters-2`: the request now carries
  `rangeFrom`/`rangeTo` (the `RequestSelectedFilterDTO` type already declared both); a removable
  chip is rendered per range/date entry (formatted with the filter's
  `filterRange.rangePrefix`/`rangeSuffix`, cleared via the existing
  `addRemoveRangeFilter(filter, undefined, undefined)` action); and each counts 1 in the badge.

- **Owner dashboard showed catalog-only controls.** The shared `FiltersDialog` rendered the
  dynamic per-category filters, the basic/advanced show-more split, and a `DELETED`
  product-state option on the owner dashboard, where web's dashboard is narrow
  (search/order/price/state-ACTIVE-INACTIVE). Fixed in `catalog-filters-3`: the dynamic-filter
  block + show-more are gated on `currentPortalScope !== 'dashboard'` (reusing the predicate the
  region/city control already uses), and the dashboard product-state options are restricted to
  an explicit `[ACTIVE, INACTIVE]` list. The `ProductState` enum is untouched (DELETED stays
  valid elsewhere); the public catalog surface is unchanged.

**Active-filter count aligned and single-sourced.** Mobile summed total option counts (a
2-option filter counted 2; range/date counted 0); web counts 1 per selected-filter entry. The
formula was duplicated verbatim across `FiltersDialog.tsx` and `FloatingFiltersButton.tsx`.
Both now call one shared `src/lib/utils/getActiveFilterCount.ts` that counts 1 per
`selectedFilters` entry plus the unchanged price/order/region/city terms and the dashboard-only
product-state term. `isDashboard` is a parameter because the dashboard term depends on
`usePortalScope`, which lives outside the filter store — so a store selector could not own the
whole formula. The two surfaces can no longer drift.

**Region/city auto-collapse added (Finding 5).** When a user selected every city of a region
one-by-one, mobile left N city selections; web collapses to the region. `catalog-filters-4` adds
the collapse to the city-add path only — when the last city of a region is selected it drops
that region's cities and pushes the region, reproducing byte-for-byte the end-state the
region-select path already produced (same `setRegionCityValues` action, no new abstraction). A
single-city region collapses on its one city. Region-select and city-unselect behavior are
unchanged. Chips and count need no change — `SelectedFiltersDisplay` renders the `regions` map,
and `getActiveFilterCount` sums `regions.length + cities.length`, so a collapsed region counts 1,
matching web.

**Three Finding-4 items closed with NO code work — including one retracted false alarm.**

- **Price bounds:** `priceRange.from/to` are `BigDecimal`; Jackson's default scalar coercion
  turns the JSON string `"1000"` into `BigDecimal` identically to the number `1000`, so mobile's
  digit-string price bounds filter correctly. The "parse to number" change I would have applied
  as a safe default was dropped. (Backend audit, `audit-catalog-filters-backend-price.md`.)
- **Random suppression:** the backend drops `function_score` random ordering when `searchText`
  is non-blank OR `orderBy` is set (`DefaultProductsFilterQueryBuilder.applyRandomIfNeeded`,
  test-backed). Mobile's suppression (`applyRandom && !searchText && !orderBy`) already matches —
  a correct no-op, not a divergence.
- **Region/city wire shape — RETRACTED false alarm.** A backend read reported the server consumes
  `selectedRegionAndCityValues` with inner `regionIds`/`cityIds` (`List<Long>`), flagging mobile's
  `selectedRegionsAndCities: {regions,cities}` as a silent-drop bug. A targeted web wire audit
  settled it: web sends `selectedRegionsAndCities: { regions: RegionDTO[], cities: CityDTO[] }`
  (full objects, plural outer key) with no transform between the store and the POST body, and
  web's region/city filtering works in production. Mobile already sends the identical shape. The
  live backend consumes the object shape; the backend read was of a stale/sibling DTO the search
  endpoint does not bind for this path. **No region/city wire change was made**; telling mobile
  to "rename to `regionIds`/`cityIds`" would have broken working parity to chase a phantom.

**Settled, no work (for the record):** FilterType enum (four values, no SELECT — confirmed both
sides), definition source + `order`-merge, SINGLE/MULTI dispatch (both collapse to a multi-select
checkbox list; `multiselection` is NOT honored on the catalog surface on either platform — the
only component honoring it is the out-of-scope update-product form), RANGE/DATE rendering, search,
ordering, currency, free-boolean, basic/advanced split (catalog), clear-all, random suppression
rule, and the region/city request shape.

**Out of scope (confirmed, not addressed here):** the update-product form's missing
filters/category, the product-creation dialog's region/city defaulting, and admin filter surfaces
(mobile is consumer-only since chat α). These are separate tracked items.

**Process note — the targeted web wire audit earned its place.** When the backend audit and the
web catalog audit disagreed about the region/city wire shape, a one-question read-only web audit
was run rather than drafting a fix off the backend read alone. It retracted a phantom bug and
prevented a parity regression. The general lesson: when two audits disagree about a wire contract,
the reference platform's actual serialized body is ground truth — verify it before briefing a fix.

**Factual vs inferred.** The five findings, the three shipped fixes, the three no-work closures,
and the retraction are factual (four engineer session summaries + four read-only audits). The
code-complete / Ψ-pending posture (do not promote to `mobile-stable` until on-device smoke) is
Igor's confirmed call.

---

## 2026-06-01 — Expo system theme shipped (code); mobile mirrors web's tri-state pattern, no new keys

`oglasino-expo` gains a `'system'` (follow-OS) theme option, reaching parity with web's tri-state theme. Code-complete on `new-expo-dev` across three sessions (`-1` audit, `-2` implementation, `-3` key-removal fix); `verifying` pending on-device Ψ. Spec: [`features/expo-system-theme.md`](features/expo-system-theme.md). Web does not change.

**The pattern mirrored, the mechanism not.** Following the 2026-05-30 consent-mode-mobile precedent, mobile adopts web's three-state-choice → two-state-resolved machine but none of web's web-only mechanism (cookie, consent gate, SSR pre-paint snippet, `<html>` class). New app-owned `useThemeStore` holds `theme ∈ {light,dark,system}` (the choice, default `'system'`) and `resolvedTheme ∈ {light,dark}` (never `'system'`). `setTheme(choice)` resolves + bridges + persists the choice; an internal `_setResolvedTheme(resolved)` serves the OS listener and never persists. `'system'` resolves via RN `Appearance.getColorScheme()`; a live `Appearance.addChangeListener` runs only while the choice is `'system'`.

**The apply bridge is the RN-specific glue.** Web's store applies via `.dark`-on-`<html>`; mobile can't, because NativeWind owns the binary `colorScheme` that ~20 consumers, `NAV_THEME`, and the CSS-var cascade all read. The store bridges via `colorScheme.set(resolved)`. **Invariant: `'system'` never leaves the store** — only resolved `'light'`/`'dark'` reaches NativeWind, `NAV_THEME` (`Record<'light'|'dark'>`, would return `undefined` on a `'system'` index), and the binary ternaries. This keeps all ~20 existing consumers correct with zero changes.

**Persistence is ungated AsyncStorage of the choice** (dedicated `themeStorage.ts`, key `theme_choice`, per the `consentStorage`/`authStorage` house style). Theme is functional data, not analytics — no consent gate (web gates its cookie; mobile drops both the gate and web's `SyncThemeFromCookie` reconciliation component). Default `'system'` when absent.

**One control, reachable from both surfaces.** A 3-segment Sun/Monitor/Moon control in `PortalConfigDialog` replaces the prior binary flip toggle. The `DashboardSidebar` binary toggle is removed and replaced with a button that opens `PortalConfigDialog` — theme is switched from one control, single source of truth, still reachable from the dashboard.

**No backend, no new translation keys.** An implementation pass invented three `portal.config.theme.option.*` keys to label the segments; they were removed in session `-3`. Web's equivalent control is icon-only with zero per-segment label keys — parity requires none. Each segment's `accessibilityLabel` is the raw value string, matching web's `aria-label={value}` exactly (including web's known a11y gap, logged separately).

**Process note — implementation ran before the spec existed.** Session `-2` implemented off the chat transcript before the Phase 4 spec was authored, and a mid-session brief-file swap (an unrelated bug-batch brief overwrote `.agent/brief.md`) caused a stray `ProductUserDetails.tsx` edit that was reverted clean. The spec was authored at close, against the as-built code. The invented-keys error traces to the implementation outrunning the spec; it was caught and corrected.

**Alternatives considered and rejected:**

- Per-segment label translation keys (the invented `-2` approach) — rejected; web has none, parity requires none, and they manufactured a phantom cross-repo backend dependency.
- NativeWind's native `colorScheme.set('system')` (NativeWind supports `'system'`) — rejected per the invariant; the store owns system resolution so `NAV_THEME` and every binary ternary only ever see a resolved value.
- Generic `userPreferenceStorage` as the persistence host — rejected for a dedicated `themeStorage.ts` matching the per-concern house style; `userPreferenceStorage` is a zero-caller wrapper the codebase doesn't use as the pattern.
- A separate `SyncThemeFromSystem`-style listener component — folded into `ThemeInit` (one headless init), avoiding a near-empty second component.
- Touching the bootStore gate machine to slot theme into a gate — rejected; unconditional `ThemeInit` mount applies the resolved theme under the boot overlay before paint, with no risk to the gate machine's invariants.

**Factual vs inferred.** The store/storage/listener/bridge/control/sidebar shape, the key removal, the test/lint counts, and the no-backend conclusion are factual (three session summaries + two web reference checks against the actual `ToggleButton.tsx`). The pending-Ψ posture is Igor's confirmed call.

---

## 2026-05-31 — Chat I (mobile backend-calls reduction) closed: three mobile fixes + one backend cache; most of §I already absorbed by predecessors

Chat I opened against the §I scope but a fresh Phase-2 audit on `new-expo-dev` (mobile) + `dev` (backend) confirmed the predecessors (Φ1 single-flight, boot-redesign Gate 4, version-checksums, expo-maintenance-split) had already absorbed most of it. Residual work shipped as `features/backend-calls-reduction-mobile.md`:

- **C1 (mobile) — dropped dead `idToken`.** `syncUserToBackend` no longer sends `idToken` in the `/auth/firebase-sync` body or makes the explicit `getIdToken()` call. X5 answered by backend read-only audit: `LoginRequest` has no `idToken` field; the token comes only from the `Authorization` header (interceptor-attached). Body is now `{ profileImageKey, providerId }`. Session `oglasino-expo-backend-calls-reduction-2`.
- **C2 (mobile) — cached `getUserForId`.** Was uncached, re-fetched per screen revisit (product detail, user profile). Now id-keyed read-through in the existing `userCache.ts`, cleared on logout via the existing `clearUserCache` call. Session `-3`.
- **C4 (mobile) — skip per-login Firestore read.** `ensureUserInFirestore` already skipped the doc-creation write for returning users; now also skips the `getDoc` read via a persisted per-uid AsyncStorage flag (`firestoreEnsuredStorage.ts`). The flag is set only on a confirmed create (`created !== null`), so a failed create doesn't permanently skip. Session `-4`.
- **C5 (backend) — cached `findActiveBaseSiteCodes`.** Ran uncached on every `/api` request via `BaseSiteFilter` → `getAllBaseSites`, for all clients. Now a no-TTL Redis cache `redisActiveBaseSiteCodes` (new `getActiveBaseSiteCodes()` `@Cacheable` via `@Lazy self`), warmed by `CacheWarmupService`, evictable via `CacheAdminController`. The per-request Postgres hit is gone. 710 tests green. Session `oglasino-backend-backend-calls-reduction-2`.

**The "active base sites per request" issue (issues.md 2026-05-31) was two things.** The mobile symptom was not reproduced on `new-expo-dev` — the request interceptor reads `selectedBaseSite` from the cached bootStore slot and only sets the `X-Base-Site` header; no per-request fetch. The real per-request cost was the backend's uncached codes query, fixed by C5. Igor's read: the mobile-side observation was Next.js cache behavior / an older build, not current `new-expo-dev` code.

**Deliberately left uncached:** `VersionController` and `VersionChecksumService` still read `findActiveBaseSiteCodes` directly — they build version/translation checksums and must read fresh DB state. Correct, not a gap. Do not route through the cache.

**Scope held:** C5 is the cache only. Removing `@Transactional` from `getAllBaseSites` and per-key single-flight remain with the `connection-pool-hardening` backlog feature, untouched.

**Per-pathname version check (original §I item) dropped** — boot-redesign deleted `AppVersionConfigInit.tsx`; the version check now fires once per boot (Gate 2), never on navigation. Confirmed in the audit, nothing to do.

**Status:** mobile C1/C2/C4 code-complete on `new-expo-dev`, on-device verification deferred to Ψ. Backend C5 shipped on `dev`. Both audits archived to `sessions/`.

---

## 2026-05-31 — Image-pipeline chat H closed: mobile polish complete, `adopted` (pending Ψ)

Chat H (mobile image-pipeline polish) closed. The image-pipeline core was already implemented and validated on `new-expo-dev` (see the 2026-05-30 entry); H was the final polish pass — strings, dead code, a progress-text touch-up, and one decision. Mobile status flips to `adopted`; `mobile-stable` follows on-device smoke (Ψ).

**What shipped (all on `new-expo-dev`):**

- **Progress text** wired into the product upload dialog (`UploadedProductDialog`) — it now subscribes to the shared `useUploadProgressStore` and renders per-stage status via the existing `stageLabel` helper (INPUT namespace, already-seeded `image.processing.*` keys). Previously a bare spinner. Session `oglasino-expo-image-pipeline-5`.
- **B15 — three hardcoded Serbian strings** in `ImageSourceSheet.tsx` (a live sheet, reachable from product create/edit + review) moved to translation keys. Backend seeded `image.source.title` (DIALOG), `image.source.camera` / `image.source.gallery` (BUTTONS) ×4 locales — EN+RS final, RU+CNR placeholder (native review owed, Risk Watch). Mobile swapped the three inline strings. The spec's original "B15 = two strings in two files" framing was wrong: the second file (`ProductReviewImageImport.tsx`) was dead and was deleted, not translated.
- **Dead code removed:** `ProductReviewImageImport.tsx` (zero importers; live review path is `ImagesImport`) and `app/__smoke__/upload.tsx` (self-marked smoke harness, prod-reachable expo-router route) deleted; comment references cleaned.
- **Duplicate `isPngInput`** consolidated to one definition in `processImage.ts`, imported by `uploadImages.ts`. No import cycle.

**The one decision — AppState cleanup: rely on the sweeper, don't add teardown.** Recorded in full in the 2026-05-31 AppState entry below. Grounded in a web reference audit showing web has zero abandonment-time cleanup and leans on the same backend sweeper; both platforms upload late, so mobile is not more exposed than web.

**Three scope items dropped after Phase 2 re-confirmation against `new-expo-dev` (not fixed, because not problems):**

- Stale DTO fields (`oldName`/`oldDescription`/`productState`/`moderationState`) — already removed by chat A's `toUpdateWirePayload` allow-list; backend DTO confirmed clean too.
- Dead `productUpdateNameValidator.ts` — does not exist; zero references.
- Review-upload scope — not a bug; reviews upload `scope="product"` (the `ImageScope` type doesn't even contain `review`).

**Reference audits (read-only) corroborated two cross-repo facts, neither chat-H scope:** the backend `scope` enum has no `REVIEW` member (spec wire-table line is misleading — reviews use the product prefix; spec correction owed separately), and the seed carries 29 `image.*` keys / 116 rows, not the spec's claimed 17/68 (undercount, nothing broken). Both logged for a future docs reconciliation; not fixed here.

**Remaining gate:** on-device smoke (Ψ) per `features/image-pipeline-mobile-test-cases.md` (created this chat), blocked on the pending iOS+Android rebuild. Flips `adopted` → `mobile-stable` when it passes.

**Factual vs inferred:** the shipped code, the dropped items, and the decision are factual (four reference/Phase-2 audits + the C1/B15 session summaries). The spec-undercount and review-scope-enum notes are factual (backend reference audit). The "spec correction owed" routing is Mastermind's call, not yet a committed edit.

---

## 2026-05-31 — Image-pipeline chat H: mobile relies on the backend orphan sweeper for abandoned uploads (no AppState teardown), matching web

Chat H closed the open AppState question for the mobile image pipeline: **do not add `AppState` teardown to cancel in-flight uploads when the app backgrounds; rely on the backend orphan sweeper.** Grounded in a web reference audit, not just reasoning.

**The question.** When a user uploads product images and abandons the flow before the product is created, the already-uploaded R2 bytes become orphans (uploaded to storage, referenced by no entity). Should mobile add lifecycle code to cancel/clean up on backgrounding, or rely on the backend sweeper?

**Decision: rely on the sweeper.** Reasons:

- **Web does exactly this.** A read-only web audit (2026-05-31) found web has zero abandonment-time cleanup — no `beforeunload`/`pagehide`/`visibilitychange` handler, no unmount cleanup, no cancel-handler delete, no AbortController wired into the create flow. Web's own code comments say the backend sweeper reclaims orphans. Adding AppState teardown to mobile would make mobile _more_ defensive than the reference platform, for a problem web doesn't solve client-side.
- **Both platforms upload late.** Mobile uploads bytes only at the final create step (`UploadedProductDialog` mount, expo audit item 8), exactly like web (Step-4 `UploadedProductDialog`). So mobile's orphan window is the same shape as web's — narrow, and only between the R2 PUT landing and the create POST resolving. Mobile is not more exposed than web.
- **The sweeper is real and runs in prod.** `ProductImagesRemovalJob` (backend audit) sweeps `public/products/` + `public/profiles/` with a 24h grace, `enabled=true` in prod. It is the architecture's chosen backstop.

**Cost of the decision:** orphaned R2 bytes linger up to ~24h before the sweeper reclaims them. Pre-launch, no scale concern. The alternative (AppState teardown) is net-new RN lifecycle code running on every backgrounding event — a new failure surface to close a window the sweeper already covers.

**Note on sweeper timing:** the sweeper fires at 03:00 **Europe/Belgrade** (the container's `TZ`), not 03:00 UTC — the "03:00 UTC" wording in the spec and code javadocs is inaccurate for this deployment. The 24h grace window is the figure that matters and it is correct; the wall-clock time is not load-bearing for this decision.

> **Amendment 2026-06-06 (timestamp-zone-utc):** this note's premise is now inverted. The container `TZ` was flipped `Europe/Belgrade` → `Etc/UTC` (`Dockerfile`), so the sweeper now fires at 03:00 **UTC** and the "03:00 UTC" wording in the spec and code javadocs is **accurate** as of this date. The 24h-grace point above stands unchanged; only the wall-clock-zone observation is superseded. See the 2026-06-06 "Timestamp Zone — UTC" entry at top and [features/timestamp-zone-utc.md](features/timestamp-zone-utc.md).

**Considered and rejected:** AppState teardown — earns its complexity only if abandoned-upload orphans were a measured problem, which they are not pre-launch; the sweeper is the defense-in-depth the architecture already chose, and web confirms it is the platform's pattern.

---

## 2026-05-31 — Chat D closed: mobile filtering & search polish (oglasino-expo, new-expo-dev)

Adopted web/backend filtering & search behavior on mobile via code review (on-device Ψ
pass still pending). Shipped:

- B5 — replaced RN-invalid `<div>` with `<View>`/`<Text>` in the filter fall-through
  (two sites).
- B6 + structural — `clearAllFilters` now resets the product/moderation state arrays;
  the dashboard active-filter count includes product-state at both `FiltersDialog` and
  `FloatingFiltersButton` (gated on dashboard scope); both render-time-setState patterns
  in the live filter dialog moved out of render.
- B7 + cleanup — search submit on a category screen now navigates to the category path
  (carrying the term via the portal store) instead of dropping to root; removed a
  user-visible `Test123` debug placeholder; suppressed `applyRandom`/`randomSeed` when
  searchText/orderBy is set (portal/catalog side), matching the backend's own
  suppression.
- B8 — the public user/seller list no longer reads the portal filter store; it is driven
  solely by `ownerId`, ending the filter bleed-through.
- Dialog-refresh web-parity fix — an in-place filter change now resets the list to page 0
  and re-fetches (previously required pull-to-refresh); this also fixed a pagination
  defect that mixed old/new-filter results.
- Cleanup — deleted dead `navigation/Filters.tsx` (zero importers); converted three
  `console.error` calls to `logServiceError`.

Decisions:

- **B9 dropped.** State-filter option labels render raw enum strings (ACTIVE/INACTIVE).
  This matches web (web also renders raw enums; backend has no per-value translation
  keys), so raw labels are parity, not a gap. No backend seed brief. A future i18n pass
  could translate them on both platforms.
- **Seller-page "more from seller" resolved by removal, not by the planned section swap.**
  Web's carousel depends on pagination; mobile's exhaustive infinite scroll makes a
  seller-scoped below-list section structurally duplicate-or-empty. Mobile now shows only
  the main list there. (`MORE_FROM_SELLER` stays correct on the paginated product-detail
  screen.) The Platform-adoption spec section was corrected accordingly.
- **Deep-link parser kept as a seam, not deleted.** `parseFiltersFromQueryParams` is
  uncalled but retained; B7's fix is deep-link-compatible by construction (category
  reconstructable from the path). Wiring deep-linking is future work needing a
  wire-DTO→FilterState adapter.

Deferred / open (tracked in issues.md): SELECT-type filter dispatch; the uncommitted-
branch integrity risk. On-device Ψ checks listed below.

Process note: a tool-output corruption incident occurred in one engineering session
(fabricated test numbers + false file-corruption claim), caught by the engineer and
re-verified clean in a fresh session. Logged to Risk Watch with an escalation
recommendation; see state.md.

---

## 2026-05-31 — Mobile messaging adoption onto the frozen `messaging.md` contract — code-complete on `new-expo-dev`, pending Ψ

`oglasino-expo` adopted the frozen [`features/messaging.md`](features/messaging.md) contract, **code-complete** on `new-expo-dev` across five engineer briefs behind a read-only Phase-2 audit (`audit-messaging-adoption.md`): the n=1 audit, Brief 1 store-core fixes (`oglasino-expo-messaging-adoption-2`), Brief 2 screen + UI fixes (`-3`), the Cycle B dyad break (`-4`), and the `ChatUserFunctionsDialog` deleted-peer guard (`-5`). Posture is **pending Ψ** on-device verification after the iOS+Android rebuild — **not `mobile-stable`**. Everything below is factual (grounded in the five session summaries plus the three Phase-2 spec-validations — backend, web, rules); inferred items are marked.

**Behavior brought to parity with web.** The existing-chat send was rewritten into one atomic 4-op `writeBatch`, including the previously-missing chat-root `lastMessage`/`lastUpdated` merge that had left the root stale after the first message. `sendMessage` and `deleteChat` now **throw on failure** (after their cleanup) and the screens catch + toast; `deleteChat` is Firestore-first. `getActiveChat`'s fallback derives the counterparty from `data.users` (not the non-existent `data.withUserFirebaseUid` root field) and enriches via the cache-aware user fetch. Mark-seen moves `seenLocal` into the after-commit `.then`. Added: the deleted-peer fallback render + crash guard (shared `isDeletedPeer` helper), the chat-list block badge, chat-list load-more, URL linkify (schemes restricted by construction), five hardcoded strings swapped to existing keys, the `tempProductReason → tempProductContext` rename, and the Cycle B dyad break.

**Banked decisions (decisions, not open questions):**

- **No new backend translation seeds.** All five hardcoded mobile strings already had backend keys in all four locales (`search.placeholder`, `messages.select.placeholder`, `messages.load.more`, `blocking.label`, `blocked.by.label`) — mobile consumes the existing keys. No backend translation brief was needed.
- **Forward block-doc field is `blockedUserId`.** Web and mobile both write `blockedUserId` on the forward index `userblocks/{owner}/blocked/{blocked}`; spec §3.4's `blockerId` was the stale party (`blockerId` lives only on the reverse index `userblocksReverse`). The spec is corrected in §3.4 (see Docs/QA close-out).
- **Send/delete-failure UX is "store throws, screen catches + toasts,"** mirroring web — translations live at the React boundary, so the store re-throws rather than toasting internally.
- **Delete-failure toast reuses `messages.send.failed.toast`.** No delete-specific key is seeded — "Try again" is acceptable copy for the rare delete-failure path. This resolves the only copy nuance surfaced in the briefs; it is a decision, not an issue.
- **Cycle B resolved by a lazy `require` on the narrower edge, not a new registry module** (Part 4a — the fix is smaller than the cycle it removes). See [`issues.md`](issues.md) 2026-05-28 Cycle B entry (now `fixed`).

**One gap filed, not resolved here.** Backend spec-validation found spec §3.6's admin view-token override (admins minting view-tokens for arbitrary chats) is not implemented backend-side — a non-participant admin gets 403 `NOT_CHAT_MEMBER`. The surface is backend + web admin, **not** mobile (consumer-only), so it does not block this adoption. Filed as the 2026-05-31 `issues.md` entry (open/medium) for separate Mastermind triage: correct the spec (aspirational) or open a backend brief (missing feature). §3.6 left untouched pending that.

**Verification caveat (low, inferred-confidence note).** Brief 3's reported lint count (80 warnings / 0 errors, the held baseline) was asserted-not-rendered because a harness output glitch swallowed the final lint stdout; the figure rests on deterministic `eslint-disable-next-line` semantics bracketed by measured 81-bare / 82-misplaced runs. Igor confirms on his own `npm run lint` at commit; the only knob is the one `eslint-disable` directive line — no logic is implicated.

---

## 2026-05-31 — Consent Mode — Mobile shipped (Part C cleanup + Part G consent UI) — F-facing gate contract, `allowNotifications` deliberate gap, first-run-prompt placement amended to `ready`

[`features/consent-mode-mobile.md`](features/consent-mode-mobile.md) is code-complete and merged-ready on `new-expo-dev`, status `shipped` — on-device runtime verification (string resolution + privacy-link navigation against a seeded build) is carried by the Ψ pass, like the rest of this branch's `verifying` items. Built across three engineer sessions plus the backend seed: Part C cleanup (`oglasino-expo-consent-mode-mobile-1`), Part G build (`-2`), the first-run prompt at the amended placement (`-3`), and the COOKIES seed (`oglasino-backend-consent-mode-mobile-1`). This entry records the three items the spec's implementation-order step 4 calls for; the consent model itself is in the 2026-05-30 entry below.

**The F-facing gate contract — and there is no in-repo caller yet, by design.** G ships `isAnalyticsConsentGranted()` (`src/lib/consent/analyticsGate.ts`) — synchronous, default-deny, reads the device-local consent decision off `useConsentStore`. Chat F's analytics-SDK init is the intended (and only) consumer: F gates init on `(ATT-allows-on-iOS, or Android) AND isAnalyticsConsentGranted()`, and subscribes to the reactive store so toggling analytics off stops collection without an app restart. **The gate has no caller in the repo today — that is intentional**: G builds the signal, F consumes it when F lands. A future reader who finds an "unused" gate should not treat it as orphaned; it is waiting for chat F.

**`allowNotifications` is a deliberately-dead toggle after C's cleanup.** Part C cleaned the settings screen but deliberately left the non-functional `allowNotifications` toggle in place (never seeded, never sent) — a future notifications feature owns wiring it. The screen therefore retains exactly one intentionally-dead toggle. Recorded so nobody "fixes" it into a collision with that future feature. (Consistent with the 2026-05-30 entry's reversal of the pre-merge chat-C "remove or wire it" question.)

**First-run-prompt placement amended from `intro-picker` to `ready`.** The spec originally placed the prompt in the `intro-picker` boot state (in/after `BaseSiteSelector`). Implementation (brief 2) found it collides with the boot-redesign portal-mount invariant and misses existing users, and stopped rather than force it; Mastermind amended the placement to `ready`. Two reasons: (a) the privacy link (`/privacy`) only resolves once the portal `<Stack>` is mounted, which the boot redesign restricts to `ready`/`updating` — at `intro-picker` there is no navigator and the link would be dead; (b) an existing user with a stored base site boots straight to `ready` and never enters `intro-picker`, so an `intro-picker`-slotted prompt would skip every existing user with no decision, whereas the `ready` placement reaches them. The prompt ships as a one-time `ready`-gated overlay in `app/_layout.tsx` (gated `hydrated && decision === null`), alongside the other visible overlays — not as an `AppInit` child (that headless slot is chat F's analytics-init home). No boot-machine invariant was touched.

**Translations.** The seven `mobile.consent.*` keys seeded into the existing COOKIES namespace across all four locales (EN final; RS/RU/CNR placeholder, pending native-translator review per the established precedent); COOKIES 33 → 40 keys. The COOKIES checksum is auto-derived on boot — no stored value was hand-edited.

---

## 2026-05-30 — Chat G (C+G merged): mobile consent model — single-axis device-local analytics consent + `allowNotifications` deliberately untouched

The merged C+G Mastermind chat authored [`features/consent-mode-mobile.md`](features/consent-mode-mobile.md) (Phase 4) behind three read-only audits. Spec status is `not-started` — Phase 5 engineering briefs come next. The decisions below are the ones not obvious from the spec body.

**Model: single-axis, two-state analytics consent.** Mobile's consent is one signal (`analytics: boolean`) plus a `decidedAt` sentinel and a `version`, stored device-local in AsyncStorage via a per-concern `consentStorage` module + a reactive `useConsentStore` (the confirmed mobile house style — `authStorage`/`softUpdateDismissal`, not the zero-caller `userPreferenceStorage` wrapper). It exposes one imperative, synchronous, default-deny gate `isAnalyticsConsentGranted()`. **Rejected** the web four-signal Consent Mode model (no cookies, no SSR `<head>` snippet, no Google Consent Mode on a phone) and ATT-only gating (leaves EU Android ungated). The subtle point: web applies its imperative `if (granted)` gate to _cookie persistence_ and delegates analytics to Google _declaratively_; mobile has no declarative path, so it applies the same imperative gating _shape_ to **analytics-SDK init** — the opposite axis from where web applies it. Mobile mirrors web's pattern, not its target. Mobile's other AsyncStorage data (base-site, language, theme, card-size, auth, push timing, translation cache) is "strictly necessary"/"functional" and needs no gate; disclosure of it is a privacy-policy concern, separate from consent.

**F-facing contract.** G owns the consent decision (storage + store + gate) and the consent UI (first-run prompt + settings toggle). F (mobile analytics) owns the analytics SDK install/init and the iOS App Tracking Transparency prompt. ATT is Apple's IDFA gate, **not** GDPR analytics consent — F's init gates on **both**: `(ATT-allows-on-iOS, or Android) AND isAnalyticsConsentGranted()`. F subscribes to the reactive store so toggling analytics off stops collection without an app restart. G installs no SDK, touches no ATT, and does not wire the dead `firebaseAnalytics.ts` (F replaces it with a native init).

**`allowNotifications` deliberately left untouched.** Part C cleans the settings screen but does **not** remove or wire the non-functional `allowNotifications` toggle (never seeded, never sent) — a future notifications feature owns wiring it. After C's cleanup the screen retains one intentionally-dead toggle. Recorded here so it is not mistaken for an oversight. (This reverses the pre-merge chat-C "remove or wire it" open question.)

**C+G merged into one chat** because both touch `app/owner/dashboard/user.tsx`. C runs first (removes `allowPreferenceCookies` end-to-end, the two dead cookie-label sections referencing five backend-deleted COOKIES keys, dead `ConsentData`/`GlobalCookie` types, the B2 promo-emails double-set / B17 placeholder / B16 console-error fixes, and a save-failure message); G builds the consent UI on the cleaned screen. No separate C chat opens. New `mobile.consent.*` strings seed into the existing COOKIES namespace (already in mobile's fetch list — zero fetch/namespace changes; EN final, RS/RU/CNR placeholders for native review), plus a COOKIES checksum.

**Three read-only audits (expo / web / backend) confirmed the going-in assumptions** before any implementation: the backend removed `allowPreferenceCookies` end-to-end; the five COOKIES keys (`required.label`, `required.description`, `required.sub.description`, `config.label`, `config.description`) are absent backend-side and still referenced mobile-side (the broken-labels regression is real and visible today); and the Φ4 "grenade" was already defused and B18 already fixed by the foundation work. The audits also surfaced three out-of-scope findings now in [`issues.md`](issues.md) (backend `allowPhoneCalling` write-path ghost; web `allowPreferenceCookies` dead type fields; web consent spec-vs-code drifts). Audit deliverables archived to `sessions/`.

---

## 2026-05-30 — Chat A: mobile product create/edit/validation rebuilt onto the frozen `product-validation` contract (code-complete, pending Ψ)

The `oglasino-expo` product create/edit/validation surface was rebuilt onto the frozen [`features/product-validation.md`](features/product-validation.md) contract. Code-complete on `new-expo-dev` across briefs A1–A5 plus a cleanup brief, a URL-parity brief, and translation-seed sessions. A mobile-vs-web delta audit confirms behavioral parity with web except items Igor is deferring to the post-smoke-test pass. **Not yet verified on a device** — status is `code-complete / pending Ψ on-device smoke test`; this entry does **not** promote product-validation to `mobile-stable`, and the image-pipeline (product's image dependency, validated separately) is **not** claimed done here.

**The rebuild.** The retired combined `POST /secure/product/addUpdate` is replaced by the three frozen endpoints `create` / `update` / `pre-validate`. The update wire payload is narrowed by allow-list (`toUpdateWirePayload`) — fixing the conventions Part 11 trust-boundary violation: the client no longer sends `oldName` / `oldDescription` / `productState` / `moderationState` / categories / `regionAndCity` / `free`. Client validation is reduced to structural-only with error keys migrated to the `ERRORS` `product.<field>.<code>` form; content moderation is server-only; per-field server errors render via `parseServiceError`. Briefs A1–A5 on `new-expo-dev`.

**Six banked decisions:**

- **Rebuild onto the frozen contract** (above). The three endpoints are reused as-is, no mobile-specific routes; mobile adoption was frontend-only RN work.
- **MIME-undefined-passes (RN translation).** The client flags `product.image.invalid_type` only for a _present_ non-allowlisted MIME type; an _absent_ `mimeType` passes through to the server boundary. RN's picker `mimeType` is optional/inconsistent on iOS (web validates the browser's reliable `File.type`), so a strict "absent == reject" would block legitimate iOS uploads. The server is the real content-type boundary. This is a correct RN translation of web's behavior, not a deviation.
- **`__system` handling (option A).** Mobile's `parseServiceError` excludes `field: null` from `byField` (differs from web, which collapses such errors to `byField['__system']`). Mobile reads the object-level / rate-limit entry by scanning `errors` for `field === null` via the shared `findSystemError`, leaving `parseServiceError` unchanged (it is shared with `ReportDialog`). Step-routing consumes `byField` keys, so a system-only failure routes to the fallback step 3 — web-identical behavior with no shared-helper change.
- **System/user translation keys.** Mobile renders the `translationKey` the backend _sends_ (post-error-code-split: `system.*`, `user.*`); it never hardcodes a system/user key. The only client-constructed keys are `product.<field>.*`. The live rate-limit key is `system.rate_limited`.
- **Create-success UX (RN translation of web).** In-app navigation on "View link" plus `triggerDashboardReload` (a Zustand nonce remount) is the RN equivalent of web's `window.open(_blank)` + `window.location.reload()`. Web's `onFinish` hook is dead (no caller) and was removed on mobile.
- **Product URL standardized (banked direction).** The mobile product URL is to be aligned to web's canonical `https://oglasino.com/{locale}/product/{id}/{slug}` (no `www`), via the URL-parity brief — fixing the prior `www.oglasino.rs` (`getNormalizedProductUrl`) / `www.oglasino.com` (`ShareProductButton`) / raw-slug divergence and adding the locale segment. The mobile delta audit (`oglasino-expo-product-validation-8`) surfaced this divergence as an open parity gap; the standardization is banked here as Igor's decision. **Landed-in-code status not verifiable from the archived artifacts** — no discrete URL-parity session summary was present in `oglasino-expo/.agent/` to archive, and the latest summaries (A5 / delta audit) still show the divergent base URLs. Reconcile against the actual `new-expo-dev` code in the post-smoke pass.

**Remaining gate.** On-device smoke (Ψ) is the last gate before `mobile-stable`. Product create exercises the image-upload pipeline, so product Ψ depends on the pending iOS+Android rebuild landing first (the rebuild must also carry the image-pipeline V6/V9 fixes and the `netinfo` module — see `state.md` Risk Watch). Igor adds issues after the on-device smoke test; no `issues.md` entries were written in this close-out.

**Factual vs inferred.** Factual (from the A1–A5 / cleanup / Phase-2 audit / delta-audit / translation-seed session summaries and the read-only web/backend reference audits, all archived to `sessions/`): the endpoint rebuild, the allow-list narrowing, five of the six decisions as implemented, behavioral parity (minus the open URL gap) per the delta audit, the live `system.rate_limited` key. Inferred / banked-by-Mastermind (confirmed by Igor): the do-not-promote / pending-Ψ posture, the MIME-undefined and `__system`-option-A translations as correct-not-deviation, and the product-URL standardization (banked as direction — landed status owed; see the bullet above).

---

## 2026-05-30 — Mobile image-pipeline adoption validated and brought to conformance; built off-process, verified via audit chain

The mobile (`oglasino-expo`) half of the [image pipeline](features/image-pipeline.md) was found **already implemented** on `new-expo-dev` — but it had landed **outside this orchestration**, via `feature/image-pipeline-v2` (squash commit `016da95`, labelled "changes by claude for image-pipeline, not fully tested"). That off-process landing is why `state.md` still recorded mobile as `not-started`: no orchestrated chat had adopted it.

**Validation chain instead of a build brief.** Because the code already existed but was self-labelled "not fully tested," a four-step read-only validation chain was run rather than a from-scratch implementation: web reference audit → backend reference audit → mobile current-state audit → mobile deep validation (tests run). The chain confirmed the implementation **structurally complete and contract-conformant** except for two gaps, now fixed.

**Reference-audit-first surfaced stale spec.** Auditing web and backend code first (before validating mobile) proved that several `features/image-pipeline.md` sections were stale against the real code — the "Deferred items" list (view-token Zustand store and comprehensive retry policy were both already built on both platforms), the four divergent translation-key names, the upload `scope` enum (missing the live `review` scope), and the variant table (missing the `original` passthrough). All corrected in this close-out.

**Two conformance gaps fixed** in session `oglasino-expo-image-pipeline-4` (tests 109 → 110):

- **V6 — HEIC stage label localization.** Mobile requested `image.processing.converting-heic` (hyphen); the seeded key is `image.processing.converting_heic` (underscore), so SR/RU/CNR users got a silent English fallback. Closed via a `STAGE_KEY_OVERRIDES` entry (`'converting-heic' → 'converting_heic'`) + test.
- **V9 — orphan cleanup on save-failure.** `cleanupOrphanImages` was wired for the product and review surfaces but missing on the avatar and chat surfaces. Closed by wiring it into both, matching the product/review pattern.

**V4 decision (Igor) — locked.** Mobile keeps its 429 behavior: it honors `Retry-After` and retries once, whereas web surfaces immediately. Web parity is **not** required — mobile's behavior is the better mobile UX. This is **not a defect** and is **not filed as a bug**; recorded here only.

**Remaining gate.** On-device smoke (gate 2) is the last gate before `shipped`/`mobile-stable`, executed by Igor against the test-case doc at [`features/image-pipeline-mobile-test-cases.md`](features/image-pipeline-mobile-test-cases.md). The installed dev builds predate the V6/V9 fixes and the `netinfo` module, so the pending iOS+Android rebuild must land first for smoke to be meaningful (see `state.md` Risk Watch).

**Honesty note.** This feature's mobile half was built off-process and labelled "not fully tested." It was brought into the orchestration via a validation chain — not a from-scratch implementation — which is precisely why a validation chain (not just a build brief) was run, and why the spec required reconciliation against the as-built code.

**Factual vs inferred.** Everything above is **factual** — drawn from the four `oglasino-expo-image-pipeline` session summaries (current-state audit, audit deliverable, deep validation, V6/V9 fix), the web and backend reference audits, and Igor's V4 decision. Nothing inferred.

---

## 2026-05-29 — Φ4 (mobile service layer + error contract) shipped — error-surfacing plumbing, review-report wire, offline gate

The last of the four Expo structural-foundation chats ([`features/expo-service-error-contract.md`](features/expo-service-error-contract.md)) is shipped. Three code briefs landed on `oglasino-expo` `new-expo-dev` (2026-05-29) behind a read-only Phase-2 audit. Status: `shipped (code)` — on-device airplane-mode verification is deferred to the Ψ chat. With Φ4 done, the Expo structural foundation program is complete and the A–I feature queue opens; chat A (mobile validation rebuild) is first, since it was blocked on this service layer.

**What shipped (3 briefs, all `oglasino-expo` `new-expo-dev`).**

- **Brief 1 — error-surfacing plumbing.** One shared helper `parseServiceError` (`src/lib/utils/parseServiceError.ts`) parses the Part 7 `{errors:[{field,code,translationKey}]}` array into `{ errors, byField }` (first-error-per-field wins; object-level `field: null` errors stay in `errors`, excluded from `byField`). It renders nothing, never touches i18n, and tolerates any non-contract error (network, malformed body, non-axios throw, `null`) by returning `{ errors: [], byField: {} }` — never throws. ~14 services were converted from swallow-and-return-sentinel to surface-via-throw (`logServiceError(...)` then `throw err`), one consistent pattern; throw was chosen over a discriminated-result return because it kept every service signature and call site unchanged. Unreachable non-2xx else-branches removed as cleanup.
- **Brief 2 — review-report wire.** `reportedReviewId` threaded button → dialog → body; the two dashboard review cards (`ReceivedReviewCard`, `GivenReviewCard`) re-pointed from `reportedProductId` to `reportedReviewId`; the dishonest required typing on `ReportDialog`'s target ids made optional; the dead `error`-flag boolean inversion fixed (`reportService.ts:16,29`); the two REVIEW error codes (`REPORTED_REVIEW_ID_REQUIRED`, `REPORTED_REVIEW_NOT_FOUND`) confirmed to resolve on mobile off the backend ERRORS seed and surfaced as translated messages off brief 1's seam. No mobile-side key authored; no missing-key flag.
- **Brief 3 — offline gate.** `@react-native-community/netinfo` installed (11.4.1, the exact SDK-54-pinned version, not in expo-doctor's mismatch list). A connectivity Gate 0 added ahead of the maintenance gate in `bootStore` (`NetInfo.fetch()` once; only a certain `isConnected === false` diverts to offline — online / unknown (`null`) / a thrown read fall through to the maintenance gate unchanged); a new `'offline'` `BootStatus` + offline screen (reuses `BaseSiteSelector` via an `isOffline` prop); reconnect re-entry via `OfflineReconnectInit` mirroring `MaintenancePollInit`; non-destructive re-entry honoring the three boot-redesign invariants.

**Foundation scope held.** Error-surfacing is plumbing only — services hand the structured error up; no screen wiring. Turning `code`/`translationKey` into displayed text is the feature chats' job (chat A onward).

**The error-code split was invisible to mobile.** Mobile keys on `code`, which the four-enum split left byte-identical; `isErrorWithCode` is unaffected.

**Review-report scope.** The wire fix (dashboard cards → `reportedReviewId`) landed here; no public-review report surface was added (none exists, none wanted). This absorbed the planned standalone `oglasino-expo-review-reports` mobile chat — it is not opened separately.

**`reportedUserId` reconciliation.** REVIEW reports carry only `reportedReviewId`; reporter identity is server-derived; the dialog's target-id types were made optional to match the real per-type mounts. Confirmed against the backend `review-reports` contract.

**Offline copy is a justified hardcoded fallback.** A backend-seeded ERRORS key cannot resolve at the offline point by construction (Gate 0 runs before i18n init, and offline is exactly when the ERRORS fetch fails). The offline screen uses a minimal hardcoded Serbian string matching the file's existing `INTRO_FALLBACK` precedent. Localized offline copy, if ever wanted, belongs in an app-bundled i18n resource, not a backend key — out of scope.

**Risk Watch closed.** The "Mobile service layer silently swallows backend validation errors" row (structural audit F18) is closed by this chat.

**Carry-forward (recorded here because chat A and the Ω chat read `decisions.md` at startup):**

- **Chat A inherits a list of callers that now throw.** Brief 1's surface-via-throw means ~10 callers now receive a throw where they previously got a sentinel — three with no catch at all: `app/owner/dashboard/user.tsx:163`, `BasicInfoProductDialog.tsx:82`, `FollowUserButton.tsx:43`. This is the designed seam: foundation surfaces, chat A catches and displays. The full caller list is in the brief-1 (error-plumbing) session summary [`sessions/2026-05-29-oglasino-expo-service-error-contract-2.md`](sessions/2026-05-29-oglasino-expo-service-error-contract-2.md) — note `-2`, not `-1` (`-1` is the Phase-2 audit summary). Chat A's intake should start from it.
- **For the Ω structural-sweep chat:** dead `init/baseSitesService.ts:fetchBaseSites` (legacy swallow, superseded by Gate 3); dead `updateUserAuth` export; the expo-doctor patch drift on 11 Expo-owned packages (`npx expo install --check` resolves it). None are netinfo.
- **Cosmetic, for whichever chat polishes the report surface:** the two review cards pass `report.product.*` title/description copy on a REVIEW report (wire correct, header text says "product"); a `report.review.*` title/description pair would be a backend-seed add.

**Factual vs inferred.** Factual (from the three shipped session summaries + the Phase-2 audit, all archived to `sessions/`): the helper / services / wire / gate as shipped, the invariant compliance, the netinfo version, the key-resolution mechanism, the offline chicken-and-egg, the caller list. Inferred (Mastermind calls confirmed by Igor): foundation-plumbing-only scope; review-wire-here / public-surface-dropped / standalone-chat-absorbed; keep-the-hardcoded-fallback.

---

## 2026-05-29 — version-checksums-per-language shipped — `/versions` translation checksums are now per-(namespace, language)

`GET /api/public/versions` ([`features/version-checksums-per-language.md`](features/version-checksums-per-language.md)) now keys translation checksums by namespace AND language. The per-NS-collapsed-across-languages transitional contract is replaced. Backend shipped on `dev`; mobile shipped on `new-expo-dev`; coordinated single deploy, web unaffected. End-to-end smoke passed against a freshly-reset backend DB: a single-language admin edit moved only that language's checksum in live `/versions`, a user on a different language refetched nothing, a user on the changed language refetched exactly that namespace.

**Headline.** `VersionsResponseDTO.translations` changed from `Map<String,String>` (namespace → checksum) to `Map<String, Map<String,String>>` (namespace → language → checksum). The hash algorithm, the `key|value` line format, and the 16-hex (SHA-256 first-16) truncation are unchanged — only the scope of the hashed material narrowed from all-languages-combined to one language. A translation edit in one language now bumps only that `(namespace, language)` checksum. The observable payoff the feature was built for: a Serbian mobile user refetches nothing when a Russian-only translation changes.

**Four languages, not three.** The per-language space is `sr`, `cnr`, `en`, `ru`. CNR (Montenegrin) is a real fourth language with its own stored translation rows — confirmed live (`COMMON`'s four per-language checksums are all distinct). The `cnr → SR` collapse in `moderation/SupportedLanguage` is moderation-only and is NOT on the checksum or translation-serving path; conventions Part 9's "Montenegrin aliases to SR" is routing/moderation only and does not touch translation data.

**Posture B — every pair carries a checksum.** Every `(namespace, language)` pair gets a real checksum, including genuinely-empty pairs, which carry the empty-content hash `e3b0c44298fc1c14` rather than being omitted or empty-string. This preserved the mobile gate's existing missing-checksum semantics unchanged (a `undefined` per-language lookup still means "namespace absent from the response" → maintenance, exactly as before). Posture A (omit absent pairs) was rejected because it would have split the single `undefined` meaning into two and enlarged the mobile change. Cost of B: a genuinely-empty pair fetches once per install then matches forever — self-healing, never a maintenance trigger. No genuinely-empty pair exists in current seed data, so this cost is latent.

**Backend (`oglasino-backend`, `dev`).** `VersionChecksumService.computeTranslationChecksum` now takes `(namespace, langCode)`, sourcing rows from the existing per-language `loadTranslationsFromDb`. The old all-languages overload and its now-orphaned `TranslationRepository` collaborator were deleted. The config checksum keys moved from 22 `translations.checksum.<NS>` rows to 88 `translations.checksum.<NS>.<lang>` rows (22 namespaces × 4 languages), pre-seeded per the Part 12 V1-fold convention; the old 22 rows were removed from the seed file; `persistChecksum`'s throw-if-missing invariant was kept, not relaxed. Boot rebuild (`onAppReady`) and admin-edit invalidation (`rebuildTranslationCacheAsync`) both tightened from namespace-wide to per-(namespace, language) — the admin-edit tightening is core scope (it is what makes a one-language edit stop recomputing sibling languages) and incidentally removes the prior concurrent-same-namespace-edit race. The live language set is derived from `languageRepository.findAll()`, not hardcoded; it returns exactly the four codes, so 88 is correct. The `redisTranslations` payload cache was already per-(namespace, language) and was not touched. `getVersions` stays parameterless — trust boundary clean (Part 11).

**Mobile (`oglasino-expo`, `new-expo-dev`).** A four-edit-point swap, all in `bootFreshness.ts` and `bootStore.ts`: (1) the `VersionsResponse` type's `translations` value widened from `string` to `Partial<Record<string, string>>`; (2) `translationsChecksumKey(ns, lang)` now returns `translations_checksum_<NS>_<lang>` — the `lang` arg, already in the signature and passed at every call site from the boot-redesign isolation, became load-bearing with zero call-site churn; (3) `isNamespaceStaleForActiveLanguage` reads `backendChecksums[ns]?.[activeLang]`; (4) the gate-body read at `bootStore.ts:408` indexes `[activeLang]` so the gate persists the per-language checksum **string**, not the map object. The fourth edit point is the one the boot-redesign's three-helper isolation contract did not enumerate — the Phase 2 audit found it by reading the gate body; without it the gate would have persisted the entire per-language map as the stored checksum, corrupting every later boot's comparison. A dedicated test guards it. The fetch primitive, i18n registration, catalog block, payload storage key, and 20-namespace intersection were untouched.

**The three boot-redesign amendments this feature was said to "owe" are already in sync.** The boot-redesign close (2026-05-29) folded in the dev-seed wording, the per-language payload key shape, and the namespace-count drift. This feature did not re-do them; it confirms they are settled. The roadmap pointer the boot-redesign feature added (`future/version-checksums-per-language.md`) is closed — that file was superseded by `features/version-checksums-per-language.md` during this feature's Phase 4 and deleted.

**`/v2` parallel endpoint rejected.** Pre-launch, no real users, the coordinated single deploy cost is near-zero and a parallel endpoint is permanent debt. The existing endpoint changed shape in place.

**Carried for stage/post-launch (not blockers):** the backend's stale-local-DB note — on an already-seeded (non-reset) local DB the 22 retired `translations.checksum.<NS>` rows linger as unused rows (the new code never reads them; they vanish on a clean reset); harmless. The id gap at seed ids 58–79 (retired rows) is cosmetic, commented as retired, left as-is to avoid churning the catalog rows.

**Alternatives considered and rejected:** posture A (omit absent pairs) — enlarges the mobile change by splitting the `undefined` meaning; flat composite-key DTO (`"<NS>.<lang>"`) — the mobile swap codes against the nested map; relaxing `persistChecksum`'s throw-if-missing to create-on-write — the pre-seed is the safety contract; a dedicated per-language repository query — reused the existing `loadTranslationsFromDb` per the simplicity guideline; `/v2` parallel endpoint — coordinated deploy is cheaper pre-launch.

**Factual vs inferred.** The architecture, the four-language confirmation, the backend and mobile edits, the four-edit-point mobile swap, and the smoke result are factual — drawn from the two Phase 2 audits (now feature-scoped ground truth), the backend and mobile Phase 5 session summaries, and Igor's coordinated end-to-end smoke. Nothing in this entry is inferred.

---

## 2026-05-29 — Review reporting built end-to-end; `ReportType.REVIEW` + validated `reportedReviewId`

Reporting a review was wired on web but absent on backend — `ReportType` was `{PRODUCT, USER}`, and web sent `reportType="REVIEW"` with the review id in the `reportedProductId` slot. The unknown enum value failed Jackson deserialization and returned HTTP 500 (not the `getOwnerId`-misfile/422 the original `issues.md` entry claimed — the audit corrected this; see the issues.md amendment). Built the missing backend half and aligned the web wire. Code-complete on backend (`dev`) and web (`stage`), 2026-05-29.

**Backend.** `ReportType` gains `REVIEW`; the `report_type` CHECK constraint widened to `('PRODUCT','USER','REVIEW')` (pre-prod V1 fold, no new migration, Part 12); `Report` entity + `report` table gain a `reported_review_id` column (plain `Long`, nullable, no FK — mirrors the `reportedProductId` posture); `ReportRequestDTO` gains `reportedReviewId`. A REVIEW branch in `DefaultReportService` validates server-side: required-check → existence-check (`reviewRepository.findById(...).orElseThrow(REPORTED_REVIEW_NOT_FOUND)`) → self-report block. Dedupe extended from a binary ternary to a 3-way switch (`report:user:<id>:target:REVIEW:<reviewId>`, 24h TTL unchanged). Two new `ReportErrorCode` constants (`REPORTED_REVIEW_ID_REQUIRED`, `REPORTED_REVIEW_NOT_FOUND`, both 422) + 8 seed rows.

**Self-report is reviewer-only, null-safe.** Reject if the reporter is the review's `reviewer`; the `targetUser` (the seller the review is about) is explicitly allowed to report — that is the primary legitimate use. The reviewer-id read is null-guarded because `Review.reviewer` is nulled on author anonymisation; a null author is never "self," so an anonymised review stays reportable by its target. The engineer caught the NPE the literal spec expression would have produced and implemented it defensively; the spec was amended to match (§6.3).

**No deletion-state guard for REVIEW** (unlike the USER branch's `REPORTED_USER_PENDING_DELETION`) — a review is its own object, reportable regardless of whether its author or subject is pending deletion. Locked decision.

**Trust boundary (Part 11): clean.** Reporter identity stays server-derived from `SecurityContextHolder`. The only client-supplied value, `reportedReviewId`, is resolved against authoritative data via `findById` (coded 422 if absent) and the self-report check reads `review.getReviewer().getId()` from the persisted row, never a client claim. Same posture as the 2026-05-28 USER/PRODUCT trust-boundary fix. The persist block was tightened so each report type persists only its own validated target slot (a small bonus hardening of USER reports, which previously persisted the client's raw `reportedProductId` unvalidated).

**Web.** `reportedReviewId` added to `ReportRequest`; `ReceivedReviewCard` mount swapped from `reportedProductId` to `reportedReviewId`; field threaded through `ReportButton`/`ReportDialog`; the two new REVIEW codes surfaced as translated messages (gated on `code`, rendering the backend `translationKey`) instead of the generic fail toast.

**Considered and rejected:** (a) remove the wire, (c) repurpose to a PRODUCT report against `review.reviewedProduct.productId` — both rejected per Igor's "the feature is missing, build it." A participation check beyond existence + self-report — rejected per the 2026-05-28 precedent (existence + self-report + dedupe closes the abuse vector). Repurposing `reportedProductId` for reviews instead of a dedicated field — rejected; dedicated `reportedReviewId` is cleaner.

**Known low-severity follow-ups (not filed as issues per Igor's no-new-issues call this chat):** (1) a cross-feature error-shape bug found mid-feature — `reportService.ts`/`reviewService.ts` read `(err as AxiosError).response?.status`, but `BACKEND_API` rejects unwrapped, so the 406/403/409 status branches were dead; fixed this chat on its own `fix/report-406-error-shape` branch (grep-verified the blast radius was exactly those two sites). (2) `serviceLog.ts:24` reads only the nested `errorCode` shape, silently dropping error-code from log lines on `BACKEND_API` rejections — logging fidelity only, one-line tolerant-read fix available. (3) No `reportService`/`reviewService` service-test suite exists; the REVIEW-code surfacing path has no automated coverage. (4) Three `?? .data` tolerant readers signal a settled contract read defensively; a future consolidation could simplify. None block; recorded here so they survive without an issues.md entry.

**Factual vs inferred.** The backend build, the trust-boundary verdict, the null-safe self-report fix, the dedupe extension, the web wire swap, and the 406-bug fix are all factual (shipped sessions + audit Part C). The build-not-remove, reviewer-only self-report, no-deletion-guard, and dedicated-field decisions are Mastermind calls confirmed by Igor.

---

## 2026-05-29 — `ProductErrorCode` split into domain enums behind a shared `ErrorCode` interface

`ProductErrorCode` had become a dumping ground — 47 constants, 11 of them not product errors. Split the 11 out: 7 system/request-level codes into a new `SystemErrorCode`, 4 user/profile codes into a new `UserErrorCode`. `ProductErrorCode` now holds only its 36 genuine product codes; `ReportErrorCode` unchanged in membership. The misprefixed translation keys were renamed in lockstep: `product.system.*` → `system.*` (7 keys), `product.user.setup_incomplete` → `user.setup_incomplete`, `displayName.*` → `user.display_name.*` (44 seed rows total, renamed in place across the four locale files — no ID/text/row-count change). Code-complete on backend (`dev`) and web (`stage`), 2026-05-29.

**The interface was the load-bearing decision.** A pure enum-home move could not compile: `ProductValidationException` and `ProductErrorResponse` were hard-typed to `ProductErrorCode`, and the moved codes flow through them (8 of the 11). The backend engineer surfaced this before writing code (challenge-the-brief working as intended). Resolution: a shared `interface ErrorCode { name(); getTranslationKey(); getHttpStatus(); }` implemented by all four enums; `ProductValidationException` + `ProductErrorResponse` widened to `ErrorCode`. One exception type still carries a mixed-domain code list (the 403-beats-422 severity test stays green). The earlier "mirror ReportErrorCode exactly, no interface" framing was a description of the old shape mistaken for a constraint — the interface earns its place under Part 4a with four concrete implementers.

**`resolveTranslationKey` reworked to a registry.** The linear `ProductErrorCode.valueOf → ReportErrorCode.valueOf` chain was replaced with a single static `Map<String,String>` built once at class load from all four enums' `name() → translationKey` pairs, failing loud (`IllegalStateException`) on a duplicate constant name across enums. Gives the one-lookup simplicity of a single enum while keeping the domain boundaries.

**Wire is unchanged.** `code` (the enum `.name()`), HTTP status, and the `{field, code, translationKey}` envelope are byte-identical; only the enum a code lives in and its `translationKey` string moved. Web keys UI off `translationKey`, not `code`, so the only web touch was renaming the one hardcoded literal (`productService.ts` 429-synth) and four test fixtures — grep-confirmed zero `product.system.*` literals remain in web.

**Seed-coverage tests.** Three new tests (`SystemErrorCodeTest`, `UserErrorCodeTest`, `ReportErrorCodeTest`) clone `ProductErrorCodeTest`'s shape; `ReportErrorCodeTest` closes a pre-existing gap (that enum had no seed-coverage enforcement). All four error-code enums now enforce that every constant's key resolves in the EN `ERRORS` seed.

**Considered and rejected:** (1) one combined `ErrorCode` enum holding all 47+ — recreates the dumping ground under a new name, biggest diff, cross-domain message-name collision risk in `resolveTranslationKey`; the registry gives the lookup simplicity without it. (2) Per-enum exceptions mirroring `ReportValidationException` — can't carry a mixed-domain list, breaks the severity-precedence test, changes 8 throw sites by exception type. (3) Decoupling `FieldError` from the enum with raw fields + overloaded constructors — larger internals change, loses the compile-time enum guarantee. (4) Renaming the 4 user/profile codes' constant `.name()` — out of scope; they're the wire `code` and Jakarta message string. (5) Leaving the 4 user/profile codes in `ProductErrorCode` (7-only split) — leaves user codes in a class named for products; the registry and per-enum tests cost the same for 11 as for 7.

**Factual vs inferred.** The split membership, the interface mechanism, the registry, the 44 renamed rows, the web one-literal touch, and the trust-boundary cleanliness are all factual (two read-only audits + the shipped sessions). The two-enum-vs-one-enum decision and the `displayName.*` → `user.display_name.*` reshape are Mastermind calls confirmed by Igor.

---

## 2026-05-29 — `expo-boot-redesign` shipped — gate state machine replaces `AppContext.bootstrap`

The Expo boot redesign ([`features/expo-boot-redesign.md`](features/expo-boot-redesign.md)) is shipped. Igor ran the full manual regression on a real device after the step 7d Phase 1 commit — cold-start scenarios, the boot-loop regression test, language/base-site switches, and dialog/chrome i18n — and all scenarios passed. The redesign is live in `oglasino-expo` on `new-expo-dev`. This entry closes the feature and folds in the seven spec amendments produced during it.

**Headline.** A gate state machine (`bootStore`) replaces `AppContext.bootstrap` as the sole boot authority and the sole source of truth for `selectedBaseSite` / `language` / `config` / `status`. Version-checksum freshness (Gate 4, via `/versions`) replaces the old 19–20-namespace cold-start burst — a returning user whose content is current fetches zero namespaces. A per-platform floor/ceiling version model drives Gate 2. A single Stack-level mount gate at `app/_layout.tsx` enforces "the portal mounts only when `ready`/`updating`" structurally, rather than per-screen.

**Deleted from the old path:** `AppContext.tsx`, `AppVersionConfigInit.tsx`, `RequireBaseSite.tsx`, `src/i18n/i18n.ts` (`initI18n`), `src/i18n/loader.ts`, `src/i18n/loadNamespaces.ts`, the `apiStore` `globalThis` barrier, the `_bootstrap: true` flag and its type augmentation, and all boot diagnostic `console.log`s. All 26 former `AppContext` consumers were re-pointed to `bootStore` selectors.

**Added:** `HardUpdateScreen.tsx` and `SoftUpdateModal.tsx` (extracted from the deleted `AppVersionConfigInit`, driven by `bootStore.latestVersion` and `softUpdate`), `softUpdateDismissal.ts` (the 24h-per-version dismissal memory ported verbatim — same AsyncStorage key, so existing dismissals carry forward), `MaintenancePollInit.tsx` (the active-checker poll that runs while in maintenance and detects maintenance-clear), and the `ensureBaseSiteOverviews` action (lazy fetch of the dialog's base-site list, cached for the app session).

**Loop-prevention invariants.** The three invariants — a single empty-deps mount effect, the machine writes `status` while views only read it, and re-entry destroys nothing — together with the structural defense that `bootStore` and its leaves import zero `authStore`/`chat-store`, all hold under live regression.

**The seven spec amendments** (all from this feature, folded here rather than logged separately):

1. **Part 3 seed posture.** "Seed data is for dev only" → corrected to a safe-default-all-envs seed (`0.0.0` floor / `0.0.0` ceiling, `ON CONFLICT DO NOTHING`). A dev-only seed would brick a fresh prod boot on a first-startup race; the safe-default seed makes Gate 2 a no-op until the real ceiling lands via the EAS post-build hook. (Source: backend B2 session.)
2. **Part 5 storage key shape.** `translations_<NS>` → `translations_<NS>_<lang>`. Per-language payloads need per-language keys. Implemented mobile-side; the `/versions` response is still per-NS-collapsed (per-NS-per-language is the follow-up feature).
3. **Part 4 namespace count.** 20 (mobile) vs 22 (backend). Two backend namespaces (`ADMIN_PAGES`, `BACKEND_TRANSLATIONS`) are deliberately omitted from the mobile enum because mobile never renders admin or server-side strings. The freshness gate drives staleness off the mobile enum, not the backend keyspace, so the difference is intentional.
4. **Part 8 step-6 test wording.** The original phrasing implied the gate registered only stale slices; the implementation registers the full in-memory map (cached-or-fresh) on every pass so i18n always holds the complete active-language bundle. Test wording reconciled to the "register the full in-memory map" logic.
5. **Part 4 / Part 8 step 6 — `changeLanguage`.** Gate 4's register step must call `await i18n.changeLanguage(activeLang)` after `init`/`addResourceBundle`. `addResourceBundle` registers bundles but does not switch the active language; without `changeLanguage`, a runtime language switch registers the new strings but the UI keeps rendering the old language.
6. **Part 1 portal mount predicate.** The predicate is `bootStatus ∈ {ready, updating}`, enforced structurally by one Stack-level conditional in `app/_layout.tsx` (not per-screen). `<RequireBaseSite>` (deleted in 7b) was the prior per-screen enforcement; the structural Stack gate (added in 7c) replaces it. `updating` is included so freshness/language/base-site transients (`ready → updating → ready`) do not unmount the portal. The Part 1 "no mount→status-write path" structural-defense claim is amended to its **convergent form**: the sole such path (the product deep-link base-site auto-switch at `product/[...productData].tsx:96-129`) is `ready`-guarded and idempotent once the base site matches, so it cannot drive a boot loop.
7. **Part 4 / Part 8 step 6 — chrome gating.** The "register the active language into i18n before `ready` so the portal never paints raw keys" invariant holds in steady state but has a one-frame react-i18next event-propagation lag at the `updating → ready` boundary. The opaque splash overlay covers the `updating` window, so the only visible artifact is the boundary frame. Portal header chrome (and any `selectedBaseSite`-keyed translation consumer inside the gated Stack) must therefore gate on `bootStatus === 'ready'`, not on `selectedBaseSite`, so first render reads already-initialized/already-switched i18n synchronously. Implemented at `app/(portal)/_layout.tsx:16` — one layout-level gate covers `CategoryNavigation`, `TopBar`, and `ConsumerProtectionBanner`.

(An earlier candidate amendment — claiming `pickBaseSite` transits `booting` — was **retracted mid-feature** when the source-of-truth read showed `pickBaseSite` writes no status. The seven above are the final set; the retracted one is intentionally excluded.)

**Pre-launch follow-ups carved out by the spec** (recorded so they are not lost):

- Wire the real Play Store / App Store deep-link URLs in `HardUpdateScreen` and `SoftUpdateModal`; the current value is the placeholder `https://memento-tech.com`. This must land before the production force-update flow has any real value.
- Smoke the EAS post-build ceiling-write hook on the next preview build cut (first live write to `/internal/app/version/ceiling`).

Both are tracked in `state.md` Risk Watch.

**Forward pointer.** [`features/version-checksums-per-language.md`](features/version-checksums-per-language.md) is the follow-up feature for per-NS-per-language checksums on `/versions`. It is backend work first, then a three-internal mobile swap already isolated in `bootFreshness.ts` for exactly this purpose. It is **blocked on backend** (the `/versions` response shape change), so it is not the immediate next Expo step.

**Next Expo step — Igor's pick (recorded here so the choice is visible):**

- **Option A — parked maintenance-poll decision.** The spec transition-table row "ready | active-checker detects maintenance on" is not wired: `MaintenancePollInit` runs its interval only while `status === 'maintenance'`, so it detects maintenance-clear but not maintenance-on-from-`ready`. Decide either to (a) implement runtime maintenance detection from `ready` (a second poll that runs while ready and calls `reEnter()` on maintenance-on), or (b) amend the spec transition table to say only maintenance-clear is detected. Small either way.
- **Option B — pre-launch carve-outs.** Wire the real store deep-link URLs (above). Small; required before the force-update flow is useful.
- **Option C — wait for backend.** Once backend lands the per-NS-per-language `/versions` shape, do the three-internal swap in `bootFreshness.ts` per the forward-pointer doc. Under an hour of engineer time; isolated for exactly this swap during step 6.

**Factual vs inferred.** The architecture, deletions, additions, the seven amendments, and the regression pass are factual — drawn from the `oglasino-expo` boot-redesign session summaries (1–14), the `oglasino-expo` post-pick consumers session, the backend boot-redesign sessions, and the feature audits (now archived to `sessions/`). The next-Expo-step list is the brief's framing of options for Igor, not a committed decision.

---

## 2026-05-28 — `markAsSeen` skipAuth ban reversed — auth is not load-bearing on `/seen`

The prior 2026-05-28 fix brief that wrapped `markAsSeen` in `after()` explicitly forbade `skipAuth: true`, reasoning that stripping the Bearer token would break server-side owner-exclusion identification. That reasoning is reversed. `skipAuth: true` is now applied to the `markAsSeen` call in `oglasino-web/src/lib/service/nextCalls/productService.ts`, and is in fact required for the call to dispatch at all from a Next 15+ `after()` callback.

Owner exclusion on the `/seen` path is enforced TWICE, neither dependent on auth on `/seen` itself:

- **Web gate.** `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` gates `markAsSeen` on `if (!owner.iamActive)`, excluding the owner before the call is scheduled. `owner.iamActive` became server-authoritative after the #8 auth-forward fix (decisions.md 2026-05-28 #8 entry) — `getUserForId` now forwards auth, so the backend computes `iamActive` against the real principal.
- **Backend independent guard.** `DefaultProductSeenService` excludes by comparing the product's stored `owner.id` to the authenticated principal at the entry of `/seen` processing. Crucially, the owner-id comparison reads from the product row, not from a Bearer claim; an anonymous `/seen` request still compares "product owner id" to "no principal" and proceeds with the increment, because no principal trivially is not the owner.

`/seen` therefore does not need viewer identity to enforce owner exclusion. The `skipAuth: true` path is safe on this `/public/*` fire-and-forget call, and is required because Next 15+ forbids `cookies()` reads inside an `after()` callback — the auth-forwarding hop at `fetchApi.ts:39` (which calls `await cookies()`) throws before the fetch dispatches, and the throw is swallowed by `markAsSeen`'s try/catch. Bypassing `cookies()` via `skipAuth` is what lets the call actually leave the Next process.

The reversal is scoped to the `markAsSeen` call only. It does NOT generalize to a blanket "skipAuth is fine on any `/public/*`" rule — every other `/public/*` consumer must still justify on its own merits whether viewer identity is needed downstream.

**Alternatives considered and rejected:**

- **Read `cookies()` outside `after()` and thread an explicit Bearer header through.** Rejected as more invasive than the underlying need — `markAsSeen` does not consume viewer identity, so threading auth through is preserving a posture the endpoint does not require, at the cost of touching the request helper and the call site.
- **Move the increment to a client-side `useEffect` that calls a server action.** Rejected — largest code change of the three, and shifts the counted population from "rendered RSC" to "client-script-executed," which is a semantic change to the view counter, not just a transport fix.
- **Leave the ban in place and drop `after()` entirely.** Rejected — un-`after()`ed fire-and-forget inside an RSC render is the original bug (Next 15/React 19 do not guarantee flush before the request lifecycle ends).

**Factual vs inferred.** The two-layer owner-exclusion mechanism is factual — confirmed across the iamActive auth-forward audit, the deep web audit, and the dedup-fix sessions. The "does not generalize to all `/public/*`" scope note is the chat's framing of intent, drawn from the narrowness of the change rather than from a separately audited rule.

---

## 2026-05-28 — Product view dedup — SETNX per-viewer-per-window replaces token-bucket; CF-Connecting-IP keying

The `/seen` endpoint's increment was gated by a token-bucket rate limiter (capacity 20, refill 1/sec), which never throttles a normal human refresh — every page reload counted as a fresh view, making the counter a self-spam vector. Replaced wholesale with a Redis `SET key NX PX <window-ms>` deduper keyed `view:<productId>:<viewerId>`: one increment per viewer per product per window. If the SETNX succeeds, the increment proceeds; if not, the call is a no-op.

**Window is config-driven.** New configuration key `redis.product.view.dedup.window.ms`, default `43200000` (12 hours). Tunable without redeploy via the existing admin configuration surface.

**Viewer-key IP derivation was fixed alongside the deduper — this was REQUIRED, not optional.** The viewer-id derivation now reads `CF-Connecting-IP` first (matching `RateLimitFilter.callerKey`'s existing convention), falling back to `request.getRemoteAddr()`. Behind Cloudflare the raw remote address is the worker's upstream peer, identical for every visitor — a deduper keyed on the raw remote address would collapse every web visitor into one viewer key per product, swinging from the original bug (every refresh counts) to the opposite bug (one view per product across all web users on stage and prod). CF-Connecting-IP is the real client IP and is what the rate-limit filter has been keying on for non-view-count traffic; this brings `/seen` in line.

**Known limitation: web sends no `X-Device-Id`, so the web key reduces to per-network (per client IP), not per-browser.** Two users on the same NAT count as one viewer. This is an acceptable undercount for a view counter — the goal of the dedup is to close the self-refresh spam vector, not to produce an exact-unique-viewer count. If `oglasino-expo` mobile adoption ships an `X-Device-Id` header (or if a future web enhancement sends a stable per-installation id), the deduper picks per-browser/per-install granularity automatically without code change.

**Cleanup folded in (same chat).** The now-dead rate-limiter machinery was deleted as part of this work: `RateLimiterService`, `RedisRateLimiterService`, the inline Lua refill script, and the two backing config rows (rows 14/15 in `data-configuration.sql`). Two mis-named view-count TTL config keys were renamed to the `redis.product.view.<thing>` convention — `*.rate.limiter.product.owner.ttl` → `redis.product.view.owner.ttl`, and `*.rate.limiter.delta.ttl` → `redis.product.view.delta.ttl`. The rename was verified safe by a read-only audit confirming no rate-limiter code reads these keys; they belong to the view-count path under their misleading names.

**Alternatives considered and rejected:**

- **Reconfigure the existing token bucket to capacity = 1.** Functionally a deduper, but harder to reason about (the rate-limiter framing implies "throttling" rather than "single increment per window") and keeps a misleading mechanism on the call path. Replacing with an explicit SETNX deduper makes the contract readable at the call site.
- **Ship the deduper without the CF-Connecting-IP fix.** Rejected — would have regressed from "self-spam" to "one view per product across all web users," a worse failure mode than the bug being fixed.
- **Send `X-Device-Id` from web in this same brief.** Rejected as cross-repo and out of scope. The deduper's per-network granularity is acceptable for v1; web `X-Device-Id` adoption is a separate future enhancement.

**Factual vs inferred.** The mechanism (SETNX + CF-Connecting-IP), the new config key and its default, the rate-limiter cleanup, and the TTL key renames are all factual — confirmed across `oglasino-backend-seen-dedup-1`, `oglasino-backend-viewcount-cleanup-1`, and the `oglasino-backend-viewcount-ttl-keys-audit-1` read-only audit. Stage confirmation is factual: Igor's two-viewer smoke — single viewer refreshing counts once, a second viewer on a different network counts again.

---

## 2026-05-28 — Φ3 scope expansion: boot/HTTP-layer refactor absorbed mid-execution

Φ3 (performance foundation) scope was expanded mid-execution to absorb a boot/HTTP-layer refactor, after manual smoke exposed a latent gap introduced by Φ2's always-mounted `<Stack>`: child routes (notably the product feed) mount before `AppContext` bootstrap sets the base-site/language codes, firing API calls with null `X-Base-Site`/`X-Lang` headers. The refactor delivered: (1) a codes-derived, `globalThis`-pinned bootstrap barrier in `apiStore.ts` replacing the prior convention-coupled promise (Brief 18); (2) a `<RequireBaseSite>` per-screen render gate preventing base-site-dependent screens from mounting before a base site exists, placed inside screen bodies to satisfy Φ2's no-conditional-navigator constraint (Briefs 19, 19a); (3) breaking the `api.ts ↔ authStore ↔ authService` require cycle by extracting interceptor-side auth effects into a registered hook (Brief 18). The chat-store require-cycle triangle (Cycle B) was deliberately deferred to chat B, where the chat-store reshape already lives; it is benign at runtime (all cross-calls dynamic via `getState()`).

---

## 2026-05-28 — Barrier-review lesson: verify the invariant on every resolve path, not just that resolution occurs

When reviewing a gating/barrier mechanism, verify that every resolve path establishes the gate's actual invariant, not merely that resolution always occurs. Brief 14 shipped a barrier that resolved on terminal _status_ (`ready`/`maintenance`/`select-base-site`); the `select-base-site` path reached a terminal status without setting codes, so the barrier released requests with null headers. Brief 15 corrected the invariant to "codes are set." Future barrier reviews must check invariant coverage per path.

---

## 2026-05-28 — Φ3 boot-loading issue UNRESOLVED — handed to a fresh Mastermind chat

As of session end (2026-05-28), Φ3 is code-complete but smoke-FAILING on an unresolved cold-start boot-loading issue. On first run / cleared storage, the app hangs on the loading spinner: bootstrap's `Promise.all([checkIfMaintenance, getAppConfiguration, fetchBaseSites])` does not resolve because two of the four boot-time requests — `GET /public/config` and `GET /public/baseSite/details` — never reach the backend, while `GET /public/maintenance/active` and `GET /public/app/version/android` consistently do. Established facts: (a) both failing endpoints return 200 quickly via curl from a laptop AND via the phone's own browser, ruling out backend, payload size, and network reachability; (b) the app's axios request interceptor passes all four requests through correctly (`_bootstrap: true` honored, barrier not blocking — confirmed via `[BOOT]` instrumentation); (c) backend is not under load; (d) the two failing endpoints are the two large-payload responses, the two succeeding are tiny; (e) the distinguishing variable is that the app fires four requests concurrently at cold start whereas the browser fetches one at a time. Leading hypothesis at handoff: a client-side concurrency / transport behavior specific to the four-parallel cold-start burst, NOT app bootstrap logic and NOT the boot refactor (the refactor's barrier, render gate, and cycle break all verified functioning). **A new Mastermind chat will own resolving this issue.** Diagnostic instrumentation (`console.log('[BOOT]', ...)`) is currently LIVE in `src/lib/config/api.ts`'s request interceptor and must be removed by the resolving chat as part of its fix.

---

## 2026-05-28 — 429-scoped synth reintroduced at the edge-worker boundary

Reversed part of the 2026-05-14 web session 4 removal of `ensureSystemErrorKey`. A narrowly scoped 429-only normalizer (`parseProductErrorsForStatus`) now synthesizes a `{field: null, code: 'RATE_LIMITED', translationKey: 'product.system.rate_limited'}` entry when the parsed validation result has no `SYSTEM_ERROR_KEY` entry. Applied at the six 429-handling sites in `oglasino-web/src/lib/service/reactCalls/productService.ts`. The shared `parseProductValidationErrors` util is unchanged — contract discipline holds for 400/422/403/500.

**Why reintroduce.** A 429 means exactly one thing (rate limited). The Cloudflare edge router worker (conventions Part 8, the edge boundary) can emit a 429 that does not pass through Spring's `RateLimitFilter` and therefore may not carry the unified body shape. Without the synth, the create wizard's cooldown gate (`BasicInfoProductDialog.onNextInternal`) silently no-ops — the user is stuck on step 1 with no inline reason and a Next button that keeps re-firing the same 429. The bug surfaced in `issues.md` 2026-05-14 ("Malformed 429 leaves create wizard silently stuck"); the post-Φ1 backend code path is well-formed but the edge boundary is not guaranteed to be.

**Why narrowly scoped.** The web session 4 rationale (trust the contract, don't paper over backend bugs) still holds for 400/422/403/500 — those statuses can mean many things and a synth would hide real bugs. 429 carries a single semantic. The synth is 429-only by code: the helper checks `status !== 429` and returns the parsed result unchanged. A `// NOTE:` block on the helper documents the scope and the edge-worker rationale so a future reader reads it as a deliberate, scoped guard rather than blanket defensiveness.

**Alternatives considered and rejected:**

- **Backend-only fix.** Rejected — the contract-emitting backend is already well-formed via `RateLimitFilter`; the gap is at the Cloudflare edge worker, which is a separate repo and a separate trust boundary. A client-side 429 guard is the right place to absorb edge-boundary shape drift.
- **Reintroduce the synth in `parseProductValidationErrors` itself, parameterized by status.** Rejected — the shared parser stays strict per the 2026-05-14 web session 4 contract-discipline rationale; the synth lives at the call sites that have status information.
- **Inline the 429 check at each of the six callsites.** Rejected per Part 4a — six identical defensive blocks earn the small helper; inline would force the 429-only contract to be re-derived six times by future readers.
- **Synthesize on 400/422/403/500 too.** Rejected — those statuses do not have a single unambiguous fallback translation key, and synthesizing would mask backend bugs that the strict contract is designed to surface.

---

## 2026-05-28 — Report-submit trust-boundary closed; per-target dedupe; new `ReportErrorCode` family

The report-submit endpoint (`POST /api/secure/report/add`) was the last unaudited Part 11 surface (`issues.md` 2026-05-16). The 2026-05-28 read-only audit (`sessions/2026-05-28-oglasino-backend-trust-boundary-audit-1.md`) confirmed a target-side violation: reporter identity was server-derived but `reportedUserId` / `reportedProductId` were taken verbatim from the client with no application-level validation, no self-report block, no participation check; `reportedProductId` had no FK at all. The 2026-05-28 fix session (`sessions/2026-05-28-oglasino-backend-report-trust-boundary-fix-1.md`) closes the gap with six coordinated changes:

1. **Application-side existence checks** before persisting. USER → `userRepository.findById(reportedUserId).orElseThrow(REPORTED_USER_NOT_FOUND)`; PRODUCT → `productRepository.getOwnerId(reportedProductId).orElseThrow(REPORTED_PRODUCT_NOT_FOUND)`. The lazy `entityManager.getReference` is gone; a non-existent target is now a coded 422, not an uncaught `DataIntegrityViolationException` 500.
2. **Self-report block.** USER: reject if `reporter == reportedUserId`. PRODUCT: reject if `product.owner.id == reporter` (`getOwnerId` is a single-column query, no full entity load). Code: `REPORT_SELF_NOT_ALLOWED`.
3. **Deletion-state guard.** Reject if the target user has `deletionStatus = PENDING_DELETION`. Code: `REPORTED_USER_PENDING_DELETION`.
4. **Per-target dedupe key.** Redis key changed from `report:user:<reporterId>` to `report:user:<reporterId>:target:<type>:<targetId>`. A reporter can now report distinct targets within 24h; the web's existing "already reported" 406 semantics become correct per-target. Cache TTL unchanged (24h).
5. **DTO validation + Part 7 codes-only.** `@Valid` on the controller, Jakarta `@NotNull` on `reportType` / `reportOption`, `@NotBlank` + `@Size(max=2000)` on `description`. All `IllegalStateException` throw sites replaced with a new typed `ReportValidationException` carrying a new `ReportErrorCode` family (10 constants mirroring `ProductErrorCode`'s `translationKey + httpStatus` shape). `GlobalExceptionHandler` gained a `handleReportValidation` arm and the Jakarta message-name lookup (`resolveTranslationKey`) now resolves against both `ProductErrorCode` and `ReportErrorCode`. Responses are now coded 400 / 422 / 406 — never 500 on a validation gap.
6. **Dead `if` deleted** at the old `DefaultReportService:34-37` (exact duplicate of `:26-28`, no functional effect).

**Trust-boundary verdict:** clean (per Part 11). Every client-supplied id is now resolved against authoritative server data or rejected with a coded 4xx.

**Wire-shape change for `oglasino-web`:** the existing 406 path still returns a body-less 406 response — no change. Failure responses on the create path now carry the unified `{errors:[{field, code, translationKey}]}` envelope (previously: 500 with a stack trace). The web side should map the new `REPORTED_USER_NOT_FOUND`, `REPORTED_PRODUCT_NOT_FOUND`, `REPORT_SELF_NOT_ALLOWED`, `REPORTED_USER_PENDING_DELETION`, and the four `REPORT_*_REQUIRED` / `_TOO_LONG` codes to translated UI copy. Tracked as a follow-up web brief.

**Alternatives considered and rejected:**

- Adding a participation check (e.g. "must have viewed the user's page" or "must share a conversation"). Rejected per the audit's recommendation — existence + self-report + dedupe closes the abuse vector at the entry point; a participation check adds friction without meaningful protection (an attacker can scroll a public profile too).
- Generalising `ProductValidationException` to accept any typed-code enum. Rejected as too invasive for this brief; the sibling pattern keeps surface-specific severity logic readable.
- `@AssertTrue` Bean Validation for cross-field target-id presence. Rejected — derives awkward wire field names (`reportedUserIdValid`) from validator method names. Service-level checks emit clean `field` codes.
- Lightweight user-existence-only projection instead of `findById`. Rejected — the brief specified `findById.orElseThrow(...)`, and the same row carries the deletion status the guard needs.

---

## 2026-05-28 — #8 isFollowingCurrent fixed via Fix A (drop `skipAuth` on `getUserForId` + portal `getProductDetails`)

`oglasino-web` removed `skipAuth: true` from `getUserForId` (`src/lib/service/nextCalls/userService.ts`) and from the `portal` branch of `getProductDetails` (`src/lib/service/nextCalls/productsSearchService.ts`). Auth is now forwarded on both fetches, so the backend can populate viewer-dependent fields (`isFollowingCurrent`, `iamActive`) authoritatively. A signed-in viewer who follows X now sees "Unfollow" correctly on both `/user/X` and any product page owned by X; one click unfollows as the user expects.

**Why Fix A over Fix B (drop the field from the public endpoint + small `/secure/follow/status` call) or Fix C (client-side `useFollowsStore`):** the prior `skipauth-footprint-audit` confirmed both consumer routes (`/user/[userId]`, `/product/[productId]/[productName]`) are already dynamic for unrelated reasons, so the route-level Full Route Cache cost of dropping `skipAuth` is zero. Only per-viewer Data Cache fragmentation increases on the `getUserForId` and `getProductDetails` fetches, which the 5-minute and 60-second revalidate windows mitigate. Fix A is the smallest code change (two-line removal in two services) and avoids the round-trip cost of B or the cross-cutting state of C. B and C remain principled fallbacks if per-viewer Data Cache fragmentation becomes a measured problem in production.

**Latent twin.** Without addressing the portal branch of `getProductDetails`, the product page's owner `FollowUserButton` would have remained broken. `ProductDetailsDTO` embeds `owner: UserInfoDTO`, and the same `skipAuth: true` was stripping viewer identity on the same shape. Fixed in the same brief to keep the two surfaces from drifting.

**Staleness fold-in.** `markFollowUser` previously did not invalidate the `user:${userId}` Data Cache tag, so even a correct seed went stale for up to 5 minutes after a follow / unfollow action. Added `await revalidateUserCache(userId)` on the success branch (reusing the existing Server Action that `DeleteAccountConfirmationDialog` and `AccountStateDialogsInit` already call). Three lines, no new mechanism.

**Adjacent fixes folded in:**

- Owner null-guard on `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:91`. `getUserForId` returns `UserInfoDTO | null`, but the file declared `owner: UserInfoDTO` and dereferenced `owner.iamActive` without a guard. Now `if (!owner) notFound();`, mirroring the existing `if (!baseSite) notFound();` pattern on the same page.
- `FollowUserButton` accessibility. Click handlers were on inner SVG icons rather than the `<button>` element; the `<button>` had no `type="button"`, no `onClick`, no `aria-label`. Keyboard activation didn't work. Moved the handler to the `<button>`, added `type="button"`, set `aria-label` to the same string the tooltip uses (`tButtons('follow.user.tooltip')` / `tButtons('unfollow.user.tooltip')`).

**Originating bug:** `issues.md` 2026-05-17 entry "`UserInfoDTO.isFollowingCurrent` seed is non-authoritative because `getUserForId` uses `skipAuth: true`". Two prior read-only audits informed the fix: `isfollowing-seed-audit-1` (recommended Fix B; Mastermind elected Fix A given the audit-2 cache-cost finding) and `skipauth-footprint-audit-1` (verified Fix A's cache cost is zero at the route level and bounded at the fetch level).

---

## 2026-05-27 — Product price threshold aligned to `>= 1` across create and update paths

Both `DefaultProductService.createProduct` and `DefaultProductService.updatePriceCurrency` now enforce identical rules for non-free-zone products: `price != null && price >= 1`. The threshold was previously inconsistent — create path rejected `<= 0`, update path rejected `< 1`. A product created at `0.50` was valid on create but could not be updated at the same price.

The create path's threshold was changed to match the update path's, not the other way around. Reasoning: rejecting `0 < price < 1` is the stricter, more defensible posture for a marketplace where prices below 1 (in the user's chosen currency) are unlikely to represent legitimate listings — most are likely typos, accidental zero-padding loss, or test data. Update-path symmetry was the dominant signal.

Comparison style also unified: both paths now use `getPrice().compareTo(BigDecimal.ONE) < 0` rather than the create path's previous `getPrice().doubleValue() <= 0`. Avoids the implicit lossy `BigDecimal → double` conversion.

If any product was historically created with `0 < price < 1` (web Zod check has been the practical gate, so this likely never happened in production), it cannot be updated at the same price under either the old or new rule — already stuck. The new rule does not introduce any new stuck state; it closes a small inconsistency window.

**Alternatives considered and rejected:**

- **Loosen the update path to `<= 0` instead.** Rejected — accepting `0 < price < 1` is harder to defend as legitimate marketplace data than rejecting it. The stricter posture is the correct default.
- **Make the threshold configurable.** Rejected per conventions Part 4a — single setting today, no foreseeable second setting, belongs as a constant.
- **Use a per-currency threshold** (e.g., `1 RSD` vs `1 EUR` are very different amounts). Rejected for v1 — adds complexity without clear product signal. If a per-currency minimum becomes a real product need, revisit.

**Originating bug:** `issues.md` 2026-05-23 entry "Create path missing `PRICE_REQUIRED` enforcement." The initial fix (commit `4c2c5e4`, 2026-05-25) added the guard with the inconsistent `<= 0` threshold. The 2026-05-27 alignment session (`oglasino-backend-create-path-test-coverage-1`) brought the threshold in line and added 16 unit tests covering the full create-path surface.

---

## 2026-05-27 — version-checksums feature shipped

Shipped per-namespace translation checksums and per-base-site catalog checksums via a single public endpoint `GET /api/public/versions`. Designed for mobile (Expo) to drive selective per-namespace refetch instead of polling the full translation set on every app open. Feature is code-complete on the backend; Expo adoption is the natural follow-up (tracked in `state.md` as Expo backlog row).

**Scope shipped — original feature (Briefs 1–6):**

- 22 translation namespace checksums + 3 base-site catalog checksums = 25 checksums total, persisted in `configuration` table, truncated SHA-256 (16 hex chars), content-derived from `key|value` rows (translations) and structured `CAT|/FIL|/OPT|/RNG|/ROP|` lines (catalog).
- New `VersionChecksumService` at `service/impl` owns checksum computation and Redis cache rebuilds for both versioned cache cohorts (`redisBaseSite::*`, `redisTranslations::*`).
- New `VersionController` exposing `GET /api/public/versions` with `Cache-Control: max-age=300, public, stale-while-revalidate=86400`. Public route under existing `permitAll()` matcher.
- Per-base-site cache split: `redisBaseSites` (single key holding `List<BaseSiteDTO>`) replaced by `redisBaseSite::<code>` per-key cache. `DefaultBaseSiteService.getAllBaseSites()` now composes from per-key reads via `@Lazy` self-injection (Part 13 blessed pattern, `DefaultBaseCurrencyService` precedent).
- Translation cache refactor: raw `StringRedisTemplate` + gzip + Base64 + JVM-synchronized rebuild path migrated to standard Spring `@Cacheable` with `Map<String, String>` value type. `GZipService` deleted (96 lines). 131 lines removed from `DefaultTranslationService`. `cache.translations.redis` toggle and `translations.version` counter deleted end-to-end.
- Soft replacement throughout: cache writes atomically overwrite existing keys (`cacheManager.getCache(name).put(key, value)`), never `DEL` first. Users always see populated cache during rebuilds.
- Boot ordering: `@Order(1)` config cache load → `@Order(2)` catalog sync → `@Order(3)` `VersionChecksumService.onAppReady` (base sites first, then translations; conditional rebuild — only when checksum changed or Redis empty) → `@Order(10)` `CacheWarmupService` (non-versioned caches) + readiness flip.
- Async admin path: `VersionChecksumService.rebuildTranslationCacheAsync` (`@Async`) called from `DefaultTranslationService.updateTranslation`. Admin edit returns at transaction commit; Redis updates on background thread within ~1-2 seconds.
- `REFERENCE_DATA_CACHE` constant extracted to `ReferenceDataCacheControl.INSTANCE` (fourth caller — `VersionController` joined three existing controllers).

**Scope shipped — `/baseSite/details` removal (Brief 7):**

Web's two usages of `/api/public/baseSite/details` were trivially removable:

- `getAllBaseSites()` in `app/actions/getBaseSiteServer.ts` was dead code (zero importers) — deleted.
- `sitemap.ts` only read `code`, `allowedLanguages`, `defaultLanguage` from the heavier endpoint — migrated to `/api/public/baseSite/overviews` (lighter `BaseSiteOverviewDTO` carries all three fields with identical types).
- Backend endpoint `/api/public/baseSite/details` kept in place; Expo adoption may use it.

**Scope shipped — admin translation page UX upgrade (Briefs 8–9):**

- Lazy load on `/admin/translations`: no fetch on page mount; gate on `hasFilters = !!params.namespace || !!params.key || !!params.value`. Empty state shows helper message ("Select a namespace or type to search").
- Pre-fill new-value textarea in `AdminConfigChangeDialog` (was blank, forcing full retyping for any edit).
- No-op early return in `AdminConfigChangeDialog.onApplyInternal` — fixed missing `return` after `onClose()` that fired update requests for unchanged values.
- Toast on save success / save failure replaces inline error display in the dialog. Both `TranslationsTable` and `ConfigTable` adopted.
- Result count display above translation table.
- Refresh translation cache button relocated from `/admin/translations` to `/admin/cache` (later subsumed by Brief 11's unified cache page).
- 13 new translation keys seeded (Brief 9) across all 4 locales (EN final, RS/RU/CNR Mastermind-drafted placeholders pending native-translator review at feature close — same posture as 2026-05-21 Consent Mode v2 and 2026-05-19 User Deletion).

**Scope shipped — unified admin cache page (Briefs 10–13):**

- New `AdminCacheDescriptor` enum lists all 8 admin-managed caches with `warmupSupported` flag. Authoritative source for both backend routing and frontend rendering.
- New backend endpoints: `GET /api/secure/admin/cache/list`, `POST /api/secure/admin/cache/{cacheName}/clear`, `POST /api/secure/admin/cache/{cacheName}/warmup`. Warmup endpoint routes internally — `CacheWarmupService` for the 4 non-versioned caches, `VersionChecksumService` for the 2 versioned caches.
- `CacheWarmupService.warmup()` split into four per-cache methods enabling per-cache admin warmup.
- Web `/admin/cache` page restructured: single unified list of all 8 caches, per-row buttons (Obriši for all, additional Obriši + Zagrej for the 6 warmup-supported caches). Bulk operations now orchestrated client-side via `Promise.all` over per-cache endpoints. Standalone "Keš prevoda" section deleted; `redisTranslations` joins the unified list. `RefreshTranslationCacheButton` file deleted.
- 5 old admin cache endpoints (`GET /admin/cache`, `POST /admin/cache/evict/{name}`, `POST /admin/cache/evict`, `POST /admin/cache/warmup`, `POST /admin/cache/refresh`) deleted from `CacheAdminController` after web migration (Brief 13).
- 3 new translation keys seeded (Brief 12) across all 4 locales for the new button labels and toast messages. Same placeholder posture as Brief 9.

**Decisions worth capturing:**

- **Soft replacement over evict-then-rebuild.** Versioned caches never go through `@CacheEvict` for content updates. The rebuild path writes the new value, atomically replacing the old. Users never see empty cache during rebuilds. Trade-off: a brief window of "old value visible" during async rebuild after admin edit, which is acceptable for translation content.

- **Cache ownership split: `CacheWarmupService` owns non-versioned caches, `VersionChecksumService` owns versioned caches.** Each service has different rebuild semantics — `CacheWarmupService` warms unconditionally at boot, `VersionChecksumService` skips when checksum matches AND Redis is populated. The admin cache page abstracts this difference behind a unified API (`AdminCacheDescriptor`); frontend doesn't know which service owns which cache.

- **Per-base-site cache split (Path 1).** `redisBaseSites` single-key cache replaced by per-key `redisBaseSite::<code>`. `getAllBaseSites()` composes from per-key reads. Caller refactors deferred — most callers consume the full list and continue to work unchanged. If a future feature wants direct per-code access without composing the list, it can call `BaseSiteService.getBaseSiteByCode(code)` directly. The `loadBaseSiteByCodeFromDb` method (cache-bypassing DB read) was extracted from `getBaseSiteByCode` to support the rebuild path.

- **Translation cache: `Map<String, String>` as cache value type, `List<TranslationDTO>` preserved on the wire.** The cache-natural shape is the map; the wire-natural shape (for backwards compatibility with `oglasino-web`) is the list of DTOs. Controller adapts inline. Web is not touched.

- **Brief 4 to Brief 6 midway state with synchronous `@CacheEvict`.** Brief 4 used synchronous `@CacheEvict` in `updateTranslation` as a temporary state; Brief 6 replaced it with async `rebuildTranslationCacheAsync` once `@EnableAsync` infrastructure was in place. Acceptable cost: brief user-visible latency window on admin edits during the Brief 4→6 interval on `dev` only.

- **Circular dependency `DefaultTranslationService` ↔ `VersionChecksumService` resolved with `@Lazy`.** Field injection in `DefaultTranslationService` combined with constructor injection in `VersionChecksumService` and `TranslationService` interface usage created a cycle Spring detected at boot. Resolved by adding `@Lazy` to the `VersionChecksumService` field in `DefaultTranslationService`. Recorded in `issues.md` for future reference if `DefaultTranslationService` migrates to constructor injection.

**Process learnings:**

- **Mid-feature scope expansion requires immediate spec amendment, not deferred-to-close.** The feature underwent two scope expansions in chat (`/baseSite/details` removal, then admin translation UX, then unified admin cache page). Spec amendments were appended in chat but `§11`'s brief order list wasn't kept current — the web engineer on Brief 11 paused implementation correctly because the spec didn't list Briefs 9-11. A small Docs/QA session amended `§11` mid-chat. Carry forward: any further scope expansion gets a spec amendment BEFORE the engineer brief, not after.

- **`oglasino-web` uses `stage` branch, not `dev`.** Mastermind wrote `dev` in every web brief in this chat; engineer correctly stayed on `stage` per the hard rule (don't switch branches) and flagged on every brief. Carry forward: future web briefs say `stage`. Backend stays `dev`.

  - **Correction 2026-06-01 (supersedes the carry-forward above for future briefs).** `oglasino-web` now works on `dev`. The entire create + update product-parity feature ran on web `dev` (all four web session summaries declare `Branch: dev`), and Igor has confirmed `dev` is the current web branch. **Future web briefs say `dev`, not `stage`.** The dated `stage` records in earlier entries (review-reports and error-code-domain-split, both 2026-05-29; expo-maintenance-split) remain accurate as history — web was on `stage` then; this corrects only the forward-looking rule, it does not rewrite where past work landed.

- **RU translations use Latin transliteration with `''` (SQL-escaped apostrophe) for soft signs in infinitive verb forms.** Brief 9 engineer flagged Mastermind's RU drafts were missing this convention. Carry forward: future Mastermind RU drafts include `''` in transliterated infinitives (e.g., `sokhranit''`, `obnovit''`, `udalos''`). Brief 12 RU drafts already applied this.

- **Backend feature work in this chat had no `oglasino-docs/sessions/` session-summary files.** Engineers wrote summaries to `oglasino-backend/.agent/` (correct per Part 5) but Mastermind didn't keep canonical `oglasino-docs` summaries. The web engineer surfaced this as a gap when looking up Brief 10's existence. Carry forward: confirm canonical recording mechanism — either `oglasino-docs/sessions/` should be populated as work lands, or the convention is to rely on per-repo `.agent/` folders. Decide and document in `conventions.md` if needed.

**What's not yet done (Expo adoption):**

The `oglasino-expo` mobile app does not yet consume `/api/public/versions`. The whole feature was built FOR this consumer; current mobile still polls translations on app open (or whatever the current behavior is). Expo adoption is a future feature with its own Mastermind chat:

- Phase 1: how does Expo currently fetch translations and base-site data?
- Phase 2-3: design the polling cadence, the per-namespace refetch trigger, the cache invalidation strategy on the mobile side.
- Phase 5: implement.

Tracked as `oglasino-expo-version-checksums-1` (or similar — slug convention per state.md) in the Expo backlog.

---

## 2026-05-27 — Expo-router `<Stack>` cannot be conditionally rendered; Φ2 (Expo navigation foundation) shipped

Φ2 replaces every bare `<Slot />` in `oglasino-expo` with native `<Stack>` and `<Tabs>` navigators per spec [`features/expo-navigation-foundation.md`](features/expo-navigation-foundation.md). All six layouts use native navigators. BottomBar accepts `BottomTabBarProps` and reads active tab from navigation state. DashboardSidebar uses `router.replace()`. Four `Dimensions.get('window')` sites migrated to `useWindowDimensions()`. `+not-found.tsx` renders the custom TopBar pattern. All §7 definition-of-done items pass: tsc clean, lint 0 errors / 80 warnings (Φ1 baseline), 109 tests passing, expo-doctor unchanged.

**The expo-router constraint discovered during manual smoke.** Conditionally rendering a `<Stack>` causes expo-router to reinitialize the navigation container on every mount, which remounts the parent layout, which resets parent state, which can produce an infinite loop. Bare `<Slot />` doesn't register a navigator, so the conditional-rendering pattern was safe with the pre-Φ2 shape. Switching to `<Stack>` exposed the incompatibility.

The fix: always-mount `<Stack>` and render the non-`ready` states (loading, maintenance, select-base-site) as absolute-positioned overlays on top of it. `AppInit` mounts only when `status === 'ready'`. The portal layout guards its chrome (TopBar, CategoryNavigation, ConsumerProtectionBanner) on `selectedBaseSite` being defined, because the always-mounted Stack renders the portal layout before bootstrap completes.

A secondary bug in `AppContext.tsx` was fixed in the same session — the main effect had `state.status` in its dependency array, which caused a double-bootstrap pattern. `state.status` removed from deps; a `statusRef` was added so the polling interval can read current status without re-triggering the effect. This bug pre-existed Φ2; the navigator remount loop made it visible by amplifying the symptom.

**Constraint this establishes.** Any future navigator (`<Stack>` or `<Tabs>`) in this codebase must be always-mounted, not conditionally rendered. If a navigator's content depends on state that may not be ready yet, gate the navigator's _children_ on that state — or render an overlay on top of the always-mounted navigator. This sits alongside the 2026-05-25 route-group-layout-files constraint as the second expo-router-specific gotcha worth carrying forward.

**Process notes worth recording, since this chat made the mistakes that surfaced these constraints:**

- The Mastermind chat invented a "divergence" in `notifications.tsx`'s auth guard placement and instructed an engineer fix. Lint caught it (6 React-hooks-rule violations). The patch was reverted in the same chat. Lesson: when an audit reports clean, the next step is smoke, not "let me clean up the audit's notes first." The audit's three flagged items were intentional shapes, not deviations.
- The Mastermind chat then authorized a fix to `AppContext.tsx` (a Φ1-era file untouched by Φ2) based on a single engineer's investigation, without first asking whether the symptom pre-existed the chat. The fix was correct on its own merits (the effect-dep bug was real) but didn't solve the visible symptom, because the symptom was driven by the separate navigator remount loop in `_layout.tsx`. Lesson: when a smoke regression surfaces, the first question is "did this exist before this chat?" Establishing the baseline first prevents wasted detours.
- The actual root cause was found by an engineer who added diagnostic `console.log` lines and read the Metro output. Code-reading + runtime instrumentation beat theorizing. The investigation pattern (read-only diagnostic instrumentation, cleanup at end of session) is worth using earlier next time.

**Alternatives considered and rejected:**

- **Wrap the conditional Stack in a `display: 'none'` container while keeping the conditional structure.** Rejected — React Native still runs render functions for hidden children, so route components would still crash accessing undefined `AppContext` data. Plus the navigator container reinitialization is driven by mount/unmount, not by visibility.
- **Roll back to `<Slot />` and re-evaluate Φ2.** Rejected — `<Slot />` is the structural defect Φ2 exists to fix (audit finding F12). The fix is to make Stack always-mounted, not to abandon Stack.
- **Add the `selectedBaseSite` guard inside each chrome component instead of at the portal layout level.** Rejected per Part 4a — three callers (TopBar, CategoryNavigation, ConsumerProtectionBanner) would each carry the same guard; one guard at the layout level covers all three.
- **Investigate the AppContext effect-dep bug as a separate bug-fixer chat per conventions.** Considered but rejected — the bug was discovered during Φ2 smoke and the navigator remount loop required touching `_layout.tsx` and `(portal)/_layout.tsx` anyway, so folding the small `AppContext.tsx` change into the same session was cheaper than opening a parallel chat. Documented here so the precedent is clear: Φ2 closed with one Φ1-era file change, attributable to symptom amplification by Φ2 work.

**Effect on downstream chats:** Φ3 (performance foundation) opens next per the program spec. Φ3's intake should reference this entry — the always-mounted Stack pattern means tab persistence works as intended (no remount on status changes), which is the foundation Φ3's re-render math sits on top of.

**Factual vs inferred.** All findings, fixes, and verifications above are factual (produced by the smoke session and the engineer's diagnostic instrumentation). The §11 spec note about the original conditional-rendering assumption being inferred is factual — the spec was drafted before runtime verification was possible.

---

## 2026-05-27 — Theme cookie persistence shipped (replaces next-themes)

Replaced `next-themes` with a minimal cookie-based theme system in `oglasino-web`. The system mirrors the established card-size cookie pattern (Zustand store + consent-gated cookie writes + in-memory-wins-on-grant reconciliation + SSR pre-paint snippet). Shipped across two engineer briefs (Brief 1 and Brief 1b) on `feature/theme-cookie-persistence` 2026-05-27. Closes the deferred theme portion of Item 4 from the cookies-closing chat (2026-05-22, `decisions.md`).

**What shipped:**

- New Zustand store `useThemeStore` (`src/lib/store/useTheme.ts`) exposing `theme: 'light' | 'dark' | 'system'` and `resolvedTheme: 'light' | 'dark'`. `setTheme` updates in-memory state unconditionally, applies the `.dark` class on `<html>` synchronously, and writes to `globalCookie.theme` only when preference consent is granted. Mirrors `useCardSizeStore.setSize` exactly.
- SSR pre-paint `<script>` in `<head>` (`src/lib/theme/ssr.ts`, wired at `app/layout.tsx`) that reads `globalCookie.theme` server-side and sets `.dark` on `<html>` before paint. Snippet is idempotent — `classList.add('dark')` is a no-op when the class is already present.
- Server-side class rendering on `<html>` for concrete cookie values (`'dark'` cookie → `<html class="... dark ...">` from SSR). Eliminates hydration mismatch on cookie-concrete branches.
- `suppressHydrationWarning` on `<html>` to handle the `'system'` and absent-cookie branches where the server cannot resolve `prefers-color-scheme`. This was initially dropped in Brief 1 on incorrect reasoning, restored in Brief 1b after manual smoke surfaced a hydration warning.
- `SyncThemeFromCookie` — reconciliation on consent-grant transition. In-memory wins when non-default; cookie wins when in-memory is default (`'system'`).
- `SyncThemeFromSystem` — OS `prefers-color-scheme` listener active only when `theme === 'system'`.
- Tri-state segmented toggle (Sun/Monitor/Moon) in `PortalConfigDialog`. `WithTooltip` and `IconButton` wrappers dropped — replaced with plain `<button>` elements (engineer's Part 4a simplification on top of the brief's spec).
- Three consumer migrations (`OglasinoIcon`, `sonner`, `ToggleButton`).
- `next-themes` uninstalled; `ThemeProvider.tsx` deleted.
- `GlobalCookie.theme` widened from `'light' | 'dark'` to `'light' | 'dark' | 'system'`. Default when absent: `'system'` (resolved client-side via `prefers-color-scheme`).

**Fixed in passing:** `OglasinoIcon`'s stale `useEffect` deps (audit observation 2 — logo now updates correctly on OS dark-mode toggle when `theme === 'system'`); `ToggleButton`'s `mounted`-guard (audit observation 5 — no longer needed since SSR renders concrete classes and `suppressHydrationWarning` covers the system branch); `disableTransitionOnChange` prop (audit observation 1 — was a no-op, no theme-specific CSS transitions exist).

**Mastermind analysis error worth recording.** The spec's original §9 risk paragraph framed `<html>` as outside React's hydration tree and concluded `suppressHydrationWarning` was unnecessary. In Next.js App Router, `<html>` is rendered by React as `app/layout.tsx`'s root element. Brief 1 followed the (incorrect) spec and dropped the attribute; manual smoke caught the resulting hydration warning; Brief 1b restored the attribute and added server-side class rendering for cookie-concrete branches. The system worked correctly — spec is fallible, smoke is the safety net, fix landed in two lines of code. Recording so a future Mastermind chat doesn't repeat the analysis: any element rendered by React in App Router (including `<html>` and `<body>`) is in the hydration tree.

**Alternatives considered and rejected:**

- **B1-A (next-themes custom storage adapter).** Unavailable in v0.4.6 — `ThemeProviderProps` exposes no pluggable storage mechanism. Confirmed by the read-only audit.
- **B1-B (next-themes mirror pattern).** `setTheme` writes to localStorage unconditionally; no way to gate on consent. Confirmed by the read-only audit.
- **Keeping `disableTransitionOnChange`.** No theme-specific CSS transitions exist; the prop was a no-op.
- **Replacing `suppressHydrationWarning` with a wrapper component.** Rejected per Part 4a — the attribute is the correct React mechanism for this exact use case (server can't know `prefers-color-scheme`); any wrapper would hide the attribute behind unnecessary abstraction.
- **Adding `'system'` later as a separate UI pass.** Rejected during Phase 1 — Path A's replacement is the natural moment to fold it in; revisiting the toggle UI and reconciliation logic twice was the worse trade.
- **Splitting Brief 1 into 1a (foundation) and 1b (UI + consumers).** Rejected — the engineer could see the whole picture and the changes interlock cleanly. Brief 1b (the actual second brief) is a separate hydration fix, not a foundation/UI split.

**Tasks remaining (post-launch):** native-translator review of the new toggle labels is not needed — the segmented control uses icons only, no text. No further work queued for this feature.

---

## 2026-05-26 — Expo cloud setup rebuilt; per-tier three-profile structure shipped

The previous EAS cloud project (`cbc496aa-bbd8-4c8e-b46e-c4c70a61f867`) had accumulated invalid credentials, broken builds, and a half-empty `.env.development` (four Firebase config values never populated, meaning the mobile app had never successfully authenticated against Firebase end-to-end on a real device). Replaced wholesale.

New cloud project `@oglasino/oglasino-expo` (ID `382cac59-45fa-420e-ab80-7fc709e6d2f3`) created in the `oglasino` org with a clean three-tier build profile structure (`development` / `preview` / `production`) corresponding to per-tier bundle IDs (`com.oglasino.development` / `com.oglasino.preview` / `com.oglasino`) and per-tier Firebase Apps. The three-tier model is reversibly extensible to additional tiers (e.g., per-feature internal testing) by registering a new bundle ID + Firebase App pair and adding a fourth ternary branch.

Bundle IDs are independent in Apple Developer and Google Play; all three tiers can co-exist on one device. Each tier has an independent Firebase App for analytics segregation and an independent provisioning profile / signing identity. The decision to make bundle IDs per-tier (versus reusing `com.oglasino` across all three tiers as the previous setup did) is the user-experience reversal from the earlier "one bundle ID across tiers" Firebase setup. The cost of the reversal was ~90 minutes of Firebase + Apple console work; the benefit is permanent — no need to uninstall one tier to install another for comparison testing.

EAS Workflows configured for auto-build on push: `preview` branch → preview profile (both platforms in parallel); `main` branch → production profile (both platforms in parallel). Development tier is manual-only via local `eas build`. Path filters scoped to `app/`, `src/`, `assets/`, `app.config.ts`, `package.json`, `package-lock.json`, `eas.json`, and the tier-specific `.env` file — doc-only pushes don't waste build minutes.

Five `app.config.ts` ternaries branch all per-tier configuration on `APP_ENV`: bundle ID/package, iOS Google service file, Android Google service file, display name (`Oglasino Dev` / `Oglasino Preview` / `Oglasino`), URL scheme (`oglasino-dev://` / `oglasino-preview://` / `oglasino://`). Adding a sixth tier-specific value (e.g., per-tier icon, per-tier splash color) would extend the same pattern; if the file ever needs a sixth ternary, an engineer should evaluate consolidating to a typed `envValues` map.

Firebase initialization in `src/lib/client/firebaseClient.ts` reads six `EXPO_PUBLIC_FIREBASE_*` env vars at module-evaluation time, validates none are missing (throws clearly naming any absent vars), and calls `initializeApp()` from the JS Firebase SDK. Defensive throwing prevents the historically-common opaque-failure pattern where Firebase initializes with `undefined` config and fails cryptically later.

The Android keystore is per-profile, EAS-managed; the development tier's keystore was generated during the first dev build. SHA-1 and SHA-256 fingerprints extracted via `eas credentials → Download existing keystore` + `keytool -list -v` and registered against `com.oglasino.development` in `oglasino-stage-49abb`. Keystore backed up to `oglasino-private/01-secrets/oglasino-infra/oglasino-expo-android-keystore.jks` with password. EAS Cloud is the canonical store; local backup is recovery insurance.

First development build on EAS Cloud succeeded on both iOS and Android. iOS validated end-to-end on a physical iPhone: build install → Developer Mode enable → dev client connection to Metro → Firebase email/password login → backend round-trip authenticated. Android email/password login also validated. Google Sign-In on iOS not yet tested (deferred); Google Sign-In on Android requires final rebuild to bake the updated google-services file into the APK.

Two new docs deliverables capture the as-built state and forward-looking deploy gates:

- [infra/expo/cloud-setup.md](infra/expo/cloud-setup.md) — canonical state spec describing current cloud, build, and code-level architecture
- [infra/expo/pre-deploy-checklist.md](infra/expo/pre-deploy-checklist.md) — forward-looking checklist of items needed before preview / production tiers can ship to testers or users (per-tier keystore extraction, APNs key, Play Console listing, etc.)

**Alternatives considered and rejected:**

- **Keep the previous single-bundle-ID setup** (`com.oglasino` across all three tiers, distinguished only by Firebase project). Rejected — the side-by-side-install constraint became a real friction point during the rebuild discussion. The user-experience reversal cost was acceptable; the long-term benefit of independent tier identities is permanent.
- **Use `com.oglasino.dev` instead of `com.oglasino.development` for the development tier bundle ID.** Rejected — `.dev` was the previous-iteration bundle ID, and reusing the suffix would have created ambiguity with the older Firebase Apps. `.development` matches the EAS profile name and removes any confusion.
- **Two tiers instead of three** (development = preview, no separate preview tier). Rejected — the preview tier exists for QA testing without a dev client (production-shaped bundle, no Expo dev menu, no Metro dependency), which is a distinct purpose from the development tier's hot-reload workflow.
- **Commit Google service files as gitignored, manage via EAS Secrets.** Rejected — Firebase documentation explicitly notes these are not secret (they're embedded in every compiled APK/IPA). Maintaining EAS Secrets for non-secret files adds tooling complexity for no security benefit.
- **Use EAS Updates / OTA channels in this work.** Rejected — out of scope; OTA updates are independent of cloud setup. Future work.
- **Set up push notifications (APNs key + FCM service account) in this work.** Rejected — push setup is its own substantial workstream and was deferred to a separate future chat. APNs key generation, .p8 upload to Firebase, FCM service account creation, native push token registration testing across iOS and Android each warrant focused attention.
- **Run a final Android + iOS rebuild before closing the chat.** Deferred at Igor's request — final rebuild will land after the chat closes to consolidate all post-first-build code changes (firebaseClient.ts hardening, per-tier name/scheme, updated google-services.development.json) into one rebuild. Tracked in `state.md` Risk Watch.

The Expo cloud setup is `shipped` for the development tier; preview and production tiers will reach `shipped` when their first builds succeed and the items in the pre-deploy checklist are addressed.

---

## 2026-05-25 — Free-zone OG image semantic correction shipped (SEO deferred-batch Item A)

The free-zone page's structured-data `image` field and `openGraph.images` / `twitter.images` previously fell back to brand-mark defaults — the structured-data field to `basicMetadata.appLogo` via the `basicMetadata.defaultOgImageUrl || basicMetadata.appLogo` chain at `generateFreeZonePageStructuredData.ts:39`, and the metadata generator's OG/Twitter images to `basicMetadata.defaultOgImageUrl` (a generic site-wide OG asset). All three surfaces now point at a new page-specific hero asset at `public/free-zone-hero.png` — a 1500×1000 raster export of the existing `GivingBackBigIcon` SVG on background `#4b6769`. Asset produced manually by Igor via a DevTools console export script. Two files touched in `oglasino-web` (`generateFreeZonePageStructuredData.ts`, `generateFreeZonePageMetadata.ts`); the structured-data fix is a one-line URL swap, the metadata-generator fix updates `openGraph.images` and `twitter.images` to the same URL with actual asset dimensions (1500×1000) and page-title alt text. 229 web tests passing, no new lint warnings, `tsc --noEmit` clean. Verified via curl on `/rs-sr/blog/free-zone` that JSON-LD `image`, `og:image`, `og:image:width`, `og:image:height`, `og:image:alt`, and `twitter:image` all serve the new asset.

**Origin.** Item A of the three-item SEO deferred batch carried forward from the SEO foundation feature close (2026-05-24, `decisions.md`). Handoff at `.agent/handoffs/seo-deferred-batch.md`. The structured-data fix was the named target in the handoff; the engineer correctly extended scope to `openGraph.images` and `twitter.images` after finding the same semantic problem (generic default OG image, not the logo specifically as the handoff predicted) on the adjacent metadata surfaces.

**Asset notes.** Native SVG viewBox is 750×500 (3:2). PNG exported at 2× native (1500×1000) on solid `#4b6769` background. Transparent background was considered and rejected because OG image renders against unpredictable social-platform backgrounds (light or dark). 1200×630 OG-standard ratio was considered and rejected in favor of preserving the icon's native proportions — modern crawlers tolerate non-standard ratios, and crop-to-fit on the social-platform side preserves the asset on click-through.

**Alternatives considered and rejected.**

- **Omit the `image` field entirely.** Rejected — the asset exists, and a working OG link preview on Facebook / Twitter / LinkedIn is worth the five-minute fix on a page with reasonable share-intent (free-zone is a higher-engagement surface than generic catalog pages).
- **Use the SVG file directly.** Rejected — Schema.org `image` and `og:image` consumers handle SVG inconsistently; PNG is the safe format. Asset is one-time work, not maintenance burden.
- **Hardcode the asset URL instead of using `${basicMetadata.baseUrl}`.** Rejected — `basicMetadata.baseUrl` is the established pattern for every other asset URL in the metadata generators; matching the pattern is one of the Part 4a simplicity guidelines (match the surrounding code's style).
- **Extract a `freeZoneHeroImageUrl` constant or add it to `basicMetadata`.** Rejected per Part 4a — one consumer, no reuse foreseeable; inline template literal earns its placement.

**Status of deferred-batch siblings.** Items B (intro page hreflang cluster splitting) and C (per-category authored SEO descriptions) remain deferred. Item B is parked pending post-launch Search Console data showing actual cannibalization or canonical-selection issues on the all-locales intro cluster — speculative engineering pre-data was explicitly rejected. Item C is parked pending Igor's content-authoring decision on tier (Tier 1 = 14 top-level categories × 4 locales = 56 rows, Tier 2 = ~80 categories, Tier 3 = full tree ~700 categories). Neither item has open engineering work.

---

## 2026-05-25 — Image alt text translation shipped

Four translation keys seeded in the `METADATA` namespace across EN / RS / RU / CNR (16 seed rows), five code sites swapped in `oglasino-web`. Code-complete across two backend sessions and one web session on `feature/image-alt-translation`, preceded by three read-only audits (web audit 1, web base-site audit, backend category-catalog audit).

**What shipped.** Four keys (`intro.image.alt`, `about.hero.image.alt`, `intro.og.image.alt`, `intro.meta.description`) seeded across four locales. Five web sites swapped: two intro page `<img alt>` sites sharing one key (`app/page.tsx` lines 29 and 38), one about page hero `<img alt>` (`about/page.tsx` line 41), one intro og:image alt (`generateIntroPageMetadata.ts` line 53), one intro page meta description (`generateIntroPageMetadata.ts` line 17) that cascades to three metadata surfaces (`metadata.description`, `openGraph.description`, `twitter.description`). Backend IDs: EN 3269-3272, RS 5369-5372, RU 7469-7472, CNR 1169-1172.

**The base-site question.** The web base-site audit confirmed the translation schema has no base-site dimension (`Translation` entity has no `base_site_id` column; `TranslationsController` takes only `namespace` and `lang`; backend ignores the `X-Base-Site` header on the translations endpoint). All four images are base-site-neutral: the intro page is outside the `[locale]` segment and renders identically for all visitors; the about page has per-base-site URLs but identical content across base sites sharing a language. Spec kept keys language-only. If per-base-site translation is ever needed, the schema must be widened first — substantial backend + schema feature, separate work.

**The C2 fold-in.** The Serbian description `'Vaša platforma za kupovinu i prodaju'` on `generateIntroPageMetadata.ts:17` was flagged by web audit 1 as finding C2, confirmed base-site-neutral by the base-site audit, and folded into scope after the backend category-catalog audit determined the English meta description value. Three-key spec → four-key spec mid-feature; Phase 5 ran two backend sessions (one before amendment seeding three keys, one after seeding the fourth) plus one web session.

**The English meta description.** Final value: `Buy and sell on Oglasino — the marketplace for fashion, home, electronics, tools, hobbies, sports, and services. Free to post.` Drafted against the actual top-level categories in the `rs` and `me` catalogs (10 categories: Free Zone, Women, Men, Kids, Home, Electronics, Tools and Materials, Hobbies, Services, Health and Care). Length stays under 130 chars for Google truncation budget. Front-loads "Buy and sell on Oglasino" so brand + verb survive truncation.

**Process notes.** Two worth recording:

1. Phase 5 backend brief 1 drafted before the Phase 4 spec amendment landed — the backend engineer correctly caught the brief-vs-spec mismatch pre-implementation. The brief asked for four keys; spec authorized three. Spec wins per conventions Part 3, brief is the drift. Lesson: when a Mastermind chat decides to widen scope mid-feature, the spec amendment lands before the next engineering brief, not after.
2. Translation seed IDs reserved in backend session 1 were still available in session 2 — clean handoff via the prior session summary documenting the reserved slots (EN 3272, RS 5372, RU 7472, CNR 1172).

**Alternatives considered and rejected:**

- **Per-base-site translation variants.** Rejected — no schema dimension, no content variation across base sites for these images, would have required substantial backend + schema feature work.
- **Skipping the C2 description fold-in.** Rejected — same file as site 4, zero marginal cost to add, high impact since Serbian description was shipping to EN/RU/CNR search results.
- **Folding in the brand-name `'Oglasino'` title.** Rejected — intentional brand-only title on the locale-selector landing page.
- **Folding in the `© Memento Tech` copyright footer.** Rejected — universally non-translated pattern (`©` symbol + proper noun).

---

## 2026-05-25 — Expo-router route-group layout files collapse children into one navigator screen

Discovered during Φ2 Brief 1 implementation (`oglasino-expo`, root + portal navigators). A route group with a `_layout.tsx` becomes a single screen of its parent navigator, regardless of how many route files live inside. Children are not hoisted to the parent navigator's children list when a layout file is present.

Concretely: with `app/(portal)/(secured)/_layout.tsx` present, the portal `<Tabs>` saw two children (`(public)` and `(secured)`) instead of the four tab destinations (Home / Favorites / Notifications / Messages) required by the Φ2 spec. Verified from expo-router source `flattenDirectoryTreeToRoutes` in `getRoutesCore.js`: child hoisting to the parent navigator only happens when the group has no layout file.

**Resolution for Φ2:** the `(secured)` group has no `_layout.tsx`. The three screens carry inline auth guards (`useAuthStore((s) => s.user)` + `useAuthStore((s) => s._hasHydrated)` + `<Redirect href="/" />`) at the top of their function bodies, matching Φ1's F13 pattern. Functionally equivalent to a single layout guard, distributed across three sites. See `features/expo-navigation-foundation.md` Section 3.4 for the spec text.

**Constraint for future navigator work.** When a tab or stack navigator needs N children, the N route entries must be either (a) direct route files at the parent navigator's level, or (b) route groups with no `_layout.tsx`. If a group needs its own layout (for shared chrome, providers, or guards), that group becomes one navigator child — and that child must be intended as a single tab/stack screen, possibly containing its own nested navigator.

**Implication for Φ3+ and downstream navigator work:** any nested-navigator design in this codebase must verify against this constraint before drafting briefs. The auth-guard pattern can be either layout-based (one group, one shared guard, one tab/stack child) or screen-inline (N screens, N inline guards, N tab/stack children). The choice is determined by whether N screens need to be direct children of the parent navigator. Both shapes work; both keep F13 semantics intact (audit Section 5).

**Alternatives considered and rejected:**

- **Re-introducing `(secured)/_layout.tsx` with a `<Stack>` and centralizing the auth guard.** Rejected because it breaks the four-tab structure — the portal `<Tabs>` would see `(secured)` as one tab screen containing a Stack, not as three separate tab destinations. The Φ2 spec's four-tab pattern (Fork A — nested navigators) requires four direct children of the portal `<Tabs>`.
- **Keeping `<Slot />` in `(secured)/` instead of deleting the file entirely.** Rejected because expo-router treats any `_layout.tsx` (including bare `<Slot />`) as a layout boundary that collapses children. The presence of the file, not its contents, is what causes the collapse.
- **Restructuring the directory tree to move the three secured screens out of `(secured)/`.** Rejected as more disruptive than three inline guards. Preserves the directory's role as a logical grouping for auth-required screens without requiring a layout file.

---

## 2026-05-25 — Φ1 (Expo auth lifecycle foundation) shipped

The first of four Expo structural foundation chats (Φ1) closes today. All seven structural auth-lifecycle findings from the 2026-05-24 structural audit are addressed in `oglasino-expo`:

- **F1 — `initAuthListener` wired.** Listener subscribes to Firebase `onIdTokenChanged` from `AppInit`, gated on `_hasHydrated`. Module-scoped single-flight guard (`inFlightUid` + `lastSyncedUid` + 2-second `lastSyncedAt` drop window) mirrors web's `UseTokenRefresh` pattern. The listener handles `X-Account-Restored: true` (sets `restored: true`) and `USER_BANNED`/`EMAIL_BANNED` error responses (sign out + sets `accountBanned: true`).
- **F2 — `logoutFirebase` calls `auth.signOut()` before `GoogleSignin.signOut()`.** Firebase session now drains correctly on logout. Closes the session-leak risk (the old user's Bearer token no longer attaches to subsequent API requests).
- **F3 — Direct store cleanup on logout.** `authStore.logout()` now directly clears chat, favorites, notifications, portal filter, and dashboard filter stores. Component-driven cleanup remains as a secondary safeguard. **The catalog was deliberately excluded** — initially via Brief 2-bis on the grounds that catalog is not user-scoped, then conclusively via the dead-code cleanup brief (see below) which removed the catalog store entirely.
- **F4 — Foreground re-validation.** New `ForegroundRevalidationInit` component subscribes to React Native's `AppState`. On `active` after >5 minutes of `backgrounded`/`inactive` state, calls `syncUserToBackend` if a user is signed in. Threshold is a file-scoped constant (`FOREGROUND_REVALIDATION_THRESHOLD_MS = 5 * 60 * 1000`), locked per Phase 1 decision. Response handling delegates to F1/Brief 4A's plumbing.
- **F5 — 401/403 axios response interceptor.** The existing response interceptor in `api.ts` now handles 401 (force token refresh via `getIdToken(true)`, retry once with single-flight queueing for concurrent 401s, sign out on second failure) and 403 with `USER_BANNED` or `EMAIL_BANNED` codes (sign out, set `accountBanned: true`, return a never-resolving promise). Other 403s fall through unchanged. The interceptor matches on Q2-verified backend response shape: `{errors: [{field: null, code: 'USER_BANNED', translationKey: 'user.banned'}]}`.
- **F13 — Auth guards on secured routes.** Three layouts (`(portal)/(secured)/_layout.tsx`, `owner/_layout.tsx`, `owner/dashboard/_layout.tsx`) now read `user` and `_hasHydrated` and redirect unauthenticated users to `/` via expo-router's `<Redirect>`. Auth guards transparently cover deep-link entry (expo-router routes deep links through layout trees); deep-link handler wiring itself remains deferred.
- **F22 — Hydration race resolved in init components.** `ChatsInit` and `NotificationsInit` now gate their effect bodies on `_hasHydrated`, eliminating the cold-start double-init (subscribe → unsubscribe → subscribe / fetch → reset → fetch). `InitFavoritesStore` was correctly identified as not needing the gate — it uses Firebase Auth's `onAuthStateChanged` directly, not Zustand state, so the race doesn't apply.

**New component shipped:** `AccountStateDialogsInit.tsx`. Mirrors web's component of the same name. Subscribes to `restored` and `accountBanned` Zustand flags, opens the appropriate dialogs via the existing `InfoDialog` system, clears each flag inside the same effect. A subscription stub for `accountJustDeleted` is in place for chat E (mobile user-deletion adoption) to activate — chat E adds the flag to `AuthState`, replaces a local constant with a store selector, and adds dialog-opening logic to the effect body.

**Dead-code cluster removed.** A late-session audit (prompted by Igor noticing an empty `useCatalogStore.ts` file on disk) revealed that the catalog "store" referenced in the structural audit was already dead code with zero callers in the codebase. A small cleanup brief deleted three unreachable files: `src/lib/store/useCatalogStore.ts` (Zustand store, zero callers — the only caller had been added by Brief 2 and removed by Brief 2-bis), `src/lib/hooks/useCatalog.ts` (React hook, zero callers), and `src/lib/storage/catalogStorage.ts` (AsyncStorage wrapper, only called by the dead hook). The live catalog data path remains entirely in `AppContext.tsx`. `CatalogDTO` and other live types were preserved. The cleanup means the catalog-versioning backlog item (added below) now applies to the `AppContext.tsx` catalog data path specifically, not to a Zustand store.

**The audit's framing of the catalog "store" was misleading from the start.** The structural audit (Finding 3) described `useCatalogStore` as "never cleared, once-only init" — implying live but stale. In fact it had been dead code since before Φ1 opened. Two briefs (Brief 2 and Brief 2-bis) operated on the file based on the audit's framing without anyone noticing it had no callers. This was caught at session-summary time, not in production, but it's worth recording as a process signal: audit findings should be verified for liveness, not only correctness. Mastermind's brief-drafting process now includes a file/symbol/caller existence check before drafting any brief that depends on a specific file or method.

**Backend constraint respected.** No backend code changes in Φ1. Mobile conforms to existing backend contracts. Q2 (USER_BANNED 403 response shape) was answered via a read-only backend verification chat — confirmed: all three 403 paths (`FirebaseAuthFilter` mid-session, `firebase-sync` for disabled user, `firebase-sync` for banned email) produce the identical envelope shape. Spec at `user-deletion.md` §8.8 documented the wrong `translationKey` values (`errors.user.banned` / `errors.email.banned`) but the actual code emits `user.banned` / `email.banned`. Spec correction applied as part of this closing batch.

**Engineering footprint.** Eight engineer briefs in `oglasino-expo` plus one read-only backend verification: F2 (one file), F3 + F3-correction (one file across two sessions), F22 (two files), F1 Brief 4A (four files including one new utility), F1 Brief 4B (three files including one new component), F13 (three layout files), F4 (one new component plus AppInit mount), F5 (one file), catalog dead-code cleanup (three files deleted). 109 tests passing throughout. All sessions cleared baselines for tsc/lint/test/expo-doctor with zero new regressions.

**One small gap introduced and recorded as Ω scope.** Φ1 added two code paths that call `auth.signOut()` outside of `authStore.logout()` (the axios 403 interceptor and the foreground re-validation listener). The auth listener's null-path cleanup only sets `user: null`; the full store cleanup lives in `authStore.logout()`. Result: stores retain stale data briefly when sign-out is triggered from an interceptor rather than the logout action. Low severity — the user is immediately on the public surface and stale data is invisible. Fix path: either add full store cleanup to the listener's null-path or route interceptor sign-outs through `authStore.logout()`. Recorded in `issues.md`.

**Two-listeners observation also recorded for Ω.** `InitFavoritesStore` continues to use `listenAuthState` (`onAuthStateChanged`) while the new `initAuthListener` uses `onIdTokenChanged` directly. Both fire on overlapping events but with different timing for token rotations. Benign for now (favorites listener doesn't call `syncUserToBackend`); structural untidiness for Ω cleanup. Recorded in `issues.md`.

**Circular module dependency** introduced by Brief 4A (`api.ts` → `authStore.ts` → `authService.ts` → `api.ts`) — dynamic only, no runtime issue, tests pass. The right fix is extracting the axios instance to a leaf module. Recorded in `issues.md`.

**Effect on downstream chats:**

- **Chat E (mobile user-deletion adoption) is massively smaller.** Auth listener, logout fix, store cleanup, foreground re-validation, 401/403 interceptor, auth guards, hydration race, and the `AccountStateDialogsInit` ban/restoration scaffold are all done. E retains only the deletion UI surface — the danger zone on the user settings screen, the deletion-in-flight flag on `useAuthStore`, the activation of the `accountJustDeleted` subscription stub in `AccountStateDialogsInit`, and the post-deletion dialog wiring. See `.agent/handoffs/expo-chat-e.md` for the handoff brief.
- **Bucket-1 bug B1 (session leak on logout) closed by F2.**
- Φ2 (navigation foundation) opens next per the program spec `features/expo-structural-foundation.md` section 9.

**New backlog item:** Catalog versioning — backend extension to expose a catalog version that mobile (and web) consult before deciding whether to refetch. Surfaced when the F3 catalog-clear was reverted in Brief 2-bis and reinforced by the dead-code cleanup. Now applies to the `AppContext.tsx` catalog data path specifically.

**Alternatives considered and rejected:**

- **Adding `_hasHydrated` gate inside `initAuthListener` itself** (in addition to the gate at the `AppInit` call site) — rejected as defense-in-depth that adds noise; the AppContext readiness → `AppInit` mount sequence implies `_hasHydrated` will be true by the time the listener subscribes. Brief 4A added the explicit gate at the call site only.
- **Extracting a `SessionGuard` component for the three F13 layout guards** — rejected per Part 4a; three callers with two-line guards each don't earn an abstraction with a fourth caller not foreseeable.
- **Including catalog clear in F3's logout cleanup** — initially shipped, then reverted in Brief 2-bis. Catalog is not user-scoped; clearing on logout forces unnecessary re-fetch. Later cleanup brief deleted the catalog "store" entirely as it had zero callers.
- **Treating Q2 as a backend-change decision** — rejected per Igor's binding constraint on backend stability. Q2 was scoped as read-only verification only; mobile conforms to whatever shape web already consumes in production.
- **Adding a dedicated "session expired" dialog on second-401** — rejected; sign-out is sufficient in Φ1. Could be added in Φ3/Ω.
- **Consolidating `uploadImages.ts`'s local 401 retry with F5's global interceptor** — rejected for Φ1 scope (different HTTP clients, different error body shapes). Ω consolidation target.
- **Auto-dismiss after 10 seconds on the mobile restoration dialog (matching web)** — rejected because mobile's `InfoDialog` has no timer support; adding it would require a feature addition to `InfoDialog` that's out of Φ1's polish scope. User dismisses manually.
- **Restoring `useCatalogStore.ts` to its committed state before declaring Φ1 done** — rejected when the cleanup audit revealed the file had zero callers and was always dead code. Restoring a dead file just to delete it later is busywork.

**Factual vs inferred:** all findings, fix shapes, verifications, and the dead-code cluster removal are factual (produced by the eight Φ1 engineer briefs plus the backend Q2 verification plus the cleanup brief). The "introduces one new low-severity gap" framing is factual (the engineer surfaced it in Brief 7's summary). The Ω-routing decisions for the three carry-forward items (store-cleanup gap, two-listeners observation, circular import) are inferred — they're sensible follow-ups per Part 4b, not Mastermind-locked decisions. The note that "the audit's framing of the catalog store was misleading from the start" is factual (the dead-code cleanup audit established this with grep evidence and git history).

---

## 2026-05-25 — Expo foundation work is tentative; full revert reserved

The Expo structural foundation chats (Φ1–Φ4) are proceeding on main, but the quality of the output is unverified. Igor will test the results after the work lands. If the work does not meet expectations, all Expo foundation changes across repos will be reverted — and all corresponding docs changes (session archives, feature spec updates, state table rows, decisions referencing shipped status) will be reverted in oglasino-docs to match.

This means: no status in state.md or feature specs should be treated as final until Igor confirms the Expo foundation work passes manual testing. Until then, all Expo foundation docs are provisional.

**Alternatives considered and rejected:**

- Feature branch for Expo work — rejected; single-branch workflow (main only) is the established convention for oglasino-docs and the code repos.
- Ship and patch forward — rejected; if the structural foundation is wrong, patching on top compounds the problem. Clean revert is cheaper.

---

## 2026-05-24 — SEO foundation feature close

Eighteen briefs across web and backend over 2026-05-19 to 2026-05-24, two read-only audits. Shipped: structured data as `<script>` tags (was `<meta>`); per-base-site hreflang clusters with per-cluster x-default; `<html lang>` with proper BCP-47 tags; server-side breadcrumbs with shared chain resolver; product cards as `<a href>` anchors in initial HTML (SSR-renderable after vestigial Zustand guard removed); abandon localized URL paths; heading hierarchy with sr-only h1s on home/catalog and visible promotions on the other four pages; sr-only catalog SEO descriptions; small fixes batch including apex canonical host, user canonical with userId, robots.txt locale-aware patterns, og:image translated alts, publisher.logo URL, priceCurrency uppercase, metadataBase override removal, MetadataTranslator pattern migration across 12 files; JSON-LD removed from privacy/terms; hreflang verification script wired as `npm run test:hreflang`.

In-feature spec defects caught and resolved (no separate issues.md entries needed): `PrivacyPolicy`/`TermsOfService` not valid Schema.org types → `WebPage + about` → ultimately JSON-LD removed; BCP-47 collision in locale mapping → per-base-site clustering; localized URL seeds were unused fiction → abandoned with hardcoded English paths; x-default direction was suboptimal → per-cluster.

Quality gate: brief 2c verification script catches hreflang regression; Schema.org Validator owed by Igor on three pages (product, privacy [now empty], terms [now empty]).

**Per-cluster x-default for hreflang:** each base-site cluster designates its Serbian-language variant as `x-default`. rs cluster → `rs-sr`; rsmoto cluster → `rsmoto-sr`; me cluster → `me-sr`. Serbian is the largest user share per base site. The intro page (`/`) is the sole exception: its all-locale cluster uses apex `/` as `x-default` because the intro serves as the locale-selector landing page.

**Alternatives considered and rejected:**

- Global `x-default` pointing at a single URL across all clusters — rejected; each cluster is independent, and a single `x-default` cannot represent three independent markets.
- Using English (`rs-en`) as `x-default` on the rationale that English is the "international fallback" — rejected; Google serves `x-default` to users whose locale doesn't match any cluster sibling, and Serbian users comprise >90% of traffic. The English variant is for the remaining international minority, not the default.
- Omitting `x-default` entirely — rejected; Google's documentation recommends including it; omission causes Google to auto-select, which may not match operator intent.

---

## 2026-05-24 — Expo structural audit closed; foundation chats added to queue, A–I rescoped

A Mastermind chat ran a read-only structural audit of `oglasino-expo` across seven categories (lifecycle, performance, state-shape, web-seam, RN-convention, platform-API, and cross-repo trust boundaries). The audit produced 26 findings (F1–F26), 18 out-of-scope observations, and 3 cross-repo questions (Q1–Q3). Full audit archived at [`sessions/2026-05-24-oglasino-expo-structural-audit-1.md`](sessions/2026-05-24-oglasino-expo-structural-audit-1.md) and [`sessions/audit-expo-structural.md`](sessions/audit-expo-structural.md).

**Rating: 5.5/10.** The app functions but is structurally wrong for the platform in foundational ways. The dominant categories of wrongness are: (1) **lifecycle gaps** — auth listener never called, logout doesn't sign out of Firebase, no 401/403 interceptor, no foreground auth re-validation, no network state detection; (2) **performance shape** — zero `React.memo` across the entire codebase, whole-store Zustand subscriptions causing re-render storms, `expo-image` installed but never imported, AppContext value not memoized; (3) **RN convention violations** — every layout uses bare `<Slot />` with no native Stack or Tab navigators, message list not inverted, `<div>` elements in RN code.

**Five most consequential findings:**

1. **F1 — `initAuthListener` never called.** The auth store defines a Firebase `onAuthStateChanged` listener but zero call sites exist. The app relies entirely on stale AsyncStorage persistence. A banned/deleted user sees the app as if still logged in.
2. **F2 — `logoutFirebase` doesn't call `auth.signOut()`.** Firebase session persists after logout; axios interceptor continues attaching the old user's Bearer token to every API request.
3. **F5 — No 401/403 response interceptor.** Banned/deleted users are never force-logged-out. Individual API calls fail with scattered errors instead of a clean ban-notice dialog.
4. **F8/F9 — Zero React.memo and expo-image unused.** The product feed — the app's primary surface — re-renders every card on every state change and has no image disk caching. `expo-image` is in `package.json` but never imported; all 14+ image sites use RN `Image`.
5. **F12 — All layouts use `<Slot />`; no native navigators.** No native transitions, no swipe-back, no tab state preservation. The app feels like a web SPA in a native shell.

**Decision: insert four foundation chats (Φ1–Φ4) before chat A in the queue.** The audit surfaced cross-cutting issues that do not map to any single feature chat. Performance-shape wrongness (F7, F8, F9, F10, F25) and navigation rearchitecture (F12) affect every feature surface equally. Distributing these fixes across A–I would produce inconsistency, and building features on broken foundations (no auth listener, no native navigation, performance issues) would require rework later.

The four foundation chats:

- **Φ1 — Auth lifecycle foundation.** F1, F2, F3, F4, F5, F13, F22. Fixes auth listener, logout, store cleanup, foreground re-validation, 401/403 interceptor, auth guards, hydration race. Why first: every other chat depends on auth being correct. Lifecycle findings are security/trust failures, not polish.
- **Φ2 — Navigation foundation.** F12, F13 (integration with navigators), F26. Replaces bare `<Slot />` with `Stack` and `Tabs` navigators. Why second: feature chats need to know the navigation shape they're building inside.
- **Φ3 — Performance foundation.** F7, F8, F9, F10, F20, F25, F23. Zustand selectors, React.memo, expo-image adoption, AppContext memoization, virtualized selectors, scroll throttle, chat store split. Why third: needs navigation in place (Tabs persistence changes re-render math).
- **Φ4 — Service layer + error contract foundation.** F18, F24, F6. Implements Part 7 error contract across all 17 services, fixes reportService boolean inversion, adds NetInfo for offline-vs-maintenance distinction. Why fourth: the seam between mobile and backend. Makes chat A possible.

**Decision: A–I rescoped.** Most chats become smaller because foundation work absorbs cross-cutting concerns. Chat E becomes massively smaller (auth plumbing done in Φ1). Chat H loses `expo-image` adoption (moved to Φ3 because images affect the product-feed re-render math addressed by Φ3).

**Decision: append two new chats at the end.**

- **Ω — Final structural sweep + dead-code removal.** Covers the audit's 18 out-of-scope observations: `react-dom` and `dotenv` deps, dead service functions, `healthCheckService` deletion, `loginWithFacebookFirebase` stub, `cookie/` directory, `UserMenu` broken paths, `setActiveChatId` type mismatch, chat list pagination wiring, `KeyboardAvoidingView` standardization, store directory rename, `var` → `const`/`let`, `getUniqueID` replacement.
- **Ψ — Runtime verification pass.** Not a code chat. A QA-style chat to run the app on real devices (mid-range Android + iPhone) and verify structural fixes produced intended runtime behavior. Covers findings that could not be verified without runtime: F6 maintenance-vs-offline, F7 re-render storms (profiler trace), F22 hydration race (cold-start timing).

**Three cross-cutting decisions locked in this chat:**

1. **Φ1 starts immediately.** Nothing in flight blocks it.
2. **`expo-image` adoption (P3 from `expo-release-readiness.md`) moves from chat H into Φ3** because images affect the product-feed re-render math addressed by Φ3.
3. **Cross-repo question Q2 (backend ban-response shape) must be answered before Φ1 writes the 401/403 interceptor.** A small backend audit chat opens in parallel with Φ1 to answer Q1, Q2, Q3.

**Three cross-repo questions raised by the audit:**

- **Q1** — re-confirming X3: does the old `POST /secure/product/addUpdate` endpoint still exist?
- **Q2** — new: does the backend USER_BANNED 403 response include a machine-readable code in the error body?
- **Q3** — re-confirming X1: does `POST /secure/push/token` validate `userId` against the authenticated principal?

**New spec:** [`features/expo-structural-foundation.md`](features/expo-structural-foundation.md) covers Φ1–Φ4 as a multi-chat program, analogous to `expo-release-readiness.md` for A–I. Revised full queue order: Φ1 → Φ2 → Φ3 → Φ4 → A → B → C → D → E → H → G → F → I → Ω → Ψ.

**Alternatives considered and rejected:**

- **One mega-foundation chat covering all of Φ1–Φ4.** Rejected: too large to manage as one chat; the four areas are independent enough to sequence.
- **Distributing structural fixes across the existing A–I queue.** Rejected: cross-cutting issues (React.memo, expo-image, Zustand selectors, navigators) do not map cleanly to any one feature chat; doing them per-feature produces inconsistency.
- **Skipping the foundation and going straight to A–I.** Rejected: A–I would build on broken foundations (no auth listener, no native navigation, performance issues), requiring rework later.
- **Treating the audit as informational only without changing the queue.** Rejected: the audit's lifecycle findings (F1, F2, F5) are security/trust failures, not polish.

**Factual vs inferred:** the audit findings, rating, and category breakdown are factual (produced by the read-only audit). The four-chat decomposition and sequencing are factual (confirmed by Igor in the structural audit Mastermind chat). The rescoping of A–I as "most chats become smaller" is inferred from the scope overlap between the foundation chats and the feature chats — specific scope reductions are documented in [`features/expo-structural-foundation.md`](features/expo-structural-foundation.md) section 4.

---

## 2026-05-24 — Admin removal shipped (mobile, chat α)

Admin surface deleted from `oglasino-expo`. Mobile is now a consumer-only app with no admin UI. Admin functionality remains web-only in `oglasino-web`.

**What shipped:**

- 49 admin-only files deleted (12 routes, 21 components, 8 services, 6 chat types, 1 review type, 1 navigation config).
- 13 coupling points resolved across 10 shared files: `PortalScope` narrowed to `'dashboard' | 'portal'`; `useCardSizeStore.adminSize` removed; `useAdminFilterStore` export removed; three `getAdmin*` wrappers removed from `productsSearchService`; `getReactAdminSuggestions` removed from `suggestionsService`; admin ModerationState filter removed from `FiltersDialog`; `ProductCard` admin deps removed; `SearchInput` admin route removed; four admin dialog IDs removed; `ADMIN_PAGES` namespace removed from `TranslationNamespace` enum; `SelectFilter` namespace swapped from `ADMIN_PAGES` to `COMMON` (done prior to session).
- Stale Firebase Admin SDK credential file (`oglasino-dev-firebase-adminsdk-fbsvc-002e6b2f58.json`) deleted. Key invalidated in Firebase.

**Verification:** `npx tsc --noEmit` (10 pre-existing errors, zero new), `npm run lint` (18 pre-existing errors + 82 warnings, zero new), `npm test` (109 passed), `npx expo-doctor` (17/18 pre-existing pass, zero admin regression).

**No backend, web, router, or Firestore Rules changes required.** The `filter.all.label` translation key was already moved from `ADMIN_PAGES` to `COMMON` namespace in all four backend locale seed files by Igor prior to this session.

---

## 2026-05-23 — Google Analytics v1 shipped (web complete; mobile deferred)

Complete client-side GA4 instrumentation in `oglasino-web` plus one backend field in `oglasino-backend`. Code-complete across nine engineer briefs landed on `dev` between 2026-05-22 and 2026-05-23. Stage GA4 property (`oglasino-stage-49abb`, `G-P0LEVEJ0V9`) configured 2026-05-23 with custom dimensions, conversion events, internal-traffic filter, and 14-month retention. Prod GA4 property (`oglasino-prod-7e5db`, `G-GNKB4WBNC0`) configured 2026-05-23 with the same runbook, with two post-launch items deliberately deferred (see "Post-launch items" below).

**What shipped:**

- **Foundation** (`src/lib/analytics/`): `track(event, params)` helper with three guards (server-side, `__og_consent_loaded` falsy, `window.gtag` absent) plus `NEXT_PUBLIC_GA4_DEBUG_MODE` injection; `trackError(err, context)` helper with `Error` normalization and 200-char message truncation; full unit coverage.
- **Script load** (`app/layout.tsx`): two `<Script strategy="afterInteractive">` blocks gated on `NEXT_PUBLIC_GA4_MEASUREMENT_ID`, with `gtag('config', ..., { send_page_view: false })` so the SDK's automatic page_view is disabled and `GA4RouteListener` fires it manually on pathname change.
- **Stub initializers mounted**: `GA4RouteListener` (sibling to `NavigationProgressBar` in `app/[locale]/layout.tsx`) fires `page_view` on pathname change; `GA4UserIdSync` (mounted from `AppInit.tsx`) calls `gtag('set', { user_id })` after `useAuthResolved()` is true. The third diagnostic initializer `GA4ErrorListener` registers `window.addEventListener('error', ...)` and `window.addEventListener('unhandledrejection', ...)` with `instanceof Error` normalization.
- **Backend `wasRegister` field** (`oglasino-backend`): new `SyncResult(User user, boolean wasRegister)` record returned by `FirebaseAuthService.getOrCreateUser`. `wasRegister` derived inside the synchronized `createUserSynchronized` block at the actual `INSERT` branch — concurrent-create race correctly resolves with the loser thread reporting `false`. `AuthUserDTO` carries the field on the wire; the error response (`ProductErrorResponse` shape per Part 7) intentionally does NOT carry it.
- **12 events plus automatic `page_view`** wired across the firing surfaces:
  - `page_view` — pathname change.
  - `sign_up`, `login` — `UseTokenRefresh.tsx` post-`setUser`, discriminated by backend-authoritative `wasRegister`. Provider mapping from `firebaseUser.providerData[0]?.providerId` (`password → email`, `google.com → google`, `facebook.com → facebook`).
  - `product_view` — new `ProductViewTracker` client island mounted from the product detail page, `is_owner_view` computed from `useAuthStore.user.id === ownerId`.
  - `product_create_started` — `CreateNewProductDialog.tsx` mount effect when the wizard's first user-facing step (`currentStep === 0`, `ImageSelectionProductDialog`) mounts.
  - `product_create_completed` — `UploadedProductDialog.tsx` first statement inside the `result.type === 'success'` branch.
  - `contact_seller_clicked` — three buttons: `CallUserButton.tsx` (top of `handleClick`, with new `productId` prop plumbed from `ProductFunctions.tsx`); `StartMessageButton.tsx` (happy-path branch only — not on login-dialog or block-info-dialog branches); `FavoriteButton.tsx` (add-transition only via `const isAdding = !isFavorite` captured AFTER the `!user` and owner early returns).
  - `message_sent` — `useChatStore.sendMessage` after `await batch.commit()` resolves, before the optimistic-message replace.
  - `search` — `SearchInput.commitSearch` with 100-char truncation.
  - `view_search_results` — new `ViewSearchResultsTracker` client island mounted from home and catalog pages, fires once per mount when a non-empty search term is present.
  - `filter_change` — `FilterManager.tsx` SYNC-TO-URL effect, gated by `hydrated` + `newUrl !== current` so URL hydration doesn't trigger spurious events; `filters_active` built from category-key presence only (no filter values).
  - `exception` — `app/error.tsx`, `app/global-error.tsx`, plus `GA4ErrorListener`. Four `boundary` parameter values: `'route'`, `'global'`, `'window_onerror'`, `'unhandled_rejection'`.
  - `form_submit_failed` — `parseProductValidationErrors` refactored to return `{ byField, list }` dual-view shape preserving Part-7 `code` end-to-end. Backend-validated forms (product create step 1 + step 4, product update) fire with real Part-7 codes; client-Zod forms (register, login, profile update) fire with synthetic codes defined inline at each validation branch. `SuggestCategoryDialog` left unchanged — boolean error shape too coarse.

**Key architectural decisions captured during the feature:**

- **Single `track` helper, no per-event helpers.** Three guards keep events safe pre-consent and pre-`gtag.js`-load. Every event call site uses `track(...)`; no direct `window.gtag(...)` calls outside the helper.
- **`send_page_view: false` on `gtag('config', ...)`.** GA4's automatic `page_view` would miss Next.js client-side route transitions. The custom `GA4RouteListener` captures them.
- **`useAuthResolved` gates `user_id` and `is_owner_view`.** `gtag('set', { user_id })` only fires after Firebase auth has resolved AND the store has hydrated. `is_owner_view` on `product_view` waits for resolution before firing, so anonymous-vs-owner classification is correct.
- **`wasRegister` is backend-authoritative.** The frontend's `nextRegisterDisplayName` cell continues to serve its display-name role on the `/auth/firebase-sync` POST but is no longer the sign-up discriminator. The backend bit captures registration on every auth method (email+password, Google, Facebook) — the audit's "first-time social-auth looks like login" concern is closed.
- **PII contract enforced at every firing site.** Free-text fields (name, description, displayName, email, phone, message bodies, image URLs) never enter event payloads. Audit § PII-leakage notes is the contract. Verified by the brief 5 engineer's `## PII check` discipline and carried forward to briefs 6, 7, 9 by the same shape.
- **`form_submit_failed.error_code` split between backend codes and synthetic codes.** Product flows fire with real Part-7 codes preserved through `parseProductValidationErrors` (which now returns `{ byField, list }` — the dual-view shape is the engineer's design judgment that improved on the brief's literal proposal). Register / login / profile update fire with synthetic codes defined inline at each validation branch; the synthetic set is documented in the brief 9 session summary.
- **`FavoriteButton` add-only and `StartMessageButton` happy-path-only.** Locked design calls in the brief 5+9 review. Friction interactions (anonymous click, blocked user, not-logged-in) do not fire `contact_seller_clicked`. The events measure committed engagement, not click attempts.
- **Filter-change values vs keys.** `filter_change.filters_active` carries category KEYS only (`'price'`, `'region'`, `'search'`, `'filter:condition'`), not VALUES. Filter-option values would be high-cardinality and borderline-PII (specific category names from user-controlled filters).
- **No backend changes beyond `wasRegister`.** GA4 v1 is client-side telemetry. Trust boundaries unaffected — no event payload feeds any server-side decision.
- **Reporting Identity stays "Blended" on both properties.** Default GA4 setting; enables Consent Mode v2 behavioral modeling tier (third tier after User ID and Device ID). Google Signals deliberately left OFF on both properties because enabling it would conflict with the Privacy Policy commitment that `ad_*` consent signals stay permanently denied.

**Stage and prod GA4 admin configured per the runbook in the spec's § GA4 admin-side setup runbook:**

- Properties identified and Web data streams confirmed. Both Web data streams have correct STREAM URL set (prod's STREAM URL was blank when Firebase auto-created the property; corrected to `https://oglasino.com` during this work).
- 19-20 event-scoped custom dimensions registered per property.
- Conversion events marked on stage: `sign_up`, `product_create_completed`, `contact_seller_clicked`, `message_sent`. **Marking on prod is deferred until at least one event of each type has been received post-deploy** (GA4 doesn't allow marking an event as a conversion before it exists in the events list). See "Post-launch items" below.
- Consent Mode v2 settings: Reporting Identity is "Blended" on both properties (default), enabling behavioral modeling. Google Signals deliberately left OFF.
- Internal-traffic filter active per property: Igor's home IP added, filter state flipped from Testing to Active.
- Data retention: 14 months with reset-on-new-activity.
- DebugView verified working with `NEXT_PUBLIC_GA4_DEBUG_MODE=true` and the stage measurement ID during stage manual verification.

**Manual end-to-end verification:** Igor walked the full event matrix against stage 2026-05-23, including the seven-step DevTools verification of the script load and consent-gate behavior. All 12 events fired correctly, consent gating worked (Reject-all → cookieless pings, no `_ga`/`_gid` cookies; Accept-all → cookies + identifiers), first `page_view` did not race (theoretical concern flagged by brief 2 engineer didn't manifest).

**Process notes from the feature:**

- **Audit-driven Phase 5 is durable.** The Phase 2 read-only audit caught five real issues before they hit Phase 5: `CallUserButton` missing product context, social-auth `wasRegister` discriminator gap, `error_code` source for form failures, `next/script` not-yet-used misstatement, `addDoc` reference rot in the spec. All resolved in Phase 3 amendments before any engineer began coding.
- **Sequential audits do drift.** A Phase 2 audit's line numbers and call-site assumptions can drift between audit and execution. Three small drifts surfaced during code briefs (CallUserButton call site is `ProductFunctions.tsx` not `UserDetails.tsx`; `parseProductValidationErrors` returns a flat map not per-error objects; `currentStep === 0` is `ImageSelectionProductDialog` not `BasicInfoProductDialog`). All caught by engineers' "Brief vs reality" discipline.
- **`parseProductValidationErrors` dual-view shape (brief 9) is the engineer's improvement on Mastermind's proposal.** The brief's literal proposal would have forced ~30 consumer-site mechanical edits across three files. The engineer's `{ byField, list }` shape preserved the existing consumer ergonomics and added per-error iteration where the brief needed it. Pattern for future briefs: when a refactor's blast radius isn't obvious from the audit, the engineer's judgment at implementation time can improve on the brief.
- **`issues.md` ceiling discipline worked.** GA4 v1 added one entry (`GlobalError` naming) and closed it in the same feature, for net zero. Several free-floating observations from engineers (the `selectedModeratioStates` typo, `useAuthStore` "sole caller" comment drift, `refreshUser` second-caller, FavoriteButton `console.error` in catch, seventh `setErrorMessage` site in profile_update) were held without filing — preserved in session summaries for future cleanup chats.

**Cross-repo coordination:** the feature touched `oglasino-web` (eight code briefs) and `oglasino-backend` (one code brief). No cross-repo briefs — Phase 3 amendment 2 split brief 3 (backend `wasRegister`) and brief 4 (web auth events) into separate sessions per conventions Part 3.

**Post-launch items (durable record):**

Two prod-side items were deliberately deferred. Both have natural triggers post-launch and are not blocking GA4 v1 from being considered shipped.

1. **Mark four conversion events on the prod GA4 property.** Cannot be done pre-launch because GA4 requires at least one of each event to have been received before it can be marked as a conversion. Within the first day or two of production traffic, all four events (`sign_up`, `product_create_completed`, `contact_seller_clicked`, `message_sent`) will accumulate naturally. Once they appear in **Admin → Events** on `oglasino-prod-7e5db`, toggle "Mark as conversion" ON for each of the four. The other events (`page_view`, `login`, `search`, `view_search_results`, `filter_change`, `product_view`, `product_create_started`, `exception`, `form_submit_failed`) stay as engagement events.

2. **Search Console verification + GA4 link.** `oglasino.com` is not yet a verified property in Google Search Console. Search Console is the diagnostic surface for SEO health (crawl errors, indexed pages, search queries that lead to the site, structured-data validation, sitemap submission). Verification is a DNS TXT record setup against `oglasino.com`, takes ~10-15 minutes via Cloudflare. Recommended timing: **1-2 days before prod deploy** so the property is verified and ready to capture crawl data from launch day. Then post-launch (1-2 weeks after deploy, once Google has had time to crawl), come back to GA4 → **Admin → Product links → Search Console links → Link** and connect the verified Search Console property to the `oglasino-web-prod` data stream. The GA4-side link pulls organic-search performance data (queries, impressions, CTR, position) into GA4 reports — only useful once Search Console has data, which only happens once the site is indexed.

**Outstanding pre-launch action:** legal-drafts chat notification (per spec § Open items at draft time). Privacy Policy may need a paragraph update reflecting GA4's cookieless-ping behavior under `analytics_storage: denied`. Drafted as a separate Igor task; not blocking the feature shipping but blocking the legal review milestone.

**Mobile (`oglasino-expo`) is deferred to a future Mastermind chat.** Mobile analytics has a distinct SDK story (`@react-native-firebase/analytics` or `expo-firebase-analytics`) plus App Tracking Transparency on iOS. Queued in the Expo backlog table.

**Alternatives considered and rejected:**

- Combining Consent Mode v2 and GA4 v1 into one feature — rejected per the 2026-05-20 decisions entry; consent has independent UX and legal implications; bundling obscures decisions.
- Server-side GA4 via Measurement Protocol — rejected for v1; doubles event-modeling work for a benefit (ad-blocker resilience) that doesn't justify the cost pre-launch.
- Single measurement ID across stage and prod — rejected per the spec; stage traffic would pollute prod reports.
- Loading `gtag.js` via raw `<script>` synchronously — rejected; `next/script` with `strategy="afterInteractive"` defers until after the inline consent snippet executes, preserving Consent Mode v2's ordering contract.
- Firing `contact_seller_clicked` on Favorite REMOVE — rejected; un-favoriting is the opposite engagement signal, would produce a meaningless metric.
- Firing `contact_seller_clicked` on `StartMessageButton`'s friction branches (not-logged-in, blocked) — rejected; friction isn't engagement.
- Extracting a shared `fireContactSeller(method, productId, sellerId)` helper across the three contact-seller buttons — rejected per Part 4a; three callers, three lines each, abstractions earn their introduction.
- Firing on `SuggestCategoryDialog` failure — rejected; boolean error shape carries no field or code, would produce a meaningless event.
- Stack traces in `exception` event payload — rejected per spec; too large and may contain PII fragments from URL params.
- Filter values in `filter_change.filters_active` — rejected; high-cardinality and borderline-PII.
- Turning on Google Signals — rejected on both properties; conflicts with the Privacy Policy commitment that `ad_*` consent signals stay permanently denied.
- Generating synthetic test events from a non-internal IP to enable conversion marking on prod pre-launch — rejected; first-day-of-launch traffic will produce real events naturally within minutes, generating fake events pollutes the prod property's data history for the sake of a setup convenience.
- Setting up Search Console pre-feature-launch — rejected; Search Console has no data to show until the site is publicly reachable AND has been crawled by Google. Verification can happen anytime, but the GA4-side link only adds value once organic-search data exists. Defer to 1-2 days before deploy.

---

## 2026-05-22 — Cookies-closing

The cookies-closing Mastermind chat closed today. Four work items: card-size button display fix, isDashboard → portalScope migration (including dashboard → owner rename), userPreferenceService removal, and language + theme as preference cookies. Items 1–3 shipped as spec'd. Item 4 shipped the language portion with significant divergence from the original spec; theme deferred to a future Mastermind chat with handoff doc at `.agent/handoffs/theme-path-a.md`.

### Item 4 — language shipped, theme deferred

- **Cookie carries bare language code** (`'sr' | 'en' | 'ru' | 'cnr'`), not tenant-locale compound. Spec decision #1 — unchanged.
- **Cookie-wins-over-URL middleware redirect** runs in `proxy.ts` (Next.js 16 naming). When cookie's language differs from URL's language AND target tenant supports cookie's language, redirect. When cookie's language isn't allowed by target tenant, URL wins (cookie preserved as preference). Spec decision #2 — implemented and shipped.
- **`getLocalizedPath` helper** preserves paths uniformly across language change. Product URLs use canonical-slug rewrite after data resolves (decision #3-A from the revised spec). Removed earlier reset-to-tenant-root behavior for product paths. Spec decision #3 (revised) — shipped.
- **`localeDetection: false` and `localeCookie: false`** in `routing.ts`. Custom proxy logic is the sole locale authority. Spec decision #4 — shipped.
- **Theme persistence path B1-A (`next-themes` custom storage adapter) was unavailable in v0.4.6.** Path A (replace `next-themes` with minimal cookie-based theme system) chosen but deferred to future chat. See handoff. Spec decisions #5–#6 — deferred.
- **Reconciliation semantics diverge by preference type** (spec amendment to decision #7):
  - **Card-size:** in-memory-wins-on-grant. The in-memory state reflects a real user-click signal; if it differs from the cookie on consent grant, write the in-memory value to the cookie. Handled by `SyncCardSizeFromCookie`.
  - **Theme:** in-memory-wins-on-grant (will follow same pattern when implemented).
  - **Language:** cookie-wins. The in-memory locale (from `useRoutingLocale()`) reflects the URL, which can be a forced fallback from a cross-tenant redirect rather than a user preference. No URL→cookie sync component exists; cookie is set only by explicit user choice via PortalConfigDialog.
- **Defensive cookie parsing in middleware.** Malformed cookie or missing fields fall through to URL wins. Spec decision #8 — shipped.
- **`extraProducts.recently.viewed.title` translation key stays.** Spec decision #9 — known shortcut, not corrected in this work.
- **No first-visit cookie seed from URL.** Spec decision #10 — shipped.
- **Theme UI stays bi-state.** "System" option out of scope; logged to `issues.md`. Spec decision #11 — shipped (deferred-component).

### Architectural changes during Item 4 (not in original spec)

- **Routing locale separated from formatter locale.** The compound routing locale (`me-cnr`, `rs-sr`) was passed to `next-intl`'s message formatter, which calls `Intl.Locale("me-cnr")` and throws on invalid BCP-47 tags. Fix: `request.ts` now derives bare language (`cnr`, `sr`) for next-intl's formatter; the compound stays as the routing identity. Two new primitives — `useRoutingLocale()` (client) and `getRoutingLocale()` (server) — return the compound and replaced 22 of the 38 prior `useLocale()` / `getLocale()` call sites. The remaining 4 sites consume the bare formatter locale for `Intl.*` APIs and stay on next-intl's `useLocale()`. Option B chosen from a four-option investigation; full reasoning in the briefs.

- **`src/i18n/navigation.ts` rewritten to wrap `createNavigation`'s exports.** Brief 3d's bare-locale change broke `next-intl`'s `<Link>`, `useRouter`, and `usePathname` (they internally read `useLocale()` and used bare for URL prefixing). New wrappers source the compound from `useRoutingLocale()` / `useParams()`. 18 import sites unchanged at call sites; wrapper transparent. Additional 15 plain `useRouter` sites from `next/navigation` migrated to wrapped router across Briefs 3f and 3i.

- **Cross-tenant routing rules.** Product page uses code-equality (`baseSite.code !== productDetails.baseSiteOverview.code`). User page uses domain-equality (`baseSite.domain !== userDetails.baseSiteOverview.domain`). Asymmetry is deliberate: products are tenant-scoped (rs and rsmoto don't share inventory despite sharing a domain); users are domain-scoped (a user belonging to either rs or rsmoto is accessible from both domains because they share one). Confirmed with Igor.

- **PortalConfigDialog tenant-change behavior.** On tenant change, `/product/*` and `/catalog/*` paths drop to tenant home (because product IDs and catalog slugs are tenant-scoped). Other paths preserve. On language change, all paths preserve (since slugs are language-independent). Single shared `getLocalizedPath` helper handles language; inline regex in PortalConfigDialog handles tenant.

- **Cookie-preservation on cross-tenant redirects.** When the user is bounced to a tenant that doesn't support their preferred language (e.g., `cnr` user redirected to `rs` from product cross-tenant guard), URL's language defaults to the new tenant's default, but cookie's `lang` stays at user's actual preference. User returning to a tenant that supports their preference gets restored to their language. Applies to product cross-tenant guard, user cross-tenant guard, and PortalConfigDialog tenant switcher.

- **`globalCookie` cleared on consent decline.** When preference consent transitions to denied, `globalCookie` is deleted entirely. Resolves write-side/read-side consent asymmetry: write-side gates on consent (proper), read-side (proxy cookie-wins) trusts the cookie's presence — under Option 4 (chosen from a four-option investigation), the cookie only exists when consent was granted, so the read-side correctly trusts it. Includes self-healing migration on app init for pre-consent-system cookies. Spec amendment captured below.

- **`useLanguageStore` collapsed.** The store had no readers after Brief 3b deleted `SyncLanguageFromCookie`. PortalConfigDialog now writes the cookie directly via `updateGlobalCookie('lang', lang)` under same-tenant + consent gates. One file deleted.

- **Routing-cookie categorization.** `globalCookie.lang` is treated as a preference cookie (consent-gated on write, cleared on decline). This is a deliberate choice — an alternative reading would categorize it as functional (strictly necessary for multilingual routing), which would have removed the write-side consent gate entirely. Chose preference categorization for symmetry with card-size and theme and to keep the consent-gating story consistent across all `globalCookie` fields.

### Asymmetries to remember

- Routing locale ≠ formatter locale. `useRoutingLocale()` for URL construction; `useLocale()` for `Intl.*`.
- Products use code-equality on cross-tenant comparison; users use domain-equality.
- Reconciliation semantics: card-size and theme in-memory-wins-on-grant; language cookie-wins (no sync component).
- `<Link>` and `useRouter` from `@/src/i18n/navigation` (wrapped); `<a>` and `Link` from `next/link` consume URLs directly (no wrapping).

---

## 2026-05-22 — Brevo SMTP integration shipped; pattern for transactional email send

Backend now sends outbound transactional email via Brevo's SMTP relay (`smtp-relay.brevo.com:587` STARTTLS) on the free tier. Wiring is identical across dev / stage / prod; the per-environment difference is which Brevo SMTP key is used (`oglasino-backend-prod` for prod; `oglasino-backend-stage` shared by stage and dev) and which Brevo-verified From address gets stamped (`noreply@oglasino.com` for prod; `noreply-stage@oglasino.com` for stage + dev). Reply-To is `support@oglasino.com` on every environment so user replies on `noreply-*` mail land in the monitored inbox.

**The shape:**

- `oglasino.email.*` `@ConfigurationProperties` carries the envelope (From / From-name / Reply-To). All values come from environment variables — no defaults baked in, missing variable fails fast at startup.
- `spring.mail.host/port/username/password` plus the `mail.smtp.auth=true` / `mail.smtp.starttls.enable=true` / `mail.smtp.starttls.required=true` block wires Spring Boot's `MailSenderAutoConfiguration` — no `MailConfig @Configuration` class needed.
- `EmailService.sendPlainText(to, subject, body)` is the only entry point this brief; per-feature briefs (User Deletion confirmations, password reset, ban notifications, etc.) extend the interface for their own needs.
- `DefaultEmailService` uses `JavaMailSender.createMimeMessage()` + `MimeMessageHelper` to build a proper-headered `MimeMessage`, sets From + From-name + Reply-To from `EmailProperties`, sends. UTF-8 throughout.
- The brief ships a throwaway smoke endpoint `POST /api/secure/admin/email/test` (admin-gated, returns 500-with-`{exception, message}`-body on `MailException`) so Igor can prove deliverability end-to-end. **The endpoint is deleted by the next email-using-feature brief.**

**Trust boundary:** the From / From-name / Reply-To are server-derived from `EmailProperties` and never client-supplied. The smoke endpoint accepts `to`/`subject`/`body` from an authenticated admin but does not validate per-target (the throwaway lifetime + admin gate are the protection); permanent admin-broadcast surfaces would need their own trust-boundary work. The smoke endpoint's `{exception, message}` 500-response shape is a Part 7 (error contract) deviation scoped to the throwaway endpoint's lifetime; the next email-using-feature brief deletes the endpoint and the deviation goes with it. `EmailService.sendPlainText` itself is internal — server-side callers only.

**Free-tier quota:** Brevo free tier is 300 emails/day, shared across prod + stage (dev reuses stage's key). Per-feature briefs that ship high-volume transactional email need to either move Brevo to a paid tier or migrate to a different provider; the `EmailService` interface is the swap-out seam.

**Firebase auth emails (registration verification, password reset, email change) remain on Firebase's default sending infrastructure (`noreply@<project>.firebaseapp.com`) for now.** Decision deferred to post-launch — see [`.agent/handoffs/email-followups.md`](.agent/handoffs/email-followups.md) for the Option-C unification plan that migrates Firebase auth emails to Brevo via Firebase Admin SDK's `generateEmailVerificationLink` / `generatePasswordResetLink` / `generateVerifyAndChangeEmailLink`. Tracked as a Risk Watch item in `state.md`.

**Alternatives considered.** A `MailConfig @Configuration` class — rejected, Spring Boot autoconfigures `JavaMailSender` from `spring.mail.*` properties; the YAML config plus `EmailProperties` is all that's needed. Inlining `DefaultEmailService` into the throwaway controller — rejected, would need to be undone the moment the next per-feature brief lands. Per-target validation / allowlist on the smoke endpoint — rejected per brief; the throwaway lifetime plus admin gate is the protection. Customizing Firebase email templates in the Firebase Console (Option B) — rejected as a half-measure; users would still see two visually-distinct email systems (Firebase auth emails styled one way, Brevo feature emails styled another) and two sending infrastructures to debug. Option C (full unification on Brevo) is the target end-state; Option A (Firebase-as-is) is the launch state.

---

## 2026-05-21 — Portal home cold-load cascade: three fixes shipped

The portal home (`/[locale]`) fired 4 `POST /api/public/product/search` calls per hard refresh and 3 per normal refresh, plus duplicate `POST /api/auth/firebase-sync` and `POST /api/secure/push/token` on hard refresh, with a visible product-list swap during load (the "jerk"). Two read-only investigations + three small fixes in `oglasino-web` closed the cascade.

**Net effect:** hard refresh 4 → 2 product searches, 2 → 1 firebase-sync, 2 → 1 push/token. Normal refresh 3 → 1 product search (the other two were already 1). Visible jerk eliminated.

**Three independent root causes, three fixes:**

1. **FilterManager SYNC TO URL trailing-slash false-positive + redundant `setTimeout(router.refresh)`** (`src/components/client/initializers/FilterManager.tsx`). The effect's URL-canonicalization gate compared `newUrl` (always trailing-slash on the no-filters home) to `current` (no trailing slash) and fired `router.replace('/rs-sr/') + setTimeout(() => router.refresh(), 0)` on every refresh. In Next 16, `router.replace` to a new pathname already refetches RSC; the `setTimeout(refresh)` was leftover from an older Next version where `replace` did not consistently refetch. One firing → two RSC re-renders → two extra product-search POSTs. Fix: normalize the trailing slash in the gate (so the false positive doesn't trip on the canonical home URL), and delete the redundant `setTimeout(router.refresh)`. Lever A is route-agnostic (no other FilterManager-mounted route had the false positive because only the home produces `pathname === '/'` from next-intl's `usePathname`); Lever B applies to every legitimate FilterManager trigger anywhere. Both shipped in one session.

2. **`randomSeed: Date.now()` in the `FilteredProductList` React key** (`src/components/client/product/FilteredProductList.tsx`). The no-filters home synthesizes `{ applyRandom: true, randomSeed: Date.now() }` in `parseFiltersFromQueryParams` on every SSR render. The inner `ProductList` was keyed on `JSON.stringify(filtersData)`, so a new seed → new key → React unmounts the previous list and mounts a fresh one with the new ordering. After the FilterManager fix this was mostly latent on the canonical home, but any _real_ filter change still causes one re-render with a fresh seed and would re-surface the jerk. Fix: destructure `randomSeed` out before stringifying for the key calculation; the seed still reaches the backend via the unmodified `filtersData` on the request side. Locks the symptom out unconditionally.

3. **No single-flight guard on `onIdTokenChanged` cascade** (`src/components/client/initializers/UseTokenRefresh.tsx`). The Firebase SDK emits `onIdTokenChanged` twice on cold session restoration (persisted token, then refreshed token ~8 ms later); both emissions fired `syncUserToBackend` + `initPushForAuthenticatedUser` independently. The "sole hydrator" contract (`decisions.md` 2026-05-21 Consent Mode v2 entry) was intact — there was exactly one listener — but the listener had no idempotency. Fix: `useRef`-held `{ inFlightUid, lastSyncedUid, lastSyncedAt }` with a 2-second drop window for same-uid back-to-back emissions; `writeFirebaseTokenCookie(token)` hoisted into the listener so it runs on every emission (the cookie must mirror the rotated token even when the second backend sync is dropped); guard resets on sign-out so a sign-out → sign-in handshake re-fires a fresh cascade; `lastSyncedAt` only set on success (failed syncs retryable immediately). The 2-second window is a constant, not configurable per Part 4a — the value has one setting and no foreseeable second setting.

**Cross-cutting patterns surfaced for future reference:**

- **`router.replace` + `setTimeout(router.refresh)` is a double-fetch anti-pattern in Next 16.** Any place in the codebase still doing this should be reviewed. The audit's adjacent observation #5 flagged this generally; the fix removes it from FilterManager. If other surfaces have the same pattern, they have the same bug.
- **`useRef`-held single-flight is the right shape for `onIdTokenChanged` (and analogous) listeners.** The Firebase SDK's documented cold-restoration double-emission behavior is benign as long as the listener has an idempotency guard. The guard pattern (`inFlightUid` + `lastSyncedUid` + recent-completion window) is reusable for any other listener-driven backend cascade if one appears.
- **React `key={JSON.stringify(x)}` requires the stringified shape to exclude any field that is per-render-volatile but should not cause remount.** The `randomSeed` case is the canonical example: backend wants it, React must not see it.

**Cross-repo coordination:** none. All three fixes confined to `oglasino-web`. No backend, router, firestore-rules, or docs (besides this entry) changes required. The `userPreferenceService` tracking-cookie surface and the favorites-icon flicker (audit-1 adjacent observation; smaller-scale visual churn that lands on top of the seed jerk) remain as separate follow-up items.

**Engineering footprint:**

- Audit-1: read-only, 5 files audited, 5-question structured report.
- Audit-2: instrumentation pass, 2 temporary `console.log` lines added and removed in the same session.
- Brief 1 (FilterManager): +2 / −2 lines in one file. 206 tests green.
- Brief 2 (FilteredProductList): destructure-and-rest projection on the key, one-line comment. Tests green.
- Brief 3 (UseTokenRefresh): +50 / −3 lines in one file, manual verification owed by Igor before final closure. 206 tests green.

**Alternatives considered and rejected:**

- **Stabilize `randomSeed` across the lifecycle of one user visit** (cookie-stored seed, per-route token). Rejected for the immediate fix — larger surface, addresses a different UX question (which products appear on re-paginate). Could be revisited as follow-up if the random-on-every-render UX is itself a problem.
- **Centralize single-flight at the `syncUserToBackend` / `initPushForAuthenticatedUser` service layer.** Rejected — different blast radius, and the only listener-driven caller is `UseTokenRefresh`. The other caller of `syncUserToBackend` (`useAuthStore.refreshUser` from `UserBasicDataSelectorDialog`) is user-action-triggered and has different semantics; guarding it at the service layer would force false-positive drops on user-initiated re-syncs.
- **Move `writeFirebaseTokenCookie` out of `syncUserToBackend` entirely** (so the cookie write lives only in the listener and in `refreshUser`). Rejected — bigger blast radius across two files, and the brief asked the fix to be confined to `UseTokenRefresh.tsx`. The in-function call dedupes via token-string identity (`authTokenCookie.ts:36`), making the redundant call safe. If the structural duplication ever feels untidy, a future brief can consolidate.
- **A generic URL-canonicalization helper for the trailing-slash case.** Rejected — only one call site has this problem (FilterManager's home-route URL construction); a helper for one caller is premature per Part 4a.

---

## 2026-05-21 — Consent Mode v2 shipped

A complete cookie consent system in `oglasino-web` plus translation seeds in `oglasino-backend`, modeled on Google's Consent Mode v2 four-signal protocol. Code-complete across both repos as of 2026-05-21.

**What shipped:**

- **Foundation** (`src/lib/consent/`): `ConsentData` type with literal-typed `necessary` + permanently-denied `ad_*` fields; client-side cookie helpers (`readOgConsent`, `writeOgConsent`); SSR helper (`readConsentForSsr`); Zustand store (`useConsentStore`); shared side-effects helper (`applyConsent`); preference-gating helper (`isPreferenceConsentGranted`).
- **SSR snippet**: inline `<script>` in `app/layout.tsx`'s `<head>` calls `gtag('consent', 'default', ...)` synchronously before any other script, populated from the user's `og_consent` cookie or all-denied defaults.
- **Banner**: rebuilt Drawer with twin Accept-all / Reject-all primaries plus Customize expander. Non-dismissible on first visit, dismissible on reopen. Mounted in portal and owner layouts.
- **Footer link**: "Cookies" entry, last position in company navigation. Reopens the banner via `useConsentStore.requestReopen()`.
- **Dedicated settings page**: `/owner/cookies` hosts the three toggles (necessary, preference, analytics) with immediate-write semantics matching the banner. No Save button. Sidebar entry under User section as sibling to Settings.
- **Card-size cookie gating**: `usePortalCardSize.ts` and `useDashboardCardSize.ts` no-op their writes when consent is undecided or denied.
- **20 + 4 = 24 keys in `COOKIES` namespace + 1 key in `NAVIGATION` namespace** seeded across EN / RS / RU / CNR via inline-append to `0001-data-web-translations-*.sql`. EN copy final; RS / RU / CNR marked as placeholders pending native-translator review.
- **Legacy banner removed**: `CookieBanner.tsx`, `firebaseAnalytics.ts`, `PREFERENCE_COOKIE_NAME` alias, `GlobalCookie.cookieConsent` field, `migrateLegacyConsent` function, and all legacy translation keys (`banner.header` / `banner.description` / `banner.required.label` / `banner.preference.cookies` / `banner.accept.label` / `config.label` / `config.description` / `required.label` / `required.description` / `required.sub.description`) deleted from web code and backend SQL.
- **Backend mirror removed**: `allowPreferenceCookies` dropped end-to-end from `AuthUserDTO`, `UpdateUserDTO`, `LoginRequest`, `User` entity, `users` table, and all five propagator call sites. Consent state is browser-local. The cross-device sync question is deferred to follow-up work.

**Key architectural decisions (in execution order):**

- **One cookie, browser-scoped, single source of truth.** `og_consent` is the canonical store. The store is a thin cache.
- **Immediate-write everywhere consent is mutated.** Banner, settings page, footer link — all flip state on click, no Save button. The original spec had the settings page on a bulk-save form; mid-feature Igor's UX call moved it to a dedicated immediate-write page.
- **SSR snippet must be synchronous inline `<script>`, not `next/script`.** Consent Mode v2 requires the `gtag('consent', 'default', ...)` call to populate `dataLayer` before any other script can read it. The Next.js + Turbopack framework chunks above the inline script are `async`, so the parser blocks on the inline script first.
- **`sanitizeForSnippet` defense-in-depth.** Runtime validation of the four interpolated signal values before they reach the inline `<script>` or `gtag('update', ...)` call. TypeScript guarantees the shape at compile time, but a future migration or corruption can't reach the wire.
- **No legacy migration shim.** Pre-production: no real users to migrate from the legacy `{ necessary, preference }` cookie shape. Brief 7 stripped the migration path entirely.
- **No backend mirror.** The pre-Consent-Mode-v2 `allowPreferenceCookies` field was duplicate bookkeeping with no server-side consumer. Brief C removed it. Cross-device sync becomes a clean follow-up feature when there's a product need.

**Cross-repo coordination:**

This feature crossed `oglasino-web` and `oglasino-backend` cleanly via per-repo briefs. One process error (brief 8 was drafted as cross-repo by Mastermind in violation of conventions Part 3; engineer correctly refused; split into 8 + 8b) and one rule-misreading (brief 8 framed Rule 3's dedicated-file exception incorrectly; engineer caught mid-session) — both logged to `issues.md` as Mastermind process learnings.

**SSR cache hygiene note (development):**

Late in the feature, Igor encountered an SSR Suspense failure on `/rs-sr` and `/rs-sr/owner/*` (admin worked, `/` worked). Surfaced as `TypeError: controller[kState].transformAlgorithm is not a function` — Node's WHATWG Streams abort cleanup, with the underlying throw masked by Next.js framework frames. Diagnostic web session could not reproduce in a fresh dev-server environment. Resolved by `rm -rf .next` + restart on Igor's machine. Root cause: Turbopack/HMR cache corruption from cumulative `'use client'` module hot-replacements during the feature's many sequential briefs. Not a code defect. Worth keeping in mind for future features that land many briefs touching the same `'use client'` modules in sequence — periodic dev-server restarts avoid the symptom.

**Follow-up work queued in the next-Mastermind handoff brief** (`.agent/handoffs/consent-mode-v2-followups.md`):

- Remove `userPreferenceService` tracking cookie surface and its dependent consumers.
- Persist language and theme as preference cookies.
- Cross-device consent sync (backend storage of `ConsentData`, login-flow sync, logout reconciliation, multi-user/multi-browser support).

**Alternatives considered and rejected:**

- Two cookies (one per logged-in identity, one anonymous) for multi-user/multi-browser support — rejected per Igor as too complex for v1; queued in the handoff brief.
- Backend storage of full `ConsentData` for cross-device sync — rejected for v1 because the platform has no real users yet and the partial-mirror today buys nothing.
- Cookie clearing on logout for shared-browser hygiene — rejected per Igor's call; deferred to the cross-device sync work in the handoff brief.
- Preserving the legacy `{ necessary, preference }` migration shim — rejected because pre-production has no users to migrate.

**Outstanding pre-launch action:** native-translator review of placeholder RS / RU / CNR copy across 25 keys, matching the User Deletion precedent.

---

## 2026-05-20 — Messaging feature shipped (backend + web + rules complete; mobile deferred)

End-to-end user-to-user messaging on Oglasino is production-ready: Firestore Security Rules hardened, web client send flows fixed and made atomic, auto-linkify for URLs in messages, chat list "Load more", admin chat-list receiver bug fixed, public anonymous `/api/public/notification/test` endpoint removed, and a Sunday cleanup cron that anonymizes hard-deleted users' chat data using a `"deleted:<original-uid>"` sentinel. Mobile (`oglasino-expo`) adoption is the only remaining piece, deferred to its own post-launch Mastermind chat per the existing Expo backlog pattern.

**The headline production-defect fix:** the new-chat send batch was failing `permission-denied` at the message-create step because the Firestore rule used `get(/chats/{chatId})` — which evaluates against pre-batch state, where the chat document does not yet exist. Switching all three `chats/{chatId}/messages` rules (read, create, update) from `get()` to `getAfter()` lets the rule see the in-flight chat doc creation in the same batch. The atomic four-write batch (chat root + both sidecars + first message) now lands cleanly. The same rule pass also locked field-shape immutability across the message-update rule and locked the notification create rule to `if false` (Firebase Admin SDK bypasses rules anyway; the prior `request.auth == null || admin claim` rule was a critical trust-boundary violation letting anonymous internet users write notifications under any user's path).

**Key engineering and architectural decisions across the feature:**

- **The chatId algorithm is the contract.** `sorted([uidA, uidB]).join('_')` is computed identically in web (`useChatStore.ts:380, :683`) and backend (`DefaultTrustReviewService.java:114-116`). The convention is documented in `features/messaging.md` §2 and §3.1. Diverging from it silently breaks trust-review. No shared package today; each client re-implements the one-line scheme — promotion to a shared package is post-launch if a third client appears.
- **`getAfter()` is the load-bearing primitive for atomic chat-creation flows.** Brief 1 changed all three message rules; future Firestore-rules work on this surface (or any surface that needs batched parent-and-child creation) should reach for `getAfter()` first.
- **`get('productId', null)` for optional-field immutability.** Plain `==` comparison on a field that may be absent on both sides is engine-version-dependent in Firestore Rules. The `get(field, default)` form yields a defined value on both sides regardless of presence. Codified as a Phase 5 mid-stream amendment to the spec (§4.4) after the rules engineer surfaced the absent-`productId` case during test seeding.
- **Anonymize-not-delete on user hard-delete.** Per the User Deletion feature's precedent (reviews are anonymized, not deleted, per its 2026-05-18 entry), messages from hard-deleted users have their `senderId`/`receiverId` rewritten to `"deleted:<original-uid>"`. The recipient's chat history reads coherently — "deleted user said X, then I said Y" instead of a missing message. The sentinel format cannot collide with any real Firebase UID because Firebase UIDs never contain `:`.
- **Cron candidate-tracking via a dedicated `pending_chat_cleanup` table.** Considered (and rejected) folding the cleanup state into the existing `users` row as a tombstone. Chose a dedicated queue table for clean separation: User Deletion's hard-delete writes one row, the messaging cron consumes from the table, the cleanup is idempotent, per-user failures don't block the batch. The table sits in `V1__init_schema.sql` per the pre-prod schema fold (conventions Part 12).
- **`messaging_cleanup_locks` is a sibling table, not an extension of `user_deletion_locks`.** Brief 5's engineer correctly identified that `user_deletion_locks` is a per-user admin-action table (locks-user-from-deletion), not a job-lock primitive. A sibling table with the proper singleton shape (`UNIQUE (job_name)`) is the right fix. If multiple crons later need singleton locks, a shared `scheduled_job_locks` table can be extracted in a future refactor brief; today only this cron uses the new table. The brief's initial premise that the locks table could be reused was incorrect, surfaced by the engineer at implementation time.
- **Send-failure UX via React-boundary toast, not store-internal toast.** `useChatStore.sendMessage` now rethrows after cleanup; the React caller in `Messages.tsx` awaits and surfaces `notify.error`. Matches the pre-existing pattern for image-upload-failure toasts in `MessageInput.tsx`. The alternative — passing a translator function or callback through the store API — was considered and rejected because next-intl doesn't have a clean non-React access path and pushing translation into the store would have coupled two store actions to one toast call each.
- **Navigation-aware temp state lifecycle.** `tempReceiver` and `tempProductContext` are cleared whenever the user navigates away from `/messages`, but preserved on send failure so retry from the same context works. `ChatsWatcher` was rewritten to use a previous-pathname guard (clear only when transitioning _away_ from `/messages`, not _into_ it from a fresh Start-Message click).
- **Anchor visible-text vs `href` choice on auto-linkify.** A bare `www.example.com` URL displays as `www.example.com` (the user's raw text) but its `href` is the linkify-normalized `http://www.example.com` (so the browser navigates correctly). "What you see is what you click" reads as anti-mask language ("no 'click here' wrappers"), not as a literal `text === href` requirement. Engineer's call, mastermind-confirmed.
- **`users/{userId}` Firestore collection tightened to owner-only.** Documents on this path carry PII (email, fcmToken, displayName, profileImageKey); the pre-fix `allow read: if true` was a privacy leak. No code in `oglasino-web` or `oglasino-backend` reads from this path in Firestore (audits confirmed); the mobile app may break post-rule-change and gets handled in the mobile catch-up chat.
- **RU translations use Latin transliteration by convention.** Brief 4's engineer flagged the entire RU seed file using Latin transliteration as a possible drift; Igor confirmed it's the established convention. Documented here so future RU translation work matches.
- **The `users.blocked` Firestore field is vestigial.** Surfaced during the §4.2 rule-tightening — present on user docs, read by no code. Logged separately for future investigation (`issues.md`-tracked since the audits dated 2026-05-20).
- **The `fcmToken` location concern.** FCM tokens live on `users/{userId}` Firestore docs today. Even with the rule locked to owner-only, this is the wrong storage location — backend already has push-token rows in Postgres. Migration is post-launch work.

**This decision folds in (and effectively obsoletes) the prior `state.md` Risk Watch entry "Messaging fix unblocks User Deletion smoke (current entry, mostly about messaging being broken in local stack)."** Messaging now works; User Deletion's smoke pass against the full deleted/banned/pending-deletion gating surface is the User Deletion feature's responsibility and stays open as a `state.md` Risk Watch item.

**Engineering footprint:**

- Brief 1 — Firestore rules + 23 tests (`oglasino-firestore-rules`), plus a follow-up Brief 1b for the `get('productId', null)` amendment + test 24. 24 tests passing.
- Brief 2 — Web messaging core (`oglasino-web`). Eleven sub-edits: rename + clearing fix + atomic batches + send-failure toast + temp state lifecycle + `getActiveChat` fallback + Load more + `deleteChat` reorder + mark-seen reorder + deleted-user fallback rendering + polish bundle. 154 web tests still passing post-change.
- Brief 3 — Auto-linkify (`oglasino-web`). `linkify-it@5.0.0` chosen over `linkifyjs@4.3.3` based on size (5× smaller) and license. 12 new tokenizer tests.
- Brief 4 — Backend admin + endpoint cleanup + translation seeds (`oglasino-backend`). `getReceiverFromChatDoc` fixed (high-severity per-page receiver bug); `NotificationsControllerTest` removed; two new `MESSAGES_PAGE` keys seeded in four locales; `NotificationCategoryId.MESSAGE` confirmed zero callers and intentionally left as a stub for future push-notification work. 539 backend tests passing.
  - **2026-06-05: superseded** — MESSAGE is now live (set in `DefaultMessageNotificationService`); no longer a zero-caller stub.
- Brief 5 — Sunday cleanup cron (`oglasino-backend`). New `pending_chat_cleanup` and `messaging_cleanup_locks` tables in V1; `DefaultMessagingCleanupService` with `@Scheduled` cron; cross-feature touch into `DefaultUserDeletionService.runHardDelete` (one repository insert). 11 cron + integration tests; first production Sunday is the live smoke (tracked in `issues.md`).
- Brief 6a — `chats.load.more` key seed (backend) + web swap. One translation key seeded in four locales, one component call site swapped.
- Brief 6b — This Docs/QA close-out.

**Reasoning.** The feature shipped end-to-end because the data model was right from the start — chats, sidecars, blocks, deterministic chatIds, separation of Firestore for realtime + Postgres for everything else. The bugs were all in the _implementation_ against that correct foundation. A rewrite was considered (and rejected in this chat's Phase 1 discussion); patching the existing implementation cost less and inherited the worn-in handling of edge cases the audits did not flag.

**Alternatives considered.** Full rewrite of `useChatStore` and the send flow — rejected per Phase 1 discussion; data model is sound, bugs are localized, rewrite cost is ~2-3× for an outcome that's contractually identical. Bundling the admin welcome chat (the "new user gets admin chat that can't be blocked or removed" idea) into this feature — rejected per intake discussion; it composes cleanly on top of the messaging data model and gets its own future Mastermind chat post-launch.

**The closure gate** per conventions Part 3 and bootstrap §7: this Mastermind chat does not close until every drafted config-file edit produced in this chat has been applied to disk by Docs/QA. The Brief 6b application is the gate. Once Igor confirms commit, the chat closes; the next chat (the admin welcome chat, or the mobile catch-up, or something else) starts fresh.

---

## 2026-05-20 — Google Analytics v1 split into two features: Consent Mode v2 foundation, then GA4 instrumentation

Two web audits (GA4 discovery, consent audit) surfaced that the existing cookie banner is structurally one step short of what Consent Mode v2 needs: two-category model (`necessary`, `preference`) instead of Google's four-signal shape; no SSR-aware default-consent emission; no Reject-all CTA; no anonymous re-entry to the banner; consent gates nothing in code today. GA4 v1 cannot ship until those gaps close.

The work is therefore two features, executed in order: **Consent Mode v2** (`features/consent-mode-v2.md`) ships first as the foundation; **Google Analytics v1** (`features/google-analytics-v1.md`) ships second on top of it. Both specs are authored together so the GA4 contracts depending on consent decisions (cookie name, `ConsentData` shape, SSR snippet location) are locked at the same time the consent feature spec is locked.

**Key decisions (10) folded into the two specs.**

1. **Two features, consent first.** GA4 v1 is blocked on Consent Mode v2 shipping.
2. **All four Consent Mode v2 signals modeled** (`analytics_storage`, `ad_storage`, `ad_user_data`, `ad_personalization`), with the three `ad_*` signals permanently `denied` per the Privacy Policy's no-marketing-pixels commitment. Modeling all four now means future marketing-tooling decisions don't need a schema migration.
3. **Banner UX: twin Accept-all / Reject-all primaries with a Customize expander.** Regulator-defensible; current single "Accept selected" CTA replaced.
4. **Footer "Manage cookie preferences" link reopens the same Drawer** rather than routing to a dedicated page. One source of truth for the consent decision.
5. **Consent storage splits into its own `og_consent` cookie**, leaving `globalCookie` for non-consent prefs. SSR read becomes a single cookie read + parse rather than entangled with card-size and language. One-time client + SSR migration fallback for ~60 days reads the legacy `globalCookie.cookieConsent` if present; cleanup chore scheduled post-migration.
6. **`sign_up` vs `login` discriminator** via a `wasRegister: boolean` returned from `syncUserToBackend`. Preserves the listener-is-sole-hydrator contract in `UseTokenRefresh.tsx`. Two alternatives rejected: firing `sign_up` from `registerUserFirebase` (would land without `user_id`); a parallel one-shot cell (duplicates state).
7. **`select_promotion` dropped from v1.** Codebase has no promotion surface today; aspirational events in the schema confuse the next reader. Event returns when a promotion surface lands.
8. **`page_view` does NOT fire on filter change.** Filter mutations on catalog/home produce URL changes via `router.replace`; firing `page_view` per slider drag would inflate the metric. A separate `filter_change` event is added to v1 in its place.
9. **Stale `2026-05-19-oglasino-web-ga4-discovery-1.md`** in `oglasino-web/.agent/` ignored — Igor confirmed it's stale. Engineer's session counted as `<n>=2` per conventions Part 5 archive-pointer rule. The stale file is archived at GA4 v1 feature close by Docs/QA.
10. **Card-size and language cookies move behind the `preference` category.** They are written today regardless of consent — a pre-existing gap the consent feature closes. Logged as a Risk Watch item in `state.md` and closed when consent ships.

**Event set, v1.** Final composition is 12 events plus the automatic `page_view`: `sign_up`, `login`, `product_view`, `product_create_started`, `product_create_completed`, `contact_seller_clicked`, `message_sent`, `search`, `view_search_results`, `filter_change`, `exception`, `form_submit_failed`.

**Cross-repo seams.** Consent: backend keeps the single-boolean `AuthUserDTO.allowPreferenceCookies` mirror; widens later only when a future feature needs server-side awareness of analytics consent. GA4: no backend changes; no Firestore Rules changes; no router-worker changes. Translation seeds for the consent feature follow the conventions Part 6 Rule 3 large-feature exception (18 keys × 4 locales in a dedicated SQL file).

**Legal review handoff.** Both features include "Docs/QA notifies legal-drafts chat" in their definition-of-done. Consent ships with new Reject-all CTA, new category model, new footer entry point — privacy-policy-draft.md may need paragraph updates. GA4 ships with analytics processing live — same notification trigger.

**Alternatives considered.**

- One feature combining consent + GA4 (rejected — consent has user-visible UX and legal-surface implications independent of GA4; ships even if GA4 slips; bundling obscures decisions).
- Analytics-only Consent Mode v2 signal model (rejected — locks in a schema change if marketing tooling ever lands).
- Three banner CTAs visible at all times (rejected — clutters the banner; standard pattern is twin primary + customize expander).
- Dedicated `/cookie-preferences` page (rejected — two surfaces showing the same data, drift risk).
- Keep consent nested in `globalCookie` (rejected — bigger SSR parse on every request, mixed security posture, cookie name obscures intent).
- Fire `sign_up` from `registerUserFirebase` directly at primitive completion (rejected — `useAuthStore.user` is null at that moment, so the event lands without `user_id`).
- Keep `select_promotion` and instrument it on `ExtraProductsComponent` cards (rejected — those aren't promotions, stretching the term pollutes the analytics).
- Fire `page_view` on every URL change including filter mutations (rejected — inflates session event count and skews engagement metrics).

---

## 2026-05-19 — Backend → web cache revalidation: best-effort fire-and-forget pattern

Backend POSTs `{"tags": ["user:<id>"]}` to web `/api/revalidate` after a user state transition commits, so other viewers of the affected profile see fresh `UserInfoDTO` inside the 5-minute SSR `revalidate: 300` window. Implemented via `UserStateChangedEvent` + `@TransactionalEventListener(AFTER_COMMIT, fallbackExecution = true)` + `DefaultWebRevalidationService`. Failure is logged and swallowed. Shared-secret header authentication (Option A) chosen over IP allowlist (Option B) or no-auth (Option C) because shared-secret is the simplest defensible model for server-to-server inside a hosted environment.

**Reasoning.** Frontend-driven revalidation (the existing self-deletion / self-restoration flow) covers the actor-side cache; the missing piece was viewer-side caches when _another_ user is the actor (admin bans, hard deletes, scheduled deletions). Best-effort is correct because the 5-minute TTL is the upper bound on staleness — a failed revalidate is recoverable without operator action.

**Alternatives considered.** Explicit post-commit method calls (Option B per the brief) — rejected because every new state-transition path would need to remember to call the client; event publication composes with future paths without code edits at the consumer.

---

## 2026-05-19 — Banned-state UI visibility design

When a user is banned (`disabled = true`), the system mirrors the existing pending-deletion content-hiding posture: products flip to `INACTIVE` with `deactivated_by_system = true`, public profile returns 404, messaging is gated on the counterparty side, and the `UserInfoDTO.state` field returns `BANNED`. Banned wins over pending-deletion in the composition rule (`UserState.resolve(disabled, deletionStatus)`); admin unban during pending-deletion does NOT restore products (only when `deletionStatus = ACTIVE`).

**Reasoning.** The pre-B2 design banned a user without hiding their content — meaning banned-users' listings stayed reachable via direct URL, and counterparty messaging stayed enabled. Symmetric content-hiding closes the privacy gap while preserving the pending-deletion grace period semantics (unban doesn't auto-restore deletion intent).

---

## 2026-05-19 — Backlog-triage closed for User Deletion feature

Round-3 handoff identified 21 backlog items (B1–B21). This pass closed: B1 (push-token detach), B2 (banned-state UI), B3 (auth-listener consolidation, B3-a path), B6 (filename typo), B8 (cookie-write conditional), B10 (cache invalidation), B11 (InfoDialog dismissal), B12 (cross-browser ban catch), B14 (audit log for unban), B15 (dead code in UsersFilterRequestDTO), B16 (Javadoc/impl mismatch), B17 (per-row lock query N+1), B18 (projection cardinality smell), B20 (admin Users detail page state indicators). One item out of scope: B19 (mobile parity — separate Mastermind chat post-merge). Closed via 9 backend sessions + 6 web sessions on `dev` branch.

---

## 2026-05-19 — Cache-invalidation profile fix (Next.js 16)

The `/api/revalidate` route handler used `revalidateTag(tag, 'default')`, which under Next.js 16 is the long-cache profile (renews tag lifetime under the default profile), not immediate invalidation. Backend revalidation calls succeeded at HTTP level but did not invalidate the SSR cache; users saw stale `UserInfoDTO` for up to 5 minutes after state transitions. Fixed by switching to `revalidateTag(tag, { expire: 0 })` — `CacheLifeConfig` with zero expiry forces immediate invalidation.

**Reasoning.** The round-3 handoff named this as a project-wide bug. This pass fixed the `/api/revalidate` site. Other call sites (`oglasino-web/app/actions/cacheActions.ts` and possibly others) still need the same fix — tracked as separate audit follow-up.

---

## 2026-05-19 — Pre-production schema fold for V1 migrations

All schema additions and renames during the pre-launch period edit `V1__init_schema.sql` in place, rather than creating new `V2__*.sql` migration files. This applies to all tables, columns, indexes, and constraints touched by the User Deletion feature. The convention will end when production deploys; from that point onward, new migrations are append-only.

**Reasoning.** The repo has no production deploy yet; rebuilding the schema from a single V1 file on every dev/stage reset is simpler than managing a growing migration chain. Documented in conventions.md.

---

## 2026-05-18 — Image Router registered as the seventh engineer repo

A new repo `oglasino-image-router` holds a Cloudflare Worker that gates image PUT/GET against R2 for Oglasino's image pipeline. The Image Router engineer agent is now the seventh engineer agent (eighth agent total counting Mastermind).

Stack: TypeScript + Cloudflare Workers + Wrangler 4 + Vitest 3 with `@cloudflare/vitest-pool-workers`; JOSE 6 for JWT verification; Node 22+. The worker has internal structure (handlers/, lib/, types) and is more substantial than the existing `oglasino-router` which is a single file. Deployment via `wrangler deploy` (any env) is forbidden for the agent.

The Image Router is a separate Worker from `oglasino-router`. They share a stack family (Cloudflare Workers + TypeScript + Wrangler) but have different responsibilities: `oglasino-router` is the edge boundary doing domain matching, maintenance gating, and origin forwarding; `oglasino-image-router` gates image upload/retrieval against R2 with JWT-verified access. They have different care areas, different bindings, and different deploy lifecycles. Two repos, two agents.

The `CLAUDE.md` for this repo is marked provisional. A full audit session is planned as the first work in the repo, after which the `CLAUDE.md` is rewritten against the actual code rather than against initial inference.

Updates to meta-documents: Part 2 (repo layout), Part 3 (agent list, deploy ban list), Part 4 (cleanliness commands), Part 9 (stack table), Mastermind bootstrap Phase 5 brief-order, DevOps bootstrap Phase 2 check surfaces.

**Reasoning.** The image pipeline is a distinct security boundary from the edge router. Mixing image-upload gating into `oglasino-router`'s code would compromise both — the edge router's small-file-fast-changes profile, and the image worker's JWT-verification trust-boundary discipline. Separating them keeps each Worker focused on one job.

**Alternatives considered.** Fold the image worker into `oglasino-router` as a second handler (rejected — different deploy lifecycle, different bindings, different trust profile, would force a sprawling combined `CLAUDE.md`). Have the existing Router agent own both repos (rejected — agent-per-repo discipline is already in place across the system; one agent owning two repos is a precedent we don't need and don't want).

---

## 2026-05-18 — Firestore Rules registered as the sixth engineer repo

A new repo `oglasino-firestore-rules` holds the Firestore Security Rules separately from `oglasino-backend`. The Firestore Rules engineer agent is now the sixth engineer agent (seventh agent total counting Mastermind).

Stack: Firestore Security Rules language for rules; TypeScript + Vitest + `@firebase/rules-unit-testing` v4 for tests; Firebase Tools 13 for the local emulator; Node 20+. Deployment via `firebase deploy --only firestore` is forbidden for the agent (same deploy ban as every other engineer agent).

The existing conventions Part 3 rule "No Firestore security rule edits without an explicit instruction in the brief" stays load-bearing; it now disambiguates by naming `oglasino-firestore-rules` as the rules' home. The Backend agent does not edit Firestore rules — that's the Firestore Rules agent's exclusive lane.

Updates to meta-documents: Part 2 (repo layout), Part 3 (agent list, deploy ban list, hard-rules note), Part 4 (cleanliness commands), Part 9 (stack table), Mastermind bootstrap Phase 5 brief-order, DevOps bootstrap Phase 2 check surfaces.

**Reasoning.** Firestore rules deserve their own deploy lifecycle separate from the Spring backend's; the previous arrangement of rules-in-backend forced the rules to ride the backend's deploy cadence. Splitting also enables independent rule-tests via the emulator without spinning up a Spring context.

**Alternatives considered.** Keep rules in `oglasino-backend` and have Backend agent edit them when authorized (rejected — Backend agent is Spring/Java only, the JS/TS test fixtures and Firestore Rules DSL fall outside that stack discipline). Have Backend edit `oglasino-firestore-rules` under a narrow cross-repo exception (rejected — same reason; the JS test code makes this not a true narrow exception).

---

## 2026-05-18 — Part 4a (Simplicity) enforcement tightened

Part 4a has been in conventions since 2026-05-13 but the recurring failure mode is engineers ticking "confirmed" on the Conventions check without engaging with the rule. The fix tightens enforcement, not the rule itself.

**What changed.** Part 4a gains an "Enforcement" section requiring structured evidence in every session summary, in three categories: complexity added (with justification), complexity considered and rejected, complexity simplified or removed. Each category accepts "nothing" but must be explicitly written. The Conventions check entry for Part 4a is replaced with a pointer to the evidence, removing the one-word "confirmed" option. The session summary template carries a new required block in "For Mastermind" capturing the three categories.

Mastermind verdicts a session as REVISE if the structured evidence is absent or if the evidence reveals undeclared complexity additions.

**Reasoning.** Engineers self-attesting to "confirmed" on a complex rule is a known weak enforcement pattern. The Conventions check has succeeded at catching brief-vs-conventions drift on rules that have a specific factual answer (Part 6 namespace, Part 7 wire shape). It has been weakest on Part 4a, which requires judgment. Asking engineers to _name_ their decisions — what they added, what they didn't, what they removed — raises the cost of skipping the rule and gives Mastermind something concrete to verdict on.

**Alternatives considered.**

- Add SOLID, DRY, KISS, Separation of Concerns, Modularity, Encapsulation as named principles in conventions (rejected — adds vocabulary, not enforcement; failure mode is engineers ticking boxes, not engineers lacking vocabulary; six new principles to tick instead of one).
- Make Part 4a absolute (ban new abstractions without two existing callers; ban configuration for single-value cases) (rejected — already considered and rejected in the original Part 4a decision; absolute rules kill good engineering instincts along with bad).
- Add Boy Scout Rule and YAGNI to conventions (rejected — Boy Scout directly contradicts Part 4b which is already in conventions; YAGNI duplicates Part 4a's configuration guideline).
- Add the principles list as a vocabulary section alongside enforcement (rejected — Option B from the chat discussion; deferred until enforcement-only proves insufficient. Lower-cost path tried first).

---

## 2026-05-18 — QA Preparation: content phase rules and patterns established

Sessions 8–19 authored every in-scope portal page topic plus the six in-scope feature/flow topics (17 in total). The simplification sweep across batches 1–5 and the final cleanup brief (this session) settled a stable set of rules and patterns for QA-topic content. Captured here so future authoring — owner-dashboard topics and the deferred `report-flow` — inherits them.

**1. Trust-boundary findings.** Two flow-topic audits produced full chain-of-evidence trust-boundary content too dense for the topic body and routed instead as verification asks for separate features:

- `start-message-flow` payload (session 12). Firestore rules are the sole boundary for `chats/<id>` writes. The rules must enforce: (a) the authenticated `request.auth.uid` is one of the two participants on the chat document being created or updated; (b) the `withUser.id` field on the request matches the other participant — not a forged third user; (c) the `productId` field, when present, references a product whose owner is the other participant — preventing an attacker from forging an "interested in <other-user's-product>" context; (d) the message body fields cannot be written by the recipient on behalf of the sender. Routed to the Firestore-rules-tightening backlog feature.

- `follow-flow` payload (session 13). The Spring controller `POST /secure/follow/<userId>` is the boundary. The controller must enforce: (a) the path-variable `userId` resolves to an existing, non-deleted user — preventing follows against ghost ids; (b) the `userId` is not the authenticated caller's own id — preventing self-follow rows; (c) repeat follows from the same caller-target pair are either idempotent or rejected — preventing duplicate-row growth on the join table. Routed as a one-shot read-only audit in `oglasino-backend`.

**2. Trust-boundary check model.** The depth standard set in sessions 10, 11, 13: trust-boundary checks must trace the chain — client supply → server derivation → identity layer → authorization. Each verification ends with a single labelled conclusion: "clean," "clean conditional on X," or "concern/violation," with named (a)-(c) checks for the verification surface. Engineers writing trust-boundary checks at this depth is the standard for Part 11 going forward.

**3. Trust-boundary content belongs in audits and decisions, not topic body.** Precedent set in session 17; applied at scale in session 18. The topic body is for testers; the chain-of-evidence — request shape, claim flow, server checks — lives in session summaries and `decisions.md`. Topics carry the _consequence_ of trust-boundary findings (a Known-issue pitfall, a routed verification task) but not the chain itself.

**4. Flow-topic structural rule.** Established session 11: `optionsControls` is used when a flow has concrete surfaces to enumerate; skipped when purely behavioural. All four authored flow topics (`category-navigation`, `start-message-flow`, `follow-flow`, plus the future `report-flow`) used `optionsControls`. The schema's other four optional arrays (`howToUse`, `whatToExpect`, `pitfalls`, `qaChecklist`) carry the behavioural content.

**5. Flow-topic `route` field rule.** Established this brief: primary-surface-only. The `route` string carries the dominant entry surface; additional surfaces are enumerated in `optionsControls`. No schema change — the field stays `string`. Applied to `category-navigation`, `start-message-flow`, `follow-flow` in session 19.

**6. Stale-defect-bullet rule.** Established sessions 14–15: if the underlying fix has landed, the topic bullet is cut entirely — no "this used to behave differently" footnote. Applied across batches 1–5 and confirmed no-op in session 19's final stale-defect verification pass.

**7. "Known issue" pitfall pattern legitimized.** Bugs that materially affect a QA cycle before they are fixed may appear in the topic as a "Known issue" pitfall cross-referenced to the matching `issues.md` entry by date and title. Bugs unrelated to the topic's surface stay in `issues.md` only and are not surfaced in the topic. Precedent: `privacy-page`, `terms-page`, `global-header-search`, `category-navigation`. Promoted to standing pattern in this brief: `catalog-page` (slug failure modes), `product-page` (Serbian tooltips), `category-navigation` (Kategorije Serbian label), `follow-flow` (isFollowingCurrent seed), `messages-page` (silent text-send failure, chat-list cap). All five share the same prose shape: "Known issue: <symptom>. Tracked in issues.md (<date> entry — <title>); until fixed, <tester guidance>."

**8. Triage outcomes.** The five-topic triage in chat 2 decided six feature/flow topics in scope: `global-header-search`, `portal-config-dialog`, `category-navigation`, `start-message-flow`, `follow-flow`, plus the deferred `report-flow`. Dropped during triage: `filters`, `product-list-and-pagination`, `favorite-a-product-flow`, `cross-baseSite-redirect-flow`. The dropped list is recorded here so the rationale survives — each was judged either covered adequately inside an existing page topic (`filters` and `product-list-and-pagination` are inside `home-page` and `catalog-page` already; `favorite-a-product-flow` is inside `product-page` and `favorites-page`) or not material as a QA topic (`cross-baseSite-redirect-flow` is a single-redirect mechanic, not a flow).

**9. Brief-authoring lessons.**

- Entry-point candidate lists are unverified hints, not facts. Briefs that name candidate entry points must phrase them as "confirm in code which actually exist" rather than as assertions. Surfaced session 12 (the brief named a messages-page kebab entry that did not exist) and session 13 (only 1 of 3 named follow-handoff candidates was real).

- Stop pre-stating `<n>` in briefs whenever prior archived sessions exist. Engineers determine `<n>` from their own `.agent/` folder. Conditional on the conventions Part 5 amendment in this same session (block 7A below) — once Docs/QA leaves an archive-pointer stub in the source repo's `.agent/` folder, the engineer's own count remains correct after Docs/QA has archived-and-deleted prior session files.

**10. Session-summary "Config-file impact" pattern.** Sessions 13–18 used this section consistently — a short index, listed for each of the four config files as either "no change" or "draft below for Docs/QA." Formalised in conventions Part 5 in this session (block 7B below).

**Reasoning.** Rules established during the content phase needed a durable home outside of session-summary archives. Future authoring (owner-dashboard topics; the deferred `report-flow`) inherits this rule set without re-litigating per topic.

**Alternatives considered.** Recording each rule as a separate `decisions.md` entry at the time it was established (rejected — would have produced ten thin entries; the consolidated form is cheaper to read and more durable). Folding all ten into `meta/conventions.md` directly (rejected — these are feature-specific patterns, not project-wide conventions; the decisions log is the right home, conventions Part 5 captures only the engineer-template structural rule).

---

## 2026-05-18 — User Deletion feature shipped (backend complete; frontend pending separate session)

Backend implementation of self-service account deletion with 7-day grace period, scheduled hard-delete cron, Firebase orphan reconciliation, ban-with-reason flow, and 12-month re-registration prevention is complete and patched per PR review. Frontend work proceeds in a separate Mastermind chat using the canonical spec at [`features/user-deletion.md`](features/user-deletion.md) as authoritative.

**Key engineering decisions captured during the feature:**

- **Pre-production schema fold.** `V1__init_schema.sql` was edited in place rather than adding a new Flyway migration. The decision is documented at the top of V1 and was scoped tightly: no preserved environment had V1 applied, so the fold was safe. Post-launch, all schema changes go in new Flyway migrations as normal.
- **Cron expressions live in `application*.yaml`, not the `Configuration` table.** `@Scheduled(cron = ...)` resolves at bean-construction time from the Spring Environment and cannot read from the runtime `Configuration` cache. The three cron knobs (`user.deletion.hard.delete.cron`, `user.deletion.audit.purge.cron`, `user.deletion.firebase.reconciliation.cron`) sit in YAML; the other twelve user-deletion knobs (batch sizes, retentions, the reconciliation enable flag, the cascade threshold) live in `Configuration` and remain DB-tunable. See corrected spec §3.9 and §7.
- **`user_deletion_requests` rows are intentionally ephemeral.** `ON DELETE CASCADE` on the FK to `users(id)` deletes the request row when the user row is deleted (Mastermind Option B). Spec §8.1 step 15 is removed (no row to update); the audit log carries the durable record. The dead `deleteCompletedOlderThan` repository method was removed in Phase 7. See corrected spec §8.1 and §9.3.
- **Translation seeds for large features may use dedicated SQL files.** The user-deletion feature seeded 33 keys per language across 7 namespaces; doing this inline would have meant ~14 multi-line edits across 8 dense (1000+ line) existing seed files. Engineers shipped four dedicated `0004-data-user-deletion-translations-{EN,RS,RU,CNR}.sql` files instead. Codified as an exception to Part 6 Rule 3 (10+ keys across multiple namespaces threshold).
- **`TransactionTemplate.executeWithoutResult` is blessed alongside `@Lazy self` for cache-aware self-calls.** The user-deletion feature used `executeWithoutResult` for the explicit transactional-boundary split in `DefaultUserDeletionService.runHardDelete`; the existing `@Lazy self` precedent from `DefaultBaseCurrencyService` (decisions.md 2026-05-14) handles caching-proxy-aware self-calls. Both patterns are blessed; mixing them in one method is a smell. Codified as new Part 13.
- **Partial-index `WHERE` predicates must be IMMUTABLE.** Postgres rejects partial indexes whose predicates call STABLE or VOLATILE functions (error 42P17). The `user_deletion_locks` partial-index predicate had to be dropped during V1 work; the active-vs-expired check now runs at query time. Codified as new Part 12.
- **`getBooleanConfig` (silent-false on missing) is the right pattern for destructive batch toggles.** The Firebase reconciliation job uses `getBooleanConfig`, not `getRequiredBooleanConfig` — silent-false is the safer fallback for a destructive cron job than throwing at tick time. The Configuration seed sets the key explicitly so the default never applies in normal operation. See corrected spec §9.2.
- **`disapproved` reviews authored by the deleting user are deleted at hard delete.** The original spec covered approved (anonymize), pending (delete), and reviews-about (delete) but was silent on the disapproved-authored case; PR review caught that `userRepository.delete(user)` would FK-fail without it. Patch session 6 added `ReviewRepository.deleteByReviewerIdAndDisapproved`. See corrected spec §3.3 and §5.

**This decision folds in the prior backlog item "Account-disabling & token-revocation enforcement"** from `state.md`. The veto plumbing landed inside user-deletion: backend `FirebaseAuthFilter` reads `authData.disabled()` (Phase 6), `setDisabled(true)` is called on ban (Phase 5), `revokeRefreshTokens` is called on ban AND on Day-0 deletion request (Phase 5 / patch session 6). The backlog entry is removed by this decision.

**PR review** surfaced 4 critical+high backend bugs (cron-restore race, disapproved-authored review FK violation, audit-log trigger-overwrite, audit-log purge before close); all patched in backend session 6 with focused tests. Backend test suite ends at **444 passing, 0 failing**. Frontend Mastermind chat opens with the corrected spec as the authoritative input.

**Reasoning.** Backend is the platform-level deliverable that frontend and mobile build against; flipping the feature to a shipped state on the backend half unblocks the frontend chat and the future mobile adoption. The corrections above were uncovered by engineers during implementation and by the PR review; capturing them in the canonical spec (rather than only in session summaries) is what keeps future readers from re-litigating decisions that are already settled.

**Alternatives considered.** Hold the shipped flag until frontend lands (rejected — the spec is authoritative for frontend's chat input regardless of state.md status; flipping when backend is done is the established pattern, with frontend tracked via the platform-adoption section on the feature spec). Park the spec corrections as `issues.md` entries (rejected — the corrections are spec-text changes, not bugs; the spec is the canonical surface and stale spec misleads the next reader). Codify the schema/Spring rules inside the user-deletion spec only (rejected — both rules apply repo-wide; conventions Parts 12 and 13 is where they're discoverable to future engineers without context on this feature).

---

## 2026-05-17 — Docs/QA is the sole writer of the four config files; session-closure gate added

Three structural changes propagated across `conventions.md`, the four chat bootstraps, and the five CLAUDE.md files. The configuration audit close-out surfaced enough drift (auth-description issue marked `fixed` while conventions still carried the wrong text; the new Docs/QA `.agent/` archival permission layered without removing the older blanket "never edits other repos" rule; the conventions/CLAUDE.md "four agents" count contradicting the Router agent's own self-identification as a fifth) that the fix is structural, not a typo sweep.

**Docs/QA writes; everyone else drafts.** `conventions.md`, `decisions.md`, `state.md`, and `issues.md` are now written by Docs/QA only. Mastermind, bug chat, DevOps chat, and legal drafts chat draft text in their session output and hand the draft to Igor. Igor briefs Docs/QA in a separate session. Docs/QA applies. Igor commits. Engineer agents have read access via the sibling docs repo; they never write.

Docs/QA may make small independent fixes — typos, broken links, stale dates, formatting consistency — without an upstream draft. Anything substantive (a new rule, a contradiction resolution, a status flip, a new entry) requires an upstream drafter. When in doubt, Docs/QA treats the change as substantive and asks Igor.

This **flips** the 2026-05-13 Docs/QA hard rule "Never edits `decisions.md` or `meta/conventions.md`." That rule was load-bearing for the period when Igor was the sole writer of config files. In practice it produced a recurring drift pattern: the canonical example is the 2026-05-14 `issues.md` entry on the Firebase-auth Parts 9 / 11 wording — drafted by Docs/QA in a session summary, marked `fixed` in `issues.md`, but the conventions edit itself never landed because no agent had write authority and the manual apply step got skipped. The new model removes that gap: the draft and the apply are now two explicit steps with a named agent responsible for each.

**Session-closure gate.** No chat closes with pending config-file drafts. If a Mastermind / bug / DevOps / engineer / legal-drafts session produces a draft for any of the four config files and that draft has not been applied by Docs/QA, the originating chat does not mark itself complete. "Drafted but pending" is not a valid closure state. The Mastermind feature spec at `features/<slug>.md` is held to the same gate — Phase 4 does not end until the spec file exists on disk.

**Conventions Part 3 was rewritten** to capture both rules in one place: the agent list (six agents now, with Router added — see the next entry), the cross-repo edit rules including the narrow Docs/QA `.agent/` exception preserved from the 2026-05-15 decision, the new "Config-file writes" section, the new "Session-closure gate" section, and the updated Mastermind and Docs/QA hard rules.

**Conventions Part 5 was rewritten** to add the "Config-file impact" section to the engineer session-summary template — a short index at the end of every summary that lists, for each of the four config files, either "no change" or "draft below for Docs/QA." Makes pending edits scannable without scrolling through "For Mastermind." Part 5 also disambiguated the `<n>` indexing prose ("first session for a slug starts at `<n>=1`, producing a filename ending in `-<slug>-1.md`"), resolving the earlier contradiction between "starts at `1`" and "starts at `-1`."

**Reasoning.** The drift pattern was real and recurring. Three layers of "never edits" / "may now edit `.agent/`" / "drafts but doesn't write" had accumulated in conventions without the old layer being narrowed. The cost of leaving it was a future agent reading the wrong layer first. The cost of fixing it was one structural pass plus the closure gate, which costs nothing per session and prevents a class of bug worth more than the rule's overhead.

**Alternatives considered.** Keep the existing model and just enforce "Igor applies" more carefully (rejected — that's been the model and it's the model that produced the drift). Give Mastermind direct write access to `decisions.md` and `state.md` (rejected — Mastermind doesn't run in the docs repo and would need a separate session anyway; routing through Docs/QA is cheaper and keeps one agent accountable for what hits disk). Make every drafting chat call its own Docs/QA session inline (rejected — would force a context switch mid-chat and bloat the chat that produced the draft; the draft-then-Docs/QA-session-later flow is what the closure gate is designed for).

---

## 2026-05-17 — Router acknowledged as the fifth engineer agent

The `oglasino-router` repo has a real CLAUDE.md and runs as a Claude Code engineer agent, but every other meta-document treated the system as having four engineer agents (Backend, Web, Mobile, Docs/QA). Conventions Part 2's repo layout omitted `oglasino-router/`. Conventions Part 3 listed five agents but only the four-engineer-agent shape. Four of the five CLAUDE.md files said "you are one of four engineer agents" while `oglasino-router/CLAUDE.md` correctly said five. DevOps Phase 2's propagation map was the only meta-document that named the router repo explicitly.

**The Router agent is now first-class** in every meta-document. Conventions Part 2 lists `oglasino-router/` in the repo layout. Conventions Part 3 names six agents (Mastermind + five engineer agents) with the Router agent's responsibility, branch, and hard rules captured alongside the other engineer agents. The five engineer-agent CLAUDE.md files all say "five engineer agents (Backend, Web, Mobile, Router, Docs/QA)." Mastermind bootstrap Phase 5 mentions router as a possible brief slot in the typical brief order (backend → web → router if affected → mobile → docs cleanup).

**Conventions Part 4 gained the Router lint/test commands** (`npm run lint`, `npm test`). Part 9's stack table gained an "Edge" row covering the Cloudflare Worker / TypeScript / Wrangler 4 / Vitest stack. Part 8 gained a bullet calling out the worker as the edge boundary for the platform.

**The Router agent's hard rules** in conventions Part 3 capture the deploy bans (`wrangler deploy`, `wrangler dev` against production resources, live KV writes outside explicit brief authorization) and the four critical-care areas the worker depends on (maintenance matrix, fail-open KV reads, admin-request regex, `redirect: "manual"` forwarding, stage `noindex` header). The latter four are documented in detail in `oglasino-router/CLAUDE.md` and not duplicated in conventions.

**DevOps bootstrap Phase 2** now references conventions Part 2 for the repo inventory rather than re-enumerating repos inline. The 2026-05-15 maintenance-gate split decision retroactively validates this — that work touched four repos (router, backend, web, droplet) and the DevOps chat managed the propagation correctly because Phase 2's map shape worked. The change is to stop duplicating the repo inventory in the bootstrap.

**Reasoning.** The Router agent's absence from conventions was a coverage gap, not a policy decision. The 2026-05-15 maintenance-gate work showed the router repo is fully integrated into DevOps and Mastermind chats; only the conventions file (and the four downstream CLAUDE.md files) lagged. Same drift pattern as the Firebase-auth wording or the Gradle/Maven mismatch — a rule that contradicts the operational reality. Closed at the conventions level so future agents inherit the correct count.

**Alternatives considered.** Document the Router agent in DevOps bootstrap only and leave conventions agnostic (rejected — engineer-agent rules apply to the Router agent the same way they apply to Backend/Web/Mobile/Docs/QA; the bootstrap can't carry the canonical rules). Treat the Router as infra rather than as an engineer agent (rejected — it has a CLAUDE.md, gets briefs, runs Claude Code sessions, produces session summaries; it's an engineer agent in every operational sense).

---

## 2026-05-17 — Mobile catch-up session slug discipline; Expo backlog table added to state.md

Mobile is multiple features behind backend and web (product-validation, image pipeline, backend-calls reduction). The catch-up sessions need a slugging rule and a tracker.

**Slug discipline.** Mobile adoption sessions are slugged **per feature**, not per time window. A session adopting `product-validation` on mobile is `oglasino-expo-product-validation-1`, the next is `-2`, etc. Each adopted feature carries the same slug it has in `features/<slug>.md`. Time-window slugs like `mobile-catchup-2026-05` are not used.

This matches the existing `(repo, slug)` numbering rule in conventions Part 5 — that rule is designed around features, not time windows. A "catch-up" slug would have been a time-window artifact that doesn't compose with the rest of the system. Per-feature slugs keep mobile consistent with how backend and web are numbered, and let each adopted feature's `## Platform adoption` section in `features/<slug>.md` carry its own session history.

If mobile turns out to need a focused multi-feature session that doesn't map cleanly to one feature (a shared utility, a cross-cutting refactor), the Mastermind chat that opens it decides the slug at that point. The default is per-feature. Switching to a different shape requires a Mastermind decision.

**Expo backlog table.** `state.md` gains a new section, **Expo backlog**, between "Backlog" and "Decisions log." Tracks every feature that is `web-stable` or `shipped` on backend/web but not yet adopted on mobile. Docs/QA maintains it: every time a feature reaches `web-stable` or `shipped`, append a row; every time mobile adopts a feature (the slug reaches `mobile-stable` on the feature spec), remove the row or flip its status. Per the new Docs/QA write authority — small mechanical maintenance like this doesn't need an upstream draft.

Table columns: Feature, Web/Backend status, Mobile status, Adopted in session, Notes. "Mobile status" is one of `not-started`, `in-progress`, `adopted`. The table is seeded retroactively with three rows for product-validation, image pipeline, and backend-calls reduction.

**Reasoning.** Two problems were solved together. The first was the slug question — when the mobile catch-up Mastermind chat opens, the engineer agent needs to know how to name the session, and the answer is "the feature's existing slug, with `<n>` starting at 1 in the Expo repo." The second was visibility — the risk watch entry "Mobile is multiple features behind" was the only place that captured the backlog, and risk-watch prose isn't structured enough to track adoption status per feature. A table gives Igor an at-a-glance view of mobile's debt without reading the full session log.

**Alternatives considered.** One slug per catch-up window — `mobile-catchup-2026-05` (rejected — composes badly with `(repo, slug)` numbering and obscures which feature each session adopted). Punt the slug question to the Mastermind chat that opens the catch-up (rejected — predictable enough to pre-decide; the cost of not deciding is a one-question pause at the start of every mobile chat). Keep the backlog as risk-watch prose only (rejected — gets buried, and adoption status doesn't render cleanly in prose; the table is the natural shape for "feature × adoption status").

---

## 2026-05-15 — Docs/QA may write to engineer repos' `.agent/` folders for session archival

The conventions Part 3 "No cross-repo edits" rule is amended to permit Docs/QA to write within other repos' `.agent/` folders for the specific purpose of session archival. Docs/QA may copy named session files into `oglasino-docs/sessions/` and delete the source files from engineer repos after verifying the archive copy is identical.

**Reasoning:** the archival flow needs source cleanup. Without it, named session files accumulate in both locations and there's no clear "the archive is the canonical copy" boundary. Forcing Igor to delete sources manually defeats the point of having Docs/QA archive at all. The risk of granting write access is narrow because the permission is scoped to one folder name (`.agent/`) and one operation type (archival cleanup).

**Alternatives considered:** keep the rule as-is and let Igor delete sources manually (rejected — defeats the archival workflow). Grant Docs/QA broader write access to engineer repos (rejected — opens the floodgates to scope creep, no concrete need). Move archival to each engineer's responsibility (rejected — engineers can't see other repos' archive, and the centralized archive is the point of having Docs/QA at all).

---

## 2026-05-17 — Redis cache TTLs: five removed, two kept, helper signature reformed; in-memory translations layer brought under the precedent

Two read-only investigations (`sessions/2026-05-17-oglasino-backend-base-site-ttl-investigation-1.md` and `sessions/2026-05-17-oglasino-backend-cache-ttl-investigation-1.md`) inventoried every cache configured in `RedisConfig.java` against every runtime mutation path to the underlying entities. Four code sessions implemented the verdicts.

**TTL removed (five caches):** `redisBaseSites`, `redisBaseSiteOverviews`, `redisLanguage`, `redisLanguages`, `redisBaseCurrency`. None has a runtime write path to its backing entity. `BaseSite` writes are boot-only via `CatalogManager.initBaseSiteCatalog @Order(2)`, before warmup. `Language` and `Currency` are SQL-seeded with no `@Modifying` or `save`/`delete` callers anywhere in `src/main`. `CacheWarmupService` populates all five at boot (`@Order(10)`), before the `AvailabilityChangeEvent` flips readiness to `ACCEPTING_TRAFFIC` per the 2026-05-14 connection-pool decision. Admin-driven manual mutations route through `CacheAdminController`'s evict/refresh endpoints. The TTL served no correctness role; it served only as the trigger for the daily thundering-herd that hit production on 2026-05-16 (and the 24-minute variant for `redisBaseSiteOverviews` whose value was a unit-mix typo).

**TTL kept (two caches, 30 min each):** `redisUserAuth` and `redisUserInfo`. Real runtime writes exist — `DefaultUserService.saveUser`, called from `createUserSynchronized`, `updateCurrentUserData`, `assignUserRegionAndCity`, `disableUser`, and `enableUser` — explicitly paired with `@CacheEvict` for both cache regions. Eviction is the correctness mechanism. The 30-minute TTL is the documented backstop against a future code path bypassing `saveUser`; the user-deletion feature in the backlog is the named risk and will need its own eviction per the 2026-05-14 constraint. `redisUserAuth` already carried a documenting comment at `RedisConfig.java:41-46`; `redisUserInfo` got an equivalent four-line comment in this work pointing at `DefaultUserService.saveUser` / `evictUserInfoCache` as the freshness mechanisms.

**Precedent for future caches.** A cache without runtime writes should not carry a TTL; reliance on warmup-gated readiness plus admin-driven eviction via `CacheAdminController` is the correct shape. A cache with runtime writes must pair every write with `@CacheEvict` (or equivalent re-indexing, in the translations pattern) and may carry a documented TTL backstop. When adding a new cache, the documenting comment at the cache-config site states which freshness mechanism applies — either "no TTL: no runtime write path; populated by CacheWarmupService at boot, operator-driven mutations go through CacheAdminController" or "TTL is a backstop only; freshness owned by `@CacheEvict` on `<method>`."

**Helper signature.** `RedisConfig.java`'s `addTypeConfig` and `addListTypeConfig` now take `Duration ttl` where `null` means "no expiry." Replaces the prior `long ttl` (implicit minutes) form that produced the `redisBaseSiteOverviews` 24-vs-1440 transcription error. Local change; the helpers have no callers outside `RedisConfig.java`.

**Translations precedent extended to the in-memory layer.** The 2026-05-14 "Redis caching & filter control-flow fixes" decision (point 4) framed translations as no-TTL Redis with explicit re-indexing on every write. That framing was correct but incomplete: `DefaultTranslationService` also maintains an in-memory `backendTranslations` map for the `BACKEND_TRANSLATIONS` namespace, populated at boot via `@PostConstruct` and consumed by `getBackendTranslation` for backend-emitted text (push notification bodies, email content). `updateTranslation` did not refresh this map, so admin edits to `BACKEND_TRANSLATIONS` keys landed in DB and Redis but stayed stale in memory until restart. `updateTranslation` now conditionally calls `indexBackendTranslations()` when the mutated namespace is `BACKEND_TRANSLATIONS`. The translations-precedent contract now covers both layers: every translation mutation invalidates every cached representation in the same transaction boundary.

**Three orthogonal bugs fixed in the same chat.** Each surfaced as an adjacent observation during the TTL investigation and was fixed as a discrete bug:

- `DefaultUserFacade.updateCurrentUserData` evicts `redisUserInfo` after `generateUserTranslations` — closes the ordering-dependent shortBio freshness coupling the deferred-follow-up Javadoc had warned about; Javadoc deleted in the same session.
- `redisBaseCurrency` cache value is now `CurrencyDTO` instead of the `Currency` entity, mirroring every other configured cache. Closes the latent lazy-init footgun if `Currency` ever grows a `@ManyToOne`. The canonical pattern for record-DTO conversion in this codebase is direct constructor (`new CurrencyDTO(...)`), not ModelMapper — two existing call sites use this pattern; this work adds a third.

**What this does not fix.** The structural concurrent-cache-miss mechanism — multiple requests racing on a cold cache, each opening a JPA transaction via `@Transactional` on `getAllBaseSites`, exhausting the HikariCP pool — is unchanged. Removing the TTL closes the daily-expiry trigger (and the 24-min `redisBaseSiteOverviews` variant) but does not make the cold-cache path itself safe under concurrent misses on the next deploy or restart. Per-key single-flight deduplication and removing `@Transactional` from `getAllBaseSites` remain in `features/connection-pool-hardening.md`.

**Reasoning.** Inventoried mutation paths matched the verdicts cleanly: five caches with zero runtime writes had nothing for eviction-on-write to do, so TTL was the only freshness mechanism and was purely backstop. With warmup-gated readiness already covering post-deploy cold caches and admin-evict covering operator mutations, TTL had no role left except triggering pool exhaustion at expiry. The two user caches with real writes and paired eviction matched the translations precedent in spirit (write-then-invalidate); their TTL backstops a known future risk (user-deletion, account-disabling enforcement) and is documented at the code site.

**Alternatives considered.**

- `Optional<Duration>` for the helper signature instead of nullable `Duration` — rejected because `Optional` as a method parameter is the documented Java anti-pattern, the helper is `private`, and Spring's own `RedisCacheConfiguration.entryTtl` API treats absence as "don't call."
- `long` with a `0`-or-`-1` sentinel for "no expiry" — rejected because it preserves the implicit-unit footgun that produced the original transcription error.
- Per-cache one-line rationale comment versus a single block comment above the five TTL-less call sites — engineer's choice landed on a single block comment for the shared rationale; per-cache splitting available if grep-ability becomes useful later.
- Unconditional `indexBackendTranslations()` on every `updateTranslation` — rejected for `COMMON`, `BUTTONS`, `INPUT`, etc. paying a `BACKEND_TRANSLATIONS` rebuild they never benefit from. Conditional gate is one enum compare.
- ModelMapper for the `Currency → CurrencyDTO` swap — rejected because two existing call sites use direct constructor and adding a third pattern would violate Part 4a (match the surrounding code's style). ModelMapper's default record matching in 3.2.6 works more by accident than by explicit bean configuration; direct construction is type-safe and one line shorter.

**Accepted-and-known items** logged in `state.md` Risk Watch rather than `issues.md` per Igor's disposition on this chat:

- `DefaultTranslationService.backendTranslations` field is non-`volatile`; concurrency window now exercised more frequently than before. Pre-existing, low severity, one-keyword fix.
- `PriceQueryGenerator.getPriceRangeQuery2` has a dead `var baseCurrency` assignment. Pre-existing, very low severity, fix in passing.
- `CatalogDTO.categories` and `CatalogDTO.orderTypes` are perpetually null on the BaseSite cache path. Investigate during the `connection-pool-hardening` feature work.

---

## 2026-05-15 — Maintenance gate split into two KV flags; deploy flows full-lockdown with manual three-step restore

The Cloudflare router worker's single-flag maintenance gate is now a two-flag gate. The propagation has landed across all four affected repos (worker, backend, web, droplet) and is documented here for the record.

**Worker level.** `oglasino-router` (prod and stage) reads two string-typed KV keys from the `CONFIG` namespace: `maintenance.active` and `admin.bypass.disabled`. Only the literal `"true"` is truthy; absent / non-`"true"` is treated as `false`, so the default with no KV entries is open. The pre-existing third input `use.backend.check` is unchanged. The full behavior matrix lives in [`infra/cloudflare/maintenance.md`](infra/cloudflare/maintenance.md).

**Deploy flow (both backend and web).** Backend (`oglasino-backend`) and web (`oglasino-web`) deploy workflows flip **both** `maintenance.active = "true"` and `admin.bypass.disabled = "true"` via `curl` against the Cloudflare REST API before the deploy artifact lands, then wait 60 seconds before deploying. The 60-second wait is 2× the worker's `cacheTtl = 30` so every edge sees both flags `"true"` before the new artifact is reachable. There is **no auto-off** and **no backend liveness probe** in the deploy flow — both flags remain ON after the deploy completes and the workflow's final message points the operator at the restore runbook.

**Restore runbook — three manual steps, in order.** (1) `/opt/oglasino/scripts/admin-bypass-allow.sh` on the droplet flips `admin.bypass.disabled` to `"false"` so admin traffic can reach the panel. (2) Refresh caches in the admin panel — warmed under admin-only load. (3) `/opt/oglasino/scripts/maintenance-off.sh` on the droplet flips `maintenance.active` to `"false"` and resumes end-user traffic against a warmed cache. Ordering is load-bearing: step 1 before step 2 because the operator cannot otherwise reach the panel; step 2 before step 3 because flipping `maintenance.active` off cold dumps end-user traffic at an empty cache.

**Deliberately deferred.** (a) The backend `MaintenancePageController` trust-boundary defect — the maintenance state is anonymous-toggleable from the backend's own admin surface, which violates the server-as-trust-boundary rule (Part 11). Routed to Mastermind as feature work, not addressed in this propagation. (b) Three worker-internal findings: `use.backend.check` sits outside the fail-open `try`/`catch`; the worker's matrix-comment block doesn't mention `use.backend.check`; the admin locale-prefix regex hardcodes `xx-xx` and excludes the three `rsmoto-*` web locales. The first two are logged in [`issues.md`](issues.md) as small fixes; the third is out of scope and belongs with the next locale-routing feature.

**Reasoning.** A single-flag gate cannot distinguish "deploy in progress — keep operators in, keep users out" from "all traffic blocked." Splitting the bypass off into its own flag composes a master/bypass model that delivers both modes with one switch each. The manual three-step restore is the operational standard Igor wants: an explicit human pause between "deploy completed" and "users routed at it" gives the operator a chance to verify caches and surface health before public traffic arrives.

**Alternatives considered.**

- **Per-surface flags (`portal.maintenance.active` + `admin.maintenance.active`).** Rejected. Two independent surface gates require two independent decisions on every deploy; the master+bypass model collapses that to a single primary toggle with an opt-out, which is cheaper to reason about and was already the worker-level design that landed.
- **Auto-off on successful deploy.** Rejected. Igor wants the three-step restore as the operational standard, with the operator's cache-refresh as a deliberate checkpoint between admin-only and public traffic. Auto-off would skip that checkpoint.
- **Replacing the broken backend liveness probe with a Cloudflare-fronted one.** Rejected. The original probe wasn't load-bearing for the deploy decision — its absence does not change correctness. Delete-and-don't-replace was chosen over rewrite-and-keep.

---

## 2026-05-15 — Legal drafts: Privacy Policy and Terms of Use authored

Pre-lawyer drafts of the Privacy Policy and Terms of Use authored for Oglasino, plus a lawyer handoff package. Files: `legal/privacy-policy-draft.md`, `legal/terms-of-use-draft.md`, `legal/lawyer-handoff.md`.

**Key choices captured in the drafts:**

- **Controller identity.** Igor Stojanović, individual sole proprietor, Niš, Serbia. Planned incorporation post-launch handled via Terms Assignment clause + future Terms update.
- **Contact addresses.** Both mailboxes are live and operational as of 2026-06-11. support@oglasino.com — general support and appeals (also the email Reply-To and the contact-form destination); privacy@oglasino.com — privacy channel, display-only on the contact page (not form-routed in v1).
- **Lawful bases.** Contract for most processing (account, profile, listings, messages, reviews, auto-translation). Legitimate interest for moderation, IP rate-limit logging, audit retention. Consent for preference cookies and the five communication-preference toggles.
- **Retention.** Active accounts: while account is active. IPs: 30 minutes. Application logs: 90 days. Deletion audit (hashed): 30 days. Banned-user audit (hashed email): 12 months.
- **No data sale, no AI training on user content.** Mirrored in both documents as explicit user-facing commitments.
- **International transfers.** OpenAI and reCAPTCHA (both US) covered under DPF + SCCs. All other processors EU-hosted (Firebase EU region, R2 to be set EU pre-launch, DigitalOcean fra1, Vercel fra1).
- **Children.** 18+ self-declared at registration.
- **Governing law.** Serbian. Jurisdiction: exclusive Niš with EU consumer carve-out. Controlling language: Serbian.
- **Account deletion.** 7-day soft delete with reporting window — profile remains publicly visible with "Scheduled for deletion" badge, phone hidden, listings hidden, messaging blocked both ways, reports remain open. Hard delete after 7 days. Admin postponement supported (transparent model) for legal investigations and internal abuse investigations.
- **Content rules.** Permitted with conditions: hunting weapons, tobacco/alcohol, home-bred and agricultural animals. Prohibited list covers illegal items, prescription medication, counterfeit, stolen goods, adult content, restricted animals, IP-infringing items, hate content, misleading listings, spam, personal contact info in listings, and external links.
- **Termination.** Discretionary, proportionate. Notice for minor violations, immediate for serious. Email-based appeals. Admin-initiated termination has no 7-day grace period. 12-month re-registration prohibition for banned accounts.

**Pre-launch action items the operator has committed to (logged separately):**

1. Set up and monitor privacy@oglasino.com. (done 2026-06-11)
2. Set up and monitor support@oglasino.com. (done 2026-06-11)
3. Configure Cloudflare R2 bucket jurisdiction to EU.
4. Accept OpenAI's data-processing addendum.
5. Add "Manage cookie preferences" footer link.

**Adjacent observations surfaced during intake (logged for routing to feature work):**

- `Report.reporter` and `Review.reviewer` are non-nullable `@ManyToOne` FKs that will block user hard-deletion per the spec. Routing needed.
- Review image cleanup may not be covered by the current user-deletion spec.
- User-deletion spec data-inventory table specifies profile is hidden at Day 0, but the operator's actual intent (clarified during intake) is profile-stays-visible-with-status-badge so reporting can continue. Spec must be corrected before implementation.
- Reviews left by deleted users are kept (anonymized); the review-creation UI should disclose this to users at posting time.
- Existing `oglasino-platform/privacy.md` and `oglasino-platform/terms.md` are factually inaccurate against the platform (Facebook login, SMS, Supabase, active Google Analytics — none of which exist). New drafts supersede them entirely; old files to be archived or deleted.
- User-deletion spec config values (`retention-normal-deletion-days`, `retention-banned-months`) were updated by operator to 30/12 during intake; verify the spec doc reflects these so implementation matches the Privacy Policy.

**Reasoning.** The drafts were authored against the platform's audited reality (entities, deletion spec, feature behavior) rather than the prior `oglasino-platform/` drafts, which were placeholder material not aligned with the platform as built. Plain-English style was chosen on operator preference; translation into Serbian, English (final), and Russian is a separate workstream.

**Alternatives considered:** Drafting against the existing `oglasino-platform/privacy.md` and `terms.md` as a base (rejected — they were factually inaccurate, structurally insufficient for GDPR, and editing them would have produced compromised documents rather than clean ones). Publishing without a lawyer review (rejected — operator considered this for cost reasons, agent declined per `meta/legal-drafts-bootstrap.md` Section 4, operator accepted and committed to lawyer engagement).

---

## 2026-05-14 — Connection-pool exhaustion incident: warmup-gated readiness + auth-cache TTL

**Context.** Production-adjacent (stage) logs showed HikariCP connection-pool exhaustion under concurrent load — ~20 simultaneous requests draining an 8-connection pool, cascading into server-wide 500s. Investigation traced the mechanism: `BaseSiteFilter` runs on every request and calls a `@Cacheable + @Transactional` method (`DefaultBaseSiteService.getAllBaseSites`); on a cold-cache window, concurrent misses each open a JPA transaction and grab a connection, and Spring's `CacheInterceptor` has no per-key locking to deduplicate them. `/api/public/translations` appeared in the failure logs as a victim of upstream filter-chain contention, not a culprit. The incident surfaced on stage (pool size 8); the same mechanism is latent on prod (pool size 18).

**Decisions.**

1. **Cache warmup now gates readiness.** `CacheWarmupService` publishes `AvailabilityChangeEvent(ReadinessState.ACCEPTING_TRAFFIC)` only after the warmup pass completes. The app starts in `REFUSING_TRAFFIC`. This closes the post-deploy cold-cache window — traffic is not routed until the global caches are warm. Chosen over moving warmup into an earlier lifecycle phase, because that would race ahead of `CatalogManager` seeding the `Catalog` rows that `getAllBaseSites` dereferences. The warmup pass flips readiness even if an individual cache fails to warm — the gate is "warmup has run," not "warmup is perfect" — so a transient single-cache failure cannot permanently brick readiness.

2. **The docker-compose healthcheck targets readiness, not liveness.** `infra/docker-compose.yml` and `infra/docker-compose-stage.yml` healthchecks now hit `/actuator/health/readiness` instead of `/actuator/health/liveness`. Liveness reports `CORRECT` early in boot, before warmup; without this change the warmup-gating in decision 1 would be operationally inert because nothing consulted the readiness signal. `start_period` was raised from 90s to 120s (≈2× the confirmed ~60s typical boot+warmup) so the stricter check does not cause a boot loop. If a deploy ever boot-loops on this healthcheck, the response is to lengthen `start_period` further, not to revert — the gate is the load-bearing fix.

3. **`redisUserAuth` TTL raised from 1 minute to 30 minutes.** An investigation traced every production code path that mutates an auth-relevant field (`userRole`, `subscriptionType`, `subscriptionActive`, `disabled`) and confirmed every one routes through `DefaultUserService.saveUser`, whose `@CacheEvict` is the real invalidation mechanism. The 1-minute TTL was a pure safety net with nothing relying on it. 30 minutes matches `redisUserInfo`'s TTL. The eviction `@CacheEvict` — not the TTL — remains the correctness mechanism; the TTL is a backstop.

**Constraint this establishes.** Any future code path that mutates an auth-relevant field on a `User` row must route through `saveUser` (or carry its own `@CacheEvict` for `redisUserAuth`). With a 30-minute TTL, an unevicted change is stale for up to 30 minutes instead of 1. In particular, the planned user-deletion feature must evict `redisUserAuth` for the deleted user's `firebaseUid`.

**Scoped out as feature-sized work (see `features/`):** per-key single-flight deduplication of concurrent cache misses; removing `@Transactional` from `getAllBaseSites`; the `redisBaseSites` 24h-TTL cold-cache trigger (the warmup gate closes the _post-deploy_ window, not the daily-TTL-expiry one); `RateLimitFilter` categorisation of public reference-data endpoints; and the four `@Component` filters having no explicit `@Order`. These were deliberately not fixed in the bug-fixer chat because the honest fix is structural.

---

## 2026-05-14 — Redis caching & filter control-flow fixes (bug-fixer chat)

**Context.** A bug-fixer chat working the connection-pool incident surfaced several adjacent, independently-real bugs in the Redis caching layer and the request-filter chain. Each was investigated and fixed in the same chat.

**Decisions.**

1. **`DefaultBaseCurrencyService` self-invocation bypass fixed.** `convertToBaseCurrency()` and `updateBaseCurrency()` called `this.getBaseCurrency()` directly, bypassing the Spring proxy and the `@Cacheable` annotation — so a price-range-filtered product search hit the DB twice for base-currency data that should have been cached. Fixed via `@Lazy` self-injection (`self.getBaseCurrency()`). This is the first `@Lazy` self-reference in the codebase; it sets the precedent for proxy-respecting self-calls.

2. **`DefaultTranslationService.updateTranslation` made explicitly transactional.** The method mutated a `Translation` entity but had no `@Transactional`, persisting only as a side effect of OSIV plus a later call's transaction. It now carries its own `@Transactional` — if OSIV is ever disabled, admin translation edits would otherwise have silently stopped persisting.

3. **`CurrentLanguageFilter` control flow corrected, and `X-Lang` handling made explicit.** The filter's `try` block wrapped `filterChain.doFilter(...)`, silently swallowing _all_ downstream exceptions and turning genuine 500s into empty 200s. `doFilter` was moved outside the `try` — downstream exceptions now propagate on every route. Additionally: a missing or unresolvable `X-Lang` header now returns **400 `LANG_MISSING_OR_INVALID`** on language-dependent public routes, while a confirmed allowlist of language-independent public routes (`/api/public/baseSite/*`, `/api/public/config`, `/api/public/translations`, `/api/public/health/check`, `/api/public/maintenance/*`, `/api/public/verify-recaptcha`, `/api/public/app/version/*`) continues with language unset. The 400 rule applies only to `/api/public/*` — secure and auth routes were not inventoried for language-dependence and pass through unchanged. Missing and malformed `X-Lang` are treated identically.

4. **Translations Redis cache confirmed to have — correctly — no TTL.** An investigation expecting to find and remove a harmful TTL on the translations cache found there was never a TTL to remove. Every DB-write path to `oglasino_translation` (boot SQL seed; admin `updateTranslation`) triggers a full Redis reindex, so the cache cannot go stale. No change made; recorded so the absence of a TTL is understood to be deliberate and correct, not an oversight.

**Routed out, not fixed here.** `ProductErrorCode` contains cross-cutting error codes that do not belong in a product-scoped enum (`LANG_MISSING_OR_INVALID`, added by decision 3, followed an existing bad precedent). Fixing only the new code would leave the enum inconsistently cleaned; the honest fix is a refactor moving all cross-cutting codes to a proper enum, which is beyond a bug-fixer chat's mandate. Logged to `issues.md` as a high-severity entry — the one deliberate, documented exception to this chat's "nothing parks to issues.md" rule, made because the real fix is feature-sized.

---

## 2026-05-14 — QA Preparation: imageKey rename correction (supersedes earlier entry)

The earlier entry today — "QA Preparation: QaImage field renamed url → imageKey" — was wrong on two points. It claimed the rename was "no code rework" and that `features/qa-preparation.md` had been "updated to match." Both were false. The renderer (`page.tsx`) references the field directly, so renaming the type alone fails typecheck; the Docs/QA pilot session caught this when it attempted the rename and `tsc` rejected it. And the spec was never edited — it still showed `url?: string` with the old comment.

The actual change: a thin web brief renames `QaImage.url` to `imageKey` across three files — `topics.ts` (the type), `page.tsx` (the renderer reference plus the local variable `renderableImageUrls` → `renderableImageKeys`), and `features/qa-preparation.md` (the schema definition and the `images` note). No content values change — no topic entry had set the field yet.

This entry supersedes the earlier one. The earlier entry stays in the log (append-only), but its premise and its "spec updated" claim are void — this entry is the accurate record.

**Reasoning:** the field holds an R2 object key, not a URL — `ProductImageCarousel` consumes keys. A field named `url` that wants a key misleads every content author who fills it. The original error was Mastermind's: the decision was drafted assuming the field was unreferenced, without that being verified. Logged plainly so the correction is on the record and the "verify before claiming no-rework" lesson is visible.

**Alternatives considered:** deleting the earlier entry (rejected — decisions.md is append-only; a wrong entry gets superseded, not erased). Leaving the rename for a future regular web brief (rejected — no regular web brief was queued; the field would stay misnamed indefinitely and every image entry would inherit it).

---

## 2026-05-14 — QA Preparation: QaImage field renamed url → imageKey

The `QaImage` type in the QA Preparation schema had a field `url`, documented as "Cloudflare URL — filled in after the image is uploaded." The web rebuild session surfaced that `ProductImageCarousel` — the component the QA page reuses to render images — does not consume URLs. It consumes R2 object keys, which it runs through `publicImageUrl(key, 'hero')` to build the final CDN URL. A full URL placed in that field would produce a broken nested URL at render time.

The field is renamed `url` → `imageKey`. The doc comment is corrected to "R2 object key, not a URL." `features/qa-preparation.md` is updated to match: the `QaImage` type definition and the `images` note in the Schema section.

No code rework — the rebuild session shipped before any topic had an image entry, and the two example topics have an `images` entry with `name`+`description` and no key. The example in `topics.ts` needs the field name corrected; that rides along with the next web touch or the first docs brief that adds an image.

**Reasoning:** a field named `url` that requires a key is a trap for content authors. Caught before the Docs/QA phase, so the cost is one rename plus a spec edit. Caught after, it would be a find-and-replace across every authored topic with an image.

**Alternatives considered:** make `ProductImageCarousel` URL-aware (rejected — changes a shared component the product surface depends on, out of this feature's scope, and only needed because the field was misnamed). Keep `url` and instruct authors to paste a key anyway (rejected — the field name should tell the truth about what goes in it).

---

## 2026-05-14 — QA Preparation: schema and structure decisions

The QA Preparation feature builds one in-app QA reference page at `/[locale]/design` in `oglasino-web`, content-driven, covering every in-scope page, feature, and flow. The existing `/design` page is the structural template — its layout and data model are kept, its example content is discarded and rebuilt. Five decisions settled after the route inventory + structure audit (`oglasino-web` audit, 2026-05-14):

**Content stays TypeScript, not JSON.** The existing `app/[locale]/design/topics.ts` exports a `QaTopic` type and a `qaTopics: QaTopic[]` const, statically imported and bundled. This is kept. The `QaTopic` type serves as the content schema — Docs/QA authors entries against it with typecheck enforcement. Rejected: JSON file + runtime load — more moving parts, loses type enforcement, and the hot-swap benefit is near-zero since Docs/QA runs with a TS toolchain.

**No prod-exclusion mechanism.** The page can be visible in production. Its content only describes behavior already reachable through the portal — there is nothing sensitive to gate. The audit's "no guard exists" finding is acknowledged and accepted, not treated as a gap.

**`type` field added to the schema.** Every topic is one of `'page' | 'feature' | 'flow' | 'other'`. The current schema is a single flat record with no type discriminator; the feature explicitly deals in four kinds of topic. The field makes each entry self-describing and lets the header group or filter by type later.

**Model A — pages and features are both top-level topics.** A page topic and a feature topic are peers in the flat `qaTopics` list, not nested. A page that has main features references them as separate topics; some pages have no features and reference nothing. Rejected: Model B (nesting `features: QaFeature[]` inside a page topic) — requires new schema, new rendering, and new header logic; the audited page was built flat and adopting nesting is real web work for a gain that Model A plus the `type` field already delivers.

**Cross-references between topics.** A page topic links to its feature/flow topics so a QA tester clicks and scrolls to the sibling topic on the same page. The spec defines the mechanism — current direction is a dedicated `relatedTopics` field holding topic `id`s, separate from `relatedLinks` (external URLs) — to be finalized in `features/qa-preparation.md`.

**Branch:** `feature/qa-preparation`, branched off `dev`.

**Reasoning:** the feature is mostly docs work with a thin web slice. Keeping the existing structure and making the smallest schema additions that serve the four-topic-type model keeps the web work minimal and front-loads the effort onto content authoring, which is where it belongs. All five decisions bias toward the audited reality over a rebuild.

**Alternatives considered:** captured per-decision above.

---

## 2026-05-14 — Conventions Part 4 corrected: backend is Maven, not Gradle

`meta/conventions.md` Part 4 (Cleanliness) listed the backend formatter and test commands as `./gradlew spotlessCheck` and `./gradlew test`. The backend repo is Maven. Engineers have been running `mvn spotless:check` and `mvn test` all along — the convention was wrong and the engineers' actual practice was right. Part 4's backend line is corrected to the Maven commands.

This surfaced during the product-filtering bug sweep: two backend sessions reported running `mvn` commands, and the engineer flagged the convention mismatch explicitly in its session summary.

**Consequence for queued work:** two backend briefs drafted but not yet run — the `oldName` trust-boundary fix and the `translationKey`-on-the-wire change — reference `./gradlew` in their hard rules and definition-of-done sections. Both briefs must have their commands corrected to Maven before they are handed to the backend engineer. The briefs are drafts in chat history, not committed artifacts, so this is a correction-at-hand-off, not a committed-file change.

**Open item:** the corrected line uses unscoped `mvn spotless:check` and `mvn test`. The original Gradle line implied "for touched modules" scoping. If the backend is multi-module and scoped runs are wanted, the exact `mvn -pl <module>` invocation needs Igor's confirmation — this entry does not guess at Maven module-scoping syntax. Unscoped commands are correct (the filtering-sweep backend sessions ran them and passed), just potentially broader than necessary.

**Reasoning:** a convention that contradicts actual practice is worse than no convention — it either gets ignored (eroding the conventions file's authority) or followed by a new agent and breaks. Same category as prior "convention didn't match reality" corrections. Cheap to fix, and it unblocks the two queued briefs from inheriting a wrong command.

**Alternatives considered:** guessing the Maven module-scoping syntax to preserve the "touched modules" intent (rejected — wrong syntax in a conventions file is the exact problem being fixed; better to ship the correct unscoped command and flag scoping as a follow-up).

---

## 2026-05-14 — Image convention and session-file naming added to conventions

Two additions to `meta/conventions.md`.

Part 1 gains an `### Images` subsection. Docs and any agent referencing images write the image reference as if the file exists, name it kebab-case and descriptive (`product-creation-step-2.png`, not `screenshot1.png`), and add an HTML comment above the reference describing what the image should show — scaled to image complexity. Images live in an `assets/` folder next to the referencing doc. Igor supplies the actual files after the doc is written.

Part 5's session-file rule changes. Previously every engineer wrote `.agent/last-session.md`, overwritten each session — so all session history except the latest was lost. Now the engineer writes the summary to `.agent/yyyy-mm-dd-<repo>-<slug>-<n>.md` (a permanent, uniquely-named record) and _also_ to `.agent/last-session.md` (an exact copy, kept so Igor and existing briefs have a predictable path to the latest). `<n>` is sequential per `(repo, slug)` pair, not per day; the engineer counts existing `*-<slug>-*.md` files in its own `.agent/` folder and adds one. Per-repo counting is necessary because an engineer cannot see another repo's `.agent/` folder. The rule applies going forward; pre-rule `last-session.md` files are not backfilled because overwritten content cannot be recovered.

**Reasoning:** the image convention lets docs be written before screenshots exist, without placeholder cruft, and keeps the guidance for each missing file attached to the spot it's needed. The session-naming change fixes real data loss — every engineer session before this rule destroyed the previous session's summary. Numbered permanent files give the feature a full session history; the `last-session.md` copy preserves backward compatibility with every brief and habit that already points at that path.

**Alternatives considered:** for images — visible italic captions instead of HTML comments (rejected — the guidance is a note to the file-supplier, not reader-facing content; alt text already serves the reader). For session naming — per-day numbering (rejected — a slug's sessions don't map cleanly to days; per-slug sequential is the meaningful order). Dropping `last-session.md` entirely once named files exist (rejected — every existing brief and Igor's own workflow point at that path; keeping it as a duplicate costs nothing). Backfilling old session files (rejected — impossible; overwritten content is gone).

---

## 2026-05-13 — Simplicity guideline and adjacent-observations rule added to conventions

Two related additions to `meta/conventions.md`:

**Part 4a — Simplicity.** Guidelines (not bans) on when complexity earns its place. Engineers exercise judgment but explain non-obvious choices in the session summary's "For Mastermind" section. Configuration, abstractions, defensive code, and stylistic deviations are all available when they serve a concrete purpose — and visible when they do.

**Part 4b — Adjacent observations.** Engineers flag bugs and issues they notice outside their brief's scope. Flags include file path, severity guess, and an explicit "out of scope" acknowledgement. Mastermind triages flags into `issues.md`, follow-up briefs, or wontfix.

Session summary template gains one row in "Conventions check" covering both parts.

**Reasoning:** prior session output suggested engineers default to over-engineering (extra abstractions, configuration for non-varying values, defensive checks where the contract is tight). Parallel concern: engineers don't always flag adjacent issues, so bugs that should have been visible get rediscovered later. The two rules pull in opposite directions on attention — simplify what you write, broaden what you notice — which is the right balance for code quality.

**Alternatives considered:** a strict ban list ("no new abstractions without two existing call sites," etc.) — rejected because it kills good engineering instincts along with the bad. A "be simple" exhortation without checkable substance — rejected because vague rules don't change behaviour.

---

## 2026-05-13 — Translation namespaces reconciled with code; VALIDATION frozen

Conventions Part 6 Rule 1 listed 5 namespaces. The `TranslationNamespace` enum in `oglasino-backend` has 22. Conventions were stale. Reality is correct.

Conventions Part 6 Rule 1 is rewritten to list all 22 namespaces in the same grouping the enum uses (CORE, UI, LAYOUT, GLOBAL FEATURES, PAGES, META, BACKEND), with a one-line purpose for each. `INPUTS` (singular `INPUT` in code) is dropped from the convention — the namespace does not exist on disk.

**`VALIDATION` namespace is frozen.** No new keys are added to `VALIDATION`. Anything that would have gone there goes to `ERRORS` instead. Existing `VALIDATION` keys remain in place until a post-launch migration moves them all to `ERRORS`. Rationale: `ERRORS` and `VALIDATION` overlap in practice — both hold user-facing "this is wrong, fix it" strings. One namespace for everything red-on-screen is simpler than two.

Conventions Part 6 Rule 4 (the `product.<field>.<code_lowercase>` pattern) places keys in `ERRORS`. The queued `translationKey` backend brief produces keys for `ERRORS`.

**Reasoning:** the split between `VALIDATION` and `ERRORS` is conceptually defensible but breaks down in the product creation flow, where the same backend code can surface in either context. One bucket simplifies both authoring and consumption. Freezing now (rather than migrating now) avoids a multi-locale data migration during a feature push.

**Alternatives considered:** migrate all `VALIDATION` keys to `ERRORS` immediately (rejected — touches every locale in seed SQL and every consumer in web; scope explosion mid-feature). Keep both namespaces active and document the split (rejected — the split doesn't hold in practice, as the product validation case demonstrates). Drop `VALIDATION` from the enum entirely (rejected — existing keys still resolve through it; removing the enum value breaks live translations).

---

## 2026-05-13 — Session summary template gains "Obsoleted" and "Conventions check" sections

Two additions to the engineer session summary template in `meta/conventions.md` Part 5, between "Cleanup performed" and "Known gaps / TODOs":

**Obsoleted by this session.** What this session made dead but did not delete (stale tests, unreferenced code, contradictory docs). "Nothing" is a valid answer but must be written. Distinct from "Cleanup performed" — cleanup is what was deleted, obsoleted is what should have been deleted but was not, with reasons.

**Conventions check.** Explicit confirmation that the engineer checked Part 4 (cleanliness) and Part 6 (translations) against the work performed, plus any other part of conventions that applied. "N/A this session" is valid where a part does not apply (e.g., Part 6 on a session that touches no translations).

Conventions Part 4 gains a line referring to the new template section, tying the dead-code rule to its diagnostic question.

**Reasoning:** Part 4's existing rule ("if a refactor obsoletes old code, the old code is deleted in the same session") relies on the engineer noticing what's obsolete. The "Obsoleted" section forces the noticing. The "Conventions check" section catches brief-vs-conventions drift — the calibrated-challenge rule covers the same ground but operates at chat time; the template covers the same ground at session-end time. Two checkpoints, one issue.

**Alternatives considered:** trust the calibrated-challenge rule alone (rejected — already failed once in observable conditions). Add a generic "I followed conventions" confirmation (rejected — too vague to fail honestly; named parts force engagement). Move the dead-code rule out of Part 4 into Part 5 entirely (rejected — Part 4 is the rules, Part 5 is the deliverable; they reinforce each other).

---

## 2026-05-13 — "Obsoleted by this session" added to engineer session template

Engineers explicitly answer "what does this session obsolete?" at the end of every session. The answer goes in a new section of the session summary template, between "Cleanup performed" and "Known gaps / TODOs."

The two sections are distinct: "Cleanup performed" is what was deleted in this session; "Obsoleted by this session" is what is now dead but was not deleted (with reasons), or "nothing." A refactor session that leaves a stale test file is the typical case — the test isn't a cleanup target of the refactor itself, but it's now dead, and the next session shouldn't have to rediscover it.

Conventions Part 4 gains one line tying the section to the existing dead-code rule. Conventions Part 5 gains the template section.

**Reasoning:** the existing Part 4 rule ("if a refactor obsoletes old code, the old code is deleted in the same session") relies on the engineer noticing what's obsolete. Adding the diagnostic question to the template forces the noticing.

**Alternatives considered:** add the question to brief-authoring habits only, with no convention change (rejected — habits drift across many briefs; the template is the durable place). Combine obsoleted-tracking into "Cleanup performed" (rejected — mixing what-was-done with what-should-be-done loses the signal).

---

## 2026-05-13 — Engineers required to challenge briefs, with calibrated threshold

All four engineer `CLAUDE.md` files rewritten to include a "Challenging the brief" section. Engineers must read code before writing code, compare brief to actual code, and surface real mismatches in a "Brief vs reality" section before implementing.

The challenge threshold is deliberately calibrated to filter noise. Engineers challenge real bugs only: trust boundary violations, missing or wrong code references, hidden dependencies, regression risks, wire-shape mismatches, off-counts. Engineers do not challenge naming, ordering, formatting, plural vs singular labels, casing, comment phrasing, or "this could be more elegant" thoughts.

Trust boundary check is now an explicit engineer responsibility. If a brief tells an engineer to violate a trust boundary, the engineer flags it CRITICAL and refuses to implement.

**Reasoning:** the `oldName` bug existed in the code, was listed in the backend engineer's audit, but was never challenged. Engineers read code and Mastermind does not — engineers are the last line of defense on brief-vs-reality discrepancies. The first pass at this rule was too broad and surfaced cosmetic issues (singular vs plural namespace labels) at the same priority as real bugs. The calibrated version explicitly excludes cosmetic categories.

**Alternatives considered:** leaving the responsibility entirely with Mastermind (rejected — Mastermind cannot read code, cannot catch this class of bug alone). Making it advisory rather than mandatory (rejected — advisory rules degrade under deadline pressure). Letting engineers challenge anything (rejected — produced noise on the validation feature when an engineer surfaced a singular/plural labeling difference as if it were a bug).

---

## 2026-05-13 — Feature lifecycle and trust boundary principle codified

Two new sections added to `meta/conventions.md`: Part 10 (Feature lifecycle) and Part 11 (Trust boundaries).

Part 10 codifies:

- One Mastermind chat per feature. When the feature ships, the chat closes.
- Five phases per chat: Intake, Audit, Seam analysis, Canonical spec, Engineering briefs.
- Engineers audit code as ground truth, not pre-existing reports or drafts.
- Pre-existing material is input only, never authoritative.

Part 11 codifies the server-as-trust-boundary principle, with the `oldName` bug as the originating example. A value used in a moderation, authorization, or state-transition decision must be derived from auth, read from the server's database, or otherwise unforgeable by the client. Client-supplied "before" values for change detection are forbidden.

**Reasoning:** Mastermind chats had grown polluted across multiple features. Reusing chats caused context drift, stale assumptions, and process erosion. The `oldName` bug exposed a trust boundary violation that no current convention prohibited. Both gaps closed at the convention level so future agents inherit the rules.

**Alternatives considered:** keeping the Feature Audit Phase as Mastermind-only behavior (rejected — engineers also need to know audits are required and what an audit brief looks like). Adding trust boundaries as a one-line note in the security section (rejected — highest-cost class of bug seen so far, deserves its own Part).

---

## 2026-05-13 — Mastermind bootstrap rewritten for per-feature chats

`meta/mastermind-bootstrap.md` fully replaced. New version defines the per-feature chat lifecycle, the five phases, the strictness rules (corrected from over-strict failure mode), the trust boundary rule, and explicit scope limits on what a chat does not do.

Removed from the old bootstrap: long-running Mastermind assumption, cross-feature institutional memory expectation.

Added: phase discipline, "current code is ground truth" rule, chat closes when feature ships, scope-creep prevention.

**Reasoning:** the previous bootstrap was built for one long Mastermind chat covering many features. Context polluted, decisions drifted, the same agent reasoned over stale assumptions. The new bootstrap forces a clean planning surface per feature.

**Alternatives considered:** patching the existing bootstrap to add the lifecycle phases (rejected — the long-running-chat assumption was load-bearing in the old structure; a patch would leave contradictions in the file). Two bootstraps, one for "complex features" and one for "simple bug fixes" (rejected — adds a classification decision before every chat; not worth the complexity).

## 2026-05-13 — Mastermind operating rules formalized

Four operating upgrades applied to Mastermind: strictness, writing style, self-contained instructions, lead-with-a-recommendation. Captured in meta/mastermind-bootstrap.md. New Mastermind chats bootstrap with this file alongside conventions.md, state.md, and decisions.md.

Alternatives considered: keeping the rules implicit in conventions.md (rejected — operating-style rules are about how an agent communicates, not about project content; mixing them clutters both files).

## 2026-05-13 — Feature spec is authoritative over sibling-repo docs

When a doc in a sibling code repo (`oglasino-backend/docs/`, `oglasino-web/docs/`, `oglasino-expo/docs/`) overlaps with a feature spec in `oglasino-docs/features/`, the feature spec wins. Engineers reading the sibling-repo doc must defer to the feature spec on any disagreement.

Added to `meta/conventions.md` Part 1 as a standing rule.

**Reasoning:** the cross-repo consolidation tracked in `future/cross-repo-docs-consolidation.md` will eventually migrate the overlapping docs into `features/`. Until then, engineers need a single, obvious tiebreaker. Without this rule, every overlap is a judgment call.

**Alternatives considered:** scope the rule only to the three currently-overlapping files (rejected — future feature specs will hit the same overlap problem). Migrate the overlapping docs now (rejected — already deferred to post-launch per `future/cross-repo-docs-consolidation.md`).

## 2026-05-13 — Agent architecture and conventions

### Stack: Claude Code per repo, Mastermind in desktop app

Igor uses Claude Code (already installed, v2.1.140) in each of `oglasino-backend`, `oglasino-web`, `oglasino-expo`, `oglasino-docs`. Mastermind runs as a long-running Claude Desktop chat. Igor is the message bus.

**Alternatives considered:** custom Node service against Anthropic SDK (rejected — duplicates Claude Code infrastructure, costs extra tokens on top of Max x20); LangGraph (overkill for 4 agents); Cursor/Windsurf (don't give clean Mastermind/Engineer separation).

### Engineers do not commit

Engineers stage changes on disk only. Igor reviews working-directory diffs in IntelliJ/VS Code (file-color differentiation in the diff view) and runs `git add` / `git commit` himself.

**Alternatives considered:** engineer commits to feature branch only (rejected — Igor prefers diff color review over commit review for catching issues).

### Mastermind reviews summaries, not raw diffs

Engineers produce structured session summaries in `.agent/last-session.md`. Mastermind reads only summaries. If something looks off, Mastermind asks for the relevant code via Igor.

**Reasoning:** prevents Mastermind's context from filling up with every line of code change. Igor explicitly wanted long-running Mastermind conversations.

### Image pipeline create flow: Model C

Hold images in memory through the wizard. Upload to R2 only at step 4. Progressive status messages during step 4 ("Uploading images..." → "Creating product..." → done).

**Alternatives considered:** Model A (upload at step 1, current behavior — rejected, accepts orphan leak on wizard abandonment); Model A+ with pending/promote pattern (rejected for now, deferred to v2 — too much new infrastructure for 1-month timeline).

### Content moderation v2 parked

The policy-scoring engine proposal (Signal with score+confidence, weighted aggregation, ALLOW/WARN/REVIEW/BLOCK tiers) is correct in theory but premature. Parked in [`future/content-moderation-v2.md`](future/content-moderation-v2.md).

**Reasoning:** no production traffic yet means no real abuse data to calibrate weights. Current binary rule system ships and works. Revisit 2–3 months post-launch.

### Translation namespaces are fixed; no parent/child key collision

Translations live in fixed namespaces (ERRORS, INPUTS, COMMON_SYSTEM, DASHBOARD_PAGES, PAGE_SPECIFIC). Keys cannot have parent-child collision (e.g. `some.key` and `some.key.suffix` cannot coexist) — use `.label` suffix on the parent.

**Reasoning:** the translation library rejects parent-child collision. This is a hard library constraint, not a stylistic choice.

### Pre-validate endpoint added at step 2 of create wizard

New `POST /api/secure/products/pre-validate` runs content moderation on `name` + `description` only. Web wizard calls it before advancing from step 2 to step 3. Returns HTTP 200 in both clean and violation cases; clients check `errors.length`.

**Reasoning:** current create UX punishes users by collecting errors only at final submit. Pre-validate at step 2 catches content issues early. Update flow doesn't need this — it's single-page.

### Repo-internal docs stay; new docs go to `oglasino-docs/`

Existing `<repo>/docs/` folders remain (getting-started, deployment, troubleshooting). No new files added there. All new documentation authored in `oglasino-docs/`.

**Reasoning:** long-term goal is single source of truth in `oglasino-docs/` for eventual Jira/Confluence migration. Moving existing docs is busywork; growing new docs in code repos perpetuates the split.

### Docs/QA's first session is a repo sweep

Before any engineering brief runs, Docs/QA inventories `oglasino-docs/` and proposes a reorganization plan (what to move, what to keep, what new folders to add). Igor approves before further work.

**Reasoning:** the docs repo has pre-existing structure I haven't fully audited. Adding `features/`, `sessions/`, `future/`, `legal/` blindly might collide with existing folders.
