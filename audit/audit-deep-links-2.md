# Audit (re-audit) — oglasino-expo: Deep Linking + `+native-intent` readiness

**Repo:** oglasino-expo
**Branch:** `new-expo-dev` (confirmed `git rev-parse --abbrev-ref HEAD`)
**Type:** READ-ONLY audit. No code changed, no config edited, no native files touched, no git ops, no builds.
**Date:** 2026-06-04
**Method:** Code read fresh from disk on `new-expo-dev`. All native claims verified against the **generated** files (`ios/OglasinoDev/Info.plist`, `ios/OglasinoDev/OglasinoDev.entitlements`, `android/app/src/main/AndroidManifest.xml`), not inferred from `app.config.ts`. expo-router `+native-intent` support verified against the **installed package** in `node_modules`, not from docs. The prior `audit-deep-links.md` was **not** opened until all findings below were formed; the "Diff vs prior audit" section was written last.

---

## Keystone summary (read this first)

- **Product** route → app path `/product/{id}/{slug}`, catch-all `[...productData]`, **id at index 0** (`productData[0]`). Confirmed.
- **User** route → app path `/user/{id}/...`, catch-all `[...userData]`, id at index 0 (`userData[0]`).
- **Catalog** route → app path `/catalog/{...categoryRoutes}`, catch-all `[...categories]`; segments matched against each category's `route` field.
- **There is NO locale segment in the app route tree.** App paths are `/product/{id}/{slug}`; web's canonical shareable URL is `https://oglasino.com/{locale}/product/{id}/{slug}` (locale = compound base-site+language, e.g. `rs-sr`). This is the mismatch the rewrite hook must reconcile.
- **`+native-intent` with `redirectSystemPath` IS supported** by the installed `expo-router@6.0.24` (verified in `node_modules/expo-router/build/getLinkingConfig.js`). The file `app/+native-intent.ts` does **not** currently exist.

The rewrite hook must reconcile: strip a leading routing-locale segment (e.g. `rs-sr`) from inbound `https://oglasino.com/{locale}/product/{id}/{slug}` → app route `/product/{id}/{slug}`.

---

## 1. Custom scheme — current state

**Per-environment scheme, defined in config** (`app.config.ts:31-32`, applied at `:40`):

```ts
const appScheme =
  ENV === 'production' ? 'oglasino' : ENV === 'preview' ? 'oglasino-preview' : 'oglasino-dev';
// ...
scheme: appScheme,
```

So config defines `oglasino` (prod) / `oglasino-preview` (preview) / `oglasino-dev` (dev).

**Generated native files — verified directly (these carry only the DEVELOPMENT tier, because prebuild last ran with `APP_ENV=development`):**

- **iOS** `ios/OglasinoDev/Info.plist:25-46` — `CFBundleURLTypes` has **three** URL-type dicts:
  - `CFBundleURLSchemes` = `oglasino-dev` (`:30`) and `com.oglasino.development` (`:31`)
  - `com.googleusercontent.apps.1091292210835-…` (`:37`) — Google Sign-In reversed client id
  - `exp+oglasino-expo` (`:43`) — Expo dev-client/Expo Go tunneling scheme
  - **The production scheme `oglasino` is NOT present** in the generated Info.plist (only `oglasino-dev`).
- **Android** `android/app/src/main/AndroidManifest.xml:29-35` — one inbound `intent-filter` on `.MainActivity` with `action.VIEW` + `category.DEFAULT` + `category.BROWSABLE`, and two `<data>` schemes: `oglasino-dev` (`:33`) and `exp+oglasino-expo` (`:34`). **No `oglasino` (prod) scheme; no http/https host.**

**Does `oglasino://product/123/slug`-style routing work today?** Mechanically yes — expo-router auto-maps the `app/` tree to paths, so a custom-scheme URL whose path matches a route opens that screen. **But on the current build it is `oglasino-dev://product/123/slug`, not `oglasino://…`** — the prod scheme only lands in the native files when a production-profile prebuild/build runs. `app.config.ts` defines all three; only the dev one is currently generated on disk.

## 2. The linkable route surface

`expo-router` enabled: `"main": "expo-router/entry"` (`package.json:3`), plugin `'expo-router'` (`app.config.ts:89`). File tree (route groups in parentheses are stripped from the URL):

| App route file | Deep-link path | Dynamic params / notes |
|---|---|---|
| `app/(portal)/(public)/index.tsx` | `/` | Home product feed |
| `app/(portal)/(public)/product/[...productData].tsx` | `/product/{id}/{slug}` | **catch-all**; `productData[0]` = id → `parseInt` (`product/[...productData].tsx:55-62`) |
| `app/(portal)/(public)/user/[...userData].tsx` | `/user/{id}/...` | **catch-all**; `userData[0]` = id → `parseInt` (`user/[...userData].tsx:22-27`) |
| `app/(portal)/(public)/catalog/[...categories].tsx` | `/catalog/{...}` | **catch-all**; path segments matched to category `route` (`utils.ts:76-107`, `getCategoriesFromPath`) |
| `app/(portal)/(public)/about.tsx` | `/about` | static |
| `app/(portal)/(public)/pricing.tsx` | `/pricing` | static |
| `app/(portal)/(public)/privacy.tsx` | `/privacy` | static |
| `app/(portal)/(public)/terms.tsx` | `/terms` | static |
| `app/(portal)/(public)/blog/free-zone.tsx` | `/blog/free-zone` | static |
| `app/(portal)/(secured)/favorites.tsx` | `/favorites` | auth-guarded |
| `app/(portal)/(secured)/messages.tsx` | `/messages` | auth-guarded |
| `app/(portal)/(secured)/notifications.tsx` | `/notifications` | auth-guarded |
| `app/owner/products/[productId].tsx` | `/owner/products/{productId}` | auth-guarded; dashboard update screen |
| `app/owner/products/index.tsx`, `analytics`, `balance`, `follows`, `reviews`, `user`, `account-verification`, `not-ready` | `/owner/*` | auth-guarded |
| `app/+not-found.tsx` | (fallback) | unmatched paths |

**Product is confirmed catch-all `[...productData]` with id at index 0** (`product/[...productData].tsx:61`: `const productIdParam = productData?.[0];`).

The three obvious public link targets are **product, user, catalog**. Static pages (about/pricing/privacy/terms/free-zone) are also link-addressable. Secured/owner routes are reachable by path but guarded.

## 3. Locale segment — the keystone

**Confirmed: the app route tree has NO locale segment.** There is no `[locale]` or `${locale}` directory anywhere under `app/`. The product path is literally `/product/{id}/{slug}` (built by `getNormalizedProductUrl` in the relative form, `utils.ts:137`: `` `/product/${productId}/${slug}` ``).

Language/base-site are **runtime state, not route state** — they come from `useBootStore` (`selectedBaseSite.code`, `language.code`), e.g. `ProductScreenContent` reads `useBootStore`, and `ShareProductButton.tsx:33` composes `` `${selectedBaseSite.code}-${language.code}` `` (e.g. `rs-sr`) only when *building outbound web URLs*. Nothing in the route tree consumes a locale path segment.

**The mismatch, plainly:** web's canonical shareable product URL is

```ts
// utils.ts:136 (withPrefix=true overload)
`https://oglasino.com/${locale}/product/${productId}/${slug}`
```

A user copies `https://oglasino.com/rs-sr/product/123/slug`. The leading `/rs-sr` segment does not exist in the app's route model, so an as-is inbound match would fail (land on `+not-found`). A `+native-intent` `redirectSystemPath` hook stripping the locale prefix is the keystone that makes verified inbound links resolve.

## 4. expo-router version + `+native-intent` support

- **Installed (verified from `node_modules`):** `expo-router@6.0.24`, `expo@54.0.35`, `expo-linking@8.0.12`. (`package.json` declares ranges `expo-router ~6.0.23`, `expo ~54.0.33`, `expo-linking ~8.0.11`; the resolved/installed versions are the `.24`/`.35`/`.12` above.)
- **`+native-intent` / `redirectSystemPath` IS supported.** Verified directly in the installed build, not from docs:
  - `node_modules/expo-router/build/getLinkingConfig.js` resolves the file via regex `/^\.\/\+native-intent\.[tj]sx?$/` and, when present, calls `nativeLinking.redirectSystemPath({ path, initial: true })` inside `getInitialURL` (`:71-72`, `:78-79`) and threads `nativeLinking` into `subscribe(...)`.
  - `node_modules/expo-router/build/types.d.ts:41` / `ExpoRoot.d.ts:11` define the `NativeIntent` type with `redirectSystemPath?: (event: { path; initial }) => Promise<string> | string` (plus experimental `legacy_subscribe`).
- **`app/+native-intent.ts` does NOT currently exist** (confirmed: `ls app/+native-intent.ts` → absent; no `+native-intent.*` file anywhere under `app/`). The keystone hook is available to add but unbuilt.

## 5. Universal/App Links config — current state (what's ABSENT)

Verified absent, directly:

- **iOS Universal Links — absent.** No `ios.associatedDomains` in `app.config.ts` (the `ios` block is `:48-63`; no associatedDomains key). The iOS entitlements file `ios/OglasinoDev/OglasinoDev.entitlements` contains **only** `aps-environment = development` — no `com.apple.developer.associated-domains`. Repo-wide grep for `associatedDomains` / `applinks:` / `associated-domains` (excluding node_modules/Pods/build) → **zero matches**.
- **Android App Links — absent.** The only inbound `intent-filter` (`AndroidManifest.xml:29-35`) carries custom schemes only (`oglasino-dev`, `exp+oglasino-expo`). **No `android:autoVerify="true"`, no `android:host`, no `https` data in an intent-filter.** Grep for `autoVerify` → zero matches.
- **The `https` distinction (verified independently):** `AndroidManifest.xml:8-14` has `<queries><intent>…<data android:scheme="https"/></intent></queries>`. This is an **outbound capability declaration** (lets the app query/launch `https` URLs in other apps, e.g. for `Linking.openURL`/`canOpenURL` package visibility on Android 11+). It is **NOT** an inbound `intent-filter` and does **not** make the app a handler for `https://oglasino.com/...`. The prior audit drew this distinction; I confirm it independently — the `https` entry is inside `<queries>` (line 12), structurally separate from the `<activity>`'s `<intent-filter>` block (lines 25-35).
- **`.well-known` association files** (`apple-app-site-association`, `assetlinks.json`) — not in this repo (they belong on the web domain, not here). Not applicable to audit of this repo beyond noting the cross-repo dependency.

## 6. Existing inbound-link / navigation engine

**The proven navigation layer** is `src/notifications/components/PushNotificationsInit.tsx`. `handleNavigation(category, data)` (`:75-105`) routes notification taps by category:

```ts
case 'MESSAGE':         router.push('/messages'); break;
case 'SAVED_PRODUCT':   if (data?.productId)
                          router.push(getNormalizedProductUrl(data.productId, data.productName ?? '')); break; // relative form → /product/{id}/{slug}
case 'NAVIGATION':      if (data?.navigate) router.push(data.navigate); break;       // arbitrary in-app path from backend
case 'PRODUCT_EXPIRATION':
case 'PRODUCT_EXPIRED': router.push('/notifications'); break;
default:                /* no blanket redirect — stay on booted screen (home) */ break;
```

So destinations a universal link would target (`/product/...`, `/messages`, `/notifications`, and arbitrary backend-supplied `data.navigate`) are already exercised and stable. A universal-link entry point would feed the **same** `router.push` targets — no new navigation layer needed.

**Cold-start dedup (quoted):** `handleResponse(response, fromBoot)` (`:114-135`) computes `const dedupKey = notification.request.identifier ?? \`date:${notification.date}\`;` and, when `fromBoot`, reads `AsyncStorage.getItem(LAST_HANDLED_RESPONSE_KEY)` (`LAST_HANDLED_RESPONSE_KEY = 'LHNR'`, `:53`) — if `lastHandled === dedupKey` it returns without navigating, then persists the key. This stops a stale `getLastNotificationResponse()` from re-firing on every cold start (`:107-135`, the boot effect at `:145-155`).

**Is there any `Linking.getInitialURL` / `Linking.addEventListener` inbound handler?** **No.** `react-native`/`expo-linking` `Linking` is used **outbound only**: `Linking.openURL` to leave the app — `tel:` in `CallUserButton.tsx`, ko-fi in `SupportButton.tsx`, external URLs in `HardUpdateScreen`/`SoftUpdateModal`, message-link handling in `messages/utils.ts`/`Message.tsx`, the password-reset funnel in `LoginDialog.tsx` (§7), and app-store/version links in `AppVersionConfigurationDialog.tsx`. There is **no inbound** `getInitialURL`/`addEventListener` handler and **no custom `linking` config / `getStateFromPath`** override — **expo-router owns all inbound routing.** (Notification taps go through the `expo-notifications` response listener, a separate channel from URL linking.)

## 7. The `appDeepLink` scaffold from password-reset

**There is NO inbound `appDeepLink`-equivalent on mobile.** The password-reset path is a one-way **outbound funnel to web**, not an inbound deep link.

Mechanism (`LoginDialog.tsx:79-84`, `openPasswordReset`):

```ts
const { selectedBaseSite, language } = useBootStore.getState();
const compoundLocale =
  selectedBaseSite && language ? `${selectedBaseSite.code}-${language.code}` : null;
Linking.openURL(getForgotPasswordUrl(Constants.expoConfig?.extra?.env, compoundLocale));
```

`getForgotPasswordUrl(env, compoundLocale)` (`utils.ts:150-158`) is **tier-aware**:

```ts
const webBase = env === 'production' ? 'https://oglasino.com' : 'https://stage.oglasino.com';
return compoundLocale ? `${webBase}/${compoundLocale}/forgot-password` : `${webBase}/forgot-password`;
```

So the mobile "Reset password" link opens the device browser at web's `/{locale}/forgot-password` (prod → `oglasino.com`, preview/dev → `stage.oglasino.com`), composing the same `rs-sr`-style compound locale that `ShareProductButton` uses. It fires **no auth call** and never re-enters the app — the user completes reset on web. There is no return deep link back into the app, no inbound handler, and no `appDeepLink` token.

**How this feature must reconcile with it:** the locale-composition logic (`${baseSite.code}-${language.code}`) is the *same* compound-locale shape a universal link would arrive carrying. When `+native-intent` strips an inbound locale prefix, the "should the link switch app language?" decision (§ For Mastermind) is the inbound mirror of what `getForgotPasswordUrl`/`ShareProductButton` do outbound. The two URL builders (`getForgotPasswordUrl` tier-aware, `getNormalizedProductUrl` prod-hardcoded) are also the natural unification point — see §8.

## 8. `expo-linking` availability

**`expo-linking` is already a dependency.** `package.json:53` declares `~8.0.11`; installed/resolved `8.0.12` (verified from `node_modules/expo-linking/package.json`). **No new package is needed for the JS side** of inbound linking.

## 9. Per-environment / bundle IDs

Per-tier bundle IDs (`app.config.ts:7-12`):

| Tier | Bundle ID | Scheme | App name |
|---|---|---|---|
| **production** | `com.oglasino` | `oglasino` | Oglasino |
| preview | `com.oglasino.preview` | `oglasino-preview` | Oglasino Preview |
| development | `com.oglasino.development` | `oglasino-dev` | Oglasino Dev |

Both iOS (`ios.bundleIdentifier`, `:50`) and Android (`android.package`, `:65`) use the same `bundleId` per tier. **Universal/App Links are domain-bound** (`oglasino.com`), so v1 realistically attaches to **production only**: bundle `com.oglasino` + scheme `oglasino`. (The currently-generated native files on disk are the **development** tier — `com.oglasino.development` / `oglasino-dev` — so universal links cannot be exercised on the present dev build regardless; they require a production prebuild/build plus web-hosted association files.)

---

## Diff vs prior audit (`audit-deep-links.md`, 2026-06-03)

I formed all findings above from code first, then opened the prior audit. **They agree on every substantive point** (custom scheme works per-tier; route tree is the linkable surface with product as catch-all id-at-0; no locale segment — the central blocker; no Universal/App Links; the `https`-in-`<queries>` vs inbound-intent-filter distinction; notification engine is the proven nav layer with `LHNR` cold-start dedup; all `Linking` usage is outbound; `expo-linking` already present; production-only domain binding). Differences are **precision additions**, not contradictions:

1. **Generated native files carry ONLY the dev-tier scheme.** The prior audit's §1.1 correctly lists `oglasino-dev` in the Info.plist, but its TL;DR phrases the win as "`oglasino://product/123/slug` opens the app today." More precisely: on the *current on-disk build* the working scheme is **`oglasino-dev://…`**; the prod `oglasino` scheme is absent from the generated `Info.plist`/`AndroidManifest.xml` until a production-profile prebuild runs. I trust my version (verified `grep oglasino ios/.../Info.plist` returns only `oglasino-dev`, `com.oglasino.development`, `exp+oglasino-expo`) — the distinction matters because nobody should expect the prod scheme (or universal links) to resolve on the dev build.

2. **Explicit confirmation that the INSTALLED expo-router supports `+native-intent`.** The prior audit *proposes* `app/+native-intent.ts` and assumes availability but does not verify it against the package. I verified `redirectSystemPath` is wired in `expo-router@6.0.24`'s `getLinkingConfig.js` (invoked in `getInitialURL` and `subscribe`). This closes the brief's Q4 explicitly — the keystone hook is confirmed available, not assumed.

3. **Google reversed-client-id URL scheme.** The prior audit's §1.1 lists the iOS schemes as `oglasino-dev`, `com.oglasino.development`, `exp+oglasino-expo` and omits the `com.googleusercontent.apps.…` scheme (`Info.plist:34-39`). Minor — that scheme is Google Sign-In, irrelevant to product links — but the inventory is incomplete; my §1 lists all three URL-type dicts.

4. **Password-reset scaffold (brief Q7) — not covered by prior audit.** The prior audit (dated 2026-06-03, same day as the password-reset issue) does not mention `getForgotPasswordUrl` / `LoginDialog.openPasswordReset`. My §7 documents it: it is an **outbound** tier-aware funnel to web with **no inbound return path**, and its compound-locale logic is the outbound mirror of the inbound locale problem.

5. **Share-URL host is prod-hardcoded across all tiers (pre-existing, logged).** `getNormalizedProductUrl` (`utils.ts:136`) hardcodes `https://oglasino.com` regardless of build tier, unlike the tier-aware `getForgotPasswordUrl`. Already tracked in `issues.md` (2026-06-03, medium, open). The prior audit cites `utils.ts:136` as "web-canonical" but does not flag the tier mismatch. Relevant here because a universal-link rollout's AASA path patterns and any "open in app" affordance should reconcile share-URL host with the chosen tier domains, and the two URL builders are the natural unification point.

6. **Unverified claim carried by the prior audit: "10 routing locales."** The prior audit repeatedly cites `.agent/audit-routing-locale-parity.md` for "10 routing locales." I did **not** re-verify that number from code (it is outside the deep-linking code surface, and the locale set is derived at runtime from base-site × language, not a static constant in this repo). I neither confirm nor dispute it — see "couldn't verify" below. The *structural* point (locale prefix is compound and must be stripped) holds regardless of the exact count.

**Net:** the prior audit is accurate and trustworthy on the architecture and the plan. Where we differ I trust my code-verified, more-precise version (items 1–3). Items 4–5 are coverage the prior audit lacks. Item 6 is a claim I flag as unverified rather than endorse.

---

## For Mastermind

- **Cross-repo dependencies (hard blockers for verified `https://` links):**
  - **Web must host** `https://oglasino.com/.well-known/apple-app-site-association` (iOS) and `/.well-known/assetlinks.json` (Android), `application/json`, no redirect. Mobile cannot place these. Without them, no app-side change makes verified links open the app. (Web brief, parallelizable with the in-repo `+native-intent` work.)
  - **Android `assetlinks.json` needs the release/Play-signing SHA-256 cert fingerprint** — sourced from EAS credentials (Igor/EAS), not obtainable by an engineer agent (no deploy/credential access).
  - **Native config edit** (`ios.associatedDomains` + `android.intentFilters` in `app.config.ts`) is required for Phase 2 and is a **forbidden config edit without an explicit brief** — not done here, only noted.
  - **Web alignment:** confirm web's emitted share URL shape (`https://oglasino.com/{locale}/product/...`) matches the AASA path patterns and the `+native-intent` strip logic. Reconcile the prod-hardcoded share host (`utils.ts:136`) against the chosen tier domains.

- **The language-switch product question (explicit, for Igor):** should an inbound `rs-en` link switch the app's language to English (and base-site to `rs`)? Mechanically the `+native-intent` hook can capture the stripped locale and the app already composes/decomposes this exact compound shape (`ShareProductButton`, `getForgotPasswordUrl`). It is a **product decision**, not a mechanical one: a user tapping a shared `rs-en` link arguably expects English; but silently switching an existing user's language/base-site from a tapped link may be unwanted. Recommend: strip-and-route in v1 (resolve the screen), and treat language adoption as a separate, explicit decision. The inbound design should mirror the outbound locale logic already in `ShareProductButton`/`getForgotPasswordUrl` for consistency.

- **Couldn't verify:**
  - The "10 routing locales" figure from `.agent/audit-routing-locale-parity.md` (Diff item 6) — not re-derivable from the deep-linking code surface; the valid locale set is runtime-derived from base-site × language in `useBootStore`, not a static list in this repo. Whoever builds `+native-intent` must decide whether to strip *any* leading two-token `xx-yy` segment heuristically or validate against a known/derived set (the prior audit's note 1 — don't hard-code the list — stands).
  - Universal-link end-to-end behavior cannot be validated from code or in a simulator (the OS verification handshake needs real devices + live `.well-known` files) — an on-device (Ψ) item once Phases 1–3 land.

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): **nothing** — read-only audit, no code/abstractions introduced.
  - Considered and rejected: **nothing** — no implementation decisions made.
  - Simplified or removed: **nothing** — no code changed.

- **Suggested `state.md` Expo-backlog row (Docs/QA to add — I do not write `state.md`):** there is currently no deep-linking row. If tracked: *"Deep linking (universal/app links) — not-started. Keystone `app/+native-intent.ts` locale-strip (in-repo, testable; expo-router 6.0.24 confirmed to support `redirectSystemPath`). Blockers: native config edit (`app.config.ts`), web-hosted `.well-known` files (cross-repo), release-cert SHA-256 (EAS). Production-domain-only. See `oglasino-expo/.agent/audit-deep-links-2.md`."*

---

## Conventions check

- **Part 4 (cleanliness):** N/A — read-only audit, no code, no imports, no debug logging, no files beyond this audit + the session summary.
- **Hard rules:** no commit/push/checkout; no config-file edits (`app.config.ts` / `eas.json` / native files untouched — only described); no cross-repo edits; no builds/deploys. Native claims verified **directly** against `Info.plist` / `OglasinoDev.entitlements` / `AndroidManifest.xml`, and `+native-intent` support verified against the installed `expo-router` build — not assumed from `app.config.ts` or docs.
- **Config-file impact:** none written. `app.config.ts` change and a `state.md` backlog row are *drafted/described only* for a future brief / Docs/QA.
