# Audit — Expo release readiness: consent mode v2

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-23
**Mode:** read-only audit. No code changes.

---

## Section 1 — `allowPreferenceCookies` inventory

| File:line | Context | Action |
|---|---|---|
| `src/lib/types/user/AuthUserDTO.ts:13` | Type definition | Declares `allowPreferenceCookies?: boolean` on the response DTO |
| `src/lib/types/user/UpdateUserDTO.ts:12` | Type definition | Declares `allowPreferenceCookies?: boolean` on the request/response DTO |
| `app/owner/dashboard/user.tsx:46` | UI state | `useState(false)` — local state for the toggle |
| `app/owner/dashboard/user.tsx:67` | Response handler | Reads `details.allowPreferenceCookies` from `GET /secure/user/update` response, falls back to `false` |
| `app/owner/dashboard/user.tsx:97` | Request body field (change detection) | Compares current toggle value against last-fetched value to detect user changes |
| `app/owner/dashboard/user.tsx:172` | Request body field | Sends `allowPreferenceCookies` in the `POST /secure/user/update` request body |
| `app/owner/dashboard/user.tsx:274` | UI element (Switch toggle) | Renders a Switch whose value is bound to the `allowPreferenceCookies` state |

### Consequence given the backend removed the field

Per the consent-mode-v2 spec (Cross-repo seams): "Brief C removed the field end-to-end (column, DTOs, entity, propagator code)."

- **Reading from `GET /secure/user/update` response (line 67):** `details.allowPreferenceCookies` will always be `undefined`. The `|| false` fallback means the toggle always initializes as OFF. **No runtime break** — the value is silently absent, default kicks in.
- **Sending in `POST /secure/user/update` request body (line 172):** The backend removed `allowPreferenceCookies` from `UpdateUserDTO` (the backend DTO). Spring's Jackson deserialization ignores unknown properties by default (`DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES` is false unless explicitly enabled). **No runtime break** — the field is silently ignored.
- **Change detection (line 97):** Compares the always-`false` toggle against the always-`undefined` fetched value. `false !== undefined` evaluates to `true`, meaning the change-detection logic always treats the preference as "changed" even when the user hasn't touched the toggle. **Functional impact:** the "no changes detected" guard (`tDash('user.no.change')`) will never fire if no other field has changed, because this comparison always evaluates as dirty. Every profile save attempt will proceed to the backend even if nothing meaningful changed. Not a break, but unnecessary network traffic and a confusing UX (user clicks save without changing anything → success toast instead of "no changes" message).
- **UI toggle (line 274):** Renders a Switch labeled with `tCookies('config.label')` and `tCookies('config.description')`. The toggle is functional in appearance but non-functional in effect — changes are sent to the backend, which ignores them. The user sees a working toggle that does nothing.

**Summary:** No runtime break. The toggle is a functional ghost — renders, accepts input, sends data, all silently ignored by the backend. The change-detection false positive is the most concrete behavioral impact.

---

## Section 2 — Other removed consent fields

Per the consent-mode-v2 spec, **only `allowPreferenceCookies` was removed** from the backend. The spec's Cross-repo seams section states: "No backend changes to `/auth/firebase-sync` request shape beyond removing the `allowPreferenceCookies` field."

The following fields are communication preferences, NOT consent fields. They were **not removed** by the consent-mode-v2 feature:

| Field | Spec says | Mobile occurrences |
|---|---|---|
| `allowEmails` | **Kept** — communication preference, not cookie consent | `AuthUserDTO.ts:15`, `UpdateUserDTO.ts:14`, `user.tsx:48,68,98,173,302` |
| `allowPromoEmails` | **Kept** — communication preference | `AuthUserDTO.ts:16`, `UpdateUserDTO.ts:15`, `user.tsx:49,69,99,174,315` |
| `allowPhoneCalling` | **Kept** — communication preference | `AuthUserDTO.ts:17`, `UpdateUserDTO.ts:16`, `UserInfoDTO.ts:14`, `user.tsx:50,70-71,100,175,330`, `ProductFunctions.tsx:35` |
| `allowNotifications` | **Kept** — communication preference | `AuthUserDTO.ts:14`, `UpdateUserDTO.ts:13`, `user.tsx:47,289` |

No fields were renamed by the consent-mode-v2 feature. The consent-mode-v2 spec created an entirely new `ConsentData` shape with Consent Mode v2 signals (`analytics_storage`, `ad_storage`, `ad_user_data`, `ad_personalization`, `preference`), stored in a new `og_consent` cookie. This is a web-only construct; mobile has no analog.

---

## Section 3 — Consent UI on mobile

**No consent banner, dialog, toggle screen, or consent-gating mechanism exists on mobile.** This is the expected answer.

- **Consent banner/dialog:** None. No component renders a consent prompt on app launch or anywhere else.
- **`COOKIES` namespace usage:** The `COOKIES` enum value is declared at `src/i18n/types.ts:39`. It is referenced in exactly one place: `app/owner/dashboard/user.tsx:35` (`useTranslations(TranslationNamespace.COOKIES)`). This usage is for rendering preference toggle labels on the user settings screen, not for a consent UI.
- **Sign-up consent:** The `RegisterDialog` (`src/components/dialog/dialogs/RegisterDialog.tsx`) collects `displayName`, `email`, `password` only. No consent checkbox, no terms-acceptance toggle, no privacy policy acknowledgment.
- **Onboarding consent:** The `BaseSiteSelector` (`src/components/init/BaseSiteSelector.tsx`) is the first-launch screen. It presents a site selector (country/region). No consent step.
- **Settings consent toggles:** The user settings screen at `app/owner/dashboard/user.tsx` renders preference toggles for cookies, notifications, emails, promo emails, and phone calling. These are communication preferences, not a consent mechanism. The "Strictly necessary" toggle (line 254-263) is a disabled always-on display — it was part of the old web cookie consent UI pattern copied to mobile. It does not gate anything on mobile.

---

## Section 4 — User profile preference toggles

Every preference toggle on the user profile/settings surface at `app/owner/dashboard/user.tsx`:

| Toggle | Source of truth | What it controls | Backend field | Status |
|---|---|---|---|---|
| Strictly necessary (line 254-263) | Hardcoded `value={true}`, `disabled` | Display-only, gates nothing | None (no state, no request field) | Labels use deleted translation keys |
| Preference cookies (line 265-277) | `useState(false)`, seeded from `GET /secure/user/update` response | Cookie preference toggle — non-functional on mobile | `allowPreferenceCookies` | **REMOVED from backend** |
| Notifications (line 279-292) | `useState(false)`, **never seeded from response** | Notification preference — appears functional but state never initializes or saves | `allowNotifications` (in type but **not included in request body**) | Never reads, never writes — completely dead toggle |
| Emails (line 294-303) | `useState(false)`, seeded from response | Email communication preference | `allowEmails` | Active |
| Promo emails (line 305-318) | `useState(false)`, seeded from response | Promo email preference | `allowPromoEmails` | Active, **but has initialization bug** (see below) |
| Phone calling (line 320-333) | `useState(false)`, seeded from response | Phone calling visibility on product pages | `allowPhoneCalling` | Active |

### Toggles corresponding to removed fields

Only the **Preference cookies** toggle corresponds to a field the spec says was removed (`allowPreferenceCookies`). See Section 1 for consequence analysis.

### Adjacent bugs found on this surface (not consent-specific, flagged for Mastermind)

1. **`allowPromoEmails` initialization bug** (`user.tsx:70`): Line 69 correctly sets `setAllowPromoEmails(details.allowPromoEmails || false)`, but line 70 immediately overwrites it with `setAllowPromoEmails(details.allowPhoneCalling || false)`. The promo emails toggle displays the phone calling value. Line 71 then correctly sets `setAllowPhoneCalling(details.allowPhoneCalling || false)`. Net effect: `allowPromoEmails` toggle always shows `allowPhoneCalling`'s value on initial load.

2. **`allowNotifications` toggle is completely non-functional** (`user.tsx:47,289`): State is declared (`useState(false)`) and a Switch renders at line 289, but the useEffect that seeds state from the backend response (lines 57-76) never calls `setAllowNotifications`. The state never initializes from the server. The `saveChanges` function's change detection (lines 91-100) does not check `allowNotifications`. The request body assembly (lines 163-176) does not include `allowNotifications`. The toggle renders, accepts user input, and discards it on save.

---

## Section 5 — GA4 / analytics consent signals

**No code on mobile conditionally fires analytics events based on consent state.** There is no analytics event firing code on mobile at all. The `src/lib/client/firebaseAnalytics.ts` file exports a `getFirebaseAnalytics()` function that initializes `firebase/analytics`, but it has zero callers anywhere in the codebase. No `logEvent`, no `gtag`, no consent-gated analytics logic exists.

---

## Section 6 — iOS ATT (App Tracking Transparency)

- **`expo-tracking-transparency` is NOT installed.** `package.json` contains only `firebase` as a direct dependency. No tracking transparency package is declared.
- **No ATT prompt code exists on mobile.** Grep for `requestTrackingPermission`, `TrackingTransparency`, `tracking-transparency` returned zero results.
- **No ATT prompt fires** — there is nothing to fire.
- **Mobile does nothing with any tracking permission result** — no tracking prompt, no tracking SDK integration, no conditional analytics behavior.

This means: if mobile ever ships analytics (GA4 via `@react-native-firebase/analytics` or similar), an ATT prompt must be wired before any tracking SDK initializes on iOS. Without ATT, Apple will reject the app from the App Store if it includes any tracking framework. The analytics brief's territory covers the SDK wiring; this audit confirms the consent-prompt side is completely absent.

---

## Section 7 — Future-consent-UI surface area

Surfaces a future mobile-consent feature would need to touch:

| Surface | File path | What needs to happen |
|---|---|---|
| Sign-up screen | `src/components/dialog/dialogs/RegisterDialog.tsx` | Consent-on-signup pattern if required (terms acceptance checkbox, privacy policy link) |
| Settings/preferences screen | `app/owner/dashboard/user.tsx` | Rework: remove dead `allowPreferenceCookies` toggle, remove dead `allowNotifications` toggle, restructure remaining toggles. If mobile-specific consent toggles are needed, add them here or on a dedicated screen. |
| Onboarding screen | `src/components/init/BaseSiteSelector.tsx` | Could be the ATT prompt location (before analytics SDK init). No consent step today. |
| Privacy policy link | `app/(portal)/(public)/privacy.tsx` | Already exists. Fetches markdown from GitHub. May need to link from consent UI. |
| Terms of service link | `app/(portal)/(public)/terms.tsx` | Already exists. Same pattern as privacy. |
| Footer navigation | `src/lib/navigation/companyNavigations.tsx` | Currently has About, Pricing, Privacy, Terms. Web added "Cookies" as the last entry — mobile equivalent (or equivalent "Privacy settings" entry) would go here. |
| Dashboard sidebar | `src/lib/navigation/dashboardNavigations.tsx` | Web has `/owner/cookies` as a sidebar entry. Mobile equivalent would be a new entry under the Account section. |
| Analytics initialization | `src/lib/client/firebaseAnalytics.ts` | Currently dead code. When analytics is wired, initialization must gate on consent state (ATT result on iOS, user preference on Android). |

---

## Section 8 — Translation coverage

- **Is the `COOKIES` namespace fetched anywhere on mobile?** Yes, at `app/owner/dashboard/user.tsx:35` via `useTranslations(TranslationNamespace.COOKIES)`.
- **Which keys are referenced?**
  - `required.label` (line 254) — **DELETED** from backend in Brief C
  - `required.description` (line 255) — **DELETED** from backend in Brief C
  - `required.sub.description` (line 257) — **DELETED** from backend in Brief C
  - `config.label` (line 267) — **DELETED** from backend in Brief 7b
  - `config.description` (line 268) — **DELETED** from backend in Brief 7b
  - `notifications.label` (line 281) — likely still exists (not listed as deleted)
  - `notifications.description` (line 282) — likely still exists
  - `notifications.warning` (line 283) — likely still exists
  - `email.label` (line 296) — likely still exists
  - `email.description` (line 297) — likely still exists
  - `email.warning` (line 298) — likely still exists
  - `email.promo.label` (line 307) — likely still exists
  - `email.promo.description` (line 308) — likely still exists
  - `email.promo.warning` (line 309) — likely still exists

- **Impact of deleted keys:** 5 translation keys referenced by mobile were deleted from the backend's seed SQL. When mobile fetches the `COOKIES` namespace, these 5 keys will be absent. Depending on the translation framework's fallback behavior, the UI will either show raw key strings (`required.label`, `config.label`, etc.) or empty strings where these labels should appear. The "Strictly necessary" section and "Preference cookies" section of the user settings screen are visually broken.

- **Are there hardcoded consent-related strings?** No. The privacy screen fetches markdown from a GitHub URL. No inline consent copy.

---

## Section 9 — Dead code

| Item | File path | Description |
|---|---|---|
| `ConsentData` type | `src/lib/types/cookie/ConsentData.ts` | Defines the old `{ necessary: boolean; preference: boolean }` shape. Imported only by `GlobalCookie.ts`. No runtime usage anywhere. Dead type. |
| `GlobalCookie` type | `src/lib/types/cookie/GlobalCookie.ts` | Defines `GlobalCookie` with `cookieConsent: ConsentData` plus card-size and lang fields. Imported by nothing outside its own file. Dead type. Mobile is not a web browser — it has no cookies. |
| `getFirebaseAnalytics()` | `src/lib/client/firebaseAnalytics.ts` | Exports a function that initializes `firebase/analytics`. Zero callers in the entire codebase. Contains `typeof window !== 'undefined'` guard — a web pattern, not a mobile pattern. Dead code. Web deleted its equivalent during consent-mode-v2 cleanup. |
| `allowPreferenceCookies` on `AuthUserDTO` | `src/lib/types/user/AuthUserDTO.ts:13` | Field no longer returned by the backend. Always `undefined` at runtime. Dead type field. |
| `allowPreferenceCookies` on `UpdateUserDTO` | `src/lib/types/user/UpdateUserDTO.ts:12` | Field sent to backend, silently ignored. Dead type field. |
| `allowNotifications` state + toggle | `app/owner/dashboard/user.tsx:47,289` | State declared, toggle rendered, but never initialized from response and never included in save request. Completely non-functional. Dead UI. |
| `allowPreferenceCookies` state + toggle + change detection | `app/owner/dashboard/user.tsx:46,67,97,172,274` | Toggle renders, state initializes to `false` (backend returns `undefined`), sends to backend (ignored). Functional ghost. |

### Commented-out consent code

No commented-out consent code found. The `loginWithFacebookFirebase` function at `src/lib/services/authService.ts:200-228` has a large commented-out block, but it's Facebook login code, not consent-related.

### Stale consent state in stores

No consent state in any Zustand store. The auth store (`src/lib/store/authStore.ts`) carries `AuthUserDTO` which includes `allowPreferenceCookies`, but this is a response-shape field, not a consent-state store.

---

## Section 10 — Trust boundary check

**Does mobile ever send a consent decision as a request body field that the server then trusts?**

Yes — `app/owner/dashboard/user.tsx:172` sends `allowPreferenceCookies` in `POST /secure/user/update`. Pre-consent-mode-v2, the backend stored this value in the `users` table and propagated it to `AuthUserDTO`. Post-removal, the backend ignores the field entirely. No trust boundary issue today because the server no longer reads the field.

The other preference fields (`allowEmails`, `allowPromoEmails`, `allowPhoneCalling`) are communication preferences sent via the same endpoint. The server stores them and uses them to control features:
- `allowPhoneCalling` gates whether the user's phone number is shown to other users (confirmed by `ProductFunctions.tsx:35` — `callingAllowed={userInfo.allowPhoneCalling || false}`).
- `allowEmails` / `allowPromoEmails` presumably gate backend email sending.

These are not consent decisions in the Consent Mode v2 sense. They are feature toggles. The client supplies a preference; the server stores and enforces it. This is the correct pattern — the server is the source of truth for the stored value, and the client sends updates.

**Does mobile ever read a consent state from the server response and use it client-side to enable features?**

`allowPhoneCalling` is read from `UserInfoDTO` at `ProductFunctions.tsx:35` to control whether the "Call" button appears on product pages. This is a server-derived value used for UI display gating — the server decides whether the user allows phone calls, the client respects the decision. Correct pattern.

No other consent-adjacent state is read from the server and used client-side to enable or disable features.

---

## Section 11 — For Mastermind

### `allowPreferenceCookies` impact

Every occurrence is listed in Section 1. Summary:
- **6 occurrences** across 3 files (2 type definitions, 4 in `user.tsx`)
- **No runtime break.** Backend ignores the field on receive; field is `undefined` on response.
- **Behavioral impact:** change-detection false positive (always-dirty comparison) and a ghost toggle that accepts input but does nothing.
- **Fix:** remove `allowPreferenceCookies` from `AuthUserDTO`, `UpdateUserDTO`, and all `user.tsx` references. Remove the "Preference cookies" toggle and the "Strictly necessary" display-only toggle from the settings screen.

### Other removed fields

No other fields were removed by consent-mode-v2. See Section 2.

### Deleted translation keys

5 COOKIES-namespace keys used by mobile were deleted from the backend seed SQL:
- `required.label`, `required.description`, `required.sub.description` (Brief C)
- `config.label`, `config.description` (Brief 7b)

The settings screen sections that use these keys will show raw key strings or empty labels. **This is a visible user-facing regression** if mobile is used against the current backend.

### No mobile consent UI exists

Mobile has no consent banner, no ATT prompt, no consent-gating logic, and no analytics SDK wired. This means:
- **For ATT/GA4 in a release context:** iOS App Store submission requires an ATT prompt before any tracking. Until `expo-tracking-transparency` is installed and the prompt is wired, mobile cannot ship with any analytics SDK. The GA4 mobile adoption chat must scope ATT as a prerequisite.
- **For general consent:** mobile users have no way to express consent preferences. The web's `og_consent` cookie system is browser-only and has no mobile analog. A future mobile-consent feature chat needs to design a native consent mechanism (possibly backed by AsyncStorage or a backend-stored consent record for cross-device sync).

### Cross-feature observations (one line each)

- **`allowPromoEmails` initialization bug** (`user.tsx:70`): line 70 overwrites the correctly-set `allowPromoEmails` with `allowPhoneCalling`'s value. Severity: medium — user sees wrong promo-email preference on load. File: `app/owner/dashboard/user.tsx:70`. I did not fix this because it is out of scope.
- **`allowNotifications` toggle is completely dead** (`user.tsx:47,289`): state never initializes from response, never saves. Severity: low — toggle renders but does nothing. File: `app/owner/dashboard/user.tsx:47`. I did not fix this because it is out of scope.
- **`firebaseAnalytics.ts` is dead code** (`src/lib/client/firebaseAnalytics.ts`): zero callers, web-style `typeof window` guard. Web deleted its equivalent. Severity: low. I did not fix this because it is out of scope.
- **`ConsentData.ts` and `GlobalCookie.ts` are dead types** (`src/lib/types/cookie/`): web-cookie types with no mobile analog, no importers outside themselves. Severity: low. I did not fix this because it is out of scope.
- **`user.tsx:74` has `console.error(error)`** in the catch block of `getUserDetails`. Conventions Part 4 prohibits ad-hoc debug logging. Severity: low. I did not fix this because it is out of scope.
- **`user.tsx:230` has hardcoded Serbian** `"DODAJ ZA BASE SITE"` — a development placeholder, Part 6 translation violation. Severity: low. I did not fix this because it is out of scope.
- **`user.tsx:178` uses `var` keyword** instead of `const`/`let` (`var toastMessage = ...`). Severity: cosmetic. I did not fix this because it is out of scope.

### Questions whose answers would change scope of a future mobile-consent feature chat

1. Should mobile consent be stored locally (AsyncStorage) or synced to backend? The web decision was browser-local (`og_consent` cookie) with cross-device sync deferred. Mobile may want the same pattern or may want server-backed storage from the start.
2. Does mobile need the full Consent Mode v2 four-signal model, or only ATT + a simplified preference toggle? Mobile has no cookies, so the cookie-consent framing doesn't apply.
3. Should the existing communication preference toggles (notifications, emails, promo emails, phone calling) be separated from any future consent UI? Currently they share a screen and use the `COOKIES` namespace, which is misleading on mobile.
4. Is there any plan to ship analytics before the full mobile-consent feature lands? If yes, ATT must be scoped as a standalone prerequisite.

---

## Section 12 — Cleanup performed

`none needed` — read-only audit.

---

## Section 13 — Obsoleted by this session

`nothing` — read-only audit.

---

## Section 14 — Conventions check

- Part 4 (cleanliness): N/A this session. Adjacent cleanliness violations flagged in Section 11 (`console.error`, hardcoded Serbian, `var` keyword, dead code).
- Part 4a (simplicity): N/A — no code written.
- Part 4b (adjacent observations): 7 adjacent observations flagged in Section 11 with file paths, severity, and "out of scope" note.
- Part 6 (translations): COOKIES namespace usage spot-checked. 5 deleted keys identified. Hardcoded Serbian placeholder identified at `user.tsx:230`.
- Part 7 (error contract): `POST /secure/user/update` request shape reviewed. Mobile sends `allowPreferenceCookies` which the backend no longer recognizes. Jackson's default behavior (ignore unknown properties) prevents a 400 error. No break, but the field should be removed from the mobile DTO.
- Part 11 (trust boundaries): Reviewed. `allowPreferenceCookies` was a client-supplied preference stored by the server — now removed. Remaining preference fields (`allowPhoneCalling`, `allowEmails`, `allowPromoEmails`) are stored by the server and enforced server-side (correct pattern). No trust-boundary violation.

---

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (adjacent bugs flagged in Section 11 for Mastermind to triage; Mastermind decides whether to file)
