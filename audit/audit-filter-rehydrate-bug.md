# Audit — filter-rehydrate-bug (logo-nav home lands with filter params in URL but filter UI empty)

**Repo:** oglasino-web · **Branch:** dev · **Mode:** READ-ONLY (no code changed)
**Date:** 2026-06-05

---

## TL;DR — root cause

The bug is **not** a mount-relocation regression and **not** a `lastHydratedPathRef` /
`hydrated`-flag suppression. It is a **navigation-target + filter-scope** mismatch:

- The Oglasino logo (`HomeLink`) **always navigates to `/` (home)** and copies the **current
  query string verbatim** onto it (`HomeLink.tsx:12-13`).
- Home's `FilterManager` HYDRATE only builds **category context** when
  `pathname.startsWith('/catalog')` (`FilterManager.tsx:63`). On `/` there is no category
  context, so any **category-scoped** filter param (`f_<key>` for a filter that lives under a
  catalog category, not in `baseSite.catalog.topFilters`) **fails the lookup and is dropped**
  (`FilterManager.tsx:108-114`, the `if (!filter) return` at line 114).
- The home Filters UI independently only renders `topFilters` + the *current* page's category
  filters, and actively **prunes** any selected filter not in that set
  (`Filters.tsx:66-94`).

The **BACK button works** because the history pop returns to the **catalog URL that owns the
category context** (`/catalog/<cat>?f_<key>=…`), where HYDRATE's path-derived
`categoriesFromPath` resolves the same key. The divergence is the navigation **target**, not a
remount/ref difference — both routes remount fresh.

**The trigger is a *category-scoped* filter applied on a catalog page, then clicking the logo.**
A top-level filter (`baseSite.catalog.topFilters`) does **not** reproduce it — top filters
resolve on `/` too.

> **One discrepancy with the brief's wording — flagged, needs a 2-min live confirm.** The brief
> says the params *stay* in the URL. The code I read would, after HYDRATE drops the unresolved
> key, fire the SYNC effect (`FilterManager.tsx:209-322`) and/or the Filters prune effect
> (`Filters.tsx:86-94`) and **`router.replace` the orphaned `f_<key>` param out of the URL** —
> i.e. the code tends to *strip* it, not keep it. The user-visible conclusion is identical
> either way (home cannot show the category filter), but the exact URL end-state should be
> confirmed on a live repro. See Q4 note.

---

## Evidence by question

### Q1 — What does the logo button do?

`Header.tsx:19-22` wraps the `OglasinoIcon` in `<HomeLink>`.

`HomeLink` (`src/components/client/buttons/HomeLink.tsx`):

```tsx
const searchParams = useSearchParams();
const queryString = useMemo(() => searchParams?.toString() || '', [searchParams]);
return (
  <Link href={queryString ? `/?${queryString}` : `/`} ...>
```

- It is a **next-intl `<Link href>`** (`@/src/i18n/navigation`), i.e. a **client push** with
  automatic locale prefixing — not `router.replace`, not a full navigation.
- Target is **always `/` (home)**, never the originating catalog route.
- It **carries the query string** — it copies `useSearchParams()` of *the page you click it on*
  verbatim onto `/`. So clicking the logo **from a filtered catalog page** produces
  `/<locale>?f_<key>=…` (home + the catalog's params).

`pathname` is locale-stripped (confirmed by `FilterManager.tsx:291-292`,
`pathname === '/' ? '' : pathname` then re-prepending `locale`), so home's `pathname` is `/`.

Note on the "navigate to a product page" step in the brief: product links are built by
`getNormalizedProductUrl` → `"/product/<id>/<name>"` with **no query string**
(`utils.ts:110-117`). So a product page has empty `searchParams`, and the logo there renders
`href="/"` (no params). The params-in-URL symptom therefore requires the logo to be clicked
**from a page that still carries the filter params** — i.e. the catalog (or home) itself, e.g.
after pressing BACK from the product onto the catalog, then clicking the logo.

### Q2 — How does HYDRATE trigger on logo→home?

HYDRATE keys on `[baseSite, pathname]` (`FilterManager.tsx:204`).

- **(a) Fresh-mount?** YES. `/` and `/catalog/[[...slugs]]` are distinct route segments under the
  shared `(portal)/(public)/layout.tsx`. Post-relocation the engine is mounted in the *page*
  (`app/[locale]/(portal)/(public)/page.tsx:64`), not the layout, so navigating
  catalog → `/` unmounts the catalog page subtree (and its `FilterManager`) and mounts a **new**
  home `FilterManager`. `lastHydratedPathRef` is a `useRef` (`FilterManager.tsx:31`) → resets to
  `null` on the new instance.
  *(Edge case: clicking the logo while already on `/` is a query-only change to the same route →
  no remount. That is a no-op same-URL case, not the bug.)*
- **(b) Does HYDRATE re-run?** YES. On the fresh mount `baseSite` is present (module-level
  `useBaseSiteStore`, already populated) and `lastHydratedPathRef.current (null) !== pathname
  ('/')`, so the guards at `FilterManager.tsx:58-59` pass and the effect body runs.
- **(c) Is `lastHydratedPathRef` / `hydrated` suppressing the re-hydrate?** **NO.** The fresh
  mount resets `lastHydratedPathRef` to `null`, so the `ref === pathname` guard (line 59) does
  not fire. And the persisted `hydrated === true` store flag does **not** gate HYDRATE at all —
  HYDRATE never reads `hydrated` (only SYNC does, line 210). So the brief's suspected suppression
  mechanism is **not** what's happening.

HYDRATE runs — it just **cannot resolve a category-scoped `f_<key>` on `/`**:
`categoriesFromPath` stays empty (`FilterManager.tsx:61-92` only populates it under
`/catalog`), so the filter lookup `topFilters.find || topCategory…find || subCategory…find ||
finalCategory…find` (lines 108-112) misses, and line 114 `if (!filter) return` drops it.
`setFilters(filtersToAdd)` (line 160) is then called with the category filter **absent**.

### Q3 — Why BACK re-hydrates but logo doesn't (the exact divergence)

Both are client navigations and both remount the destination page fresh — so the divergence is
**not** "history pop vs push" and **not** mount lifecycle. The divergence is the **target route**:

| Action | Lands on | `pathname` at HYDRATE | `categoriesFromPath` | Category `f_<key>` resolves? |
|---|---|---|---|---|
| BACK   | `/catalog/<cat>?f_<key>=…` (originating route) | `/catalog/<cat>` | built from path (`:61-92`) | **YES** (`:110-112`) → UI populated |
| LOGO   | `/?f_<key>=…` (always home)                    | `/`              | empty (guard `:63` false)  | **NO** (`:114` drops it) → UI empty |

So BACK returns you to the route that *owns* the category context; the logo sends you to `/`
which lacks it. Same HYDRATE code, different `pathname` → different resolution outcome.

### Q4 — Which of the three failure modes?

It is the **middle one: HYDRATE runs but its category-filter branch is effectively a no-op on
`/`** — the key is dropped at `FilterManager.tsx:114`. Specifically **not**:

- *not* "logo nav doesn't trigger HYDRATE" — it does (Q2b);
- *not* "home doesn't mount the FilterManager" — it does
  (`app/[locale]/(portal)/(public)/page.tsx:64`).

Reinforced by the home Filters UI, which builds its control set from `topFilters` + the current
page's category filters only and **removes** any `selectedFilters` entry not in that set:
`Filters.tsx:86-94` calls `addRemoveOptionFilter(filterOption.filter, undefined)` (a removal — see
`useFilterStore.ts:74-76`). So even a value left in the store would be pruned and never rendered
on `/`.

**URL end-state caveat (flag):** after the category key is dropped, `selectedFilters` settles
empty, and the SYNC effect (`FilterManager.tsx:209-322`, gate at 210, `router.replace` at 321)
rebuilds the URL from the now-empty store → it would **strip** the orphaned `f_<key>` param. The
Filters prune effect compounds this. By static reading the param is therefore *removed*, which is
slightly different from the brief's "params still in URL." Recommend confirming the exact
settled URL on a live repro (catalog → apply a **category** filter → BACK isn't needed; just
click the logo) — the discrepancy may be a transient the brief author caught mid-settle. It does
not change the root cause or the user-visible "home filter UI not populated."

### Q5 — Regression from the mount relocation? **No — pre-existing.**

The two code facts that cause the bug are untouched by the relocation:

1. The logo target + query-copy (`HomeLink.tsx`) — unrelated to FilterManager mounting.
2. The category-context gate `pathname.startsWith('/catalog')` (`FilterManager.tsx:63`) and the
   drop-on-miss at line 114 — unchanged by the relocation (the mount-fix session only deleted
   `isAllowedPath` and moved the mount from 3 layouts to 5 pages; see
   `.agent/2026-06-04-oglasino-web-filter-mount-fix-1.md`).

Under the **old** layout-level mount the `FilterManager` stayed continuously mounted across
catalog → product → home, but the HYDRATE effect still re-ran on every `pathname` change (deps
`[baseSite, pathname]`) and still hit the same `pathname.startsWith('/catalog')` gate. On landing
at `/` with a catalog category param it would have **resolved no category context and dropped the
key in exactly the same way**. The relocation changed *mount lifecycle* (continuous vs
per-page), but the category-resolution logic and the logo target are identical, so the
user-visible outcome on logo→home is the same before and after. **Not a regression.**

---

## Fix-direction notes (not implemented — audit only)

Out of scope to fix here, but the resolution space, for Mastermind:

1. **Make the logo strip filter params** (target `/` with no query) — simplest; matches the
   intuition that "go home" clears the filtered view. Changes UX (home becomes unfiltered).
2. **Make the logo preserve only home-resolvable params** (drop `f_<key>` for non-top filters
   before building the href) — preserves top-filter/search/region/price/order, drops what home
   can't show.
3. **Make home able to resolve all base-site filters** (resolve `f_<key>` against the union of
   `topFilters` + every category's filters, not just path-derived context) — biggest blast
   radius; also affects what the home filter panel renders.

Decision is product/UX, not mechanical — hand to Mastermind.

---

## Adjacent observations (Part 4b)

- **`Filters.tsx:86-94` prune effect** silently removes store filters not present on the current
  page. Combined with the shared portal store across `/` and `/catalog`, this means *navigating
  from catalog to home actively discards* category selections from the store, not just hides
  them. Relevant to any fix that tries to "keep" filters across the logo hop. Severity: medium
  (it's load-bearing behavior a fix must account for). Not fixed — audit only.
- **Pre-existing `react-hooks/exhaustive-deps` warnings** on both `FilterManager.tsx` effects
  (already flagged in `.agent/2026-06-04-oglasino-web-filter-mount-fix-1.md`). Touching the dep
  arrays would change hydrate/sync timing. Severity: low. Not fixed.
</content>
</invoke>
