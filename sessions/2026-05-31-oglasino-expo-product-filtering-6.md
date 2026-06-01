# Session summary — Expo product-filtering (session 6)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (stayed; no commit/push/branch switch)
**Date:** 2026-05-31
**Task:** B8 + MORE_FROM_SELLER swap + applyRandom suppression (user side) — three fixes on
the public user-products screen path (`UserProductsList.tsx`).

## Headline

Fixes 1 (B8 — stop reading the portal store) and 3 (applyRandom suppression / no-guard case)
shipped exactly as briefed. **Fix 2 (RANDOM_PRODUCTS → MORE_FROM_SELLER with excludeIds) was
NOT implemented** — it is blocked on a brief-vs-reality mismatch the brief itself told me to
report rather than force. Igor (this session) decided to **kick Fix 2 to Mastermind** as a
platform-gotcha. Details in "Brief vs reality" and "For Mastermind" below. Scope held to a
single file (`UserProductsList.tsx`); the factory and its callers were left untouched because
the factory-signature change existed only to serve Fix 2.

## Brief vs reality

1. **Fix 2 — the initial-page ids are NOT in hand in `UserProductsList` (blocker)**
   - Brief says: "the audit indicates the main list's products are in hand in this component,
     so this should be reachable" — pass `MORE_FROM_SELLER(userDetails.id, <initial-page ids>)`.
   - Code says: `UserProductsList` never holds the products. It passes only `fetchPage` and a
     statically-memoized `extraSections` down to `ProductList`. The products live in
     **`ProductList`'s own `useState`** (`ProductList.tsx:51`), populated asynchronously inside
     `loadNextPage` (`ProductList.tsx:83-85`). There is no callback exposing them back up to the
     parent, and `extraSections` is built once at mount (`useMemo(..., [])`) before any fetch
     occurs. `ProductList.tsx` is explicitly out of scope per the brief ("Touch nothing else").
     The audit (`audit-product-filtering-expo-current.md` Q4, lines 181-205) actually says the
     products live in `ProductList` and the extra section renders only when `!hasMore` — it does
     **not** say `UserProductsList` holds them.
   - Why this matters: the exclude-ids cannot be sourced at the construction point without an
     out-of-scope change to the shared `ProductList` component.
   - Recommended resolution: Mastermind design decision (see #3 below) — accepted by Igor; Fix 2
     deferred.

2. **Fix 2 — `excludeIds` is `number[]`, not `string[]` (minor; would have followed code)**
   - Brief says: extend the factory to `excludeIds: string[]`; "`excludeIds: string[]` already
     exists on `ProductsFilterDTO`."
   - Code says: `ProductsFilterDTO.excludeIds?: number[]` (`ProductsFilterDTO.ts:15`), and the
     current factory takes `excludeProductId: number` (`productExtraSections.ts:83`). Product ids
     are numeric throughout (e.g. `MORE_FROM_SELLER(productDetails.ownerId, productDetails.id)` at
     `product/[...productData].tsx:251`). `string[]` would have broken `tsc` and the wire shape.
   - Why this matters: had Fix 2 proceeded, the signature would have been `excludeIds: number[]`,
     not `string[]`. Recorded so the eventual Fix 2 chat uses the right type.

3. **Fix 2 — bottom-mounted MORE_FROM_SELLER is duplicate-or-empty on mobile (platform gotcha)**
   - Brief says: swap in `MORE_FROM_SELLER` to stop the seller's own products being duplicated by
     the site-wide `RANDOM_PRODUCTS` carousel (web parity).
   - Code says: the extra section renders **only when `!hasMore`** (`ProductList.tsx:214`), i.e.
     after infinite scroll has exhausted **every** product from that seller into the main list.
     `MORE_FROM_SELLER` filters by the same `ownerId`, so by the time it renders it can only return
     the seller's products that are already in the main list. Excluding the first page leaves
     pages 2+ as duplicates; excluding **all** loaded ids returns empty. Web's design works because
     web shows one paginated page at a time and "more from seller" surfaces *other* pages; mobile's
     exhaustive infinite scroll has nothing genuinely "more" to show beneath it.
   - Why this matters: even with the ids reachable, the swap as specified yields a
     duplicate-only (or empty) carousel — arguably worse than today's site-wide random.
   - Recommended resolution: Mastermind decides whether the seller page needs a second section on
     mobile at all, or a different mount/exclude model. Accepted by Igor; deferred.

## MORE_FROM_SELLER callers (found, deliberately NOT touched)

The brief asked me to grep every caller before changing the signature. I did, so the eventual
Fix 2 chat has the list ready:

- **Definition:** `src/lib/extra-section/productExtraSections.ts:83` —
  `MORE_FROM_SELLER = (ownerId: number, excludeProductId: number)`.
- **Caller 1:** `app/(portal)/(public)/product/[...productData].tsx:10` (import), `:251`
  (`MORE_FROM_SELLER(productDetails.ownerId, productDetails.id)`) — the product-detail page's
  "more from this seller" section. If/when the signature becomes `excludeIds: number[]`, this
  call becomes `MORE_FROM_SELLER(productDetails.ownerId, [productDetails.id])`.

No callers were modified this session — the signature was not changed because Fix 2 is deferred.
The factory file (`productExtraSections.ts`) was not touched at all.

## How I sourced the initial-page ids for the exclude

I could not. What IS available at the `extraSections` construction point in `UserProductsList`:
- `userDetails.id` (the ownerId) — available.
- The first-page product ids — **not available**; they are inside `ProductList`'s internal
  `products` state with no upward callback, and `ProductList` is out of scope. (See Brief vs
  reality #1.)

## Implemented

### Fix 1 — B8: stop reading the portal filter store (`UserProductsList.tsx`)

Removed the entire `usePortalFilterStore(useShallow(...))` read block. The five store fields read
were exactly: `searchText`, `selectedFilters`, `selectedOrder`, `selectedPriceRange`,
`selectedRegionsAndCities`. **All five were used only in the request body** (the `filtersData`
`useMemo`, old `:38-64`) — confirmed by reading the whole component; none fed any other logic,
effect, or render branch. The main-list body is now driven solely by the owner:

**Before:**
```ts
{ ownerId: userDetails.id, applyRandom: true, searchText, orderBy: selectedOrder,
  priceRange: selectedPriceRange, selectedRegionsAndCities: {...}, filters: [...] }
```
**After:**
```ts
{ ownerId: userDetails.id, applyRandom: true }
```
(`fetchPage` still adds `randomSeed` per page, unchanged — so the wire body is
`{ ownerId, applyRandom: true, randomSeed }`.)

Removed the now-dead imports `usePortalFilterStore` and `useShallow`. The `useMemo` dependency
array dropped to `[userDetails]`.

### Fix 3 — applyRandom/randomSeed suppression (user side): no guard needed

Confirmed the **"no guard needed"** case (the brief's likely outcome). After Fix 1 the user list
carries no `searchText` and no `orderBy` from anywhere — the body is `{ ownerId, applyRandom }`
only — so there is no search/order to suppress against. Per the brief I did **not** add a dead
guard reading the store fields I just removed; `applyRandom: true` remains as the seller-listing
default ordering. No `(x ?? '').trim()` is needed because no possibly-undefined value is referenced
anywhere in the component — the `undefined.trim()` class of bug (session 5's fix in the sibling
`FilteredProductList`) cannot recur here.

### Fix 2 — deferred (not implemented). See "Brief vs reality" + "For Mastermind".

`RANDOM_PRODUCTS` remains mounted on the user screen (`UserProductsList.tsx:31`). The brief said
"do not leave RANDOM_PRODUCTS mounted" — that instruction is part of the Fix 2 swap, which Igor
deferred this session, so RANDOM_PRODUCTS stays until Mastermind resolves the platform-gotcha.

## Files touched

- `src/components/product/UserProductsList.tsx` (−39 / −0 net; two edits: imports + the
  store-read/`filtersData` block). No other file touched.

## Tests

- `npx tsc --noEmit` — 0 errors (clean), before and after.
- `npx expo lint` — 80 problems (0 errors / 80 warnings), before and after (no new findings).
  `npx eslint` on `UserProductsList.tsx` alone: **0 problems** after the edit.
- `npx vitest run` — 24 files / 325 tests passed, before and after.

| Check | Before | After |
|---|---|---|
| `npx tsc --noEmit` | 0 errors | 0 errors |
| `npx expo lint` | 80 (0 err / 80 warn) | 80 (0 err / 80 warn) |
| `npx vitest run` | 24 files / 325 tests | 24 files / 325 tests |

## Cleanup performed

- Removed the dead `usePortalFilterStore` and `useShallow` imports left by Fix 1.
- Removed the five store-field reads and their use in the request body.
- No commented-out code, no debug logging, no TODO/FIXME added. Verified all remaining imports in
  `UserProductsList.tsx` are still used (`RANDOM_PRODUCTS` is still consumed by `extraSections`).

## Obsoleted by this session

Nothing. (Fix 1 removed live store-bleed plumbing, but that code was functioning, not dead — its
removal is the fix, not the obsoleting of something already dead. The factory and its callers are
untouched and remain live.)

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change this session. **Closure-gate note:** chat-D is NOT complete — Fix 2 is
  deferred to a Mastermind design decision — so the chat-D / product-filtering row in `state.md`'s
  Expo backlog table must **not** be flipped toward `mobile-stable` on the basis of this session.
  No `state.md` edit is required *from this session*; the row stays as-is pending the Fix 2
  resolution. Stated explicitly per the closure gate: no unstated config-file dependency.
- issues.md: no change made by me (Docs/QA is sole writer). I have **drafted** a suggested
  issues.md entry for the deferred Fix 2 in "For Mastermind" — Mastermind decides whether to file
  it.

## Conventions check

- **Part 4 (cleanliness):** confirmed. No commented-out code, no unused imports/vars (dead
  `usePortalFilterStore`/`useShallow` removed), no debug logging, no TODO/FIXME. tsc/lint/test
  evidence above.
- **Part 4a (simplicity):** see structured evidence in "For Mastermind".
- **Part 4b (adjacent observations):** flagged in "For Mastermind".
- **Part 6 (translations):** N/A this session — no user-visible strings added or changed.
- **Part 7 (error contract):** N/A — no error-handling path touched.
- **Part 8 (architectural defaults) / Part 11 (trust boundaries):** confirmed. The user list still
  forwards a `ProductsFilterDTO` to the server (`getPortalProducts`) and makes no local trust
  decision; removing the store reads narrows the body to `{ ownerId }`, server-enforced as before.
- **Slug discipline:** session slugged `product-filtering` (= the slug existing sessions carry),
  order 6 (prior: -1, -2, -3, -4, -5).

## Known gaps / TODOs

- **Fix 2 not implemented** (RANDOM_PRODUCTS → MORE_FROM_SELLER swap). Deferred to Mastermind by
  Igor's decision this session. No TODO/FIXME comment was added to the code; this is the matching
  session-summary entry. RANDOM_PRODUCTS remains mounted at `UserProductsList.tsx:31`.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - *Added (earned complexity):* nothing. No new abstraction, helper, type, or config value.
  - *Considered and rejected:* (1) adding an `onInitialProductsLoaded` callback to `ProductList`
    so `UserProductsList` could capture first-page ids — rejected as out-of-scope (touches a
    shared, perf-sensitive component the brief forbids) and because it still wouldn't fix the
    duplicate-or-empty semantics (Brief vs reality #3); (2) `MORE_FROM_SELLER(userDetails.id, [])`
    with an empty exclude — rejected because, given the `!hasMore` mount, it duplicates the
    seller's own exhausted main list (worse than today); (3) a null-safe guard in Fix 3 — rejected
    because after Fix 1 there is no possibly-undefined value to guard (no dead guard, per brief).
  - *Simplified or removed:* deleted the five-field portal-store read and its `useShallow`
    subscription from `UserProductsList`; the request body shrank from 7 fields to 2 (`ownerId`,
    `applyRandom`), the `useMemo` dep array from 7 entries to 1.

- **Fix 2 is a genuine web→mobile design gap, not a quick swap.** The brief's swap assumes web's
  paginated single-page main list. On mobile, `ProductList` is exhaustive infinite scroll and only
  mounts the extra section at `!hasMore`, so a same-`ownerId` "more from seller" section beneath it
  is structurally duplicate-or-empty regardless of the exclude. The two sub-problems (ids not
  reachable in-scope; duplicate-or-empty semantics) both point at the same conclusion: this needs a
  design decision, not a one-line swap. Options for Mastermind:
  1. Drop the second section on the seller page entirely on mobile (the main list already shows the
     whole seller catalog — arguably the cleanest mobile answer).
  2. Keep a section but change what it shows (e.g. site-wide RANDOM_PRODUCTS *with* the seller's
     loaded ids excluded — "discover other sellers", not "more from this one"), which would require
     a `ProductList` callback to expose the loaded ids upward (scope expansion).
  3. Implement the web-literal exclude (requires the `ProductList` callback AND accepts
     pages-2+ duplicates) — not recommended.

- **Suggested `issues.md` entry (draft — Mastermind/Docs decide whether to file):**
  > **2026-05-31 — Mobile seller-page "more from seller" section is structurally duplicate-or-empty**
  > **Severity:** medium · **Status:** open
  > **Found in:** `oglasino-expo` `src/components/product/UserProductsList.tsx:31` (mounts
  > `RANDOM_PRODUCTS`); `src/components/product/ProductList.tsx:214` (extra section renders only at
  > `!hasMore`).
  > **Detail:** chat-D Fix 2 (swap `RANDOM_PRODUCTS` → `MORE_FROM_SELLER` with `excludeIds`) could
  > not be implemented as briefed. (a) The first-page ids aren't reachable in `UserProductsList` —
  > products live in `ProductList`'s internal state with no upward callback, and `ProductList` is
  > out of scope. (b) Even with the ids, the section renders only after infinite scroll exhausts the
  > seller's catalog into the main list, so a same-`ownerId` section beneath it returns duplicates
  > (exclude first page) or nothing (exclude all). Web's paginated model doesn't translate to
  > mobile's exhaustive infinite scroll. Needs a design decision (drop the section, repurpose it, or
  > expand scope to add a `ProductList` ids callback). `RANDOM_PRODUCTS` left mounted in the
  > meantime. Fixes 1 (B8 store-bleed) and 3 (applyRandom no-guard) shipped this session.

- **Adjacent observation (Part 4b) — none new in the touched file.** `UserProductsList` is clean
  after the edit (0 lint problems). Pre-existing repo-wide warnings (lint stayed at 80) are tracked
  elsewhere and were not in scope.

## Definition of done checklist

- [x] Fix 1: `UserProductsList` no longer reads `usePortalFilterStore`; main list driven by
      `{ ownerId, applyRandom: true }` (+ per-page `randomSeed`). No portal-filter bleed-through.
- [x] Fix 3: confirmed "no guard needed"; no dead store-field guard added; no `undefined.trim()`.
- [ ] Fix 2: **deferred to Mastermind** (Igor's decision this session). RANDOM_PRODUCTS left
      mounted; factory and its one caller untouched and reported above.
- [x] `npx tsc --noEmit` clean.
- [x] `npx expo lint` — no new findings (80 → 80).
- [x] `npx vitest run` — green (325).
- [x] Session summary written to both `.agent/` files with Brief-vs-reality, the MORE_FROM_SELLER
      caller list, and the ids-sourcing report.
