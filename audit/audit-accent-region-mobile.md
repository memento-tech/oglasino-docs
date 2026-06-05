# Audit — Mobile deep-link region/city accent matching

**Repo:** `oglasino-expo` · **Branch:** `new-expo-dev` · **Mode:** read-only, no code changes
**Date:** 2026-06-05
**Question:** Web has a confirmed accent bug — region/city URL tokens are accent-stripped on
write (`Niš → nis`) but the hydrate read can't match them back, so accented regions/cities
silently fail. Does mobile's deep-link region/city matching have the SAME defect?

**One-line verdict:** **No.** Mobile is *not* affected. Its deep-link read already uses the
correct slug-vs-slug strategy (`toQueryParam(t(label))` on both sides), the exact strategy the
brief identifies as web's correct SEND path. An accented `?regions=nis` for `Niš` resolves on
mobile, and an existing test proves it green on the current tree.

---

## Q1 — `parseRegionsAndCities` verbatim + how it matches inbound tokens

`src/lib/utils/filtersUtil.ts:233-253` (current line numbers):

```ts
function parseRegionsAndCities(
  params: QueryParams,
  baseSite: BaseSiteDTO,
  t: LabelTranslator
): SelectedRegionsAndCities {
  const regionSlugs = splitList(params['regions']);
  const citySlugs = splitList(params['cities']);

  // Web is the canonical emitter and bakes the region/city auto-collapse into the
  // URL before serialization, so the inbound set is applied verbatim (no collapse
  // replay here). Match by slug-vs-slug on the translated label.
  const regions = (baseSite.regions ?? []).filter((region) =>
    regionSlugs.includes(toQueryParam(t(region.labelKey)))
  );

  const cities = (baseSite.regions ?? [])
    .flatMap((region) => region.cities ?? [])
    .filter((city) => citySlugs.includes(toQueryParam(t(city.labelKey))));

  return { regions, cities };
}
```

**How it matches an inbound token to a `RegionDTO`/`CityDTO`:** it does **not** convert the
inbound token at all. It takes each candidate's translated label (`t(region.labelKey)` /
`t(city.labelKey)`), runs it through `toQueryParam(...)` to produce a *slug*, and checks whether
that slug is in the set of inbound slugs (`regionSlugs.includes(...)`). The inbound token stays
raw; the candidate side is normalized down to the same slug space.

This is the **`toQueryParam(label)` normalization** option from the brief's Q1 menu — **not** a
lossy `fromQueryParam`-style conversion and **not** a raw compare. Both sides land in the same
accent-stripped, lowercased slug space, so the comparison is accent-robust by construction.

`toQueryParam` itself (`src/lib/utils/filtersUtil.ts:46-55`):

```ts
export function toQueryParam(str: string): string {
  // 1. Normalize accents
  const normalized = str.normalize('NFD').replace(/[̀-ͯ]/g, '');
  // 2. Encode dashes as a special token to avoid conflict
  const escapedDashes = normalized.replace(/-/g, '_d_');
  // 3. Replace spaces with dashes, lowercase everything
  return escapedDashes.replace(/\s+/g, '-').toLowerCase();
}
```

NFD-decompose → strip combining marks (`̀-ͯ`) → escape literal dashes → spaces→dash →
lowercase. `Niš → nis`, `Novi Sad → novi-sad`.

---

## Q2 — The round-trip: where is the slug WRITTEN, and does write match read?

**Mobile has no region/city write side.** Mobile is a pure *consumer* of deep links; it never
emits a region/city slug.

Evidence:
- `toQueryParam` has exactly **five** references repo-wide, all in `filtersUtil.ts`, all on the
  **read/parse** side (`:46` definition, `:61` doc comment, `:157` filter-options match, `:245`
  region match, `:250` city match). There is no `toQueryParam` call in any share/URL-builder,
  navigation, or store-serialization path. Grep across `src/` confirms zero emit-side callers.
- The code itself states the contract: `filtersUtil.ts:241-243` — *"Web is the canonical emitter
  and bakes the region/city auto-collapse into the URL before serialization."* The write side
  (NFD-strip + lowercase) lives in **web**, outside this repo.
- Mobile's only outbound URL builders are product *share* links (`getNormalizedProductUrl` /
  `getForgotPasswordUrl` in `src/lib/utils/utils.ts`) — neither carries region/city filter
  tokens.

So the round-trip is **web writes the slug → mobile reads it**. The relevant question becomes
"does mobile's read normalization match web's write normalization?" Mobile's read normalizes the
candidate label with the *same* NFD-strip + lowercase recipe web uses to write the slug
(`str.normalize('NFD').replace(/[̀-ͯ]/g, '').toLowerCase()`, plus the shared `_d_`
dash-escape). Because mobile normalizes its *own* candidate down to that identical slug space —
rather than trying to un-slug the inbound token (the lossy direction that bites web's hydrate) —
the read aligns with the write. No identity gap.

---

## Q3 — VERDICT: does `?regions=nis` (for `Niš`) resolve, or silently drop?

**It resolves.** Walk-through with `Niš`:

- Inbound: `?regions=nis` → `regionSlugs = ['nis']`.
- Candidate `Niš`: `toQueryParam(t('region.nis'))` = `toQueryParam('Niš')` = `'nis'`.
- `['nis'].includes('nis')` → **true** → region selected.

This is the **same class of operation web's correct SEND path uses** (raw token vs
`toQueryParam(label)`), and explicitly **different** from the web hydrate defect (lossy
`fromQueryParam` vs raw `t(label)`, which would compare `'nis'` to `'Niš'` and miss). Mobile's
hydrate never reconstructs a raw label to compare against, so the accent never has a chance to
break the match.

**Proven by an existing test**, run green on the current tree:
- `src/lib/utils/filtersUtil.test.ts:217-220` — `it('matches an accented region by slug (nis →
  Niš)')`, fixture `region.nis → 'Niš'` (`:23`, `:108-109`), asserts `?regions=nis` resolves to
  region id `200`.
- Ran: `npx vitest run src/lib/utils/filtersUtil.test.ts -t "region"` → **3 passed** (the
  accented-region case among them), 0 failed.

Classification: **a different (non-)bug from web** — mobile is *fine*; it already adopted the
correct strategy. ASCII regions are unaffected too (`Beograd → beograd`, matches `?regions=beograd`;
see test `:223-225`, `:242`).

---

## Q4 — Minimal fix shape (if broken)

**No fix required.** Mobile already implements the correct alignment (read normalizes candidate
labels with `toQueryParam`, matching web's write recipe). ASCII still matches (confirmed Q3 +
tests `:223-225`, `:228-230`, `:242`). Nothing to change.

---

## Q5 — Any OTHER mobile path reading region/city (or any label) from a token with the lossy pattern?

Swept every token-reading branch in the deep-link parser. **None use a lossy `fromQueryParam`
pattern; there is no `fromQueryParam` function in the repo at all.** Classification of each
inbound axis (`parseFiltersFromQueryParams`, `filtersUtil.ts:73-104`):

| Axis | Path | Match strategy | Lossy? |
|------|------|----------------|--------|
| Regions / cities | `parseRegionsAndCities` `:233-253` | `toQueryParam(t(label))` slug-vs-slug | No — correct |
| Filter options (`f_<key>`) | `parseSelectedFilters` `:156-158` | `tokens.includes(toQueryParam(t(option.labelKey)))` — same slug-vs-slug | No — correct |
| Order | `parseOrder` `:203-210` | `order.code === code` — raw code compare (no labels) | No — N/A |
| Price range | `parsePriceRange` `:212-231` | bounds kept verbatim; currency by `.code` | No — N/A |
| Range/date filters | `parseRangeTokens` `:173-201` | numeric `from:`/`to:` parse | No — N/A |
| `search_text` | `:83-84` | applied as the slug verbatim (matches web backend) | No — N/A |

The only two label-matching axes (regions/cities and filter options) both use the identical
correct `toQueryParam(t(label))` slug-vs-slug strategy. **No lossy reader exists on mobile.**

---

## Q6 — Is this matching shared with the SEND path (region-city-mobile-request-2)?

**Separate code, and the send path is already correct — and never touches the accent question at
all.** The send path is **entity-id-based**, not slug-based:

- `toRegionAndCityValues` (`filtersUtil.ts:37-44`) projects the selected `RegionDTO`/`CityDTO`
  objects to `{ regionIds, cityIds }` numeric arrays (`region.id`, `city.id`).
- Sent as `selectedRegionAndCityValues` in the search request body —
  `src/components/product/FilteredProductList.tsx:94`.

The SEND path never stringifies a label, never builds a slug, and never normalizes accents — it
ships the backend's own numeric ids. So it is **immune to the accent class of bug by
construction**, and it shares *no* normalization code with the deep-link read. The deep-link read
(`toQueryParam`-based slug matching) and the send (`id`-based projection) are independent. The
`region-city-mobile-request-2` fix is unaffected by, and orthogonal to, anything here: it remains
correct and lossless because it carries ids, not labels.

---

## Summary table

| Q | Answer |
|---|--------|
| Q1 | `parseRegionsAndCities` `:233-253` matches via `toQueryParam(t(label))` — slug-vs-slug normalization, not lossy, not raw |
| Q2 | Mobile has **no** region/city write side; web is sole emitter. Read normalization (NFD-strip+lowercase via `toQueryParam`) matches web's write recipe |
| Q3 | `?regions=nis` for `Niš` **resolves** (proven by passing test `:217-220`). Different from web's bug — mobile is fine |
| Q4 | No fix needed; ASCII confirmed still matching |
| Q5 | No other lossy reader; the only other label axis (`f_` filter options, `:156-158`) uses the same correct strategy; no `fromQueryParam` exists |
| Q6 | Send path is separate, id-based (`region.id`/`city.id`), accent-immune by construction; not shared with the read; already correct |

## Adjacent observation (Part 4b — not in scope, not fixed)

- **(low, doc-accuracy)** `filtersUtil.ts:60-61` claims the mobile parser "Mirrors web's SSR
  `parseFiltersFromQueryParams` (filtersHelper.ts)" and is accent-robust "unlike a
  `fromQueryParam`-against-raw-labels match." Per the brief, web's *hydrate* is the
  `fromQueryParam`-against-raw-labels match that *has* the bug. If so, the comment's "mirrors web"
  is imprecise: mobile actually mirrors web's correct **SEND** strategy and **diverges from** web's
  buggy hydrate — i.e. mobile is correct *because* it did not copy web's hydrate. Cannot confirm
  web-side from this repo (cross-repo, read access is docs-only). Flagged for Mastermind to
  reconcile the comment against web's actual hydrate once the web fix lands; mobile behavior is
  correct regardless of the comment's wording. No code change made.
