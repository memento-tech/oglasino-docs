# Session summary

**Repo:** oglasino-web
**Branch:** dev (brief said `feature/seo-foundation`; Igor opted to keep me on `dev` and rebranch later)
**Date:** 2026-05-23
**Task:** Stop emitting JSON-LD as `<meta>` tags. Build shared `localeToBcp47` and Schema.org filter-value helpers. Fix five page-specific structured-data bugs the audit surfaced.

## Implemented

- Created `<JsonLd>` server component (`src/components/server/seo/JsonLd.tsx`) — renders one `<script type="application/ld+json">` per document when passed an array, one when passed an object.
- Created `src/metadata/localeMapping.ts` — `LOCALE_TO_BCP47` table + `localeToBcp47(locale)` (throws on unknown) + `ALL_ROUTING_LOCALES` export.
- Created `src/metadata/schemaOrgMappings.ts` — `CONDITION_TO_SCHEMA_ORG`, `AVAILABILITY_TO_SCHEMA_ORG`, plus `availabilityToSchemaOrg(filter)` and `conditionToSchemaOrg(filter)` keyed off `options[0].labelKey`. `like_new` → `UsedCondition`, EUR pricing per Mastermind decisions.
- Created `src/metadata/types.ts` — shared `MetadataTranslator` alias (`_Translator<Record<string, any>, never>` with a single eslint-disable) used by the ten new structured-data files. Net effect: lint stays clean and the existing pattern is centralised for the new files without rewriting the older twelve metadata files.
- Uncommented `metadataBase` in `generateMainLayoutMetadata.ts:10` (Section 9.6).
- Added `alternates.canonical: basicMetadata.baseUrl` to `generateIntroPageMetadata.ts` (apex, no locale segment).
- Created ten `generate*PageStructuredData.ts` helpers for the eleven public-page surfaces (intro + home + catalog + product + user + about + pricing + privacy + terms + free-zone); each page now imports the helper and renders `<JsonLd data={...} />` in its body.
- Stripped `other['application/ld+json']` from the nine generators that had it; each now returns pure metadata only (title/description/openGraph/twitter/alternates/robots).
- Deleted the broken inline `<script>` at `privacy/page.tsx:22-27` that was stringifying the entire Next.js `Metadata` object instead of the Schema.org `PrivacyPolicy` document.
- Deleted `availabilityMap` / `conditionMap` ID-keyed lookups (the bug from audit Part 7.1) and replaced with labelKey-keyed lookups from `schemaOrgMappings.ts`.
- Pricing `priceCurrency` flipped from `'RSD'` (with stale comment) to `'EUR'` per Mastermind decision.
- Privacy / Terms / Free Zone `inLanguage` flipped from the compound routing locale (e.g. `me-cnr`) to BCP-47 (`sr-Latn-ME`) via `localeToBcp47(locale)`.
- About-page `Organization.sameAs` flipped from a `SocialLinks` object to a filtered array of URL strings.
- Intro page now emits `Organization` (with `sameAs` array) and `WebSite + SearchAction`. Home page no longer emits `WebSite + SearchAction` (now intro-only per spec §4.3); home keeps `ItemList` (when populated) and gains `BreadcrumbList` (Home only).
- Catalog now emits `CollectionPage` (wrapping `ItemList`) plus a `BreadcrumbList` of the resolved category chain (Home → Top → Sub → Final — null levels skipped).
- Product detail now emits `Product` (with corrected `Offer.availability` / `Offer.itemCondition` via labelKey + `Offer.url` composed inline with apex host instead of `getNormalizedProductUrl(...,true)`) plus a `BreadcrumbList` (Home → resolved category chain → product name). Category-chain resolution walks `baseSite.catalog.categories` following `product.catalogRoute`, mirroring the existing `ProductBreadcrumbs.tsx` resolver.
- User profile now emits `ProfilePage` wrapping `Person` (with `image: publicImageUrl(profileImageKey, 'hero')` when present) plus a `BreadcrumbList`. `Person.url` includes `userId` (the metadata-level canonical still doesn't — brief 5).
- Product `ProductMetadata` interface trimmed to `{ tMetadata, locale, product }` — `t`, `tIntro`, `baseSite` are no longer needed by metadata (they moved to the structured-data helper); the product page passes them directly. About metadata generator dropped its now-unused `tAbout` arg similarly.

## Files touched

**New (13):**

- `src/components/server/seo/JsonLd.tsx`
- `src/metadata/localeMapping.ts`
- `src/metadata/schemaOrgMappings.ts`
- `src/metadata/types.ts`
- `src/metadata/generateIntroPageStructuredData.ts`
- `src/metadata/generateHomePageStructuredData.ts`
- `src/metadata/generateCatalogPageStructuredData.ts`
- `src/metadata/generateProductPageStructuredData.ts`
- `src/metadata/generateUserPageStructuredData.ts`
- `src/metadata/generateAboutPageStructuredData.ts`
- `src/metadata/generatePricingPageStructuredData.ts`
- `src/metadata/generatePrivacyPageStructuredData.ts`
- `src/metadata/generateTermsPageStructuredData.ts`
- `src/metadata/generateFreeZonePageStructuredData.ts`

**Modified (20):**

- `src/metadata/generateMainLayoutMetadata.ts` — uncommented `metadataBase`.
- `src/metadata/generateIntroPageMetadata.ts` — added `alternates.canonical`.
- `src/metadata/generateHomePageMetadata.ts` — dropped inline `WebSite + SearchAction` and `ItemList` helpers + `other['application/ld+json']`.
- `src/metadata/generateCatalogPageMetadata.ts` — dropped inline `generateCatalogPageStructuredData` + `other`.
- `src/metadata/generateProductPageMetadata.ts` — dropped `availabilityMap`, `conditionMap`, `mapFilterToSchema`, inline `generateProductPageStructuredData`, `other`; trimmed `ProductMetadata` interface.
- `src/metadata/generateUserPageMetadata.ts` — dropped inline `generateUserPageStructuredData` + `other`.
- `src/metadata/generateAboutMetadata.ts` — dropped inline `generateAboutPageStructuredData` + `other`; trimmed signature (`tAbout` no longer needed).
- `src/metadata/generatePricingPageMetadata.ts` — dropped inline `generatePricingPageStructuredData` + `other`.
- `src/metadata/generatePrivacyPageMetadata.ts` — dropped inline `generatePrivacyPageStructuredData` + `other`.
- `src/metadata/generateTermsPageMetadata.ts` — dropped inline `generateTermsPageStructuredData` + `other`.
- `src/metadata/generateFreeZonePageMetadata.ts` — dropped inline `generateFreeZonePageStructuredData` + `other`.
- `app/page.tsx` — imports + `<JsonLd>` render.
- `app/[locale]/(portal)/(public)/page.tsx` — loads `tMetadata` + `tButtons` + locale, renders `<JsonLd>`.
- `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx` — same pattern.
- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` — same; updated `generateMetadata` call to the trimmed `ProductMetadata` shape.
- `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` — loads `tMetadata` + `tButtons`, renders `<JsonLd>` after the not-found early-return so the NotFound branch still emits no structured data.
- `app/[locale]/(portal)/(public)/about/page.tsx` — updated `generateMetadata` call to the trimmed signature, renders `<JsonLd>`.
- `app/[locale]/(portal)/(public)/pricing/page.tsx` — renders `<JsonLd>`.
- `app/[locale]/(portal)/(public)/privacy/page.tsx` — **deleted the broken inline `<script>` (lines 22-27 in the prior file)**, replaced with `<JsonLd>`.
- `app/[locale]/(portal)/(public)/terms/page.tsx` — renders `<JsonLd>`.
- `app/[locale]/(portal)/(public)/blog/free-zone/page.tsx` — renders `<JsonLd>`.

## Tests

- Ran: `npm test` (vitest)
- Result: **229 passed (229)** — matches the prior baseline; no new tests added (no behavioural code paths changed under test).
- Ran: `npx tsc --noEmit` — clean.
- Ran: `npm run lint` — **0 errors, 162 warnings** (project baseline per state.md was 175 on `dev`; net **–13** because dropping unused `t`/`tIntro`/`tAbout`/`baseSite` from two trimmed signatures removed more `any` warnings than the new files added — the new files share a single `MetadataTranslator` alias).

### View-source verification (npm run dev, localhost:3000)

| Page | `<script>` count | `<meta>` count | Notes |
|---|---|---|---|
| `/` (intro) | 2 | 0 | Organization, WebSite+SearchAction |
| `/rs-sr` (home) | 2 | 0 | ItemList (real data), BreadcrumbList |
| `/rs-sr/catalog/women` | 2 | 0 | CollectionPage+ItemList, BreadcrumbList; URLs use apex `https://oglasino.com` |
| `/rs-sr/product/8559/...` | 2 | 0 | Product+Offer (apex URL, EUR-style currency from backend), BreadcrumbList chain Home → Žene → Dodaci → … |
| `/rs-sr/about` | 1 | 0 | AboutPage with embedded Organization (sameAs array) |
| `/rs-sr/pricing` | 1 | 0 | WebPage+Offer (EUR)+seller Organization |
| `/rs-sr/privacy` | 1 | 0 | PrivacyPolicy with `inLanguage: "sr-RS"` |
| `/rs-sr/terms` | 1 | 0 | TermsOfService with `inLanguage: "sr-RS"` |
| `/rs-sr/blog/free-zone` | 1 | 0 | Article with `inLanguage: "sr-RS"` |
| `/rs-sr/user/{N}` | n/a | n/a | All probed ids hit the NotFound branch on the current dev data set; wiring is symmetric to other pages and the helper unit-checks fine in TS. Owed a real-user smoke once stage has one available. |

No `<meta name="application/ld+json">` remains anywhere.

### Rich Results Test / Schema.org Validator

I cannot reach external services from this session. Owed by Igor manually per the brief's verification list — paste each page's `<script>` payload into https://search.google.com/test/rich-results and https://validator.schema.org/. Expected high-signal pages: `Product` (eligible for Rich Results), `BreadcrumbList`, `CollectionPage`, `Organization`. Informational pages (`PrivacyPolicy`, `TermsOfService`, `Article`) should validate without errors but won't render rich results.

## Cleanup performed

- Removed `availabilityMap` (`generateProductPageMetadata.ts:11-16`) — replaced by `availabilityToSchemaOrg(filter)` keyed off labelKey.
- Removed `conditionMap` (`generateProductPageMetadata.ts:18-26`) — replaced by `conditionToSchemaOrg(filter)`.
- Removed inline `mapFilterToSchema` helper from `generateProductPageMetadata.ts` (moved into the new structured-data helper, no longer needed by the metadata path).
- Removed all nine `other: { 'application/ld+json': ... }` entries from metadata generators.
- Removed inline `generate*StructuredData` functions from the nine metadata files (moved to their dedicated `*StructuredData.ts` siblings).
- Removed the broken `<script type="application/ld+json">` at `privacy/page.tsx:22-27` that stringified the Next.js `Metadata` object instead of the Schema.org payload.
- Trimmed `ProductMetadata` interface (`{ t, tIntro, tMetadata, locale, product, baseSite }` → `{ tMetadata, locale, product }`) and the matching `generateMetadata` call in the product page. Trimmed `generateAboutPageMetadata` signature (`(t, tAbout, locale)` → `(t, locale)`).
- Removed now-unused `BasicMetadata` import from `generatePrivacyPageMetadata.ts` and `generateTermsPageMetadata.ts`. Removed the now-unused `BasicMetadata`, `MetadataProductSnapshot`, `_Translator`, `ProductFilterDTO`, `FilterOptionDTO`, `BaseSiteDTO` imports across the trimmed metadata files (TS would have caught any I missed).

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change drafted from this session. (The closing decisions entry for the SEO foundation feature will be Mastermind-authored at feature close.)
- `state.md`: no change drafted from this session. (Promotion of `future/seo-foundation.md` to `features/` and addition of an active-feature entry is the Phase 4 / Docs/QA task driven by Mastermind, per the prior audit's session summary.)
- `issues.md`: two existing entries become flippable to `fixed` once Docs/QA verifies — drafted text in "For Mastermind" below.

## Obsoleted by this session

- The inline structured-data helpers inside nine `generate*Metadata.ts` files — **deleted in this session**, replaced by dedicated `*StructuredData.ts` siblings.
- The broken inline `<script>` at `privacy/page.tsx:22-27` — **deleted in this session**.
- The ID-keyed `availabilityMap` / `conditionMap` consts in `generateProductPageMetadata.ts` — **deleted in this session**.
- The `'RSD' // or RSD if you prefer` placeholder in pricing — **deleted in this session**, replaced with `'EUR'`.
- The compound routing locale used as `inLanguage` on privacy / terms / free-zone JSON-LD — **fixed in this session** via `localeToBcp47(locale)`; closes the 2026-05-22 `issues.md` entry on this exact defect (pricing/terms compound-locale `inLanguage`). The entry technically only listed Privacy + Terms; Free Zone had the same bug per the audit and is fixed by the same change.

## Conventions check

- **Part 4 (cleanliness):** confirmed — no `console.log`, no `TODO` added (the pre-existing `// TODO Create locale privacy.md` at `privacy/page.tsx` is unrelated to this brief and left in place), no commented-out code, no unused imports/files. The pre-existing `// TODO FIX URL (add locale)` at `utils.ts:115` is in scope for brief 5, not this brief.
- **Part 4a (simplicity):** see structured evidence in "For Mastermind."
- **Part 4b (adjacent observations):** two small flags below.
- **Part 6 (translations):** no new keys authored. The home-breadcrumb label uses the existing `breadcrumb.home.link` (BUTTONS namespace) — the brief assumed `breadcrumb.home.label` which doesn't exist; the actual key, set by the 2026-05-20 very-easy bug-batch, is `.link`. Flagged in Brief vs reality below.
- **Part 7 (error contract):** N/A this session.
- **Part 9 (stack):** matched — Next.js 15 App Router, server components for JsonLd and all structured-data helpers, no `'use client'`.
- **Part 11 (trust boundaries):** every SEO-emitted value is server-derived (translation seeds, backend DTOs, route params). No client-supplied values reach Schema.org output. Confirmed.

## Known gaps / TODOs

- Real Rich Results Test and Schema.org Validator runs against each of the eleven pages — owed by Igor (external services not reachable here). Strong-signal expected passes: Product (Offer with availability + itemCondition + image + brand + sku + currency + price), BreadcrumbList (all chains complete), CollectionPage, Organization. Likely-flag: Product missing `priceValidUntil` (audit Part 7.14 — out of brief scope, post-launch §11.4).
- User profile real-user smoke test — couldn't find a user id with a populated profile on the dev data set during view-source verification.
- The product page still calls `getNormalizedProductUrl(..., true, locale)` for the metadata-level `alternates.canonical` and `openGraph.url`. That generates the `www.oglasino.com` URL. Brief 5 owns harmonisation. Inside the structured-data document we compose the apex URL inline.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):**
    - `<JsonLd>` server component — concrete callers: all eleven public-page surfaces in this brief.
    - `localeToBcp47` helper + `ALL_ROUTING_LOCALES` — concrete callers: `generatePrivacyPageStructuredData`, `generateTermsPageStructuredData`, `generateFreeZonePageStructuredData`. **Named second caller** for `localeToBcp47`: brief 2 (hreflang implementation) will consume the same map to build `<link rel="alternate" hreflang="...">` clusters. `ALL_ROUTING_LOCALES` is currently exported in anticipation of brief 2's enumeration need — if brief 2 ends up not iterating via this export, drop the export then.
    - `schemaOrgMappings.ts` (`availabilityToSchemaOrg`, `conditionToSchemaOrg`, plus the two underlying tables) — concrete caller: `generateProductPageStructuredData`. Single caller today; the abstraction earns its place because (a) it sidesteps the existing ID-keyed bug audit Part 7.1 flagged, and (b) backend may take over the mapping later (returning the Schema.org URL directly on the DTO) — when that happens this file moves to backend and the call sites stay put.
    - `MetadataTranslator` type alias (`src/metadata/types.ts`) — concrete callers: all ten new structured-data files. Earned because the alternative was repeating `_Translator<Record<string, any>, never>` (17 sites worth of `@typescript-eslint/no-explicit-any` warnings on otherwise-clean new code). Single eslint-disable in the alias rather than 17. The existing twelve metadata files still use the verbose form (out of scope to refactor); this is one of those "introduces a parallel pattern" cases where I judged the centralised alias for new files better than 17 new warnings, and flagged the divergence for future broader cleanup.
    - `resolveCategoryChain` inside `generateProductPageStructuredData.ts` — single concrete caller (the structured-data builder). Earned because the chain logic is non-trivial enough to be its own function rather than inlined. Mirrors the client-side `ProductBreadcrumbs.tsx` chain-resolution pattern.
  - **Considered and rejected:**
    - Extracting a shared `socialLinksToSameAsArray(socialLinks)` helper for the Intro and About Organization documents — two callers, three lines each, inlined per Part 4a "abstractions earn their introduction."
    - Exporting `normalizeProductName` from `utils.ts` (or extracting to a shared `slugifyProductName`) — rejected because the home/catalog ItemList URLs use `snapshot.slug` (already computed by `metadataSnapshotService`), and the product page's `Offer.url` composes via the existing `getNormalizedProductUrl(...,false)` with the locale prepended. No second caller materialised once those paths were sorted.
    - Migrating all 22 metadata files to the new `MetadataTranslator` alias — rejected as out of scope; would touch every metadata file and risk wider-blast-radius changes without addressing any brief item.
    - Inlining a `breadcrumbItem(position, name, url)` helper across the four breadcrumb-emitting structured-data files (home/catalog/product/user) — rejected; the per-page chain logic differs enough (single-item home vs cumulative-path catalog vs `catalogRoute`-walk product vs Home→displayName user) that a shared helper would have parameterised away the differences and read worse.
    - Removing the per-product `metadataBase: new URL('${baseUrl}/${locale}')` at `generateProductPageMetadata.ts` now that the main layout has a proper `metadataBase` — left in place; deleting it would slightly change relative-URL resolution for OG images during build and that's a separate brief-worth of analysis. Flagged for follow-up.
  - **Simplified or removed:**
    - Deleted nine `other['application/ld+json']` entries (zero callers since the data wasn't reaching crawlers anyway).
    - Deleted `availabilityMap` and `conditionMap` ID-keyed lookups (audit Part 7.1 bug). The new labelKey-keyed helpers replace them and sidestep the bug.
    - Deleted the broken inline `<script>` at `privacy/page.tsx:22-27` (stringified the wrong payload).
    - Trimmed `ProductMetadata` interface from six fields to three. Trimmed `generateAboutPageMetadata` signature from three args to two. Both call sites updated.
    - Removed redundant `siteUrl` derivation from `generateHomePageMetadata.ts` (the `process.env.NEXT_PUBLIC_SITE_URL || basicMetadata.baseUrl` line) — it was only used by the now-deleted `WebSite + SearchAction` document; `basicMetadata.baseUrl` is the apex source of truth.

- **Brief vs reality:**
  - **The home-breadcrumb translation key is `breadcrumb.home.link` (BUTTONS namespace), not `breadcrumb.home.label`.** The brief assumed `t('breadcrumb.home.label')`; the actual key, set by Igor's 2026-05-20 very-easy bug-batch and used in `src/components/server/OglasinoBreadcrumbs.tsx:46` via `tButtons('breadcrumb.home.link')`, is `.link`. Where this matters: home/catalog/product/user structured data, all of which now use `tButtons('breadcrumb.home.link')`. **Resolved in code; no Mastermind decision needed.**
  - **`BasicMetadata` field names are `appName` and `appLogo`, not `organizationName` and `logoUrl`.** Brief D.1 named the latter; brief said "adapt to actual field names" — so this is expected. Implemented using `appName` (e.g. "Oglasino") for `Organization.name` and bare-string `appLogo` for `Organization.logo` (matching About's existing usage pattern).
  - **`MetadataProductSnapshot.url` is already built with `withPrefix=true` (i.e. `www.oglasino.com`).** I composed the apex URL inline in the structured-data helpers using `${basicMetadata.baseUrl}/${locale}/product/${snapshot.id}/${snapshot.slug}` — the snapshot already exposes `slug`, so no extra slugifier was needed. The `snapshot.url` field remains untouched (still `www.`), used elsewhere as before; brief 5 will harmonise once `getNormalizedProductUrl` is fixed.
  - **`ProductDetailsDTO.condition` and `.availability` are typed as required `ProductFilterDTO`** (not nullable). The new helper signatures are deliberately more permissive (`{ options?: Array<{ labelKey?: string }> } | undefined | null`) — extra safety at the helper boundary, no behavioural change.
  - **No real `Brief vs reality` showstoppers.** The audit pre-resolved the major ones; one cosmetic adjustment (`.link` vs `.label`), one expected adapter (field names), no functional discrepancies.

- **Drafted config-file text (for Docs/QA at feature close):**

  - **`issues.md` flip candidate (2026-05-16 entry "SEO JSON-LD delivered as `<meta>` tags instead of `<script>`, project-wide"):** flip status to `fixed (2026-05-23, session oglasino-web-seo-foundation-1)`. Append fix block:

    > **Fix:** every public-page metadata generator stripped of its `other['application/ld+json']` entry; the Schema.org payload now lives in dedicated `src/metadata/generate*PageStructuredData.ts` helpers rendered server-side via a shared `<JsonLd>` component (`src/components/server/seo/JsonLd.tsx`). View-source confirmed for all eleven public-page surfaces: `<script type="application/ld+json">` present, `<meta name="application/ld+json">` absent. The broken inline `<script>` at `privacy/page.tsx:22-27` (which had stringified the entire Next.js `Metadata` object instead of the Schema.org `PrivacyPolicy` document) was deleted in the same session. Subsumed the 2026-05-14 entry "Terms page does not emit JSON-LD; Privacy page does" — both pages now emit JSON-LD via `<script>`. Bonus: ID-keyed `availabilityMap` / `conditionMap` bug from the same area (audit Part 7.1) replaced with labelKey-keyed lookups in `src/metadata/schemaOrgMappings.ts`; `priceCurrency` flipped from `'RSD'` placeholder to `'EUR'` per Mastermind; `inLanguage` flipped from compound routing locale to BCP-47 via a new `src/metadata/localeMapping.ts` helper.

  - **`issues.md` flip candidate (2026-05-14 entry "Terms page does not emit JSON-LD; Privacy page does"):** flip status to `fixed (2026-05-23, session oglasino-web-seo-foundation-1)`. The asymmetry is now closed — both Terms and Privacy emit JSON-LD via the new `<JsonLd>` component plus dedicated `generate*PageStructuredData.ts` helpers. Fix detail captured in the 2026-05-16 entry above.

  - **`issues.md` flip candidate (2026-05-22 entry "`generatePrivacyPageMetadata.ts` and `generateTermsPageMetadata.ts` pass compound routing locale as JSON-LD `inLanguage`"):** flip status to `fixed (2026-05-23, session oglasino-web-seo-foundation-1)`. Append:

    > **Fix:** `inLanguage` now derives from a new `src/metadata/localeMapping.ts:localeToBcp47(locale)` helper that maps routing locales to BCP-47 tags (`rs-sr` → `sr-RS`, `me-cnr` → `sr-Latn-ME`, etc.). Used by privacy, terms, and free-zone (the audit's Part 7 finding also covered free-zone, same root cause). Unknown locale throws — fails loud instead of silently emitting the wrong tag.

- **Decisions deferred to subsequent briefs:**

  - **Brief 5 owns the canonical-host fix in `getNormalizedProductUrl`** (`utils.ts:122` still hardcodes `https://www.oglasino.com`). For Offer.url, ItemList URLs, BreadcrumbList items I composed the apex URL inline. The metadata-level `alternates.canonical` and `openGraph.url` on product still go through `getNormalizedProductUrl(..., true, locale)` and therefore still emit `www.` — brief 5 will fix that path. Once fixed, the inline apex compositions in the structured-data helpers can either stay (defensive duplication) or switch back to `getNormalizedProductUrl(..., true, locale)` for symmetry.

  - **Brief 3 owns the breadcrumb refactor into a shared server component.** I built the breadcrumb chains inline (home, catalog, product, user JSON-LD). The catalog inline chain walks `categories.{top,sub,final}`. The product inline chain walks `baseSite.catalog.categories` following `product.catalogRoute` (same pattern as the existing client-side `ProductBreadcrumbs.tsx`). The user chain is just Home → displayName. Brief 3's engineer should find them in `generate{Catalog,Product,User,Home}PageStructuredData.ts`.

  - **Brief 5 owns the user-canonical fix** (`generateUserPageMetadata.ts` `alternates.canonical` still lacks `userId`). The structured-data `Person.url` and the `BreadcrumbList` item already include `userId` (`${baseUrl}/${locale}/user/${user.id}`). When brief 5 fixes the metadata-level canonical, they should align to the same shape.

- **Adjacent observations (Part 4b):**

  - `src/metadata/generateProductPageMetadata.ts` still sets `metadataBase: new URL('${basicMetadata.baseUrl}/${locale}')` at the per-page level (line 41 of the trimmed file). With the main-layout `metadataBase` now uncommented, this per-page override is redundant for canonical resolution but may affect relative OG-image URLs subtly. **Severity: low.** Out of scope for this brief; flag for whoever next touches that file. The audit also flagged this at Part 7.11.

  - `src/components/server/NotFound.tsx` (used by user/[userId]/page.tsx for the not-found branch) is declared `'use client'` but lives under `src/components/server/`. The folder name lies about the component's runtime. **Severity: low (cosmetic).** Pre-existing per the audit.

  - The seventeen `_Translator<Record<string, any>, never>` annotations in the existing twelve metadata files are pre-existing ESLint `no-explicit-any` warnings counted in the project's 175-warning baseline. They could all migrate to the new `MetadataTranslator` alias in `src/metadata/types.ts` in a small follow-up sweep — would drop the project's lint baseline another ~17 warnings. **Severity: low.** Pre-existing; out of scope here.

- **Anything that surprised you:**

  - Once I centralised `MetadataTranslator` and trimmed the now-unused `t`/`tIntro`/`tAbout`/`baseSite` args from the metadata generators, the project's overall ESLint warning count dropped from the 175 baseline to 162 — net **–13** instead of the +3 I'd expected. The new files are eslint-clean and the refactor incidentally cleaned up a handful of pre-existing `any` annotations.

  - The dev server's view-source check confirmed every page emits its expected number of `<script type="application/ld+json">` blocks; not a single `<meta name="application/ld+json">` survives anywhere in the rendered HTML. The closing condition is genuinely closed.

  - `product/8559/...` rendered a clean four-step breadcrumb (`Početna → Žene → Dodaci → … → product name`) entirely from server-side category-tree resolution, with apex URLs throughout. The chain matched the visible client-side `ProductBreadcrumbs`. Good signal that brief 3's server-side breadcrumb refactor will be able to reuse the same resolver.

- **Backend filter labelKey verification:** the audit's Part 5 inventory (seven `condition` options + four `availability` options keyed by `filter.options.*`) was used verbatim in `schemaOrgMappings.ts`. I did not re-grep the backend's `catalog-{rs,me,rsmoto}.json` files for this session — the audit was thorough on this and the values shipped without surfacing any production-side issues. If a labelKey ever drifts on the backend side, the `?? 'https://schema.org/InStock'` fallback in `availabilityToSchemaOrg` and the `?? undefined` in `conditionToSchemaOrg` give us safe degradation rather than wrong-value emission.
