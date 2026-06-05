# Audit — Consent-field cleanup targets (C) + consent UI build-surface (G)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (uncommitted working tree, audited as-is)
**Date:** 2026-05-30
**Slug:** consent-mode-mobile-expo
**Mode:** READ-ONLY. No files touched. Grep/ripgrep + file reads only.
**Task (from brief):** Re-confirm the consent-field cleanup targets and their current line numbers (C), and discover the surface a new mobile consent UI would build on (G), against the post-Φ1–Φ4 `new-expo-dev` tree. The prior consent audit (`.agent/audit-expo-readiness-consent-mode.md`, dated 2026-05-23, on `dev`) is treated as stale candidates; every claim re-confirmed below.

> **Method note.** All file:line references are the CURRENT state of `new-expo-dev`. The two single-most-load-bearing findings (the Φ4 grenade and the C3 double-set bug) were re-verified by direct file read after the fan-out, not just by grep. The settings screen `app/owner/dashboard/user.tsx` is **357 lines** today (prior audit's references topped out around line 333).

---

## PART C — Cleanup target confirmation

### C1 — `allowPreferenceCookies` inventory (re-confirmed)

Grep: `rg -ni "allowPreferenceCookies" src/ app/`

**8 occurrences across 3 files** (prior audit said 6; the set of 3 files is unchanged — `AuthUserDTO.ts`, `UpdateUserDTO.ts`, and the settings screen `app/owner/dashboard/user.tsx`). The increase from 6→8 is line-level splitting, not new logic: the prior audit collapsed the Switch's `value`/`onValueChange` to one row, and counted the seeding line once.

| File:line | Context | Classification |
|---|---|---|
| `src/lib/types/user/AuthUserDTO.ts:13` | `allowPreferenceCookies?: boolean;` | type declaration |
| `src/lib/types/user/UpdateUserDTO.ts:12` | `allowPreferenceCookies?: boolean;` | type declaration |
| `app/owner/dashboard/user.tsx:46` | `const [allowPreferenceCookies, setAllowPreferenceCookies] = useState(false);` | state |
| `app/owner/dashboard/user.tsx:67` | `setAllowPreferenceCookies(details.allowPreferenceCookies \|\| false);` | response-read (seeding) |
| `app/owner/dashboard/user.tsx:97` | `allowPreferenceCookies !== userDetails.allowPreferenceCookies \|\|` | change-detection |
| `app/owner/dashboard/user.tsx:176` | `allowPreferenceCookies,` | request-send (save body) |
| `app/owner/dashboard/user.tsx:286` | `value={allowPreferenceCookies}` | UI element (Switch value) |
| `app/owner/dashboard/user.tsx:287` | `onValueChange={setAllowPreferenceCookies}` | UI element (Switch handler) |

**Delta vs prior audit:** all five logical sites survive, shifted down by the post-Φ4 growth of the file: state `:46` (same), seeding `:67` (same), change-detection `:97` (same), save-body `:172 → :176`, Switch `:274 → :284-288`.

### C2 — The settings screen, current shape

`app/owner/dashboard/user.tsx`, **357 lines**.

1. **Key line numbers**

   - Preference-toggle `useState`: `:46` `allowPreferenceCookies`, `:47` `allowNotifications`, `:48` `allowEmails`, `:49` `allowPromoEmails`, `:50` `allowPhoneCalling`.
   - Response-seeding effect (reads `getUserDetails(user)` → state): `:57-76`, with the toggle seeders at `:67-71`. (`getUserDetails` resolves the `GET /secure/user/update` response; see C3 for the bug inside.)
   - Change-detection block: `:91-103`, toggle comparisons at `:97-100`.
   - Request-body assembly: `:167-180` (the object literal passed to `updateUser({...})`), toggle fields at `:176-179`.
   - Rendered Switches: `:274` (required cookies — hardcoded `disabled value`, no state), `:284-288` (`allowPreferenceCookies`), `:299-303` (`allowNotifications`), `:314` (`allowEmails`), `:325-329` (`allowPromoEmails`), `:340-344` (`allowPhoneCalling`).

2. **The Φ4 grenade — DEFUSED.** The `await updateUser({...})` save call is now at **`user.tsx:167`** and **DOES have a try/catch** around it today. The prior Φ4 brief-1 summary's claim — "`:163` has `await updateUser({...})` with NO try/catch" — **no longer holds.** Exact current error handling, `:165-203`:

   ```ts
   let result;
   try {
     result = await updateUser({ id: user.id, firebaseUid: user.firebaseUid, email,
       displayName, profileImageKey, shortBio, phoneNumber,
       regionAndCity: selectedRegionAndCity,
       allowPreferenceCookies, allowEmails, allowPromoEmails, allowPhoneCalling
     }).finally(() => setLoading(false));
   } catch (err) {
     // Orphan cleanup: the avatar landed in R2 but the profile save threw —
     // fire-and-forget DELETE so the key doesn't leak (mirrors productService).
     if (uploadedAvatarKey) void cleanupOrphanImages([uploadedAvatarKey]);
     throw err;                                    // :185 — RE-THROWS
   }
   // :188+ success/failure toast; the !result (non-throw falsy) path shows tError('unknown')
   ```

   So `updateUser` throwing no longer crashes uncaught *at the call site* — but the catch **re-throws** (`:185`). `saveChanges` is invoked as a fire-and-forget handler (`onPress={saveChanges}` at `:348`), so a throw becomes an **unhandled promise rejection with no user-facing toast on the throw path** (the success/failure toast block at `:188-202` is skipped on throw). The non-throw falsy-result path (`!result`, `:190-196`) *does* toast `tError('unknown')`. **Implication for C:** the grenade no longer needs a try/catch added, but when C edits this save body it should decide whether the throw path needs a user-facing error toast (today it's silent). See Part 4b observation #1.

3. **Per-toggle wiring (re-confirms B2/B3):**

   | Toggle | Rendered | Seeded from response | In change-detection | In save body |
   |---|---|---|---|---|
   | required cookies (no state; hardcoded `disabled value`) `:274` | yes | n/a | no | no |
   | `allowPreferenceCookies` `:284` | yes | yes `:67` | yes `:97` | yes `:176` |
   | `allowNotifications` `:299` | yes | **no** | **no** | **no** |
   | `allowEmails` `:314` | yes | yes `:68` | yes `:98` | yes `:177` |
   | `allowPromoEmails` `:325` | yes | yes `:69` **then overwritten `:70`** | yes `:99` | yes `:178` |
   | `allowPhoneCalling` `:340` | yes | yes `:71` | yes `:100` | yes `:179` |

### C3 — B2: `allowPromoEmails` initialization bug (re-confirmed — STILL EXISTS)

Response-seeding effect at `:57-76`. The bug is live, `user.tsx:69-70` (verified by direct read):

```ts
69:  setAllowPromoEmails(details.allowPromoEmails || false);   // correct
70:  setAllowPromoEmails(details.allowPhoneCalling || false);  // overwrites with phone-calling's value
71:  setAllowPhoneCalling(details.allowPhoneCalling || false); // phone-calling seeded correctly
```

Net effect unchanged from the prior audit: the promo-emails toggle initializes to `allowPhoneCalling`'s value on load. **Delta:** prior audit cited `:70`; current location is identical (`:69-70`).

### C4 — B3: `allowNotifications` dead toggle (re-confirmed — all 4 facts hold)

1. **State declared:** YES — `user.tsx:47`.
2. **Toggle rendered:** YES — `user.tsx:299-303` (`value={allowNotifications}` at `:301`, `onValueChange={setAllowNotifications}` at `:302`).
3. **Seeded from response:** NO. The seeding effect (`:57-76`) never calls `setAllowNotifications`.
4. **In change-detection / save body:** NO to both (`:91-103` and `:167-180` contain no `allowNotifications`).

Grep proof (only two hits — declaration + render, nothing in the effect, change-detection, or save body):
```
$ rg -n "allowNotifications" app/owner/dashboard/user.tsx
47:  const [allowNotifications, setAllowNotifications] = useState(false);
301:              value={allowNotifications}
```

The toggle flips local UI state that is never read back, compared, or sent — a completely dead control. Note: the DTO field **does** exist (`AuthUserDTO.ts:14`, `UpdateUserDTO.ts:13`), so the "remove vs wire" decision is open — the wire would be trivial (seed + change-detect + send), or the toggle+state can be removed. **Delta:** prior audit cited `:47,289`; current is `:47` (state) and `:299-303` (render; `value` at `:301`).

### C5 — B4: the deleted-keys claim (mobile half — THE TRAP)

1. **The five COOKIES keys — all present, all referenced ONLY in `user.tsx`** (no second consumer; the superficially-similar hits in other files are a *different namespace*, not a collision):

   | Key | Current file:line | Context |
   |---|---|---|
   | `required.label` | `user.tsx:266` | `tCookies('required.label')` |
   | `required.description` | `user.tsx:267` | `tCookies('required.description')` |
   | `required.sub.description` | `user.tsx:269` | `tCookies('required.sub.description')` |
   | `config.label` | `user.tsx:279` | `tCookies('config.label')` |
   | `config.description` | `user.tsx:280` | `tCookies('config.description')` |

   Non-collisions confirmed: `AppVersionConfigurationDialog.tsx:32` calls `tDialog('app.version.required.description', …)` (DIALOG namespace, full key `app.version.required.description`); `PortalConfigDialog.tsx:58` calls `tDialog('portal.config.description')` (DIALOG namespace). Neither is the bare COOKIES key.

2. **Translation hook:** `user.tsx:35` — `const tCookies = useTranslations(TranslationNamespace.COOKIES);` (namespace `COOKIES`, hook `useTranslations`). Same as prior audit (`:35`).

3. **Every OTHER COOKIES-namespace key the screen calls today:**

   | Key | file:line |
   |---|---|
   | `notifications.label` | `user.tsx:293` |
   | `notifications.description` | `user.tsx:294` |
   | `notifications.warning` | `user.tsx:295` |
   | `email.label` | `user.tsx:308` |
   | `email.description` | `user.tsx:309` |
   | `email.warning` | `user.tsx:310` |
   | `email.promo.label` | `user.tsx:319` |
   | `email.promo.description` | `user.tsx:320` |
   | `email.promo.warning` | `user.tsx:321` |

   **Note (new vs prior audit):** the phone-calling toggle's labels at `:334-336` use `tDash(…)` / **`DASHBOARD_PAGES`** namespace (`phone.number.setup.label/description/warning`), NOT COOKIES. The prior audit did not record the phone-calling labels' namespace.

   **Delta:** the five deleted-key references survive, shifted down (`254/255/257/267/268 → 266/267/269/279/280`); the `notifications.*` / `email.*` / `email.promo.*` references survive, shifted down (`281-283/296-298/307-309 → 293-295/308-310/319-321`). The mobile half of B4 is confirmed: if the backend SQL deleted `required.label`, `required.description`, `required.sub.description`, `config.label`, `config.description` from the COOKIES seed, these five mobile references will resolve to absent keys (raw key string or empty label, per i18n fallback) on the "Strictly necessary" + "Preference cookies" sections. The pairing with the backend audit determines which references break vs resolve.

### C6 — Dead types (re-confirmed — both deletable)

Both files exist at the flagged paths. No barrel (`src/lib/types/cookie/index.ts` does not exist).

- **`src/lib/types/cookie/ConsentData.ts`** (`:1` `export type ConsentData = {`). Only importer is `GlobalCookie.ts:2` (used at `GlobalCookie.ts:7` `cookieConsent: ConsentData;`).
  ```
  $ grep -rn "ConsentData" . --exclude-dir=node_modules --exclude-dir=.git
  src/lib/types/cookie/GlobalCookie.ts:2:import { ConsentData } from './ConsentData';
  src/lib/types/cookie/GlobalCookie.ts:7:  cookieConsent: ConsentData;
  src/lib/types/cookie/ConsentData.ts:1:export type ConsentData = {
  ```
- **`src/lib/types/cookie/GlobalCookie.ts`** (`:6` `export type GlobalCookie = {`; also `:4` `export type Lang = 'sr' | 'en';`). Zero importers of `GlobalCookie`, of its `Lang` re-export, or of the file path. Only non-self references are two prose rows in `jobs/rn_sync/FRONTEND-CHANGES-FOR-RN.md` (a docs markdown table, not a TS import).
  ```
  $ grep -rn "from.*cookie/GlobalCookie" --include="*.ts" --include="*.tsx" src/ app/   → (none)
  $ grep -rn "types/cookie" --include="*.ts" --include="*.tsx" src/ app/                → (none)
  ```

**Verdict:** prior audit's claim is **confirmed** — `ConsentData` is imported only by `GlobalCookie.ts`, and `GlobalCookie.ts` is imported by nothing outside itself. Both form a closed dead chain, **deletable together** (the `Lang` re-export also has zero importers, so deletion is safe).

### C7 — B17 / B16 / B18 on the settings surface (re-confirmed)

1. **B17 — `"DODAJ ZA BASE SITE"`: STILL PRESENT.** `user.tsx:242` — `<Text>DODAJ ZA BASE SITE</Text>`. It is a **rendered, untranslated placeholder/scaffold label** (a `<Text>` child of a bare `<View>` at `:241-243`) sitting between the profile-fields card and the region card — not a button. **Delta:** prior `:230 → :242`.
2. **B16 — `console.*` in `user.tsx`: ONE.** `user.tsx:74` — `.catch((error) => console.error(error));` (error handler of the seeding promise). **Delta:** prior `:74 → :74` (unchanged).
3. **B18 — `var` in `user.tsx`: GONE.** `rg -n "\bvar\b" app/owner/dashboard/user.tsx` returns no matches. The line the prior audit flagged is now `let toastMessage = …` (`:188`); `let result;` is at `:165`. **Delta:** prior `:178 var → now let` (the B18 item on this surface no longer exists).

### C8 — AuthUserDTO / UpdateUserDTO overlap with chats E and F

**`src/lib/types/user/AuthUserDTO.ts` — 13 fields:** `id` `:5`, `firebaseUid` `:6`, `displayName` `:7`, `email` `:8`, `baseSite` `:9`, `regionAndCity` `:10`, `profileImageKey` `:11`, `providerId?` `:12`, `allowPreferenceCookies?` `:13`, `allowNotifications?` `:14`, `allowEmails?` `:15`, `allowPromoEmails?` `:16`, `allowPhoneCalling` `:17`.

**`src/lib/types/user/UpdateUserDTO.ts` — 14 fields:** `id` `:4`, `firebaseUid` `:5`, `displayName` `:6`, `email` `:7`, `profileImageKey` `:8`, `shortBio` `:9`, `phoneNumber?` `:10`, `regionAndCity?` `:11`, `allowPreferenceCookies?` `:12`, `allowNotifications?` `:13`, `allowEmails?` `:14`, `allowPromoEmails?` `:15`, `allowPhoneCalling` `:16`, `providerId?` `:17`.

**E/F additions confirmed absent** (so C is removals only, no collision). `rg -n "\b(disabled|state|deletionStatus|wasRegister)\b"` on both DTOs returns no matches for any of E's planned `disabled`/`state`/`deletionStatus` or F's `wasRegister`. C's edits to these two DTOs (removing `allowPreferenceCookies`, and — if the "remove" decision wins — `allowNotifications`) do not touch any field E or F will add.

---

## PART G — Consent UI build-surface discovery

### G1 — AsyncStorage usage pattern

**Current key set** (corrects two prior-audit names — see notes):

| Key (literal) | Defined/used at |
|---|---|
| `auth-storage` | `src/lib/store/authStore.ts:257` (Zustand persist name) |
| `access_token` | `src/lib/storage/authStorage.ts:3` (get/set/remove `:6/:10/:14`) |
| `base_site` | `src/lib/init/baseSitesService.ts:8` (get `:12`, set `:17`) |
| `app_language` | `src/i18n/storage.ts:5` (get `:9`, set `:14`) |
| `card-size-preferences` | `src/lib/store/useCardSizeStore.ts:16` (get `:34`, set `:61`) |
| `translations_<ns>_<lang>` | `src/lib/store/checksumStorage.ts:29` (payload; get `:59`, set `:68`) |
| `translations_checksum_<ns>_<lang>` | `src/lib/store/bootFreshness.ts:47` (used in `checksumStorage.ts:36,44`) |
| `base_site_catalog_checksum` | `src/lib/store/checksumStorage.ts:26` (get `:48`, set `:52`) |
| `dismissed_optional_update_version_data` | `src/lib/store/softUpdateDismissal.ts:16` (get `:25`, remove `:33`, set `:45`) |
| `SPPL` | `src/notifications/components/PushNotificationsInit.tsx:52` (get `:55`, set `:154`) |
| `@user_pref:<key>` (prefix) | `src/lib/store/userPreferenceStorage.ts:5` (generic wrapper — see below) |

Plus `src/lib/client/firebaseClient.ts:49` passes AsyncStorage into `getReactNativePersistence(...)` (Firebase-managed, no app key).

Prior-name corrections: "BASE_SITE" → literal `base_site`; **"APP_VERSION_CONFIG" is NOT an AsyncStorage key** (the only match is a dialog enum, `dialogRegistry.ts:19`); "translations_*" is two shapes (payload + checksum).

**Wrapper vs direct (point 2):** Both exist, but the **prevailing house style is a dedicated per-concern module with its own key constant calling `AsyncStorage.getItem/setItem/removeItem` directly** (e.g. `authStorage.ts:6`, `i18n/storage.ts:9`, `softUpdateDismissal.ts:25`). A generic wrapper `userPreferenceStorage` (`src/lib/store/userPreferenceStorage.ts:7`, JSON-encodes + `@user_pref:` namespaces every key) **exists but has zero callers** outside its own file — so it is *not* the de-facto convention despite being available.

**Zustand-persist-backed-by-AsyncStorage (point 3):** Confirmed — **only the auth store** uses it. `src/lib/store/authStore.ts`: `persist` import `:4`, wrap `:64`, config `:256-263` (`name: 'auth-storage'` `:257`, `storage: createJSONStorage(() => AsyncStorage)` `:258`, `partialize: (state) => ({ user: state.user })` `:259`, `onRehydrateStorage` sets `_hasHydrated` `:260-262`). `useCardSizeStore` does NOT use the middleware — it persists manually via direct calls.

**For a `useConsentStore`:** two house-style templates exist — (a) the Zustand `persist` middleware pattern (`authStore.ts:256`) for a reactive slice that components subscribe to, or (b) a dedicated per-concern module + key constant + direct `AsyncStorage` calls (`authStorage.ts` / `softUpdateDismissal.ts`) for a simple read-once gate. The unused `userPreferenceStorage` wrapper is **not** the convention.

### G2 — Where a consent gate would sit for analytics (the F-facing signal)

1. **No analytics SDK / init today — CONFIRMED.** Zero hits in `src/`+`app/` for `@react-native-firebase/analytics`, `expo-firebase-analytics`, `logEvent`, `setAnalyticsCollectionEnabled`.
2. **`src/lib/client/firebaseAnalytics.ts` is DEAD — CONFIRMED.** Sole export `getFirebaseAnalytics` (`:6`) has zero callers; the module is never imported. It uses the **web** `firebase/analytics` SDK gated on `typeof window !== 'undefined'` (never true on native RN), so it could not function on mobile even if called. `rg -n "getFirebaseAnalytics" src/ app/` → definition line only; `rg -n "firebaseAnalytics" src/ app/` → no import hits.
3. **Where init mounts.** `app/_layout.tsx` `RootLayout` is the single entry. Init mounts gated on `bootStatus`: `:77` `{bootStatus === 'ready' && <AppInit />}`, `:78` `<MaintenancePollInit />`, `:79` `<OfflineReconnectInit />`; plus a module-scope `registerAuthInterceptors()` at `:25`. **`AppInit` (`src/components/init/AppInit.tsx`) is the "ready"-state fan-out** — children mounted unconditionally inside it at `:22-28`: `AccountStateDialogsInit`, `CardSizeInit`, `ChatsInit`, `ForegroundRevalidationInit`, `PushNotificationsInit`, `NotificationsInit`, `InitFavoritesStore` (plus `initAuthListener()` in its own effect `:15-18`).

   **"Here" for F's consent-gated analytics init:** a new child of `AppInit` (`AppInit.tsx:21-29`, alongside `PushNotificationsInit`), which runs only once `bootStatus === 'ready'` (`app/_layout.tsx:77`). That child reads G's consent signal. The dead `firebaseAnalytics.ts` cannot be reused — a real RN analytics init needs the native SDK.

### G3 — Where a consent UI surface would mount

1. **First-launch / onboarding:** `src/components/init/BaseSiteSelector.tsx` exists. It renders a full-screen intro/boot-gate over a darkened `intro.jpg` with logo + `INTRO`-namespace copy, in one of two states: base-site picker (`:134-152`, buttons calling `setBaseSiteForCode(code)` `:139`) or message-only (`:116-124`, offline/maintenance copy). **Mounted from `app/_layout.tsx:119-121`**, gated on `bootStatus`: `offline` `:119`, `maintenance` `:120`, and **`intro-picker` `:121`** (the first-run/onboarding path) — the natural slot for a first-run consent prompt.
2. **Dedicated screen:** current `app/owner/` route tree —
   ```
   app/owner/_layout.tsx
   app/owner/dashboard/{_layout,account-verification,analytics,balance,follows,not-ready,reviews,user}.tsx
   app/owner/dashboard/products/{[productId],index}.tsx
   ```
   There is **no** `cookies`/`privacy` slot under `app/owner/`. A "Privacy settings" screen would naturally drop into `app/owner/dashboard/privacy.tsx`, alongside `user.tsx` (the flat settings-page group wired into the sidebar). (The public read-only policy pages live under `(public)` — see G4 — and are a different thing.)
3. **Footer / sidebar nav (both exist):**
   - **Footer `src/lib/navigation/companyNavigations.tsx`** (consumed at `Footer.tsx:45`): `about.label`→`/about` (`:2-5`), `pricing.label`→`/pricing` (`:6-9`), `privacy.label`→`/privacy` (`:10-13`), `terms.label`→`/terms` (`:14-17`). A "Privacy settings" entry slots next to `privacy.label`.
   - **Sidebar `src/lib/navigation/dashboardNavigations.tsx`** (consumed at `DashboardSidebar.tsx:121`): `owner.products.label`→`/owner/dashboard/products` (`:5-10`), `owner.balance.label`→`/owner/dashboard/balance` (`:11-16`), `owner.promo.sub.label` collapsible (`:17-36`), **`owner.account.sub.label` collapsible (`:37-63`)** with sub-items user/shop/reviews/follows/verification, `owner.analytics.label`→`/owner/dashboard/analytics` (`:64-69`). A "Privacy settings" sidebar item naturally goes as a new sub-item inside the `owner.account.sub.label` group, pointing at the `/owner/dashboard/privacy` slot from point 2.

### G4 — Privacy policy / terms surfaces (for linking from consent UI)

Both exist at the **prior-audit paths (unchanged)**; the `(portal)/(public)` segments are route groups, so resolved routes are `/privacy` and `/terms`.

- `app/(portal)/(public)/privacy.tsx` — `PrivacyScreen` (`:6`); renders `BackToHomeButton` + `MarkdownViewer` + `Footer` in a `ScrollView`; markdown URL `:12` = `https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy.md`.
- `app/(portal)/(public)/terms.tsx` — `TermsScreen` (`:6`); same structure; markdown URL `:12` = `.../terms.md`.
- `src/components/MarkdownViewer.tsx` takes a `url` prop, **fetches it at view time over the network** (`:26` `await fetch(url)`, `:28` `setContent(await res.text())`, rendered via `react-native-markdown-display`); error key `markdown.fild.load` on failure (`:40`). Content is **not bundled** — requires network at view time.
- Canonical link targets: `companyNavigations.tsx:12` `/privacy`, `:16` `/terms`. A consent UI links via `router.push('/privacy')` / `Link href="/privacy"`.

### G5 — ATT boundary (informational, for the G/F split)

ATT side **entirely absent** — confirms the prior audit. `rg -n "expo-tracking-transparency|requestTrackingPermission|TrackingTransparency" src/ app/` → no hits; `rg -n "expo-tracking-transparency" package.json` → no hits. No import, no permission-request call, no dependency. ATT stays chat F's job; G's spec documents the boundary.

### G6 — i18n: how mobile gets translation strings

1. **Fetch + register.** Per-namespace fetcher `src/i18n/fetchNamespace.ts:26-48`, endpoint `:31-33` `BACKEND_API.get('/public/translations?namespace=${namespace}&lang=${lang}')` — one GET per `(namespace, lang)`, flat `{key,value}[]` → nested via `toNested`. Registration into the single i18n instance happens in `src/lib/store/bootStore.ts` (Gate 4): refetch-when-stale `:491`, `i18n.init({ ns: Object.values(TranslationNamespace), defaultNS: COMMON_SYSTEM })` on first boot `:528-552`, incremental `i18n.addResourceBundle(...)` for updates. Consumer hook: `src/i18n/useTranslations.ts:3-6` (wraps `react-i18next`'s `useTranslation`).
2. **Namespaces fetched / COOKIES included.** Enum `TranslationNamespace` at `src/i18n/types.ts:6-38` — **20 members**, with **`COOKIES = 'COOKIES'` at `types.ts:27`**. The boot freshness gate iterates **every** member: `bootStore.ts:451` `for (const ns of Object.values(TranslationNamespace))`. There is no curated subset — the local enum IS the fetch list (comment `bootStore.ts:446-448`: fetching is "driven by the LOCAL enum (20 entries), never the backend's 22-key set"). Registration uses the same full list (`bootStore.ts:541`).
3. **Verdict:** **COOKIES is already fetched, cached, freshness-checked, and registered** on the active language — **so G's new consent strings seeded under `COOKIES` need ZERO mobile fetch/namespace change.** The only decision point: reuse `COOKIES` → zero mobile change; introduce a *new* namespace (e.g. `CONSENT`) → a one-line enum addition in `src/i18n/types.ts` (everything downstream picks it up automatically). Caveat for the spec: the backend must emit a checksum + strings for whatever namespace the consent keys land in (`bootStore.ts:455-473` treats a missing backend checksum for a known namespace as proceed-with-cache or route-to-maintenance).

---

## Section Z — Trust boundary check (Part 11)

The current chat changes no trust decision. (1) The cleanup removes `allowPreferenceCookies`, a field the backend already ignores end-to-end post-consent-mode-v2 (the mobile send at `user.tsx:176` is silently dropped by Jackson; the response-read at `:67` is always `undefined`). Mobile makes **no local trust decision** off any consent value: `allowPreferenceCookies` gates nothing on device, and the remaining preference fields (`allowEmails`, `allowPromoEmails`, `allowPhoneCalling`) are server-stored, server-enforced communication preferences — the client supplies a preference, the server is the source of truth (correct pattern; `allowPhoneCalling` read-back at `ProductFunctions.tsx` is display-gating only, not a client trust decision). (2) G's stored consent decision is **device-local** (AsyncStorage) and read **client-side only** to gate the client's own analytics init — it is not sent to the server for any moderation, authorization, or state-transition decision. No trust boundary is crossed by either C or G.

---

## For Mastermind

### Stale-audit delta summary (the #1 deliverable)

Prior audit = `.agent/audit-expo-readiness-consent-mode.md`, 2026-05-23, on `dev`. Status on `new-expo-dev` today:

| Item | Prior claim (2026-05-23) | Status on new-expo-dev | Current file:line |
|---|---|---|---|
| **B-`allowPreferenceCookies` inventory** | 6 occ / 3 files | **HOLDS** (files same; now 8 occ — line-split, not new logic) | `AuthUserDTO.ts:13`, `UpdateUserDTO.ts:12`, `user.tsx:46,67,97,176,286,287` |
| **Φ4 grenade (updateUser no try/catch)** | `:163`, NO try/catch, throws uncaught | **DOES NOT HOLD — defused** | try/catch at `user.tsx:166-186`, call `:167`; catch re-throws `:185` |
| **B2 `allowPromoEmails` init bug** | `:70` overwrites with phone-calling | **HOLDS** | `user.tsx:69-70` |
| **B3 `allowNotifications` dead toggle** | declared/rendered/never seeded/never sent | **HOLDS** (all 4 facts) | state `:47`, render `:299-303`; DTO field exists `UpdateUserDTO.ts:13` |
| **B4 five deleted COOKIES keys (mobile refs)** | refs at `254/255/257/267/268` | **HOLDS** (refs survive, shifted) | `user.tsx:266,267,269,279,280`; hook `:35` |
| **Other COOKIES keys (notifications/email/promo)** | "likely still exist", `281-283/296-298/307-309` | **HOLDS** (confirmed referenced) | `user.tsx:293-295,308-310,319-321` |
| **Dead types ConsentData / GlobalCookie** | closed dead chain, deletable | **HOLDS** | `src/lib/types/cookie/ConsentData.ts`, `GlobalCookie.ts` |
| **B17 `"DODAJ ZA BASE SITE"`** | `:230`, placeholder | **HOLDS** (rendered `<Text>` label) | `user.tsx:242` |
| **B16 `console.*` in user.tsx** | `console.error` at `:74` | **HOLDS** (one occurrence) | `user.tsx:74` |
| **B18 `var` in user.tsx** | `var toastMessage` at `:178` | **DOES NOT HOLD — gone** (now `let`) | `user.tsx:188` `let toastMessage`; no `var` in file |

**Net:** every C cleanup target survives the Φ1–Φ4 foundation work **except two that the foundation already fixed** — the Φ4 grenade (a try/catch now wraps the save call) and B18 (`var → let`). C's actual edit scope is: remove `allowPreferenceCookies` from `AuthUserDTO`/`UpdateUserDTO`/`user.tsx`; decide remove-vs-wire on `allowNotifications`; fix or remove the B2 double-set; remove the dead `ConsentData`/`GlobalCookie` chain; translate/remove the B17 placeholder; remove the B16 `console.error`. The five COOKIES key references are the cross-repo pairing point with the backend SQL audit.

### G build-surface summary (for the Phase 4 spec)

- **AsyncStorage pattern:** prevailing house style = dedicated per-concern module + own key constant + direct `AsyncStorage` calls. The generic `userPreferenceStorage` wrapper exists but is **unused** (zero callers) — not the convention.
- **Consent-store pattern:** the only Zustand-persist-backed-by-AsyncStorage precedent is the auth store (`authStore.ts:256-263` — `persist` + `createJSONStorage(() => AsyncStorage)` + `partialize` + `onRehydrateStorage`). A `useConsentStore` can mirror this (reactive slice) or use the simpler per-concern-module template (read-once gate).
- **Analytics-init mount point (F-facing):** a new child of `AppInit` (`src/components/init/AppInit.tsx:21-29`), which mounts only when `bootStatus === 'ready'` (`app/_layout.tsx:77`). The dead web-only `firebaseAnalytics.ts` cannot be reused. F's init reads G's signal here.
- **Candidate UI mounts:** first-run = the `intro-picker` boot state rendering `BaseSiteSelector` (`app/_layout.tsx:121`); settings = post-cleanup `user.tsx` or a new `app/owner/dashboard/privacy.tsx` screen (no such slot exists today); nav entries = footer `companyNavigations.tsx` (next to `/privacy`) and/or the `owner.account.sub.label` sidebar group in `dashboardNavigations.tsx`. Policy links: `/privacy` + `/terms` (network-fetched markdown, `privacy.md`/`terms.md`).
- **i18n/namespace situation:** COOKIES is already fetched (boot loops the full 20-member enum; `COOKIES` at `types.ts:27`). New consent strings under `COOKIES` → **zero mobile change**. A new namespace → one-line enum add in `src/i18n/types.ts`. Backend must emit a checksum for whatever namespace is chosen.
- **ATT boundary:** entirely absent (no package, no code). Stays chat F; G's spec documents it.

### Part 4b — adjacent observations (file:line, severity)

1. **Save-throw path is silent to the user** — `app/owner/dashboard/user.tsx:181-186` + handler `:348`. The catch re-throws `err`; `saveChanges` runs as a fire-and-forget `onPress`, so a thrown save error becomes an unhandled promise rejection with **no error toast** (only the non-throw `!result` path at `:190-196` toasts). Severity: **medium** — a backend failure on profile save shows the user nothing. Relevant to C since C edits this function. Not fixed (read-only audit).
2. **B2 double-set `setAllowPromoEmails`** — `user.tsx:69-70`. Severity: **medium** (user sees wrong promo-email preference on load). In C's scope.
3. **`allowNotifications` dead toggle** — `user.tsx:47,299-303`. Severity: **low** (renders, does nothing). Remove-vs-wire decision for C.
4. **B16 `console.error`** — `user.tsx:74`. Severity: **low** (Part 4 ad-hoc logging). In C's scope.
5. **B17 untranslated placeholder `"DODAJ ZA BASE SITE"`** — `user.tsx:242`. Severity: **low** (Part 6 hardcoded string, dev scaffold). In C's scope.
6. **Dead web-only `firebaseAnalytics.ts`** — `src/lib/client/firebaseAnalytics.ts`. Severity: **low** (zero callers; `typeof window` web guard cannot run on RN). C/Ω-adjacent dead code; F will build a real native init instead.
7. **Dead `ConsentData`/`GlobalCookie` cookie types** — `src/lib/types/cookie/`. Severity: **low** (closed dead chain). In C's scope.
8. **Unused `userPreferenceStorage` wrapper** — `src/lib/store/userPreferenceStorage.ts`. Severity: **low** (a generic AsyncStorage wrapper with zero callers; either adopt it for the consent store or it's dead). Flagged for the G spec / Ω sweep.

### Part 4a — simplicity evidence

- Added (earned complexity): nothing — read-only audit, no code written.
- Considered and rejected: nothing — no code written.
- Simplified or removed: nothing — no code written.

### Config-file impact

**Config-file impact: none (read-only audit).** No `conventions.md` / `decisions.md` / `state.md` / `issues.md` edit is required or drafted by this session. The adjacent observations above are surfaced for Mastermind to triage into `issues.md` at its discretion; this session does not draft those entries. No implicit config-file dependency is left unstated.
