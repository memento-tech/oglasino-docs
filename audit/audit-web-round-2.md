# Audit — Five web findings (A6 through A10)

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-27
**Task:** Read-only audit of five issues.md entries (A6–A10) confirming each is still present, the proposed fix shape is correct, and no hidden coupling exists.

---

## Finding A6 — `PortalConfigDialog.navigate` declared async but never awaits

**issues.md location claim:** `src/components/client/dialogs/PortalConfigDialog.tsx`
**Actual location:** `src/components/popups/dialogs/PortalConfigDialog.tsx` — the issues.md path is stale; the file lives under `popups/dialogs/`, not `client/dialogs/`.

### 1. Locate the helper

`src/components/popups/dialogs/PortalConfigDialog.tsx:83-103`:

```typescript
const navigate = async (tenant: string, lang: Lang) => {
  const isSameTenant = tenant === baseSite?.code;

  // Only a same-tenant click expresses a language preference. On a tenant change,
  // `lang` is a computed fallback from getLanguageCodeOrDefault() — not a user choice —
  // so leave the cookie alone so the user's actual preference is preserved across tenants.
  if (isSameTenant && isPreferenceConsentGranted()) {
    updateGlobalCookie('lang', lang);
  }

  // Cross-tenant: drop /product and /catalog paths — product is owned by the source tenant
  // (cross-tenant guard snaps back) and catalog slugs differ across tenants.
  const dropPath = /^\/(product|catalog)(\/|$)/.test(cleanPath);
  const target = isSameTenant
    ? getLocalizedPath(pathname, lang) + queryString
    : `/${tenant}-${lang}${dropPath ? '' : cleanPath}${queryString}`;

  router.push(target);

  onClose();
};
```

### 2. Body contents

Line-by-line scan of the body (lines 84–103):

- `const isSameTenant = tenant === baseSite?.code;` — synchronous comparison
- `if (isSameTenant && isPreferenceConsentGranted())` — synchronous check
- `updateGlobalCookie('lang', lang);` — synchronous cookie write
- `const dropPath = /^\/(product|catalog)(\/|$)/.test(cleanPath);` — synchronous regex
- `const target = isSameTenant ? ... : ...;` — synchronous string build
- `router.push(target);` — `next/navigation` `useRouter().push()` returns `void`
- `onClose();` — synchronous callback

**Zero `await` expressions. Zero Promise-returning calls whose result is consumed. The `async` keyword is a no-op.**

### 3. Call sites within PortalConfigDialog.tsx

Two call sites:

**Line 165:**
```typescript
navigate(baseSite.code, lang.code as Lang);
```
Not awaited. Return value not captured.

**Line 184:**
```typescript
onClick={() => navigate(site.code, getLanguageCodeOrDefault(site) as Lang)}>
```
Not awaited. Return value not captured. The `onClick` handler's return type is `void | undefined`; the implicit `Promise<void>` from `async` is silently discarded.

### 4. External call sites

`navigate` is a `const` declared inside the `PortalConfigDialog` component function body (line 83). It is not exported. It cannot be referenced outside this component.

Grep for `PortalConfigDialog` across the repo confirms external references are limited to:
- `src/components/popups/DialogManager.tsx:22` — imports the default export (the component), not `navigate`
- `app/[locale]/design/topics.ts` — string references in QA topic descriptions

### 5. Fix shape

Remove the `async` keyword from line 83. No behavioral change:
- No `await` in the body to break
- No caller awaits or consumes the returned Promise
- Return type changes from `Promise<void>` to `void`; no downstream type dependency

### Verdict for A6

**Safe to remove `async`.** No callers depend on Promise return. No `await` in body. No external consumers. One additional note: the issues.md file path is stale — the file lives at `src/components/popups/dialogs/PortalConfigDialog.tsx`, not `src/components/client/dialogs/PortalConfigDialog.tsx`.

---

## Finding A7 — `getPathname` exported from `src/i18n/navigation.ts` but unused

### 1. Locate the export

`src/i18n/navigation.ts:10-14`:

```typescript
export function getPathname({ href, locale }: { href: string; locale: string }): string {
  if (!href.startsWith('/')) return href;
  if (href === '/') return `/${locale}`;
  return `/${locale}${href}`;
}
```

No imports — pure string manipulation. Depends on nothing.

### 2. Caller inventory

Grep for `getPathname` across `src/` and `app/` (excluding `node_modules` and the file itself):

**Zero matches.** No file imports `getPathname` from `@/src/i18n/navigation` or any other path. The only references to `getPathname` are within `navigation.ts` itself:
- Line 5: comment mentioning it
- Line 10: the definition
- Line 20: internal usage by `redirect()` — `nextRedirect(getPathname(args), type);`

### 3. Symmetry argument

Everything exported from `src/i18n/navigation.ts`:

- Line 3: `export { Link, useRouter, usePathname } from './navigation-client';` — three re-exports from the client module
- Line 10: `export function getPathname(...)` — explicitly defined
- Line 16: `export function redirect(...)` — explicitly defined

These are individually listed exports, not a wholesale re-export bundle. Each is explicitly chosen.

The comment at lines 5–8 explains the intent:
```
// Mirror next-intl's redirect/getPathname surface, but make the routing
// locale a required, caller-supplied value. Both run server-side and have
// no callers in the repo today — kept for API completeness so future
// callers don't reach for `next/navigation`'s redirect directly.
```

Both `getPathname` and `redirect` have zero external callers. The comment acknowledges this and frames both as forward-looking API completeness.

### 4. Cost of removal

If `getPathname` is removed:
- `redirect` at line 20 breaks — it calls `getPathname(args)` internally.
- But `redirect` also has zero external callers. Both could be removed together, or `getPathname` logic could be inlined into `redirect`.
- No TypeScript module-level type derivations depend on `getPathname`.
- No external consumer hidden by grep.
- No documentation reference in the repo.

### Verdict for A7

**Safe to delete.** Zero external consumers, no derived types, no documentation reference. The sole internal consumer (`redirect`) also has zero external callers — both can be removed together. The comment at lines 5–8 explicitly acknowledges both are unused today; the "kept for API completeness" rationale is a deliberate editorial choice, not a technical dependency.

---

## Finding A8 — `AppInit`'s `setLocale(locale)` uses empty deps array

### 1. Locate the effect

`src/components/client/initializers/AppInit.tsx:24-26`:

```typescript
useEffect(() => {
  setLocale(locale);
}, []);
```

### 2. What `setLocale` does

`src/lib/config/api.ts:9-13`:

```typescript
let storedLocale: string | null = null;

export function setLocale(locale: string) {
  storedLocale = locale;
}
```

Mutates a pure module-scope variable. No React state, no store, no ref. Not a re-render trigger. The variable is read by the request interceptor at `api.ts:97`:

```typescript
const { tenant, oglasinoLocale: lang } = getTenantLocale(storedLocale);

config.headers.set('X-Base-Site', tenant);
config.headers.set('X-Lang', lang);
```

Every outgoing API request reads `storedLocale` to set the `X-Base-Site` and `X-Lang` headers.

### 3. Where `locale` comes from in `AppInit`

`src/components/client/initializers/AppInit.tsx:22`:

```typescript
const locale = useRoutingLocale();
```

`useRoutingLocale()` at `src/i18n/useRoutingLocale.ts:9-11` returns `params.locale` from `useParams<{ locale: string }>()` — the `[locale]` URL segment, which is the **compound routing locale** (e.g., `me-cnr`, `rs-sr`, `rs-en`), not the bare formatter locale.

### 4. AppInit's mount lifecycle

`AppInit` is mounted at `app/[locale]/layout.tsx:48`:

```tsx
<AppInit />
```

This is inside the `[locale]` dynamic layout. In Next.js App Router, the `[locale]` layout is keyed by the `locale` dynamic parameter. When a user navigates from `/rs-sr/...` to `/rs-en/...` (or `/me-cnr/...`), the `locale` parameter changes, which causes the `[locale]` layout to remount its tree — including `AppInit`.

Therefore, `AppInit` effectively remounts on every locale change, and the `useEffect(..., [])` fires fresh each time with the new locale value.

### 5. Consequence of `storedLocale` staling

If `storedLocale` stayed at the original value while the user navigated to a different locale URL:

Consumer at `api.ts:97-100`:
```typescript
const { tenant, oglasinoLocale: lang } = getTenantLocale(storedLocale);
config.headers.set('X-Base-Site', tenant);
config.headers.set('X-Lang', lang);
```

`getTenantLocale('rs-sr')` produces `{ tenant: 'rs', oglasinoLocale: 'sr' }`. If the user navigated to `rs-en` but `storedLocale` stayed `rs-sr`, all API requests would send `X-Lang: sr` instead of `X-Lang: en`. Backend would return Serbian content instead of English.

The stale-locale scenario is currently masked by the layout remount (§4), but the structural gap remains — a future refactoring that changes the layout's mount behavior would silently break API locale headers.

### 6. Fix shape

**Adding `locale` to deps is safe:**
- `setLocale` mutates a module-scope variable. It does not trigger a React state change, does not cause a re-render, does not change `locale`'s value. No infinite loop possible.
- The effect becomes `useEffect(() => { setLocale(locale); }, [locale]);` — fires on mount and on every `locale` change.
- No consumer breaks — the only behavior change is that `storedLocale` stays current even if `AppInit` somehow survives across locale navigations.

**Alternative shapes considered:**
- Remove the effect and call `setLocale(locale)` from the component body: would work, but effects are the conventional place for side effects in React.
- Move `setLocale` into the request interceptor reading the live URL: would require accessing the routing locale from outside React context, which is not straightforward in the request interceptor (module-scope code).

### Verdict for A8

**Add `locale` to deps.** Safe — no infinite loop (`setLocale` mutates module-scope state only, no re-render). Complete — the only consumer of `storedLocale` is the request interceptor at `api.ts:97`, which would correctly read the updated value. Currently masked by `[locale]` layout remount behavior, but the structural gap is real and the fix is a one-character change (`[locale]` instead of `[]`).

---

## Finding A9 — Hardcoded `rs-sr` in intro page SearchAction urlTemplate

### 1. Locate the urlTemplate

`src/metadata/generateIntroPageStructuredData.ts:30-37`:

```typescript
potentialAction: {
  '@type': 'SearchAction',
  target: {
    '@type': 'EntryPoint',
    urlTemplate: `${basicMetadata.baseUrl}/rs-sr/catalog?searchText={search_term_string}`,
  },
  'query-input': 'required name=search_term_string',
},
```

Function signature at `generateIntroPageStructuredData.ts:4-6`:

```typescript
export function generateIntroPageStructuredData(
  tMetadata: MetadataTranslator
): Array<Record<string, unknown>> {
```

No `locale` or `baseSite` parameter — receives only a translation function.

### 2. Intro page nature

`app/page.tsx` is the intro page. Route: `/` (root, no `[locale]` segment). It renders a locale-selector landing page where the user picks their base site from a list of buttons. The page has no "current locale" — it is locale-agnostic by design.

### 3. What other intro-page metadata helpers do

`src/metadata/generateIntroPageMetadata.ts`:

- Title: hardcoded `'Oglasino'` (line 15) — brand name, intentionally not translated
- Description: `tMetadata('intro.meta.description')` (line 15) — translated via a single `tMetadata` function; no locale parameter
- OG locale: hardcoded `'rs-sr'` at lines 27-28:
  ```typescript
  const { alternateLocale } = buildOgLocales('rs-sr', 'all-locales');
  const ogLocale = bcp47ToOgLocale(localeToBcp47('rs-sr'));
  ```
- `x-default` hreflang: points at apex `basicMetadata.baseUrl` (line 25) — deliberately locale-agnostic

The metadata generator handles the no-locale problem by hardcoding `rs-sr` for OG locale (a required field) while using apex `x-default` for hreflang. Same pattern as the SearchAction urlTemplate.

### 4. SEO foundation decisions context

Per `decisions.md` 2026-05-24 SEO foundation entry:

> **Per-cluster x-default for hreflang:** each base-site cluster designates its Serbian-language variant as `x-default`. [...] The intro page (`/`) is the sole exception: its all-locale cluster uses apex `/` as `x-default` because the intro serves as the locale-selector landing page.

The intro page's locale-agnostic nature is deliberate. The `x-default = apex` decision explicitly says the intro page should NOT commit to a locale for hreflang purposes. However, the SearchAction urlTemplate commits to `rs-sr`, which is in tension with this locale-agnostic design — Google's sitelinks search box would land users on `rs-sr` regardless of their preference.

### 5. Fix shape options

**(a) Keep hardcoded + comment.**
Cost: one line. Resolves audit resurfacing. Does not fix the UX: sitelinks search always lands on Serbian.

**(b) Change to `/catalog?searchText={search_term_string}` (no locale prefix).**
Cost: one-line URL change. Risk: the catalog route lives at `app/[locale]/(portal)/(public)/catalog/...` — a request to `/catalog` without a `[locale]` segment would not match any route. The Cloudflare router or `proxy.ts` middleware may redirect unprefixed paths to a default locale, but this is unverified from the audit. If the redirect doesn't exist, Google crawls a 404.

**(c) Use apex `/` as SearchAction target.**
Cost: one-line URL change. Effect: search would land on the locale-selector page, not a catalog — defeats the purpose of a SearchAction.

**(d) Make it locale-aware by accepting a `locale` parameter.**
Cost: function signature change + caller change in `app/page.tsx`. Problem: the intro page has no "current locale" to pass. Contradicts its locale-selector role.

### Verdict for A9

**Cannot decide from audit alone.** The hardcoded `rs-sr` is in tension with the `x-default = apex` decision, but all realistic alternatives have trade-offs requiring a product/SEO judgment call. Additional signal needed:

1. Whether the Cloudflare router or `proxy.ts` redirects unprefixed `/catalog` to a default locale (determines viability of option b).
2. Whether Serbian is the intentional default for sitelinks search (Serbian is >90% of traffic per the SEO foundation decision — hardcoding may be the least-bad option).
3. Whether the sitelinks search box actually materializes on the intro page given it's a locale-selector, not a content page.

---

## Finding A10 — Favorites "recently viewed" carousel is misnamed

### 1. Locate the carousel

`app/[locale]/(portal)/(protected)/favorites/page.tsx:52-61`:

```tsx
<ExtraProductsComponent
  title={tExtraProducts('recently.viewed.title')}
  filter={{
    excludeIds: initialProductsData.products.map((prod) => prod.id),
  }}
  paging={{
    page: 0,
    perPage: 10,
  }}
/>
```

The heading uses `tExtraProducts('recently.viewed.title')` from the `EXTRA_PRODUCTS` namespace (line 20: `const tExtraProducts = useTranslations(TranslationNamespaceEnum.EXTRA_PRODUCTS);`).

### 2. Filter shape

The exact filter object is:
```typescript
{
  excludeIds: initialProductsData.products.map((prod) => prod.id),
}
```

This is a `ProductsFilterDTO`. It excludes the user's currently-favorited products from the results. No `recentlyViewed`, `viewedAt`, `lastSeen`, or any time-based criterion is present.

The service call: `ExtraProductsComponent` calls `fetchExtraProducts(filter, paging)` at `src/lib/service/reactCalls/extraProductsService.ts:13`:

```typescript
const res = await BACKEND_API.post('/public/product/search', {
  productsFilter,
  paging,
});
```

Backend endpoint: `POST /api/public/product/search`. The filter tells the backend "give me products, excluding these IDs." The result is arbitrary products that are not in the user's favorites — ordered by whatever the backend's default sort is, NOT by "recently viewed."

### 3. What `recently.viewed.title` translates to

The key `recently.viewed.title` is in the `EXTRA_PRODUCTS` namespace. The actual translated value lives in the backend SQL seed file (cross-repo — cannot verify verbatim). The key name strongly implies the heading reads something like "Recently viewed" or its equivalent in Serbian/Russian/Montenegrin.

### 4. ExtraProductsComponent / extra-products feature context

`ExtraProductsComponent` at `src/components/client/product/ExtraProductsComponent.tsx` is fully functional:
- Lazy-loads via `requestIdleCallback` + `useTransition`
- Fetches from `/public/product/search`
- Renders a `ProductCarousel`
- Hides itself if no products returned
- No TODO/FIXME comments anywhere in the file

The `EXTRA_PRODUCTS` namespace exists in `TranslationNamespaceEnum.ts:21`. Grep finds the component used in two page files. The feature is **shipped and functional** — the mismatch is between what the heading says ("recently viewed") and what the filter does ("non-favorited products, default sort order").

### 5. Sibling carousels on other pages

**User page** (`app/[locale]/(portal)/(public)/user/[userId]/page.tsx:112-122`):

```tsx
<ExtraProductsComponent
  title={tExtraProducts('similar.products')}
  filter={{
    ownerId: userId,
    excludeIds: productsData.products?.map((prod) => prod.id),
  }}
  paging={{ page: 0, perPage: 10 }}
/>
```

Uses key `similar.products`. Filter: products by the same owner, excluding already-displayed ones. Heading matches the filter intent — "similar products" from the same seller. No mismatch on this page.

**Product page** (`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`): Does NOT use `ExtraProductsComponent`. No carousel at all.

**Favorites page** has a second carousel immediately below the first (`page.tsx:63-73`):

```tsx
<ExtraProductsComponent
  title={tExtraProducts('similar.products')}
  filter={{
    applyRandom: true,
    excludeIds: initialProductsData.products.map((prod) => prod.id),
  }}
  paging={{ page: 0, perPage: 10 }}
/>
```

Uses key `similar.products`. Filter adds `applyRandom: true`. So the favorites page has two carousels:
1. "Recently viewed" → `{ excludeIds }` (no randomization, no recently-viewed criterion)
2. "Similar products" → `{ applyRandom: true, excludeIds }` (random products, excluding favorites)

The first carousel is non-random non-favorited products; the second is random non-favorited products. The only difference is `applyRandom`. Neither has a "recently viewed" tracking mechanism.

### 6. Fix shape options

**(a) Rename the heading.**
New key: something like `extra.products.title` or `more.products.title` or `discover.products.title`. Needs seeding across 4 locales (EN, RS, RU, CNR). One SQL append in the backend repo. Web change: swap `'recently.viewed.title'` to the new key at one site.

**(b) Change the filter to actually fetch recently-viewed products.**
Requires a "recently viewed" tracking mechanism — client-side (localStorage/sessionStorage) or server-side. No such mechanism exists today. This is a feature, not a fix.

**(c) Delete the first carousel entirely.**
The second carousel already shows random products. Removing the first would leave one carousel on the favorites page. The first carousel's purpose (non-random non-favorited products) is marginal when the second already covers "show the user other products."

**(d) Park as known issue until clarified.**
Acknowledge the mismatch, leave as-is, decide when the extra-products feature gets its own Mastermind chat.

### Verdict for A10

**Cannot decide from audit alone.** The heading/filter mismatch is confirmed — "recently viewed" implies time-based tracking that does not exist. The extra-products feature is functionally complete (not unfinished code), but the first carousel's heading makes a promise the filter doesn't keep. Igor needs to decide: rename the heading to match the filter (option a — cheapest, ~1 new translation key), delete the first carousel since the second already covers discovery (option c), or park until the favorites page gets a design pass (option d). Option (b) is a feature, not a fix.

---

## Verdict per finding

| Finding | Verdict |
|---------|---------|
| A6 | **Safe to remove `async`.** No callers depend on Promise return. |
| A7 | **Safe to delete.** Zero consumers. `redirect` (sole internal consumer) also has zero external callers. |
| A8 | **Add `locale` to deps.** Safe (no infinite loop). Currently masked by layout remount. |
| A9 | **Cannot decide from audit alone.** Hardcoded `rs-sr` is in tension with x-default = apex. Needs product/SEO call on whether Serbian default is acceptable for sitelinks search. |
| A10 | **Cannot decide from audit alone.** Heading/filter mismatch is confirmed. Needs Igor's product call on whether to rename the heading, delete the section, or park. |

---

## For Mastermind

### Adjacent observations (Part 4b)

1. **A6 file path is stale in issues.md.**
   - issues.md says `src/components/client/dialogs/PortalConfigDialog.tsx`
   - Actual location: `src/components/popups/dialogs/PortalConfigDialog.tsx`
   - Severity: low (cosmetic — issues.md references a non-existent path)
   - I did not fix this because it is out of scope (read-only audit, and config-file writes are Docs/QA only).

2. **`redirect` function in `src/i18n/navigation.ts` also has zero external callers.**
   - Same file as A7's `getPathname`. Lines 16-21.
   - The comment at lines 5-8 acknowledges both are unused and frames them as forward-looking API completeness.
   - If A7 is fixed (delete `getPathname`), `redirect` should be evaluated in the same brief — it's the only consumer of `getPathname` and itself has no consumers.
   - Severity: low
   - I did not fix this because it is out of scope.

3. **`generateIntroPageMetadata.ts` also hardcodes `rs-sr` for OG locale (lines 27-28).**
   - Same pattern as A9's SearchAction urlTemplate. `buildOgLocales('rs-sr', 'all-locales')` and `bcp47ToOgLocale(localeToBcp47('rs-sr'))`.
   - If A9 is fixed, the OG locale hardcoding should be reviewed in the same brief — the two are coupled decisions about the intro page's locale representation.
   - Severity: low (OG locale is a required field; `rs-sr` as primary OG locale for an all-locales page is defensible but worth recording)
   - I did not fix this because it is out of scope.

4. **Favorites page first and second carousels have nearly identical filters.**
   - First: `{ excludeIds }` → non-random non-favorited products
   - Second: `{ applyRandom: true, excludeIds }` → random non-favorited products
   - The only difference is `applyRandom`. Both exclude the same IDs. If the backend's default sort is deterministic, the first carousel may show a predictable subset of products that partially overlaps with the second carousel's random selection. The UX value of two carousels with near-identical filters is questionable.
   - Severity: low
   - I did not fix this because it is out of scope.

### Config-file impact

- **issues.md:** A6's file path should be corrected from `src/components/client/dialogs/PortalConfigDialog.tsx` to `src/components/popups/dialogs/PortalConfigDialog.tsx`. Draft correction for Docs/QA:

  In the 2026-05-22 entry "PortalConfigDialog.navigate is declared async but never awaits," change:
  > **Found in:** `oglasino-web/src/components/client/dialogs/PortalConfigDialog.tsx`

  To:
  > **Found in:** `oglasino-web/src/components/popups/dialogs/PortalConfigDialog.tsx`

- conventions.md: no change
- decisions.md: no change
- state.md: no change

### Part 4a simplicity evidence (required)

- Added (earned complexity): nothing (read-only audit)
- Considered and rejected: nothing (read-only audit)
- Simplified or removed: nothing (read-only audit)

## Cleanup performed

None needed (read-only audit — no code changes).

## Obsoleted by this session

Nothing (read-only audit).

## Conventions check

- Part 4 (cleanliness): N/A — read-only audit, no code changes
- Part 4a (simplicity): N/A — see structured evidence above
- Part 4b (adjacent observations): four observations flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

None.
