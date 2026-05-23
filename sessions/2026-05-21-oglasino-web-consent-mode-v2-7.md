# Session summary

**Repo:** oglasino-web
**Branch:** dev (Igor's working branch; feature is `feature/consent-mode-v2` per brief header)
**Date:** 2026-05-21
**Task:** Brief 7 — Consent Mode v2 cleanup. Delete the dead surfaces that briefs 1-6 + 8b obsoleted but left in place. Mostly mechanical: delete dead files, narrow the `GlobalCookie` type, drop the duplicate alias. The session ends with a drafted follow-up brief for the backend engineer to delete the now-unconsumed legacy SQL rows.

**Mid-session scope expansion (Igor's verdict, in-chat):** §4 expanded from "narrow `GlobalCookie.cookieConsent` and shim `migrateLegacyConsent` with a local `LegacyGlobalCookie` type" to "strip the legacy consent migration path entirely" after a Brief-vs-reality finding surfaced a second consumer of `GlobalCookie.cookieConsent`. Igor confirmed the platform is pre-production with no users to migrate from; the entire legacy migration path was removed in this brief. Verdict + backend-contract resolution captured below.

## Brief vs reality

Read the brief, the spec's `Storage` (the `cookieConsent` removal mention) and `Translation seeds` closing notes, brief 3/5/6/8b session summaries, and grepped every input the brief named.

One substantive finding surfaced **before** writing the type narrowing in §4:

1. **The `GlobalCookie.cookieConsent` field has a second type-bound consumer the brief's audit didn't account for.**
   - Brief says (§4): "Verify by grep: nobody else reads `cookieConsent` off a `GlobalCookie`-typed value."
   - Code says: `src/lib/service/reactCalls/authService.ts:139,146` — `syncUserToBackend` runs on every Firebase login / token refresh / rotation (per the file's own header comment at 169-176, this is the listener-driven POST from `onIdTokenChanged`). Line 146 sends `allowPreferenceCookies: globalCookie && globalCookie.cookieConsent?.preference` on every `POST /auth/firebase-sync`, where `globalCookie = getGlobalCookie()` is typed `GlobalCookie | null`.
   - Why it matters: deleting the field per the original brief breaks `tsc --noEmit` at this site. Also a latent pre-existing correctness problem — for post-Consent-Mode-v2 users with only `og_consent` (no legacy cookie), `globalCookie.cookieConsent?.preference` was already evaluating to `undefined`, so the wire field went out as `null` on every firebase-sync. The legacy reader was already drifting from the new source of truth.
   - I stopped before touching `GlobalCookie.ts`, surfaced the finding to Igor with three resolution options ((a) rewire to `readOgConsent()` with legacy fallback, (b) `LegacyGlobalCookie` cast at the call site, (c) defer §4 entirely), and waited.

Two minor brief-vs-reality items noted in passing, neither halting:

- **`firebaseAnalytics.ts` lives at `src/lib/config/firebaseAnalytics.ts`, not `src/lib/service/firebaseAnalytics.ts` as §2 states.** Same module identity, zero consumers; the audit listed the file correctly but the brief's prose path drifted. Deleted at the actual path.
- **The §5 §7 numbering note (`required.*` triple).** Brief 8b already confirmed and documented these are consumed by the necessary toggle and stay seeded. Verified again in §5 of this session for the seven other keys; nothing changed.

**Igor's verdict (in-chat):** Take option (a) but stripped — no legacy fallback at all. Platform is pre-production, no users to migrate. The entire legacy `globalCookie.cookieConsent` migration path comes out in this brief: delete `migrateLegacyConsent`, strip the legacy fallback from `useConsentStore.hydrate()` and `readConsentForSsr`, rewire `syncUserToBackend` to `readOgConsent()?.preference === 'granted'`, prune the now-dead unit-test cases for legacy paths. Backend contract for `allowPreferenceCookies` when no `og_consent` exists: send `false` (non-nullable boolean column; `=== 'granted'` is false for null / `'denied'` / cookie-absent, so no `null` ever goes on the wire). Continue with §5–§8 as the original brief specified.

## Implemented

- **§1 — `src/components/client/CookieBanner.tsx` deleted.** Grep returned only the file's own internal export name (`export default function CookieBanner()`). No test file. No other consumers. Brief 3 unmounted the file from both layouts; this session removes the file.
- **§2 — `src/lib/config/firebaseAnalytics.ts` deleted.** Grep for `firebaseAnalytics`, `getFirebaseAnalytics`, `config/firebaseAnalytics`, `firebase/analytics`, dynamic-import variants, and `require(...)` variants returned only the file's own contents. Zero consumers. Deleted. (Brief named the path as `src/lib/service/firebaseAnalytics.ts`; the actual path was `src/lib/config/firebaseAnalytics.ts`. Brief 8b's audit listed the module by name without a path. Path-only discrepancy.)
- **§3 — `PREFERENCE_COOKIE_NAME` alias removed.** The only consumer was `src/lib/service/userPreferenceService.ts` (7 references across the file's six methods). Migrated all references to `GLOBAL_COOKIE_NAME` — the string value is identical (`'globalCookie'`), so cookie-write behaviour is byte-identical. Then deleted the alias line in `oglasinoCookies.ts:4` and simplified the `COOKIE_NAMES` union type from `typeof GLOBAL_COOKIE_NAME | typeof PREFERENCE_COOKIE_NAME` to `typeof GLOBAL_COOKIE_NAME`.
- **§4 — legacy consent migration path stripped entirely (expanded per Igor's verdict).**
  - `src/lib/types/cookie/GlobalCookie.ts`: removed `cookieConsent: ConsentData;` field and the `import { ConsentData } from './ConsentData';` line. The type now carries only `dashboardCardSize`, `portalCardSize`, `lang`.
  - `src/lib/types/cookie/ConsentData.ts`: file deleted — the legacy `{ necessary, preference }` shape's only importer (`GlobalCookie.ts`) no longer references it, so the module is dead. Per conventions Part 4: "If a refactor obsoletes old code, the old code is deleted in the same session."
  - `src/lib/consent/cookie.ts`: removed `migrateLegacyConsent` (entire function + export), removed `import { getGlobalCookie } from '../service/oglasinoCookies'`. The module now exports only `OG_CONSENT_COOKIE_NAME`, `readOgConsent`, `writeOgConsent`, `deleteOgConsent`.
  - `src/lib/store/useConsentStore.ts`: removed the `migrateLegacyConsent` import and the legacy-fallback branch inside `hydrate()`. New shape: `hydrate()` reads `og_consent` once and sets `{ consent: existing-or-null, hydrated: true }`. Updated the field comment on `consent` to drop the "migrated legacy cookie" phrase.
  - `src/lib/consent/ssr.ts`: removed the legacy-fallback branch from `readConsentForSsr` and the now-unused `import { GLOBAL_COOKIE_NAME } from '../service/oglasinoCookies'`. Function now reads `og_consent` once; falls through to `DENIED_DEFAULTS` if the cookie is absent, malformed, or shape-invalid.
  - `src/lib/service/reactCalls/authService.ts`: rewired the firebase-sync payload at line 146 from `globalCookie && globalCookie.cookieConsent?.preference` (which evaluated to `undefined` for any post-Consent-Mode-v2 user) to `readOgConsent()?.preference === 'granted'` (always a clean boolean per Igor's backend-contract decision). Dropped the `getGlobalCookie` import and the `const globalCookie = getGlobalCookie()` local — both were uses solely for this read. Added an `import { readOgConsent } from '../../consent/cookie'`.
  - `src/lib/consent/cookie.test.ts`: deleted the `migrateLegacyConsent` describe block (5 tests) and the corresponding mock plumbing (`getGlobalCookieMock`, the `vi.mock('../service/oglasinoCookies', …)` block, the `getGlobalCookieMock.mockReset()` line in `beforeEach`, and the `migrateLegacyConsent` import). Remaining suite: 12 tests covering `readOgConsent` (7), `writeOgConsent` (3), `deleteOgConsent` (1) — wait, recount: 7 + 3 + 1 = 11. Confirmed against the file.
  - `src/lib/consent/ssr.test.ts`: deleted 4 tests asserting legacy-fallback behaviour (`falls back to legacy globalCookie.cookieConsent when og_consent is missing`, `maps legacy preference=false to denied`, `prefers og_consent over the legacy cookie when both are present`, `does not throw on malformed legacy globalCookie`). Kept the og_consent-present, og_consent-absent, og_consent-shape-invalid, and og_consent-non-JSON paths plus all five `sanitizeForSnippet` tests.
  - `src/lib/store/useConsentStore.test.ts`: deleted the `migrateLegacyConsent` mock plumbing (hoisted `migrateLegacyConsentMock`, the `vi.mock` mapping, the `mockReset` line) and the `migrates the legacy cookie, persists it, and marks hydrated` test. The remaining `hydrate` describe now covers two paths (og_consent present, og_consent absent — formerly "neither cookie exists"). `setConsent` and `requestReopen / clearReopen` describes unchanged.
- **§5 — seven legacy `COOKIES` keys verified unconsumed in `oglasino-web`.** Grep across `app/` and `src/` for each of the seven keys returned **zero** literal-match hits in web code for `banner.header`, `banner.description`, `banner.required.label`, `banner.preference.cookies`, `banner.accept.label`. Three substring hits surfaced for `config.label` and `config.description`, all of which are unrelated keys in different namespaces — verified by reading each call site:
  - `src/lib/data/sectionNavigation.ts:125` → `admin.config.label` (admin section navigation; different parent path, different namespace context).
  - `src/components/admin/cache/CacheEvictionPanel.tsx:132` → `cache.frontend.config.description` (`ADMIN_PAGES` namespace, deep nested path).
  - `src/components/popups/dialogs/PortalConfigDialog.tsx:115` → `portal.config.description` (`DIALOG` namespace).

  None of the three is the bare `COOKIES.config.label` / `COOKIES.config.description` key. All seven §5 keys are unconsumed by web. Proceeding to §6.
- **§6 — backend SQL row deletion brief drafted in "For Mastermind."** Tracks deletion of the seven keys' rows from `0001-data-web-translations-{EN,RS,RU,CNR}.sql`.

## Files touched

- src/components/client/CookieBanner.tsx (deleted, −91)
- src/lib/config/firebaseAnalytics.ts (deleted, −18)
- src/lib/types/cookie/ConsentData.ts (deleted, −4)
- src/lib/types/cookie/GlobalCookie.ts (−3 / −1 lines)
- src/lib/service/oglasinoCookies.ts (−1 / +0; alias deleted, COOKIE_NAMES union narrowed)
- src/lib/service/userPreferenceService.ts (−1 / +1 import; +6 / −6 in-file PREFERENCE_COOKIE_NAME → GLOBAL_COOKIE_NAME via replace_all)
- src/lib/consent/cookie.ts (−18 / −1; `migrateLegacyConsent` and import removed)
- src/lib/store/useConsentStore.ts (−12 / +5; legacy branch in hydrate removed, comment shortened)
- src/lib/consent/ssr.ts (−28 / +0; legacy-fallback branch + unused import removed)
- src/lib/service/reactCalls/authService.ts (−3 / +2; rewired line 146 + import swap)
- src/lib/consent/cookie.test.ts (−59 / −13; migrate describe + mock plumbing removed)
- src/lib/consent/ssr.test.ts (−54 / +11; 4 legacy tests removed, 2 small tests retained, no new tests)
- src/lib/store/useConsentStore.test.ts (−24 / −4; migrate mock + one test removed)

Three files deleted outright. No edits outside this set. No new dependencies. No new components, no new helpers.

## Tests

- Ran: `npx tsc --noEmit` — **exit 0, clean** (no diagnostics).
- Ran: `npx vitest run src/lib/consent src/lib/store/useConsentStore.test.ts` — **35 passed, 0 failed across 5 files**. The 10 legacy-path tests (5 in cookie.test.ts + 4 in ssr.test.ts + 1 in useConsentStore.test.ts) were intentionally deleted along with the code they covered, per Igor's verdict.
- Ran: `npm test` (full suite) — **211 passed, 0 failed across 17 files**. Baseline at end of brief 8b was 221; this session's net is −10 tests (legacy-path coverage gone with the code).
- Ran: `npm run lint` (full repo) — **1 error, 179 warnings**. Same pre-existing error brief 8b flagged: `src/components/popups/dialogs/AdminReportOverviewDialog.tsx:14:10 'ta' is defined but never used`. That file sits in Igor's in-flight working tree (listed `M` in gitStatus at session start). Not introduced by this session; out of scope per brief 8b's precedent. Warnings dropped from 182 (brief 8b tail) to 179 — the three deleted files / removed code paths each contributed at least one warning that's now gone.
- Ran: `npx eslint` on touched files only — **0 errors, 3 warnings**. All three warnings are on pre-existing `: any` annotations in `src/lib/service/oglasinoCookies.ts` (lines 18, 31, 39) that were not touched by this session's edit. No new warnings introduced.
- New tests added: none (this is a deletion brief; brief §7 explicitly says "No new tests added in this brief").

### Manual verification — dev-server probe

Per brief §8, four cases. Dev server (`npm run dev`) booted clean (`✓ Ready in 279ms`, Next.js 16.2.6 + Turbopack) with no module-not-found, dead-import, or compile errors after the deletions and the rewires. Probed three routes:

1. **App still boots.** Dev server up, no errors in the startup log. `GET /` → HTTP 200 (549ms). `GET /rs-sr` → HTTP 200 (1.94s). The new banner (brief 3) is the active mount; the deleted `CookieBanner.tsx` is no longer imported anywhere in the layout tree.
2. **Settings page still works.** `GET /rs-sr/owner/user` → HTTP 200 (512ms). No compile errors. The preference toggle (brief 8b copy swap) and the analytics toggle (brief 5) continue to render against `og_consent`-backed state. No interactive UI verification driven this session — same defer-to-Igor's-end-of-session-smoke posture as briefs 3/5/8b.
3. **Legacy migration is gone (per Igor's verdict, replaces the original brief §8.3).** Code-level verification: no `migrateLegacyConsent` reference remains in `oglasino-web` (grep clean). No `cookieConsent` reference on a `GlobalCookie`-typed value (grep clean). A stale `globalCookie={cookieConsent:{…}}` cookie in a browser is now silently ignored — the banner opens normally, asks for a fresh decision, and on Accept/Reject writes `og_consent`. The legacy `cookieConsent` field continues to live on the user's existing cookie until it naturally expires or is overwritten, but nothing in `oglasino-web` reads it ever again.
4. **No dead-import errors.** Inspected the dev-server log after compiling `/`, `/rs-sr`, `/rs-sr/owner/user`. Two errors surfaced, both pre-existing and not caused by this session:
   - `[product.next.search] backend returned non-2xx { status: 0, errorCode: 'connection.timeout' }` — backend not running locally; expected and pre-existing.
   - `MISSING_MESSAGE: Could not resolve COOKIES.footer.link.label in messages for locale rs-sr` — brief 4 added the consumer (`ManageCookiesFooterButton.tsx:20`), brief 8 hasn't seeded the row yet; next-intl falls back to literal key in dev. Same gap brief 3 documented and brief 8 will close. Not introduced by this session.

   Zero errors mentioning `firebaseAnalytics`, `CookieBanner`, `PREFERENCE_COOKIE_NAME`, `migrateLegacyConsent`, or anything else this session deleted. Dev server stopped cleanly after the probe.

## Cleanup performed

- Deleted three modules outright: `src/components/client/CookieBanner.tsx`, `src/lib/config/firebaseAnalytics.ts`, `src/lib/types/cookie/ConsentData.ts`.
- Deleted the `migrateLegacyConsent` function from `src/lib/consent/cookie.ts` along with its sole import (`getGlobalCookie`).
- Deleted the legacy-fallback branch from `useConsentStore.hydrate()` and `readConsentForSsr`.
- Deleted 10 unit tests (5 + 4 + 1) covering the now-removed legacy paths.
- Deleted the `PREFERENCE_COOKIE_NAME` alias and narrowed the `COOKIE_NAMES` union type.
- Removed unused imports surfacing during the rewires: `GLOBAL_COOKIE_NAME` from `ssr.ts`, `getGlobalCookie` from `cookie.ts` and `authService.ts`, `migrateLegacyConsent` from `useConsentStore.ts`.

No commented-out code. No `console.log` added (the one pre-existing `console.warn` in `cookie.ts` for the non-Secure dev case is unchanged). No `TODO`/`FIXME` added.

## Config-file impact

- **conventions.md:** no change.
- **decisions.md:** no change.
- **state.md:** no change. Brief 9 owns the eventual `Consent Mode v2` → `shipped` status flip when the feature ships.
- **issues.md:** no change. Two adjacent observations flagged in "For Mastermind" — neither rises to an entry on its own; both are existing concerns brief 6 / brief 5 already routed.

Closure gate: confirmed. No implicit config-file dependency from this session.

## Obsoleted by this session

- `src/components/client/CookieBanner.tsx` — deleted (was unmounted by brief 3; this session removes the file).
- `src/lib/config/firebaseAnalytics.ts` — deleted (zero consumers per audit and grep).
- `src/lib/service/oglasinoCookies.ts:4` `PREFERENCE_COOKIE_NAME` alias — deleted (duplicated `GLOBAL_COOKIE_NAME`'s string value).
- `src/lib/types/cookie/GlobalCookie.cookieConsent` field — deleted (no longer the source of truth; `og_consent` owns consent state).
- `src/lib/types/cookie/ConsentData.ts` (legacy `{ necessary, preference }` shape) — deleted (orphaned once `GlobalCookie.cookieConsent` removed).
- `migrateLegacyConsent` in `src/lib/consent/cookie.ts` — deleted entirely (no users to migrate from per Igor's verdict).
- Legacy fallback branch in `useConsentStore.hydrate()` — deleted.
- Legacy fallback branch in `readConsentForSsr` — deleted.
- The `globalCookie && globalCookie.cookieConsent?.preference` read in `syncUserToBackend` — replaced with `readOgConsent()?.preference === 'granted'`.
- 10 unit tests covering the removed legacy paths — deleted.

Cross-repo follow-up obsoletion (not in scope; backend brief drafted in "For Mastermind"): seven `COOKIES`-namespace translation rows in `0001-data-web-translations-{EN,RS,RU,CNR}.sql`.

## Conventions check

- **Part 4 (cleanliness):** confirmed. No commented-out code; no debug logging; no orphaned imports on touched files (`tsc --noEmit` clean, no eslint unused-import warnings on touched files); the refactor's obsoleted code (legacy `ConsentData.ts`, `migrateLegacyConsent`, legacy SSR/store fallbacks, legacy unit tests) was deleted in the same session as the refactor that obsoleted it.
- **Part 4a (simplicity):** see structured evidence in "For Mastermind."
- **Part 4b (adjacent observations):** two flagged in "For Mastermind."
- **Part 6 (translations):** confirmed (one validation flagged). Seven `COOKIES` keys identified as unconsumed in `oglasino-web` — drafted backend brief in "For Mastermind" owns the SQL row deletion. Rule 2 (no parent/child collision): N/A — deletion only. Rule 3 (Backend agent appends to existing SQL file): N/A — this is a delete-from-SQL follow-up; the backend brief handles the convention check.
- **Other parts touched:**
  - **Part 3 (no cross-repo edits):** confirmed. No file outside `oglasino-web` modified. The SQL row deletion is routed as a drafted brief for the backend engineer, not a direct edit.
  - **Part 11 (trust boundaries):** confirmed. The rewire at `syncUserToBackend` swaps one client-supplied UX preference (`allowPreferenceCookies`) for another (also client-supplied, via the `og_consent` cookie). No moderation, authorization, or state-transition decision is touched; the backend mirror's nature as a UX-gating field is unchanged. The wire shape (`{ allowPreferenceCookies: boolean, …optional displayName }`) is identical; only the source of the boolean changed.

## Known gaps / TODOs

- **`COOKIES.footer.link.label` translation not yet seeded.** Brief 4 added the consumer (`ManageCookiesFooterButton.tsx:20`); brief 8 owns the seed. Dev-server log surfaces a `MISSING_MESSAGE` fallback to the literal key string in `rs-sr` (and presumably the other three locales) until brief 8 lands. Pre-existing gap, not introduced by this session. Brief 3 documented the same fact for the 14 banner keys brief 8 also seeds.
- **Interactive UI verification deferred.** Dev server boot, route compilation, and the 211-test green run are the gates this session exercises. Interactive Drawer / Settings-page click-through is deferred to Igor's standard end-of-session manual smoke. No new component or behaviour added in this brief, so the surface to manual-smoke is narrower than briefs 3/5.
- **Pre-existing lint error in `AdminReportOverviewDialog.tsx`.** Same finding brief 8b flagged. Out of scope per brief 8b's precedent (the file is in Igor's in-flight working tree). Not fixed.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):** nothing. This was a deletion brief — no new abstractions, no new patterns, no new configuration. The one structural change beyond a delete-and-fix-import sweep was rewiring `syncUserToBackend`'s firebase-sync payload from the legacy `globalCookie` read to `readOgConsent()?.preference === 'granted'`. That isn't new complexity — it's a 1-line site swap that reuses the brief-1 helper already imported by half a dozen consent surfaces in the repo, and it actually removes the (`globalCookie`-fetching) local variable and one import.
  - **Considered and rejected:**
    - **Igor's option (b) — local `LegacyGlobalCookie` cast at the call site.** The original brief's framing. Type-mechanic only, zero behaviour change, kept the brief mechanical. Rejected by Igor in favour of (a)-stripped because the platform is pre-production; keeping a migration shim and the legacy field around earns no upside and leaves a latent firebase-sync bug (post-Consent-Mode-v2 users sending `null` on `allowPreferenceCookies`) in place. The chosen path deletes more code overall.
    - **Igor's option (c) — defer §4 entirely.** Would land §1–§3 + §5 + §6 in this brief and route §4 to a later brief. Rejected because the audit had already confirmed the call-site population is single-step, the platform constraint (pre-production) eliminates the legacy-users concern, and splitting the work across two briefs doubles the surface to verify without buying coverage.
    - **A backend-contract widening to make `allowPreferenceCookies` nullable.** Considered while drafting the call-site rewire. Rejected — the existing column is non-nullable boolean per the precedent Igor confirmed; widening it crosses into backend territory the engineer agent doesn't own, would need a Mastermind decision, and the `=== 'granted'` semantic ("user must explicitly opt in for `true`; anything else is `false`") is the cleaner contract anyway. No `null` ever goes on the wire.
    - **Deleting `src/lib/service/userPreferenceService.ts` while migrating its imports.** Brief 6's session summary flagged this surface for removal via an `issues.md` entry. Out of scope here per brief 6's verdict; the `GLOBAL_COOKIE_NAME` rewire just relocates the import name, no behavior change. The eventual removal happens in the dedicated single-file brief the `issues.md` entry tracks.
    - **A separate test that pins the new behaviour at `syncUserToBackend` (e.g., a unit test asserting the firebase-sync payload sends `false` when `og_consent` is absent).** Considered. Rejected because (a) the call site is two lines deep inside an `axios.post` invocation that would require non-trivial mocking infrastructure to assert against, (b) `readOgConsent` already has 7 unit tests covering its return shape, and the `?.preference === 'granted'` translation is a 1-character logical step that's not test-worth-pulling on its own, (c) the brief explicitly says "No new tests added in this brief."
  - **Simplified or removed:**
    - Three modules deleted outright (`CookieBanner.tsx`, `firebaseAnalytics.ts`, legacy `ConsentData.ts`).
    - `migrateLegacyConsent` function gone (≈18 lines + its import).
    - Legacy fallback branch in `useConsentStore.hydrate()` gone (≈8 lines).
    - Legacy fallback branch in `readConsentForSsr` gone (≈25 lines including comments).
    - `PREFERENCE_COOKIE_NAME` alias gone; `COOKIE_NAMES` union narrowed from two members to one.
    - 10 unit tests covering deleted code paths gone.
    - One local variable + one import dropped from `authService.ts`.
    - Net: the consent-related surface area in `oglasino-web` shrinks by ≈150 lines of code + tests this session, replacing a forked "new + legacy" shape with a single canonical `og_consent` read.

- **Drafted follow-up brief for the backend engineer (route after this brief approves):**

  ````markdown
  # Brief — Consent Mode v2 legacy translation-key SQL cleanup

  **Session for:** backend engineer agent in `oglasino-backend`
  **Branch:** `dev` (Igor's working branch)
  **Spec:** `oglasino-docs/features/consent-mode-v2.md` — § Translation seeds closing notes ("After this feature ships and the old banner is removed, a follow-up chore can delete the unused keys; not in scope here.").
  **Upstream:** `oglasino-web` brief 7 verified zero web-side consumers for the seven keys below as of 2026-05-21. Brief 8 (backend) had already seeded the new `COOKIES`-namespace keys via the dedicated `consent-mode-v2-translations` SQL file pattern.

  ---

  ## Task

  Delete seven legacy `COOKIES`-namespace translation rows from each of the four locale SQL files. These keys are no longer consumed by `oglasino-web` after briefs 3, 5, 7, and 8b. The new banner UX and `/owner/user` settings page route through the brief-8-seeded `COOKIES.banner.*` and `COOKIES.settings.*` shapes; the rows below are stranded.

  ---

  ## Keys to delete

  - `banner.header`
  - `banner.description`
  - `banner.required.label`
  - `banner.preference.cookies`
  - `banner.accept.label`
  - `config.label`
  - `config.description`

  Seven keys × four locale files = up to 28 rows. (If any locale never had one of these keys seeded, that locale gets fewer rows.)

  ---

  ## Files to edit

  - `oglasino-backend/src/main/resources/data/translations/0001-data-web-translations-EN.sql`
  - `oglasino-backend/src/main/resources/data/translations/0001-data-web-translations-RS.sql`
  - `oglasino-backend/src/main/resources/data/translations/0001-data-web-translations-CNR.sql`
  - `oglasino-backend/src/main/resources/data/translations/0001-data-web-translations-RU.sql`

  Each row lives in the `COOKIES`-namespace section of its file. Grep each file for the key as a single-quoted literal (`'banner.header'`, `'config.label'`, etc.) to locate the row to delete. Each match should be a single `INSERT INTO translations …` row matching the convention's Part 6 Rule 3 layout.

  ---

  ## What stays seeded

  **Do NOT delete `required.label`, `required.description`, `required.sub.description`.** Brief 8b confirmed (and brief 7 re-verified) these three are still consumed by the necessary cookies toggle on `/owner/user`. Leaving them in place.

  All `banner.*` keys other than the four listed above stay (the new banner consumes them — they were seeded by brief 8). All `category.*`, `footer.*`, `settings.*` keys stay (brief 8 seeded them for the new banner / settings UX).

  ---

  ## Verification

  - `./mvnw spotless:check` — clean.
  - `./mvnw test` — full suite passes. Spring-context startup against the four edited files is the boot smoke that catches broken SQL (FK violations, namespace-section corruption, etc.).
  - Grep each of the seven keys post-deletion across all four SQL files — expected: zero remaining matches.

  ---

  ## Hard rules

  - Branch is `dev`. Do not branch off, do not switch branches, do not commit.
  - No deploys.
  - No edits outside `oglasino-backend`. The web side is already clean as of brief 7 (2026-05-21).
  - No new files in `oglasino-backend/docs/`.
  - No edits to `conventions.md` / `decisions.md` / `state.md` / `issues.md` directly. Draft any needed text in "For Mastermind."

  ---

  ## Challenging the brief

  If any of the seven keys turns out to be consumed by a non-`oglasino-web` surface (mobile / admin / backend-rendered email or push notification), stop and flag. Brief 7's audit was web-only; this brief inherits that scope, but a backend-side consumer (a `BACKEND_TRANSLATIONS` lookup that happens to reference a `COOKIES` key, etc.) would be an out-of-scope finding worth surfacing before deletion.

  If `./mvnw test` fails after the deletions because a Java code path resolves one of the seven keys, that's the same flag — stop and surface.

  ---

  ## Definition of done

  - Seven legacy `COOKIES` rows deleted from each of the four locale files.
  - `./mvnw spotless:check` clean.
  - `./mvnw test` full suite passes.
  - Grep returns zero remaining matches for each of the seven keys across all four SQL files.
  - Session summary at `.agent/yyyy-mm-dd-oglasino-backend-consent-mode-v2-<n>.md` and `.agent/last-session.md`.
  ````

- **Adjacent observations (Part 4b) flagged for Mastermind triage:**

  - **`syncUserToBackend`'s firebase-sync payload now sends a real boolean instead of a sometimes-null one.** Severity: medium (silently fixed a pre-existing latent bug). Pre-Consent-Mode-v2, the field's value came from `globalCookie.cookieConsent?.preference` — `boolean | undefined`. For post-Consent-Mode-v2 users with only `og_consent` and no legacy cookie, this was already evaluating to `undefined`, which JSON-stringifies to `null` (or omits the field — depends on axios serialization). The backend column is non-nullable boolean per Igor's confirmation, so any `null`/missing on the wire was relying on backend coalescing to `false`. The new shape (`readOgConsent()?.preference === 'granted'`) sends a real `false` for all non-granted states. Flagging because the latent bug existed in production-shape code (just not in production yet, since pre-launch), and the fix landed quietly as a byproduct of the type narrowing. No issues.md entry drafted because the fix is in this brief, but worth Mastermind awareness — the equivalent fix on the mobile side, when consent v2 adopts there, has the same pattern to think about.
  - **The legacy `ConsentData.ts` module's deletion makes the new `ConsentData` from `src/lib/consent/types.ts` the canonical one.** Pre-Consent-Mode-v2 the project carried two `ConsentData` types in different modules. The legacy one (`src/lib/types/cookie/ConsentData.ts`, `{ necessary: boolean, preference: boolean }`) is gone. Anything in this repo or sibling repos that imported the legacy shape would break — none did. Flagging because mobile (when it adopts) inherits a single canonical type instead of a fork, which is the cleaner story to start from. Severity: low.

- **Recommendation for brief 9 (Docs/QA close-out).** Brief 9 is the close-out brief. Two `state.md` Risk Watch entries this session relates to:
  - **"Non-essential cookies set without consent gating (accepted-and-known until Consent Mode v2 ships)."** This brief is mechanical cleanup, not gating — brief 6 already gated. Risk Watch row should flip to closed once the feature ships per the spec's Definition of done. Brief 6 already drafted the close-out language in its summary.
  - **No new Risk Watch entries from this session.**

- **No drafted text for `conventions.md`, `decisions.md`, `state.md`, or `issues.md`.** Closure gate satisfied — no implicit config-file dependency from this session. The Brief-vs-reality finding's resolution stayed in-chat with Igor and did not require any of the four config files to change.
