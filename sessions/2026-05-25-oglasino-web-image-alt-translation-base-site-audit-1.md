# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-25
**Task:** Read-only audit of base-site-aware translation machinery to determine whether the image-alt-translation feature stays at four keys (language-only) or expands to per-base-site variants.

## Implemented

- Read-only audit. No code changes.
- Traced the full translation resolution path: web frontend → backend API → database schema.
- Answered five sub-task questions about translation machinery, base-site identity, image rendering, intro page status, and about page status.
- Confirmed recommendation: keep the spec at language-only keys.

## Files touched

None (read-only audit).

## Tests

Not applicable (read-only audit).

---

## Sub-task 1 — Translation lookup mechanism

**Verdict: the codebase has NO base-site-aware translation machinery. Translations resolve identically regardless of base site.**

Evidence trail:

1. **Backend schema** — `Translation` entity at `oglasino-backend/.../entity/Translation.java` has columns: `id`, `language_id`, `translation_namespace`, `translation_key`, `translation_value`, `created_at`, `updated_at`. Unique constraint: `(translation_namespace, translation_key, language_id)`. **No `base_site_id` or `base_site` column.**

2. **Backend controller** — `TranslationsController.java:30-38` defines `GET /api/public/translations?namespace={namespace}&lang={lang}`. Takes only `namespace` and `lang` as `@RequestParam`. Does NOT read `X-Base-Site` header. Calls `translationService.getTranslationsForNamespace(namespace, lang)`. The base site is structurally invisible to the translation endpoint.

3. **Web frontend sends `X-Base-Site` but it's inert for translations** — `translationsCache.ts:40` sends `headers: { 'X-Base-Site': tenant }` on every translation fetch. The web caches per `(namespace, lang, tenant)` via `unstable_cache` (`translationsCache.ts:85-92`). However, since the backend ignores the header for this endpoint, the per-tenant cache stores redundant identical copies. Three cache entries for `(METADATA, sr, rs)`, `(METADATA, sr, rsmoto)`, `(METADATA, sr, me)` all hold the same data.

4. **Backend translation seed files** — SQL files at `oglasino-backend/src/main/resources/data/translations/` insert rows as `(id, language_id, namespace, key, value, timestamp)`. No base-site column in any seed file. No translation key uses a base-site suffix (no `intro.image.alt.rs` pattern).

5. **No precedent** — No existing translation key in the entire schema has a base-site dimension. No runtime lookup swaps strings by base site.

**Files that determined the answer:** `oglasino-backend/.../entity/Translation.java` (schema), `oglasino-backend/.../controller/TranslationsController.java` (endpoint), `oglasino-web/src/translations/lib/translationsCache.ts` (fetch mechanism), `oglasino-web/src/i18n/request.ts` (resolution config).

---

## Sub-task 2 — What each base site is

| Base site | Country | What it sells | Languages |
|-----------|---------|---------------|-----------|
| `rs` | Serbia | General marketplace — all product categories | SR, EN, RU |
| `rsmoto` | Serbia | Moto-specific catalog — independent subset of categories | SR, EN, RU |
| `me` | Montenegro | General marketplace — own catalog | SR, EN, RU, CNR |

**Key facts:**

- `rs` and `rsmoto` share a domain but have **independent catalogs** (per `seo-foundation.md` §1.3: "rs and rsmoto are independent catalogs serving the same country — no cross-site duplication concern").
- Each base site has its own `BaseSiteDTO.catalog` tree with its own category hierarchy.
- The catalog config lives in the backend. On the web side, `BaseSiteDTO` is fetched at render time via `getBaseSiteServer()` (`app/actions/getBaseSiteServer.ts`), which calls `GET /public/baseSite/${tenant}`. The `BaseSiteDTO` carries a `.catalog` field with the category tree for that base site.
- The three base sites do NOT share a category catalog. Each has its own.

**Where catalog config lives:** backend — the `BaseSite` entity references a `Catalog` entity, which holds the category hierarchy. Web-side access is through the API response. I did not explore backend catalog entity code beyond what was visible through the web-side `BaseSiteDTO` type (per brief constraint: no exploring backend code beyond translation seeds).

---

## Sub-task 3 — Per-base-site image rendering on intro and about

**Verdict: all four code sites use a single image across all base sites. None is per-base-site.**

| Site | File:Line | Image source | Per-base-site? |
|------|-----------|-------------|----------------|
| 1 | `app/page.tsx:29` | `const INTRO = '/intro.jpg'` (line 12, module-level constant) | **No.** Same static asset for all visitors. |
| 2 | `app/page.tsx:36` | Same `INTRO` constant | **No.** Same image, responsive mobile variant of site 1. |
| 3 | `about/page.tsx:41` | `src="/oglasino-hero2.jpg"` (hardcoded string literal) | **No.** Same static asset for all routing locales. |
| 4 | `generateIntroPageMetadata.ts:53` | `basicMetadata.defaultOgImageUrl` → `t('og.image.url')` via `generalMetadata.ts:21` | **No.** Resolves from `METADATA` namespace translation. Since backend translations have no base-site dimension (Sub-task 1), the same OG image URL is returned for all base sites. |

The intro page (sites 1-2) renders a single `/intro.jpg` at the top of the page, then lists base-site cards below. The image is the intro page's image, not any specific base site's image. **This matches what the code shows:** the image is a module-level constant outside any base-site loop or conditional.

The about page (site 3) renders `/oglasino-hero2.jpg` as the hero image. The path is a hardcoded string literal with no base-site parameterization.

---

## Sub-task 4 — The intro page's status

**Verdict: the intro page is base-site-neutral. Keys stay language-only.**

1. **Rendered identically regardless of base site.** The intro page lives at `app/page.tsx` — outside the `[locale]` route segment. When `request.ts` processes the intro page, `requestLocale` is empty, `hasLocale()` returns false, and the config falls back to `routing.defaultLocale = 'rs-sr'` (`routing.ts:19`). Translations resolve via tenant=`rs`, lang=`sr`. Every visitor sees the same intro page content regardless of which base site they will eventually pick.

2. **`generateIntroPageMetadata.ts` does NOT take a base site argument.** Signature: `generateIntroPageMetadata(tMetadata: MetadataTranslator): Promise<Metadata>`. It receives only the translator function. It produces one set of metadata for the entire intro page. No base-site parameter, no base-site-conditional logic.

3. **SEO model: one canonical intro page.** The intro page's hreflang block lists ALL 10 routing locales as siblings (`generateIntroPageMetadata.ts:24-26`, iterating `ALL_ROUTING_LOCALES`). `x-default` points to the apex `basicMetadata.baseUrl` — i.e., `/` — not to any base-site-scoped URL (`generateIntroPageMetadata.ts:27`). This is a single all-locales cluster, not three per-base-site clusters.

**Consequence for the feature:** `intro.image.alt`, `intro.og.image.alt`, and the proposed `intro.meta.description` key (from audit 1's C2 finding) do NOT need base-site variants. The intro page is genuinely base-site-neutral. These keys are language-only: one value per language, shared across all base sites.

---

## Sub-task 5 — The about page's status

**Verdict: the about page is served under all 10 routing locales with base-site-scoped hreflang clusters, but content is identical across base sites sharing a language. Keys stay language-only.**

1. **Route:** `app/[locale]/(portal)/(public)/about/page.tsx` — inside the `[locale]` segment. Rendered at `/rs-sr/about`, `/rsmoto-sr/about`, `/me-sr/about`, `/rs-en/about`, etc. — all 10 routing locales produce their own about page URL.

2. **Hreflang mode: `'base-site-scoped'`.** `generateAboutPageMetadata.ts:17-19` calls `buildAlternateLanguages(locale, 'base-site-scoped', ...)`. This means each base site has its own hreflang cluster:
   - `rs` cluster: `/rs-sr/about`, `/rs-en/about`, `/rs-ru/about` (x-default = `/rs-sr/about`)
   - `rsmoto` cluster: `/rsmoto-sr/about`, `/rsmoto-en/about`, `/rsmoto-ru/about` (x-default = `/rsmoto-sr/about`)
   - `me` cluster: `/me-sr/about`, `/me-en/about`, `/me-ru/about`, `/me-cnr/about` (x-default = `/me-sr/about`)

3. **Content is NOT differentiated per base site.** The about page renders translations from `ABOUT_PAGE` and `METADATA` namespaces. Since the backend has no base-site column (Sub-task 1), `rs-sr/about` and `rsmoto-sr/about` show identical content — same headings, same mission text, same testimonials, same hero image (`/oglasino-hero2.jpg`). The only difference is the hreflang cluster metadata and the canonical URL.

4. **The hero image is the same.** `src="/oglasino-hero2.jpg"` (line 41) — hardcoded path, no base-site parameter.

**Consequence for the feature:** `about.hero.image.alt` does NOT need base-site variants. Even though the about page has per-base-site URLs and hreflang clusters, the visible content (including the image and its alt text) is the same across all base sites. The alt text is a language-only key: one value per language.

---

## Cleanup performed

None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, no violations introduced
- Part 4a (simplicity): N/A this session (read-only audit)
- Part 4b (adjacent observations): see "For Mastermind" section
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

None.

## For Mastermind

### Part 4a simplicity evidence (required)

- Added (earned complexity): nothing (read-only audit)
- Considered and rejected: nothing
- Simplified or removed: nothing

### Answers to the six questions

**1. Translation machinery verdict.**
The codebase does NOT support base-site-aware translation today. Adding it would require: (a) a new `base_site_id` column on the `oglasino_translation` table, (b) a schema migration, (c) rewriting the backend translation endpoint to accept and filter by base site, (d) modifying the unique constraint from `(namespace, key, language_id)` to `(namespace, key, language_id, base_site_id)`, and (e) seeding per-base-site translation rows. This is a substantial schema + backend change, not a small spec amendment.

**2. What each base site sells.**
- `rs` — Serbia general marketplace (all categories)
- `rsmoto` — Serbia moto-only catalog (independent from `rs`)
- `me` — Montenegro marketplace (own catalog)

Each has its own category catalog. They do not share catalogs.

**3. Per-base-site rendering on intro and about.**
All four code sites use a single image, not per-base-site:
- Sites 1-2: `/intro.jpg` (module constant)
- Site 3: `/oglasino-hero2.jpg` (hardcoded literal)
- Site 4: `t('og.image.url')` (translation, but same across all base sites since no base-site dimension in translations)

**4. Intro page status: base-site-neutral.**
One canonical intro page. Outside the `[locale]` segment. Falls back to `defaultLocale` for translation resolution. All-locales hreflang cluster with x-default at apex `/`. `generateIntroPageMetadata` takes no base site argument.

**5. About page status: base-site-scoped URLs, but content is base-site-identical.**
Served at all 10 routing locales with per-base-site hreflang clusters (`'base-site-scoped'` mode). But content is identical across base sites sharing a language because the backend has no base-site translation dimension. The hero image is the same hardcoded path for all.

**6. Recommendation: keep the spec at language-only keys (option (a)).**

Three reasons:

**(a) No machinery.** The translation system has no base-site dimension. Adding one would be a substantial backend + schema feature, far exceeding the scope of a three-key alt-text translation task.

**(b) No content variation.** All four images are the same across all base sites. There is no per-base-site image to describe differently. The about page's hreflang scoping is a URL-cluster strategy, not a content-differentiation strategy.

**(c) No current need.** Even if the three base sites sell different things (rs = general, rsmoto = moto, me = Montenegro), the intro and about pages don't carry base-site-specific product imagery. The intro page is the locale-selector landing. The about page describes the company, not a base site's catalog.

**If per-base-site alt text is ever needed** (e.g., if the about page gains per-base-site hero images, or if product-listing pages need image alt text that names the base site), the translation schema must be widened first. That is a separate feature. The image-alt-translation feature as spec'd targets four static images that are genuinely base-site-neutral.

The spec should stay at 4 keys × 4 languages = 16 seed rows (the 3 original keys from the spec + the C2 `intro.meta.description` key recommended by audit 1). No per-base-site variants.

### Part 4b adjacent observations

1. **Web-side translation cache stores redundant per-tenant copies.** `translationsCache.ts` caches translations per `(namespace, lang, tenant)`, but the backend returns identical data for all tenants. Three cache entries hold the same data. **Severity: low** — memory/storage cost only, no correctness impact. The redundancy is an artifact of the `X-Base-Site` header being sent uniformly by all API calls (standard HTTP client setup), not because translations actually vary. **I did not fix this because it is out of scope and has no correctness impact.**

2. **About page content duplication across base sites.** `/rs-sr/about` and `/rsmoto-sr/about` render byte-for-byte identical content (same language, same translations, same image). Google may flag these as near-duplicate pages despite them being in separate hreflang clusters. The `base-site-scoped` hreflang mode was a deliberate SEO foundation decision (decisions.md 2026-05-24), so the duplication is accepted. If it causes crawl-budget or ranking problems post-launch, a consolidation (shared about page with `all-locales` hreflang) could be considered. **Severity: low** — SEO question, not a bug. **I did not fix this because it is out of scope and was a deliberate SEO decision.**

3. **Intro page description is still hardcoded Serbian** (confirmed by audit 1, finding C2). `generateIntroPageMetadata.ts:17` — `'Vaša platforma za kupovinu i prodaju'`. Cascades to `<meta name="description">`, `og:description`, and `twitter:description` for every locale. Audit 1 recommended folding this into the image-alt-translation feature as a fourth key. **Severity: medium** — EN/RU/CNR users see Serbian in search results. **I did not fix this because it is out of scope for this audit, but audit 1's recommendation to fold it in stands.**
