# Audit — Store-rehydrate regression on return to `/` and `/catalog`

**Repo:** oglasino-web · **Branch:** dev · **Mode:** READ-ONLY (no code changes)
**Date:** 2026-06-05
**Confirmed requirement (Igor):** product URLs stay CLEAN (no filter params). The filter selection
lives in the STORE (module-level Zustand singleton, survives client nav). On returning to `/` or
`/catalog` from a product page, the page must REHYDRATE FROM THE STORE — repopulate BOTH the filter
controls AND the URL from the store — NOT clear the store. "It worked like that BEFORE." Find the
regression.

---

## TL;DR — verdict

**Yes — the regression is the FilterManager mount relocation** (`2026-06-04-oglasino-web-filter-mount-fix-1.md`),
and it is currently **uncommitted on `dev`** (working tree only — `HEAD` still has the BEFORE wiring).
The relocation changed TWO load-bearing things at once:

1. **Mount level: 3 LAYOUTS → 5 PAGES.** Before, `FilterManager` was mounted in `(public)/layout.tsx`.
   The product page lives **inside** that same layout group, so the layout — and the `FilterManager`
   instance inside it — **persisted, never unmounting, across home → product → home.** After, it is
   mounted in `page.tsx`; the product page does not mount it, so home → product **unmounts** it and
   product → home **fresh-mounts** a new instance.
2. **`isAllowedPath()` removed.** Before, both effects early-returned on non-filter routes (the product
   page was not in the allowlist), so navigating onto the product page never ran HYDRATE/SYNC and never
   touched the store.

These two together produced the BEFORE "rehydrate from store" behavior **as an emergent property**: a
persistent instance whose `lastHydratedPathRef` stayed `'/'` skipped the clearing HYDRATE on return, and
SYNC then pushed the surviving store back to the URL. Removing both at once deleted that property. The
prior audit (`audit-hydrate-skip-clientnav.md`) correctly pinned the AFTER mechanism (fresh mount →
HYDRATE-against-empty-URL clears the store → stale-closure SYNC re-writes params). This audit pins the
BEFORE behavior and the exact regression point.

---

## Git evidence (the relocation is uncommitted; HEAD = BEFORE)

- `HEAD:app/[locale]/(portal)/(public)/layout.tsx` mounts `<SelectableFilterManagerWrapper portalScope="portal" />`
  (line 11). `HEAD:app/[locale]/admin/layout.tsx:29` and `HEAD:app/[locale]/owner/layout.tsx:30` mount the
  admin/owner scopes. → **At HEAD, FilterManager mounts in the three LAYOUTS.**
- `git grep -n "SelectableFilterManagerWrapper" HEAD -- app/` returns **only the three layouts** — no page
  mounts at HEAD.
- Working tree (current): the three layout mounts are **gone**; the wrapper is now in five PAGES
  (`(public)/page.tsx:64`, `(public)/catalog/[[...slugs]]/page.tsx:185`, `owner/products/page.tsx:45`,
  `admin/products/page.tsx:43`, `admin/products/[userId]/page.tsx:48`). Matches the filter-mount-fix
  session summary exactly.
- `HEAD:src/components/client/initializers/FilterManager.tsx` still contains `isAllowedPath()` (function at
  line 54; guards at HYDRATE line 79 and SYNC line 232). The working-tree `FilterManager.tsx` has **no**
  `isAllowedPath` (deleted by the same session).
- `git log --oneline -- "app/[locale]/(portal)/(public)/layout.tsx"` → last commit `73ab239 "Pre production
  commit"`; the relocation has **no commit** — it is staged-on-disk only (Igor commits). So `HEAD` faithfully
  represents the BEFORE state and is directly inspectable.

**Product page is inside the FilterManager-mounting layout (the linchpin):**
`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` sits under
`(portal)/(public)/`, sharing `(public)/layout.tsx` with home (`(public)/page.tsx`) and catalog
(`(public)/catalog/[[...slugs]]/page.tsx`). In the App Router a layout persists across navigation between
its children and does not remount — so the layout-mounted `FilterManager` was continuously mounted across
home ↔ product ↔ catalog.

**Store is a singleton that survives nav (and `hydrated` with it):**
`src/lib/store/useFilterStore.ts:46` (`create<FilterState>(...)`, no `persist`, no unmount reset);
`:189` `usePortalFilterStore = createFilterStore(false, false)` — module-level. `hydrated` is a store field
(`:22`, init `false` at `:60`, setter `:61`). It survives client nav but resets to `false` on hard refresh
(fresh JS context). This is the key discriminator used in Q3.

---

## Q1 — Under the OLD layout-level mount, was FilterManager CONTINUOUSLY MOUNTED across home → product → home?

**YES. Confirmed by code + git.**

- The product page is a child of `(public)/layout.tsx` (path evidence above). At HEAD, that layout mounts
  `FilterManager` (`HEAD:(public)/layout.tsx:11`). Next.js preserves a shared layout across navigation
  between its children — it does not remount — so the `FilterManager` instance **did not unmount** on the
  product page. The product page merely swapped the layout's `{children}`.
- Because the instance persisted, its `lastHydratedPathRef` (`FilterManager.tsx:31`, a `useRef`) and the
  singleton store both stayed intact across the round trip.

Walk the BEFORE round trip against `HEAD:FilterManager.tsx`:

1. **On filtered home (`/`):** HYDRATE runs (baseSite ok `:78`; `isAllowedPath('/')` true `:79`;
   `ref(null) !== '/'` `:80`), populates the store, sets `lastHydratedPathRef.current = '/'` (`:223`),
   `setHydrated(true)`. SYNC keeps the URL synced.
2. **home → product (`/product/...`):** `pathname` changes → HYDRATE re-fires (deps `[baseSite, pathname]`).
   Guard `:79` `!isAllowedPath()` — the product path is **not** in the allowlist (`exactAllowed = {'/',
   '/owner/products'}`, plus `/admin/products*`, `/catalog*` — see `HEAD:FilterManager.tsx:54-72`) → **early
   return BEFORE line 223.** So `lastHydratedPathRef` **stays `'/'`** and the store is **untouched.** SYNC
   also early-returns (`:231` `ref('/') !== '/product/...'`; and `:232` isAllowedPath). **Nothing clears the
   store on the product page.**
3. **product → home (`/`):** same persistent instance. HYDRATE re-fires: baseSite ok; `isAllowedPath('/')`
   true; **guard `:80` `lastHydratedPathRef.current('/') === pathname('/')` → TRUE → EARLY RETURN.** HYDRATE
   does **not** run → **store is NOT cleared.** Then SYNC fires (`:231` `ref('/') === '/'` and `hydrated`
   already true; `:232` ok) → it builds the URL **from the still-populated store** and `router.replace`s it.
   The filter controls render from the surviving store.

**Net BEFORE:** store survives, no clearing HYDRATE on return, and SYNC (store → URL) repopulates the URL
and controls. **That is exactly Igor's required behavior** — achieved emergently by (persistent instance →
`lastHydratedPathRef` stays `'/'`) + (`isAllowedPath` blocking the product page so the ref never advanced).

---

## Q2 — Is the relocation the behavioral flip? BEFORE persists / AFTER remounts-and-clears?

**YES. The relocation is the regression cause.**

AFTER (working-tree `FilterManager.tsx`, per-page mount, `isAllowedPath` gone) — confirmed by the prior
audit and re-verified here:

- home → product **unmounts** `FilterManager` (the product page does not mount the wrapper; grep confirms
  the wrapper is absent from the product page).
- product → home **fresh-mounts** a new instance → `lastHydratedPathRef.current = null` (`:31`).
- HYDRATE on return: guard `:58` baseSite ok; guard `:59` `null === '/'`? **No → passes → HYDRATE RUNS.** It
  reads `window.location.search` (`:94`) — the product link strips the query (`getNormalizedProductUrl`,
  `src/lib/utils/utils.ts:110-116`; logo therefore forwards `'/'`, `HomeLink.tsx:8-13`), so search is empty →
  it **clears the store** (price `:170-178`, filters `:160`, etc.).
- SYNC (`:209`) fires on the same commit with the stale render-0 closure (store not yet re-rendered) and
  `router.replace`s the **old** params back (`:321`) — async — leaving **URL has params, store empty,
  controls empty.** Because HYDRATE keys on `[baseSite, pathname]` (not search), the re-injected params never
  re-hydrate (`:204`).

So the flip is exactly: **BEFORE = persistent instance, `lastHydratedPathRef` stays `'/'`, store survives,
no clearing HYDRATE on return; AFTER = fresh remount, `ref===null`, HYDRATE-from-empty-URL clears the
store.** Both halves of the relocation are causal:

- The **mount-level change** removed the persistence → the instance no longer survives, so `ref` is `null`
  instead of `'/'` on return (kills the Q1-step-3 skip).
- The **`isAllowedPath` removal** independently removed the product-page block; even a still-persistent
  instance would now run HYDRATE on the product page (pathname `/product/...`, `ref !== pathname`) and clear
  the store *while leaving home*. So `isAllowedPath` was load-bearing too.

Removing both simultaneously is what deleted the BEFORE property.

---

## Q3 — Minimal correct fix given Igor's required behavior

**Recommended: keep per-page mounting + add a store-aware guard at the top of HYDRATE (brief option (a),
discriminated via (b)).** Make the BEFORE behavior **explicit** instead of an emergent side effect of where
the component is mounted.

### (a) Skip the clearing HYDRATE when the URL is param-less but the store has filters

At the top of the HYDRATE effect, **only at a genuine fresh mount** (`lastHydratedPathRef.current === null`),
if the URL carries **no** filter params **and** the store already holds filter values, **do not clear** —
set `lastHydratedPathRef.current = pathname`, leave `hydrated` true, and **return**. SYNC then runs
(`ref === pathname && hydrated`) and pushes store → URL (`:321`), repopulating the URL and the controls from
the store. This reproduces the Q1-step-3 + SYNC sequence exactly, without relying on instance persistence.

### (b) Distinguish "user cleared filters" from "returned from a param-less product page"

The clean discriminator already exists: **the store-level `hydrated` flag** (`useFilterStore.ts:22/60`).
It is on the singleton, so:

- **Hard refresh / genuine fresh load:** fresh JS context → `hydrated === false` at mount. Run HYDRATE
  normally (reads the server-delivered URL; param-less means genuinely no filters). Correct.
- **Client-nav return from product:** the singleton survived → `hydrated === true` at mount **and** the
  store still holds filter values **and** the URL is param-less → this is the "came back from a clean product
  URL" case → skip the clear, let SYNC repopulate.
- **User explicitly cleared filters:** that is a store mutation on an already-mounted instance (not a fresh
  mount), and the SYNC effect strips the URL in response. It does not involve the HYDRATE-at-mount path, so
  the guard never fires for it. Even the degenerate "cleared, then product, then back" case is safe: store
  is empty, so the "store has filters" condition is false → HYDRATE runs and harmlessly confirms empty.

So the guard condition is precisely:
`lastHydratedPathRef.current === null && hydrated === true && storeHasAnyFilter && urlHasNoFilterParams`.
The `=== null` clause is essential so that legitimate **pathname-change** re-hydrations still work — e.g.
catalog category switch `/catalog/x → /catalog/y` (the catch-all does not remount; the persistent instance
has `ref !== null` and re-hydrates from the new URL as today). The guard must touch **only the fresh-mount
case**.

### (c) No params on the product URL

This fix lives entirely inside `FilterManager` and the store. It does **not** add params to the product URL
(it never touches the product-card link or `getNormalizedProductUrl`). Igor's clean-URL requirement is
honored. (This is the structural reason to prefer it over the prior audit's recommended option (d), which
added the filter query to the product-card href — Igor refused that.)

### Loop-avoidance

No HYDRATE↔SYNC loop:
- HYDRATE keys on `[baseSite, pathname]` (`:204`); SYNC's `router.replace` changes only the **search**, not
  the pathname, so it never re-triggers HYDRATE.
- The guard early-returns *before* mutating any store setter, so it does not itself cause a re-render/re-run.
- SYNC's own `if (newUrl !== current)` (`:296`) is the second loop-breaker; once the URL matches the store it
  stops. HYDRATE and SYNC use the same `toQueryParam`/`labelKey` serialization, so the round trip is a
  fixpoint.
This is the identical coordination the BEFORE code relied on; the guard only restores the "don't clear on
return" half.

---

## Q4 — Does the fix differ for `/` vs `/catalog`?

**No — one fix covers both, and it must.** Both `/` and `/catalog` use the **portal** scope
(`SelectableFilterManagerWrapper portalScope="portal"` at `(public)/page.tsx:64` and
`(public)/catalog/[[...slugs]]/page.tsx:185`), both bind `usePortalFilterStore` — the **same singleton** —
and both fresh-mount the manager on return from a product page post-relocation. The guard lives inside
`FilterManager`, so it applies uniformly to every scope (portal/owner/admin) with no per-route branching.

Two catalog-specific notes, neither of which changes the fix:
- **Category context.** Catalog HYDRATE builds category context from the path (`:63-92`) to resolve
  category-scoped `f_<key>` params. The store-skip path **bypasses URL parsing entirely** (it trusts the
  store), so category resolution is irrelevant in the skip case — the store already holds resolved filter
  objects. (This is a separate, pre-existing concern tracked by `audit-filter-rehydrate-bug.md`: the logo
  always targets `/`, where category-scoped filters can't re-resolve from a URL. That bug is about
  re-resolving from the URL on `/`; the store-skip fix sidesteps it for the return-nav case by not reading
  the URL at all.)
- **Catalog x → y.** Category-to-category nav does not remount the catch-all page, so the instance persists
  with `ref !== null` → the `=== null` clause excludes it → it re-hydrates from the new URL as today. The
  fix does not disturb it.

So: same store, same component, same guard — covers both return targets.

---

## Q5 — Partial-revert vs. keep-per-page-mount + guard (honest assessment)

Two viable shapes; I recommend the second.

**Option A — partial revert: restore continuous (layout-level) mounting.**
- *Restores BEFORE exactly*, with proven behavior and zero new conditional logic in HYDRATE.
- *But* a clean revert is not just moving the mount back — it must also **restore `isAllowedPath()`**.
  Without it, the layout-mounted manager runs HYDRATE/SYNC on the product page (and every other `(public)`
  child) and clears the store *while leaving home* (Q2). So Option A = re-add the layout mounts **and**
  re-add the ~19-line string allowlist + its two guards. That **undoes the deliberate W7 cleanup** the
  filter-mount-fix session performed (the "mount-everywhere-then-string-filter" anti-pattern), and the
  allowlist is a hand-maintained route list that drifts from reality (Part 4a smell). It also keeps the
  "rehydrate from store" behavior as an **implicit, accidental** consequence of mount placement — fragile to
  the next person who moves the mount.
- *When A wins:* if Mastermind wants the smallest behavioral risk and is willing to re-accept the W7
  anti-pattern, a straight `git checkout HEAD -- FilterManager.tsx (public)/layout.tsx owner/layout.tsx
  admin/layout.tsx page.tsx ×5` restores a known-good state.

**Option B — keep per-page mounting + add the store-aware guard (a)/(b). RECOMMENDED.**
- *Keeps the W7 cleanup* (no string allowlist returns). The five page mounts stay.
- *Makes the store-survival behavior explicit*: the guard states "on return to a filter surface with a
  surviving store, trust the store" as code, rather than leaning on where the component happens to be
  mounted. That is better engineering — the behavior no longer depends on a layout/page coincidence.
- *Cost:* ~4 lines of guard in HYDRATE and one small `storeHasAnyFilter` helper; localized, unit-testable
  (fresh-mount + empty-URL + populated-store → skip; fresh-mount + empty-URL + empty-store → run;
  ref!==null → unaffected). Slightly more logic in the effect.
- *Risk:* the guard must be gated on `ref === null` so it touches only fresh mounts (catalog x→y must keep
  re-hydrating). That is the one careful clause, called out in Q3/Q4.

**Recommendation: Option B.** It restores the BEFORE behavior (store survives on return; SYNC repopulates
URL + controls), honors the clean-product-URL requirement (no product-link change), and preserves the
deliberate W7 simplification — at the cost of a small, explicit, testable guard rather than re-introducing
the implicit-persistence + string-allowlist coupling. Reserve Option A for the case where Mastermind
prioritizes a zero-new-logic revert over keeping the W7 cleanup.

---

## Cross-repo / contract notes

- No backend or wire-shape involvement; entirely a web client-state/navigation regression.
- The bad AFTER state is user-visible beyond cosmetics: the SSR product list is driven by the URL via
  `filterHydrationSSR` (`FilterHydrationSSRInit.tsx`), so URL-has-params + empty control = filtered results
  with no visible/clearable filter. Either fix shape makes control, store, and URL agree. (Already flagged by
  `audit-hydrate-skip-clientnav.md` as a medium–high issues.md candidate.)

## Closure note

Per the prior audits and this one, the relocation is **uncommitted on `dev`**; `HEAD` is the BEFORE state and
is directly checkout-able if Option A is chosen. No config-file edits are required by this read-only audit.
