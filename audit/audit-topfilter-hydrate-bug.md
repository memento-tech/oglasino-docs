# Audit — topfilter-hydrate-bug (PRICE + REGION on `/`)

**Repo:** oglasino-web · **Branch:** dev · **Mode:** READ-ONLY (no code changed)
**Date:** 2026-06-05

---

## TL;DR

The brief's central hypothesis — *"price and region are dropped in HYDRATE on `/`, behind
the same `categoriesFromPath` / `/catalog` gate the prior audit attributed only to `f_<key>`
filters"* — **is not supported by the code.** I traced both paths line-by-line:

- **PRICE hydrate is ungated.** `FilterManager.tsx:162-178` reads `from`/`to`/`free`/`currency`
  straight from `window.location.search` and calls `setPriceRange(...)`. It has **no**
  dependency on `pathname`, `categoriesFromPath`, or the `/catalog` check. It runs and writes the
  store on `/` exactly as it does on `/catalog`.
- **REGION hydrate is also ungated** (same: `FilterManager.tsx:185-194`, no category dependency)
  — **but it carries a real, independent bug** that is *not* `/`-specific: it matches URL values
  through the **lossy** `fromQueryParam` against the **raw** `t(labelKey)`, so any accented
  region/city name (Niš, Čačak, Užice, Vrnjačka Banja…) fails to resolve and is dropped — on
  **both** `/` and `/catalog`.
- The `categoriesFromPath` gate (`FilterManager.tsx:63`) and the drop-on-miss
  (`FilterManager.tsx:114`) the prior audit identified are **real and correct**, but they apply
  **only** to the generic `f_<key>` loop (lines 103-158). They do **not** touch price (162) or
  region (185). So the prior audit's "top filters resolve on `/`" is **correct for the
  price/region hydrate-read code path** — I re-verified it specifically, as the brief asked, and
  it holds.

**So there is no `/`-specific gate that swallows price or region.** That part of the brief is
refuted by the code (see "Brief vs reality" below). What *is* real and demonstrable:

1. **Region hydrate is broken for accented names** via a `fromQueryParam`/`toQueryParam`
   normalization mismatch (FilterManager.tsx:185-194). This is the same root cause as the
   region **SEND-null** bug — see Q6. **One root cause, two symptoms.**
2. The category-scoped `f_<key>` drop on `/` from the prior audit is unchanged and still valid.

I could **not**, by static reading, find a mechanism that uniquely makes *price* fail on `/`
while succeeding on `/catalog`. If the live repro shows price specifically blank on home, the
cause is **not** the hydrate read and needs a runtime check (see Q4 + "Needs live confirmation").

---

## Q1 — PRICE hydrate path (read URL → store). Is it gated behind `/catalog` / `categoriesFromPath`?

**No gate.** `FilterManager.tsx:162-178`:

```ts
const from = params.get('from') || '';
const to = params.get('to') || '';
const free = params.get('free') === 'true';
const currencyCode = params.get('currency');
const hasPrice = from !== '' || to !== '';
setPriceRange({
  from, to, free,
  selectedCurrency:
    hasPrice && currencyCode
      ? baseSite.allowedCurrencies.find((c) => c.code === currencyCode)
      : undefined,
});
```

- `params` is `new URLSearchParams(window.location.search)` (`FilterManager.tsx:94`).
- This block sits **after** the `categoriesFromPath` resolution but does **not** read
  `categoriesFromPath`, `pathname`, or anything `/catalog`-derived. `selectedCurrency` resolves
  against `baseSite.allowedCurrencies` (global, not path-scoped).
- The only early returns in the effect are `if (!baseSite) return;` and
  `if (lastHydratedPathRef.current === pathname) return;` (`FilterManager.tsx:58-59`) — neither
  is `/`-conditional. The `f_` loop's `return`s (lines 104, 114) are inside a `params.forEach`
  callback and only skip one iteration; they do **not** exit the effect, so price always runs.

**Conclusion: price IS written to the store on `pathname === '/'`.** No drop in the hydrate read.

## Q2 — REGION/CITY hydrate path. Gated behind catalog/category resolution?

**No category gate** — but **functionally broken for accented names.** `FilterManager.tsx:185-194`:

```ts
const regionValues = params.get('regions')?.split(',').map(fromQueryParam) || [];
const cityValues   = params.get('cities')?.split(',').map(fromQueryParam) || [];

setRegionCityValues({
  regions: baseSite.regions?.filter((r) => regionValues.includes(t(r.labelKey))) || [],
  cities:
    baseSite.regions?.flatMap((r) => r.cities || [])
      .filter((c) => cityValues.includes(t(c.labelKey))) || [],
});
```

- `baseSite.regions` is the global region list (not path-scoped); the same on `/` and `/catalog`.
- **The bug:** URL values are run through `fromQueryParam` (`filtersHelper.ts:25-36`) and compared
  to the **raw** `t(r.labelKey)`. `fromQueryParam` is a **lossy** inverse of the `toQueryParam`
  (`filtersHelper.ts:14-23`) that produced the URL value in SYNC (`FilterManager.tsx:273,280`):
  it restores dashes→spaces and capitalizes leading letters, but **cannot restore accents or
  internal casing** (`toQueryParam` NFD-strips accents and lowercases).

Round-trip check (ran `toQueryParam`→`fromQueryParam` against real SR names):

| label | URL stored (`toQueryParam`) | `fromQueryParam(url)` | HYDRATE match (`=== t(labelKey)`) |
|---|---|---|---|
| `Niš` | `nis` | `Nis` | **false** |
| `Čačak` | `cacak` | `Cacak` | **false** |
| `Užice` | `uzice` | `Uzice` | **false** |
| `Vrnjačka Banja` | `vrnjacka-banja` | `Vrnjacka Banja` | **false** |
| `Novi Sad` | `novi-sad` | `Novi Sad` | true |
| `Beograd` | `beograd` | `Beograd` | true |

So accented/diacritic regions & cities **never resolve** and are dropped from the store. This is
**not** `pathname`-dependent — it fails identically on `/` and `/catalog`. Plain ASCII names
(Beograd, Novi Sad) happen to survive, which is likely why the bug looks intermittent / looks
like it "works on catalog."

## Q3 — Line-by-line: landing on `/` with price+region params, where is each dropped?

Walking the HYDRATE effect (`FilterManager.tsx:57-204`) for `pathname === '/'`:

1. `:58` `baseSite` present → continue.
2. `:59` fresh mount → `lastHydratedPathRef.current` is `null` (new `useRef`, `:31`) → continue.
3. `:63` `pathname.startsWith('/catalog')` is **false** → `categoriesFromPath` stays `{}`.
   **This is the only `/`-vs-`/catalog` branch in the whole effect.**
4. `:94` `params = URLSearchParams(window.location.search)` → contains `from/to/regions/cities/...`.
5. `:103-158` `f_<key>` loop: for a **category-scoped** filter, the lookup at `:108-112` is
   `topFilters.find || categoriesFromPath?.topCategory… || sub… || final…`. On `/` only
   `topFilters` is consulted (the three `categoriesFromPath?.…` are `undefined`), so a
   category filter misses and `:114 if (!filter) return` drops it. **← the prior audit's
   finding; correct, and scoped to `f_<key>` only.**
6. `:162-178` **PRICE** — runs unconditionally → `setPriceRange(...)`. **NOT dropped.**
7. `:181-183` order → `setOrder(...)`. Not dropped (resolves against `baseSite.catalog.orderTypes`).
8. `:185-194` **REGION/CITY** — runs unconditionally, but the `fromQueryParam`-vs-`t(labelKey)`
   mismatch (Q2) drops any accented entry. **Dropped by a *value-matching* bug, NOT by a
   `/`-gate.**
9. `:202-203` `lastHydratedPathRef.current = pathname; setHydrated(true);`

**There is no line where price is dropped on `/`.** Region is dropped at `:189/:193` only for
accented names, and that drop is path-independent. The `categoriesFromPath` gate that the prior
audit (correctly) attributes to `f_<key>` does **not** also swallow price/region — they bypass it
entirely.

## Q4 — Why does the SAME hydrate populate price+region on `/catalog` but not on `/`?

**For price: it doesn't differ.** There is no path-dependent branch on the price path; price
hydrates the same on both routes. **The prior audit's "top filters resolve on `/`" is correct
for price, and the brief's premise here is wrong for price** — the contradicting lines are
`FilterManager.tsx:162-178` (ungated) vs the brief's "price is dropped on `/`."

**For region: it also doesn't differ by path** — region fails for accented names on `/catalog`
too (Q2 table). If the live repro genuinely showed region resolving on `/catalog` but not on `/`,
the most likely explanation is **not** the hydrate read but **store retention + SYNC timing**:

- The filter store is a **module singleton with no `persist`** (`useFilterStore.ts:46,189`), so
  its values survive client navigation. The **product detail page does not mount
  `SelectableFilterManagerWrapper`** (mount sites are only home `page.tsx:64`, catalog
  `page.tsx:185`, owner/admin products) — so catalog → product → BACK never clears the store; the
  region the user saw "stay" on catalog can be the **retained singleton value**, not a fresh
  resolve.
- On the logo→home mount, because the singleton `hydrated` flag is already `true` (set on
  catalog) and HYDRATE sets `lastHydratedPathRef.current = pathname` synchronously *before* the
  SYNC effect runs in the same commit, the SYNC gate (`FilterManager.tsx:210`,
  `lastHydratedPathRef.current !== pathname || !hydrated`) **passes on first mount** and SYNC can
  fire immediately. If HYDRATE has just emptied region (accented-name drop), SYNC rebuilds the URL
  from the now-empty store and **`router.replace`s region/city out of the URL**
  (`FilterManager.tsx:270-282, 321`). That is the cascade into Q6.

**This timing path needs a runtime check** — see "Needs live confirmation." Statically I can
confirm the *mechanism exists*; I cannot confirm it is what fires in Igor's exact repro, and it
does **not** uniquely depend on `/` (it depends on the store being emptied by the region
value-bug, which is path-independent).

> **Direct contradiction to flag:** the brief says "the price+region params ARE in the URL but
> the UI is NOT hydrated." The region cascade above would **strip** region from the URL (SYNC
> replace), which is the opposite of "params stay." Either (a) price (which is NOT stripped,
> since price hydrates fine and SYNC writes it back) is what stays in the URL while only the
> *category* `f_<key>` looks empty, or (b) the exact settled URL differs from the brief's wording.
> The prior audit raised the same "code tends to strip, not keep" caveat. **Confirm the exact
> settled URL on a live repro** (it changes which of price/region/category is the real culprit).

## Q5 — Does home MOUNT the price + region CONTROLS (not just the engine)?

**Yes — both controls render on home.** `page.tsx:65` mounts `SelectableFiltersWrapper
portalScope="portal"` → `SelectableFiltersWrapper.tsx:21-27` renders `<Filters>` →
`Filters.tsx:203-215` renders `<PriceFilter>` and `Filters.tsx:217-221` renders
`<RegionCityFilter>`, both **unconditionally** (inside the `(!isDesktopStrict || mobileOpen)`
block, which is layout-only, not data-gated). The `advancedFilters` prune effect
(`Filters.tsx:86-94`) operates **only on `selectedFilters`** — it never touches
`selectedPriceRange` or `selectedRegionsAndCities`.

So "store hydrated but UI not rendered" is **ruled out for price/region**: the controls are
present on home and read directly from the store (`Filters.tsx:204,218`). If price/region show
empty on home, it is because the **store value is empty**, not because the control is missing.
(Contrast: the *category `f_<key>`* control genuinely isn't rendered on home, because
`advancedFilters` only includes `topFilters` + the current page's category filters,
`Filters.tsx:66-84` — that's the prior audit's separate UI-prune point.)

## Q6 — Cross-link to the region SEND-null bug. Shared root cause?

**Yes — single root cause, but it lives on the HYDRATE side, and the two paths use *inverse*
(mismatched) normalization strategies:**

- **HYDRATE read** (`FilterManager.tsx:185,189`): `fromQueryParam(urlValue)` compared to **raw**
  `t(labelKey)`. → **lossy / broken for accents.**
- **SEND build** (`filtersHelper.ts:228-238`, `getRegionAndCitiesData`): **raw** `urlValue`
  compared to `toQueryParam(t(labelKey))`. → **robust / correct** (normalizes the candidate the
  same way the URL was written).

They do **not** share a helper; they apply opposite transforms, and only the hydrate side is
wrong. The cascade that produces "selectedRegionAndCityValues arrives null at the backend":

1. HYDRATE drops accented region from the store (Q2).
2. SYNC (`FilterManager.tsx:270-282`) rebuilds the URL from the emptied store and `router.replace`s
   region/city **out of the URL** (`:321`).
3. The server refetch (`page.tsx:41/114` → `filterHydrationSSR` → `parseFiltersFromQueryParams` →
   `getRegionAndCitiesData`, `filtersHelper.ts:223-244`) now sees **no** `regions`/`cities`
   params → `{ regionIds: [], cityIds: [] }` → effectively a null region filter at the backend.

So the SEND path's own code is **correct**; it goes null because the broken hydrate **erased the
region from the URL upstream**. Fix the hydrate normalization and the SEND-null disappears too.
(This cascade through SYNC strip is the part that wants a 2-minute live confirm — see below.)

## Q7 — Minimal fix location + shape (NOT implemented). Same fix for price and region?

**Different fixes — they are different problems.**

- **PRICE:** no hydrate-read fix needed; `FilterManager.tsx:162-178` is correct and ungated. If a
  live repro proves price specifically blank on home, the fix is in **SYNC/mount timing**, not the
  read — investigate the SYNC-on-first-mount firing (`FilterManager.tsx:210`) and whether the
  store is empty at the home mount render. Do **not** "fix" the price read; it isn't broken.

- **REGION/CITY:** fix the normalization mismatch at the **hydrate read**, making it match the
  SEND path's robust strategy. Minimal shape at `FilterManager.tsx:185-194`: drop the
  `.map(fromQueryParam)` and compare the **raw** URL token to `toQueryParam(t(r.labelKey))` /
  `toQueryParam(t(c.labelKey))` — i.e. mirror `getRegionAndCitiesData` (`filtersHelper.ts:228-238`)
  exactly. That single change makes accented regions resolve into the store, which (via SYNC)
  keeps them in the URL, which fixes the SEND-null. One edit, both symptoms.
  - Adjacent: the **`f_<key>` SINGLE/MULTI option** hydrate (`FilterManager.tsx:121`) already uses
    the correct `toQueryParam(t(opt.labelKey))` strategy — so the inconsistency is isolated to the
    region/city read. Worth aligning region to that existing-correct pattern rather than inventing
    a new one.

- **CATEGORY `f_<key>` on `/`** (the prior audit's finding) is a **third, separate** decision
  (logo target vs. home filter scope) and is out of this brief's price/region scope — leave to the
  prior audit + Mastermind.

---

## Brief vs reality

I read the brief and the code. Before drawing conclusions I re-verified the price/region hydrate
path specifically (as the brief instructed, not trusting the prior audit). Two of the brief's load-
bearing assumptions are contradicted by the code:

1. **"Price is dropped on `/` behind a `categoriesFromPath` / `/catalog` gate."**
   - Brief says: price (and region) are swallowed by the same gate the prior audit blamed for
     `f_<key>`.
   - Code says: `FilterManager.tsx:162-178` (price) and `:185-194` (region) read from
     `window.location.search` and write the store **with no `pathname`/`categoriesFromPath`
     dependency**. The gate at `:63`/`:114` is consumed **only** by the `f_` loop (`:103-158`).
   - Why this matters: the proposed mental model ("one gate also swallows price/region") would send
     a fix to the wrong place. Price has no hydrate-read defect; region's defect is a value-matching
     bug, not a path gate.

2. **"Region fails to hydrate specifically on `/` (works on `/catalog`)."**
   - Brief says: region is `/`-specific.
   - Code says: region's failure (`fromQueryParam` vs raw `t(labelKey)`, `:185/:189`) is
     **path-independent** and triggers only for **accented** names — verified by round-trip
     (Niš/Čačak/Užice/Vrnjačka Banja fail; Beograd/Novi Sad pass). The apparent "works on catalog"
     is most plausibly the **retained singleton store** (product page doesn't remount the manager),
     not a successful catalog hydrate.

3. **"The price+region params stay in the URL."** The region path, once it drops the value, would
   **strip** region from the URL via SYNC `router.replace` (`:321`) — the opposite of "stay." This
   needs a live URL check to settle which param actually persists (Q4 note).

I have not changed any code (read-only audit). Please pass these to Mastermind: the region fix is
clear and high-confidence (Q7); the price-on-`/` symptom is **not** reproduced by the hydrate code
and needs the live confirmation below before any price-side change is scoped.

## Needs live confirmation (≤5 min, isolates the real culprit)

On `/` (home), with a **fresh** state each time, set exactly one thing in the URL and reload:

1. `/?from=100&to=500&currency=<code>` — does the **price** control populate on home? (Code says
   **yes**.) If no → it's a SYNC/mount-timing bug, not the read.
2. `/?regions=beograd` (ASCII) vs `/?regions=nis` (accented) — ASCII should populate, accented
   should **not** (confirms the `fromQueryParam` bug).
3. `/?f_<somecategorykey>=…` for a **category** filter — should **not** populate (prior audit).

Then the catalog→product→BACK→logo flow, watching the **settled URL** after landing on home, to
confirm whether region/price params are **stripped** (SYNC) or **kept**.

---

## Adjacent observations (Part 4b)

- **medium** — Region/city hydrate normalization mismatch, `FilterManager.tsx:185-194`: uses lossy
  `fromQueryParam` against raw `t(labelKey)`; SEND (`filtersHelper.ts:228-238`) uses the correct
  `toQueryParam` strategy. User-facing: accented regions never rehydrate and get stripped → backend
  region filter goes null. (This is the in-scope finding; logged here for completeness.)
- **low** — `FilterManager.tsx:185` / `:189`: even ignoring accents, `fromQueryParam`'s
  "capitalize each word" is a fragile reconstruction of display labels; comparing normalized→
  normalized (as the `f_` option path and SEND already do) is the robust idiom. Single source of
  truth would be one shared `matchByQueryParam(labelKey, urlToken)` helper used by hydrate **and**
  send.
- **low** — Pre-existing `react-hooks/exhaustive-deps` on both `FilterManager.tsx` effects
  (`:204`, `:323`), already flagged in `2026-06-04-oglasino-web-filter-mount-fix-1.md`. Touching the
  dep arrays would change hydrate/sync timing; out of scope.
