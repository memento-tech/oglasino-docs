# Audit — SEO foundation (Phase 2, read-only)

**Repo:** `oglasino-web`
**Branch:** `dev` (per Igor's checkout — not switched)
**Spec read:** `oglasino-docs/future/seo-foundation.md` (note: brief said `features/`; the file lives in `future/`. Flagged to Mastermind in the session summary.)
**Scope:** read-only inventory of every surface the spec touches against current code.

---

## Part 1 — Metadata generators inventory

The spec assumes nine `generate*` files. There are **13** files in `src/metadata/`. One is a helper, one is for private routes, eleven emit public-page metadata.

| File | Consumer | JSON-LD today | `alternates.canonical` | `alternates.languages` (hreflang) | `openGraph.locale` / `alternateLocale` | Notes |
|---|---|---|---|---|---|---|
| `generalMetadata.ts` | (helper, no consumer) | n/a | n/a | n/a | n/a | Exports `getBasicMetada` (sic). Sets `baseUrl` from `t('base.url')` which seeds to `https://oglasino.com` in all four locales — i.e. apex, not `www`. |
| `generateMainLayoutMetadata.ts` | `app/layout.tsx:33` | none | none | none | none | `metadataBase` commented out at line 10. |
| `generateIntroPageMetadata.ts` | `app/page.tsx:14` | none (no Organization, no WebSite) | **none** | none | none | Title/description are hardcoded literals (`'Oglasino'`, `'Vaša platforma za kupovinu i prodaju'`), not translation keys. |
| `generateHomePageMetadata.ts` | `app/[locale]/(portal)/(public)/page.tsx:21` | `WebSite + SearchAction` and `ItemList` (10 newest products) via `other['application/ld+json']` | yes (`${baseUrl}/${locale}`) | none | none | `SearchAction.urlTemplate` uses `${siteUrl}/catalog?searchText=…` — no locale in path. |
| `generateCatalogPageMetadata.ts` | `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:71` | `ItemList` only (no `CollectionPage`) via `other['application/ld+json']` | yes (apex + locale + slugs) | none | none | Top-of-file builds canonical from `${basicMetadata.baseUrl}/${locale}/catalog/${slugs.join('/')}` (no `searchParams` in canonical → good). |
| `generateProductPageMetadata.ts` | `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:41` | `Product + Offer + PropertyValue[]` via `other['application/ld+json']` | yes — built via `getNormalizedProductUrl(id, name, true, locale)` which **hardcodes `www.oglasino.com`** (utils.ts:122) | none | none | `metadataBase` set to `${baseUrl}/${locale}` (line 79) — unusual, normally just the origin. `Offer.availability` mapping uses `product.availability?.id` (the filter row id, not the option id) → see Part 7 finding. |
| `generateUserPageMetadata.ts` | `app/[locale]/(portal)/(public)/user/[userId]/page.tsx:36` | `Person` via `other['application/ld+json']` | yes — `${baseUrl}/${locale}${t('page.user.url')}` — **no userId in path**; every user page emits the same canonical | none | none | `Person` lacks `image`. `og:type` = `'profile'` is set. |
| `generateAboutMetadata.ts` | `app/[locale]/(portal)/(public)/about/page.tsx:17` | `AboutPage + Organization` via `other['application/ld+json']` | yes | none | none | `Organization.sameAs` set to the `socialLinks` **record/object** (not array of URLs) — Schema.org expects an array. |
| `generatePricingPageMetadata.ts` | `app/[locale]/(portal)/(public)/pricing/page.tsx:17` | `WebPage + Offer + Organization` via `other['application/ld+json']` | yes | none | none | Hardcoded `priceCurrency: 'RSD'` with `// or RSD if you prefer` comment at line 71. |
| `generatePrivacyPageMetadata.ts` | `app/[locale]/(portal)/(public)/privacy/page.tsx:12,25` | `PrivacyPolicy` via `other['application/ld+json']` | yes | none | none | Also re-invoked inside the page body at line 25 in a broken inline `<script>` that stringifies the whole **Metadata object** (not the Schema.org payload) — see Part 3. |
| `generateTermsPageMetadata.ts` | `app/[locale]/(portal)/(public)/terms/page.tsx:12` | `TermsOfService` via `other['application/ld+json']` | yes | none | none | `inLanguage` set to the routing locale string (e.g. `rs-sr`) which isn't BCP-47. |
| `generateFreeZonePageMetadata.ts` | `app/[locale]/(portal)/(public)/blog/free-zone/page.tsx:18` | `Article` via `other['application/ld+json']` | yes | none | none | `inLanguage: locale` (line 66) — same BCP-47 issue. |
| `generatePrivatePageMetadata.ts` | `app/[locale]/owner/layout.tsx:19`, `app/[locale]/admin/layout.tsx:19`, three `(protected)/*/layout.tsx` files | none | none | none | none | Sets `robots: { index:false, follow:false, googleBot:{index:false, follow:false} }`. Spec assumption that admin/owner already carry `noindex` is **correct**. |

**Severity rollup for Part 1:**
- **HIGH** — every public page emits its JSON-LD via `other['application/ld+json']` (rendered as a `<meta>` tag, invisible to crawlers). Confirms spec §4.1.
- **HIGH** — zero public pages emit `alternates.languages`. Confirms spec §6.1.
- **HIGH** — zero public pages emit `openGraph.locale`/`alternateLocale`. Confirms spec §9.7.
- **MEDIUM** — `getNormalizedProductUrl` host shape is `www.oglasino.com` while every other emitter is on apex. Confirms spec §9.1.

---

## Part 2 — Public page files inventory

Public surfaces under `app/[locale]/(portal)/(public)/` plus the apex `app/page.tsx`. Owner/admin/protected routes are out of SEO scope (gated `noindex`).

| Route | File | Inline `<script type="application/ld+json">` today | Server-rendered breadcrumb? | `<h1>` count + content | Not-found behaviour |
|---|---|---|---|---|---|
| `/` (apex intro) | `app/page.tsx` | none | n/a | 1 — wraps `t('welcome.title')` text | n/a (always serves) |
| `/{locale}` (home) | `app/[locale]/(portal)/(public)/page.tsx` | none | no | **0** | n/a |
| `/{locale}/catalog/[[...slugs]]` | `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx` | none | **no** — `<ProductBreadcrumbs productData={undefined} />` at line 142 is a client component | **0** | empty slugs → `redirect('/{locale}')`; unresolved slugs → `notFound()` (line 102 — **Seam 2 partially fixed already**) |
| `/{locale}/product/{productId}/{productName}` | `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` | none | **no** — same client `<ProductBreadcrumbs productData={productDetails} />` at line 109 | **2** — `ProductDetails.tsx:68` (product name) and `ProductDetails.tsx:80` (price); price `<h1>` is the semantic abuse the spec calls out | not found → inline `<div>` with `tCommon('product.page.not.found.title')`, HTTP **200** (lines 77-82). Cross-base-site → `redirect()` (lines 61-75). |
| `/{locale}/user/{userId}` | `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` | none | n/a | **0** | not found → `<NotFound title=… description=… />` (lines 64-70). `NotFound` is a `'use client'` component rendering a friendly UI with HTTP **200** (no `notFound()` call). Cross-base-site → `redirect()`. |
| `/{locale}/about` | `app/[locale]/(portal)/(public)/about/page.tsx` | none | n/a | **1** — `<h1>` at line 27 contains `<OglasinoIcon ... extended />` only (no text); the visible heading text `{tAbout('heading')}` lives in an adjacent `<p>` at line 30 | n/a |
| `/{locale}/pricing` | `app/[locale]/(portal)/(public)/pricing/page.tsx` | none | n/a | **1** — `<h1>` at line 32 renders `hero.subtitle.line1` + `<br>` + `hero.subtitle.line2` (line 33-35). **Duplicate `line1` is fixed** — Igor's 2026-05-20 very-easy batch closed it. | n/a |
| `/{locale}/blog/free-zone` | `app/[locale]/(portal)/(public)/blog/free-zone/page.tsx` | none | n/a | **1** — `<h1>` at line 28 renders `tFreeZone('header')` text | n/a |
| `/{locale}/privacy` | `app/[locale]/(portal)/(public)/privacy/page.tsx` | **yes — at lines 22-27** — broken: stringifies `generatePrivacyPageMetadata(t, locale)` (the whole Metadata object) instead of the Schema.org payload | n/a | **0** (h1 lives in fetched markdown — see Part 7) | always serves (fetches GitHub-hosted markdown) |
| `/{locale}/terms` | `app/[locale]/(portal)/(public)/terms/page.tsx` | none | n/a | **0** (h1 lives in fetched markdown) | always serves |

**Severity rollup for Part 2:**
- **HIGH** — Home, Catalog, User, Privacy, Terms all render zero `<h1>`. Confirms spec §7.
- **HIGH** — Product detail page emits 2 `<h1>`s including the price abuse. Confirms spec §7.
- **HIGH** — About `<h1>` wraps only the SVG. Confirms spec §7.
- **HIGH** — Catalog and Product use the client `ProductBreadcrumbs` (no initial HTML). Confirms Seam 4.
- **MEDIUM** — Product not-found returns HTTP 200, not `notFound()`. Confirms Seam 8.
- **MEDIUM** — User not-found returns HTTP 200 via the `NotFound` component (not `notFound()`). Confirms spec §9.3 extension.
- **MEDIUM** — Privacy page emits the wrong payload via its broken inline `<script>`. Confirms spec §9.13.
- **LOW** — Spec says the pricing duplicate is open; in reality it was already fixed. Spec text needs amending.
- **LOW** — Spec frames the catalog bad-slug fix as `redirect()` → `notFound()`; in reality unresolved slugs already call `notFound()` (line 102). Only the empty-slug branch (line 95-97) still `redirect()`s to `/{locale}`, which is correct behaviour for that case. Spec amendment needed for Seam 2.

---

## Part 3 — Specific files the spec touches by line number

### `src/lib/utils/utils.ts` — `getNormalizedProductUrl` (line 116-123)

Spec says: hardcodes `https://www.oglasino.com` at line 122.
Code says: confirmed — line 122 reads ``${withPrefix ? `https://www.oglasino.com/${locale}` : ''}/product/...``.
**Severity: medium.** Confirms Seam 1. Note: `// TODO FIX URL (add locale)` comment at line 115 sits above. `locale` is now threaded but only on the `withPrefix=true` branch; the no-prefix branch (used by the sitemap and the canonical-slug-rewriter) does NOT prefix `/${locale}` — sitemap entries get the locale from their outer URL builder (`${SITE_URL}/${bs.code}-${lang.code}` + url), so that's fine. Apex host is the only remaining issue.

### `app/sitemap.ts` — host, `getProductCountForBaseSite`, locale shards

Spec says: uses apex host. `getProductCountForBaseSite` exists. Locale-aware shards.
Code says:
- `SITE_URL = 'https://oglasino.com'` at line 9 — apex ✓.
- `getProductCountForBaseSite` is at lines 80-84; it fetches `(baseSite, page=0, perPage=1)` and reads `totalNumberOfProducts` — this is the "over-fetch" pattern flagged in `issues.md` 2026-05-16 as a known low-severity issue (it's still here).
- Sitemap does NOT shard per locale — `generateSitemaps()` emits `{ id: 0 }` (static) plus N product shards. Inside each product shard, the inner loop emits one URL **per locale** for every product (lines 168-174 — `for (const lang of langs) entries.push({ url: ... })`). So the product sitemap is base-site-aware + locale-aware, but the shard boundaries are not — within one shard you may see all language variants of one product alongside another product's, which can cause uneven shard sizes (PRODUCTS_PER_SHARD = 5000 counts products not URLs).
- Static URLs included at lines 22-28: `''`, `/about`, `/pricing`, `/privacy`, `/terms`, `/blog/free-zone`. The site-wide root `https://oglasino.com` is also emitted at line 107.

**Severity: low.** Sitemap is functional. Catalog category URLs are absent — explicitly deferred to post-launch §11.2 in spec. The locale × product fan-out is a known shard-size variance but not a correctness issue.

### `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx` — bad-slug branch

Spec says: `redirect('/{locale}')` for unresolved slugs at lines 92-94.
Code says:
- Line 95-97: `if (slugs.length === 0) redirect(\`/${baseSite.code}-${oglasinoLocale}\`);` (empty-slug branch, **redirect kept**)
- Line 101-103: `if (!categories) notFound();` (**bad-slug branch already calls notFound()**)

**Severity: medium for spec accuracy / low for engineering.** Seam 2 is half-resolved already. The spec needs amending — only the empty-slug `redirect` remains, and that's the correct behaviour (an empty `/catalog` URL is genuinely a redirect to home, not a 404). The two `issues.md` 2026-05-14 catalog-slug entries should already be flippable to fixed pending Igor's verification; they were not in the bug-chat closure batch.

### `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` — not-found branch

Spec says: lines 74-79 render the inline "not found" message with HTTP 200.
Code says: confirmed — lines 77-82 render `<div className="..."><p>{tCommon('product.page.not.found.title')}</p></div>` when `!productDetails || !productDetails.id`.
**Severity: medium.** Confirms Seam 8.

### `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` — not-found branch

Spec says: line referenced as the not-found branch.
Code says: lines 64-70 render the `<NotFound title=… description=… />` component on `!userDetails`. `NotFound` (`src/components/server/NotFound.tsx`) is declared `'use client'`, renders an alert UI, and does NOT call `notFound()` — so the response is HTTP 200.
**Severity: medium.** Confirms spec §9.3 extension. Note: `NotFound.tsx` is named "server" by folder but `'use client'` at line 1 — name/path mismatch is its own low-severity flag for Part 4b but out of audit scope.

### `src/components/client/product/ProductBreadcrumbs.tsx` — client/Zustand-gated

Spec says: client component, Zustand-gated, initial HTML emits nothing.
Code says: confirmed — `'use client'` at line 1, `useBaseSiteStore()` at line 17, returns `null` until `baseSite` is hydrated (line 55: `if (!baseSite) return null;`). Initial HTML for catalog and product pages contains no breadcrumb anchors.
**Severity: high.** Confirms Seam 4.

### `src/components/server/OglasinoBreadcrumbs.tsx` — "Kategorije iz" literal at line 81

Spec says: bare Serbian `Kategorije iz` literal at line 81 (or current equivalent).
Code says: **already fixed.** The bare literal is gone. Line 83-85 reads `tNavigation('breadcrumb.categories_from.label', { category: t(...) })`. Igor's 2026-05-20 very-easy bug-batch session shipped the `NAVIGATION` key + four-locale seed.
**Severity: low.** Spec text is stale on this point. The `issues.md` 2026-05-17 entry "Kategorije iz" tied to `OglasinoBreadcrumbs.tsx:81` is **already closed** by that batch.

Note also: this component is imported from `src/components/server/` but uses client hooks (`useTranslations`, `useDialogStore`, `useScreenBreakpoint`). It's effectively a client component — the file path is misleading. Out of scope but worth flagging.

### `PortalProductCard.tsx`, `UniversalProductCard.tsx` — `<div onClick={router.push(...)}>` pattern

Spec says: `<div onClick={router.push(...)}>` for product cards.
Code says: confirmed.
- `PortalProductCard.tsx:28-31` passes `onClick={() => { setIsLoading(true); router.push(getNormalizedProductUrl(productOverview.id, productOverview.name)); }}` into `UniversalProductCard`.
- `UniversalProductCard.tsx:47` consumes `onClick` on a `<div>`. No `<a href>` wraps the card.
- `getNormalizedProductUrl` here is called with `withPrefix=false` (no host, no locale prefix) — `useRouter` from `@/src/i18n/navigation` handles the locale prefix. So crawlers cannot follow the card link at all (no `href`, plus the JS handler does locale-aware navigation).
- Five other consumer files use `UniversalProductCard` (`PreviewProductCard`, `AdminProductCard`, `DashboardProductCard`, and three list wrappers). Only `PortalProductCard` (portal home + catalog + favorites + carousel) is on the SEO surface — the dashboard/admin/preview variants are noindex contexts.

**Severity: high.** Confirms Seam 5 / spec §5.

### `UserDetails.tsx` — "See user products" button + displayName heading

Spec says: line 173-178 has `<Button onClick={router.push(...)}>` for "See user products"; line 118 has displayName in a `<div>`.
Code says:
- Lines 177-186: `<Button variant="outline" onClick={(event) => { event.stopPropagation(); event.preventDefault(); router.push('/user/' + userDetails.id); }}>` — confirmed, button-not-anchor pattern.
- Lines 114-123: displayName lives inside a `<div className="flex items-center gap-2 font-medium ...">` at line 114, with the actual text `{userDetails.displayName}` at line 119. No `<h1>` wrapper.

**Severity: medium.** Confirms spec §5 + §7 user-profile h1 fix.

### `ProductImageCarusel.tsx` — `alt="Missing image"` literal

Spec says: line 65 hardcodes `alt="Missing image"`.
Code says: confirmed — line 65 reads `alt="Missing image"`. Also line 94 emits empty `alt=""` for the thumbnail strip (likely intentional — decorative).
**Severity: low.** Confirms spec §9.8.

### `src/components/server/ProductDetails.tsx`, `src/components/client/NumberOfViews.tsx` — Serbian tooltips

Spec says: hardcoded Serbian tooltips on the product detail page.
Code says: **already fixed.**
- `ProductDetails.tsx:71` uses `tCommon('product.tooltip.favorites_count.label')`.
- `NumberOfViews.tsx:25` uses `tCommon('product.tooltip.views_count.label')`.
- Igor's 2026-05-20 very-easy batch closed this `issues.md` entry; the new keys are seeded across four locales (state.md log entry).

**Severity: low (spec stale).**

### `src/components/server/MarkdownViewer.tsx` — `markdown.fild.load` typo

Spec says: line 18/22 typo `markdown.fild.load`.
Code says: **already fixed.** Lines 18 and 22 read `tErrors('markdown.field.load')` (correct spelling). Igor renamed this in the 2026-05-20 very-easy batch per state.md.
**Severity: low (spec stale).**

### `pricing/page.tsx` — `hero.subtitle.line1` duplicate

Spec says: line 31 duplicates `hero.subtitle.line1`.
Code says: **already fixed.** Lines 33-35 use `tPricing('hero.subtitle.line1')` + `<br>` + `tPricing('hero.subtitle.line2')`. The duplicate was closed in Igor's 2026-05-20 very-easy batch.
**Severity: low (spec stale).**

Caveat: spec §7.2 also wants this `<h1>` to carry the page **title** (`hero.title`), not the subtitle, with subtitle moved to a `<p>`. That work has not been done — line 32 still uses subtitle inside the `<h1>`.

### `about/page.tsx` — `<h1>` wrapping SVG; hardcoded English alt

Spec says: line 26 `<h1>` wraps the SVG with no text; line 36 hardcodes English alt.
Code says:
- Line 27 (off by one from spec): `<h1 className="mt-6 flex animate-bounce ..."><OglasinoIcon size={300} extended /></h1>` — no text inside the `<h1>`, only the icon.
- Line 37: `<Image src="/oglasino-hero2.jpg" alt={'Oglasino portal'} width={400} height={400} ... />` — hardcoded English alt.
- Lines 58, 67: mission images use `alt={tAbout(`mission.item.${item}`)}` (translated).

**Severity: medium.** Confirms spec §7 and §9.9.

### `app/page.tsx` — intro page; hardcoded English alts

Spec says: lines 25, 33 — hardcoded English alts on intro images.
Code says:
- Line 25: `<img src={INTRO} className="..." alt="Oglasino intro" />` — hardcoded English alt.
- Line 34: another `<img src={INTRO} ... alt="Oglasino intro" ... />` — same.
- Line 66: base-site flag images use `alt=""` (decorative, intentional).

**Severity: low.** Confirms spec §9.9 (intro variant).

### `app/robots.ts` — disallow patterns

Spec says: disallow rules target `/admin`, `/owner`, `/messages`, etc. without locale prefix — none match real URLs.
Code says: confirmed — lines 11-23 list `/admin`, `/admin/`, `/owner`, `/owner/`, `/messages`, `/notifications`, `/favorites`, `/test`, `/wants`, `/icons`, `/api/`. Real URLs are all `/{locale}/...` so none of the locale-scoped rules match.
- `/api/` stays apex-rooted and would match (api routes are not locale-prefixed) — that rule works.
- `/design` is **missing** from the disallow list (spec §9.5).

**Severity: high.** Confirms spec §9.5 + missing `/design` entry.

### `app/layout.tsx` — `<html lang>`

Spec says: line 36/42 sets `lang={htmlLang.split('-')[0]}` which yields `cnr` on `me-cnr` routes.
Code says: confirmed at line 59 (numbering shifted due to consent + GA4 additions): `lang={htmlLang.split('-')[0]}`. `htmlLang` is built at line 40 from `getTenantLocale(locale).locale`, which `mapToIntlLocale` produces as `${languageMap[lang]}-${regionMap[tenant]}` — e.g. `me-cnr` → `cnr-ME`. `.split('-')[0]` yields `cnr`. The spec's analysis stands.
**Severity: medium.** Confirms spec §6.6.

### `app/[locale]/layout.tsx`

Spec says: anything related to locale handling, scripts, providers.
Code says: provides `NextIntlClientProvider` with `oglasinoLocale` (bare language: `sr`, `en`, `ru`, `cnr`), mounts `<NavigationProgressBar />`, `<GA4RouteListener />` (new — from GA4 v1 brief 2), `<AuthInit />`, `<QuickRecommendButton />`, `<BaseSiteInit />`, `<TooltipProvider>` wrapping `<AppInit />`, `<Toaster />`, `<DrawerDialogManager />`, and children. Calls `notFound()` (line 28) when `routing.locales` doesn't include the URL locale, and again (line 35) when `baseSite` is null. Sets request locale (line 30).
**No SEO-relevant emissions here.**

### `src/metadata/generatePrivacyPageMetadata.ts` + `privacy/page.tsx:21-26` — broken inline `<script>`

Spec says: broken inline `<script>` that stringifies the wrong payload.
Code says:
- `generatePrivacyPageMetadata.ts` is otherwise normal (canonical, OG, Schema.org `PrivacyPolicy` via `other['application/ld+json']`).
- `privacy/page.tsx:21-27` body renders `<script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(generatePrivacyPageMetadata(t, locale)) }} />`. This stringifies the entire **Next.js Metadata object** (the `Metadata` return from `generatePrivacyPageMetadata`), not the Schema.org `PrivacyPolicy` payload. The inline `<script>` is delivered but its content is garbage (`{ title, description, alternates, openGraph, twitter, other }` shape, not Schema.org).

**Severity: high.** Confirms spec §9.13. The fix lands cleanly under spec §4.

### `src/metadata/generateIntroPageMetadata.ts`

Spec says: no canonical, no structured data; `metadataBase` commented-out line.
Code says:
- No canonical ✓ (no `alternates` block at all).
- No structured data ✓ (no `other['application/ld+json']`).
- **No `metadataBase` commented line** in this file. The commented `metadataBase: new URL(basicMetadata.baseUrl)` is in **`generateMainLayoutMetadata.ts:10`**, not here. Spec is off on the file location.

**Severity: low.** Confirms spec §9.14 (the substantive observation) and amends the file pointer.

### `src/metadata/generateUserPageMetadata.ts`

Spec says: canonical does not include userId.
Code says: confirmed — line 27 reads `const canonical = \`${basicMetadata.baseUrl}/${locale}${tMetadata('page.user.url')}\`;`. No `userId` is appended. Every user page emits the same canonical URL (e.g. `https://oglasino.com/rs-sr/user`), which collapses all user pages onto one canonical.
**Severity: high.** Confirms spec §9.4. (This is arguably the worst single canonical defect — Google sees N user pages with one canonical.)

### `src/metadata/generateFreeZonePageMetadata.ts`

Spec says: `inLanguage` set to the next-intl locale string.
Code says: confirmed — line 66 reads `inLanguage: locale,` where `locale` is the routing locale (e.g. `rs-sr`, `me-cnr`). Not a BCP-47 language tag.
**Severity: low.** Confirms spec §4.3 (free-zone bullet).

### `src/metadata/generateAboutMetadata.ts` + `generatePricingPageMetadata.ts` — `Organization.sameAs` shape

Spec says: `Organization.sameAs` shape is wrong (array vs record).
Code says:
- `generateAboutMetadata.ts:71`: `sameAs: basicMetadata.socialLinks` where `socialLinks` is the **object** `{ facebookLink, twitterLink, instagramLink, linkedInLink }` (`SocialLinks` interface in `BasicMetadata.ts:1-6`). Schema.org `sameAs` expects an array of URL strings.
- `generatePricingPageMetadata.ts`: does not include `sameAs` at all (its `Organization` block is the `seller` inside the `Offer.mainEntity` — only `name`, `url`, `logo`). Spec is wrong to call this out for pricing.

**Severity: medium for about; non-applicable for pricing.** Confirms spec §4.3 (about bullet); amends spec on pricing.

---

## Part 4 — Recently-shipped feature touchpoints

The three features shipped to `dev` after the SEO spec was drafted:

### Consent Mode v2 (shipped 2026-05-21)

- `app/layout.tsx` modified (commit `51df7e9` "Added cookies V2") — added the consent default snippet (lines 42-53) injected into `<head>` (line 61) via `<script dangerouslySetInnerHTML={{ __html: consentDefaultSnippet }} />`. Reads `og_consent` cookie via `readConsentForSsr` and `sanitizeForSnippet` at lines 5, 42-43.
- No `generate*Metadata.ts` files were touched.
- The consent snippet is inline `<script>` — not Schema.org, not crawler-relevant. **No cloaking risk** (snippet emits the same `gtag('consent', 'default', ...)` for crawlers as for users; consent defaults are `denied` for crawlers unless they sent an `og_consent` cookie, which they won't).
- No new files under `src/components/server/seo/` or `src/metadata/` (so no name collisions with planned files in spec §4).
- `<html lang>` handling unchanged.
- `ConsentBanner` rendered from `(portal)/layout.tsx:14` (client component) — visually present but not a crawler concern.

### Google Analytics v1 (shipped 2026-05-23)

- `app/layout.tsx` modified — added two `<Script strategy="afterInteractive">` blocks (lines 62-73) gated on `NEXT_PUBLIC_GA4_MEASUREMENT_ID`. Loads `gtag/js` and configures with `send_page_view: false`.
- `app/[locale]/layout.tsx` modified — added `<GA4RouteListener />` (line 43) as a sibling to `<NavigationProgressBar />`. Wires SPA-style `page_view` events.
- New initializer files under `src/components/client/initializers/`: `GA4ErrorListener.tsx`, `GA4RouteListener.tsx`, `GA4UserIdSync.tsx`, `ProductViewTracker.tsx`, `ViewSearchResultsTracker.tsx`.
- New module under `src/lib/analytics/`: `track.ts`, `trackError.ts`, plus tests.
- No `generate*Metadata.ts` files were touched.
- **No new files under `src/components/server/seo/` or `src/metadata/`** — so no naming collisions with planned files.
- `<html lang>` handling unchanged.
- No cookie/consent-gating logic that would render different content to crawlers vs users (GA4 is loaded for everyone; consent gating is on event emission inside `track()` via the `__og_consent_loaded` guard).

### Cookies closing (shipped piecemeal, latest 2026-05-22)

- `app/layout.tsx`, `app/[locale]/layout.tsx`, plus many other files including pages and stores touched per state.md log. Per Igor's 2026-05-22 entry: `portalScope` rename, `userPreferenceService` removal, language portion shipped.
- No `generate*Metadata.ts` files touched (verified by `git log --oneline --all -- src/metadata/` — last metadata change was 2026-05-13 in `73ab239 Pre production commit`; nothing since then).
- No new files under `src/components/server/seo/`.
- `<html lang>` handling unchanged (still `htmlLang.split('-')[0]`).
- No cloaking risk surfaced; cookie writes are gated by consent helper (`isPreferenceConsentGranted`) per the closed Risk Watch row.

**Severity rollup for Part 4:**
- **NONE BLOCKING.** No collisions with planned spec files. SEO surfaces unchanged. The spec's engineering plan still applies cleanly.
- **LOW (informational):** `app/layout.tsx` line numbers in the spec are off by ~25 due to consent snippet additions; treat spec line citations as approximate.

---

## Part 5 — Backend filter values for Schema.org mappings

The spec's §10.1 calls for a `src/metadata/schemaOrgMappings.ts` table that maps backend availability and condition filter values to Schema.org enum URLs. The spec leaves the actual values blank.

### Backend canonical filter values

Filter keys are declared in the `FilterValueEnum` enum (`oglasino-backend/src/main/java/com/memento/tech/oglasino/enums/FilterValueEnum.java`):

```java
CONDITION("condition"), AVAILABILITY("availability"), DELIVERY("delivery");
```

Actual filter options are seeded from the catalog JSON files at `oglasino-backend/src/main/resources/catalogJSON/catalog-{rs,me,rsmoto}.json`. **Verified identical across all three base-site catalogs.**

**Condition** (`filterKey: "condition"`, single-option, 7 options in order):
1. `filter.options.new_unused`
2. `filter.options.like_new`
3. `filter.options.used_excellent`
4. `filter.options.used_minor_traces`
5. `filter.options.used_visible_damage`
6. `filter.options.partially_damaged`
7. `filter.options.damaged_nonfunctional`

**Availability** (`filterKey: "availability"`, single-option, 4 options in order):
1. `filter.options.in_stock`
2. `filter.options.supplier_stock`
3. `filter.options.made_to_order`
4. `filter.options.unavailable`

**Delivery** (`filterKey: "delivery"`, multi-option, 4 options — for completeness; spec doesn't currently call for a Schema.org mapping here):
1. `filter.options.pickup`
2. `filter.options.courier_delivery`
3. `filter.options.free_delivery`
4. `filter.options.international_delivery`

### How the product carries these values

Backend `ProductFilterDTO` (`oglasino-backend/src/main/java/com/memento/tech/oglasino/dto/ProductFilterDTO.java`) shape:

```java
Long id;                       // filter row id (unique per filter — one row for "availability", one for "condition")
FilterType filterType;
String iconId;
String labelKey;
String filterValue;
List<FilterOptionDTO> options; // selected options (one entry for SINGLE_OPTION condition/availability)
Long selectedRangeValue;
String rangePrefix;
String rangeSuffix;
```

Web `ProductFilterDTO` (`oglasino-web/src/lib/types/filter/ProductFilterDTO.ts`) mirrors this — `id`, `filterType`, `labelKey`, `filterKey`, `options[]`, range fields.

`FilterOptionDTO` carries `{ id: Long, labelKey: String }`.

The selected option on a product detail page is the single entry in `product.availability.options[0]` (and `product.condition.options[0]`) — both `id` (option DB id) and `labelKey` (e.g. `filter.options.in_stock`) are present.

### Recommended Schema.org mapping

Mapping by **labelKey** (stable across base sites and DB resets, since the catalog JSON is the source of truth):

```ts
// Condition → ItemAvailability enum is wrong; use OfferItemCondition:
const conditionToSchemaOrg: Record<string, string> = {
  'filter.options.new_unused':         'https://schema.org/NewCondition',
  'filter.options.like_new':           'https://schema.org/NewCondition', // judgment call: arguably UsedCondition
  'filter.options.used_excellent':     'https://schema.org/UsedCondition',
  'filter.options.used_minor_traces':  'https://schema.org/UsedCondition',
  'filter.options.used_visible_damage':'https://schema.org/UsedCondition',
  'filter.options.partially_damaged':  'https://schema.org/DamagedCondition',
  'filter.options.damaged_nonfunctional':'https://schema.org/DamagedCondition',
};

const availabilityToSchemaOrg: Record<string, string> = {
  'filter.options.in_stock':       'https://schema.org/InStock',
  'filter.options.supplier_stock': 'https://schema.org/BackOrder',
  'filter.options.made_to_order':  'https://schema.org/PreOrder',
  'filter.options.unavailable':    'https://schema.org/OutOfStock',
};
```

The `like_new` → `NewCondition`-vs-`UsedCondition` call is the only one worth Mastermind sign-off — both are defensible.

**Severity for Part 5: medium for spec completion** (spec asked for these values; here they are). **Low for engineering** — once the spec carries this table, the engineer brief is mechanical.

---

## Part 6 — Trust-boundary verification

Per conventions Part 11 and spec §3, every SEO-emitted value must be server-derived.

| Surface | Source | Verdict |
|---|---|---|
| Canonical URLs | Built from `t('base.url')` (translation seed) + `${locale}` from `getRoutingLocale()` (reads `x-next-intl-locale` header set by middleware from URL segment) + path segments from route params. | **Server-derived.** Clean. |
| `getNormalizedProductUrl` (product canonical) | `productId` from URL param (validated by backend lookup), `productName` from backend `ProductDetailsDTO.name`. | **Server-derived.** Clean. |
| `hreflang` (planned) | Will derive from `routing.ts` locales constant + base-site filter from `getBaseSiteServer()`. | **Server-derivable.** No leak surface. |
| JSON-LD `Product` fields | All come from `getPortalProductDetails(productId)` → backend `ProductDetailsDTO`. `name`, `description`, `price`, `currency`, `free`, `imageKeys`, `availability/condition/deliveries/otherFilters` (filter DTOs from ES converters). | **Server-derived.** Clean. |
| `og:image` URLs | `publicImageUrl(key, 'hero')` with `CDN_BASE = process.env.NEXT_PUBLIC_CDN_URL` and backend-returned R2 `key`. | **Server-derived.** Env-var-driven, not client-supplied. |
| `BreadcrumbList` JSON-LD (planned) | Will derive from `BaseSiteDTO.catalog` tree + route slugs (catalog) or `ProductDetailsDTO.{topCategoryId,subCategoryId,finalCategoryId}` (product). | **Server-derivable.** Clean. |
| `Person` JSON-LD on user profiles | `getUserForId(userId)` → backend `UserInfoDTO`. | **Server-derived.** Clean. |
| Sitemap URLs | `getAllBaseSitesForSitemap()` (backend) + `getProductsPage(...)` (backend search). | **Server-derived.** Clean. |
| Robots rules | Static literals in `app/robots.ts`. | **Server-derived.** Clean. |
| `og:locale` and `og:locale:alternate` (planned) | Will be derived from current routing locale + `routing.ts` locales constant. | **Server-derivable.** Clean. |
| Catalog page metadata snapshot | `getMetadataProductsSnapshot(filters, 10)` — server action, takes filters built server-side from URL slugs (not `searchParams`). | **Server-derived.** Clean. The `searchParams` parameter is accepted by `generateCatalogPageMetadata` (line 14) but not used in canonical or JSON-LD construction — only passed for future extension. Verified by reading the function body. |

**One residual question:** the catalog `generateMetadata` accepts the route's `searchParams` (line 53-80) but does NOT pass them into `getMetadataProductsSnapshot` or into the canonical URL (the canonical excludes search params — correct behaviour). Crawlers will see only the canonical-slug-derived metadata. **No trust-boundary violation.**

**Severity: none.** No CRITICAL findings. Spec §3 holds: every SEO-emitted value is server-derived today, and the planned work preserves that.

---

## Part 7 — Anything the spec missed

### 7.1 — `Offer.availability` and `Offer.itemCondition` mapping is by **filter row id**, not option id

`src/metadata/generateProductPageMetadata.ts:159-160`:
```ts
availability: availabilityMap[product.availability?.id] || 'https://schema.org/InStock',
itemCondition: conditionMap[product.condition?.id] || 'https://schema.org/UsedCondition',
```

`product.availability` is a `ProductFilterDTO`, whose `id` is the **filter** row id (a single value per filter — one for "availability", one for "condition" — globally constant for a given DB). The selected option's id lives in `product.availability.options[0].id`. The current code therefore maps a single fixed filter id to a single enum URL — every product gets the same `availability` value, ignoring the selected option.

Today this is dead text (delivered as a `<meta>` tag — no crawler sees it). Once spec §4 ships, the bug becomes live unless the engineering brief fixes the lookup. The Part 5 mapping (by labelKey) sidesteps the bug entirely. **Severity: medium — flagged so spec §4 brief includes the fix.**

### 7.2 — `Schema.org` `Person` lacks `image` despite `profileImageKey` being available

`src/metadata/generateUserPageMetadata.ts:62-69` builds `Person` with `name`, `url`, `description` only. The `UserInfoDTO.profileImageKey` field is consumed in the OG block (line 48) but not the JSON-LD. Spec §4.3 (user profile bullet) calls this out; flagged here as an explicit finding so the engineering brief catches both touch sites. **Severity: low.**

### 7.3 — `app/[locale]/layout.tsx` does NOT call `setRequestLocale` until after `params` is awaited

Not SEO-relevant directly, but related to the spec's trust-in-route-params assumptions — verified the route validates locale via `hasLocale(routing.locales, locale)` (line 26-28) before any rendering. Clean. No finding; included so Mastermind knows it was checked.

### 7.4 — `app/error.tsx:14` carries hardcoded English `<h1>Something went wrong</h1>` plus English buttons

The `RouteError` boundary renders untranslated English copy. Not strictly SEO (error pages emit `<meta robots="noindex" follow="false">` via `app/[locale]/not-found.tsx` pattern — though `app/error.tsx` itself has no metadata), but worth flagging as a Part 4b adjacency since it's user-visible and the spec touches the error boundaries. **Severity: low (out of audit scope, flagged only).**

### 7.5 — `not-found.tsx` files: `app/not-found.tsx` and `app/[locale]/not-found.tsx`

Both exist. `app/[locale]/not-found.tsx:5` sets `metadata = { title: 'Not Found', robots: { index: false, follow: false } }`. Good. `app/not-found.tsx` (root) was not opened — assume similar but worth a one-line check at engineering time. Image `<img src="/404error.png" alt="404 error" />` has an English alt — same low-severity concern as Part 7.4. **Severity: low.**

### 7.6 — Catalog page is a **paginated** view of products with `searchParams` for filters

Spec §11.8 mentions this (post-launch). Pre-launch posture: the canonical for `/catalog/X/Y/Z?orderBy=…&page=2` is the slugs-only form (no searchParams). Confirmed in `generateCatalogPageMetadata.ts:40`. Good — duplicate-content risk is already mitigated. **No finding.**

### 7.7 — `app/[locale]/(portal)/(public)/page.tsx` (home) does NOT call `setRequestLocale`

But the parent layout (`app/[locale]/layout.tsx:30`) does. Verified consistent. **No finding.**

### 7.8 — Spec missed an EN-only inline literal that affects social previews

`generateHomePageMetadata.ts:53` hardcodes `alt: 'Oglasino'` on the OG image even though all four locales should pass a localized alt. The catalog generator does similarly at line 74 (`alt: categoryName || 'Oglasino'`). Pricing/about/privacy/terms/free-zone all use `basicMetadata.defaultOgImageAlt` (translation-keyed) correctly. **Severity: low.** Touches §9.7 (`og:locale`) work envelope.

### 7.9 — `MarkdownViewer.tsx` always fetches `privacy-policy.en.md` and `terms-of-use.en.md` regardless of locale

`privacy/page.tsx:31` and `terms/page.tsx:21` hardcode the English markdown URL. The "`// TODO Create locale privacy.md`" comment at `privacy/page.tsx:15` flags it. Spec §7 expects `<h1>` to live in the fetched markdown — fine for EN, but every locale serves English text today. This is content scope, not SEO-mechanics scope, but it interacts with hreflang: declaring `sr-RS` for a page that serves English content invites Google to demote both. **Severity: medium — flagged because hreflang work assumes content-locale matches URL-locale.**

### 7.10 — `pricing/page.tsx` hero-h1 still uses subtitle keys, not a `hero.title` key

Spec §7.2 wants `hero.title` (currently rendered in the green badge at line 28) moved into the `<h1>`. Right now the `<h1>` at line 32 still wraps `hero.subtitle.line1` + `hero.subtitle.line2`. Spec §7.2 fix has NOT been done — pricing-page h1 work is still owed. **Severity: medium (re-confirms spec §7.2).**

### 7.11 — `generateMainLayoutMetadata.ts:10` `metadataBase` commented out

Spec §9.6 calls this out. Confirmed: line 10 reads `// metadataBase: new URL(basicMetadata.baseUrl),`. Note: `generateProductPageMetadata.ts:79` sets a per-product `metadataBase: new URL(\`${basicMetadata.baseUrl}/${locale}\`)` — but this is an odd usage (path-in-base) and may behave subtly different from the layout-level fix. **Severity: low.** Engineering brief should harmonize.

### 7.12 — `Footer` and `Header` server components not inspected

Out of scope for the audit (no spec items reference them). Worth re-checking at engineering time whether either emits hreflang-relevant `<link>` tags or duplicate `<h1>`s in a way the per-page audit missed. **No finding logged.**

### 7.13 — `OglasinoBreadcrumbs.tsx` is path-wise a "server" component but functionally a client one

Uses client hooks (`useTranslations`, `useDialogStore`, `useScreenBreakpoint`) without a `'use client'` directive. Next.js will infer client behaviour from the consumer, but the folder placement is misleading. **Severity: low (Part 4b style flag, out of audit scope).**

### 7.14 — `generateProductPageMetadata.ts` JSON-LD does NOT include `priceValidUntil` or `seller`

Schema.org requires `priceValidUntil` for `Offer` rich-result eligibility (warning, not error). `seller` (the listing user) is also absent. Spec §4.3 doesn't call these out. **Severity: low — flagged for post-launch §11.4 expansion.**

---

## Audit confidence

**Solid:**
- Metadata generators inventory (Part 1) — read every file, every consumer is grep-confirmed.
- Public page files inventory (Part 2) — every page's h1 count and not-found path is read directly.
- Spec-referenced file checks (Part 3) — every callout opened; stale references corrected with current line numbers.
- Backend filter values (Part 5) — catalog JSON read for all three base sites, options confirmed identical.
- Trust-boundary verification (Part 6) — every SEO emitter traced to its server-derived source.

**Less confident:**
- The product-card-as-anchors finding (Seam 5 / §5) — only `PortalProductCard` is on the public SEO surface. The five other consumers (`PreviewProductCard`, `AdminProductCard`, `DashboardProductCard`, `FavoriteProductList` wrapper, etc.) are in noindex contexts. I have not opened every consumer to verify scope; if any of those leak onto a public route the brief needs to widen.
- The sitemap shard math (Part 3 `sitemap.ts`) — product shards count by **products** but the inner loop emits one URL **per language**, so the real URL count per shard varies with base-site language counts. Functionality is correct; performance characterisation may need attention at scale.
- The Schema.org enum mapping table in Part 5 — the `like_new` → New-vs-Used judgement is mine; Mastermind should sign off.

**Could not fully verify:**
- The router worker's `noindex` behaviour for stage and admin/owner paths — `oglasino-router` is a sibling repo I cannot read in this session. Spec §10 (last paragraph) and §11.9/§11.10 acknowledge this is post-launch defense-in-depth; the audit takes the spec's framing at face value.
- Whether the catalog JSON option IDs (assigned by DB sequence on first boot) collide across `rs`, `rsmoto`, and `me` reseed orderings. The labelKey-based mapping in Part 5 sidesteps this entirely; the existing `availabilityMap` / `conditionMap` ID-based code in `generateProductPageMetadata.ts` (lines 11-26) is brittle.
- Whether any `<h1>` lives inside the fetched privacy/terms markdown. Cannot fetch external URLs from this session; verification deferred to engineer.

---

# Session summary

**Repo:** `oglasino-web`
**Branch:** `dev`
**Date:** 2026-05-23
**Task:** Phase 2 read-only audit of SEO foundation — inventory current state of every surface the spec touches and surface every contradiction between the spec and the code as it exists right now. Output `.agent/audit-seo-foundation.md`.

**Audit session** — no code changes.

## Implemented

- Read the spec at `oglasino-docs/future/seo-foundation.md` (note: brief said `features/seo-foundation.md`; spec lives in `future/` because it has not been promoted to active yet).
- Walked `src/metadata/` (13 files) — confirmed every JSON-LD emission is via `other['application/ld+json']`, confirmed zero `alternates.languages` anywhere, confirmed canonical-host divergence (apex everywhere except `getNormalizedProductUrl`).
- Walked `app/[locale]/(portal)/(public)/` (9 page files) — confirmed h1 defects and not-found-with-HTTP-200 defects spec calls out, surfaced two spec-text inaccuracies (pricing duplicate already fixed; catalog bad-slug already calls `notFound()` — only empty-slug `redirect()` remains, which is correct).
- Opened every spec-referenced file by line number, confirmed or amended each. Noticed Igor's 2026-05-20 very-easy bug batch already closed three of the spec's small-fixes-batch items (`Kategorije iz` literal, Serbian tooltips, `markdown.fild.load` typo, pricing duplicate).
- Audited the three recently-shipped features (Consent Mode v2, GA4 v1, cookies-closing) for collision with planned spec files — none. Line numbers in `app/layout.tsx` and `app/[locale]/layout.tsx` are ~25 lines off in the spec due to consent/GA4 additions.
- Inspected backend filter seeds in sibling repo `oglasino-backend` (catalog JSON at `src/main/resources/catalogJSON/`) — produced the Schema.org mapping table the spec §10.1 asked for, by labelKey rather than by DB id.
- Verified trust boundaries per conventions Part 11 — clean. No client-supplied values reach SEO-emitted text.
- Surfaced 14 additional findings the spec did not enumerate (Part 7), most low/medium severity, one (7.1) feeding directly into the §4 engineering brief.

## Files touched

(none — read-only audit)

## Tests

N/A — read-only audit.

## Cleanup performed

none needed.

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change required from this audit. (The spec's planned closing entry on the SEO triage is for after Phase 5.)
- `state.md`: no change required from this audit. (Promotion of `future/seo-foundation.md` to `features/` and addition of an active-feature entry is a Phase 4 / Docs/QA task driven by Mastermind, not this audit.)
- `issues.md`: no change required from this audit, BUT three entries surfaced as already-closed-but-not-flipped during the audit walk — see "For Mastermind" below.

## Obsoleted by this session

nothing — read-only audit.

## Conventions check

- Part 4 (cleanliness): N/A (no code touched).
- Part 4a (simplicity): N/A (no complexity added or removed). Structured evidence skeleton:
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- Part 4b (adjacent observations): three low-severity items surfaced — flagged below in "For Mastermind." None fixed.
- Part 6 (translations): one violation found (`app/error.tsx` hardcoded English copy); flagged below. One per the spec's existing inventory (`alt="Missing image"`, `alt="Oglasino intro"`, etc.) re-confirmed.
- Part 10 (feature lifecycle): this is the Phase 2 audit for the `seo-foundation` feature. Phase 1 happened in Mastermind; Phase 4 (canonical spec) was drafted but not Docs/QA-applied (spec lives in `future/`, not `features/`).
- Part 11 (trust boundaries): explicitly verified in Part 6 of the audit body. Clean.

## Known gaps / TODOs

- The audit cannot fetch external URLs, so the privacy/terms markdown content (whether it starts with a top-level heading) is not verified.
- The router worker (`oglasino-router`) was not inspected — out of scope for `oglasino-web` audit.
- Whether the `like_new` condition should map to `NewCondition` or `UsedCondition` in the Schema.org table is a judgment call that needs Mastermind sign-off.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Top three findings by severity (HIGH):**
  1. JSON-LD delivery is broken project-wide (every page emits Schema.org as `<meta>` tags). Confirms spec §4.1. **Blocks rich-result eligibility entirely until fixed.**
  2. Hreflang is entirely absent (zero `alternates.languages` across 10 locales × 9 page types). Confirms spec §6.1. **Largest single tax on launch-day discoverability.**
  3. User-page canonical does not include `userId` — every user page emits the same canonical URL (Part 3 / spec §9.4). **The worst single canonical defect; collapses N user pages onto one canonical for Google.**

- **Seam confirmations vs contradictions (spec §2):**
  - Seam 1 (apex canonical) — **confirmed, partial.** Sitemap, robots, and all `t('base.url')`-based metadata already use apex. Only `getNormalizedProductUrl` (utils.ts:122) still hardcodes `www.`. Engineering envelope is one-line fix.
  - Seam 2 (bad catalog slug → 404) — **partially shipped already.** `catalog/[[...slugs]]/page.tsx:102` already calls `notFound()`; only the empty-slug `redirect()` at line 96 remains, which is correct behaviour. Spec text needs amending; the two `issues.md` 2026-05-14 catalog-slug entries may already be closeable.
  - Seam 3 (product slug deferred) — **confirmed.** No backend slug field; `ProductCanonicalSlugRewriter.tsx` exists today as a client-side `router.replace` (NEW finding — was not in spec). Worth a Mastermind decision on whether this client-side rewriter should also be killed when the proper 301 redirect path lands post-launch.
  - Seam 4 (server breadcrumbs) — **confirmed.** `ProductBreadcrumbs.tsx` is `'use client'` + Zustand-gated. Initial HTML emits nothing.
  - Seam 5 (image URL contract at web) — **confirmed.** `publicImageUrl` is web-only, backend ships R2 keys.
  - Seam 6 (`free` field) — **confirmed.** `product.free` boolean is consumed in JSON-LD `Offer.price` with `product.free ? '0' : product.price` (line 158).
  - Seam 7 (`me-cnr` independent locale) — **partially actionable.** `routing.ts` already lists `me-cnr` as a distinct locale; `mapToIntlLocale` returns `cnr-ME`; `app/layout.tsx` `<html lang>` truncates to `cnr`. Fix is in the spec; needs the conventions Part 9 correction to be applied by Docs/QA before engineering runs.
  - Seam 8 (inactive product → 404 via web) — **confirmed.** `product/.../page.tsx:77-82` returns inline `<div>` at HTTP 200.

- **Could not verify / open questions:**
  - Router worker behaviour (separate repo).
  - Whether external markdown fetched by privacy/terms has a top-level heading (cannot fetch external URLs).
  - `like_new` Schema.org mapping (judgment call).
  - Whether the cross-base-site `redirect()` paths in product and user pages (e.g. `product/[productId]/[productName]/page.tsx:72-74`) should preserve canonical apex host. They use `getNormalizedProductUrl(..., false)` which excludes host prefix, then prepend `/${targetBaseSite.code}-${finalLocale}` — the result is a relative path, so the Next.js redirect lands on whatever host is serving. Likely fine but worth a Mastermind check.

- **Adjacent observations (Part 4b):**
  - `app/error.tsx:14-17` carries untranslated English `<h1>` and body copy — severity low (file is the framework error boundary, hit on uncaught exceptions). I did not fix this because it is out of scope.
  - `app/not-found.tsx` carries `alt="404 error"` hardcoded English alt — severity low. Out of scope.
  - `OglasinoBreadcrumbs.tsx` lives in `src/components/server/` but uses client hooks without a `'use client'` directive — severity low (file naming/folder mismatch). Out of scope.
  - `NotFound.tsx` lives in `src/components/server/` but is `'use client'` at line 1 — same low-severity folder mismatch.

- **Drafted config-file text (none authored; pointers only):**
  - **`issues.md` candidates (potential status flips, for Docs/QA verification at the next opportunity):** the spec's small-fixes-batch (§9.10 Serbian tooltips, §9.11 markdown.fild.load typo, §9.12 pricing duplicate, §9.4 `OglasinoBreadcrumbs.tsx:81` Kategorije iz literal) are **already closed in code** by Igor's 2026-05-20 very-easy bug batch but the matching `issues.md` entries may not have been flipped. (state.md does mention "Net `issues.md` change: 9 entries flipped to `fixed`" for that batch — so they probably are flipped. Docs/QA can verify with one grep at next session.) **No draft text from this audit.**
  - **`state.md` candidate:** when the spec is promoted from `future/` to `features/` after Phase 4, an active-feature entry will be needed. Out of scope for the audit; flagged to Mastermind.
  - **`decisions.md`:** no draft from this audit. Mastermind authors the spec-promotion + seam-resolution closing entry at Phase 4 close.

- **Spec-location discrepancy (informational, not blocking):** brief said `oglasino-docs/features/seo-foundation.md` but the spec is at `oglasino-docs/future/seo-foundation.md`. Since the feature has not been Phase-4-promoted yet, `future/` is appropriate. Either the brief should be amended or the Phase 4 promotion should happen before any engineering brief runs.
