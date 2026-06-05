# Audit — Price filter renders empty on logo→home client nav

**Repo:** oglasino-web · **Branch:** dev · **Mode:** READ-ONLY (no code changes)
**Date:** 2026-06-05
**Bug:** On HOME at `/rs-sr?from=200&currency=eur` the Cena (price) filter control renders EMPTY,
yet the params are in the URL. Repro: home (filtered) → click product → click LOGO back to home →
filters not populated. Hard refresh of the same URL → filters populate.

---

## TL;DR — the brief's hypothesis is wrong; here is the real mechanism

The brief (Q2b) hypothesised a **reused FilterManager instance** whose `lastHydratedPathRef` still
held `'/'`, making the guard `ref === pathname` true and **skipping** hydrate. **That is not what
happens.** FilterManager **fresh-mounts** on product→home, so `lastHydratedPathRef` is `null` and the
hydrate effect **runs**. The bug is not a skipped hydrate. It is:

1. The product-card link **strips the filter query string** from the URL, so on the logo→home nav the
   home URL momentarily has **no params**.
2. Hydrate runs against that **param-less URL** and **clears** the (still-populated, in-memory) store
   → the control renders EMPTY.
3. The **SYNC effect**, firing on the same first commit with **stale render-closure values** (the
   store wasn't cleared until after render), writes those stale params **back into the URL** via an
   **async** `router.replace`.
4. Net stable state: **URL has params, store is empty, control is empty** — exactly the screenshot.

Hard refresh works because the URL carries the params *at mount time* and the store starts genuinely
empty (fresh JS context), so hydrate reads the params and populates.

Evidence and per-question walkthrough below.

---

## Key code references

- `src/components/client/initializers/FilterManager.tsx`
  - hydrate effect: lines **57–204**; guards at **58** (`if (!baseSite) return`) and **59**
    (`if (lastHydratedPathRef.current === pathname) return`); reads **`window.location.search`** at
    **94**; clears price at **170–178**; sets `lastHydratedPathRef.current = pathname` at **202**,
    `setHydrated(true)` at **203**; deps **`[baseSite, pathname]`** at **204**.
  - sync effect: lines **209–333**; guard at **210**
    (`if (lastHydratedPathRef.current !== pathname || !hydrated) return`); reads
    **`window.location.pathname + window.location.search`** at **294**; `router.replace(newUrl,…)`
    at **321**; deps include `selectedPriceRange`, `pathname`, `hydrated` (not searchParams) at
    **323–333**.
  - `lastHydratedPathRef` is a `useRef` at **31** → per-instance, `null` on every fresh mount.
- `src/components/client/initializers/SelectableFilterManagerWrapper.tsx:19` mounts
  `<FilterManager useFilterStore={usePortalFilterStore} />` for the portal scope.
- Home page renders the wrapper **in the page body**, not a layout:
  `app/[locale]/(portal)/(public)/page.tsx:64`.
- `src/lib/store/useFilterStore.ts:189` — `usePortalFilterStore` is a **module-level Zustand
  singleton** (no `persist` middleware, no reset on unmount). State (incl. `hydrated`, line **60/61**)
  survives client navigation; only a full reload re-initialises it.
- `src/lib/store/useBaseSiteStore.ts:12` — `baseSite` is likewise a singleton; once set it stays
  truthy across client nav.
- Product-card href strips the query string:
  `src/lib/utils/utils.ts:110-116` (`getNormalizedProductUrl` → `/product/${id}/${name}`, no query),
  used by `src/components/client/product/PortalProductCard.tsx:21`.
- Logo preserves the query string of the *current* page:
  `src/components/client/buttons/HomeLink.tsx:8-13` (`href={queryString ? '/?'+queryString : '/'}`).
- `usePathname` (i18n) strips the locale: home → `'/'`
  (`src/i18n/navigation-client.tsx:88-95`).
- SSR `filterHydrationSSR` (`FilterHydrationSSRInit.tsx:7`) only builds a `ProductsFilterDTO` for the
  product fetch — it **does not write the Zustand store**. The control's state comes *only* from
  FilterManager's client hydrate.

---

## Q1 — Mount boundary: fresh-mount or reused?

**FilterManager FRESH-MOUNTS on product→home. It is not kept alive across the nav.**

- FilterManager is rendered **inside the home page** (`page.tsx:64`), via
  `SelectableFilterManagerWrapper` — **not** in any layout. The layouts that *do* persist across the
  nav are `(public)/layout.tsx` (renders only `{children}` + `<Footer/>`) and `(portal)/layout.tsx`
  (Header/PortalMain/etc.) — neither mounts FilterManager.
- The product detail page (`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`)
  does **not** render FilterManager (confirmed by grep; consistent with prior audits).
- There are **no intercepting or parallel routes** under `app/[locale]` (no `(.)product`, no
  `@slot`, no `default.tsx` — verified by `find`). So the product page is an ordinary segment that
  **replaces** the home page in the tree.

Therefore: home→product **unmounts** the home `page.tsx` subtree (FilterManager unmounts);
product→home **mounts a brand-new** home `page.tsx` subtree (a fresh FilterManager instance). Because
`lastHydratedPathRef` is a `useRef` (FilterManager.tsx:31), the new instance starts with
`lastHydratedPathRef.current === null`.

**What persists across the nav is the Zustand store, not the component.** `usePortalFilterStore` is a
module singleton (useFilterStore.ts:189) with no unmount reset, so the price values *and* the global
`hydrated: true` flag set on the original filtered-home visit are still in memory when home re-mounts.

---

## Q2 — Walk the two cases of the hydrate guard

**(a) Hard refresh of `/rs-sr?from=200&currency=eur` — works. CONFIRMED.**
Fresh JS context → store re-initialised empty, `hydrated=false`; `baseSite` populated by app init;
`lastHydratedPathRef.current=null`. Hydrate effect: `baseSite` truthy (passes 58), `null !== '/'`
(passes 59), reads `window.location.search` = `?from=200&currency=eur` (the server-delivered URL) →
`setPriceRange({from:'200', selectedCurrency: eur})` → control populates. Sync then computes the same
URL → `newUrl === current` → no replace. Stable and correct.

**(b) Logo→home client nav — the "reused instance / ref skip" hypothesis is FALSE.**
FilterManager fresh-mounts (Q1), so `lastHydratedPathRef.current` is `null`, not `'/'`. The guard at
line 59 (`ref === pathname`) is **false** → it does **not** early-return → **hydrate runs**. The
guard at line 58 (`!baseSite`) is also false because `baseSite` is a persistent singleton. So
*neither* guard skips hydrate.

**What actually goes wrong (the "something else skips" path the brief asked us to pin):**
The product-card link strips the query (`getNormalizedProductUrl`, utils.ts:116), so the product URL
is `/rs-sr/product/123/foo` with **no params**. On that page `HomeLink` reads
`useSearchParams()` = empty, so the logo `href` is `'/'` (HomeLink.tsx:13). The user lands on
`/rs-sr` with **no query string**. Sequence on the fresh mount:

1. **render-0**: store still holds the stale `from=200/eur` (singleton survived the nav), so the
   control briefly shows the old value.
2. **hydrate effect** (runs first, FilterManager.tsx:57): reads `window.location.search` = `''` →
   `from=''`, `to=''`, `hasPrice=false` → `setPriceRange({from:'', to:'', free:false,
   selectedCurrency: undefined})` (lines 162–178) → **store price cleared** → control becomes EMPTY.
   Sets `lastHydratedPathRef.current = '/'` (202) and `setHydrated(true)` (203).
3. **sync effect** (runs second, same commit, FilterManager.tsx:209): guard at 210 now passes —
   `ref==='/'===pathname` and the global `hydrated` was already `true`. Its closure still holds the
   **render-0 stale price 200** (the store clear in step 2 hasn't re-rendered yet), so it builds
   `?from=200&currency=eur`, sees `newUrl !== current ('/rs-sr')`, and calls **async**
   `router.replace('/rs-sr?from=200&currency=eur')` (321).
4. **re-render** (store now empty): sync re-runs (its price dep changed to empty) and computes
   `newUrl='/rs-sr'`; but `window.location` has **not yet** updated (replace is async/pending), so
   `current` still reads `/rs-sr` → `newUrl === current` → **no replace** → it does not strip the
   pending params. The pending replace then lands → URL becomes `/rs-sr?from=200&currency=eur` and
   stays there (no pathname change, so hydrate's `[baseSite, pathname]` deps don't re-fire — see Q3).

**Stable end state: URL has params, store empty, control empty.** Matches the screenshot exactly.

So hydrate is **not** skipped — it runs and *clears* the store because the URL it reads is genuinely
param-less at that instant; the params reappear afterward from the stale-closure sync writeback.

---

## Q3 — Is `[baseSite, pathname]` the problem?

It is **half** the problem, and it matters for the fix more than for the trigger:

- On a **fresh mount**, the hydrate effect runs **once unconditionally** regardless of deps — so the
  deps array is *not* what makes hydrate run; the mount does. `pathname` is `'/'` from the first
  render; it doesn't "change."
- The deps array matters for **re-firing**: it watches `pathname` but **not the query string**.
  Hydrate reads the URL imperatively via `window.location.search` (line 94), not the reactive
  `useSearchParams()`. So when the async `router.replace` in step 3 above injects
  `?from=200&currency=eur` (search changes, pathname stays `'/'`), **hydrate does not re-run** and
  never re-reads the now-correct URL. That is precisely why the control stays empty even though the
  URL ends up holding the params.
- The same omission is in the sync effect's deps (323–333): it also keys on `pathname`, not search.

So the deps don't *cause* the initial empty read, but their omission of the search string is what
**prevents self-correction** once the URL regains the params.

---

## Q4 — Is the store empty, or stale, at the home mount?

Both, at different instants — and the final state is **empty**:

- **At render-0 of the fresh mount:** the store is **stale** — it still carries `from=200`,
  `currency=eur`, and `hydrated=true`, because the singleton survived the product nav and nothing
  resets it on unmount. (The control would flicker the old value for one frame.)
- **After the hydrate effect:** the store is **cleared to empty**, because hydrate read a param-less
  URL and called `setPriceRange({from:'', to:''…})`. This is the value the screenshot captures —
  **store empty, control empty.**
- The params in the URL are **not** a reflection of store state; they are the stale render-0 values
  written back by the sync effect's closure (Q2 step 3). So the precise framing is:
  **"store cleared by hydrate-against-an-empty-URL, then stale params re-written to the URL by sync"**
  — i.e. a hydrate-clears↔sync-rewrites divergence, *not* "store empty + hydrate skipped."

---

## Q5 — Minimal correct fix (no implementation; recommendation only)

**Goal:** returning to home should leave the price control populated and consistent with the URL,
with no hydrate↔sync loop.

### Option (b) — reset `lastHydratedPathRef` on unmount → **rejected, no-op.**
The instance fresh-mounts, so the ref is **already `null`** on the new instance. Resetting it on
unmount changes nothing. This option only matters under the (false) reused-instance hypothesis.

### Option (d) — preserve the active filter query on product navigation → **recommended (cleanest).**
Have the product-card link (`PortalProductCard`, reading `useSearchParams()` client-side and
appending to the href — **scoped to the card link only, not to `getNormalizedProductUrl`** so product
**canonical / structured-data URLs stay clean**) carry the current filter query string. Then the
product URL holds the params, `HomeLink` already forwards them (HomeLink.tsx:13), and the home URL has
the params **at mount time**. The client-nav path becomes **identical to the hard-refresh path that
already works**: hydrate reads the params, populates the store, sync finds `newUrl === current` and
does nothing.
- **No loop:** it never changes the hydrate/sync coordination — it just removes the param-less window
  that caused the bad hydrate read. Zero new race surface.
- **Cost:** product detail URLs now display filter params (cosmetic; confirm the product canonical is
  the clean URL — the SEO-foundation work set product canonicals, so this should already hold).
- **Out of FilterManager scope** — touches `PortalProductCard`. Flag for Mastermind to scope.

### Options (a)+(c) — search-aware re-hydrate inside FilterManager → **works, but more race surface.**
Make hydrate reactive to the query string: source params from `useSearchParams()` instead of
`window.location.search`, and replace the pathname-only key with a **full-URL key** (`pathname +
'?' + search`) in **both** the hydrate guard (59) and the sync guard (210), adding the search string
to **both** dep arrays.
- **Loop-avoidance (the critical part):** the current design avoids looping purely because hydrate is
  keyed on `pathname` and so never re-runs when sync rewrites the *search*; making hydrate
  search-reactive **breaks that asymmetry** and *would* reintroduce the loop unless the guard is
  generalised. The fix is a **single shared "last reconciled URL" ref written by *both* effects**:
  hydrate sets it after hydrating; **sync sets it to the URL it is about to write, immediately before
  `router.replace` (321).** Then the search-change that sync causes is seen by the now-search-reactive
  hydrate as `currentKey === lastKey` → hydrate early-returns → no re-hydrate. Only **externally**
  driven URL changes (logo, back/forward, fresh load) carry a key not equal to `lastKey`, so only
  those trigger a re-hydrate. The existing `if (newUrl !== current)` check (296) remains the second
  loop-breaker, and because hydrate/sync are inverses (same `toQueryParam`/labelKey serialisation)
  the round-trip is a fixpoint.
- **Caveat:** the async `router.replace` + reactive-`useSearchParams` lag interacts with the
  stale-closure sync writeback (Q2 steps 3–4); the shared-key guard must be written carefully to
  avoid a transient flip. This is why (d) — which deletes the param-less window entirely — is the
  safer recommendation.

**Recommendation:** Option **(d)** as primary (achieves the user-expected "filters kept", makes
client-nav == the working hard-refresh path, no new race), with **(a)+(c) + shared-key ref** as the
in-component alternative if Mastermind prefers to keep the fix inside FilterManager. Reset-on-unmount
(b) is a no-op and should not be pursued. If instead the product decision is "bare home = no filters",
the consistent-but-filter-dropping fix is to reset the store **and** `hydrated` on FilterManager
unmount — but that contradicts the user's "filters not populated" complaint, so it is not
recommended.

---

## Cross-repo / contract notes

- No backend or wire-shape involvement; this is entirely a web client-state/navigation bug.
- The SSR product list (driven by the URL via `filterHydrationSSR`) will show **filtered** results in
  the bad state (URL has `from=200`) while the control shows **no** filter — a real user-visible
  inconsistency, not only a cosmetic one. The fix should make control and URL agree.
