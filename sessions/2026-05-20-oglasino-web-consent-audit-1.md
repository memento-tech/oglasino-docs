# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-20
**Task:** Audit `oglasino-web` for the current state of cookie consent infrastructure: banner, storage, preference management UI, and integration points. Code is ground truth.

## Findings

### 1. Cookie consent banner — does one exist?

- **A banner exists.** `src/components/client/CookieBanner.tsx:19-93`. Client component (`'use client'`), implemented as a shadcn Drawer (`DrawerContent`).
  - **Mount points:** `app/[locale]/(portal)/layout.tsx:20` (covers public + protected portal routes) and `app/[locale]/owner/layout.tsx:44` (owner dashboard).
  - **Not mounted on `app/[locale]/admin/layout.tsx`** — admin surface gets no banner.
  - Status: `present`. Severity: `friction` — the banner exists, but as the lifecycle and CTA notes below show, it is structurally one step short of what Consent Mode v2 needs.

- **No third-party consent platform.** Grep for `cookiebot`, `onetrust`, `osano`, `iubenda`, `usercentrics` across `package.json`, `*.ts`, `*.tsx` is empty.
  - Status: `n/a`.

- **No "Reject all" / "Decline" affordance.** Only one CTA — "Accept selected" at `CookieBanner.tsx:86-88`. Users can untoggle `preference` and accept, but there is no single-click reject path. The drawer also cannot be dismissed (no close X, no escape escape handler, no backdrop dismissal because `open` is bound to internal state without a `onOpenChange` callback).
  - Status: `partial`. Severity: `blocking` — regulators generally require a reject option at least as prominent as accept; GA4 v1 banners under Consent Mode v2 are usually held to this standard.

- **Open condition.** `useEffect` at `CookieBanner.tsx:28-39` first short-circuits if `useAuthStore.user?.allowPreferenceCookies` is truthy (logged-in opted-in users see nothing — even on first visit if they were opted-in server-side); else checks `getGlobalCookie()?.cookieConsent` and renders the drawer when the cookie is absent.

### 2. What categories does the banner cover?

- **Two categories:** `necessary` (`ConsentData.necessary`) and `preference` (`ConsentData.preference`) — type at `src/lib/types/cookie/ConsentData.ts:1-4`.
  - `necessary` is implicit-always-true: `<Switch disabled checked>` at `CookieBanner.tsx:70`.
  - `preference` is user-toggleable, default `true` at `CookieBanner.tsx:25` and `:45-46`.
  - Status: `partial`. Severity: `blocking` for GA4 v1 — no analytics category, no marketing category. Consent Mode v2 expects at least `analytics_storage` (and ideally `ad_storage`, `ad_user_data`, `ad_personalization`); none exist in the model today.

- **What does each category gate?** Grep for `cookieConsent.preference` and `cookieConsent.necessary` across the repo: **zero callers** outside `CookieBanner.tsx` (writes) and `src/lib/service/reactCalls/authService.ts:146` (mirrors the value to the backend on `firebase-sync`). The "preference" toggle does **not** gate any code path.
  - Non-essential cookies are still written even when `preference` is unchecked: `globalCookie.portalCardSize` (`src/lib/store/usePortalCardSize.ts:27`), `globalCookie.dashboardCardSize` (`src/lib/store/useDashboardCardSize.ts`), and presumably `globalCookie.lang` (`GlobalCookie.ts:9`).
  - Status: `partial` (a toggle exists but is inert). Severity: `friction` — this is a pre-existing gap that GA4 v1 would inherit.

### 3. Consent storage

- **Single cookie: `globalCookie`** (`src/lib/service/oglasinoCookies.ts:3`, also re-exported as `PREFERENCE_COOKIE_NAME` at `:4`, same string).
  - **Set via `document.cookie`** at `oglasinoCookies.ts:33-38`. No domain, no `SameSite`, no `Secure`. `path=/`. `expires` is 365 days from `now` (default at `updateGlobalCookie`, `:14`).
  - **Value shape:** URI-encoded JSON of `GlobalCookie` (`src/lib/types/cookie/GlobalCookie.ts:6-11`) — `{ cookieConsent: ConsentData, dashboardCardSize, portalCardSize, lang }`. Consent is one nested field on a multi-purpose preference cookie.
  - Status: `present`.

- **Backend mirror:** `AuthUserDTO.allowPreferenceCookies` (`src/lib/types/user/AuthUserDTO.ts:13`), pushed on `/auth/firebase-sync` (`authService.ts:145-146`) and on user settings save (`UpdateUserDTO.allowPreferenceCookies` at `src/lib/types/user/UpdateUserDTO.ts:12`). Only mirrors the `preference` boolean; not the full consent state.
  - Status: `present` (partial mirror — preference only).

- **No localStorage or Zustand persistence for consent itself.** The card-size Zustand stores rehydrate from the cookie at boot (`usePortalCardSize.ts:12-19`, `SyncCardSizeFromCookie.tsx`), but consent state is only read from the cookie inside the banner.

- **SSR readability:** the cookie is not `HttpOnly`, so it is server-readable, but **today no server code reads it**. `app/layout.tsx` and `app/[locale]/layout.tsx` do not call `cookies()` from `next/headers` for `globalCookie`. Layouts that mount the banner (`(portal)/layout.tsx`, `owner/layout.tsx`) also do not read it.
  - Status: `partial`. Severity: `blocking` for GA4 v1 — Consent Mode v2 needs the consent decision to drive a `gtag('consent', 'default', …)` call inline in `<head>` *before* `gtag.js` loads, which means either reading `globalCookie` server-side and emitting the snippet from a server component, or a `Script strategy="beforeInteractive"` that does the JSON parse synchronously. Neither pattern exists today.

### 4. "Manage cookie preferences" UI

- **No footer link.** Footer (`src/components/server/layout/Footer.tsx:42-53`) renders `companyNavigations.tsx`, which lists `/about`, `/pricing`, `/privacy`, `/terms` — no cookies entry.
  - Status: `absent`. Severity: `blocking` — Privacy Policy draft commits to this link as a pre-launch action item (`oglasino-docs/legal/privacy-policy-draft.md:268`).

- **No mid-session re-entry for anonymous users.** Once `cookieConsent` is on `globalCookie`, `CookieBanner.tsx:33-37` returns null and there is no surface to bring it back. A logged-out user has no way to change their decision short of clearing cookies.
  - Status: `absent`. Severity: `blocking` — same Privacy Policy commitment.

- **Logged-in user toggle exists, decoupled from the cookie.** `app/[locale]/owner/user/page.tsx:280-297` renders a Switch for `allowPreferenceCookies` via `tCookies('config.label')` / `tCookies('config.description')`. Writes to backend via `updateUser`; does **not** write back to `globalCookie.cookieConsent.preference`. The two storage layers can drift out of sync (e.g., user accepts in banner → toggle off in settings → cookie still says `preference: true`).
  - Status: `partial`. Severity: `friction` — exists but inconsistent with the cookie; pre-existing gap.

### 5. Locale coverage

- **Namespace: `COOKIES`** (`src/translations/types/TranslationNamespaceEnum.ts:22`, matches `meta/conventions.md` Part 6).
- **Keys used by the banner** (`CookieBanner.tsx`):
  - `banner.header` (`:61`)
  - `banner.description` (`:63`)
  - `banner.required.label` (`:68`)
  - `banner.preference.cookies` (`:75`)
  - `banner.accept.label` (`:87`)
- **Additional `COOKIES` keys used by `app/[locale]/owner/user/page.tsx:281-329`:** `required.label`, `required.description`, `required.sub.description`, `config.label`, `config.description`, plus parallel groups for `notifications.*`, `email.*`, `email.promo.*`.
- **No hardcoded strings in the banner** — all surface copy is translated.
- **Locale seed coverage cannot be verified from this repo.** Translations are fetched at runtime from the backend (`src/translations/lib/translationsCache.ts:23-72`) via `/public/translations?namespace=COOKIES&lang=…`. Whether EN, RS, RU, CNR each have rows for the keys above is a backend SQL question.
  - Status: `present` (banner uses keys cleanly) / **`unknown`** for actual seed coverage in all four locales. Severity: `friction` — assuming the seed exists, the banner is locale-complete; if a key is missing for a locale, banner renders the literal key string.

### 6. Integration points — what does consent gate today?

- **Firebase Analytics — dormant.** `src/lib/config/firebaseAnalytics.ts:1-19` defines a lazy `getFirebaseAnalytics()`. Grep across the repo: **zero callers.** The module is dead code; analytics is not initialized.
  - Status: `n/a` (effectively absent). Severity: `nice-to-have` — the dead module is a tidy-up.

- **Firebase Auth / Firestore / Storage** initialize unconditionally at `src/lib/config/firebaseClient.ts:28`. These are strictly-necessary by the Privacy Policy's framing (auth session, messaging). Not consent-gated. Correct.

- **reCAPTCHA — loaded unconditionally.** `src/components/recaptcha/ReCaptchaWrapper.tsx:30` renders `<ReCAPTCHA size="invisible" sitekey={…} />` from `react-google-recaptcha` (`package.json:59`). Mounted from `src/components/popups/components/MetaDataProductDialog.tsx:52` and `src/components/popups/dialogs/SuggestCategoryDialog.tsx`. The Google reCAPTCHA script tag and its cookies load whenever those dialogs mount, with no consent check.
  - Status: `partial` (loads on demand at dialog mount, not on every page) but ungated.
  - Severity: `friction` — Privacy Policy draft already flags this for lawyer review (`oglasino-docs/legal/privacy-policy-draft.md:221-223` and `:272`). Not a GA4 blocker but lives in the same consent-decision surface.

- **Firebase Messaging service worker — unconditional.** `src/components/client/initializers/FirebaseWorkerInit.tsx:11` registers `/firebase-messaging-sw.js` on every visit if `serviceWorker` is available. Push permission itself is browser-prompted, but SW registration is not consent-gated.
  - Status: `partial`. Severity: `nice-to-have` — likely strictly-necessary in practice.

- **No marketing/analytics pixels.** Grep for `fbq`, `hotjar`, `mixpanel`, `posthog`, `segment.com`, `amplitude`, `datadog`, `sentry`, `plausible`, `gtag`, `googletagmanager`: **all empty.** Matches the Privacy Policy's "no marketing pixels" commitment (`privacy-policy-draft.md:264`).
  - Status: `n/a`.

- **Non-essential cookies set unconditionally:** `globalCookie.{portalCardSize, dashboardCardSize, lang}` — `usePortalCardSize.ts:27`, `useDashboardCardSize.ts`, language sync code. These fall under "preferences" by the Privacy Policy's framing but are set without checking `cookieConsent.preference`.
  - Status: `partial`. Severity: `friction` — pre-existing gap.

- **Net answer:** consent gates nothing today.

### 7. Server-side rendering considerations

- **Banner is client-only.** `'use client'` at `CookieBanner.tsx:1`. Mounts inside server layouts but only runs after hydration.
- **Anti-flash strategy:** `isLoaded` state at `:26` is false until `useEffect` runs (`:28-39`); `if (!isLoaded) return null` at `:41` prevents flash-of-banner. Trade-off is a small post-hydration delay before the banner appears for first-time visitors.
- **No SSR cookie read.** No layout or page calls `cookies().get('globalCookie')`. There is no server input that conditionally renders anything based on the consent decision.
- **Consequence for Consent Mode v2:** GA4's `gtag('consent', 'default', { analytics_storage: 'denied', ... })` snippet needs to run **before** `gtag.js`. The clean pattern is to read `globalCookie` from `cookies()` in `app/layout.tsx`, then inline the snippet via `<Script id="consent-default" strategy="beforeInteractive">…</Script>` (or equivalent). None of this scaffolding exists today.
- Status: `absent`. Severity: `blocking` for GA4 v1.

### 8. Privacy Policy and Terms cross-references

- **Banner has no Privacy Policy link.** Grep for `privacy` / `terms` in `CookieBanner.tsx`: empty. The banner does not point users to the policy.
  - Status: `absent`. Severity: `friction` — typically required for an informed-consent UI; a launch ask.

- **Privacy Policy page has no banner reference.** `app/[locale]/(portal)/(public)/privacy/page.tsx:30-31` renders `privacy-policy.en.md` via `MarkdownViewer` from a hard-coded English GitHub raw URL — no per-locale switching, no inline reference to the cookie banner or to a "Manage cookie preferences" link target.
- **Terms page same pattern.** `terms/page.tsx:20-21` renders `terms-of-use.en.md`, English-only.
- **Locale-leakage bug confirmed** — TODO at `privacy/page.tsx:14` ("Create locale privacy.md") matches the `issues.md`-tracked SEO/JSON-LD asymmetry and the rendered-English-on-all-locales known issue. The cookie banner doesn't add a new cookie-references inheritance of that bug because it doesn't reference the policy at all.
  - Status: `partial` (pages exist and are footer-linked; banner has no inbound link; locale leakage is pre-existing).
  - Severity: `nice-to-have` for GA4 v1 specifically; the locale leakage is its own pre-existing issue.

## Files touched

- none (read-only)

## Tests

- n/a

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (this audit may surface entries Mastermind chooses to log; not authored by this session)

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — read-only audit, no writes.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): one observation flagged in "For Mastermind" (dead `firebaseAnalytics.ts` module).
- Part 6 (translations): confirmed — banner uses the correct `COOKIES` namespace and unique key shapes; seed completeness is a backend question.
- Other parts touched: Part 11 (trust boundaries) — see seam below regarding the client-mirrored `allowPreferenceCookies` field.

## Known gaps / TODOs

- Backend translation-seed completeness for `COOKIES` keys across EN/RS/RU/CNR not verified — out-of-repo for the web engineer agent. Flagged as a seam in "For Mastermind."

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing — no implementation choices to make in a discovery audit.
  - Simplified or removed: nothing.

- **Overall verdict on consent-foundation readiness for GA4 v1.**

  **Not ready.** A two-category preference banner exists and the storage primitive (`globalCookie.cookieConsent`) is in place, but four structural pieces are missing before GA4 v1 with Consent Mode v2 can fire its first event safely:

  1. **No analytics (or marketing) category in the consent model.** `ConsentData` is `{ necessary, preference }`. Consent Mode v2 expects at minimum `analytics_storage`; ideally also `ad_storage`, `ad_user_data`, `ad_personalization`. The category shape needs to grow.
  2. **No SSR-aware consent read.** `gtag('consent', 'default', …)` must run before `gtag.js`. Today nothing in `app/layout.tsx` or `app/[locale]/layout.tsx` reads `globalCookie` from `next/headers`. A `Script strategy="beforeInteractive"` with an inline JSON-parsed read, or a server-component read that emits the snippet, is required.
  3. **No "Manage cookie preferences" footer link.** Privacy Policy draft commits to this as a pre-launch action item, and there is no anonymous-user re-entry to the banner today.
  4. **No "Reject all" affordance on the current banner.** A regulator-defensible Consent Mode v2 banner generally needs one. The current "Accept selected" single-CTA UX assumes the user will untoggle "preference" before clicking accept.

  Additional non-blocking observations: the current "preference" toggle gates nothing in code (card-size and language cookies are written regardless); the logged-in settings toggle at `/owner/user` writes to the backend mirror but not back to the cookie, so the two storage layers can drift; `firebaseAnalytics.ts` is a dead module that suggests an earlier intent to wire analytics that was abandoned; reCAPTCHA loads ungated (Privacy Policy already flags this for lawyer review). None of these are GA4-v1 blockers in themselves.

- **Architectural questions for Mastermind decision:**
  - Where does the "Manage cookie preferences" link go in the footer? `companyNavigations.tsx` is the natural slot (insert between `terms` and end, or above `privacy`). Decision needed on whether this link opens the same Drawer or a dedicated `/cookie-preferences` page.
  - Should the GA4 v1 consent model add only `analytics_storage`, or all four Consent Mode v2 signals up-front (`ad_storage`, `ad_user_data`, `ad_personalization`) even though the Privacy Policy commits to no marketing pixels? The conservative choice is to model all four and set the ad-* ones to `denied` permanently; that future-proofs the wire shape.
  - Should the consent storage be split out of `globalCookie` into its own dedicated cookie (`og_consent`), making the SSR read trivial (no JSON parsing required), or stay nested? Splitting it would also let the cookie itself carry `Secure`/`SameSite=Lax` independently of the preference cookie.
  - Should the logged-in settings toggle at `/owner/user` continue to be the canonical "Manage cookie preferences" surface for authenticated users, with the footer link routing logged-in users there and logged-out users to a dedicated page or Drawer reopen?

- **Backend-seam assumptions explicitly stated:**
  - **Assumption:** the `COOKIES` namespace has rows seeded for EN, RS, RU, CNR for the existing keys (`banner.header`, `banner.description`, `banner.required.label`, `banner.preference.cookies`, `banner.accept.label`, plus the `required.*` / `config.*` / `notifications.*` / `email.*` / `email.promo.*` keys consumed by `/owner/user`). Web cannot verify; backend audit can.
  - **Assumption:** the `/auth/firebase-sync` endpoint accepts `allowPreferenceCookies: boolean | undefined` as currently sent at `authService.ts:146`. If GA4 v1 grows the consent model to include analytics, the backend mirror field shape changes — current backend stores a single boolean, which would either need to widen to a shape (`{ preference, analytics, marketing }`) or split into separate boolean columns.

- **Adjacent observation (Part 4b) flagged for Mastermind triage:**
  - **`src/lib/config/firebaseAnalytics.ts` is dead code** — defines `getFirebaseAnalytics()` with zero callers anywhere in the repo. Severity: low. I did not delete it because it is out of scope for a read-only audit, and because Mastermind may want to keep it as the staging spot for GA4 wiring; the module is harmless until called. Recommend deciding its fate as part of the GA4 v1 plan.
  - **`PREFERENCE_COOKIE_NAME` is a meaningless alias** — `src/lib/service/oglasinoCookies.ts:4` defines `PREFERENCE_COOKIE_NAME = 'globalCookie'` (identical string to `GLOBAL_COOKIE_NAME` at `:3`). Severity: low — dead/duplicate naming. Likely a vestige of an earlier "split cookies" plan. I did not fix this because it is out of scope.

- **No drafted config-file text in this session.** Mastermind seam-analysis may produce drafts for `state.md` / `issues.md`; the audit itself does not.
