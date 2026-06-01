# Image Alt Text Translation

**Status:** `shipped`
**Branch:** `feature/image-alt-translation` (off `dev`)

---

## Scope

**In:** translate three hardcoded English `alt` attributes and one hardcoded Serbian `description` literal across five code sites on the intro page and about page. Four unique keys, four locales, 16 seed rows. The description key (`intro.meta.description`) was folded in mid-feature after audit 1 flagged the Serbian literal shipping to EN/RU/CNR search results.

**Out:** any other hardcoded English strings on these pages (e.g., the `'Oglasino'` brand name used as `<title>` and `metadata.title` in `generateIntroPageMetadata.ts`, the `© Memento Tech` copyright footer in `app/page.tsx`) — those are either brand/proper-noun ignores or separate work.

---

## Background

Deferred from the SEO foundation feature. The SEO brief 5b draft included these as item B3; they were removed from 5b-revised and handed off to `future/seo-image-alt-translation.md`. The SEO foundation closing entry in [decisions.md](../decisions.md) (2026-05-24) lists "image alt translation" among six deferred items. The [seo-foundation.md](seo-foundation.md) spec §9.9 documents the finding. The 2026-05-25 web and backend audits confirmed all four keys (including `intro.meta.description`, originally out of scope) as in-scope; the description fold-in was confirmed after the spec's initial three-key draft.

---

## Sites

| # | File | Line | Current literal | Proposed key | Namespace |
|---|------|------|----------------|-------------|-----------|
| 1 | `app/page.tsx` | 29 | `alt="Oglasino intro"` | `intro.image.alt` | `METADATA` |
| 2 | `app/page.tsx` | 38 | `alt="Oglasino intro"` | `intro.image.alt` | `METADATA` |
| 3 | `app/[locale]/(portal)/(public)/about/page.tsx` | 41 | `alt={'Oglasino portal'}` | `about.hero.image.alt` | `METADATA` |
| 4 | `src/metadata/generateIntroPageMetadata.ts` | 53 | `alt: 'Oglasino'` | `intro.og.image.alt` | `METADATA` |
| 5 | `src/metadata/generateIntroPageMetadata.ts` | ~17 | `const description = 'Vaša platforma za kupovinu i prodaju'` | `intro.meta.description` | `METADATA` |

Sites 1 and 2 render the same image (`/intro.jpg`) twice for responsive layout: site 1 is the left-half desktop panel (`hidden md:block md:w-1/2`), site 2 is the full-background mobile view (`md:hidden`). Same image, same alt text, same key.

Site 5 is one source variable that cascades to three metadata surfaces in the same file: `metadata.description`, `openGraph.description`, `twitter.description`.

---

## Namespace decision

**Verdict:** `METADATA` for all four keys.

Alt text is a meta-attribute of images. The og:image alt (site 4) is unambiguously METADATA. Co-locating all five sites in one namespace keeps related strings together and requires one SQL seed group to append to per Part 6 Rule 3.

---

## Translation keys

| Key | EN value | RS | RU | CNR |
|-----|----------|----|----|-----|
| `intro.image.alt` | `Oglasino marketplace introduction` | [placeholder pending native-translator review] | [placeholder pending native-translator review] | [placeholder pending native-translator review] |
| `about.hero.image.alt` | `Oglasino online marketplace` | [placeholder pending native-translator review] | [placeholder pending native-translator review] | [placeholder pending native-translator review] |
| `intro.og.image.alt` | `Oglasino marketplace` | [placeholder pending native-translator review] | [placeholder pending native-translator review] | [placeholder pending native-translator review] |
| `intro.meta.description` | `Buy and sell on Oglasino — the marketplace for fashion, home, electronics, tools, hobbies, sports, and services. Free to post.` | `Kupujte i prodajte na Oglasino — pijaca za modu, dom, elektroniku, alat, hobije, sport i usluge. Besplatno postavljanje oglasa.` [placeholder pending native-translator review] | `Pokupayte i prodavayte na Oglasino — ploshchadka dlya mody, doma, elektroniki, instrumentov, khobbi, sporta i uslug. Besplatnoe razmeshchenie.` [placeholder pending native-translator review] | `Kupujte i prodajte na Oglasino — pijaca za modu, dom, elektroniku, alat, hobije, sport i usluge. Besplatno postavljanje oglasa.` [placeholder pending native-translator review] |

---

## Task list

### Backend briefs (two sessions)

**Session 1 (completed):** seeded three keys (`intro.image.alt`, `about.hero.image.alt`, `intro.og.image.alt`) in the `METADATA` namespace across EN / RS / RU / CNR via inline-append per Part 6 Rule 3 with reserved ID slots for the fourth key.

**Session 2:** seed the fourth key (`intro.meta.description`) in the `METADATA` namespace across EN / RS / RU / CNR using the reserved ID slots from session 1.

### Web brief

Swap the five hardcoded English literals for translation calls at the five named sites:
- Sites 1 and 2 (`app/page.tsx`): the intro page is a server component outside the `[locale]` segment; it already has `const tMetadata = await getTranslations(TranslationNamespaceEnum.METADATA)` — use `tMetadata('intro.image.alt')`.
- Site 3 (`about/page.tsx`): already has `const tMetadata = await getTranslations(TranslationNamespaceEnum.METADATA)` — use `tMetadata('about.hero.image.alt')`.
- Site 4 (`generateIntroPageMetadata.ts`): receives `tMetadata` as a parameter — use `tMetadata('intro.og.image.alt')`.
- Site 5 (`generateIntroPageMetadata.ts`): same file as site 4 — replace the `const description = '...'` literal with `tMetadata('intro.meta.description')`. The translated value cascades to `metadata.description`, `openGraph.description`, and `twitter.description`.

---

## Trust boundary

Alt text and meta descriptions are display-only, not used in moderation, authorization, or state-transition decisions. No trust boundary.

---

## Definition of done

- All five sites read from translations; no English literals remain at those sites.
- Backend seed verified (four keys × four locales = 16 seed rows).
- Web swap verified (five code sites).
- Native-translator review of RS / RU / CNR added as a Risk Watch entry in `state.md`.

---

## Session log

- **2026-05-25** — `oglasino-docs` session 1: initial spec authored from `future/seo-image-alt-translation.md` handoff. Three keys, five code sites. Source file deleted.
- **2026-05-25** — `oglasino-docs` session 2: spec amended from three keys to four keys after audit 1 flagged the Serbian `description` literal (C2 finding). Fourth key `intro.meta.description` added; backend session 1 reserved ID slots confirmed; backend session 2 brief drafted.
- **2026-05-25** — `oglasino-backend` session 1: seeded three keys (`intro.image.alt`, `about.hero.image.alt`, `intro.og.image.alt`) across four locales (12 rows). IDs: EN 3269-3271, RS 5369-5371, RU 7469-7471, CNR 1169-1171. Fourth-key slots reserved.
- **2026-05-25** — `oglasino-backend` session 2: seeded fourth key (`intro.meta.description`) across four locales (4 rows) using reserved IDs EN 3272, RS 5372, RU 7472, CNR 1172. All 16 seed rows complete.
- **2026-05-25** — `oglasino-backend` category-catalog-audit: read-only audit of category topology per base site. Produced the English meta description value drafted against the actual top-level categories in `rs` and `me` catalogs (10 categories).
- **2026-05-25** — `oglasino-web` audit 1: read-only audit of all hardcoded strings on intro and about pages. Confirmed spec's three sites, flagged C2 (Serbian description) for fold-in, flagged hardcoded `rs-sr` in SearchAction urlTemplate as adjacent observation.
- **2026-05-25** — `oglasino-web` base-site audit: read-only audit confirming translation schema has no base-site dimension. All four images are base-site-neutral. Spec stays at language-only keys. Flagged redundant per-tenant translation cache entries and about page content duplication as adjacent observations.
- **2026-05-25** — `oglasino-web` session 1: swapped five hardcoded literals for translation calls at all five code sites. Three files touched. 229 tests passing, tsc/lint/hreflang green.
- **2026-05-25** — `oglasino-docs` session 3: feature close-out. Status flipped to `shipped`, config files updated, engineer sessions archived.
