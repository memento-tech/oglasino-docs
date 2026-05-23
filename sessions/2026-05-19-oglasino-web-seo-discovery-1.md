# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-19
**Task:** Audit `oglasino-web` for everything that affects SEO — indexability, structured data, sitemap, robots, canonical URLs, hreflang, rendering strategy, metadata, internal linking, headings, slugs, image alt discipline.

## Findings

Each finding: description — file:line — status — severity — note.

Format key:
- status: `correct` / `broken` / `partial` / `missing` / `n/a`
- severity: `critical` / `high` / `medium` / `low`

### 1. Metadata generation

The repo has thirteen `generate*Metadata.ts` files plus a shared `generalMetadata.ts`. The dominant delivery pattern for structured data is `metadata.other['application/ld+json']`. Next.js renders the `other` map as `<meta name="..." content="...">` tags, **not** as `<script type="application/ld+json">`. Crawlers (Google, Bing, schema.org parsers) only honor JSON-LD inside `<script>` tags. So every page below that places JSON-LD in `other` ships structured data that is technically present in HTML but is invisible to crawlers.

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| `metadata.other['application/ld+json']` delivers JSON-LD as `<meta>` not `<script>` (Home) | `src/metadata/generateHomePageMetadata.ts:64-66` | broken | critical | Home page WebSite + ItemList schema rendered as `<meta application/ld+json>` tag — invisible to crawlers. |
| Same — Catalog | `src/metadata/generateCatalogPageMetadata.ts:85-89` | broken | critical | ItemList schema unreachable. |
| Same — Product detail | `src/metadata/generateProductPageMetadata.ts:104-108` | broken | critical | Product/Offer schema with prices, condition, availability, brand — invisible. Highest-value page type for rich results. |
| Same — User profile | `src/metadata/generateUserPageMetadata.ts:50-54` | broken | critical | Person schema invisible. |
| Same — About | `src/metadata/generateAboutMetadata.ts:45-47` | broken | high | AboutPage + Organization invisible. |
| Same — Pricing | `src/metadata/generatePricingPageMetadata.ts:44-46` | broken | high | WebPage/Offer invisible. |
| Same — Privacy | `src/metadata/generatePrivacyPageMetadata.ts:45-49` | broken | medium | PrivacyPolicy schema invisible. |
| Same — Terms | `src/metadata/generateTermsPageMetadata.ts:45-49` | broken | medium | TermsOfService schema invisible. |
| Same — Free zone | `src/metadata/generateFreeZonePageMetadata.ts:44-46` | broken | high | Article schema invisible. |
| Privacy page has a real `<script type="application/ld+json">` tag — but stringifies the **entire Metadata object**, not just the schema | `app/[locale]/(portal)/(public)/privacy/page.tsx:21-26` | broken | high | The script body is `JSON.stringify(generatePrivacyPageMetadata(t, locale))` — that's the Next.js Metadata return shape (`title`, `description`, `openGraph`, etc.), not a Schema.org document. Invalid JSON-LD that crawlers reject. |
| Intro/locale-selector page (`/`) emits no canonical, no structured data | `src/metadata/generateIntroPageMetadata.ts:14-39` | partial | medium | Only title/description/og/twitter. No `alternates.canonical`, no Organization or WebSite schema for the apex domain. |
| `generateMainLayoutMetadata` sets `metadataBase` only by a commented-out line | `src/metadata/generateMainLayoutMetadata.ts:10` | broken | high | `metadataBase` is commented out at the root layout. Without it, Next.js logs a build warning and resolves relative OG/canonical URLs against `http://localhost:3000` in absence of an explicit base. Only `generateProductPageMetadata.ts:79` sets `metadataBase` locally — every other public route runs without a base, but happens to pass absolute URLs already. |
| `t('og.image.url')` and `t('og.image.alt')` come from translation table | `src/metadata/generalMetadata.ts:19-20` | partial | low | OG image URL is a translation key — works if the SQL seed has an absolute URL, fails silently if relative. Cannot verify without backend. |
| Product not-found path returns minimal Metadata with `index:false, follow:true` | `src/metadata/generateProductPageMetadata.ts:57-66` | correct | low | Correct behavior, but the route still returns 200 status rather than triggering `notFound()` — see slug handling. |
| Title pattern inconsistent: product detail hardcodes ` \| Oglasino` suffix | `src/metadata/generateProductPageMetadata.ts:71,82,97` | partial | low | Other pages title via translation key; product page hardcodes the brand suffix. Hardcoded "Oglasino" also at `generateProductPageMetadata.ts:122` brand fallback. |

### 2. Server vs client rendering

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| Home `(public)/page.tsx` is an `async` Server Component; product data fetched server-side and passed to client list | `app/[locale]/(portal)/(public)/page.tsx:24-67` | partial | high | Server fetches products and SSRs the initial HTML of the client wrapper. Product *titles, prices, top-images* are present in initial HTML via `<UniversalProductCard>` server render. BUT see Item 8 — they are wrapped in `<div onClick>` not `<a href>`, so crawlers see text without crawlable links. |
| Catalog page is async Server Component, same pattern as home | `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:80-171` | partial | high | Same SSR pattern. Same "text in HTML, no anchor href" problem. |
| Product detail page is async Server Component | `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:48-107` | correct | n/a | Product name, breadcrumb data, owner info SSR'd. |
| `ProductDetails` named server file is actually `'use client'` | `src/components/server/ProductDetails.tsx:1` | partial | medium | Lives under `src/components/server/` but starts with `'use client';`. Functionally OK (renders fine), but misleading directory placement and means description/price/filters are technically hydrated client-side rather than emitted from a true server component — still appears in HTML because the parent server page renders the client tree. |
| `UserDetails` on product page and user page is `'use client'` | `src/components/client/UserDetails.tsx:1` | partial | high | The displayName, location, bio, active-products-count are inside a client component. The text is in the initial HTML (rendered through SSR of the client tree), but `<div>`-based, no semantic role, no profile-link anchor. |
| `ProductBreadcrumbs` is client and computes the chain in `useMemo` against `useBaseSiteStore` | `src/components/client/product/ProductBreadcrumbs.tsx:1,16-53` | broken | high | Because the breadcrumb chain depends on `useBaseSiteStore` (Zustand, client), the initial server render of this component has no base site and renders nothing (`if (!baseSite) return null`). Breadcrumbs are **not in the initial HTML** for product or catalog pages. Lost internal linking and lost BreadcrumbList semantics. |
| About / Pricing / Privacy / Terms / Free-zone pages are pure server components | `app/[locale]/(portal)/(public)/{about,pricing,privacy,terms,blog/free-zone}/page.tsx` | correct | low | Static content fully SSR'd. |
| User profile page is async Server Component | `app/[locale]/(portal)/(public)/user/[userId]/page.tsx:30-89` | correct | n/a | Data is fetched server-side; rendered through client UserDetails. |
| `ExtraProductsComponent` on user profile loads in `useEffect` via `requestIdleCallback`, with no SSR data | `src/components/client/product/ExtraProductsComponent.tsx:18-32` | partial | medium | "Similar products" carousel is fetched client-side post-hydration. Initial HTML has only a skeleton. Not indexable as related-product links. |

### 3. Sitemap

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| `app/sitemap.ts` exists and shards via `generateSitemaps()` | `app/sitemap.ts:86-100` | correct | n/a | Shard 0 = static, shards 1..N = product slices of 5000 each. |
| Static-shard locale coverage iterates every base-site × allowedLanguages combo | `app/sitemap.ts:43-52,114-123` | correct | n/a | Covers all 10 routing locales (`rs-sr`, `rs-en`, `rs-ru`, `rsmoto-sr`, `rsmoto-en`, `rsmoto-ru`, `me-sr`, `me-cnr`, `me-en`, `me-ru`) per `src/i18n/routing.ts:6-15`. |
| Brief reference to "four locales" is a language list, not the route set | `src/i18n/routing.ts:6-15` | n/a | low | Brief Section 3 / 5 / 6 says "the four locales (SR/EN/RU/CNR)". The repo has 10 routes — 3 tenants × allowed languages. Surfacing for Mastermind. |
| Static pages list omits `/intro` (apex `/`) | `app/sitemap.ts:17-28,107-112` | partial | medium | Only the bare apex URL is included for the locale-selector. Per-locale URLs include `''`, `/about`, `/pricing`, `/privacy`, `/terms`, `/blog/free-zone`. No `/catalog` or `/user/...` for static. Catalog category pages are NOT in the sitemap. |
| Product URLs included in sitemap with locale prefix | `app/sitemap.ts:165-174` | correct | n/a | `/{tenant}-{lang}/product/{id}/{slug}`. |
| Sitemap has no `alternates`/hreflang entries per URL | `app/sitemap.ts:107-126,165-174` | missing | high | Each per-locale URL is emitted independently. No `<xhtml:link rel="alternate">` siblings. Google will not link the language variants. |
| `lastModified` on product entries is missing | `app/sitemap.ts:168-173` | partial | medium | Static entries have `lastModified: now`. Product entries omit it entirely. Crawlers cannot detect product updates. |
| Static entries use `lastModified: new Date()` (now) | `app/sitemap.ts:103,108,118` | partial | low | Builds at revalidation time, so the value is "now at last revalidate". Will signal continuous change to crawlers; legal pages with `changeFrequency: yearly` will look inconsistent. |
| Catalog category routes (e.g. `/catalog/elektronika`) not in sitemap | `app/sitemap.ts:17-28` | missing | high | Category pages have full metadata + SSR + canonical, but no sitemap entry. Discovery depends entirely on internal linking, which is itself partial (see Item 8). |
| Sitemap fetches via `FETCH_BACKEND_API.post('/public/product/search', ...)` per shard — N+1 cost grows with shard count | `app/sitemap.ts:54-78,128-185` | partial | low | Each shard fetches all base-site totals (twice — once in `generateSitemaps`, once per shard), then pages through. Probably fine at current scale; worth noting. |
| Sitemap exposes per-language product URLs by emitting one row per (base-site, language) tuple | `app/sitemap.ts:167-175` | partial | medium | Without hreflang, this looks like duplicate content for the same product across 3-4 languages. |
| `revalidate = 86400` for daily refresh | `app/sitemap.ts:13` | correct | n/a | Reasonable. |

### 4. Robots

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| `app/robots.ts` exists, single rule for `*` UA | `app/robots.ts:5-29` | correct | n/a | Disallows `/admin`, `/admin/`, `/owner`, `/owner/`, `/messages`, `/notifications`, `/favorites`, `/test`, `/wants`, `/icons`, `/api/`. |
| Sitemap referenced from robots | `app/robots.ts:26` | correct | n/a | `${SITE_URL}/sitemap.xml`. |
| Host directive set | `app/robots.ts:27` | correct | low | `host: 'https://oglasino.com'`. Note: `host` is a Yandex-only directive; Google ignores it. |
| Disallow paths are locale-agnostic, but routes are locale-prefixed | `app/robots.ts:11-23` | broken | high | Real URLs are `/{locale}/admin`, `/{locale}/owner`, `/{locale}/messages`, etc. Robots disallows `/admin` (apex only) — which doesn't exist as a route. Crawlable URLs like `/rs-sr/admin` will be **allowed**. Same for `/{locale}/owner`, `/{locale}/messages`, `/{locale}/notifications`, `/{locale}/favorites`, `/{locale}/wants`, `/{locale}/test`, `/{locale}/icons`. |
| `/design` route exists, not in disallow | `app/[locale]/design/page.tsx` | missing | medium | Dev-tool design system page is publicly reachable; not in the disallow list. |
| No disallow on query-string variants (`/catalog?...`, `/?...`) | `app/robots.ts:11-23` | missing | medium | Catalog accepts arbitrary filter searchParams (price, region, category etc.) — Googlebot will explore each combination as a separate URL, generating massive duplicate-content variants. Canonical (Item 5) does not include searchParams, so canonicalization helps, but a `Disallow: /*?` or per-pattern disallow would help crawl budget. |
| No `public/robots.txt` | `public/` listing | n/a | n/a | Only `app/robots.ts`. Fine — Next 15 prefers the file-based generator. |

### 5. Canonical URLs

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| Home canonical = `${baseUrl}/${locale}` | `src/metadata/generateHomePageMetadata.ts:20` | correct | n/a | Locale-scoped canonical, but `baseUrl` comes from translation table (`base.url`). |
| Catalog canonical = `${baseUrl}/${locale}/catalog/${slugs.join('/')}` | `src/metadata/generateCatalogPageMetadata.ts:40` | correct | n/a | Does not include search params — correct for canonicalization. |
| Product canonical = `https://www.oglasino.com/{locale}/product/{id}/{slug}` (hardcoded host inside `getNormalizedProductUrl`) | `src/lib/utils/utils.ts:122` + `src/metadata/generateProductPageMetadata.ts:68` | partial | medium | Host is hardcoded to `www.oglasino.com` inside `getNormalizedProductUrl` regardless of env. The sitemap uses `https://oglasino.com` (no `www`) at `app/sitemap.ts:9` and the JSON-LD ItemList URLs come through `publicImageUrl` and `getNormalizedProductUrl`. Mixed `www` vs apex creates canonical/sitemap host mismatch — Google may pick whichever it likes. |
| User profile canonical = `${baseUrl}/${locale}${t('page.user.url')}` | `src/metadata/generateUserPageMetadata.ts:27` | broken | high | The canonical URL does **not** include the user ID. Every user profile page emits the same canonical (the generic `/user` URL pattern from the translation table) — so crawlers will treat all user pages as duplicates of one page. |
| About canonical = `${baseUrl}/${locale}${t('page.about.url')}` | `src/metadata/generateAboutMetadata.ts:15` | correct | n/a | Translation provides the path slug per locale. |
| Pricing canonical | `src/metadata/generatePricingPageMetadata.ts:14` | correct | n/a | Same pattern as About. |
| Privacy canonical | `src/metadata/generatePrivacyPageMetadata.ts:15` | correct | n/a | Same. |
| Terms canonical | `src/metadata/generateTermsPageMetadata.ts:15` | correct | n/a | Same. |
| Free-zone canonical | `src/metadata/generateFreeZonePageMetadata.ts:14` | correct | n/a | Same. |
| Intro page (apex `/`) has no canonical | `src/metadata/generateIntroPageMetadata.ts:14-39` | missing | medium | The apex/locale-selector emits no `alternates.canonical`. |
| Trailing slash normalization not implemented anywhere | (no file) | missing | low | Next.js default is no trailing slash. No `trailingSlash` setting in `next.config.ts`. Default is fine but not explicit. |

### 6. Hreflang

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| `alternates.languages` is never set in any metadata generator | `src/metadata/*` | missing | critical | Zero pages emit `<link rel="alternate" hreflang="...">`. Across 10 routing locales × ~7 public route types ≈ 70 page-variants of the same content with no language linking. Google sees them as duplicates rather than language variants. |
| `x-default` alternate not set | (no file) | missing | high | No way for Google to know which locale to surface by default. |
| ME/CNR aliasing convention not represented | `src/i18n/routing.ts:6-15` | n/a | medium | Conventions Part 9 says "Montenegrin (me/cnr) aliases to SR" but the routing config lists `me-sr`, `me-cnr`, `me-en`, `me-ru` as independent locales. If me/cnr is supposed to be a content-alias of SR, that's not visible anywhere in code — neither in hreflang nor in the sitemap, both of which treat each locale independently. Surfacing as a seam to Mastermind. |
| HTML `lang` is set on `<html>` at root | `app/layout.tsx:36,42` | correct | low | `lang={htmlLang.split('-')[0]}` — yields `sr`, `en`, `ru`, `cnr`. OK. |

### 7. Structured data coverage

Combine the per-page schema findings into a coverage matrix. (Recall: only the privacy page attempts to actually emit a `<script>` tag, and it emits the wrong payload — see Item 1. So in practice, **no Schema.org documents are reaching crawlers anywhere on the site**.)

| Page | Schema present in code | Status | Severity | Note |
|---|---|---|---|---|
| Home | `WebSite` (with SearchAction) + `ItemList` of 10 newest | broken | critical | Delivered via `meta` tag (see Item 1). `Organization` not emitted on home. |
| Catalog | `ItemList` only — no `CollectionPage`, no `BreadcrumbList` | partial | high | Brief asks for `CollectionPage` + `BreadcrumbList`; only ItemList is present, and even that is `meta`-tag-only. |
| Product detail | `Product` with `Offer`, `Brand`, condition, availability, deliveries, additional properties, `Place` for location | partial | critical | Schema content is rich and accurate, but delivered via `meta` (see Item 1). Missing: `BreadcrumbList`, `aggregateRating`, `review`. `Offer.url` uses `getNormalizedProductUrl(...)` — hardcoded `www.oglasino.com` host. |
| User profile | `Person` with name, url, description | partial | high | Delivered via `meta`. Canonical itself is broken (Item 5). No `image` property — could populate with `profileImageKey`. |
| Pricing | `WebPage` with `mainEntity: Offer` (price 0, RSD) | partial | medium | Schema delivered via `meta`. Comment at `:71` (`// or RSD if you prefer`) suggests unresolved currency intent. |
| About | `AboutPage` with `mainEntity: Organization`, `contactPoint`, `review[]` | partial | high | Delivered via `meta`. `sameAs` is given as an object (`basicMetadata.socialLinks` is a record) but Schema.org expects an array of URLs — invalid even if it were a `<script>`. |
| Privacy | `PrivacyPolicy` | broken | high | One delivery via `meta` (item 1) + one via `<script>` carrying entire Metadata object (item 1). Neither valid. |
| Terms | `TermsOfService` | partial | medium | Delivered via `meta`. |
| Free zone | `Article` | partial | medium | Delivered via `meta`. Has `inLanguage: locale` but `locale` is the next-intl locale string (e.g. `rs-sr`), not a BCP-47 language code that Schema.org expects. Same issue in Privacy/Terms. |
| Apex `/` (intro) | — | missing | medium | No structured data at all. Could carry `Organization` + `WebSite`. |

### 8. Internal linking

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| Product cards on home and catalog navigate via `onClick` + `router.push`, NOT anchor tags | `src/components/client/product/PortalProductCard.tsx:27-30` + `src/components/client/product/UniversalProductCard.tsx:38-44` | broken | critical | Every product card is a `<Card>` containing a `<div onClick>` — no `<a href>`. Crawlers cannot follow products from home or category pages. Combined with no catalog category URLs in sitemap (Item 3), product discoverability outside the per-product sitemap shards is severely degraded. |
| "See user products" button on product page = `<Button onClick={router.push('/user/' + id)}>` | `src/components/client/UserDetails.tsx:173-178` | broken | high | No anchor; user profile not crawler-discoverable from product pages. |
| Catalog category navigation in `Header` uses `<Link href>` (next-intl) | `src/components/server/CategoryButton.tsx:41-58` + `src/components/server/layout/CategoryNavigation.tsx:18-29` | correct | n/a | Category links from header use proper anchors. |
| Breadcrumbs use next-intl `<Link>` | `src/components/server/OglasinoBreadcrumbs.tsx:41-67` | partial | high | Anchors exist, but the parent client component (`ProductBreadcrumbs.tsx`) renders nothing during SSR (waits on `useBaseSiteStore`) — so initial HTML has no breadcrumbs. See Item 2. |
| Footer category list uses next-intl `<Link>` | `src/components/server/layout/Footer.tsx:27-37` | correct | n/a | Top categories linked from footer — best internal-linking signal we have. |
| Footer help and company nav use next-intl `<Link>` | `src/components/server/layout/Footer.tsx:42-67` | correct | low | Routes come from `companyNavigations` / `helpNavigations` data files. |
| Apex `/` (intro page) base-site selector uses raw `<a href>` | `app/page.tsx:60-71` | correct | low | Bare `<a>` is fine; bypasses next-intl Link, no prefetch, but crawlable. |
| `ExtraProductsComponent` (similar products / user products carousel) renders only after `requestIdleCallback` | `src/components/client/product/ExtraProductsComponent.tsx:18-50` | broken | high | No links to related products in initial HTML on product or user pages. |
| Product detail page emits no `<a>` to related products, related categories, or sibling products | `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:87-106` | missing | high | Only breadcrumbs (which are missing — see above) and the owner block link to category. Category leaf is a brand-named link only. |

### 9. Heading hierarchy

| Page | H1 count | Detail | Status | Severity | Note |
|---|---|---|---|---|---|
| Home `(public)/page.tsx` | 0 | No `<h1>` anywhere in the public/page tree | broken | critical | Home has no H1 element. SEO title comes from `<title>` only. |
| Catalog | 0 | Catalog page also has no H1 | broken | critical | No H1. The bottom-category name is rendered inside a `<p>` only when there are zero products (`page.tsx:150-164`). |
| Product detail | 2 | `src/components/server/ProductDetails.tsx:69` `<h1>{productDetails.name}` and `:81` `<h1>{price}` | broken | critical | Two H1s on the same page. The price wrapped in `<h1>` is a styling abuse (this whole component is `'use client'` despite the folder). |
| User profile | 0 | `UserDetails.tsx:118` renders displayName inside `<div>` only | broken | high | No H1 for user profile. |
| About | 1 | `app/[locale]/(portal)/(public)/about/page.tsx:26` wraps `<OglasinoIcon>` only — no text | broken | high | H1 contains no text — just the SVG logo. Crawlers extract empty H1. The visible heading is in `<p>` at `:29`. |
| Pricing | 1 | `pricing/page.tsx:31` H1 with `hero.subtitle.line1` (printed twice with `<br>`) | partial | medium | H1 duplicates the same translation key with a `<br>` between — that's the visible heading text doubled. Looks like a leftover. |
| Free zone | 1 | `blog/free-zone/page.tsx:27` H1 with `header` translation | correct | n/a | Proper H1. |
| Privacy | 0 | `privacy/page.tsx` renders only `<MarkdownViewer>` | partial | medium | H1 depends on the external markdown content fetched from GitHub. Cannot verify from code. |
| Terms | 0 | Same as privacy | partial | medium | Same. |
| Intro/apex `/` | 1 | `app/page.tsx:47` `<h1>` with `welcome.title` | correct | n/a | Single H1. |
| `NotFound` server component renders H1 | `src/components/server/NotFound.tsx:17` | n/a | low | Used for 404 fallback (e.g. user not found returns NotFound component, not `notFound()`). |

### 10. Image alt discipline

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| `ProductTopImage` (home/catalog cards) uses meaningful alt | `src/components/client/product/ProductTopImage.tsx:35` | correct | n/a | `alt={productName}`. |
| Product detail carousel main image: `alt="Missing image"` regardless of state | `src/components/client/ProductImageCarusel.tsx:65` | broken | high | Hardcoded English-ish placeholder alt on every product image, every page, every locale. Should be `alt={productName}` or similar. The thumbnails have `alt=""` (decorative-OK). |
| About page hero image uses literal `'Oglasino portal'` | `app/[locale]/(portal)/(public)/about/page.tsx:36` | partial | low | Hardcoded English alt instead of a translation key. |
| About page mission images use translated text alt | `app/[locale]/(portal)/(public)/about/page.tsx:57,67` | correct | n/a | Translated alt strings. |
| About testimonials avatars: `alt={tes.name}` | `app/[locale]/(portal)/(public)/about/page.tsx:90` | correct | low | Person name alt — adequate. |
| Apex `/` intro uses `alt="Oglasino intro"` hardcoded English | `app/page.tsx:25,33` | partial | low | Hardcoded English alt. |
| Apex `/` base-site flag images: `alt=""` | `app/page.tsx:67` | correct | n/a | Decorative — empty alt is correct since the country name is in the anchor text. |
| Various dialog images use `alt=""` | `src/components/popups/dialogs/AdminReviewOverviewDialog.tsx:116`, `ViewMessageImagesDialog.tsx:30`, etc. | n/a | n/a | Not on public pages — out of scope. |

### 11. Slug handling

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| Catalog optional-catchall `[[...slugs]]` resolves via `getCategoriesFromPathSlugs` | `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:28-47,90-94` | partial | medium | If slugs do not match a category chain, function returns `undefined` and the page calls `redirect('/{locale}')` → 307 redirect to the locale root. Bad slug redirects rather than 404s, so junk URLs (typos, abuse) keep returning a 307 to home indefinitely. Crawlers and Google Search Console treat this as soft-404 / redirect-chain duplication. |
| Catalog `generateMetadata` on bad slug returns `{}` | `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:63-65` | partial | medium | Bad-slug page that redirects still computes metadata first — returning `{}` is a no-op merge with layout. Fine, but emits no signal. |
| Product page uses `[productId]/[productName]` and only `productId` is used to fetch | `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:31-32,52-53` | partial | medium | The `productName` slug segment is ignored on lookup — any name (or none) returns the product. So `/{locale}/product/123/anything` resolves to product 123. No 301 redirect to the canonical slug. Duplicate-content risk: every (id, arbitrary-slug) pair returns 200 with the same content; the canonical tag points at the normalized slug, but Google often picks one of the many crawled variants. |
| Product page redirects when `baseSite.code !== productDetails.baseSiteOverview.code` | `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:58-72` | correct | low | Used to send users to the correct base-site locale when they hit a product from the wrong site. Uses `redirect()` (307). |
| Product not-found: `productDetails` falsy → renders inline message, not `notFound()` | `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:74-79` | broken | high | Returns 200 with "not found" text. Should call `notFound()` so the response is 404. SEO impact: dead product IDs index as live 200 pages with little content. |
| Product slug normalization | `src/lib/utils/utils.ts:129-148` | correct | n/a | Lowercases, strips diacritics, replaces special chars with `-`, trims. Reasonable. |
| `// TODO FIX URL (add locale)` comment on `getNormalizedProductUrl` | `src/lib/utils/utils.ts:115` | partial | medium | Existing TODO marker. Function does take an optional locale, but `withPrefix:false` callers (e.g. `app/sitemap.ts:170`, `PortalProductCard.tsx:29`) produce locale-less paths that are then prefixed only when navigation already happens within a locale. |
| User-page slug = numeric ID only (`/user/{userId}`) | `app/[locale]/(portal)/(public)/user/[userId]/page.tsx:16-17,33-43` | partial | low | No display-name slug. Acceptable; just worth noting that user URLs are not human-readable. `NotFound` renders for missing user but page still returns 200. |

### 12. Open Graph + Twitter Cards

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| Root layout sets `openGraph.siteName`, `type: 'website'`; sets `twitter.site`, `twitter.creator`, `twitter.card: 'summary_large_image'` | `src/metadata/generateMainLayoutMetadata.ts:18-27` | correct | n/a | Sensible defaults. |
| Home / catalog / product / about / pricing / privacy / terms / free-zone set per-page `og:title`, `og:description`, `og:url`, `og:image` | (each metadata file) | correct | n/a | All include explicit absolute URLs for images via `publicImageUrl(...)` which returns CDN URLs. |
| Product detail OG image array maps every image key | `src/metadata/generateProductPageMetadata.ts:87-93` | correct | low | Multiple images emitted; first is primary. Twitter mirrors. |
| `og:locale` and `og:locale:alternate` are never set | (none) | missing | high | No localized OG locale signal — Facebook/LinkedIn use OG to pick the right language. |
| `metadataBase` not set at root layout | `src/metadata/generateMainLayoutMetadata.ts:10` (commented) | broken | high | Next.js logs a warning and resolves any relative URL against `localhost:3000`. Only product metadata sets it (`src/metadata/generateProductPageMetadata.ts:79`). Pages with already-absolute OG image URLs avoid the bug, but pages using shorthand could break. |
| OG image dimensions hardcoded `1200x630` | (multiple files) | correct | n/a | Standard Facebook size. |
| Product OG image dimensions = `800x600` | `src/metadata/generateProductPageMetadata.ts:89-92` | partial | low | Inconsistent with the rest (1200x630). Twitter `summary_large_image` recommends 2:1 (~1200x600+). |
| Twitter card images use absolute URLs via `publicImageUrl` | (multiple) | correct | n/a | OK. |
| User profile sets `openGraph.type: 'profile'` | `src/metadata/generateUserPageMetadata.ts:39` | correct | low | Correct OG type, but `og:profile:first_name`/`username` not set. |

### 13. Performance signals visible in code

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| Only `app/[locale]/(portal)/(public)/about/page.tsx` uses `next/image` | `app/[locale]/(portal)/(public)/about/page.tsx:9` | partial | high | Every other public page uses raw `<img>`. Includes: apex `/` intro (`app/page.tsx:25,31,64`), product carousel (`ProductImageCarusel.tsx:63,91`), home/catalog cards (`ProductTopImage.tsx:32`), loading screen (`app/loading.tsx`), 404 page (`app/not-found.tsx`). No automatic responsive sizes, no AVIF/WebP, no priority hints, no width/height attributes on most. |
| `loading="lazy"` only on `ProductTopImage` | `src/components/client/product/ProductTopImage.tsx:41` | partial | medium | Card thumbnails lazy. Product carousel main image (`ProductImageCarusel.tsx:63-69`) has no `loading` attr — should be eager for above-the-fold but `fetchpriority="high"` is also missing. |
| No `dynamic(... { ssr: false })` calls anywhere | (none) | correct | n/a | grep returns nothing. |
| Inter font with `display: 'swap'` | `app/layout.tsx:14-18` | correct | n/a | OK. |
| `app/page.tsx` (apex) uses `'use server'` semantics implicitly (it's `async`) but is otherwise simple | `app/page.tsx` | correct | n/a | — |
| Client bundle: home/catalog list pulls in Zustand stores and `useScreenBreakpoint` on initial render | `src/components/client/product/ProductList.tsx:1-15` | partial | low | Reasonable; not measured here. |
| Many client components are loaded unconditionally on public pages (`HeaderModeSwitch`, `CookieBanner`, `Toaster`, dialog manager, `FirebaseWorkerInit`, `AuthInit`, `BaseSiteInit`, `AppInit`, `QuickRecommendButton`, etc.) | `app/[locale]/layout.tsx:36-63`, `app/layout.tsx:44` | partial | medium | Heavy initialization tree on every public route. Affects LCP/INP indirectly. Not measurable from code alone. |
| `app/sitemap.ts` has `export const revalidate = 86400` | `app/sitemap.ts:13` | correct | n/a | — |

### 14. Indexation suppression

| Finding | File:line | Status | Severity | Note |
|---|---|---|---|---|
| No `middleware.ts` exists in the repo | (search) | n/a | n/a | grep `find -maxdepth 3 -name "middleware*"` returns nothing. So no edge logic that could inject `X-Robots-Tag`. |
| No code emits a global `noindex` meta | (search) | correct | n/a | grep `noindex` returns no matches in `src/` or `app/` (except test design topics text). |
| No code sets `X-Robots-Tag` response header | (search) | correct | n/a | Same. |
| `generatePrivatePageMetadata` returns `robots: { index:false, follow:false }` and is used by protected/admin/owner routes | `src/metadata/generatePrivatePageMetadata.ts:16-21` | correct | n/a | Correct: private pages noindex. |
| Product not-found and User not-found return `index:false` metadata | `src/metadata/generateProductPageMetadata.ts:60-64`, `src/metadata/generateUserPageMetadata.ts:16-20` | correct | low | But the response status remains 200 (see Item 11). |
| Home, catalog, product, user, about, pricing, privacy, terms, free-zone all return `robots: { index:true, follow:true }` (explicitly or inherited from root) | (all metadata generators) | correct | n/a | No accidental noindex on public content. |
| Cloudflare router worker may set `noindex` on stage (per CLAUDE.md mention) | (out of repo) | n/a | medium | Not in `oglasino-web`. Flagged in seams for Mastermind. |

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
- issues.md: no change (findings are advisory; triage will decide which become entries)

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): N/A this session (read-only)
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): observations folded into the findings tables; nothing additional outside the 14 scope items
- Part 6 (translations): N/A this session
- Part 7 (error contract): N/A this session
- Other parts touched: Part 8 (architectural defaults) — SSR/CSR strategy is implicit in Item 2; Part 9 (stack reference) — locale list referenced.

## Known gaps / TODOs

- I could not verify what the backend emits for `BasicMetadata` translation keys (`base.url`, `og.image.url`, `og.image.alt`, `keywords`, etc.) — assumed they're correctly populated, but if any of those are placeholder strings, the page-level OG/canonical/keywords would silently break. Flagged in seams.
- The brief said "four locales" — the routing config has 10. Findings use the 10-locale set; if Mastermind intends a different operational set, the sitemap, hreflang, and canonical numbers change accordingly.
- I deliberately did not read prior session summaries about SEO, `oglasino-docs/issues.md`, `oglasino-docs/decisions.md`, or `oglasino-docs/state.md` per the brief.

## For Mastermind

### Part 4a simplicity evidence (required)
- Added (earned complexity): nothing
- Considered and rejected: nothing
- Simplified or removed: nothing

### Cross-repo seams (the brief asked for these explicitly)

1. **Backend `/public/baseSite/details` and `/public/product/search` define the locale-tuple universe that the sitemap and metadata depend on.** `app/sitemap.ts:30-52` and `src/metadata/generateHomePageMetadata.ts:23` both call backend. If the backend's `BaseSiteDTO.allowedLanguages` list changes (e.g. removes RU from a tenant), the sitemap shape changes silently. Assume web has no knowledge of the canonical locale set apart from what `/public/baseSite/details` returns.

2. **Translation namespace `METADATA` is the source of truth for `base.url`, `og.image.url`, `og.image.alt`, `keywords`, `page.<x>.title`, `page.<x>.description`, `page.<x>.url`, `template.default`, and `page.user.url`.** Per conventions Part 6, those keys live in the backend SQL seed. If a key is missing or rendered as the key string itself (`page.about.url` not resolved), the metadata silently emits the literal key as a URL component or title — a critical SEO regression that won't show up in lint/typecheck. Worth adding a backend assertion that all `page.*.url` and `base.url` keys resolve to non-empty strings per locale.

3. **Cloudflare router worker (per CLAUDE.md) sets a `noindex` header on stage.** Verified there is no `noindex` in `oglasino-web` itself; if the worker is misconfigured for production, the entire site goes dark. Out of repo, but the dependency is real and should be in router QA.

4. **The hardcoded host in `getNormalizedProductUrl`** is `https://www.oglasino.com` (`src/lib/utils/utils.ts:122`), but the sitemap uses `https://oglasino.com` (`app/sitemap.ts:9`). One of these should be picked as the canonical apex. If the router enforces `www` → apex or vice versa via a 301, then internal inconsistency is at most a one-hop redirect. If not, Google will pick a winner inconsistently.

5. **Montenegrin (`me-cnr`) aliasing to SR** — conventions Part 9 says me/cnr aliases to SR. The routing config (`src/i18n/routing.ts:6-15`) treats `me-cnr` as an independent locale; sitemap emits separate URLs; no hreflang exists to indicate the alias. Either the routing should not register `me-cnr` as a primary locale, or hreflang must declare it as `sr-ME` (or similar BCP-47 form). Needs decision.

6. **Product detail JSON-LD `Offer.url` uses `getNormalizedProductUrl(..., true, locale)`** — that puts the locale prefix in the structured-data URL. The Web canonical at `:68` does the same. But the offer URL also passes through the hardcoded `www.oglasino.com` host — same host inconsistency as Item 4.

### Triage suggestions (advisory only — for Mastermind to weigh)

The audit surfaces three structural critical-class issues that affect the bulk of organic discoverability:

- **JSON-LD delivery via `metadata.other['application/ld+json']` instead of `<script type="application/ld+json">`** — affects all 13 metadata generators. Single fix pattern: emit the JSON-LD as an actual `<script>` in each page component (the privacy page demonstrates the intent; only the payload is wrong). Or implement a shared `<JsonLd>` component that returns a script tag and feed it the schema document directly, removing the `other` map entirely.
- **Product cards are not anchors** (`PortalProductCard.tsx:29`) — every home and catalog page presents a wall of product text with no crawlable links to products. Combined with the missing catalog category sitemap, the only way Google reaches product pages is via the product-shard sitemap.
- **No hreflang anywhere** — 10 locales × multiple page types being indexed as independent duplicates is the single biggest discoverability tax. Implementation is `alternates.languages: { ... }` on every page's Metadata.

Each of the above is probably its own feature chat / brief.

A second tier worth flagging together:

- Locale-prefixed paths in `robots.ts` (the `/admin`, `/owner`, `/messages` disallow rules don't actually disallow anything because real URLs are `/{locale}/admin` etc.)
- User profile canonical drops the user ID (`generateUserPageMetadata.ts:27`)
- Product page emits two H1s (`ProductDetails.tsx:69, 81`)
- Home and catalog emit zero H1s
- Privacy page's `<script type="application/ld+json">` stringifies the whole Metadata object
- About page H1 contains no text (only the SVG logo)

Lower-impact items (hardcoded English alts, missing `og:locale`, mixed `www`/apex hosts, product carousel `alt="Missing image"`) round out the picture.
