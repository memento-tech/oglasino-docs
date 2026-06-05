# Audit — oglasino-expo: Deep Linking (current state + what universal/app links would take)

**Repo:** oglasino-expo
**Branch:** `new-expo-dev` (confirmed via session git status)
**Type:** Read-only audit. No code changed. No config edited.
**Date:** 2026-06-03
**Triggered by:** Igor's question — "is deep linking available?" → "draft an audit, go deep, give suggestions on what and how."
**Scope:** Inventory what deep-linking actually works today (JS, config, and native layers), identify the gaps, and lay out a concrete, phased plan to add real shareable links (Universal Links / App Links). Code on `new-expo-dev` is ground truth.

---

## TL;DR verdict

- **Custom-scheme deep linking WORKS today.** A scheme is registered per environment in `app.config.ts` (`oglasino` / `oglasino-preview` / `oglasino-dev`), expo-router auto-maps the `app/` tree to paths, and the native manifests confirm the scheme is wired. So `oglasino://product/123/slug` opens the app on the right screen, app-installed-only.
- **Universal Links (iOS) and App Links (Android) are NOT configured.** No `associatedDomains` entitlement on iOS, no `autoVerify` https intent-filter with a host on Android, and no `.well-known` files. So real `https://oglasino.com/...` links **do not** open the app — they open the browser.
- **There is already a working in-app navigation engine for notifications** (`PushNotificationsInit.handleNavigation`) that routes by category + `data.navigate` / `data.productId`. Universal links would reuse the same `router.push` destinations — the routing targets already exist and are proven.
- **The single biggest technical blocker is a locale-prefix mismatch.** Web canonical URLs are `https://oglasino.com/${locale}/product/{id}/{slug}` (locale = a compound like `rs-sr`, 10 routing locales — see `.agent/audit-routing-locale-parity.md`). The mobile route tree has **no locale segment** (`/product/{id}/{slug}`). An incoming web URL therefore will **not** match an app route as-is. This must be solved with a path-rewrite hook (`app/+native-intent.ts`) before universal links can resolve correctly.
- `expo-linking` (`~8.0.11`) is already a dependency, so no new package is needed for the JS side.
- **This is NOT a mobile-only feature.** Custom-scheme links are fully mobile, but Universal/App Links are a **joint mobile + web** effort: web must host the two `.well-known` association files on `oglasino.com`, and that is a hard blocker — without it, no app-side change makes verified `https://` links open the app. See §7.

**Bottom line:** the app is "deep-link capable" but only via app-private custom schemes. To make **shareable https links** open the app — the thing users actually need (links in emails, SMS, the website's "open in app", marketing) — there is real work to do, and part of it is cross-repo (web must host two `.well-known` files; native config must change).

---

## 1. What exists today — evidence

### 1.1 Custom URL scheme (works)

`app.config.ts:31-40` sets an environment-specific scheme:

```ts
const appScheme =
  ENV === 'production' ? 'oglasino' : ENV === 'preview' ? 'oglasino-preview' : 'oglasino-dev';
// ...
scheme: appScheme,
```

Confirmed at the native layer (these are generated from config but were verified directly, per the "verify before relying on Read" rule):

- **iOS** `ios/OglasinoDev/Info.plist:28-43` — `CFBundleURLSchemes` registers `oglasino-dev`, `com.oglasino.development`, and `exp+oglasino-expo`.
- **Android** `android/app/src/main/AndroidManifest.xml:29-35` — one inbound `intent-filter` with `android:scheme="oglasino-dev"` and `exp+oglasino-expo`, `category BROWSABLE` + `DEFAULT`.

### 1.2 Route tree = the linkable surface (works, via expo-router)

`expo-router` is enabled (`app.config.ts:89`, `"main": "expo-router/entry"`). The file-based routes map to deep-link paths automatically. Current public/linkable paths (groups in parens are stripped from the URL):

| App route file | Deep-link path | Dynamic params |
|---|---|---|
| `app/(portal)/(public)/index.tsx` | `/` | — |
| `app/(portal)/(public)/product/[...productData].tsx` | `/product/{id}/{slug}` | catch-all; `productData[0]` = id (`product/[...productData].tsx:61`) |
| `app/(portal)/(public)/user/[...userData].tsx` | `/user/{id}/...` | catch-all; `userData[0]` = id |
| `app/(portal)/(public)/catalog/[...categories].tsx` | `/catalog/{...}` | catch-all category path |
| `app/(portal)/(public)/{about,pricing,privacy,terms}.tsx` | `/about` etc. | — |
| `app/(portal)/(public)/blog/free-zone.tsx` | `/blog/free-zone` | — |
| `app/(portal)/(secured)/{favorites,messages,notifications}.tsx` | `/favorites` `/messages` `/notifications` | auth-guarded |
| `app/owner/*` | `/owner/...` | auth-guarded |

Auth guards already cover deep-link entry into secured routes: `(portal)/(secured)/_layout.tsx`, `owner/_layout.tsx` redirect unauthenticated users to `/` via `<Redirect>` (decisions.md, Φ1 F13). So a deep link into `/messages` while logged out lands safely on home, not a crash.

### 1.3 Notification tap → navigation engine (works — and is the template for universal links)

`src/notifications/components/PushNotificationsInit.tsx:75-105` already routes taps by category:

```ts
function handleNavigation(category, data) {
  switch (category) {
    case 'MESSAGE':          router.push('/messages'); break;
    case 'SAVED_PRODUCT':    if (data?.productId) router.push(getNormalizedProductUrl(data.productId, data.productName ?? '')); break;
    case 'NAVIGATION':       if (data?.navigate) router.push(data.navigate); break;
    case 'PRODUCT_EXPIRATION':
    case 'PRODUCT_EXPIRED':  router.push('/notifications'); break;
    default:                 /* no blanket redirect */ break;
  }
}
```

Cold-start dedup is handled (`LHNR` AsyncStorage key, `getLastNotificationResponse()` re-fire guard — issues.md bug-batch-3, 2026-06-01). **Takeaway:** the destinations a universal link would target (`/product/...`, `/messages`, `/notifications`, arbitrary `data.navigate`) are already exercised and stable. Universal-link handling does not need a new navigation layer — it needs a new *entry* point feeding the same `router.push`.

### 1.4 `Linking` usage today is all outbound

`Linking.openURL` is used only to leave the app — `tel:` (`CallUserButton.tsx:65`), ko-fi (`SupportButton.tsx:12`), external sites (`HardUpdateScreen`, `SoftUpdateModal`, message link runs `messages/utils.ts`). There is **no** inbound `Linking.getInitialURL` / `addEventListener` handler, and **no** custom `linking` config or `getStateFromPath` — expo-router owns inbound routing. That's expected and fine.

---

## 2. What does NOT exist — the gaps

| Gap | Evidence | Consequence |
|---|---|---|
| **No iOS Universal Links** | No `ios.associatedDomains` in `app.config.ts`; no `applinks:` / `com.apple.developer.associated-domains` anywhere in `ios/` | `https://oglasino.com/...` opens Safari, never the app |
| **No Android App Links** | The only inbound intent-filter (`AndroidManifest.xml:29-35`) is the custom scheme; **no `android:host`, no `android:autoVerify="true"`**. The `https` entry at line 12 is inside `<queries>` (outbound capability only), not an intent-filter | `https://oglasino.com/...` opens Chrome, never the app |
| **No `.well-known` association files** | repo grep for `well-known` / `apple-app-site-association` / `assetlinks` → none (these live on the **web** domain anyway, not this repo) | Even if entitlements were added, the OS verification handshake would fail |
| **No path-rewrite hook** | `app/+native-intent.ts` NOT present (confirmed) | Web's locale-prefixed URLs can't be reconciled to the app's locale-less routes |
| **Locale-prefix mismatch** | Web emits `https://oglasino.com/${locale}/product/...` (`utils.ts:136`, and `.agent/audit-routing-locale-parity.md`); app routes have no `[locale]` segment | A raw web URL fails to match any app route |

---

## 3. The central problem: locale-prefix mismatch (read this before planning)

This is the one finding that turns "add an entitlement" into "design a small adapter."

- **Web canonical product URL** (`src/lib/utils/utils.ts:136`):
  `https://oglasino.com/${locale}/product/${productId}/${slug}` — `locale` is a compound such as `rs-sr` (base-site code + language code). Per `.agent/audit-routing-locale-parity.md`, web has **10 routing locales**, the segment is **required**, and a missing/invalid one **404s** on web.
- **Mobile route** for the same product:
  `app/(portal)/(public)/product/[...productData].tsx` → path `/product/${id}/${slug}` — **no locale segment**.

So the literal URL a user copies from the website (`/rs-sr/product/123/slug`) has a leading `/rs-sr` that the app route tree does not model. Without intervention, expo-router would try to match `/rs-sr/product/123/...` and fail (or land on `+not-found`).

**The fix is `app/+native-intent.ts`** — expo-router's official inbound-URL rewrite hook. It receives the incoming path and returns the path expo-router should actually route to. There we strip (and optionally remember) the locale segment:

```ts
// app/+native-intent.ts  (PROPOSED — not yet created)
const ROUTING_LOCALES = new Set(['rs-sr', 'rs-en', /* …the 10 from the base-site/language DTOs… */]);

export function redirectSystemPath({ path }: { path: string; initial: boolean }) {
  try {
    const url = new URL(path, 'https://oglasino.com');
    const [first, ...rest] = url.pathname.split('/').filter(Boolean);
    if (ROUTING_LOCALES.has(first)) {
      // optionally: stash `first` so the app can set language to match the shared link
      return '/' + rest.join('/') + url.search;
    }
    return path;
  } catch {
    return path;
  }
}
```

Two design notes for whoever implements:
1. **Don't hard-code the 10 locales as a literal.** The locale-parity audit shows the valid set is derivable from the base-site/language DTOs already in `useBootStore`. Prefer building the set from live catalog/base-site data so it can't drift from backend. (At native-intent time the boot store may not be hydrated yet — fallback to a generated constant list, then reconcile.)
2. **Decide whether the shared locale should switch the app's language.** A user tapping a `rs-en` link arguably wants English. That's a product call, not a mechanical one — flag for Igor.

---

## 4. What "add universal/app links" actually requires — step by step

### 4.1 iOS (Universal Links)

1. **`app.config.ts` → `ios.associatedDomains`** (config edit — needs explicit brief; forbidden to me without one):
   ```ts
   ios: {
     // …
     associatedDomains: ['applinks:oglasino.com', 'applinks:www.oglasino.com'],
   }
   ```
2. **Web hosts `https://oglasino.com/.well-known/apple-app-site-association`** (CROSS-REPO — web/backend, not this repo), served as `application/json`, no redirect, listing the App ID(s) and path patterns:
   ```json
   { "applinks": { "details": [
     { "appIDs": ["TEAMID.com.oglasino"],
       "components": [{ "/": "/*/product/*" }, { "/": "/*/user/*" }, { "/": "/*/catalog/*" }] }
   ]}}
   ```
   (Note the `/*/` to allow the locale segment.)
3. Rebuild the dev client (entitlement is native → not OTA).

### 4.2 Android (App Links)

1. **`app.config.ts` → `android.intentFilters`** (config edit — needs brief):
   ```ts
   android: {
     // …
     intentFilters: [{
       action: 'VIEW',
       autoVerify: true,
       data: [{ scheme: 'https', host: 'oglasino.com' }, { scheme: 'https', host: 'www.oglasino.com' }],
       category: ['BROWSABLE', 'DEFAULT'],
     }],
   }
   ```
2. **Web hosts `https://oglasino.com/.well-known/assetlinks.json`** (CROSS-REPO), with the package name + the release signing-cert SHA-256 fingerprint:
   ```json
   [{ "relation": ["delegate_permission/common.handle_all_urls"],
      "target": { "namespace": "android_app", "package_name": "com.oglasino",
                  "sha256_cert_fingerprints": ["<RELEASE SHA-256 — from EAS credentials>"] } }]
   ```
   **Gotcha:** the fingerprint must be the **release/Play-signing** cert, obtainable from EAS credentials. Get it from Igor — I cannot run deploy/credential commands (hard rule).
3. Rebuild.

### 4.3 App-side (this repo — the part I could implement under a brief)

1. **`app/+native-intent.ts`** — strip the locale prefix (§3). This is the keystone; without it, neither platform's verified links resolve to the right screen.
2. **(Optional) language reconciliation** from the stripped locale, if product wants it.
3. **Tests** — `+native-intent` is pure and unit-testable (path in → path out). Cover: locale-prefixed product/user/catalog, no-locale path, unknown-locale path, query strings, malformed input.

### 4.4 Per-environment caveat

Schemes and bundle IDs differ per env (`oglasino` / `oglasino.preview` / `oglasino.development`). Universal/app links are domain-bound, so they realistically attach to **production** (`oglasino.com` + `com.oglasino`) only — unless web also serves association files under preview hosts. Don't expect them to work in the dev client against localhost. Worth stating in the brief so nobody chases "it doesn't open in dev."

---

## 5. Recommended phased rollout

- **Phase 0 — decide the canonical inbound URL shape.** Product + web + mobile agree on which paths are "openable in app" (product, user, catalog are the obvious three) and confirm the locale segment is always present. *(Cross-repo decision; Igor brokers.)*
- **Phase 1 — `app/+native-intent.ts` + tests (this repo).** Pure JS, no native, no config-file edit, fully testable, ships independently and is even useful for the existing custom scheme. **Lowest risk, do this first.** This is the only phase I can do without touching forbidden config.
- **Phase 2 — native config (`app.config.ts` associatedDomains + intentFilters).** Needs an explicit brief (touches `app.config.ts`) and a dev-client rebuild.
- **Phase 3 — web hosts the two `.well-known` files.** Cross-repo (web/backend). Blocking for real verification; can be developed in parallel with Phase 1.
- **Phase 4 — on-device verification** on real iOS + Android (universal links cannot be fully validated in a simulator/emulator for the verification handshake). Igor-owned (Ψ).

Phases 1 and 3 are parallelizable. Phase 2 depends on the release cert fingerprint (Igor/EAS). Phase 4 is last.

---

## 6. Risks & gotchas (so they don't surprise anyone)

1. **Locale segment is mandatory and compound** — the rewrite hook is non-optional and must know the valid locale set (§3). Hard-coding risks drift from backend's routing locales.
2. **Release cert fingerprint** for `assetlinks.json` must come from EAS credentials — I can't fetch it (no deploy/credential access).
3. **iOS AASA caching** — Apple caches the association file on a CDN; changes can take time / a reinstall to propagate during testing. Budget for it in Phase 4.
4. **Verification requires HTTPS, no redirects, correct content-type** on the `.well-known` files — a web-side detail that silently breaks linking if wrong.
5. **Per-env domains** (§4.4) — universal links are effectively production-only unless preview hosts also serve association files.
6. **`exp+oglasino-expo` scheme** in the manifests is Expo Go / dev-client tunneling — leave it; it's not the production scheme.
7. **Auth-guarded targets** already redirect safely (§1.2), but confirm the post-login flow returns the user to the deep-linked screen if you ever link into secured routes — current guards send to `/`, not back to the intended target. Out of scope for product/user/catalog (all public), but note it if messaging links are ever shared.

---

## 7. Whose feature is this? — Cross-repo ownership (the part that is NOT mine)

**Verdict: this is NOT a mobile-only feature.** Two distinct things hide under "deep linking":

- **Custom-scheme links** (`oglasino://...`) — **fully mobile.** Works today, web not involved at all.
- **Universal / App Links** (real shareable `https://oglasino.com/...` links) — **a joint mobile + web feature, and web is a hard blocker.**

Why web is non-negotiable for the https kind: the OS will only hand a verified `https://` link to the app **after** it fetches an association file **hosted on oglasino.com** and confirms the app is authorized. Those files live on the web domain — **the mobile repo has no way to put them there.** If web doesn't host them, mobile can ship every native-config and `+native-intent` change and the links will *still* open the browser. So:

- **Web (BLOCKING) must host**, on `oglasino.com` (and decide on `www`), served as `application/json`, **no redirect**:
  - `/.well-known/apple-app-site-association` (iOS)
  - `/.well-known/assetlinks.json` (Android)
  Without these, no amount of app-side work makes verified links open. This is the gatekeeper.
- **Web (alignment)** should confirm the canonical share URL it already emits (`https://oglasino.com/${locale}/product/...`) is the exact shape the AASA / `assetlinks` path patterns and the `+native-intent` hook expect.
- **Web (consumer, optional)** is the natural place to *use* these links once live — e.g. an "open in app" affordance, and the existing App Store / Play badges (note: those badges are already flagged in issues.md:309 as linking nowhere).
- **Backend/EAS** owns the release signing-cert SHA-256 fingerprint that goes *into* web's `assetlinks.json` — so even web's Android file depends on a value mobile/EAS supplies.

**Practical consequence:** if this becomes a feature, it needs a **web brief (host two `.well-known` files) running in parallel with the mobile work**, with Igor brokering the contract between them. Mobile Phase 1 (`+native-intent`) and web's Phase 3 (`.well-known`) can proceed concurrently; verified links only light up once both land.

---

## 8. Config-file impact (per CLAUDE.md closure gate)

- **`app.config.ts`** would need `ios.associatedDomains` and `android.intentFilters` (Phase 2). **Not edited in this audit** — that's a forbidden config edit without an explicit brief. Drafted above as a proposal only.
- **`state.md` Expo backlog table:** there is currently **no** deep-linking row. If the team wants to track this, a row should be added by Docs/QA (I do not write `state.md`). Draft below under "For Mastermind."
- No `eas.json` / native-config edits proposed beyond the `app.config.ts` plugin-generated entitlements.

---

## 9. For Mastermind

1. **Deep linking is half-built.** Custom scheme + expo-router routing + notification-tap navigation all work. Universal Links / App Links — the shareable-https kind — are absent. This audit is the gap analysis + plan.
2. **Suggested `state.md` Expo-backlog row (Docs/QA to add — I can't write `state.md`):**
   > `| Deep linking (universal/app links) | not-started | App-side `+native-intent` locale-strip is the keystone (Phase 1, in-repo, testable). Native config (associatedDomains/intentFilters) + web-hosted `.well-known` files (cross-repo) + release-cert fingerprint required. See oglasino-expo `.agent/audit-deep-links.md`. |`
3. **Open product decisions for Igor:**
   - Which paths are "openable in app"? (recommend: product, user, catalog.)
   - Should a shared `rs-en` link switch the app's language to match? (§3 note 2.)
   - Production-only, or also preview hosts? (§4.4.)
4. **Cleanest first brief:** "Implement `app/+native-intent.ts` to strip the routing-locale prefix from inbound URLs, with unit tests." It's self-contained, in-repo, no forbidden config, no cross-repo blocker, and is independently useful. Everything else can follow once web hosts the association files and the release fingerprint is in hand.

---

## Conventions check

- **Part 4 (cleanliness):** read-only audit; no code, no imports, no debug logging added. N/A.
- **Hard rules:** no config-file edits (`app.config.ts` / `eas.json` / native config untouched — only *proposed* changes drafted); no other-repo edits; no git ops; no deploys. All native-file claims verified directly (`Info.plist`, `AndroidManifest.xml`) per the "verify before relying on Read" rule, not assumed from `app.config.ts`.
- **Config-file impact:** stated explicitly in §8 — `app.config.ts` change deferred to a future brief; `state.md` backlog row drafted for Docs/QA, not written by me.

**Cleanup performed:** none needed (no code changed).
