# Audit — web Tier 1 batch (4 issues)

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-27
**Task:** Read-only audit of four `issues.md` entries scoped together as single-file, narrow findings.

---

## 1. `pathnameWithoutLocale(locale)` dead-check

**issues.md entry:** 2026-05-22
**Status in issues.md:** open

### Current code

`app/[locale]/(portal)/(protected)/layout.tsx:10-14`:

```tsx
const locale = await getRoutingLocale();

if (!locale || pathnameWithoutLocale(locale) === '/favicon.ico') return <>{children}</>;

return <SessionGuard isAdminRoute={false}>{children}</SessionGuard>;
```

### `pathnameWithoutLocale` definition

`src/lib/utils/utils.ts:10-13`:

```ts
export const pathnameWithoutLocale = (pathname: string): string => {
  const segments = pathname.split('/').filter(Boolean);
  return '/' + segments.slice(1).join('/');
};
```

The function expects a pathname like `/rs-sr/some/path`. It splits on `/`, filters empty strings, drops the first segment (the locale), and rejoins. For an actual pathname `/rs-sr/favicon.ico`, the return value would be `/favicon.ico`.

### `getRoutingLocale` return value

`src/i18n/getRoutingLocale.ts:9-15`:

```ts
export async function getRoutingLocale(): Promise<string> {
  const h = await headers();
  const locale = h.get('x-next-intl-locale');
  return locale && routing.locales.includes(locale as ...)
    ? locale
    : routing.defaultLocale;
}
```

Returns a compound routing locale string — e.g., `'rs-sr'`, `'me-cnr'`, `'rs-en'`. Never a pathname.

### Analysis

When `pathnameWithoutLocale` receives `'rs-sr'` (a locale, not a pathname):

- `'rs-sr'.split('/').filter(Boolean)` → `['rs-sr']`
- `['rs-sr'].slice(1)` → `[]`
- `'/' + [].join('/')` → `'/'`

The return value is always `'/'`, which is never `=== '/favicon.ico'`.

**The issues.md claim is correct.** The favicon-skip branch is dead code — unreachable under any code path. The call passes a locale string where a pathname is expected, and the function's output can only be `'/'` for any single-segment, slash-free input.

### Severity note

The dead branch means `SessionGuard` wraps all children unconditionally in this layout, including any `/favicon.ico` request that reaches it. Whether that causes any user-visible issue depends on whether the Next.js App Router ever routes favicon requests through this layout (unlikely given Next.js's static file handling).

---

## 2. Product page `getBaseSiteServer()` null-guard

**issues.md entry:** 2026-05-22
**Status in issues.md:** open

### `getBaseSiteServer` return type

`app/actions/getBaseSiteServer.ts:14`:

```ts
export const getBaseSiteServer = cache(async (): Promise<BaseSiteDTO | null> => {
```

Returns `null` on non-2xx response (line 32) or on thrown error (line 35).

### Lines where `baseSite.X` is accessed in the product page

`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`:

1. **Line 38** (`generateMetadata`): `baseSite.code !== productDetails.baseSiteOverview.code` — no null check
2. **Line 64** (`ProductPage`): `baseSite.code !== productDetails.baseSiteOverview.code` — no null check
3. **Line 102**: `baseSite` passed as argument to `generateProductPageStructuredData()` — no null check
4. **Line 107**: `baseSite.catalog?.categories` — no null check on `baseSite` itself (optional chaining only on `.catalog`)

### Existing null-handling pattern in the codebase

The parent layout `app/[locale]/layout.tsx:31-36` already guards:

```tsx
const baseSite = await getBaseSiteServer();
// ...
if (!baseSite) {
  notFound();
}
```

This is the established pattern — the `[locale]` layout fetches `baseSite`, and if null, calls `notFound()`. Since `getBaseSiteServer` uses React `cache()`, the product page's call returns the same cached value.

A second example: `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:62-68` accesses `baseSite.catalog` without its own null check, relying on the same parent layout guard.

### Analysis

**The issues.md claim is technically correct** — the product page accesses `baseSite.code` without an explicit null check. However, **the null case is structurally unreachable in the current layout hierarchy.** The parent `[locale]/layout.tsx` calls `notFound()` when `baseSite` is null, which prevents the product page (and its `generateMetadata`) from executing.

This is a latent issue, not an active bug. It would surface if:

- The product page were moved outside the `[locale]` layout
- Next.js changed the metadata resolution order to call child `generateMetadata` before parent layout rendering (unlikely but not contractually guaranteed)

The catalog page (`catalog/[[...slugs]]/page.tsx`) has the same pattern — also relies on the parent layout guard without its own null check. So this is a codebase-wide convention, not a product-page-specific oversight.

**Fix shape if desired:** `if (!baseSite) return {};` at the top of `generateMetadata`, and `if (!baseSite) notFound();` at the top of `ProductPage`. Matches the parent layout's pattern.

---

## 3. Sitemap over-fetch for product count

**issues.md entry:** 2026-05-16
**Status in issues.md:** open

### `getProductCountForBaseSite` code

`app/sitemap.ts:80-84`:

```ts
async function getProductCountForBaseSite(baseSite: string): Promise<number> {
  // Cheapest call: fetch first page with small perPage just to read totalNumberOfProducts
  const res = await getProductsPage(baseSite, 0, 1);
  return res.totalNumberOfProducts ?? 0;
}
```

### `getProductsPage` signature and behavior

`app/sitemap.ts:54-78`:

```ts
async function getProductsPage(
  baseSite: string,
  page: number,
  perPage: number
): Promise<ProductOverviewsDTO> {
  // POSTs to /public/product/search with { productsFilter: {}, paging: { page, perPage } }
  // Sets headers: { 'X-Base-Site': baseSite }
  // next: { revalidate } (86400 seconds)
```

The call fetches 1 product (`perPage: 1`) from the search endpoint and reads `totalNumberOfProducts` from the response metadata.

### Backend count-only endpoint

Searched `oglasino-web` for any service hitting `/count` or similar: **no count-only endpoint exists** in the web codebase. The only way to get a product count is via the search endpoint's `totalNumberOfProducts` response field.

### How often `sitemap.ts` runs

`app/sitemap.ts:13`: `export const revalidate = 86400;` — **once per day** (24 hours).

### Cost analysis

Call sites for `getProductCountForBaseSite`:

1. `generateSitemaps()` (line 92): called once per base site to determine shard count
2. `productSitemapEntries()` (line 142): called once per base site per product shard

With 3 base sites and M product shards, that's 3 + 3×M invocations per revalidation cycle. However, `getProductsPage` passes `next: { revalidate: 86400 }`, so Next.js Data Cache deduplicates calls with the same request body and headers within the revalidation window. The second call for the same `(baseSite, page=0, perPage=1)` is a cache hit.

**Net cost: 3 backend search POSTs per day** (one per base site), each returning 1 product + metadata. The comment "Cheapest call" is accurate — `perPage: 1` minimizes the response payload. The over-fetch is real (one unnecessary product serialized per call) but the cost is negligible: 3 lightweight requests per day.

### Assessment

**The issues.md claim is correct** — a product search is made purely to read the count. **The practical cost is very low.** A count-only endpoint would save 3 minimal-payload search requests per day. Unless the search endpoint is expensive even for `perPage: 1` (e.g., always loading full Elasticsearch aggregations), this is a low-priority optimization.

---

## 4. `ForegroundPushInit.tsx` URL handling

**issues.md entry:** 2026-05-22
**Status in issues.md:** open

### Lines that call `router.push` with `data.navigate`

`src/components/client/initializers/ForegroundPushInit.tsx:4,8,28-47`:

```tsx
import { useRouter } from '@/src/i18n/navigation';
// ...
const router = useRouter();
// ...
notification.onclick = () => {
  const data = payload.data || {};
  let url = '/notifications';

  if (data.categoryId === 'MESSAGE') {
    url = '/messages';
  }
  if (data.categoryId === 'SAVED_PRODUCT') {
    if (data.productId && data.productName) {
      url = `/product/${data.productId}/${encodeURIComponent(data.productName)}`;
    }
  }
  if (data.categoryId === 'NAVIGATION' && data.navigate) {
    url = data.navigate;
  }
  router.push(url);
```

### Where `data.navigate` originates

`data` is typed as `Record<string, any>` (from `AppNotification.data` at `src/notifications/types/AppNotification.ts:17`). The backend sets the push notification payload data; no web-side type constrains the `navigate` field's format. No web-side comment documents the expected format.

### Shape of the wrapped `useRouter`

`src/i18n/navigation-client.tsx:20-26,59-70`:

```tsx
function prefixWithLocale(href: string, locale: string): string {
  if (!href.startsWith('/')) return href;
  if (href === '/') return `/${locale}`;
  return `/${locale}${href}`;
}

// ...
export function useRouter(): WrappedRouter {
  // ...
  push(href, options) {
    const { locale, ...rest } = options ?? {};
    baseRouter.push(resolve(href, locale), rest);
  },
```

The wrapper **always prepends the locale** to any `href` starting with `/`. Non-`/`-prefixed hrefs (relative paths, absolute URLs) are passed through unchanged.

### Double-prefix scenario

- Backend sends `data.navigate = '/some/path'` (unprefixed) → `prefixWithLocale('/some/path', 'rs-sr')` → `/rs-sr/some/path` → **correct**
- Backend sends `data.navigate = '/rs-sr/some/path'` (already prefixed) → `prefixWithLocale('/rs-sr/some/path', 'rs-sr')` → `/rs-sr/rs-sr/some/path` → **broken (double-prefixed)**
- Backend sends `data.navigate = 'https://example.com'` (absolute URL) → returned as-is → **correct** (but Next.js `router.push` with an absolute URL may not navigate as expected)

### Test fixtures

**No web-side test fixtures exist** for `data.navigate` format. No `.test.*` or `.spec.*` file references `data.navigate`.

### Second call site with same risk

`src/notifications/lib/notificationActions.ts:22-27` has the same pattern:

```ts
case 'NAVIGATION':
  return () => {
    const navigateTo = data?.navigate;
    if (!navigateTo) return;
    router.push(navigateTo);
  };
```

This function takes `router: any`, so whether it double-prefixes depends on whether the caller passes the wrapped or raw router. Both sites share the same risk profile.

### Assessment

**The issues.md claim is correct.** The wrapped `useRouter` always prepends the locale to `/`-prefixed paths. If the backend ever emits a locale-prefixed `data.navigate` URL, the result is a double-prefixed broken path. The risk is real but depends entirely on the backend payload contract, which is undocumented on the web side.

The locally-constructed URLs (`/notifications`, `/messages`, `/product/...`) are all unprefixed and work correctly with the wrapper. Only the backend-supplied `NAVIGATION` URL is at risk.

---

## Files touched

None — read-only audit.

## Tests

Not applicable — no code changes.

## Cleanup performed

None needed — read-only session.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no new entries; all four existing entries confirmed as still accurate. Mastermind may want to update the entries with the audit's findings (e.g., noting the product page null-guard is structurally covered by the parent layout, or noting the sitemap cost is negligible).

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — read-only session, no code changes
- Part 4a (simplicity): N/A — read-only session
- Part 4b (adjacent observations): see "For Mastermind" below
- Part 6 (translations): N/A this session

## Known gaps / TODOs

None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Adjacent observation (Part 4b):** `notificationActions.ts:26` (`router.push(navigateTo)`) has the same double-prefix risk as `ForegroundPushInit.tsx:47`. The function takes `router: any`, so the risk depends on whether callers pass the wrapped or raw router. File path: `src/notifications/lib/notificationActions.ts:22-27`. Severity: low (same as the ForegroundPushInit entry). I did not fix this because it is out of scope.

- **Adjacent observation (Part 4b):** The catalog page (`app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:68`) accesses `baseSite.catalog` without its own null check, same pattern as the product page. File path: `catalog/[[...slugs]]/page.tsx:68`. Severity: low (same structural coverage from parent layout). I did not fix this because it is out of scope.

- **Severity reassessment for item 2 (product page null-guard):** The parent layout `app/[locale]/layout.tsx:34-36` already calls `notFound()` when `baseSite` is null. The product page's lack of a null check is structurally unreachable in the current layout hierarchy. The fix is still correct practice (defensive coding), but the severity is lower than the issues.md entry implies — the bug cannot produce a user-visible TypeError under the current file structure.

- **Severity reassessment for item 3 (sitemap over-fetch):** The practical cost is 3 minimal-payload search requests per day, each fetching 1 product to read `totalNumberOfProducts`. Next.js Data Cache deduplicates repeated calls. Unless the backend search endpoint is disproportionately expensive for `perPage: 1`, this is a very low priority optimization. A count-only backend endpoint would be the clean fix but may not be worth the wire-change ceremony for 3 requests/day.
