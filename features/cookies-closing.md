# Cookies closing

Final cookie hygiene work after Consent Mode v2. This feature covers four items, audited and shipped sequentially:

1. Card-size button fix.
2. `isDashboard` boolean → `portalScope` string-literal union, including the `dashboard` → `owner` value rename.
3. `userPreferenceService` tracking surface removal.
4. Language and theme as preference cookies (pending audit).

Items 1 and 2 are entangled — the card-size button is broken because `isDashboard` is not threaded correctly through the wrapper chain, so the fix is the migration. They are specified together below. Item 4 will be added to this spec after its own Phase 2 audit.

## Status

- Items 1 + 2: shipped 2026-05-21.
- Item 3: shipped 2026-05-21.
- Item 4: language portion shipped 2026-05-22 (with significant divergence from the original spec — see [decisions.md](../decisions.md) 2026-05-22 "Cookies-closing" entry); theme portion deferred to a future Mastermind chat. Handoff doc at [`../.agent/handoffs/theme-path-a.md`](../.agent/handoffs/theme-path-a.md). Permanent reference for the routing + language + cookie infrastructure at [routing-and-language.md](routing-and-language.md).

## Branch

`feature/cookies-closing`, off `dev`.

## Out of scope

- Cross-device consent sync (backend storage of consent state). Deferred to a future Mastermind chat per the Consent Mode v2 closing decision (2026-05-21).
- Any backend or Firestore Rules work. Web-only.

---

## Items 1 + 2 — card-size fix and `portalScope` migration

### Background

`PortalScope` exists today as a string-literal union: `'portal' | 'dashboard' | 'admin'`, defined at `src/lib/types/ui/PortalScope.ts:1`. It is passed as a prop, never read from a store, never derived from `usePathname()`. Every page-level call site hardcodes the literal that matches its route.

`isDashboard` is a parallel boolean mechanism present at 36 line sites across 12 source files. It conflates two ideas that `PortalScope` separates: (a) "pick the secure backend URI" and (b) "render the not-portal variant of UI." The `isDashboard + isAdmin` two-flag pattern in `SearchInput` and the `productsSearchService` files is a single-axis decision (portal / dashboard / admin) expressed as two booleans.

The card-size button is broken because `SelectableFilterProductListWrapper` does not thread `isDashboard` into `FilteredProductList` → `ProductList`. `ProductList` defaults `isDashboard` to `false`, so the dashboard grid reads the **portal** card-size store at runtime regardless of route. Clicking the card-size button correctly writes to `dashboardCardSize` (via the dialog → store path that does receive `isDashboard`), but the visible grid reads `portalSize`. The `router.refresh()` workaround papers over the bug only when preference consent is granted; with consent denied, the cookie write is skipped, SSR doesn't pick up a new value, and the click has no visible effect.

Additional consequences of the same root cause:

- `ProductList`'s mount effect writes the SSR-seeded `dashboardCardSize` value into the `portalCardSize` cookie field on every dashboard page mount (when consent is granted). Cross-scope contamination is live.
- The same admin user clicking `PortalSettingsButton` on desktop vs mobile writes to different cookie fields, because admin desktop defaults `isDashboard=false` and admin mobile (via `SelectableSearchInputWrapper`) hard-codes `isDashboard={true}`.
- Admin has no dedicated card-size value; it reads `dashboardCardSize` at SSR and the portal store at runtime, inheriting whatever was last written there.

### Architectural decisions

1. **`PortalScope` stays a string-literal union.** No enum conversion. Matches existing style (`CardSelectionType`, `Lang`).

2. **`portalScope` is threaded as a prop, not read from a store or hook.** Page-level callers know their route statically and pass the literal. Wrapper components forward it. Deep components receive it as a prop. No `usePortalScope` hook, no Zustand store for scope. The shape is the same as today, extended to the components that currently take `isDashboard` instead.

3. **`dashboard` → `owner` rename across the type union and every consumer.** New values: `'portal' | 'owner' | 'admin'`. The owner-route (`/owner/*`) maps to `'owner'`. This includes the cookie field rename `dashboardCardSize` → `ownerCardSize`.

4. **`isDashboard` is removed everywhere.** No site keeps the boolean. Every consumer takes `portalScope` instead.

5. **One `useCardSize` store, keyed by scope.** Replaces `usePortalCardSize` and `useDashboardCardSize`. Shape:

```ts
type CardSizeState = {
  sizes: Record<PortalScope, CardSelectionType | undefined>;
  setSize: (scope: PortalScope, size: CardSelectionType) => void;
};
```

`setSize` writes to in-memory state unconditionally, and to the matching `globalCookie` field conditional on `isPreferenceConsentGranted()`.

6. **`globalCookie` gains `adminCardSize`.** Three independent fields, one per scope.

7. **Consent gates reads as well as writes.** The store's lazy initializer and the `SyncCardSizeFromCookie` effect both check `isPreferenceConsentGranted()` before reading the cookie. When consent is denied or undecided, the store ignores the persisted value. This closes the gap surfaced in the Consent Mode v2 closing decision, which framed reads-and-writes as both gated but on disk only writes are gated.

8. **`ProductList`'s mount effect is removed.** `SyncCardSizeFromCookie` at app init handles cookie → store seeding. The mount effect was duplicate work and was the source of the cross-scope contamination. Removing it carries a small regression risk if anything implicitly relied on the per-page-mount re-sync — the brief that touches `ProductList` calls this out explicitly.

9. **Clean break on the cookie rename. No migration shim.** Existing browser cookies with `dashboardCardSize` are not read on first page load after deploy. Card-size resets to default for any user with the old field. Pre-production posture; no real users to migrate. Matches the Consent Mode v2 precedent (Brief 7 stripped its legacy migration shim for the same reason).

10. **Default card-size stays `'small'`** across all three scopes. SSR fallbacks and store defaults all use `'small'`.

### Cookie shape

After this feature ships, `GlobalCookie` is:

```ts
type GlobalCookie = {
  portalCardSize?: CardSelectionType;
  ownerCardSize?: CardSelectionType;
  adminCardSize?: CardSelectionType;
  lang?: Lang;
};
```

Three changes from today:

- `dashboardCardSize` renamed to `ownerCardSize`.
- `adminCardSize` added.
- All four fields explicitly optional (today's type lies — fields are non-optional in the type but every consumer treats them as optional).

`lang` stays declared but unwritten until item 4 of the cookies-closing chat ships. No theme field yet (item 4).

### Component contract changes

The following components change their props as part of this work:

| Component                                                                      | Before                                       | After                                                                                    |
| ------------------------------------------------------------------------------ | -------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `ProductList`                                                                  | `isDashboard?: boolean`                      | `portalScope: PortalScope` (required)                                                    |
| `FilteredProductList`                                                          | (no scope prop)                              | `portalScope: PortalScope` (required, forwarded)                                         |
| `FavoriteProductList`                                                          | (no scope prop)                              | `portalScope: PortalScope` (required, set to `'portal'` at call site)                    |
| `UniversalProductCard`                                                         | `isDashboard?: boolean`                      | `portalScope: PortalScope` (required) — overlay logic becomes `portalScope !== 'portal'` |
| `SearchInput`                                                                  | `isDashboard?: boolean`, `isAdmin?: boolean` | `portalScope: PortalScope`                                                               |
| `TopNavigation`                                                                | `isDashboard?: boolean`                      | `portalScope: PortalScope`                                                               |
| `PortalSettingsButton`                                                         | `isDashboard?: boolean`                      | `portalScope: PortalScope`                                                               |
| `PortalConfigDialog`                                                           | dialog payload includes `isDashboard`        | dialog payload includes `portalScope`                                                    |
| `CardSelectionDialog`                                                          | dialog payload includes `isDashboard`        | dialog payload includes `portalScope`                                                    |
| `DashboardProductCard` (literal `isDashboard={true}` → `UniversalProductCard`) | hardcoded `true`                             | passes `portalScope` from its own prop                                                   |
| `AdminProductCard` (literal `isDashboard={true}` → `UniversalProductCard`)     | hardcoded `true`                             | passes `portalScope={'admin'}`                                                           |
| `PreviewProductCard` (literal `isDashboard={true}` → `UniversalProductCard`)   | hardcoded `true`                             | passes `portalScope={'owner'}`                                                           |

Service-layer signatures:

```ts
// src/lib/service/reactCalls/productsSearchService.ts
// Before: getProducts(filters, isDashboard, isAdmin)
// After:  getProducts(filters, portalScope)

// src/lib/service/nextCalls/productsSearchService.ts
// Same shape on the server-side fetch path.
```

The URI selection becomes a `switch (portalScope)` block: `'portal'` → `/public/product/search`, `'owner'` → `/secure/products`, `'admin'` → `/secure/admin/products`.

Caching policy in `nextCalls/productsSearchService.ts` follows the same switch — `'portal'` keeps `revalidate: 60`, the secure scopes keep `cache: 'no-store'`.

### SSR helper

Today, five product-list pages each inline their own `cookies().get('globalCookie')?.value` + `JSON.parse` + fallback object literal, and the fallback shape differs across pages. This feature extracts `readGlobalCookieForSsr(): GlobalCookie` to `src/lib/service/oglasinoCookies.ts`, mirroring the `readConsentForSsr` shape from Consent Mode v2.

The helper returns a fully-defaulted `GlobalCookie` shape: `{ portalCardSize: 'small', ownerCardSize: 'small', adminCardSize: 'small' }`. SSR callers stop doing `|| 'small'` inline; they just read the field.

### `oglasinoCookies.ts` cleanup

`getGlobalCookie()` today claims `: GlobalCookie | null` but body returns `getCookie(...) as GlobalCookie`, casting null through. Fix: drop the cast, return `GlobalCookie | null` honestly. Two callers (`FavoritesPage` at `app/[locale]/(portal)/(protected)/favorites/page.tsx:18,41` and any sibling that does the same) get null-guarded.

### `SimpleCardSizeButton`

Stays exactly as it is — driven by local React state in `PreviewProductDialog`, never touches store or cookie. The preview's card size must remain ephemeral so reviewing a product doesn't pollute the real portal / owner / admin values.

Single change: import and use the shared `OPTIONS` constant from `CardSelectionDialog` instead of the hardcoded inline list. Three-line change, no behavior change.

### Consent gating

`isPreferenceConsentGranted()` from `src/lib/consent/gating.ts` is the gate. The `useCardSize` store calls it in two places:

- **Lazy initializer**: if consent is granted, read the per-scope cookie field; otherwise initialize each scope to `undefined`. The runtime `||` fall-through in consumers covers undefined.
- **`setSize`**: write the in-memory state unconditionally; write the cookie field conditional on consent.

`SyncCardSizeFromCookie` does the same dance: only seeds the store from the cookie when consent is granted.

### Trust boundaries

Card-size and `PortalScope` are display-only. No request DTO carries either value. No moderation, authorization, or state-transition decision uses either. The `productsSearchService` URI selection uses `portalScope` to pick between three backend paths, but the scope literal itself is not transmitted — only the path string differs. Admin authorization is gated by `SessionGuard isAdminRoute={true}` in `app/[locale]/admin/layout.tsx`, which uses Firebase claims via the auth store, not `PortalScope`.

No CRITICAL flags.

### Cross-repo seams

None. Web-only feature. No backend changes, no Firestore Rules changes, no router-worker changes. No translation seed work.

### Risks

- **Removing `ProductList`'s mount effect.** If anything implicitly relied on the per-page-mount re-sync of cookie → store, removing the effect surfaces that as a regression. `SyncCardSizeFromCookie` at app init is expected to cover the seeding job, but the audit did not exhaustively search for non-`ProductList` consumers of the store that might have depended on the mount-effect re-sync. The brief that touches `ProductList` calls this out explicitly so the engineer goes looking.

- **Clean break on cookie rename.** Anyone testing on stage with a `dashboardCardSize` cookie loses their preference on first page load post-deploy. Expected and acceptable.

### Implementation order

One web brief, or a small series of web briefs. Recommended order if split:

1. **Foundation.** Update `PortalScope` type values (`dashboard` → `owner`). Update `GlobalCookie` type — rename `dashboardCardSize` → `ownerCardSize`, add `adminCardSize`, make all fields optional. Add `readGlobalCookieForSsr` helper. Fix `getGlobalCookie()` return-type lie. Build the new `useCardSize` store. Update `SyncCardSizeFromCookie` to use it.

2. **Migration.** Replace `usePortalCardSize` and `useDashboardCardSize` consumers with `useCardSize`. Remove the two old store files. Replace `isDashboard` with `portalScope` across all 36 consumer sites. Drop `ProductList`'s mount effect. Update `CardSelectionDialog` to use `setSize(portalScope, size)`. Update `PortalConfigDialog` to read from the new store via the scope key. Update `productsSearchService` (both files) to take `portalScope` instead of two booleans. Fix `SimpleCardSizeButton` to use the shared `OPTIONS` constant.

3. **SSR callers.** Update all five product-list pages (`/owner/products`, `/admin/products`, `/admin/products/[userId]`, portal home, catalog) to read via `readGlobalCookieForSsr` and to use `portalScope` props. Update `app/[locale]/(portal)/(protected)/favorites/page.tsx` and `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` similarly.

The whole migration can plausibly fit in one brief if the engineer is comfortable with the surface; the audit's 36-line inventory in 12 files is mechanical and well-scoped. If split, the dividing line is foundation (item 1) before migration (items 2 + 3).

### Definition of done

- `isDashboard` does not appear anywhere in `src/` or `app/`. `grep -r isDashboard oglasino-web/src oglasino-web/app` returns nothing.
- `usePortalCardSize` and `useDashboardCardSize` files are deleted.
- `useCardSize` store exists and is the sole card-size store.
- `globalCookie.dashboardCardSize` does not appear in any read site. `globalCookie.ownerCardSize` and `globalCookie.adminCardSize` are the only card-size cookie field reads.
- `readGlobalCookieForSsr` is the only SSR cookie read path.
- The card-size button works on every scope (portal, owner, admin), on every viewport (desktop, mobile), with consent granted and denied. With consent granted, the value persists across page reloads. With consent denied, the value resets to default on each fresh app init.
- `npm run lint`, `npx tsc --noEmit`, `npm test`, `npm run prettier:check` all green.

### Manual verification checklist

After implementation, before this feature closes:

- [ ] Portal home, click card-size button, change to `large`, verify portal grid renders `large`. Reload, verify `large` persists (consent granted) or resets to `small` (consent denied).
- [ ] Owner products page, repeat. Verify owner grid renders the new size, and that portal home is unchanged (no cross-scope contamination).
- [ ] Admin products page, repeat. Verify admin grid renders independently of owner and portal.
- [ ] Switch viewport from desktop to mobile on each scope; verify the mobile `PortalSettingsButton` opens the dialog at the correct scope (owner mobile → owner card-size, admin mobile → admin card-size).
- [ ] Preview a product (`PreviewProductDialog`); change card size via `SimpleCardSizeButton`; close and reopen the dialog; verify the value reset to default and that no other scope's card size changed.
- [ ] Decline preference cookies in the consent banner; navigate to a product list; click the card-size button; verify the in-memory state changes but the cookie is not written. Reload; verify card-size resets to default.

## Platform adoption

This feature is web-only. Mobile (`oglasino-expo`) has no equivalent surface today — the Expo app does not have a card-size selector or a multi-scope (portal/owner/admin) split. No mobile work queues from this feature.

## Item 3 — `userPreferenceService` removal

### Background

Audit (2026-05-21, `.agent/audit-userpreferenceservice.md`) established that `userPreferenceService` was authored in the first commit of `oglasino-web` and has had zero call sites in this repo's history. The three `track*` functions (`trackProductView`, `trackCategoryView`, `trackFiltersUse`), the `getPreferences` reader, and the `clear` action are dead end-to-end. The `UserPreference` type is unused outside the service file itself. The `ExtraProductsComponent` carousel — which the Consent Mode v2 closing decision identified as the dependent consumer — does not actually read `UserPreference` data; its filters are page-derived (`excludeIds`, `ownerId`, `applyRandom`).

The handoff brief from Consent Mode v2 framed step 4 of this work as a "remove the carousel vs. re-source from backend" decision. The audit collapses that question: the carousel was never wired to the tracking surface, so removal of `userPreferenceService` requires no carousel change.

### Decisions

1. **Delete the dead surface.** Remove `src/lib/service/userPreferenceService.ts` and `src/lib/types/cookie/UserPreference.ts` outright. Both files have zero importers outside themselves.

2. **No cookie scrubbing.** If any browser somehow accumulated `products` / `categories` / `filters` keys on the shared `globalCookie` blob, those keys persist as inert data until the next `updateGlobalCookie` write rotates them out. Pre-production posture; no real users, no migration cost. Same break as the `dashboardCardSize` → `ownerCardSize` rename.

3. **One doc reference update.** `oglasino-web/docs/02-architecture.md:235` lists `userPreferenceService.ts` under active services. The line is updated in the same session as the file deletion.

4. **`getCookie` type tightening deferred.** The audit observed that `getCookie` returns `any | null`, and that after this removal every remaining caller routes through the typed `getGlobalCookie()` wrapper. Tightening `getCookie`'s return type to `GlobalCookie | null` is feasible and small, but it touches consent-cookie infrastructure beyond this item's scope. Deferred — not folded into this work.

5. **`extraProducts.recently.viewed.title` translation key stays.** The `/favorites` page's first carousel uses this title even though the underlying feed is "default products excluding your favorites," not viewing history. The misleading copy was authored anticipating a `UserPreference.products` integration that never landed. Acknowledged as a known shortcut; not corrected in this work. Revisits when a real recently-viewed feature lands.

### Trust boundaries

`UserPreference` data was browser-local and never crossed any DTO boundary. No moderation, authorization, or state-transition decision read the cookie field. Removing the surface has no trust-boundary implications. No CRITICAL flags.

### Cross-repo seams

None. Web-only deletion. No backend changes, no Firestore Rules changes, no router-worker changes, no translation seed work.

### Implementation

One web brief, mechanical:

1. Delete `src/lib/service/userPreferenceService.ts`.
2. Delete `src/lib/types/cookie/UserPreference.ts`.
3. Update `oglasino-web/docs/02-architecture.md:235` to remove the `userPreferenceService.ts` reference.
4. Verify `grep -r userPreferenceService oglasino-web/src oglasino-web/app oglasino-web/docs` returns zero matches.
5. Verify `grep -r UserPreference oglasino-web/src oglasino-web/app` returns zero matches (the type is also gone).
6. `npm run lint`, `npx tsc --noEmit`, `npm test`, `npm run prettier:check` all green.

### Definition of done

- Both files deleted.
- Doc line updated.
- Two `grep` checks return zero matches.
- All four verification checks green.

## Item 4 — Language and theme as preference cookies

### Background

Audit (2026-05-21, `.agent/audit-language-theme.md`) inventoried the current state of language and theme persistence. Headlines:

- **Language:** `PortalConfigDialog` is the user-facing change site. It rewrites the URL inline via string concatenation and calls `router.push()` — no cookie write. `globalCookie.lang` is declared on the type but never written by any code path. The `Lang` type is `'sr' | 'en'` (stale; the next-intl routing supports four languages: `sr`, `en`, `ru`, `cnr`).
- **Theme:** `PortalConfigDialog` mounts `ToggleButton` which uses `next-themes`. The library persists to `localStorage.theme` unconditionally, with no preference-consent gate. The Tailwind v4 + CSS variable infrastructure is mature; class-on-`<html>` activates dark mode.
- **Routing:** next-intl v4 with locales as `<tenant>-<language>` (e.g. `rs-sr`, `me-cnr`). Middleware is `proxy.ts` at the repo root (Next 16 naming). `localeDetection` and `localeCookie` default to `true`.
- **Catalog and product slugs are language-independent** (Igor confirmed during seam analysis). Catalog slugs are the same across languages, so language change preserves the path. Product URLs carry a localized product name in the slug, but the product ID is authoritative — `/rs-sr/product/8571/<any-slug>` resolves to product 8571 regardless. Language change preserves the path; the product page itself rewrites to the canonical slug after data loads (see decision #3-A).

### Decisions

1. **Cookie carries bare language code, not tenant-locale.**

   `Lang` broadens to `'sr' | 'en' | 'ru' | 'cnr'`. The cookie's `lang` field is one of these four values. Tenant is hosting-determined, not user preference; a Russian-speaking user visiting any tenant that supports `ru` should see RU.

2. **Cookie-wins-over-URL middleware redirect.**

   When the cookie's `lang` differs from the URL's language portion **and** the URL's tenant allows the cookie's language (per `baseSite.allowedLanguages` for that tenant), `proxy.ts` redirects to the equivalent URL in the cookie's language.

   When the cookie's `lang` is not in the URL's tenant's allowed list, no redirect — URL wins. (Example: cookie `lang: 'cnr'`, URL `/rs-en/...`. `rs` tenant doesn't allow `cnr`. URL stays.)

   When the cookie is absent (first visit), no redirect — URL wins. The first cookie write happens on the user's first explicit language change in `PortalConfigDialog`.

   First-visit "seed cookie from URL" is intentionally **not** implemented. The URL is already what the user sees; no need to record it preemptively. The cookie starts existing the moment the user expresses a preference.

3. **One shared `getLocalizedPath(currentPath, newLang)` helper.**

   Both `PortalConfigDialog`'s language-change site and `proxy.ts`'s redirect logic call this helper. The rule is uniform:

   - Swap the leading `<tenant>-<lang>` segment to `<tenant>-<newLang>`, preserve everything after. Applies to all paths including `/product/*`, `/catalog/*`, `/owner/*`, `/admin/*`.
   - If `currentPath` doesn't have a parseable `<tenant>-<lang>` leading segment, return unchanged.

   This **corrects existing PortalConfigDialog behavior** in two places: today the dialog resets both `/catalog/*` and `/product/*` paths to the tenant root on language change. After this work, both path types preserve. Catalog preserves because catalog slugs are language-independent (no concern). Product preserves because the product ID is authoritative — `/rs-sr/product/8571/lenovo-thinkpad-x1-carbon-gen-9-izuzetan` still resolves to product 8571 even though the slug is in the wrong language. Per decision #3-A below, the product page rewrites the URL to its canonical slug after data loads.

3-A. **Product page rewrites URL to canonical slug on load.**

   After a language change (via PortalConfigDialog or via the middleware redirect), a product URL may carry the previous language's slug. The product page, once its data has resolved, compares the URL's slug to the product's canonical slug in the current language. If they differ, it calls `router.replace()` to the canonical URL silently. Browser history is not polluted.

   Edge case: between the URL hitting the page and `router.replace()` firing, the URL bar briefly shows the stale slug. Cosmetic only — the page content is correct from the first render.

4. **Disable next-intl's auto-detection and auto-cookie.**

   In `src/i18n/routing.ts`, set `localeDetection: false` and `localeCookie: false`. The cookie-wins-over-URL logic in `proxy.ts` is the sole authority on locale; next-intl no longer competes by guessing from `Accept-Language` or writing its own `NEXT_LOCALE` cookie.

   `proxy.ts`'s short-circuit on `/` is unchanged — leave whatever behavior exists today for the root URL alone.

5. **Theme persistence: replace localStorage with cookie via `next-themes`'s storage adapter (target — verify in implementation).**

   `next-themes` v0.4.6 may or may not support a pluggable storage backend. The engineering brief verifies in Step 1 by reading the library's types/docs:

   - **B1-A — Custom storage adapter supported.** Plug the cookie reader/writer into `next-themes` directly. Theme persists to cookie automatically. Single source of truth.
   - **B1-B — Fallback if no custom-storage API.** Keep localStorage as-is; mount a subscriber to `useTheme()` that mirrors changes to the cookie under a consent gate. On SSR or fresh load, the cookie seeds `next-themes` if localStorage is empty. Two-source state with explicit sync.

   B1-A is the target. B1-B is the fallback. The engineering brief decides at Step 1.

6. **Theme persistence is consent-gated.**

   Regardless of B1-A or B1-B path, the cookie write (and any reads of stored state) checks `isPreferenceConsentGranted()`. This closes the live consent-policy gap surfaced in the audit: today, `next-themes` writes localStorage on every toggle, including for users who rejected preference cookies.

7. **Reconciliation semantics differ by preference.**
   - **Card-size**: in-memory-wins-on-grant. The in-memory state reflects a real user-click signal; if it differs from the cookie on consent grant, write the in-memory value to the cookie. Handled by `SyncCardSizeFromCookie`.
   - **Theme**: in-memory-wins-on-grant. Same rationale — the in-memory state reflects a real user-click signal via the theme toggle. (Deferred to future chat; pattern intent recorded.)
   - **Language**: cookie-wins. The in-memory locale (from `useRoutingLocale()` / `getRoutingLocale()`) reflects the URL, which can be a forced fallback from a cross-tenant redirect rather than a user preference. Writing the URL's locale back to the cookie would silently overwrite the user's actual preference. The cookie is set only by explicit user-preference changes via `PortalConfigDialog`, never by sync. The proxy middleware (decision #2) is the cookie → URL reconciliation; there is no URL → cookie reconciliation.

8. **Cookie parsing in middleware is defensive.**

   `proxy.ts` reads `request.cookies.get('globalCookie')`, parses the JSON value. If parsing fails (malformed cookie, schema drift), treat as "no cookie present" — let the URL win, do not throw. Standard defensive parsing pattern; no logging needed beyond a silent fallback.

9. **`extraProducts.recently.viewed.title` translation key stays.** (Carried from Item 3.) Acknowledged as a known shortcut; not corrected in this work.

10. **No first-visit cookie seed from URL.** Already covered by decision #2 but worth calling out separately. The cookie is created on first explicit user preference change. Absence of cookie = use URL.

11. **Theme UI stays bi-state.** The audit flagged that `next-themes` supports `'light' | 'dark' | 'system'` but the UI only offers `'light' | 'dark'`. Adding "system" is a separate UI feature decision unrelated to cookie persistence. Out of scope for Item 4. Logged to `issues.md` as a separate UI feature opportunity, not as a cookie work follow-up.

12. **`globalCookie` cleared on consent decline.** When preference consent transitions from any state to denied, `globalCookie` is deleted entirely (all preference fields). Self-healing migration on app init handles pre-consent-system users (cookies predating the consent system). This resolves the write-side/read-side consent asymmetry: write-side gates on consent (proper); read-side (proxy cookie-wins) trusts cookie presence — under this rule, the cookie only exists when consent was granted, so trust is well-founded.

### Cookie shape

After Item 4 ships:

```ts
type Lang = "sr" | "en" | "ru" | "cnr";

type GlobalCookie = {
  portalCardSize?: CardSelectionType;
  ownerCardSize?: CardSelectionType;
  adminCardSize?: CardSelectionType;
  lang?: Lang;
  theme?: "light" | "dark";
};
```

Two changes from post-Item-3 shape:

- `Lang` broadened from `'sr' | 'en'` to `'sr' | 'en' | 'ru' | 'cnr'`.
- `theme?: 'light' | 'dark'` added. Mirrors the values the current UI emits, not the underlying tri-state primitive.

### Trust boundaries

Both language and theme are display-only:

- Language is a UI rendering choice. The backend's `OglasinoAuthentication.preferredLanguage` is server-derived from `redisUserAuth` per conventions Part 11 and not touched by client cookie state. The `X-Lang` header that web sends to the backend is derived from the URL locale via `getTenantLocale`, not from `globalCookie.lang`.
- Theme is purely visual. No DTO carries it. No header carries it.

No CRITICAL flags.

### Cross-repo seams

None. Web-only. `axios.create({ withCredentials: true })` means `globalCookie` is forwarded to the backend on every request, but the backend does not read it for trust decisions. No backend, router-worker, or Firestore Rules work.

### Risks

- **`next-themes` storage adapter availability.** B1-A depends on the library exposing a pluggable storage interface in v0.4.6. If not, B1-B fallback is materially more code (subscriber + sync logic). Engineering brief verifies before committing to a path.

- **Locale URL fragility on cookie redirect.** The middleware redirect rewrites URLs based on `<tenant>-<lang>` segment matching. If a URL doesn't have that exact shape (malformed, missing locale prefix, unexpected segment count), the middleware must skip the redirect rather than throw. Defensive parsing required, same shape as the cookie parse.

- **`proxy.ts` short-circuit on `/`.** Unchanged by this work, but the file is being edited. The engineer must not accidentally remove or rewrite the existing `/` short-circuit while adding cookie-wins-over-URL logic.

### Implementation order

One web brief if the engineer is comfortable. If split, the dividing line:

1. **Foundation.** Broaden `Lang` type. Add `theme?` to `GlobalCookie`. Build `getLocalizedPath` helper. Update PortalConfigDialog to use the helper (and to write the cookie under the consent gate via a new `useLanguage` store or equivalent — see brief). Set `localeDetection: false` and `localeCookie: false` in `routing.ts`.

2. **Middleware.** Extend `proxy.ts` with cookie-wins-over-URL logic. Read cookie, check tenant allowedLanguages, redirect via `getLocalizedPath` if mismatch.

3. **Theme.** Verify `next-themes` custom storage adapter API. Pick B1-A or B1-B. Wire cookie persistence under consent gate. Mid-session-grant re-hydration via the existing Sync pattern.

The whole work could fit in one brief; splitting at foundation/middleware/theme boundaries keeps each session under 100 lines of code changes.

### Definition of done

- `Lang` type is `'sr' | 'en' | 'ru' | 'cnr'`.
- `globalCookie.lang` writes happen on PortalConfigDialog language change, under preference consent gate.
- `globalCookie.theme` writes happen on theme toggle, under preference consent gate.
- `proxy.ts` reads `globalCookie.lang` and redirects when cookie language differs from URL language and target tenant allows the cookie's language.
- `localeDetection: false` and `localeCookie: false` in `routing.ts`.
- Single `getLocalizedPath(currentPath, newLang)` helper used by both PortalConfigDialog and `proxy.ts`.
- `PortalConfigDialog`'s `/catalog/*` reset behavior is gone (paths preserved across language change for catalog).
- `PortalConfigDialog`'s `/product/*` reset behavior is gone (paths preserved across language change). The product page rewrites the URL to its canonical slug after data resolves.
- Theme writes are consent-gated (closes the live `next-themes` localStorage gap).
- In-memory-wins-on-grant pattern applies to both language and theme on consent grant transition.
- `npm run lint`, `npx tsc --noEmit`, `npm test`, `npm run prettier:check` all green.

### Manual verification checklist

After implementation, before this work item closes:

- [ ] Fresh state (no cookie). Visit `/rs-en/...`. UI is EN. Change to RU via PortalConfigDialog. URL becomes `/rs-ru/...`. Cookie now has `lang: 'ru'`.
- [ ] With `lang: 'ru'` in cookie, visit `/rs-en/product/8571/lenovo-thinkpad-x1-carbon-gen-9-izuzetan` directly. **Expected:** redirected to `/rs-ru/product/8571/lenovo-thinkpad-x1-carbon-gen-9-izuzetan`. The product loads in RU. The URL then rewrites silently to `/rs-ru/product/8571/<russian-canonical-slug>` once data resolves. Browser history shows only the originally-requested URL and the canonical URL (no intermediate entry).
- [ ] With `lang: 'ru'` in cookie, visit `/rs-en/catalog/cars` directly. **Expected:** redirected to `/rs-ru/catalog/cars` (catalog path preserved). UI is RU.
- [ ] With `lang: 'cnr'` in cookie, visit `/rs-en/...`. **Expected:** no redirect — `rs` tenant doesn't allow `cnr`. UI stays EN.
- [ ] Decline preference cookies. Change language via PortalConfigDialog. URL changes (in-memory), but cookie is not written. Reload — back to default behavior (no cookie redirect).
- [ ] Accept preference cookies mid-session. If current in-memory locale (URL) differs from cookie, cookie updates to URL value.
- [ ] Theme toggle changes the UI (dark/light) and writes `globalCookie.theme` under consent gate.
- [ ] With consent denied, theme toggle updates the UI but does not write the cookie. `localStorage.theme` is also not written (regardless of which B1 path is taken, the gate covers both surfaces).
- [ ] Accept preference cookies mid-session with theme toggled from `next-themes` default. Cookie updates to current theme value.

## Item 4 — UI features parked for separate consideration

Logged to `issues.md` as separate UI improvement opportunities:

- Adding `'system'` option to the theme toggle UI (currently bi-state).
