# AUDIT — Runtime base-site switching on deep-link cold-start

**Repo:** oglasino-expo · **Branch:** `new-expo-dev` (read-only, no edits/builds/git)
**Date:** 2026-06-04 · **Slug:** deeplink-basesite-switch
**Method:** all claims read off source on disk and cross-checked with `grep`/`sed`. No reliance on prior session summaries.

---

## TL;DR

- A **runtime base-site switch primitive already exists**: `bootStore.pickBaseSite(code)` (`src/lib/store/bootStore.ts:307`), used today by `PortalConfigDialog` for an in-app base-site change. So this is **not** a from-scratch "no switch path" job.
- The **entire base-site-scoped data context is one DTO**. `BaseSiteDTO` embeds catalog (categories + topFilters + orderTypes), regions (→ cities), allowedCurrencies, allowedLanguages. One `fetchBaseSiteByCode` call loads the whole cascade. There is **no** multi-resource fan-out, **no** React Query cache, **no** per-resource invalidation to coordinate.
- **But** the two cold-start mechanisms the deep-link feature relies on are **mount-once and do NOT react to an in-place switch**: filter hydration (`useState` initializer, runs once per screen mount) and the product feed's initial load (`hasLoadedInitiallyRef`, keyed on language + `fetchPage`, **not** on `selectedBaseSite`). And the portal Stack stays **mounted** through a switch (`'ready' → 'updating' → 'ready'`), so neither re-runs.
- **Net:** a base-site-switching deep link cannot reuse the in-place `PortalConfigDialog` flow. The switch must complete **before the portal first mounts** — i.e. folded into boot (Gate 3) — or the portal subtree must be force-remounted. **Size: medium**, heavy end, because it touches the boot flow.

---

## A. The base-site model

### A1. Where base-site lives; the DTO shape; the available set

**State.** Base-site lives in `useBootStore` as `selectedBaseSite: BaseSiteDTO | null` (`bootStore.ts:85`). Sibling slots: `language: LanguageDTO | null` (`:86`), `config` (`:87`), and `baseSiteOverviews: BaseSiteOverviewDTO[]` (`:88`) — the "available set".

**`BaseSiteDTO` full shape** (`src/lib/types/catalog/BaseSiteDTO.ts:6`):
```
id, code, domain, iconId?, labelKey, flagImageKey,
defaultLanguage: LanguageDTO,
allowedLanguages: LanguageDTO[],
catalog: CatalogDTO,            // { categories[], topFilters[], orderTypes[] }  (CatalogDTO.ts:5)
defaultCurrency: CurrencyDTO,
allowedCurrencies: CurrencyDTO[],
regions: RegionDTO[],           // each { id, labelKey, cities: CityDTO[] }       (RegionDTO.ts:3)
```
So **catalog, regions/cities, and currencies are all fields on the one DTO** — the data filter resolution needs (`baseSite.catalog.topFilters`, `baseSite.catalog.orderTypes`, `baseSite.regions`, `baseSite.allowedCurrencies`) is entirely self-contained.

**The available set.** `BaseSiteOverviewDTO` (`BaseSiteOverviewDTO.ts:3`) is the lightweight list entry: `id, code, domain, iconId?, labelKey, flagImageKey, defaultLanguage, allowedLanguages` — **no catalog/regions/currencies**. It is enough to (a) enumerate which base-sites exist and (b) read each one's `allowedLanguages`. It comes from `GET /public/baseSite/overviews` via `fetchBaseSiteOverviews()` (`src/lib/init/baseSitesService.ts:40`). The full DTO comes from `GET /public/baseSite/{code}` via `fetchBaseSiteByCode()` (`:48`) — **keyed by code in the URL path, not by the X-Base-Site header**, so it is base-site-agnostic and safe to call for a site other than the currently-selected one.

**Critical timing fact:** the app does **not** hold the full available set at boot for a returning user. `baseSiteOverviews` is populated only on (a) the no-stored-site boot path (`bootStore.ts:286`) and (b) lazily via `ensureBaseSiteOverviews()` (`:365`, called by `PortalConfigDialog`'s `useEffect`). A returning user who booted from a stored site has **`baseSiteOverviews === []`** until something fetches it. (See D7 — this is the fallback-validation cost.)

### A2. How base-site is selected at boot; persistence

Selection happens in **Gate 3** (`runBaseSiteGate`, `bootStore.ts:269`):

- **Stored-site path** (returning user): `getStoredBaseSite()` reads `AsyncStorage['base_site']` and `JSON.parse`s the **full DTO** (`baseSitesService.ts:11-14`). If present, slots are set (`bootStore.ts:279`) and Gate 4 runs. The stored DTO is **trusted until Gate 4 proves a slice stale** (invariant 3). This async storage read is "the honest first-paint moment" (comment `:254-262`).
- **No-stored-site path** (first run): fetch overviews → `toIntroPicker()` (`:286-287`). The machine waits; `pickBaseSite` resumes on tap.

**Persistence writes:** `setStoredBaseSite()` writes the full DTO JSON (`baseSitesService.ts:16`); language via `setStoredLanguage`. Written at `pickBaseSite` (`bootStore.ts:329`) and at Gate 4's catalog-refresh (`:441`). Language read in Gate 3 (`:273`) is validated against `storedSite.allowedLanguages`, falling back to `storedSite.defaultLanguage` (`:274-277`).

### A3. Is there ANY runtime base-site switch path today?

**Yes — exactly one: `pickBaseSite(code)` (`bootStore.ts:307`).** It is reachable post-boot from the running app via **`PortalConfigDialog`** (`src/components/dialog/dialogs/PortalConfigDialog.tsx:56`, wired to the "domain" row `:159` `await setBaseSiteForCode(baseSite.code)`). `BaseSiteSelector` also calls it (`:58`) but only on the intro-picker / boot path.

What `pickBaseSite` does (`bootStore.ts:307-333`):
1. Find the overview by code in `baseSiteOverviews` (defensive; returns if absent — so **the overviews list must be loaded first**; `PortalConfigDialog` ensures this via `ensureBaseSiteOverviews()` on open, `:71`).
2. `fetchBaseSiteByCode(code)` (wrapped in `withGateTimeout`; failure → `toMaintenance`).
3. Language: keep current if in new site's `allowedLanguages`, else new site's `defaultLanguage` (`:321-325`).
4. `set({ selectedBaseSite: fullDto, language: lang })` (`:327`), persist DTO + language (`:329-330`).
5. `runFreshnessGate()` (`:332`) → status `'updating'` → loads/refreshes namespaces for the active language, registers i18n → `'ready'`.

So a switch action exists **and already drives the catalog/regions/currency cascade (via the DTO swap) and the namespace/i18n cascade (via Gate 4).** What it does **not** drive is the product feed and filter re-hydration — see Section B.

> `grep` confirms `selectedBaseSite` is written in exactly three places, all in `bootStore.ts`: Gate 3 stored path (`:279`), `pickBaseSite` (`:327`), Gate 4 refetch (`:442`). No external setter exists.

---

## B. The cascade — what a switch must drag with it

### B4. What is base-site-scoped, and what reloads on a switch

| Scoped data | Where it lives | Reloads on base-site switch? |
|---|---|---|
| **Catalog** (categories/topFilters/orderTypes) | `selectedBaseSite.catalog` — field on the DTO | **Yes, automatically.** `pickBaseSite` swaps the whole DTO; every consumer reads `useBootStore(s => s.selectedBaseSite)` reactively (e.g. `index.tsx:14`, `[...categories].tsx:23`). |
| **Regions/cities** | `selectedBaseSite.regions` (`RegionDTO.cities`) | **Yes** — same DTO swap. `parseRegionsAndCities` reads `baseSite.regions` (`filtersUtil.ts:228-234`). |
| **Currency** | `selectedBaseSite.allowedCurrencies` / `defaultCurrency` | **Yes** — same DTO swap. |
| **Product feed** (home/catalog list) | local `useState` in `ProductList`; data fetched via `getPortalProducts` → axios interceptor injects `X-Base-Site` fresh per request (`api.ts:44-45`) | **NO automatic reload.** The initial load is mount-once (`ProductList.tsx:114` `hasLoadedInitiallyRef`) and its re-fetch triggers are **`fetchPage` identity** (filter change, `:154-160`) and **`selectedLanguage.code`** (`:162-176`) — **never `selectedBaseSite`**. A base-site switch that keeps the language and doesn't remount leaves the feed **stale** (still showing the old site's products). |
| **Cached base-site data** | AsyncStorage `base_site` (full DTO) + catalog checksum; namespace payloads/checksums per (ns, language) | DTO + checksum rewritten by `pickBaseSite`/Gate 4. **No React Query layer anywhere** in the feed path (`grep` for `useQuery`/`queryKey` in `components/product/` → none). So there is no query-key invalidation to design — the only "cache" concerns are the AsyncStorage slots (handled) and the **mount-once refs** (not handled). |

**The cascade gap, precisely:** the DTO swap cascades for free to everything that reads `selectedBaseSite` reactively. The two things that are **read-once at mount** — `useDeepLinkFilterHydration`'s `useState` initializer and `ProductList`'s initial load — are the only pieces that a switch must additionally force. And the `X-Base-Site` header is always correct on the *next* request (interceptor reads the store live); the gap is purely that nothing *triggers* a next feed request on a base-site change.

### B5. The intro-picker path as the reference sequence

The intro-picker tap → `pickBaseSite(code)` (`BaseSiteSelector.tsx:139`) runs the sequence in A3: validate-in-overviews → `fetchBaseSiteByCode` → set slots + language → persist → `runFreshnessGate` (`'updating'` splash) → namespaces/i18n loaded → `'ready'`. **Only on `'ready'` does the portal Stack mount** (`app/_layout.tsx:103`), so the first screen mount already sees the chosen base-site's full DTO. This is exactly the property a deep-link switch wants: *switch resolves before the portal mounts.* The intro-picker gets it because it runs **pre-'ready'**; the in-app `PortalConfigDialog` switch does **not** get it because it runs **post-'ready'** with the Stack already mounted (`'ready' → 'updating' → 'ready'`, Stack never unmounts — `_layout.tsx:96-101` comment is explicit about this).

---

## C. The ordering problem (the crux)

### C6. Switch → load → hydrate sequencing

**Does filter hydration gate on `selectedBaseSite` being ready?** Yes, but only on the **boot-time** base-site, not a *switched* one:
- Home: `HomeScreen` returns `null` until `selectedBaseSite` exists, then mounts `HomeFeed` which calls `useDeepLinkFilterHydration` (`index.tsx:19-33`).
- Catalog: returns `null` until `selectedBaseSite` **and** `categoriesFromPath` resolve (the category is resolved against `selectedBaseSite.catalog`, `[...categories].tsx:26-34`), then `CatalogFeed` hydrates.

These guards wait for *a* base-site to be present. They do **not** wait for a *specific switched* base-site, because hydration is a `useState` initializer (`useDeepLinkFilterHydration.ts:30`) — **it runs exactly once, at the first mount, and never again.** If the screen mounts on the stored (`rs`) site and the base-site later switches to `me` in place, the initializer has already fired against the `rs` catalog and will not re-fire. The category guard would *re-resolve* `categoriesFromPath` against the new catalog (it's a `useMemo` on `selectedBaseSite`), but an `me`-only category path won't resolve against the `rs` catalog → the screen would have rendered `null` or mis-resolved in the interim. The guards assume the boot-time base-site; they are **not** a switched-base-site readiness signal.

**Can the single-synchronous-fetch guarantee hold?** Not as "no load step." A base-site-switching link **necessarily** has an async base-site DTO load (and an overviews fetch to validate — D7) before any filter can resolve, because an `me` region/category does not exist in the `rs` catalog. The single *synchronous-first-render* hydration the filter brief protected cannot apply while a load is pending. **However**, the single *feed* fetch can still be preserved — if the switch completes before the portal mounts, `ProductList`'s mount-once load fires exactly once, against the correct site.

**The required sequence** (each step's async/observable status noted):
1. Parse compound locale → `(baseSiteCode, langCode)` — **sync** (`+native-intent`).
2. Validate base-site against the available set — **async** (overviews fetch; awaitable; see D7).
3. If valid + differs from current: load the switched base-site's full DTO — **async/awaitable** (`fetchBaseSiteByCode`).
4. Set slots + resolve language → run freshness (namespaces/i18n) — **async/awaitable**, observable via the `status` machine (`'updating' → 'ready'`).
5. Portal mounts → filter hydration runs against the switched catalog → single feed fetch — **runs once, post-switch.**

Every step is awaitable and observable through the existing gate awaits + `status`. The catch is **where** steps 2–4 run.

**Where does the "wait for the switched catalog before hydrating" coordination live? Is there an existing readiness signal?**
- The existing `status` machine has `'updating'`, but the portal Stack mounts on **both** `'ready'` and `'updating'` (`_layout.tsx:103`) — deliberately, so an in-place switch doesn't orphan in-flight requests. So `status` is **not** a signal that holds the portal (hence hydration) off during a switch.
- **There is no existing readiness flag that gates the portal off for a switched base-site.** One of two things is needed:
  - **(Design A — recommended) Fold the switch into Gate 3.** Gate 3 reads the deep-link side-channel (D7); if a valid pending base-site differs from the stored one, it selects *that* site (fetch overviews to validate → `fetchBaseSiteByCode` → set slots/language) instead of, or in addition to, the stored site, then proceeds to Gate 4 → `'ready'`. The portal mounts **once**, already switched. Hydration's `useState` initializer and the feed's mount-once load both run against the correct site. **No new readiness flag, no remount, single feed fetch — all preserved.** Cost: a boot-flow change + the side-channel + the validation fetch.
  - **(Design B — not recommended) Post-`'ready'` init + forced remount.** A `DeepLinkBaseSiteInit` (mounted at `'ready'`, like `AppInit`) reads the stash and calls `pickBaseSite`. Because the mount-once mechanisms won't re-run, the portal subtree must be force-remounted (e.g. `key={selectedBaseSite.code}` on the Stack). Worse: the **first** mount on the stored site fires a wasted feed fetch unless an added gate holds the portal until the switch resolves — so this **breaks the single-fetch guarantee** unless you add the very readiness flag Design A avoids. More moving parts, fights the existing mount-once design.

**Conclusion for C:** base-site-switching links are a **separate cold-start path** that must interpose the switch into boot (Design A). They are *not* the same path as the same-base-site filter links already shipped (which need zero boot involvement — they just hydrate at mount). The in-place `PortalConfigDialog` switch flow is **not reusable** for a deep link that must also hydrate/feed-fetch, because it runs after the portal is mounted.

---

## D. The "base-site doesn't exist → open normally" fallback

### D7. Detection point + available-set availability + side-channel

**Available set at the branch point?** **No, not for a returning user.** At `+native-intent` time the boot store is typically unhydrated and `baseSiteOverviews === []`; even at Gate 3 time, the stored-site path never fetches overviews. So to validate "does this base-site exist," the code must **fetch the available set** (`fetchBaseSiteOverviews()` / `ensureBaseSiteOverviews()`) — an async step that does not exist on the returning-user boot path today. (Alternative: attempt `fetchBaseSiteByCode(code)` and treat a non-2xx as "doesn't exist" — but that **conflates an unknown code with a transient network error**, and the fallback contract is "no switch, no crash, open normally," whereas a network error mid-boot is the kind of thing the gates route to `maintenance`. Validating against the overviews set keeps the two distinct and is the cleaner branch.)

**Where the valid/invalid branch lives.** **Not in `+native-intent`.** `redirectSystemPath` is a pure, synchronous, side-effect-free path rewrite, unit-tested in isolation (vitest scans `src/**`; `app/+native-intent.ts:4` just re-exports it). It must not do async/store work. The branch belongs in **the boot layer just after routing resolves — Gate 3** (Design A): `valid base-site in overviews AND differs from current → switch + hydrate; invalid/unknown → drop the stash, boot normally on the stored site (no switch, no application, no crash)`. If the *overviews fetch itself* fails (can't validate), the safe default is **open normally on the stored site**, not maintenance — the user already has a working stored site; an unverifiable deep-link base-site shouldn't brick boot.

**Side-channel (no route pollution).** `redirectSystemPath({ path, initial })` already receives an **`initial` flag** that today is destructured away unused (`redirectSystemPath.ts:40`). The cleanest channel:
- Add a tiny **module-level stash** (e.g. `src/lib/navigation/pendingDeepLinkLocale.ts` exporting `setPendingLocale` / `takePendingLocale`). `redirectSystemPath`, when it strips a locale segment, calls `setPendingLocale(baseSite, language)` (optionally only when `initial === true` for cold-start). The route returned stays locale-less — the locale travels **beside** the path, never in it.
- The boot consumer (Gate 3) calls `takePendingLocale()` (read-and-clear, so a later warm navigation doesn't re-apply a stale switch).
- A module-level stash is preferable to a `bootStore` write here because it keeps `redirectSystemPath` a pure leaf (no store import in the navigation module, preserving its isolated unit test) and matches the brief's "side-channel, not store work in `+native-intent`." Gate 3 (which already imports nothing exotic) reads it.

> Note: `redirectSystemPath` runs for **every** inbound URL, warm and cold. The stash must be read-and-clear, and the consumer must no-op when the parsed base-site equals the current one (the common same-base-site case — most `rs-*` links for an `rs` user).

---

## E. The compound-locale parse

### E8. base-site/language split + validity

**Split rule.** Split the segment on the **first** `-`: `head = seg.slice(0, seg.indexOf('-'))` = base-site, `tail` = language. This yields `rsmoto-sr → (rsmoto, sr)` and `me-cnr → (me, cnr)` correctly — variable-length base-site and language codes are handled because only the first hyphen is the delimiter. The current `LOCALE_SEGMENT` regex (`redirectSystemPath.ts:25`, `/^[a-z]{2,}-[a-z]{2,}$/i`) only *tests* shape; it does not *capture* the split — extraction is new work but trivial.

**The real sets (runtime-derivable, do not hardcode).** Base-site codes and per-site languages come from `baseSiteOverviews` — each `BaseSiteOverviewDTO` carries `code` and `allowedLanguages` (`BaseSiteOverviewDTO.ts:3`). The brief's enumeration (base-sites `rs`, `rsmoto`, `me`; languages `rs/rsmoto → {sr,en,ru}`, `me → {sr,cnr,en,ru}`) matches that runtime shape, but the implementation should derive validity from the fetched overviews, not a literal list (so it can't drift when the backend adds a locale — the same principle `redirectSystemPath`'s shape-rule already follows).

**Validity check.** A parsed `(baseSite, language)` is real iff: `baseSite` ∈ `{overview.code}` **AND** `language` ∈ that overview's `allowedLanguages[].code`.

**Sub-case: base-site valid but language not allowed.** Flag for the product decision (below). The built-in precedent is `pickBaseSite`'s language slot: it keeps the current language only if allowed in the new site, else falls back to the site's `defaultLanguage` (`bootStore.ts:321-325`). So an invalid link-language would naturally degrade to the new site's default language — but that means filter slugs emitted by web *in the link's language* may not resolve (see the language facts). This is a genuine open sub-case, not a mechanical default.

---

## Facts for the OPEN language decision (gathered, not decided)

**Is display language switchable at runtime independently of base-site?** **Yes** — `bootStore.setLanguage(code)` (`bootStore.ts:343`) is the runtime primitive (used by `PortalConfigDialog`, `:57/:133`). It validates the code against the active site's `allowedLanguages`, persists, sets the slot, and re-runs Gate 4 (refetch stale namespaces for the new language → register into i18n → `i18n.changeLanguage`). Its cascade: **re-translation** (i18n bundles + `changeLanguage`), **namespace re-fetch** (only stale slices), and — importantly — **a feed re-fetch**: `ProductList` has a dedicated `selectedLanguage.code` effect that calls `onRefreshCurrent` (`ProductList.tsx:162-176`). So unlike a base-site switch, a **language switch already triggers a feed refresh.** Same `'updating'`-splash fragility as base-site (Gate 4 runs).

**Are non-current-language labels available at hydration time (prices option (a))?** **No.** Filter-option, region, and city resolution slug candidates via `t(labelKey)` — the **active-language** `COMMON_SYSTEM` translator (`filtersUtil.ts:141,228-234`; the translator passed from the screens is `useTranslations(TranslationNamespace.COMMON_SYSTEM)`, `index.tsx:25`, `[...categories].tsx:44`). Gate 4 loads namespace payloads **per active language only** (`bootStore.ts:451-525`, keyed `(ns, activeLang)`). So labels exist **only in the currently-active display language.** Consequence:
- **Option (a)** "keep the user's display language, resolve filter labels in the link's language" requires loading the **link-language** label set (an extra namespace fetch in a non-active language) **and** constructing a translator bound to that language — i18n only holds the active language. This is real extra work and a second label-set in memory.
- **Option (b)** "switch display language to the link's" brings the correct labels **for free** — Gate 4 loads the link language as the active language, and the existing `t()`-based resolution then matches web's slugs (web emits slugs in the link's language). **Cheaper and correct.**
- The slug match is the load-bearing detail: web emits filter/region/city slugs via `toQueryParam(t(label))` **in the link's language**; mobile resolution only matches if the active language at hydration time **is** the link's language. This pushes toward option (b) — or at minimum, resolving in the link's language regardless of what the display shows.

**Does switching base-site force a language change anyway?** **Sometimes — yes.** `pickBaseSite`/Gate 3 keep the current language only if it is in the new site's `allowedLanguages`, else fall back to the new site's `defaultLanguage`. Example: a user on `me-cnr` (language `cnr`) opening an `rs-*` link — `cnr` is not in `rs`'s allowed set → language falls back to `rs`'s default (`sr`). So a base-site switch can **force** a language change purely from the allowed-set constraint, independent of any product preference. `cnr` exists only under `me`; `sr`/`en`/`ru` are shared. This means option (c) ("switch language only when current is invalid in the new base-site") is *already* the built-in behavior of the language slot during a base-site switch — but it does **not** by itself make the link's filter slugs resolve unless the resulting active language equals the link's language.

---

## Verdicts

### Switch feasibility verdict
A runtime base-site switch path **exists and is reusable as a primitive**: `bootStore.pickBaseSite(code)` (also the boot-time selection in Gate 3). It already drives the **full data cascade via a single DTO swap** (catalog, categories, filters, regions, cities, currencies are all fields on `BaseSiteDTO`) plus the namespace/i18n cascade via Gate 4. **No multi-resource reload graph, no React Query keys, no per-slice invalidation to build.** What the primitive does **not** drive: the **product feed** (mount-once, not base-site-keyed) and **filter re-hydration** (mount-once `useState` initializer). Those two are the only cascade additions a switch must force.

### Sequencing verdict
Ordered steps for a base-site-switching deep link: **parse locale (sync) → validate base-site against overviews (async) → fetch switched full DTO (async) → set slots + resolve language + run Gate 4 freshness/i18n (async, observable via `status`) → portal mounts once → filter hydration runs against switched catalog → single feed fetch.** Every step is awaitable/observable. The single *synchronous-first-render* hydration the filter brief protected **cannot hold** (there is necessarily a load step first), **but** the single *feed* fetch **can** be preserved — **only if the switch completes before the portal first mounts.** That requires folding the switch into **boot Gate 3** (Design A). The in-place `PortalConfigDialog` switch (`'ready' → 'updating' → 'ready'`, Stack stays mounted) is **not reusable** for a deep link, because mount-once hydration and the feed's mount-once load will not re-run after an in-place switch, and `status` does **not** gate the portal off during a switch. **Base-site-switching links are therefore a separate, boot-integrated cold-start path, distinct from the same-base-site filter links already shipped.** No existing readiness signal holds hydration off for a switched site; Design A avoids needing one (single mount, post-switch), Design B would have to add one.

### Fallback verdict
The valid/invalid branch lives in **boot Gate 3** (the layer just after routing resolves), **not** in `+native-intent` (which stays a pure path rewrite + a write to a module-level side-channel). **The available set is NOT in hand there** for a returning user (`baseSiteOverviews === []` on the stored-site path) — validation requires an **overviews fetch** that the returning-user boot path does not perform today. Branch: valid-in-overviews **and** differs from current → switch+hydrate; invalid/unknown code → drop the stash, boot normally on the stored site (no switch, no application, no crash); overviews-fetch failure (can't validate) → safe default is open-normally-on-stored-site, not maintenance.

### Size classification
**MEDIUM (heavy end).** Not *small* — hydration does **not** already wait for a switched base-site and the feed is **not** base-site-keyed, so a switch needs new sequencing wiring. Not *large* — a switch primitive (`pickBaseSite`) and a one-DTO cascade already exist; this is not a from-scratch switch mechanism. The medium work is: (1) the compound-locale split + a module-level side-channel from `+native-intent`; (2) an overviews-validation fetch + valid/invalid fallback branch; (3) **a boot-flow change** — Gate 3 reading the side-channel and switching before the portal mounts (or, less cleanly, a portal-remount strategy). Item (3) is why this sits at the heavy end and **must not be briefed blind**: it edits the most stateful gate in the app, on the cold-start path.

---

## For Mastermind

**Boot-flow change — flag loudly.** Base-site is **architecturally boot-only today** for the cold-start hydration/feed path: the only post-boot switch (`pickBaseSite`) keeps the portal mounted and relies on reactive `selectedBaseSite` readers, but the deep-link feature's two mechanisms (filter hydration `useState` initializer; `ProductList` mount-once load) are **mount-once and do not react to an in-place switch**. To make a deep-link base-site switch hydrate + feed-fetch correctly with a single feed fetch, the switch must happen **before the portal mounts** → this **requires editing boot Gate 3** (`runBaseSiteGate`) to read a deep-link side-channel and override the stored-site selection. That is a boot-state change on the cold-start path; the implementation brief should treat it as such (the three boot invariants in `bootStore.ts:42-57` must be honored — no new effect with machine-owned deps; gate-driven advance only; re-entry destroys nothing).

**Validation cost.** A returning user has no available-set loaded (`baseSiteOverviews === []`). Deep-link base-site validation adds an **overviews fetch** to the cold-start path for links that carry a base-site segment. Decide whether to (a) always fetch overviews when a base-site stash is present, or (b) only when the parsed base-site differs from the stored one (cheaper; skips the common same-base-site case).

**Cascade gap to wire (independent of base-site links).** Adjacent finding: the in-app `PortalConfigDialog` base-site switch (`pickBaseSite` while the portal is mounted) does **not** refresh the product feed unless the switch also changes the language (the feed's only post-mount triggers are `fetchPage` identity and `selectedLanguage.code`, `ProductList.tsx:154,162` — never `selectedBaseSite`). So an in-app switch between two sites that share the user's language likely leaves the home/catalog feed showing the **old** site's products until a manual refresh or navigation. This is a **latent existing bug** surfaced by this audit, not introduced by it — worth an `issues.md` row regardless of the deep-link feature. (Could not confirm on-device; this is a code-reading inference. Read-only audit, no edit made.)

**Language decision facts (priced).**
- Display language is independently switchable at runtime (`setLanguage`), and a language switch **already** triggers a feed refresh (`ProductList` language effect) — base-site switch does not.
- Filter/region/city labels exist **only in the active display language** (Gate 4 loads `(ns, activeLang)` only; resolution uses the active `t()`). So **option (a)** (keep display language, resolve in link language) needs an **extra link-language label-set load + a second translator** — i18n holds only the active language. **Option (b)** (switch display to the link's) gets the right labels **for free** and makes web's slugs resolve. The slug-match constraint (web emits slugs in the link's language; mobile only matches if the active language equals the link's) is the deciding fact and favors (b) — or, minimally, resolving in the link's language even if display stays.
- A base-site switch can **force** a language change anyway when the current language isn't in the new site's `allowedLanguages` (e.g. `cnr` is `me`-only; an `me-cnr` user opening an `rs` link falls back to `rs`'s default `sr`). Option (c) is already the built-in language-slot behavior, but it does not by itself make a link's filter slugs resolve unless the resulting active language equals the link's.
- **Sub-case for the decision:** base-site valid but link-language not in its allowed set (a malformed or cross-site link). Default today would be the new site's `defaultLanguage`; decide whether that's acceptable or whether such links should fall through to "open normally."

**Cross-repo (web) notes — could not verify here, flag for the web/backend agents:**
- Web is the canonical link emitter. This audit assumes web emits the compound `{base-site}-{language}` segment and slugs filter/region/city values via `toQueryParam(t(label))` **in the link's language** (mirrored by `filtersUtil.ts`). The implementation brief's correctness hinges on that emission contract; worth a one-line confirmation from web that the locale segment's base-site half is always one of the real `code`s and the language half is always within that site's allowed set.
- AASA/assetlinks patterns already match only the real locale set (per the existing `redirectSystemPath` doc comment) — so the OS only delivers known-shaped locales; but **shape ≠ validity of the base-site against this build's available set**, which is why runtime validation (D7) is still required.

**Could not verify:** on-device boot timing — specifically whether `redirectSystemPath` for the initial URL always runs before Gate 3's async storage read resolves (the side-channel-before-consumer ordering Design A depends on). The gates are several `await`s deep, which makes it very likely, but the brief itself flags boot timing as fragile; the implementation brief should require an on-device check (cold-start `me-*` link with a stored `rs` site) before relying on the ordering.

---

## Conventions check

- **Part 4 (cleanliness):** N/A — read-only audit, no source edited, no `console.log`/TODO/dead code introduced. No `lint`/`tsc`/`test` run needed (no code touched).
- **Part 4a/4b:** the one adjacent observation (in-app base-site switch leaves the feed stale) is recorded in "For Mastermind" rather than acted on, per read-only scope.
- **Config-file impact:** none. This audit produces no `state.md` Expo-backlog change — no feature is adopted or removed by it; it precedes the implementation brief. No edit to any of the four governed docs files is needed or made.
- **Hard rules:** no edits, no builds, no `eas`, no git ops, stayed on `new-expo-dev`, stayed in this repo. Only file written is this audit at `.agent/audit-deeplink-basesite-switch.md`.
