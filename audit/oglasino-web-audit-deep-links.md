# Audit — oglasino-web — Deep Linking (Universal/App Links)

**Repo:** oglasino-web
**Branch:** `dev` (unchanged — read-only audit)
**Type:** READ-ONLY. No files edited.
**Date:** 2026-06-04

Every claim below was verified against source on disk. Next.js version is `^16.2.6` (`package.json`).

---

## 1. Canonical share URL shape

**Product — the single builder is `getNormalizedProductUrl`.**

`src/lib/utils/utils.ts:115-122`:

```ts
export function getNormalizedProductUrl(
  productId: number,
  productName: string,
  withPrefix: boolean = false,
  locale?: string
): string {
  return `${withPrefix ? `https://oglasino.com/${locale}` : ''}/product/${productId}/${normalizeProductName(productName)}`;
}
```

So the **emitted shareable product URL is `https://oglasino.com/${locale}/product/${productId}/${slug}`** — matching the brief's expected shape. The hostname `https://oglasino.com` is **hardcoded into this helper** (apex, no `www`). `normalizeProductName` (`utils.ts:128-147`) lowercases, strips diacritics, maps `đ→d`, and replaces non-`[a-z0-9]` runs with single hyphens.

`withPrefix=false` returns a host-less, locale-less path `/product/${id}/${slug}` used only for internal navigation/relative composition (slug rewriter, cross-site redirect, sitemap, hreflang pathBuilder) — never as a copied share URL.

Consumers that emit the **absolute** share URL (`withPrefix=true`, locale passed):
- `src/components/client/ProductFunctions.tsx:64` — share button: `getNormalizedProductUrl(productDetails.id, productDetails.name, true, locale)`, `locale` from `useRoutingLocale()`.
- `src/components/client/UserDetails.tsx:228-233` — "share this product" on a seller's card, same call with `true, locale`.
- The copied string flows into `ShareOptions` (`src/components/popups/components/ShareOptions.tsx`), which `encodeURIComponent`s it into facebook/twitter/whatsapp/telegram/viber intents and a clipboard copy. ShareOptions does not modify the URL.

**User pages — no dedicated builder; canonical is inlined in metadata.**

`src/metadata/generateUserPageMetadata.ts:28`:
```ts
const canonical = `${basicMetadata.baseUrl}/${locale}/user/${user.id}`;
```
Shape: **`${baseUrl}/${locale}/user/${userId}`** — no slug, just the numeric user id. `UserDetails.tsx:197` links internally with `href={'/user/' + userDetails.id}` (relative, locale added by routing). There is no "share this user" affordance today — only product share. `baseUrl` here is the `base.url` METADATA translation key (see §8), **not** the hardcoded apex literal that the product helper uses.

**Catalog pages — canonical inlined in metadata.**

`src/metadata/generateCatalogPageMetadata.ts:37-38`:
```ts
const slugPath = slugs.length > 0 ? `/catalog/${slugs.join('/')}` : '/catalog';
const canonical = `${basicMetadata.baseUrl}/${locale}${slugPath}`;
```
Shape: **`${baseUrl}/${locale}/catalog`** or **`${baseUrl}/${locale}/catalog/${slug}/${slug}/…`**. Again `baseUrl` is the `base.url` translation, not the apex literal.

---

## 2. The routing-locale set

There are **two hardcoded compile-time projections** of the same 10-locale set, plus a **runtime derivation** from base-site DTOs. There is no single source.

1. **next-intl routing** — `src/i18n/routing.ts:5-15`:
   ```ts
   locales: ['rs-sr','rs-en','rs-ru','rsmoto-sr','rsmoto-en','rsmoto-ru','me-sr','me-cnr','me-en','me-ru'],
   defaultLocale: 'rs-sr'
   ```
   This is what `proxy.ts` and `app/[locale]/layout.tsx` validate against (`hasLocale(routing.locales, …)`).

2. **hreflang/OG map** — `src/metadata/localeMapping.ts:14-25` defines `LOCALE_TO_BCP47` over the **same 10 keys**; `ALL_ROUTING_LOCALES = Object.keys(LOCALE_TO_BCP47)` (`localeMapping.ts:35`). This map also encodes the BCP-47 escapes the `+native-intent` rewrite will care about: `me-cnr → sr-Latn-ME`, EN/RU per base site, etc.

Both are **stable, hand-maintained constants** — not generated, not imported from a DTO. They must be kept in lockstep by hand (a divergence flag, see For Mastermind).

3. **Runtime derivation exists too.** `app/sitemap.ts:42-52` (`getAllLocales`) builds the same `${bs.code}-${lang.code}` segments from `BaseSiteOverviewDTO.allowedLanguages` (backend `/public/baseSite/…` data), and `proxy.ts:12-24` builds `ALLOWED_LANGS_PER_TENANT` by splitting `routing.locales` on `-`. So the set **is derivable at runtime** from base-site/language DTOs, and the compile-time constant is explicitly documented as "the compile-time projection of the same data" (`proxy.ts:12-15`).

**For mobile's `+native-intent`:** the canonical, dependency-free source is the 10-entry constant in `routing.ts` (mirrored in `localeMapping.ts`). It is a stable constant today; the backend base-site DTO is the underlying source of truth it projects.

---

## 3. Locale segment requiredness

**The locale segment is structurally mandatory and an invalid one 404s.**

- Every public page lives under `app/[locale]/…`. The locale layout hard-gates: `app/[locale]/layout.tsx:25-27`:
  ```ts
  if (!hasLocale(routing.locales, locale)) {
    notFound();
  }
  ```
  A request like `/product/123/foo` resolves `[locale]="product"`, fails `hasLocale`, and 404s. An unknown-but-shaped locale (e.g. `/rs-cnr/…`) is redirected to a valid one by `proxy.ts:59-65` before it reaches the page.
- `i18n/request.ts:10-12` falls back to `defaultLocale` only for the formatter config; it does **not** rescue an invalid path — the layout's `notFound()` still fires.

**No path emits a locale-less product/user/catalog *share/canonical* URL.** The only locale-less outputs are the `withPrefix=false` internal paths, and each is re-prefixed at the point of navigation:
- slug rewriter: `router.replace(`/${locale}${canonicalPath}`)` (`ProductCanonicalSlugRewriter.tsx:30`);
- cross-base-site redirect: `redirect(`/${targetBaseSite.code}-${finalLocale}${…false}`)` (`product/[productId]/[productName]/page.tsx:79-81`);
- sitemap: `${SITE_URL}/${bs.code}-${lang.code}${…false}` (`sitemap.ts:179`).

No contract hole found here. (One latent footgun, not a live hole: `getNormalizedProductUrl(id,name,true,undefined)` would render the literal `https://oglasino.com/undefined/product/…`; today every `withPrefix=true` call site passes a resolved `locale`.)

---

## 4. Existing "open in app" scaffolding

`src/lib/config/appDeepLink.ts` is the dormant scaffold. Full content:

- `export const APP_DEEP_LINK_ENABLED: boolean = false;` (line 15) — the single flip point, typed `boolean` (not literal `false`) so it reads as a toggle.
- `const APP_DEEP_LINK_SCHEME = 'oglasino';` (line 19) — **placeholder custom scheme**, explicitly marked as such.
- `openInApp(path='')` (lines 24-27) hands off via **`window.location.href = `${APP_DEEP_LINK_SCHEME}://${path…}``** — i.e. it targets the **custom scheme `oglasino://`, NOT an `https://` Universal/App Link.** No-op while the flag is false.

The header comment (lines 1-10) is explicit that this is for the `/reset` success step, that the real per-tier schemes are `oglasino://` (prod) / `oglasino-preview://` / `oglasino-dev://`, and that "the web-env → scheme mapping is a cross-repo contract Igor brokers next session" — it carries a `TODO(deep-link, next session)` pointing at `oglasino-expo .agent/audit-deep-links.md`.

**Only consumer:** `app/[locale]/(portal)/(public)/reset/ResetPasswordClient.tsx`:
- imports both symbols (`:10`);
- renders the button only behind the flag (`:154-158`):
  ```tsx
  {APP_DEEP_LINK_ENABLED && (
    <Button variant="outline" onClick={() => openInApp()}>
      {tButtons('reset.page.success.app.label')}
    </Button>
  )}
  ```
  Called with no path → `oglasino://` (bare scheme). Because the flag is `false`, the button never renders and the hand-off never fires today.

**Reconciliation note:** this scaffold is **custom-scheme** based. Universal/App Links are `https://` links the OS claims via the association files — a different mechanism. The feature must decide whether `appDeepLink.ts` becomes the single home for *both* (a custom-scheme fallback plus an `https://` deep link) or whether the `https://` path supersedes it. It should be extended, not duplicated (see For Mastermind).

---

## 5. `.well-known` hosting today

**Web serves nothing under `/.well-known/`.** Verified:
- No `public/.well-known/` directory; no `apple-app-site-association` or `assetlinks.json` anywhere in the repo (`find` over the tree, node_modules excluded — zero hits).
- `public/` root holds only images, favicon, two `firebase-messaging-sw*.js` files, `logo/`, `design/`. The served `firebase-messaging-sw.js` proves **root static serving from `public/` works** for normal (non-dot) files.
- Only two route handlers exist, both under `app/api/`: `app/api/revalidate/route.ts`, `app/api/auth/token/route.ts`. No `.well-known` route handler.
- `next.config.ts` has no `rewrites`/`redirects`/`headers` (only `serverActions.allowedOrigins`). `vercel.json` has no routing — just framework/region/`deploymentEnabled:false`.

**Could Next serve a correct `apple-app-site-association`?** Two viable mechanisms, each with a gotcha:

- **Static file** at `public/.well-known/apple-app-site-association`: (a) the file has **no extension**, so the served `Content-Type` is not guaranteed to be `application/json` (modern iOS tolerates this; historically a concern); (b) Next.js has a long-standing reluctance to serve **dot-prefixed** paths from `public/` — needs verification on 16.2.6 rather than assumed. `assetlinks.json` (it *has* a `.json` extension) is the easier of the two as a static file.
- **Route handler** at `app/.well-known/apple-app-site-association/route.ts` returning the JSON with an explicit `Content-Type: application/json`. This sidesteps both the extension/content-type and the dotfile-serving questions and is the robust choice. (It would *not* be caught by the locale layout because it lives outside `[locale]`; and `proxy.ts` skips it — see §7.)

---

## 6. App-store badges

**Confirmed dead — icons only, no link, no handler.** They live in the footer.

`src/components/server/layout/Footer.tsx:80-86`:
```tsx
<div className="flex gap-2">
  <div className="cursor-pointer xl:hover:scale-105">
    <GooglePlayGetIt />
  </div>
  <div className="cursor-pointer xl:hover:scale-105">
    <AppleStoreGetIt />
  </div>
</div>
```
`GooglePlayGetIt` (`src/components/icons/GooglePlayGetIt.tsx`) and `AppleStoreGetIt` (`src/components/icons/AppleStoreGetIt.tsx`) are **pure inline `<svg>`** badge graphics — no `<a>`, no `href`, no `onClick` anywhere in either file or at the call site. The `cursor-pointer` class is the only thing suggesting interactivity. This matches `issues.md` 2026-05-31 ("app-store / play-store badges are dead links (icons only)").

**Where real URLs wire in:** `Footer.tsx:80-86` — wrap each badge in an `<a href={storeListingUrl} …>` (or `next/link`) once the App Store / Play Store listing URLs exist. Note: store-listing URLs (install) are conceptually distinct from a Universal/App Link (open installed app to content); the badges are the natural surface for the former and a reasonable host for an "open in app" affordance.

---

## 7. The `proxy.ts` / middleware layer

**`proxy.ts` would NOT intercept a `/.well-known/*` request** — its matcher excludes it. `proxy.ts:77-79`:
```ts
export const config = {
  matcher: '/((?!api|trpc|_next|_vercel|.*\\..*).*)',
};
```
The negative lookahead `.*\..*` excludes any path containing a dot. `/.well-known/apple-app-site-association` contains a dot (in the `.well-known` segment), so the middleware **never runs** on it — no locale parse, no redirect. `assetlinks.json` (dot + extension) is likewise excluded.

For completeness on what the middleware *does* (so it's clear nothing in it touches the path): for matched paths it parses a leading `<tenant>-<lang>` segment (`LOCALE_PREFIX`, `proxy.ts:10`), redirects invalid tenant-lang combos to a cookie/first-allowed lang (`:59-65`), and applies cookie-wins-over-URL locale redirects (`:67-71`). `/.well-known/…` matches none of this even if the matcher let it through (no `<x>-<y>` first segment), but the matcher excludes it first regardless. **No locale redirect risk on the verification path.**

---

## 8. www vs apex

**Everything web emits as a canonical/share/sitemap URL uses the apex `https://oglasino.com` — never `www`.**
- Product share helper hardcodes `https://oglasino.com` (`utils.ts:121`).
- `app/robots.ts:3` `const SITE_URL = 'https://oglasino.com';` (also sets `host: SITE_URL`).
- `app/sitemap.ts:9` `const SITE_URL = 'https://oglasino.com';`.

`www.oglasino.com` appears only in **non-canonical** contexts: `next.config.ts:18-26` lists both `oglasino.com` and `www.oglasino.com` (plus the `*.vercel.app` and stage hosts) in `serverActions.allowedOrigins` for the CSRF check behind the Cloudflare worker — this does not affect which host serves files. Web does no www↔apex redirect itself (no `redirects` in `next.config.ts`, no `vercel.json` routing); host canonicalization is the worker's job.

**Caveat — two apex sources, not one literal.** Product/robots/sitemap hardcode the apex literal, but **user and catalog canonicals derive the host from the `base.url` METADATA translation** (`generalMetadata.ts:5` → consumed at `generateUserPageMetadata.ts:28`, `generateCatalogPageMetadata.ts:38`). `base.url`'s value is seeded by backend and not visible in this repo. If `base.url` ever resolves to anything other than `https://oglasino.com` (e.g. with `www`, or a per-base-site host), user/catalog canonicals would diverge from product/sitemap. Flagged below. There is also an **unused `NEXT_PUBLIC_SITE_URL`** env var (`=http://localhost:3000` in `.env.local`/`.env.local.example`) with **zero references** in `src`/`app` — a red herring for anyone hunting the canonical host.

---

## Contract surface (for `+native-intent` and the association files)

**Canonical URL shapes web emits (all apex, all locale-prefixed):**

| Surface | Shape | Builder | Host source |
|---|---|---|---|
| Product | `https://oglasino.com/{locale}/product/{id}/{slug}` | `getNormalizedProductUrl(id,name,true,locale)` — `utils.ts:121` | hardcoded apex literal |
| User | `https://oglasino.com/{locale}/user/{userId}` (no slug) | inline — `generateUserPageMetadata.ts:28` | `base.url` translation |
| Catalog | `https://oglasino.com/{locale}/catalog[/{slug}…]` | inline — `generateCatalogPageMetadata.ts:38` | `base.url` translation |

- `{slug}` (product only) = `normalizeProductName()` output: `[a-z0-9-]`, diacritics stripped, single hyphens. The slug is **cosmetic** — the `{id}` is authoritative; the client rewrites a wrong slug silently (`ProductCanonicalSlugRewriter.tsx`), so `+native-intent` path patterns can treat the slug segment as a wildcard / ignore it.
- `{locale}` is one of exactly **10** values: `rs-sr rs-en rs-ru rsmoto-sr rsmoto-en rsmoto-ru me-sr me-cnr me-en me-ru` (`routing.ts:5-15`). **The locale segment is guaranteed present** on every emitted product/user/catalog URL and is enforced by a `notFound()` on anything invalid (`app/[locale]/layout.tsx:25-27`).

**Locale set + sourcing:** a stable, hand-maintained constant duplicated in `src/i18n/routing.ts` (`routing.locales`) and `src/metadata/localeMapping.ts` (`LOCALE_TO_BCP47` keys → `ALL_ROUTING_LOCALES`); also runtime-derivable from base-site `allowedLanguages` DTOs (`sitemap.ts:42-52`). For an offline `+native-intent` rewrite, treat the 10-entry list as a constant; the AASA/`assetlinks` path patterns must match `^/(rs-sr|rs-en|rs-ru|rsmoto-sr|rsmoto-en|rsmoto-ru|me-sr|me-cnr|me-en|me-ru)/(product|user|catalog)(/.*)?$`.

**`.well-known` today:** nothing served; `proxy.ts` excludes `.well-known` paths (dot in segment) so a file/handler there is safe from locale redirects; no www/apex divergence in what web emits (apex everywhere).

---

## For Mastermind

**Recommendation — where to serve `.well-known`:**
The Cloudflare router worker fronts **every** request (per `conventions.md` Part 8: "the worker is the edge boundary"; `next.config.ts:13-16` confirms `oglasino.com` is proxied to `oglasino-web.vercel.app`). That makes the **worker** the most reliable host for `apple-app-site-association` and `assetlinks.json`: it can return the exact bytes with an explicit `application/json` content-type, sidesteps Next's extensionless-file and dotfile-serving uncertainties entirely, and guarantees the file is served on **whatever host the OS verifies** (apex and www alike) without depending on Vercel's static behaviour. If instead web is chosen, prefer a **Route Handler** (`app/.well-known/apple-app-site-association/route.ts`) over a static `public/` file, for the content-type/dotfile reasons in §5. My recommendation: **router worker serves both association files.** This is a cross-repo (`oglasino-router`) decision — flagged, not actioned.

**Reconciling with `appDeepLink.ts`:** the existing scaffold is **custom-scheme** (`oglasino://`, `:19/:26`) and disabled (`:15`). Universal/App Links are `https://` and OS-claimed — orthogonal mechanism. Recommend the feature **extend `appDeepLink.ts` as the one home** (add the `https://` canonical-link construction and/or the env→scheme map already foreshadowed in its header TODO), rather than introduce a parallel deep-link util. The `/reset` consumer and the future footer-badge consumer should both read from it.

**Adjacent observations (Part 4b):**
1. **Host-source divergence** — `src/lib/utils/utils.ts:121` (product) + `app/robots.ts:3` + `app/sitemap.ts:9` hardcode `https://oglasino.com`, while user/catalog canonicals (`generateUserPageMetadata.ts:28`, `generateCatalogPageMetadata.ts:38`) derive the host from the `base.url` translation. **Severity: medium** — if `base.url` ≠ `https://oglasino.com` the canonical host silently splits across page types, which matters for both SEO and which host an `https://` deep link should use. I did not change this (out of scope, read-only). Recommend confirming the seeded `base.url` value and, longer term, a single host constant.
2. **Locale-set duplication** — the 10-locale list is hand-maintained in both `routing.ts:5-15` and `localeMapping.ts:14-25` with no shared source or test asserting parity. **Severity: medium** — a future locale added to one and not the other would desync routing from hreflang (and from any `+native-intent` pattern copied off one of them). Out of scope; flagging.
3. **Unused env var** — `NEXT_PUBLIC_SITE_URL` (`.env.local:33`, `.env.local.example:54`) has zero references in `src`/`app`. **Severity: low** — misleading to anyone searching for the canonical host. Out of scope.
4. **`getNormalizedProductUrl` undefined-locale footgun** — `utils.ts:115-121`: `withPrefix=true` with `locale=undefined` yields `https://oglasino.com/undefined/product/…`. **Severity: low** — no live call site does this today (all pass a resolved `locale`); noting because the deep-link work will add new call sites. Out of scope.

**Part 4a simplicity evidence:**
- Added (earned complexity): nothing — read-only audit, no code added.
- Considered and rejected: nothing.
- Simplified or removed: nothing.

**Config-file impact:** none required. This audit surfaces no edit to `conventions.md`, `decisions.md`, `state.md`, or `issues.md`. (Observations 1–4 above are candidates Mastermind may route into `issues.md`, but I am not drafting those entries — they are adjacent flags, not a config dependency of this session.)

**Conventions check:** Part 4 (cleanliness) — N/A, no files edited; confirmed zero edits to any source/config file. Part 4b — observations flagged above. Part 11 (trust boundaries) — N/A (no request DTOs touched; share URLs carry only the public `{id}` which the server already validates).
