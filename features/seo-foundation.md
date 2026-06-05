# SEO Foundation

The pre-launch foundation work that makes Oglasino indexable, rankable, and rich-result-eligible across all ten routing locales on day one of production traffic. Post-launch SEO work (freshness signals, schema expansion, performance, sitemap v2) builds on top of this foundation and is tracked separately on the backlog.

---

## 1. Scope and intent

### 1.1 What this feature is

A foundation pass across `oglasino-web`, `oglasino-backend`, and `oglasino-router` that fixes every critical-class SEO defect surfaced by the 2026-05-19 audits, plus the medium-severity defects that fit in the same engineering envelope. The work targets technical correctness on launch day — not content depth, not backlinks, not authority. Those compound after launch on top of a clean foundation.

### 1.2 What this feature is not

Not a content strategy. Not a backlink campaign. Not a Core Web Vitals optimization (image migration to `next/image`, bundle reduction, INP/LCP work — all post-launch). Not a backend freshness overhaul (`Last-Modified` / `ETag` / `updatedAt` propagation to DTOs — post-launch). Not a sitemap rewrite for category pages with `<lastmod>` — post-launch, depends on the freshness work.

The spec lists those post-launch items in Section 11 so they don't get lost.

### 1.3 Market scope

Three base sites, two countries, ten routing locales:

| Base site | Country | Languages | Routes |
| --- | --- | --- | --- |
| `rs` | Serbia | SR, EN, RU | `/rs-sr/...`, `/rs-en/...`, `/rs-ru/...` |
| `rsmoto` | Serbia (moto-specific catalog) | SR, EN, RU | `/rsmoto-sr/...`, `/rsmoto-en/...`, `/rsmoto-ru/...` |
| `me` | Montenegro | SR, EN, RU, CNR | `/me-sr/...`, `/me-en/...`, `/me-ru/...`, `/me-cnr/...` |

`rs` and `rsmoto` are independent catalogs serving the same country — no cross-site duplication concern. `me/cnr` carries its own translations distinct from `me/sr` (per the 2026-05-19 conventions Part 9 correction also drafted in this work).

### 1.4 Strategy at one glance

Google ranks pages by clarity of signal more than by anything else for new sites. The four signals that matter most pre-launch:

1. **Crawlability** — can Google reach every page that should be indexed, and stay away from pages that shouldn't.
2. **Canonical identity** — for each piece of content, exactly one URL is the canonical, and every variant points at it.
3. **Locale signaling** — `hreflang` clusters tell Google "these ten URLs are language/region variants of the same content; serve the right one to the right user."
4. **Structured data** — `Product`, `Offer`, `BreadcrumbList`, `Organization`, `WebSite` JSON-LD that crawlers actually parse (i.e. delivered as `<script>` tags, not `<meta>` tags).

The foundation work in this spec delivers all four. Everything else (content depth, backlinks, freshness signals, performance) compounds on top.

---

## 2. The eight cross-repo seam resolutions

The audits surfaced eight seams between web, backend, and router. Each has a binding resolution; the engineering work in Sections 4-10 implements them.

### 2.1 Seam 1 — canonical host is `oglasino.com` (apex), not `www`

Web's `getNormalizedProductUrl` hardcodes `https://www.oglasino.com` at `src/lib/utils/utils.ts:122`. Web's sitemap uses `https://oglasino.com` at `app/sitemap.ts:9`. Router emits a 301 `www → apex` at `src/index.ts:46-51`. Resolution: web's canonical host becomes apex. Router's 301 stays as defense in depth.

### 2.2 Seam 2 — bad catalog slugs return 404

Web's catalog page redirects unresolved slugs to `/{locale}` (307) at `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:92-94`. Resolution: web calls `notFound()` instead. The two existing `issues.md` 2026-05-14 catalog-slug entries close on this fix. Tester-visible Known-issue pitfalls cross-referenced in the `category-navigation` and `catalog-page` QA topics flip at the same time.

### 2.3 Seam 3 — product slug stability deferred

Backend has no product slug field, no slug history, no rename redirect path. URLs are `/product/{id}/{name}` where the name is web-derived; backend ignores `productName` on lookup and serves the product by `productId` alone. Resolution: product ID stays the source of truth. The "compare requested slug to canonical, 301 to canonical" upgrade is a future feature on the post-launch backlog (Section 11). Pre-launch, canonical tags tell Google the right URL — abusive or stale URL accumulation is accepted as a known low-priority cost.

### 2.4 Seam 4 — server-side breadcrumbs

Web's `ProductBreadcrumbs` is a client component gated on Zustand store hydration (`src/components/client/product/ProductBreadcrumbs.tsx`); initial HTML has no breadcrumb anchors. Resolution: breadcrumbs become server-rendered. The component uses the `BaseSiteDTO.catalog` tree that web's SSR already fetches, resolves the three category IDs (topCategoryId, subCategoryId, finalCategoryId) to labelKeys, translates them via `next-intl`, and emits `<ol>` of `<Link>` anchors plus `BreadcrumbList` JSON-LD in initial HTML.

### 2.5 Seam 5 — image URL contract stays at web

Backend returns R2 keys (`public/products/{uuid}.{ext}`). Web prepends the CDN base via `publicImageUrl()` to produce absolute URLs for `og:image` and JSON-LD `Product.image`. Resolution: leave as-is. Documented contract: web owns the absolute URL shape; backend owns the key. The fragility (a CDN base change silently breaks every previously-rendered `og:image`) is accepted.

### 2.6 Seam 6 — `free` field semantics confirmed

Backend's `ProductOverviewDTO.free` boolean is derived from `topCategory.freeZone`, not from `price == 0`. Resolution: this is correct. Operator rule: if `topCategory.freeZone` is `true`, the product is free regardless of stored price (price is 0 in practice at that point). Otherwise minimum price is 1. The JSON-LD `Offer.price` for free-zone products is `"0"` with `priceCurrency` set; for non-free-zone products it's the actual price. The latent trap on `free` field naming is closed by this clarification.

### 2.7 Seam 7 — CNR is its own locale; hreflang uses `sr-Latn-ME`

Conventions Part 9 historically stated "Montenegrin (me/cnr) aliases to SR." This is wrong — `me/cnr` carries its own editorial translations. Resolution: conventions Part 9 is corrected (separate `decisions.md` entry and conventions amendment drafted in this work). For SEO: CNR is independently canonical, hreflang tag is `sr-Latn-ME` per Google's BCP-47 + ISO 15924 script-tag mechanism. `cnr` alone is not a valid hreflang code (Google supports ISO 639-1 only for language; `cnr` is ISO 639-3). The HTML `lang` attribute on `me-cnr` routes also becomes `sr-Latn-ME`.

### 2.8 Seam 8 — inactive products return 404 via web `notFound()`

Backend correctly returns 404 from `GET /api/public/product/search?productId={id}` when the product isn't `ACTIVE + APPROVED`. Web swallows the 404 and renders an inline "not found" message with HTTP 200 at `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:74-79`. Resolution: web calls `notFound()` instead. This covers inactive products by side effect — when a product flips to `INACTIVE` or `DELETED`, the next crawl hits 404 and Google deindexes within ~1-2 weeks. The 410 Gone upgrade (faster deindex; requires backend to differentiate "never existed" from "deactivated") is post-launch.

### 2.9 Other audit findings, dispositions

- **"Expired" products** — concept doesn't exist anywhere in code. Dropped.
- **`topImageKey` non-determinism** — backend `Product.imageKeys` is a `Set<String>` with no ordering; `ProductOverviewConverter.findFirst()` may pick different keys across reindexings. Logged to post-launch backlog as a low-severity backend fix (add `@OrderColumn`).
- **`UserInfoDTO` caller-context fields (`iamActive`, `isFollowingCurrent`) varying per request** — flagged for the next SSR-cache discipline audit, not in scope here.
- **Two test controllers under `/api/public/`** (`TestCreateJSON.java`, `NotificationsControllerTest.java`) — security concern, separate Mastermind chat.

---

## 3. The trust-boundary check for this feature

Per conventions Part 11, every spec must state for each user-visible value whether it is trusted from the client, derived server-side, or read from the DB. SEO work emits a lot of metadata; the trust check applies to anything that ends up in canonical URLs, structured data, or hreflang clusters.

For this feature:

- **Canonical URLs** — derived from route segments (path) and the authenticated context. Server-derived. Not trusted from the client.
- **`hreflang` siblings** — derived from `BaseSiteDTO` + `routing.ts` config. Server-derived.
- **JSON-LD `Product` fields** — `name`, `description`, `price`, `priceCurrency`, `image`, `availability` all come from the backend's `ProductDetailsDTO` response. Server-derived. The client cannot inject these.
- **`og:image` URLs** — composed from backend-returned R2 keys + web's `publicImageUrl()`. Server-derived.
- **`BreadcrumbList` JSON-LD** — composed from backend-returned category IDs + the `BaseSiteDTO.catalog` tree (also backend-returned). Server-derived.
- **`Person` JSON-LD on user profiles** — composed from `UserInfoDTO` response. Server-derived.
- **Sitemap URLs** — composed from backend-returned product listings + locale config. Server-derived.

No client-supplied values are used in any SEO-emitted text. No trust boundary violations.

---

## 4. Web — JSON-LD delivery (Critical)

### 4.1 The defect

Every page that emits structured data does so via `metadata.other['application/ld+json']`. Next.js renders the `other` map as `<meta name="application/ld+json" content="...">` tags. Google, Bing, and every schema.org parser only extract JSON-LD from `<script type="application/ld+json">` tags. The result: every Schema.org document authored in code today is invisible to crawlers.

Affected files: thirteen `src/metadata/generate*Metadata.ts` files plus one one-off `<script>` in `privacy/page.tsx` that stringifies the wrong payload (the Next.js Metadata object, not the Schema.org helper output).

### 4.2 The fix

Introduce a shared `<JsonLd>` component:

```tsx
// src/components/server/seo/JsonLd.tsx
type JsonLdProps = { data: Record<string, unknown> | Array<Record<string, unknown>> };

export function JsonLd({ data }: JsonLdProps) {
  return (
    <script
      type="application/ld+json"
      // eslint-disable-next-line react/no-danger
      dangerouslySetInnerHTML={{ __html: JSON.stringify(data) }}
    />
  );
}
```

Extract the Schema.org-producing logic from each `generate*Metadata.ts` file into a sibling `*StructuredData.ts` helper (one export per page). Page components import the helper and render `<JsonLd data={...} />` inside the JSX tree (in the page body, not in `<head>` — Next.js App Router puts scripts in `<head>` automatically when in the metadata API; from inside the page body, they render in `<body>` which is equally valid for JSON-LD).

Remove the `other: { 'application/ld+json': ... }` entry from every metadata generator.

Privacy page's existing `<script>` (`app/[locale]/(portal)/(public)/privacy/page.tsx:21-26`) is deleted and replaced by `<JsonLd data={generatePrivacyPageStructuredData(t, locale)} />`.

### 4.3 Per-page structured data inventory

For each public page type, the structured-data document(s) to emit:

**Intro / apex `/`:**
- `Organization` — name, url, logo, sameAs (from `basicMetadata.socialLinks` — fix the shape: must be array of URL strings, not a record object)
- `WebSite` with `SearchAction`

**Home (`/{locale}`):**
- `WebSite` with `SearchAction` (or skip if the apex carries it — pick one, don't emit on both)
- `ItemList` of newest 10 products (currently authored; keep)
- `BreadcrumbList` with one item (Home)

**Catalog (`/{locale}/catalog/[...slugs]`):**
- `CollectionPage` (newly added) with `mainEntity: ItemList`
- `BreadcrumbList` of the category chain
- The existing `ItemList` is folded into `CollectionPage.mainEntity`

**Product detail (`/{locale}/product/{id}/{name}`):**
- `Product` with `Offer` (existing, fix delivery; `Offer.price` and `Offer.priceCurrency` per Seam 6 rule; `Offer.availability` mapped from filter value to `https://schema.org/InStock` or `https://schema.org/OutOfStock`; `Offer.itemCondition` mapped to `https://schema.org/NewCondition` etc.; `Offer.url` uses canonical product URL with apex host)
- `BreadcrumbList` (newly added — top → sub → final category → product name)
- `aggregateRating` and `Review[]` deferred to post-launch (Section 11 — needs backend review-summary endpoint)

**User profile (`/{locale}/user/{userId}`):**
- `Person` (existing, fix delivery; add `image` from `profileImageKey` via `publicImageUrl`)
- `BreadcrumbList` with one item (Home → User)
- `ProfilePage` wrapping the Person as `mainEntity`

**About:**
- `AboutPage` with `mainEntity: Organization` (existing, fix delivery; fix `sameAs` to be an array)

**Pricing:**
- `WebPage` with `mainEntity: Offer` (existing, fix delivery; resolve the `// or RSD if you prefer` comment — pick currency definitively per market)

**Privacy:**
- Privacy pages do NOT emit JSON-LD structured data. Both pages emit metadata (title, description, canonical, hreflang, og:locale, robots: noindex on non-EN locales) only. Reason: no rich-result eligibility on Google for legal/policy page types; emitting `WebPage + about` (the brief 1b shape) added Schema.org validator surface for zero ranking signal. The earlier `@type: PrivacyPolicy` / `@type: TermsOfService` shape (brief 1) was already invalid per Schema.org.

**Terms:**
- Terms pages do NOT emit JSON-LD structured data. Same rationale as Privacy above — retracted in brief 5g.

**Free zone:**
- `Article` (existing, fix delivery; fix `inLanguage` — currently passes the next-intl locale string like `rs-sr` which isn't a BCP-47 tag; map to `sr-RS` etc.)

### 4.4 Validation

After implementation, each page is verified against Google's Rich Results Test (`https://search.google.com/test/rich-results`) and Schema.org validator. The session summary lists each page's validator status.

---

## 5. Web — Product cards as anchors (Critical)

### 5.1 The defect

Every product card on home and catalog list pages uses `<div onClick={() => router.push(...)}>` instead of `<a href>`. Crawlers cannot follow products from home or catalog pages. Combined with the absence of catalog category URLs from the sitemap (Section 11 post-launch), the only path for Google to reach product pages is via the per-product sitemap shards — losing the natural internal linking that home and catalog pages should provide.

Affected files: `src/components/client/product/PortalProductCard.tsx:27-30`, `src/components/client/product/UniversalProductCard.tsx:38-44`.

### 5.2 The fix

Wrap each card in a next-intl `<Link href>` pointing at the product's canonical URL (computed via `getNormalizedProductUrl` — see Section 7 for the host fix). The `onClick` handler is removed; `<Link>` handles the navigation. Middle-click and Ctrl+click work correctly. Prefetch works correctly. The visible UI behavior is identical.

The same pattern applies to `UserDetails.tsx:173-178` ("See user products" button) — replace `<Button onClick={router.push(...)}>` with a next-intl `<Link>` styled as a button. And to any other component that uses `onClick + router.push` to a public canonical URL.

### 5.3 Definition of done

- Every product card on home and catalog renders an `<a href>` to the product's canonical URL (with apex host).
- Middle-click on a product card opens a new tab with the canonical URL in the address bar.
- View-source on home/catalog shows `<a href="...">` wrappers around each product card.
- "See user products" button is an `<a href>`.
- No `onClick={() => router.push(...)}` patterns remain on public pages that navigate to canonical URLs.

---

## 6. Web — Hreflang implementation (Critical)

### 6.1 The defect

Zero pages emit `<link rel="alternate" hreflang="...">` today. Ten routing locales × 7+ public page types means ~70 page-variants being indexed as independent duplicates. This is the single largest tax on launch-day discoverability.

### 6.2 The mechanism

Per Google's documentation, hreflang values use ISO 639-1 (two-letter) language codes optionally combined with ISO 3166-1 Alpha-2 region codes, with the ISO 15924 script-code escape hatch for cases like Montenegrin.

### 6.3 The mapping

| Route prefix | Hreflang value | Notes |
| --- | --- | --- |
| `/rs-sr/` | `sr-RS` | Serbian, Serbia |
| `/rs-en/` | `en-RS` | English, Serbia |
| `/rs-ru/` | `ru-RS` | Russian, Serbia |
| `/rsmoto-sr/` | `sr-RS` | Serbian, Serbia (motorcycle-scoped — see Section 6.4) |
| `/rsmoto-en/` | `en-RS` | English, Serbia (motorcycle-scoped) |
| `/rsmoto-ru/` | `ru-RS` | Russian, Serbia (motorcycle-scoped) |
| `/me-sr/` | `sr-ME` | Serbian, Montenegro |
| `/me-en/` | `en-ME` | English, Montenegro |
| `/me-ru/` | `ru-ME` | Russian, Montenegro |
| `/me-cnr/` | `sr-Latn-ME` | Montenegrin (Serbo-Croatian written in Latin script, Montenegro) |

`x-default` is set to `/rs-sr/` (the largest market's primary language). Considered alternative: a locale-selector landing page at `/` as `x-default` — the apex `/` page already serves that purpose, so its hreflang is `x-default` rather than a specific locale.

**BCP-47 collision callout:** The ten routing locales do not map to ten distinct BCP-47 tags. Three pairs of rs/rsmoto locales collapse to the same tag: `rs-sr` and `rsmoto-sr` both map to `sr-RS`; `rs-en` and `rsmoto-en` both map to `en-RS`; `rs-ru` and `rsmoto-ru` both map to `ru-RS`. (The me cluster's `me-cnr` is disambiguated from `me-sr` via the ISO 15924 script tag, mapping to `sr-Latn-ME`.) This is a structural property of BCP-47 plus our market layout, not a defect. The implication: a single hreflang map keyed by BCP-47 cannot represent both rs and rsmoto pages as siblings — the rs siblings collapse into the rsmoto slots and vanish. Resolution: scope every hreflang cluster to a single base site. See §6.4.

### 6.4 Per-base-site clustering

Every public-page hreflang cluster is scoped to the current request's base site. rs request → 3-sibling cluster (`rs-sr`/`rs-en`/`rs-ru`); rsmoto request → 3-sibling cluster (`rsmoto-sr`/`rsmoto-en`/`rsmoto-ru`); me request → 4-sibling cluster (`me-sr`/`me-en`/`me-ru`/`me-cnr`). Rule applies uniformly to product, catalog, user, home, about, pricing, free zone, privacy, and terms. `x-default` per cluster: rs cluster → `rs-sr` variant; rsmoto cluster → `rsmoto-sr` variant; me cluster → `me-sr` variant. The cluster's primary locale is always the Serbian-language variant (largest user share per base site). Intro (`/`) is the one exception: cluster covers all ten routing locales (with the BCP-47 collision present), `x-default` → apex `/`.

`rs`, `rsmoto`, and `me` are independent sites with independent catalogs. A product page on `rs` is not a hreflang sibling of any page on `rsmoto` or `me` — they're different content (different products at different URLs), not language variants of the same content.

### 6.5 Implementation

Add `alternates.languages` to every `generate*Metadata.ts` file. Build a shared helper:

```ts
// src/metadata/hreflangHelper.ts
type HreflangMode = 'base-site-scoped' | 'all-locales';

export function buildAlternateLanguages(
  pathBuilder: (route: { tenant: string; lang: string }) => string,
  mode: HreflangMode
): Record<string, string> {
  // For base-site-scoped: emit only routes within the current base-site
  // For all-locales: emit all 10 routes
  // Returns { 'sr-RS': '...', 'en-RS': '...', ..., 'x-default': '...' }
}
```

Each metadata generator calls this helper with a path-builder closure. For static pages, mode is `'all-locales'`. For base-site-scoped pages (product, catalog, user), mode is `'base-site-scoped'` and the helper consumes the current base-site code to filter.

Self-referencing hreflang is required by spec — every URL in a cluster includes a hreflang link pointing to itself. The helper emits this automatically.

### 6.6 HTML `lang` attribute

`app/layout.tsx:36,42` currently sets `lang={htmlLang.split('-')[0]}` which yields `cnr` for `me-cnr` routes. This is changed to emit the full BCP-47 tag for the current locale, matching the hreflang mapping: `sr-RS`, `en-RS`, `ru-RS` for `rs/*`; `sr-RS` for `rsmoto/*`; `sr-ME`, `en-ME`, `ru-ME`, `sr-Latn-ME` for `me/*`.

### 6.7 Validation

Search Console's International Targeting report flags hreflang errors. The session summary notes verification approach (Search Console post-deploy, plus a local script that crawls a sample of pages and asserts hreflang clusters are bidirectional).

---

## 7. Web — Heading hierarchy fixes (Critical)

### 7.1 The defects

Home — zero `<h1>`. Catalog — zero `<h1>`. Product detail — two `<h1>`s (one wraps the price, semantic abuse). About — `<h1>` wraps the SVG logo with no text. User profile — zero `<h1>`. Pricing — `<h1>` duplicates the same translation key with a `<br>` between lines (looks like a leftover from copy-paste).

### 7.2 The fixes (as implemented)

Public pages emit exactly one semantically meaningful `<h1>`:

- **Home and Catalog**: `<h1 class="sr-only">` with translation-keyed text in METADATA namespace (`seo.home.h1`, `seo.catalog.h1.tpl`). Visually hidden from sighted users; remain in DOM for screen readers and crawlers. Catalog h1 interpolates `{categoryName}`.
- **Product**: visible `<h1>` carries product name. Price element demoted from `<h1>` to `<p>` (semantic-only change, zero visual difference).
- **About**: visible `<h1>` carries `tAbout('heading')` text (moved out of `<p>`). SVG icon's prior h1 wrapping removed.
- **User**: visible `<h1>` carries displayName (conditional rendering based on whether the component is rendered on the user page vs the product page seller card).
- **Pricing**: visible `<h1>` carries `hero.title` (green badge). Subtitle demoted to `<p>`.
- **Privacy / Terms** — H1 lives in the external markdown fetched from GitHub.

### 7.3 Translation keys

New keys go to the `METADATA` namespace per conventions Part 6. Names suggested: `page.home.h1.title`, `page.catalog.h1.title.tpl` (template — accepts category name as parameter), `page.about.h1.title`, `page.user.h1.title.tpl` (template — accepts displayName). Translation seeding follows conventions Part 6 Rule 3 (engineer appends to existing SQL file).

---

## 8. Web — Server-side breadcrumbs (Critical, also Seam 4)

### 8.1 The defect

`ProductBreadcrumbs.tsx` is a client component gated on `useBaseSiteStore` hydration. Initial HTML emits nothing. Breadcrumb anchors are absent from the crawler's view of every product and catalog page.

### 8.2 The fix

Move the breadcrumb chain construction to a server component. The server page already has access to:

- The `BaseSiteDTO.catalog` tree (fetched via `getBaseSite(baseSiteCode)` in the page's `async` server function).
- The product's three category IDs (`topCategoryId`, `subCategoryId`, `finalCategoryId`) from `ProductDetailsDTO`.
- The current locale from the route param.

A new `<ProductBreadcrumbs>` server component:

1. Walks the catalog tree to find each of the three category IDs.
2. Collects each category's `labelKey`.
3. Uses next-intl's server-side `getTranslations` to resolve labelKeys to display text.
4. Constructs the breadcrumb chain: Home → Top category → Subcategory → Final category → Product name.
5. Emits `<ol>` with next-intl `<Link href>` for each crumb (except the last, which is the current page).
6. Emits `<JsonLd data={...}>` with the corresponding `BreadcrumbList` document.

Same pattern for catalog pages: server component that constructs the chain from URL slug segments resolved against the catalog tree.

### 8.3 Edge cases

- Bad category IDs (don't resolve in the tree) — render no breadcrumb, log a server-side warning. This shouldn't happen if the catalog cache is fresh, but the failure mode is degraded rendering, not a 500.
- Stale catalog tree (category was renamed in admin since last cache refresh) — same as above. The post-launch backend `updatedAt` work will help freshness; pre-launch the staleness window is limited by the catalog cache TTL.

### 8.4 Removed code

The old client `ProductBreadcrumbs` and its bare-Serbian "Kategorije iz X" button (`OglasinoBreadcrumbs.tsx:81`, logged in `issues.md` 2026-05-17 as a Part 6 translations violation) are deleted in this work. The new server component uses next-intl throughout. The issues.md entry closes.

---

## 9. Web — Small fixes batch

These are small individually, large in aggregate. Land in one engineering session.

### 9.1 Canonical host fix

`src/lib/utils/utils.ts:122` — change hardcoded `https://www.oglasino.com` to `https://oglasino.com` in `getNormalizedProductUrl`. Search-and-replace verification across the repo: no other file should hardcode `www.oglasino.com`.

### 9.2 Bad catalog slug → 404

`app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:92-94` — replace `redirect('/{locale}')` with `notFound()`. Closes `issues.md` 2026-05-14 entries for both catalog-slug failure modes (bad first slug; partial-chain breakdown). Updates the `category-navigation` and `catalog-page` QA topic pitfalls — Known-issue pitfalls flip to "fixed" (Docs/QA work, drafted at session close).

### 9.3 Product not-found → 404

`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:74-79` — replace the inline "not found" `<div>` with `notFound()`. The Next.js `not-found.tsx` route handles the rendering. Same applies to `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` for missing users (currently uses `<NotFound />` component with HTTP 200 — change to `notFound()`).

### 9.4 User profile canonical includes user ID

`src/metadata/generateUserPageMetadata.ts:27` — canonical currently emits `${baseUrl}/${locale}${t('page.user.url')}` (the generic user URL pattern, same for every user). Fix to `${baseUrl}/${locale}${t('page.user.url')}/${userId}`. Every user page now emits a distinct canonical.

### 9.5 Robots disallow uses locale-aware patterns

`app/robots.ts` — the disallow rules currently target `/admin`, `/owner`, `/messages`, etc. but real URLs are `/{locale}/admin`, `/{locale}/owner`, `/{locale}/messages`. None of the existing rules match any real URL. Rewrite the disallow list to use Googlebot-compatible patterns:

```
Disallow: /*/admin
Disallow: /*/admin/
Disallow: /*/owner
Disallow: /*/owner/
Disallow: /*/messages
Disallow: /*/notifications
Disallow: /*/favorites
Disallow: /*/wants
Disallow: /*/test
Disallow: /*/icons
Disallow: /*/design
Disallow: /api/
```

`/design` was missing from the disallow list (web audit Item 4); added. `/api/` stays apex-rooted (no locale prefix). Sitemap reference and `host` directive stay.

### 9.6 `metadataBase` set on root layout

`src/metadata/generateMainLayoutMetadata.ts:10` — the commented-out `metadataBase` line is uncommented and set to `new URL('https://oglasino.com')`. This resolves the Next.js build warning and ensures relative OG/canonical URLs anywhere downstream resolve correctly.

### 9.7 `og:locale` and `og:locale:alternate` on every page

Add to each metadata generator's `openGraph` block: `locale: <current-locale-as-BCP-47>` and `alternateLocale: [<every-other-locale-in-the-current-cluster>]`. The mapping uses the same hreflang helper from Section 6.

### 9.8 Product carousel alt text

`src/components/client/ProductImageCarusel.tsx:65` — hardcoded `alt="Missing image"` on every product image. Replace with `alt={productName}` from the carousel's existing props (which already receive the product name for other purposes).

### 9.9 Hardcoded English alt text

About page hero (`about/page.tsx:36`) and intro page images (`app/page.tsx:25,33`) carry hardcoded English alts. Replace with translation keys. New keys go to the `METADATA` namespace.

### 9.10 Hardcoded Serbian tooltips

`ProductDetails.tsx:72`, `NumberOfViews.tsx:16` — Serbian literals. Replace with translation keys. Closes the existing `issues.md` 2026-05-14 entry "Hardcoded Serbian tooltips on the product detail page."

### 9.11 `markdown.fild.load` typo

`MarkdownViewer.tsx:18,22` — translation key typo (`fild` should be `file`). Rename key in `ERRORS` namespace (per conventions Part 6 Rule 1: anything that would have gone to `VALIDATION` goes to `ERRORS` now). Update the four locale SQL seeds (EN, SR, RU, CNR). Closes existing `issues.md` 2026-05-14 entry.

### 9.12 Pricing hero duplicate line

`pricing/page.tsx:32-34` — line 1 and line 2 both render `hero.subtitle.line1`. Fix to `hero.subtitle.line2` on the second. Closes existing `issues.md` 2026-05-14 entry. (Folded into the heading-hierarchy fix from Section 7.)

### 9.13 Privacy page double-emit cleanup

`app/[locale]/(portal)/(public)/privacy/page.tsx:21-26` — the broken inline `<script type="application/ld+json">` that stringifies the Metadata object is deleted. The new `<JsonLd>` component from Section 4 handles the structured data correctly. Closes the existing `issues.md` 2026-05-16 entry "SEO JSON-LD delivered as `<meta>` tags instead of `<script>`, project-wide."

### 9.14 Intro page canonical and structured data

`src/metadata/generateIntroPageMetadata.ts:14-39` — add `alternates.canonical` and `Organization` + `WebSite` JSON-LD via Section 4's pattern.

### 9.15 Abandon localized URL paths

Localized URL paths were considered but abandoned during brief 5d. On-disk English paths (`/about`, `/pricing`, `/terms`, `/privacy`, `/blog/free-zone`, `/user/{userId}`, `/catalog/{slugs}`) are the canonical URLs across all locales. The `page.*.url` translation seed family (7 keys × 4 locales) was deleted from backend seed data. Post-launch reintroduction via next-intl `pathnames` config would be a separate feature; 301 redirects from English paths can be added then to preserve existing canonical URLs.

### 9.16 Catalog SEO description

Catalog pages emit `<p class="sr-only">` SEO description paragraph below the h1. Six template keys seeded in METADATA namespace (`seo.catalog.desc.{rs,rsmoto,me}.{with.parent,no.parent}.tpl`). Per-base-site selection (rs/rsmoto/me) + parent-presence branching (with-parent uses `{categoryName}` + `{parentCategoryName}`; no-parent uses `{categoryName}` only). Visually hidden from sighted users; remain in DOM for screen readers and crawlers.

### 9.17 Canonical host apex confirmation

Canonical URLs emit apex `https://oglasino.com` host across the entire site. `getNormalizedProductUrl(...)` and all metadata generators emit apex; the `www.oglasino.com` host artifact in the previous `withPrefix=true` branch is removed. The router worker continues to emit a `www → apex` 301; this is now aligned with the canonical signal rather than contradicting it.

### 9.18 User canonical with userId

User page `alternates.canonical` and `openGraph.url` include `/${userId}` segment. Previously the canonical collapsed all N user pages onto the same URL. `Person.url` in JSON-LD already included userId (no fix needed there).

### 9.19 robots.txt locale-aware patterns

`robots.txt` uses locale-aware glob patterns: `/*/admin`, `/*/admin/`, `/*/owner`, `/*/owner/`, `/*/messages`, `/*/notifications`, `/*/favorites`, `/*/wants`, `/*/test`, `/*/icons`, `/*/design`, plus `/api/` apex-rooted. Previous bare-path patterns matched zero real URLs because all routes are locale-prefixed. `/design` added (was missing).

---

## 10. Backend and router — what changes

Backend and router are mostly untouched in this feature. The audits surfaced no critical backend defect that blocks launch indexability — backend already returns 404 correctly, already filters inactive products from public search, already returns `ProductDetailsDTO` with the data web needs to render structured data. The freshness work (Last-Modified / ETag / updatedAt propagation) is real and important but post-launch.

Router is also untouched. The audit found one production gap (admin/owner paths have no edge-level `X-Robots-Tag: noindex`) and a few small inconsistencies. Resolution: defense-in-depth `X-Robots-Tag` at the edge is deferred to post-launch. Web's `robots.txt` fix (Section 9.5) plus per-page `robots: { index: false, follow: false }` metadata on admin and owner routes (already in place via `generatePrivatePageMetadata`) provides single-layer protection. If web's protection lapses, the edge today has no fallback — but that's a defense-in-depth concern, not a launch blocker.

### 10.1 Backend — single change

One small backend deliverable in the pre-launch window: when web needs to render JSON-LD `Offer.availability` and `Offer.itemCondition` as Schema.org enums (`InStock`, `NewCondition`, etc.), the current backend response carries filter IDs only. Two options:

- Web maps the filter IDs to Schema.org enum strings client-side (table lookup in code). Cost: web carries a static mapping table.
- Backend adds a labelKey-style mapping for SEO consumers. Cost: backend work + new fields on DTOs.

**Recommend: web-side mapping table.** Backend stays untouched. Web maintains a small `filterValueToSchemaOrg` table in `src/metadata/schemaOrgMappings.ts`. Cost of being wrong: low — if backend later wants to be authoritative, the mapping moves to backend without changing web's render pattern.

---

## 11. Post-launch backlog

These are the audit findings that are real but don't fit the pre-launch envelope. Each becomes its own Mastermind chat post-launch.

### 11.1 Backend freshness signals (Backend feature)

- `BaseEntity.updatedAt` propagates to `ProductDocument.updatedAt` (ES indexer change).
- `ProductDetailsDTO`, `ProductOverviewDTO`, and category DTOs gain `updatedAt` field.
- Public product and category endpoints emit `Last-Modified` and `ETag` headers.
- Conditional `If-Modified-Since` and `If-None-Match` requests return 304 when appropriate.

Web sitemap consumes `updatedAt` after this lands.

Estimated effort: 1.5–2 weeks backend.

### 11.2 Sitemap v2 (Web feature, depends on 11.1)

- Catalog category pages added to sitemap.
- `<lastmod>` per URL using backend's `updatedAt`.
- `<xhtml:link rel="alternate">` hreflang siblings per URL (the in-XML form of hreflang).
- `changeFrequency` matched to actual edit cadence per entity type.

Estimated effort: 3-4 days web (after 11.1).

### 11.3 410 Gone semantics (Backend + web)

Backend distinguishes "never existed" from "was deactivated/deleted/banned." Product fetch for a deactivated product returns 410 Gone, not 404. Google deindexes 410s faster than 404s. Web handles 410 by rendering `notFound()` UI but preserves the 410 status to the client.

Estimated effort: 1 week backend + 2 days web.

### 11.4 Schema.org coverage expansion (Web feature, possibly backend)

- `aggregateRating` on `Product` — requires a backend review-summary endpoint returning `count`, `value`, etc. for a `productId`. Today web would need to walk paginated reviews to aggregate.
- Per-review `Review` JSON-LD on product pages (top N reviews).
- `Person.image` on user profiles.
- `LocalBusiness` schema for power-seller user profiles (verified sellers, business accounts).
- `FAQPage` on `/about` if the about page grows an FAQ section.

Estimated effort: 1 week web + 3 days backend.

### 11.5 Product slug 301 redirect (Web feature)

Web detects that the requested `productName` slug differs from the canonical slug derived from current product name. Returns 301 to canonical URL. Eliminates duplicate-URL accumulation. Per Seam 3 — operator chose to defer.

Estimated effort: 2-3 days web.

### 11.6 Image migration to `next/image` (Web feature)

Every raw `<img>` on public pages becomes `next/image`. Automatic AVIF/WebP, responsive sizes, `priority`/`fetchpriority` on hero images, `loading="lazy"` everywhere else, explicit width/height to eliminate CLS. Affects: intro page, product carousel, product cards, user profile, loading screen, 404 page, every dialog.

Estimated effort: 1 week web.

### 11.7 Performance pass (Web feature, possibly multiple)

- Bundle reduction on public pages (the heavy `app/[locale]/layout.tsx` initialization tree).
- LCP optimization (priority hints on product carousel main image, font subsetting).
- INP optimization (defer non-critical client init, route-based code splitting).
- Core Web Vitals baseline establishment + ongoing monitoring.

Estimated effort: 2-3 weeks web, ongoing.

### 11.8 Catalog filter searchParam strategy (Web feature)

The catalog accepts arbitrary filter searchParams (price ranges, region, category, etc.). Today Google would explore each combination as a separate URL — crawl budget exhaustion plus duplicate-content variants. Canonical tags help but don't prevent crawl waste. Options:

- Aggressive robots disallow `Disallow: /*\?` (with allow exceptions).
- Self-canonical on the params-less URL (already partially in place via canonical excluding searchParams).
- Render search-result-style pages as `noindex, follow` when filters are applied.

Estimated effort: 3-4 days web.

### 11.9 Edge-level `X-Robots-Tag` for admin and owner (Router feature)

Defense-in-depth header at the edge for `/*/admin/*` and `/*/owner/*` paths. The web layer remains the primary protection; this is belt-and-suspenders.

Estimated effort: 1-2 days router.

### 11.10 Stage maintenance noindex hardening (Router feature)

When stage is in maintenance, the 503 response from `MAINTENANCE_ORIGIN` doesn't carry `X-Robots-Tag: noindex`. Add it for completeness.

Estimated effort: 0.5 day router.

### 11.11 `topImageKey` determinism (Backend feature)

`Product.imageKeys` becomes ordered (add `@OrderColumn` to the join table). `ProductOverviewConverter` picks the first by stable order. Consistent `og:image` across reindexings.

Estimated effort: 1 day backend (plus migration).

### 11.12 SSR cache discipline for caller-context fields (Backend or web)

`UserInfoDTO.iamActive` and `isFollowingCurrent` vary per request. For SSR caches keyed on `userId`, this leaks one user's view of another. Either drop caller-context fields from the SSR-cached fetch, or use a separate authenticated fetch on the client. Audit needed.

Estimated effort: 1 week (mostly audit).

### 11.13 Test controllers exposed under `/api/public/`

`controller/test/TestCreateJSON.java` and `controller/test/NotificationsControllerTest.java` carry `@RequestMapping("/api/public/...")`. Either gate behind a profile or move out of `/public`. Security concern, separate Mastermind chat.

### 11.14 Deep linking — Universal Links (iOS) and App Links (Android)

Web acquires users from search; apps retain them. The handoff between the two is deep linking. Without it, every search click lands on web and stays there, even when the user has the app installed.

> **Superseded 2026-06-04 — now its own feature.** Universal/App Links shipped (code-complete) as [deep-linking.md](deep-linking.md) (router + expo), with decisions that override the sketch below: the two well-known files are **served directly by the router worker, not the web origin** (ownership shift web→router — see [decisions.md](../decisions.md) 2026-06-04), and the locale segment is **strict-validated and used to switch base-site + resolve filters**, not discarded. The bullets below are kept as the original SEO-context sketch; defer to `deep-linking.md` on any disagreement.

Mechanism:

- **Server emits two well-known files.** The **router worker** serves `/.well-known/apple-app-site-association` (no extension, `application/json` content-type) and `/.well-known/assetlinks.json` directly, tier-correct per env. Both map web URL patterns to app bundle/package identifiers. (Originally sketched as web-served; moved to the router 2026-06-04 — see `deep-linking.md` §4.1.)
- **iOS app registers Associated Domains** entitlement with `applinks:oglasino.com`. App-side handler routes incoming URLs to the matching screen in the app.
- **Android app registers intent filters** with `autoVerify="true"` for `https://oglasino.com/*` patterns. Play Store verifies against `assetlinks.json` at install time.
- **Web emits smart app banner metadata.** iOS Safari renders the `apple-itunes-app` meta tag automatically when present. Android's equivalent is a custom banner component on mobile-web pages.
- **JSON-LD `potentialAction`** added to product, user, and catalog page schemas, with `target` pointing at the app deep link URL. Google Search may then show "Open in app" affordance on SERP entries.
- **Fallback path matters.** When the app isn't installed, the universal/app link falls back to the web URL gracefully. When the app is installed but the user wants web, browser handling for long-press or "Open in browser" must work.

Cross-repo work:

- `oglasino-web` — **deferred consumer surfaces only** (no longer serves the well-known files — that moved to the router): smart app banner on mobile, `potentialAction` on JSON-LD, "Open in app" CTAs. All deferred until the app is published (see `deep-linking.md` §5 and [issues.md](../issues.md) 2026-06-04).
- `oglasino-expo` — registers Associated Domains (iOS) and intent filters (Android), implements URL → screen routing for product/user/catalog/home, handles cold-start vs warm-start deep links. **(Shipped — see `deep-linking.md` §4.2–4.4.)**
- `oglasino-router` — **serves** `/.well-known/apple-app-site-association` and `/.well-known/assetlinks.json` directly (inline, `application/json`, 200, short-circuited before the maintenance gate / origin forward / KV reads), tier-correct per env. It no longer merely "stays out of the way" for these two files — web serves nothing under `/.well-known/` for them. Ownership of these two paths moved web→router on 2026-06-04 (see `deep-linking.md` §4.1 and [decisions.md](../decisions.md)). Any other `/.well-known/*` paths (none currently in use) remain origin-forwarded as before.
- `oglasino-backend` — no changes required.

Operator prerequisites: Apple Developer Team ID, app bundle identifier, Android package name and SHA-256 cert fingerprints (debug + release).

Estimated effort: 1-2 weeks across three repos, plus app review cycle for Universal Links activation. Cannot start until both apps are submitted/approved with the corresponding entitlements.

### 11.15 ASO — App Store Optimization

App Store and Play Store have their own search algorithms, completely independent of Google web search. App title, subtitle, keywords field (iOS) / short description (Android), full description, screenshots, app preview videos, ratings, install velocity, and retention metrics all influence rank in store search.

Not engineering work — operator task. Worth listing here so the SEO conversation captures the full acquisition surface:

- **Listing copy** in three languages (SR, EN, RU) for both stores. CNR optional — App Store and Play Store don't list `cnr` as a supported language; either alias to SR or skip. Localized screenshots and preview videos per locale.
- **Keyword research** in each language. iOS keyword field is 100 chars; choose carefully. Android uses full description for keyword extraction.
- **Screenshots strategy.** First three screenshots are visible without scrolling — those carry the conversion weight.
- **Ratings strategy.** In-app review prompts at moments of high satisfaction (successful sale, positive message exchange). Avoid prompting on errors or bad UX.
- **Install velocity.** Coordinated launch push (PR, social, paid spike on launch week) accelerates store-rank lift.
- **Update cadence.** Stores favor actively maintained apps. Monthly updates with visible "What's new" entries help.
- **Cross-promotion from web.** Every web product page on mobile shows a smart app banner (per 11.14). Every web user that becomes an app user closes a deep-linking loop.

Ongoing operator effort, not bounded. Worth establishing measurement (App Store Connect Analytics, Play Console acquisition reports) and reviewing quarterly.

### 11.16 Web SEO is the user-acquisition channel; apps are the engagement channel

A strategic framing, not a feature — captured here so the post-launch roadmap doesn't lose sight of it.

Apps don't get acquired from Google web search. Web pages do. Once a user is acquired via web (search → click → web page → "open in app" or install banner → app), they live in the app for ongoing engagement. The two surfaces reinforce each other; neither replaces the other.

Implications already reflected in the backlog:

- 11.6 (image migration) and 11.7 (performance pass) aren't optional polish — they protect the mobile-web landing experience that decides whether search-acquired users bounce before installing the app.
- 11.4 (schema expansion: `aggregateRating`, `Review`) makes web product pages eligible for rich-result enhancements that lift CTR from search, which is the top of the acquisition funnel.
- 11.14 (deep linking) closes the loop by handing search-acquired sessions to the app when present.
- 11.15 (ASO) is the secondary acquisition channel — store-search install — independent of but additive to web SEO.

Cost of treating web as secondary: zero new user acquisition from organic search. Paid acquisition for classifieds CAC is high. The competitive set (Kupujemprodajem, OLX, KP) all run web-SEO-primary economics. Walking away from that channel concedes the entire organic acquisition surface to them.

This framing is preserved in the spec so future engineering decisions (e.g. deferring web mobile performance work in favor of app feature work) are made with the trade-off visible.

---

## 12. Trust-boundary check per Section 3 (repeated for engineer sessions)

Every value used in canonical URLs, hreflang siblings, structured data, sitemap entries, or robots rules is server-derived. No client-supplied values reach the rendered HTML's SEO-significant surfaces. No trust boundary violations introduced by this feature.

---

## 13. Platform adoption

Per the conventions Part 10 lifecycle: mobile (`oglasino-expo`) is not in scope for this feature. Mobile is not a search-indexable surface. When mobile adopts other features per the Expo backlog (Product Validation, Image Pipeline, Backend Calls Reduction), SEO is unaffected.

Web is the only platform for SEO foundation work.

---

## 14. Engineering brief order

When this feature enters Phase 5 (engineering execution), the brief order is:

1. **Web — JSON-LD delivery** (Section 4). Shared `<JsonLd>` component + per-page structured-data helpers. Each page validated against Google Rich Results Test.
2. **Web — Hreflang implementation** (Section 6). Shared `hreflangHelper.ts` + per-page `alternates.languages` + `<html lang>` fix.
3. **Web — Server-side breadcrumbs + heading hierarchy** (Sections 7 + 8). Combined because both touch the same page templates.
4. **Web — Product cards as anchors** (Section 5). Touches `PortalProductCard`, `UniversalProductCard`, `UserDetails`.
5. **Web — Small fixes batch** (Section 9). All sub-items in one session.

Mobile catch-up sessions for other features (Product Validation, Image Pipeline, Backend Calls Reduction) can run in parallel with these — they don't touch the same files.

Estimated total: 3-4 weeks of engineer time. The 4-week pre-launch window is tight but achievable if briefs run cleanly.

---

## 15. Validation strategy

After each engineering brief lands, the session summary lists:

- For JSON-LD work: Google Rich Results Test pass/fail per page.
- For hreflang work: Search Console International Targeting report scan (post-deploy verification, since the report depends on Google having crawled the deployed pages); plus a local script that asserts bidirectional cluster integrity from the rendered HTML.
- For canonical fixes: view-source check that every public page emits exactly one `<link rel="canonical">` with the correct apex host.
- For robots: a curl-based check against the deployed `robots.txt` confirming every locale-prefixed admin/owner path is disallowed.
- For breadcrumb work: view-source check that `<ol>` of breadcrumb anchors and `BreadcrumbList` JSON-LD both appear in initial HTML (not hydrated client-side).
- For heading hierarchy: visual audit (one H1 per page, semantic heading order).
- For product-cards-as-anchors: view-source check + middle-click manual test on home and catalog.

Final validation before launch: third-party SEO crawler (Screaming Frog, Sitebulb, or equivalent) runs against the deployed staging or pre-launch production environment. Output stored in `oglasino-docs/legal/` is the wrong place — better placed in a new `oglasino-docs/seo/` folder if persistent. Or kept transient (the validation is done once and the spec is the durable record).

---

## 16. Closure criteria

This feature is shipped when:

1. All 6 engineering briefs (Sections 4 through 9, consolidated per Section 14) are merged.
2. The validation steps in Section 15 pass.
3. The post-launch backlog (Section 11) entries are in `state.md`.
4. The conventions Part 9 correction is applied (CNR is a distinct locale).
5. The closing `decisions.md` entry is applied (this triage and the eight seam resolutions).
6. The four `issues.md` entries that close on Seam 2, 8, and Section 9 small fixes are flipped to `fixed`.
7. The two QA-topic pitfalls (`category-navigation`, `catalog-page` Known-issue pitfalls) that cross-reference the catalog-slug bugs are flipped.

The session-closure gate from conventions Part 3 applies — all Docs/QA drafts produced in this Mastermind chat must be applied before the chat closes.

---

## 17. Assumptions and open items

- Assumes the conventions Part 9 correction (CNR is independently translated, not aliased to SR) is applied by Docs/QA before any engineering brief runs.
- Assumes `me-cnr` content is in fact distinct from `me-sr` content at the translation level. If most `me-cnr` rows in the translations table are identical strings to `me-sr` rows, Google's duplicate detection may collapse the locales regardless of correct hreflang. This is a content question, not a technical one — the technical signals are correct either way.
- Assumes `rsmoto` and `rs` carry different products (separate catalogs). Operator confirmed in 2026-05-19 intake.
- Assumes backend's `BaseSiteDTO.catalog` tree is reliably fresh in web's SSR cache. The 2026-05-17 cache decisions removed TTL from `redisBaseSites` and `redisBaseSiteOverviews`; warmup-gated readiness ensures the tree is populated at boot. If admin-side category edits happen, the operator runs `CacheAdminController` evict + refresh before public traffic sees the change.
- Assumes the eventual third-party SEO crawl (Section 15) runs against either staging or a pre-launch production environment with the foundation work merged. The router emits stage `noindex` automatically; a deliberate production crawl pre-launch is also fine since the foundation is correct on both.
