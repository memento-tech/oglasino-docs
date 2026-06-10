# Audit — oglasino-web — legal-document fact verification (Q8)

- **Date:** 2026-06-10
- **Repo:** oglasino-web
- **Branch:** dev
- **Type:** READ-ONLY fact-verification audit
- **Method:** every claim dual-sourced (direct file Read + plain `grep -n`). `rg` match-highlighting corrupts matched tokens in this environment and was not relied upon.

---

## Q8(a) GA4 loading model — VERIFIED-FALSE (advanced Consent Mode, not basic)

**Evidence — `app/layout.tsx:17,47-81`:**
```
17  const MEASUREMENT_ID = process.env.NEXT_PUBLIC_GA4_MEASUREMENT_ID;
49  gtag('consent', 'default', { ad_storage: '…', …, analytics_storage: '…', wait_for_update: 500 });
70  {MEASUREMENT_ID && (
73    <Script src={`…/gtag/js?id=${MEASUREMENT_ID}`} strategy="afterInteractive" />
76    <Script id="ga4-init">gtag('config', '${MEASUREMENT_ID}', { send_page_view: false });
```
The gtag/GA script load is gated **only** by the `NEXT_PUBLIC_GA4_MEASUREMENT_ID` env var — not by consent — and runs alongside a default-denied `gtag('consent','default',…)` block. That is advanced Consent Mode.

**Plain answer:** GA4 loads upfront on every page with default-denied signals; declining suppresses analytics cookies but advanced mode still emits cookieless consent-state pings, so PP §7 "no analytics data is collected" is false (the "no analytics cookies" half is true — see (i)).

---

## Q8(b) Consent Mode signals — VERIFIED-TRUE

**Evidence:**
- Default state: `app/layout.tsx:49-55` (`gtag('consent','default',{ ad_storage, ad_user_data, ad_personalization, analytics_storage, wait_for_update:500 })`).
- Update call: `src/lib/consent/sideEffects.ts:35-40` (`gtag('consent','update',{ analytics_storage, ad_storage, ad_user_data, ad_personalization })`).
- `ad_*` hard-denied with no enabling path:
  - Literal `'denied'` types — `src/lib/consent/types.ts:12-14`.
  - Every builder sets `'denied'` — `consentDecisions.ts:17-19, 30-32, 46-48`.
  - `isValidConsent` rejects non-`'denied'` — `types.ts:45-47`.
  - `sanitizeForSnippet` falls back to `DENIED_DEFAULTS` on off-union — `ssr.ts:31-37`.

**Plain answer:** The three ad signals are denied at type, builder, validator, and SSR-sanitizer layers; no code path can grant them. Only `analytics_storage`/`preference` are user-toggleable.

---

## Q8(c) reCAPTCHA scope — PARTIAL

**Version:** reCAPTCHA **v2 invisible** — `src/components/recaptcha/ReCaptchaWrapper.tsx:30` (`size="invisible"`, `executeAsync()`); lib `react-google-recaptcha@^3.1.0` (`package.json:58`). Not v3/enterprise. Verified server-side at `/public/verify-recaptcha` (`recaptchaService.ts:6`).

**Surfaces:**
| Surface | File | Doc-claimed |
|---|---|---|
| Registration | `RegisterDialog.tsx:222` | ✓ |
| Listing creation | `MetaDataProductDialog.tsx:52` (via `CreateNewProductDialog.tsx:171`) | ✓ |
| **Login** | `LogInDialog.tsx` — grep `recaptcha` = **0** | ✗ claimed, absent |
| Product review | `ProductReviewDialog.tsx:216` | undisclosed |
| Suggest category | `SuggestCategoryDialog.tsx` | undisclosed |
| Report | `ReportDialog.tsx` | undisclosed |

**Plain answer:** v2 invisible; registration and listing-creation match, login has no reCAPTCHA (claim false), and review/suggest-category/report carry it but aren't disclosed.

---

## Q8(d) Banner re-display on policy/terms version change — NOT FOUND (unimplemented)

**Evidence — `ConsentBanner.tsx:47-71`:** banner opens only when the cookie is absent (`state.consent === null`, line 51) or the footer `reopenRequested` flag flips (line 57). `version: 1` (`types.ts:16`) is a hardcoded schema literal enforced by `isValidConsent` (`types.ts:48`), not a policy/terms stamp; no version comparison exists.

**Plain answer:** No version-stamp re-prompt exists. Bumping the hardcoded schema constant would invalidate old cookies and re-show the banner as a side effect, but nothing ties re-display to a legal-document version. PP §7/§12 promise is unimplemented.

---

## Q8(e) Cookie-preferences page & links — PARTIAL

**Evidence:**
- Page exists: `app/[locale]/owner/cookies/page.tsx`.
- Account-settings nav links it: `sectionNavigation.ts:78-81` (`url: '/owner/cookies'`) → `DashboardSidebar.tsx:29` (`NavMain`) → `app/[locale]/owner/layout.tsx`.
- Footer does NOT link the page: `ManageCookiesFooterButton.tsx:18` calls `requestReopen()`, which reopens the consent banner drawer (`ConsentBanner.tsx:57-60`).

**Plain answer:** Page exists and is linked from account settings, but the footer control reopens the consent banner rather than linking to `/owner/cookies`; PP §7/§9 "linked from both" is half-true.

---

## Q8(f) Web push — VERIFIED-TRUE (web app registers web-push tokens)

**Evidence — `src/notifications/lib/devicePush.ts:23-26,36,46`:** `getToken(messaging, { vapidKey: NEXT_PUBLIC_FIREBASE_VAPID_KEY, serviceWorkerRegistration })` after `Notification.requestPermission()`, then `attachPushTokenToBackend(token)`. Wired on auth via `UseTokenRefresh.tsx:103`. Service worker present: `public/firebase-messaging-sw.js`.

**Plain answer:** FCM web push is fully implemented and active; PP §2.11 (browser notification permissions) is correct, PP §2.15 ("push is app-only") is contradicted by the code.

---

## Q8(g) Password path — VERIFIED-TRUE (strictly client→Firebase SDK)

**Evidence:**
- Login: `authService.ts:169` `signInWithEmailAndPassword(auth, email, password)`.
- Register: `authService.ts:179` `createUserWithEmailAndPassword(auth, email, password)`.
- Reset confirm: `passwordResetService.ts:76` `confirmPasswordReset(auth, oobCode, newPassword)`.
- Reset request: `passwordResetService.ts:36` `BACKEND_API.post('/auth/request-password-reset', { email })` — email only.
- Other backend auth call: `authService.ts:134` `/auth/firebase-sync` — ID token, not password.
- grep `password` across `app/api/`, `app/actions/`, and all HTTP service calls → no password in any route, server action, or request body.

**Plain answer:** Passwords go only to the Firebase SDK from the client; Oglasino's backend/SSR/API never receive them. PP §2.1 is accurate.

---

## Q8(h) Region pinning — VERIFIED-TRUE

**Evidence — `vercel.json:4`:** `"regions": ["fra1"]`.

**Plain answer:** Deployment is pinned to `fra1` in the repo's `vercel.json`, matching PP §4.

---

## Q8(i) Cookie/storage lifetimes — VERIFIED

**Evidence:**
- `og_consent`: `Max-Age = ONE_YEAR_SECONDS = 60*60*24*365` = **31,536,000 s (365 days)** — `cookie.ts:5,37`; attrs `Path=/; SameSite=Lax; Secure` (HTTPS); deletion `Max-Age=0` (line 50).
- `_ga`: **no explicit config** — grep `cookie_expires|cookie_flags|cookie_domain|cookie_prefix` across `src/`+`app/` → none; only `gtag('config')` option is `send_page_view:false` (`layout.tsx:78`). Falls to GA default (~2 years), set only on analytics grant.
- Banner writes only `og_consent` (`writeOgConsent`); on transition into denied, `setConsent` calls `clearGlobalCookie()` (`useConsentStore.ts:44-46`) — clears, not sets.

**Plain answer:** `og_consent` lives 365 days; `_ga` lifetime is unconfigured (GA default ~2y, analytics-grant only); banner writes nothing beyond `og_consent`. PP §7 should disclose both lifetimes.

---

## Adjacent findings (contradict the quoted claims)
1. **(a)** Advanced Consent Mode permits cookieless GA pings on decline — PP "no analytics data is collected" overstates protection.
2. **(c)** Login has no reCAPTCHA despite the doc listing it.
3. **(c)** Undisclosed reCAPTCHA surfaces: product review, suggest-category, report.
4. **(e)** Footer reopens the consent banner instead of linking to `/owner/cookies`.
5. **(d)** No policy/terms-version re-prompt mechanism exists; `version` is a schema literal.
6. **(f)** Web push is live — PP §2.15 "app-only push" framing is factually wrong.
7. **(i)** `_ga` lifetime is undisclosed and unconfigured (GA default ~2y), not anything the repo pins.
