# Audit — ProductCard state badge + base-site-switch stale feed (read-only)

**Repo:** `oglasino-expo` · **Branch:** `new-expo-dev` · **Mode:** READ-ONLY (no code changed)
**Date:** 2026-06-04
**Scope:** confirm presence, as-built location, and safe fix shape for the two tracked
issues. Code is ground truth. Cross-repo reads of `oglasino-web` and `oglasino-backend`
done READ-ONLY (both siblings on disk).

---

## Item A — `ProductCard` renders raw enum string as the state badge

**Present?** **YES** — exactly as reported, line numbers have not drifted.

**As-built location:**
- `src/components/product/ProductCard.tsx:84` — `<Text>{productOverview.productState}</Text>`
- `src/components/product/ProductCard.tsx:88` — `<Text>{productOverview.moderationState}</Text>`
- Overlay gate: `ProductCard.tsx:79-80` — `productState !== ProductState.ACTIVE || moderationState === ModerationState.BANNED`.

**What I found:**

The card prints the **literal enum string** at both sites — no translation, no label
helper. The values render verbatim: `INACTIVE`, `DELETED`, `BANNED` (badge 1 prints
`productState`, badge 2 prints `moderationState` and only when `=== BANNED`).

Enum members that can reach the badges (`src/lib/types/product/ProductState.ts`,
`ModerationState.ts`):
- `ProductState`: `ACTIVE`, `INACTIVE`, `DELETED`. Badge 1 renders whenever the state is
  **not** `ACTIVE` → in practice `INACTIVE` and `DELETED`.
- `ModerationState`: `APPROVED`, `BANNED`. Badge 2 renders only for `BANNED`.

So four raw strings can surface: `INACTIVE`, `DELETED` (badge 1) and `BANNED` (badge 2).

**Translation seed — decides fix class:** there is **no localized key for any product/
moderation state, anywhere** — this needs **new seed rows**, it is *not* a wire-up-existing-
keys job.
- Mobile: grep of `src/`/`app/`/`assets/` finds zero state-label keys. The dashboard state
  **filter** dropdown itself renders raw enums too — `FiltersDialog.tsx:182-185` builds its
  options as `{ value: ns, label: ns }` (e.g. label `"ACTIVE"`/`"INACTIVE"`), so even the
  filter UI has no localized state label to borrow.
- Backend seed (`oglasino-backend/src/main/resources`): the only `product_state` hits are a
  DB `CHECK` constraint and `testProducts.json` fixtures — **no translation seed rows** for
  state labels. (`user.banned.label` exists, but that's the *user* ban indicator in
  Chats/Messages, a different concept from product moderation state.)

**WEB parity (the reference):** web **also renders raw enums** — the 06-01 decisions note is
**correct**. `oglasino-web/src/components/client/product/UniversalProductCard.tsx:47` prints
`{productOverview.productState}` and `:52` prints `{productOverview.moderationState}`,
unlocalized, same as mobile. One divergence worth noting (not a bug, a parity nuance): web
gates the whole overlay on `showOverlay = portalScope !== 'portal'` (dashboard scope only;
`UniversalProductCard.tsx:36,40`), whereas **mobile's `ProductCard` has no scope gate** — it
shows the badge on any surface when the state condition holds. In practice the public portal
feed only returns ACTIVE/APPROVED products, so it rarely shows there, but the mobile code is
unconditional.

**Surfaces that mount this `ProductCard`** (all three wrap the same shared component, so all
three inherit the raw-enum badge):
- `PortalProductCard.tsx:20` → portal feed = **home** (`index.tsx`) and **catalog**
  (`catalog/[...categories].tsx`).
- `DashboardProductCard.tsx:38` → **dashboard product list** (the surface where non-ACTIVE /
  BANNED actually appears).
- `PreviewProductCard.tsx:9` → the **Preview product dialog** (`PreviewProductDialog.tsx:206`).

**Recommended fix shape + blast radius:**

1. **New seed rows (backend, cross-repo — not a mobile edit):** add localized labels for the
   three render-reachable states, e.g. leaf keys `product.state.inactive` /
   `product.state.deleted` / `product.moderation.banned` in the namespace the dashboard cards
   already fetch (DASHBOARD_PAGES is the natural home — `FiltersDialog` reads
   `product.state.filter.label` from `tDash` there). Backend owns the seed; flag for a backend
   brief.
2. **Mobile wire-up:** a tiny label map keyed by enum → translation key, consumed at
   `ProductCard.tsx:84/88` via the existing `tDash`/`COMMON_SYSTEM` translator. One component
   touched. Blast radius = the one shared card, so all three surfaces fix at once.
3. **Consider folding in the filter dropdown** (`FiltersDialog.tsx:182-185`) onto the same
   keys so the state filter stops showing raw enums too — same key set, no extra seed.
4. **Parity decision for Mastermind:** web renders raw enums today. If mobile localizes the
   badge, mobile leads web on this. Either (a) localize both platforms together (shared seed,
   two client edits) or (b) match web and leave raw. This is a cross-repo parity call, not a
   mobile-only decision — surface it before implementing.

> Note: this exact item is already logged **open** in `issues.md` (2026-06-01 create/update
> parity carry-forward batch, first bullet) at the same `:84`/`:88` lines.

---

## Item B — in-app base-site switch leaves the product feed stale

**Present?** **YES (home feed) / partial (catalog)** — the stale-feed gap is real for the
**home** feed; the **catalog** feed mostly self-heals via a different path (details below).
The bug is confirmed by code-reading; **on-device Ψ confirmation still owed** (flagged below
and in `issues.md`).

### Brief premise to correct (read this first)

The brief/issue describes the base-site switch as *"a pure in-place store write while
`status === 'ready'`"* and asks whether it goes through a boot-status transition. **It does.**
`bootStore.ts:434 pickBaseSite` ends with `await get().runFreshnessGate()` (`bootStore.ts:459`)
— the **same** mechanism `setLanguage` uses (`bootStore.ts:480`). Both go
`ready → updating → ready`. So the difference between language and base-site is **not** a
status transition (they share it) and **not** a remount (the Stack deliberately stays mounted
through `updating` — see below). The difference is purely that **`ProductList` has a dedicated
refetch effect keyed on the language code but none keyed on the base-site code.**

### Confirmed refetch triggers in `ProductList` (ALL of them, not just the two reported)

`src/components/product/ProductList.tsx`:
- **`:61-66`** — `refreshOnFavoritesChange && shouldReload` → `onRefresh()`. Only the favorites
  list passes `refreshOnFavoritesChange={true}`; home/catalog pass the default `false`, so
  this never fires for them.
- **`:106-120`** — initial full load, **once**, keyed `[selectedLanguage, fetchPage]`, bounded
  by `hasLoadedInitiallyRef`. A later language switch must NOT re-run this (comment says so).
- **`:136-160`** — **fetchPage-identity change** → `onRefresh()` (reset to top). This is the
  reported `:154` (`if (lastFetchPageRef.current === fetchPage) return;`). `fetchPage` is
  `FilteredProductList`'s `fetchPageInternal`, whose identity depends on `filtersData`,
  `searchText`, `selectedOrder` (`FilteredProductList.tsx:115-142`) — i.e. filter/search/order/
  **category** changes. **No base-site dependency.**
- **`:162-176`** — **`selectedLanguage?.code` change** → `onRefreshCurrent()` (refetch loaded
  pages, preserve scroll). This is the reported `:162`. Uses a `currentLanguageRef` initial-
  capture guard so it skips the first render and only fires on an actual switch.

**None of the four keys on `selectedBaseSite`.** Reported lines `:154`/`:162` are accurate as
the two switch-relevant effects, but the full picture is the four above.

### What `pickBaseSite` writes (`bootStore.ts:434-460`)

- `:435-436` resolve the picked overview; `:440` fetch full DTO (`withGateTimeout`, failure →
  maintenance);
- `:448-452` keep the current language **iff** it's in the new site's `allowedLanguages`, else
  fall back to the new site's `defaultLanguage`;
- `:454` `set({ selectedBaseSite: fullDto, language: lang })` — sets both slots;
- `:456-457` persist both to storage;
- `:459` `await get().runFreshnessGate()` — the status transition.

So: **not** a pure in-place write — it's slot writes **plus** a freshness gate. It *may* also
change the language (only if the old language isn't allowed on the new site).

### Interceptor reads base-site live (header already correct)

`src/lib/config/api.ts:44-45`:
```ts
const { selectedBaseSite, language } = useBootStore.getState();
if (selectedBaseSite) config.headers.set('X-Base-Site', selectedBaseSite.code);
```
Read per request from the live store. Confirmed: the next request after a switch carries the
correct `X-Base-Site` — the gap is that **nothing triggers that next request**, exactly as the
issue states.

### Why language refetches but base-site doesn't (the actual mechanism)

Both go `ready → updating → ready` and **neither remounts** the feed: `app/_layout.tsx:103-107`
mounts the portal `<Stack>` for **both** `'ready'` and `'updating'`, with an explicit comment
(`:96-101`) that `'updating'` is included precisely so a freshness / language / **base-site**
switch does NOT unmount-remount the portal (which would orphan in-flight requests and reset
screen state). The `LoadingOverlay` floats over the still-mounted Stack. `ProductList` early-
returns during non-`ready` (`:295`) but stays **mounted**, so its `products` state and
`hasLoadedInitiallyRef` persist — when it flips back to `ready` it re-renders the **old**
products with no refetch.

Language refetches **only** because of the dedicated `:162-176` effect. Base-site has no
counterpart → no refetch. When a base-site switch *also* changes the language (old language
not allowed on new site), the `:162-176` effect fires and the feed *does* refresh — which is
exactly the "unless the switch also changes the language" caveat in the issue. ✔ matches.

### Home vs catalog

- **Home (`index.tsx` → `HomeFeed` → `FilteredProductList`):** **confirmed stale.** No category,
  so `filtersData` has no base-site-derived input; `fetchPageInternal` identity is unchanged by
  a switch; no language change in the shared-language case → `:136` and `:162` both silent →
  feed shows the previous site's products until manual pull-to-refresh or navigation.
- **Catalog (`catalog/[...categories].tsx`):** **mostly self-heals, not guaranteed.**
  `CatalogScreen` recomputes `categoriesFromPath` from the *new* `selectedBaseSite.catalog`
  (`:26-30`). Category IDs are per-base-site, so on a switch the resolved `topCategory.id` etc.
  change → `filtersData.topCategoryId` changes → `fetchPageInternal` gets a new identity → the
  `:136` effect fires `onRefresh()`. And if the current path doesn't resolve in the new site's
  catalog, `getCategoriesFromPath` returns null → `CatalogScreen` returns null (`:32`) →
  `CatalogFeed` unmounts. Either way the catalog reacts. The only residual gap is the unlikely
  case where two sites share identical category IDs for the same path. So: **report catalog as
  low-risk / largely covered, home as the real bug.**

### Critical safety check — is the naive one-line fix safe?

**A raw "add `selectedBaseSite.code` to the feed dep" is NOT safe as a literal one-liner**, but
a guarded mirror of the existing language effect **is** safe. Specifics:

- **Must key on `selectedBaseSite?.code` (the string), NOT the DTO object.** `runFreshnessGate`
  replaces the `selectedBaseSite` object on a stale-catalog refresh
  (`bootStore.ts:599 set({ selectedBaseSite: freshDto })`). Keying on object identity would
  refetch on every freshness pass; keying on `.code` (as the language effect keys on
  `.code`) avoids that.
- **Must include the initial-capture ref guard** (mirror `currentLanguageRef`,
  `ProductList.tsx:167-170`). Without it the effect fires `onRefreshCurrent` at mount, racing
  the `:106` initial-load effect. With it, it only fires on an actual switch.
- **Double-fire when the switch also changes language is already neutralized.** In the
  old-language-not-allowed case both the language effect and a new base-site effect fire in the
  same commit. `onRefreshCurrent` opens with `if (loadingRef.current) return;`
  (`ProductList.tsx:196`) and sets `loadingRef.current = true` synchronously before its first
  `await` (`:199`), so the second effect's call no-ops. No double fetch — **provided the new
  effect also calls `onRefreshCurrent`** (the reentrancy guard lives there).
- **No boot/curtain-window fire.** `ProductList` only mounts once `status` is `ready` (Stack
  gate). At mount the ref guard captures the initial code and returns; the only later code
  change is via `pickBaseSite`. Safe.

**Recommended (SAFER) fix:** add a dedicated effect in `ProductList`, a near-copy of the
`:162-176` language effect, keyed on `selectedBaseSite?.code`, with its own initial-capture
ref, calling `onRefreshCurrent()`. This inherits the `loadingRef` reentrancy guard (handles the
simultaneous-language case) and the `.code`-keying discipline. One file touched
(`ProductList.tsx`), fixes home and reinforces catalog, mirrors a pattern already proven in
this component.

**Open UX choice for the implementer (not a blocker):** the language effect uses
`onRefreshCurrent` (refetch loaded pages, preserve scroll) because the product *set* is
unchanged, only re-translated. A base-site switch changes the product set **wholesale**, so
`onRefresh` (reset to page 0, scroll to top) is arguably the more correct UX. **However**,
`onRefresh` does **not** carry the `loadingRef` guard at its entry and `loadNextPage(replace=true)`
bypasses the loading check (`:81`), so swapping to `onRefresh` reintroduces the double-fire
race with the language effect in the language-also-changed case. Recommend shipping the
**`onRefreshCurrent` mirror first** (safe), and treating reset-to-top as a follow-on only if Ψ
shows the preserved-scroll behavior is jarring — and if so, gate it behind the same reentrancy
guard rather than calling `onRefresh` raw.

**On-device Ψ required regardless** — this whole item is a code-reading inference. Verify:
switch between two base-sites that **share** the active language and confirm the home feed
swaps to the new site's products without a manual refresh; confirm no double-fetch fl/flicker;
confirm the language-also-changes case still works (no regression).

> Already logged **open** in `issues.md` (2026-06-04, "In-app base-site switch leaves the
> product feed stale (latent, pre-existing)") with the same `ProductList.tsx:154,162`
> reference and the same fix candidate.

---

## Corrections to the issue/brief text (code disagreed)

1. **Item B "pure in-place store write while `status === 'ready'`"** — wrong. `pickBaseSite`
   runs `runFreshnessGate()` (`bootStore.ts:459`), going `ready → updating → ready`, the same
   as `setLanguage`. The real differentiator is the missing dedicated refetch effect, not a
   bypassed status transition.
2. **"the feed's only post-mount refetch triggers are `fetchPage` identity and
   `selectedLanguage.code`"** — there are **four** refetch effects in `ProductList`
   (`:61`, `:106`, `:136`, `:162`); the two named are the switch-relevant ones, but the
   favorites trigger (`:61`) and the once-only initial load (`:106`) also exist.
3. **"naive one-line fix"** — a literal one-line dep addition is unsafe (object-identity
   refetch storms from the freshness-gate DTO swap, plus mount-time race). The safe fix is a
   guarded effect keyed on `.code` mirroring the language effect — small, but not a one-liner.
4. **Item A web parity** — confirmed: web renders raw enums too
   (`UniversalProductCard.tsx:47,52`), and additionally gates the overlay to dashboard scope
   (`portalScope !== 'portal'`), which mobile does not.

## Cleanup performed
None needed — read-only audit, no code touched.

## Config-file impact
No edit to the four `oglasino-docs` config files made or required by this audit. Both items
are already tracked **open** in `issues.md` (Item A: 2026-06-01 parity carry-forward batch;
Item B: 2026-06-04 stale-feed entry). No `state.md` Expo-backlog row is opened or closed by a
read-only audit. If Mastermind scopes either as a brief, the parity decision in Item A §4 and
the cross-repo backend seed dependency should be settled first.
