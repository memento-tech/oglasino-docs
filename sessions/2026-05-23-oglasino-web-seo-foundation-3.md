# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-23
**Task:** Emit hreflang clusters on every public-page metadata generator; fix `<html lang>` truncation in `app/layout.tsx`; emit `og:locale` and `og:locale:alternate` on every public-page generator.

## Implemented

- **Part A — `src/metadata/localeMapping.ts`:** added `LOCALE_TO_BASE_SITE` (private), `localesForSameBaseSite(locale)`, `HreflangMode` type, `buildAlternateLanguages(currentLocale, mode, pathBuilder)`, `bcp47ToOgLocale(bcp47)`, `buildOgLocales(currentLocale, mode)`. The existing `LOCALE_TO_BCP47` table and `ALL_ROUTING_LOCALES` export are untouched.
- **Part B — every public-page metadata generator now emits `alternates.languages` + `openGraph.locale` + `openGraph.alternateLocale`** (intro, home, catalog, product, user, about, pricing, free-zone, privacy, terms). `generateMainLayoutMetadata.ts` left alone per spec.
- **Part C — `<html lang>` fix in `app/layout.tsx`:** `htmlLang.split('-')[0]` replaced with `localeToBcp47(locale)` (falls back to `'sr-RS'` on the apex `/` route where the routing locale isn't yet resolved). `getTenantLocale` import removed.
- **Part F — Privacy + Terms `robots: noindex` for non-EN locales:** conditional block added per spec — EN locales (`rs-en`, `rsmoto-en`, `me-en`) emit normal index/follow; the other six emit `{ index: false, follow: true }`. Cluster integrity preserved (all ten siblings listed regardless).
- **Intro special case** per spec §B.1: intro composes its own `alternates.languages` map (not via the helper) so `x-default` points to the apex selector `https://oglasino.com` rather than `/rs-sr`. `og:locale` defaults to `sr_RS` per the brief's "include it, defaulted to sr_RS" decision; the apex itself is locale-agnostic.

## Files touched

- `src/metadata/localeMapping.ts` (+88 / -0) — extended with cluster helpers.
- `src/metadata/generateIntroPageMetadata.ts` (+27 / -2) — full rewrite of the languages composition (special-case x-default) + og:locale block.
- `src/metadata/generateHomePageMetadata.ts` (+9 / -1) — all-locales cluster.
- `src/metadata/generateCatalogPageMetadata.ts` (+11 / -2) — base-site-scoped cluster, slug path extracted to a `slugPath` const so the sibling URL builder reuses it.
- `src/metadata/generateProductPageMetadata.ts` (+12 / -1) — base-site-scoped cluster; sibling URLs composed via `getNormalizedProductUrl(id, name, false)` (the no-prefix branch) prepended with apex `${baseUrl}/${locale}` so the cluster sidesteps the `www.` artifact in the `withPrefix=true` branch (brief 5 fix).
- `src/metadata/generateUserPageMetadata.ts` (+12 / -1) — base-site-scoped cluster; sibling URLs include `${user.id}` (canonical still doesn't — brief 5).
- `src/metadata/generateAboutMetadata.ts` (+10 / -2) — all-locales cluster.
- `src/metadata/generatePricingPageMetadata.ts` (+11 / -2) — all-locales cluster.
- `src/metadata/generateFreeZonePageMetadata.ts` (+11 / -2) — all-locales cluster.
- `src/metadata/generatePrivacyPageMetadata.ts` (full rewrite, +27 / -4) — all-locales cluster + conditional `robots: noindex` for non-EN locales.
- `src/metadata/generateTermsPageMetadata.ts` (full rewrite, +27 / -4) — same.
- `app/layout.tsx` (+3 / -2) — `<html lang>` derivation switched to `localeToBcp47(locale)` with `'sr-RS'` fallback for the apex; dead `getTenantLocale` import removed.

**No new files. No deletions.**

## Tests

- Ran: `npx tsc --noEmit` — **clean**.
- Ran: `npm run lint` — **0 errors, 162 warnings** (unchanged from the brief-1b / brief-2 baseline).
- Ran: `npm test` (vitest) — **229 passed (229)**.
- Ran: `npm run dev` + manual `curl` view-source on representative surfaces. See verification table below.

### View-source verification

| URL | hreflang block | `<html lang>` | og:locale block | Notes |
|---|---|---|---|---|
| `/` (intro) | **only 7 entries + x-default** (BCP-47 collision drops the 3 rs URLs — see Brief vs reality) | `sr-RS` (apex fallback) ✓ | 9 og:locale:alternates with 3 duplicate `sr_RS`/`en_RS`/`ru_RS` ✓ | x-default → apex (correct per spec §B.1) |
| `/rs-sr/about` (all-locales) | **only 7 entries + x-default**, rsmoto URLs win the 3 rs-* BCP-47 slots | `sr-RS` ✓ | 9 alternates, 3 duplicates ✓ | x-default → `/rs-sr/o-nama` ✓ |
| `/me-cnr` (home, all-locales) | same 7-entry collision; `<link hrefLang="sr-Latn-ME" href="/me-cnr">` correctly present | **`sr-Latn-ME` ✓** (was truncating to `cnr`) | og:locale = `sr_Latn_ME` ✓ | x-default → `/rs-sr` ✓ |
| `/rs-sr/product/8573/...` (base-site-scoped) | **3 entries** (`/rs-sr`, `/rs-en`, `/rs-ru`) **+ x-default** — clean, no collision within the rs cluster ✓ | `sr-RS` ✓ | 2 alternates (`en_RS`, `ru_RS`) ✓ | |
| `/me-cnr/product/8573/...` | cross-base-site `redirect()` to `/rs-sr/product/8573/...` per existing behaviour (product 8573 belongs to rs) | n/a (redirected) | n/a | Out of scope; confirms redirect path still works. |
| `/rs-sr/privacy` | 7+x-default cluster + **`<meta name="robots" content="noindex, follow">`** ✓ | `sr-RS` ✓ | 9 alternates ✓ | Privacy noindex (non-EN locale) flips correctly. |
| `/rs-en/privacy` | 7+x-default cluster + **`<meta name="robots" content="index, follow">`** ✓ | `en-RS` ✓ | 9 alternates ✓ | EN locale stays indexable. |

**Cluster integrity:** For the per-base-site clusters (product, catalog, user) the bidirectional check passes by construction — each cluster sibling computes its URL with the same `pathBuilder(l)` formula, so back-references are symmetric. **For all-locales clusters the cluster is structurally degraded** — see Brief vs reality. I did not run the optional `scripts/verify-hreflang.ts` script; the BCP-47 collision finding makes the script's "expected siblings" assertion ill-defined until Mastermind decides which of the resolution options below to take.

## Cleanup performed

- Removed the dead `getTenantLocale` import from `app/layout.tsx` (its only call site moved to `localeToBcp47`).
- Inlined the `t('page.about.url')`, `t('page.pricing.url')`, etc. translation calls into a single `pagePath` const per file so the `pathBuilder` closure can reuse the same path string for every sibling without re-invoking the translator (translator is bound to the current request's locale anyway; the seeds are identical across the four METADATA locales — verified by grepping `oglasino-backend/src/main/resources/data/translations/0001-data-web-translations-{EN,RS,RU,CNR}.sql` for each of the five URL keys).
- No commented-out code added. No debug logging. No TODO/FIXME added. The pre-existing `// TODO FIX URL (add locale)` at `utils.ts:115` is in scope for brief 5 and left in place.

## Config-file impact

- `conventions.md`: no change.
- `decisions.md`: no change drafted from this session. (The seam-resolution close on per-base-site vs all-locales clustering is a Mastermind call — see Brief vs reality for the question.)
- `state.md`: no change drafted.
- `issues.md`: one possible new entry; deferred to Mastermind (see "For Mastermind / Drafted config-file text" below). The audit's Part 1 `alternates.languages` and `openGraph.locale`/`alternateLocale` HIGH-severity findings flip to `partially fixed` — emitted on every public-page generator, but the all-locales cluster is structurally degraded per the BCP-47 collision finding.

## Obsoleted by this session

- The `htmlLang.split('-')[0]` truncation in `app/layout.tsx` — **deleted this session.** Closes spec §6.6.
- The `getTenantLocale(locale).locale` derivation in `app/layout.tsx` — **deleted this session** (replaced by `localeToBcp47(locale)`); the `getTenantLocale` import is gone too.
- No metadata-generator code was deleted in this session (the brief was purely additive on the metadata side).

## Conventions check

- **Part 4 (cleanliness):** confirmed.
- **Part 4a (simplicity):** structured evidence in "For Mastermind."
- **Part 4b (adjacent observations):** one flag below.
- **Part 6 (translations):** N/A this session — no new keys authored; existing keys (`page.about.url`, `page.pricing.url`, etc.) consumed as before.
- **Part 7 (error contract):** N/A.
- **Part 11 (trust boundaries):** every emitted value is server-derived — routing locale (header, set by middleware from URL), translation seeds, route params validated by the existing handlers, base-site lookup from `getBaseSiteServer()`. No client-supplied values reach hreflang or og:locale.

## Known gaps / TODOs

- The all-locales hreflang cluster is **structurally degraded** by BCP-47 collisions between rs and rsmoto. See Brief vs reality for the concrete view-source evidence and the three resolution options. Not fixed this session — needs Mastermind direction.
- Schema.org Validator / Google Rich Results Test / hreflang testers cannot be run from this session (no external network from inside the harness). Owed by Igor: paste the all-locales pages into Ahrefs' or Merkle's hreflang testers to confirm Google's tolerance / behaviour for collision-affected clusters.
- The cluster integrity script (`scripts/verify-hreflang.ts`) from spec §G was not built. Engineer's-call decision: the BCP-47 collision finding makes the script's "expected siblings" assertion ill-defined until Mastermind picks a resolution. Once Option A or B (below) is chosen, the script becomes deterministic and worth writing as a CI artifact.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - **Added (earned complexity):**
    - `LOCALE_TO_BASE_SITE` (private const) — single source of truth for which routing locales share a base site. Three concrete callers (`localesForSameBaseSite`, both inline in `buildAlternateLanguages` and `buildOgLocales`).
    - `localesForSameBaseSite(locale)` exported — concrete callers: `buildAlternateLanguages`, `buildOgLocales` (twice each, via the helpers' mode branch). **Named second outside caller:** brief 3 (server-side breadcrumbs) may consume this for canonical-URL composition per-base-site; if brief 3 doesn't need it, drop the export then. Per the spec's note in Part A.
    - `HreflangMode` type — two-value union (`'all-locales' | 'base-site-scoped'`); the alternative was a `boolean` flag (`isAllLocales`), rejected because the named-string form makes the call site read better (`'all-locales'` vs `true` is more legible).
    - `buildAlternateLanguages` and `buildOgLocales` — concrete callers: all ten public-page generators. Earned because the path-building and the cluster-membership rule combine to roughly seven lines per page if inlined; centralising both into one well-named helper drops the per-page wiring to a four-line `pathBuilder` closure plus two assignments.
    - `bcp47ToOgLocale(bcp47)` — single concrete caller today (used inside `buildOgLocales` and once in the intro special case). Earned because the underscore-vs-hyphen difference between BCP-47 and Open Graph is a Facebook-spec subtlety worth naming. Single helper, one line of body, but the name carries the why.
  - **Considered and rejected:**
    - A per-page hreflang helper (e.g. `withHreflang(metadata, locale, mode, pathBuilder)` returning a new `Metadata`) — rejected. The current `alternates: { canonical, languages }` and `openGraph: { ..., locale, alternateLocale }` shape is so flat that a wrapper would obscure rather than help.
    - Consolidating the boilerplate of the ten metadata generators into a shared template — rejected as out of scope and risky-blast-radius (the files diverge enough in their other fields).
    - Hardcoding the path component (`/about`, `/cenovnik`, etc.) instead of using `t('page.about.url')` — rejected; the existing pattern uses the translation key and the seeds are identical across locales, so reusing the existing key keeps the per-page generator consistent with itself.
    - Reversing the iteration order in `buildAlternateLanguages` so `rs` wins the BCP-47 collision instead of `rsmoto` — **rejected for this session.** Would paper over the structural problem without solving it; see Brief vs reality. Mastermind should pick a resolution direction first.
    - Threading a "winner per BCP-47" override into the helper — rejected as premature design before Mastermind decides whether the collision case should silently pick a winner at all.
  - **Simplified or removed:**
    - The `htmlLang.split('-')[0]` truncation in `app/layout.tsx` — replaced by the direct `localeToBcp47(locale)` derivation. One less unsafe string operation.
    - The `getTenantLocale` import in `app/layout.tsx` — removed (the indirection went away with the truncation).

- **Brief vs reality (CRITICAL — Mastermind decision needed):**

  - **BCP-47 collisions silently drop the rs URLs from every all-locales hreflang cluster.** The spec table lists ten routing locales × ten BCP-47 tags as if it were a 1:1 mapping. It isn't — three pairs collide:
    - `rs-sr` and `rsmoto-sr` both map to BCP-47 `sr-RS`.
    - `rs-en` and `rsmoto-en` both map to BCP-47 `en-RS`.
    - `rs-ru` and `rsmoto-ru` both map to BCP-47 `ru-RS`.

    The brief's pseudocode (and my implementation) uses `Record<string, string>` keyed by BCP-47:
    ```ts
    for (const locale of siblings) {
      languages[localeToBcp47(locale)] = pathBuilder(locale);
    }
    ```
    With iteration order `[rs-sr, rs-en, rs-ru, rsmoto-sr, rsmoto-en, rsmoto-ru, me-sr, me-en, me-ru, me-cnr]`, the rsmoto assignments overwrite the earlier rs assignments. Concrete view-source on `/rs-sr/about`:

    ```
    hrefLang="sr-RS"      href="https://oglasino.com/rsmoto-sr/o-nama"   ← /rs-sr/o-nama is DROPPED
    hrefLang="en-RS"      href="https://oglasino.com/rsmoto-en/o-nama"   ← /rs-en/o-nama is DROPPED
    hrefLang="ru-RS"      href="https://oglasino.com/rsmoto-ru/o-nama"   ← /rs-ru/o-nama is DROPPED
    hrefLang="sr-ME"      href="https://oglasino.com/me-sr/o-nama"
    hrefLang="en-ME"      href="https://oglasino.com/me-en/o-nama"
    hrefLang="ru-ME"      href="https://oglasino.com/me-ru/o-nama"
    hrefLang="sr-Latn-ME" href="https://oglasino.com/me-cnr/o-nama"
    hrefLang="x-default"  href="https://oglasino.com/rs-sr/o-nama"
    ```

    The canonical and `x-default` still point to `/rs-sr/o-nama`, so Google has a workable canonical signal. But the rs base site's three locale URLs are not first-class siblings in the cluster — they appear only as the x-default target. The rsmoto URLs ARE in the cluster, so a Serbian-language searcher Google routes via hreflang lands on rsmoto's general home page, not rs's. The largest market loses cluster recognition.

    This affects **all seven all-locales pages** (intro, home, about, pricing, free-zone, privacy, terms).

    **Per-base-site pages are unaffected** — each cluster has 3 (rs/rsmoto) or 4 (me) members with distinct BCP-47 tags within the cluster. Verified clean on `/rs-sr/product/8573/...` (emits all three rs URLs as siblings).

    **og:locale:alternate also affected but less severely:** Open Graph uses arrays not maps, so the alternates list emits all 9 siblings (including the three duplicates). Crawlers tolerant of duplicate OG values are unaffected; strict parsers may dedupe and pick one.

    **Three resolution options:**

    1. **Option A — treat all-locales as per-base-site.** The spec's premise ("home/static content is the same across rs/rsmoto/me, just in different languages") is empirically false: rs sells general goods, rsmoto sells motorcycles, me serves Montenegro. The "language variants of the same content" criterion that hreflang is built to express doesn't hold cross-base-site. Under this option:
       - rs home cluster: `[rs-sr, rs-en, rs-ru]` + x-default → 3 entries.
       - rsmoto home cluster: `[rsmoto-sr, rsmoto-en, rsmoto-ru]` + x-default → 3 entries.
       - me home cluster: `[me-sr, me-en, me-ru, me-cnr]` + x-default → 4 entries.
       - x-default per cluster: rs-sr for rs cluster, rsmoto-sr for rsmoto cluster, me-sr for me cluster.
       - **Engineering impact:** every "all-locales" call site flips to `'base-site-scoped'`. Code change is one constant flip per generator. Privacy/Terms x-default behaviour needs a Mastermind call (since the EN content lives on rs-en, rsmoto-en, me-en — three separate clusters).
       - **SEO impact:** Google sees three independent clusters. rs and rsmoto compete for the same Serbian-language searcher; me serves Montenegro. This matches user intent: a Serbian user looking for motorcycles wants rsmoto; a Serbian user looking for general goods wants rs. Cluster splitting reflects the underlying product split.
    2. **Option B — pick a "winner" per BCP-47 in the all-locales cluster.** Iteration order in `buildAlternateLanguages` flipped so rs wins (the largest market). rsmoto's home/static URLs would not appear in the cluster — they'd still resolve correctly when crawled directly, but Google wouldn't see them as alternates of rs's pages. **Net result:** rsmoto loses cluster recognition for its home/static pages.
    3. **Option C — disambiguating subtags.** Use `sr-RS-x-rsmoto` (private-use subtag) or similar. Non-standard; per IANA, private-use subtags aren't recognised by Google's hreflang processor (they'd be ignored or treated as a separate unindexed locale). Probably not viable.

    **My recommendation:** Option A. The cluster structure should mirror the content structure, and rs/rsmoto/me serve genuinely different inventories. The spec §6.4 already segregates product/catalog/user this way; extending to home/static is internally consistent. Privacy/Terms could keep the current "all ten in one cluster" approach if Mastermind wants — those pages do share content (the English markdown). Open to either.

    **I have not picked a direction unilaterally.** The current code ships with Option B's failure mode (rsmoto wins by iteration order) but no judgement was applied to that choice — it's accidental.

  - **`page.user.url` translation seed inconsistency.** While verifying that the path translation keys are stable across the four locales (so the `pathBuilder` reusing the current-locale `t(...)` value is safe), I noticed:
    - `page.about.url`, `page.pricing.url`, `page.terms.url`, `page.privacy.url`, `page.free.zone.url` — all four METADATA locales (EN/RS/RU/CNR) seed to **the same value** (`/o-nama`, `/cenovnik`, `/uslovi`, `/politika-privatnosti`, `/dzabe-zona`).
    - `page.user.url` — EN/CNR/RU seed to `/user`, RS seeds to `/korisnik`. **This is a stale seed.** The actual user page route is `/user/{userId}` (`app/[locale]/(portal)/(public)/user/[userId]/page.tsx`); `/korisnik` doesn't resolve. The user generator's metadata-level `canonical` uses `tMetadata('page.user.url')` so on an RS-locale request the canonical points at `/korisnik` — which 404s.

    My code consumes `/user/${user.id}` literally for the hreflang sibling URLs (per the spec's explicit example), so the cluster URLs are correct. But the metadata-level canonical on RS is still broken (and brief 5 owns the user-canonical fix). Flagged so brief 5's engineer is aware of the seed problem in addition to the missing `userId` problem.

  - **`<html lang>` source in `app/layout.tsx` did have routing-locale access** (via the `x-next-intl-locale` header set by middleware), so the fix lands cleanly in `app/layout.tsx` itself — no need to move to `app/[locale]/layout.tsx`. The apex `/` route has no locale segment so the header is null; I fall back to `'sr-RS'` (matching the x-default cluster choice).

  - **All 10 metadata generators accept `locale` as a parameter** already (good — no signature changes were needed). The `ProductMetadata` interface still has `locale` post brief-1 trim; the `generateUserPageMetadata` and `generateCatalog/HomePageMetadata` already take `locale`; about/pricing/free-zone/privacy/terms all take `(t, locale)`.

  - **Catalog and product slugs are stable across locales of the same base site.** Verified:
    - `CategoryDTO.route` is a single non-localized string (`labelKey` is the i18n source). Catalog tree is loaded from per-base-site JSON (`catalog-{rs,me,rsmoto}.json`), one tree per base site shared across locales.
    - `ProductOverviewDTO.name` is a single canonical string (the seller-authored listing name). `normalizeProductName(product.name)` is locale-agnostic.

    Both checks pass — the simple `pathBuilder(l) = ${baseUrl}/${l}${slugPath}` closures are correct.

- **Hreflang cluster integrity verification:**
  - **Per-base-site clusters (product, catalog, user):** confirmed clean by construction. The `pathBuilder` is the same formula evaluated per sibling, so back-references are symmetric. Concrete spot-check: `/rs-sr/product/8573/...` emits exactly `[rs-sr, rs-en, rs-ru]` + x-default → `/rs-sr/...`. No collisions inside the cluster.
  - **All-locales clusters (intro, home, about, pricing, free-zone, privacy, terms):** **structurally degraded** by BCP-47 collisions. Each cluster emits 7 entries instead of 10. The three rs URLs are silently dropped. See Brief vs reality.
  - **`x-default` always emits** per spec; confirmed on every spot-checked surface.

- **`<html lang>` verification:**
  - `/me-cnr` → `lang="sr-Latn-ME"` ✓ (was `cnr`).
  - `/rs-sr/about` → `lang="sr-RS"` ✓ (was `sr`).
  - `/me-cnr/product/8573/...` → `lang="sr-Latn-ME"` ✓.
  - `/` (apex) → `lang="sr-RS"` ✓ (apex fallback; was `sr`).
  - `/rs-en/privacy` → `lang="en-RS"` ✓.

- **Privacy/Terms `noindex` confirmation:**
  - `/rs-sr/privacy` → `<meta name="robots" content="noindex, follow">` ✓.
  - `/rs-en/privacy` → `<meta name="robots" content="index, follow">` ✓.
  - Cluster preserved: both pages emit the same 7-entry sibling list + x-default + EN-locale → indexable.

- **Drafted config-file text (for Docs/QA at feature close):**

  - **`issues.md` candidate (potential new entry — Mastermind to confirm):**
    > **2026-05-23 — Hreflang cluster degradation on all-locales pages from BCP-47 collisions.** The spec's "10 siblings per all-locales page" design collides with BCP-47 — `rs-sr`/`rsmoto-sr`, `rs-en`/`rsmoto-en`, `rs-ru`/`rsmoto-ru` each share a BCP-47 tag. Next.js `alternates.languages: Record<string, string>` keys collapse to 7 entries; rsmoto wins by iteration order, the three rs URLs drop from the cluster (canonical and x-default still point to rs-sr so the canonical signal survives). Decision pending — Mastermind to choose between Option A (treat all-locales as per-base-site, splitting into three independent clusters of 3 + 3 + 4), Option B (iteration-order flip so rs wins, rsmoto loses cluster recognition), or Option C (private-use BCP-47 subtags — not viable per Google's processor).
  - **`issues.md` flip candidate** (audit Part 1, "zero pages emit `alternates.languages`"): flip to `partially fixed (2026-05-23, session oglasino-web-seo-foundation-3)` with note that per-base-site clusters ship correctly; all-locales clusters are degraded per the new entry above. Full close pending Mastermind's resolution direction.
  - **`issues.md` flip candidate** (audit Part 1, "zero pages emit `openGraph.locale`/`alternateLocale`"): flip to `partially fixed (2026-05-23, session oglasino-web-seo-foundation-3)`. Same caveat — og:locale:alternate emits duplicates on all-locales pages due to the same BCP-47 collision (less severe than the hreflang case because OG uses array semantics; crawlers see duplicates rather than missing values).
  - **`issues.md` flip candidate** (spec §6.6, `<html lang>` truncation): flip to `fixed (2026-05-23, session oglasino-web-seo-foundation-3)`. `app/layout.tsx` now derives `<html lang>` from `localeToBcp47(locale)` with an apex fallback to `'sr-RS'`; view-source confirmed `/me-cnr` → `sr-Latn-ME`.
  - **Spec amendment to `oglasino-docs/features/seo-foundation.md` §6.3 / §6.4:** the §6.3 mapping table needs an explicit "BCP-47 collision" callout. The §6.4 per-base-site clustering rule should extend to home/static if Mastermind picks Option A. Drafted spec text deferred until Mastermind picks a direction.

- **Adjacent observations (Part 4b):**

  - **The intro page's apex fallback in `<html lang>` (`'sr-RS'`) is hardcoded.** Acceptable for now since `rs-sr` is the x-default cluster choice (spec §6.3) — they should track each other. If Mastermind ever changes x-default away from rs-sr, this fallback should follow. Severity: low (cosmetic coupling). Out of scope this brief.

  - **`generateProductPageMetadata.ts` still sets `metadataBase: new URL(\`${basicMetadata.baseUrl}/${locale}\`)` at the per-page level** (audit Part 7.11 — pre-existing). With the layout-level `metadataBase` uncommented per brief 1, this is redundant. I did not touch it (out of scope), but the per-page override may interact subtly with relative OG-image URL resolution. Severity: low. Brief 5 territory.

  - **The view-source check on `/rs-sr/catalog/zene` returned not-found** (`<meta name="robots" content="noindex"/><meta name="next-error" content="not-found"/>`) — `zene` isn't a valid `rs` catalog slug (the catalog JSON enumerates `women`, `men`, `kids`, etc. not Serbian-language slugs). The catalog `route` resolution per the audit uses the English-like top-level keys. **No production impact on hreflang code** — the metadata generator's path-building works for valid slugs (and Next.js calls `generateMetadata` with the same `params` as the page, so if the page route resolves the metadata one does too). Just a verification-data gap. Severity: low.

- **Anything that surprised you:**

  - **The BCP-47 collision was hiding in plain sight in the spec table** — three pairs of rows have the same BCP-47 column value, the same og:locale column value, and different cluster scopes. The brief's pseudocode `for (const locale of siblings) { languages[localeToBcp47(locale)] = ... }` writes to a `Record<string, string>` so the collision behaviour is silent collapse. I caught it only at view-source time. The brief's Definition-of-Done verification step ("`<link rel="alternate" hreflang="...">` block lists the expected siblings per the cluster mode") would also catch it — but only if the engineer interpreted "expected siblings" as "ten siblings on an all-locales page" and noticed seven. I'd guess most engineers (and the brief's author) would interpret a 7-entry list as "ten unique BCP-47 tags, my data only has seven, this is correct."

  - **The current implementation ships a silent rsmoto-wins outcome** due to iteration order. Not a deliberate Mastermind choice; just where the `Record` collapse happens to land. If Mastermind picks Option B, the choice should be deliberate (likely rs wins, not rsmoto).

  - **The `<html lang>` fix needed `'sr-RS'` as the apex fallback, not the brief's "use locale" pattern.** The apex `/` route runs before `[locale]` is resolved, so `x-next-intl-locale` is null. I picked `'sr-RS'` to match the x-default cluster choice; the previous code's `'sr'` fallback was bare-language only.

  - **Path translation seeds for the five "URL" keys are identical across all four METADATA locales** (modulo the `page.user.url` exception flagged above) — verified by grepping the four backend SQL files. This made the `pagePath` consolidation safe (one `t(...)` call per page, reused for every sibling). If a future locale ever localizes the path, the all-locales hreflang URLs would silently break — but that's already a problem with the existing canonical computation, not something this brief introduced.
