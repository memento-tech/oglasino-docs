# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-27
**Task:** Read-only audit of three `issues.md` entries (privacy/terms English-only markdown, no request cancellation on pagination, redundant per-tenant translation cache entries)

---

## 1. Privacy and Terms render English markdown across all locales

**issues.md entry:** 2026-05-14, severity low, status open.

### Fetch URLs (exact, with surrounding code)

**`app/[locale]/(portal)/(public)/privacy/page.tsx:15-28`:**

```tsx
// TODO Create locale privacy.md
export default async function PrivacyPage() {
  return (
    <div className="mb-2 flex w-full flex-col items-center justify-center pt-10">
      <div className="w-[90%] xl:w-[80%]">
        <MarkdownViewer
          url={
            'https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.en.md'
          }
        />
      </div>
    </div>
  );
}
```

**`app/[locale]/(portal)/(public)/terms/page.tsx:15-27`:**

```tsx
export default async function TermsPage() {
  return (
    <div className="mb-2 flex w-full flex-col items-center justify-center pt-10">
      <div className="w-[90%] xl:w-[80%]">
        <MarkdownViewer
          url={
            'https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/terms-of-use.en.md'
          }
        />
      </div>
    </div>
  );
}
```

### TODO comment

**Confirmed.** `privacy/page.tsx:15` carries `// TODO Create locale privacy.md`. The terms page does **not** carry a matching TODO.

### Locale variable available at fetch site

Both pages live under `app/[locale]/...`. They import `getRoutingLocale` (used in `generateMetadata`) which returns the compound routing locale (e.g. `'rs-sr'`, `'me-cnr'`, `'rs-en'`, `'me-ru'`).

The routing locale is available in the page component body via `await getRoutingLocale()`. To extract the bare language, pass through `getTenantLocale(locale)` which splits on `-` and returns `{ tenant, oglasinoLocale }` — the `oglasinoLocale` value is the bare language code (`'sr'`, `'en'`, `'ru'`, `'cnr'`).

Neither page currently reads the locale in its render function — only in `generateMetadata`.

### MarkdownViewer shape

`src/components/server/MarkdownViewer.tsx:7` — server component, accepts `{ url: string }`.

```tsx
export default async function MarkdownViewer({ url }: { url: string }) {
  // ...
  const res = await fetch(url, { cache: 'force-cache', next: { revalidate: 3600 } });
  // ...
  content = await res.text();
  // ...
  return (
    <article className="prose ...">
      <ReactMarkdown ...>{content}</ReactMarkdown>
    </article>
  );
}
```

The component takes a single `url` prop, fetches the raw text, and renders it as markdown. To support per-locale content, the caller constructs the URL with the appropriate locale suffix — the component itself is locale-agnostic.

### Locale-suffixed files in the external repo

Verified by HTTP HEAD requests to `raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/`:

| File | Status |
|---|---|
| `privacy-policy.en.md` | **200** (exists) |
| `terms-of-use.en.md` | **200** (exists) |
| `privacy-policy.sr.md` | **404** (does not exist) |
| `privacy-policy.ru.md` | **404** (does not exist) |
| `privacy-policy.cnr.md` | **404** (does not exist) |
| `terms-of-use.sr.md` | **404** (does not exist) |

**Only the English `.en.md` files exist.** No per-locale variants have been created.

### Web routing locales

From `src/i18n/routing.ts:5-16`, the ten routing locales are:

```
rs-sr, rs-en, rs-ru, rsmoto-sr, rsmoto-en, rsmoto-ru, me-sr, me-cnr, me-en, me-ru
```

The four distinct bare languages are: **`en`, `sr`, `ru`, `cnr`**.

(Montenegrin `cnr` aliases to Serbian per conventions Part 9 — whether it should share `sr` markdown or have its own `cnr` variant is an Igor/legal decision.)

### Issues.md entry accuracy

The entry's claim is correct in substance: both pages fetch hardcoded English-only URLs. Two minor inaccuracies in the entry's phrasing:

- Entry says `privacy.md` / `terms.md`; actual filenames are `privacy-policy.en.md` / `terms-of-use.en.md`.
- Entry says line `:14`; the TODO is on line `:15` and the URL is on lines `:21-23` (privacy) / `:20-22` (terms).

### Cross-repo dependency — CRITICAL

**This fix is NOT web-only.** The web swap is trivial (~5 lines per page), but the per-locale markdown files must exist in `memento-tech/oglasino-platform` before the swap can land. Currently only `.en.md` files exist.

Required before the web fix:

1. Igor (or legal-drafts chat) authors/translates `privacy-policy.sr.md`, `privacy-policy.ru.md`, `terms-of-use.sr.md`, `terms-of-use.ru.md` in the `memento-tech/oglasino-platform` repo.
2. Decision on `cnr`: does Montenegrin get its own `privacy-policy.cnr.md` / `terms-of-use.cnr.md`, or does it share `sr`? Conventions Part 9 says "Montenegrin (me/cnr) aliases to SR" — if that extends to legal content, the web fix maps `cnr` → `sr` in the URL construction.

### Recommended URL shape

The existing files already use `<base-name>.<lang>.md` (e.g. `privacy-policy.en.md`). The per-locale variants should follow the same pattern:

```
privacy-policy.sr.md
privacy-policy.ru.md
privacy-policy.cnr.md  (or alias to sr)
terms-of-use.sr.md
terms-of-use.ru.md
terms-of-use.cnr.md    (or alias to sr)
```

### Recommended fallback strategy

If a locale-specific file doesn't exist (404), fall back to English. This is safe because:

- MarkdownViewer already handles fetch failures (shows `tErrors('markdown.field.load')` — a translated error message, not a crash).
- A two-step approach (try locale, fall back to `en`) is better than hard-failing, because it gracefully handles the period between deploying the web code and uploading all locale files.
- The fallback could be implemented either in the page (try locale URL first, if 404 try `.en.md`) or in MarkdownViewer (accept a fallback URL prop).

### Recommended web fix shape

```tsx
// In privacy/page.tsx (and terms/page.tsx similarly)
const locale = await getRoutingLocale();
const { oglasinoLocale: lang } = getTenantLocale(locale);
const effectiveLang = lang === 'cnr' ? 'sr' : lang; // if cnr aliases to sr
const url = `https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.${effectiveLang}.md`;
```

---

## 2. No request cancellation on pagination

**issues.md entry:** 2026-05-14, severity low, status open.

### Existing autocomplete AbortController pattern

**Where it lives:** `src/components/client/SearchInput.tsx:70-95`.

```tsx
useEffect(() => {
  if (!debouncedTerm || debouncedTerm.length <= 2) {
    setResolvedProducts([]);
    setPopupOpen(false);
    return;
  }

  const controller = new AbortController();

  getAutocompleteSuggestions(
    { ...productsFilter, searchText: debouncedTerm, ... },
    controller.signal
  ).then((res) => {
    if (controller.signal.aborted) return;
    setResolvedProducts(res);
    setPopupOpen(true);
  });

  return () => controller.abort();
}, [debouncedTerm, categories]);
```

**How it's plumbed to the service:** `getAutocompleteSuggestions` in `SearchInput` calls `getPortalAutocompleteSuggestions(filter, signal)` → `getAutocompleteSuggestions(filter, portalScope, signal)` → `BACKEND_API.post(uri, body, { signal })` at `productsSearchService.ts:87-94`. The `signal` parameter flows through the full chain. The `catch` block checks `signal?.aborted` and returns `[]` silently on cancellation.

A second AbortController pattern exists in `src/messages/components/MessageInput.tsx:47-179` for message sending — same `useRef<AbortController>` approach but manual (not effect-driven).

### Current paging call shape in ProductList

**`src/components/client/product/ProductList.tsx:78-91`:**

```tsx
const onDisplayPageChange = async (page: number) => {
  setLoading(true);
  setDisplayPage(page);

  const result = await onNextPage({
    page,
    perPage: PRODUCTS_PER_PAGE,
  });

  setProducts(result.products);
  setLoading(false);

  window.scrollTo({ top: 0, behavior: 'instant' });
};
```

Key observations:

- **No AbortController.** The `onNextPage` callback does not accept a signal.
- **No stale-response guard.** `setDisplayPage(page)` runs synchronously, then the async call resolves at an arbitrary time. If the user clicks page 2, then page 3, the sequence is: `setDisplayPage(2)` → `setDisplayPage(3)` → (page 2 response arrives) `setProducts(page2Products)` → (page 3 response arrives) `setProducts(page3Products)`. If page 3's response arrives first and page 2's arrives second, the user ends up seeing page 2's products while the selected page button shows page 3.
- **`setLoading(false)` races similarly.** The first-to-resolve response sets `loading=false`, hiding the spinner while the second request is still in flight.

### Paging service call signature

**`src/lib/service/reactCalls/productsSearchService.ts:132-154`:**

```tsx
const getProducts = async (
  productsFilter: ProductsFilterDTO,
  paging: PagingDTO,
  portalScope: PortalScope
): Promise<ProductOverviewsDTO> => {
  const uri = productSearchUri(portalScope);
  try {
    const res = await BACKEND_API.post(uri, { productsFilter, paging });
    // ...
  }
};
```

No `signal` parameter. To add cancellation, the signature would change to `(productsFilter, paging, portalScope, signal?: AbortSignal)` and pass `{ signal }` as the third arg to `BACKEND_API.post`, matching the autocomplete pattern.

### Surfaces using ProductList (blast radius)

`ProductList` is used by **two** wrapper components, covering **six** page surfaces:

1. **`FilteredProductList`** → **`SelectableFilterProductListWrapper`** — used on:
   - Portal home (`app/[locale]/(portal)/(public)/page.tsx`) — `portalScope='portal'`
   - Catalog (`app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx`) — `portalScope='portal'`
   - User profile products (`app/[locale]/(portal)/(public)/user/[userId]/page.tsx`) — `portalScope='portal'`
   - Owner products (`app/[locale]/owner/products/page.tsx`) — `portalScope='owner'`
   - Admin products (`app/[locale]/admin/products/page.tsx`) — `portalScope='admin'`
   - Admin per-user products (`app/[locale]/admin/products/[userId]/page.tsx`) — `portalScope='admin'`

2. **`FavoriteProductList`** — used on:
   - Favorites page (`app/[locale]/(portal)/(protected)/favorites/page.tsx`) — `portalScope='portal'`

All seven surfaces share the same `ProductList` component and the same `onDisplayPageChange` handler. **Any fix to `ProductList` would apply uniformly across all seven surfaces.**

### Filter/sort change race condition

Filter and sort changes go through `FilterManager` → URL update → server-side RSC re-render → `SelectableFilterProductListWrapper` receives new `filtersData` → `FilteredProductList`'s `key={JSON.stringify(keyFiltersData)}` changes → **`ProductList` is fully remounted**. The remount resets all state (products, displayPage, loading) and discards any in-flight requests (orphaned promises update state on an unmounted component — React ignores this).

**Filter/sort changes do NOT have the stale-response race.** The race is specific to **pagination clicks within a living `ProductList` instance.**

The favorites effect at `ProductList.tsx:97-114` has a similar (but distinct) pattern: it re-fetches page 0 when `favoriteIds` changes. This is also not abort-guarded, but since it always fetches page 0, the race window is narrow (only if the user rapidly toggles favorites while on the favorites page).

### Recommended fix shape

Two approaches, matching the existing codebase patterns:

**Option A — AbortController ref (matches MessageInput pattern):**

```tsx
const abortRef = useRef<AbortController | null>(null);

const onDisplayPageChange = async (page: number) => {
  abortRef.current?.abort();
  const controller = new AbortController();
  abortRef.current = controller;

  setLoading(true);
  setDisplayPage(page);

  const result = await onNextPage({ page, perPage: PRODUCTS_PER_PAGE }, controller.signal);

  if (controller.signal.aborted) return;
  setProducts(result.products);
  setLoading(false);
  window.scrollTo({ top: 0, behavior: 'instant' });
};
```

Requires widening `onNextPage` prop to accept `signal?: AbortSignal`, and threading signal through `getProducts` / `getFavorites`.

**Option B — sequence counter (simpler, no service change):**

```tsx
const seqRef = useRef(0);

const onDisplayPageChange = async (page: number) => {
  const seq = ++seqRef.current;
  setLoading(true);
  setDisplayPage(page);

  const result = await onNextPage({ page, perPage: PRODUCTS_PER_PAGE });

  if (seq !== seqRef.current) return; // stale
  setProducts(result.products);
  setLoading(false);
  window.scrollTo({ top: 0, behavior: 'instant' });
};
```

Option B is simpler (no service signature change, no AbortSignal plumbing) but does not cancel the in-flight HTTP request — it only discards the stale response. Option A actually cancels the request, saving bandwidth and backend load.

---

## 3. Redundant per-tenant translation cache entries

**issues.md entry:** 2026-05-25, severity low, status open.

### Cache key construction

**`src/translations/lib/translationsCache.ts:85-92`:**

```tsx
export const getNamespaceTranslations = unstable_cache(
  fetchNamespaceTranslations,
  ['translations-v1'],
  {
    revalidate: 86400,
    tags: ['translations'],
  }
);
```

`unstable_cache` builds the cache key from the **keyParts array** (`['translations-v1']`) **plus all function arguments** (`namespace`, `lang`, `tenant`). This means the effective cache key is the tuple `('translations-v1', namespace, lang, tenant)`.

### Where tenant enters the cache key

`tenant` enters through the call chain:

1. `src/i18n/request.ts:14` — `const { tenant, oglasinoLocale } = getTenantLocale(routingLocale);`
2. `src/i18n/request.ts:16` — `const messages = await loadAllNamespaces(oglasinoLocale, tenant);`
3. `src/i18n/internalRequest.ts:11` — `namespaces.map((ns) => getNamespaceTranslations(ns, lang, tenant))`
4. `src/translations/lib/translationsCache.ts:85` — `unstable_cache(fetchNamespaceTranslations, ...)` — `tenant` is the third argument, automatically part of the cache key.

`getTenantLocale` at `src/translations/lib/util/getTenantLocale.ts:27-33` splits the routing locale on `-`: `'rs-sr'` → `tenant='rs'`, `oglasinoLocale='sr'`.

### Confirmation that backend ignores X-Base-Site for translations

**`src/translations/lib/translationsCache.ts:29`:**

```tsx
const url = `${apiUrl}/public/translations?namespace=${namespace}&lang=${lang}`;
```

The URL query parameters are `namespace` and `lang` only — no `tenant`. The `X-Base-Site` header is sent at line 40:

```tsx
const res = await fetch(url, {
  headers: { 'X-Base-Site': tenant },
});
```

But per the issues.md entry (confirmed by the 2026-05-25 image-alt-translation base-site audit and the decisions.md 2026-05-25 entry), the backend `TranslationsController` takes only `(namespace, lang)` and ignores `X-Base-Site`. The header is sent uniformly by convention but is inert for translations.

### Quantified redundancy

Ten routing locales decompose to these `(tenant, lang)` pairs:

| Routing locale | tenant | lang |
|---|---|---|
| rs-sr | rs | sr |
| rs-en | rs | en |
| rs-ru | rs | ru |
| rsmoto-sr | rsmoto | sr |
| rsmoto-en | rsmoto | en |
| rsmoto-ru | rsmoto | ru |
| me-sr | me | sr |
| me-cnr | me | cnr |
| me-en | me | en |
| me-ru | me | ru |

Four unique languages: `sr`, `en`, `ru`, `cnr`. Per language, tenant count:

- `sr`: 3 tenants (rs, rsmoto, me) — 3 identical cache entries per namespace
- `en`: 3 tenants (rs, rsmoto, me) — 3 identical cache entries per namespace
- `ru`: 3 tenants (rs, rsmoto, me) — 3 identical cache entries per namespace
- `cnr`: 1 tenant (me) — 1 cache entry per namespace (no redundancy)

With **22 namespaces** (`TranslationNamespaceEnum` has 22 values):

- **Total cache entries:** 22 × 10 = **220**
- **Needed cache entries:** 22 × 4 = **88**
- **Redundant entries:** 220 − 88 = **132** (60% redundancy)

### All callers of the cache

The sole caller of `getNamespaceTranslations` is:

**`src/i18n/internalRequest.ts:11`:**

```tsx
namespaces.map((ns) => getNamespaceTranslations(ns, lang, tenant))
```

Called from `loadAllNamespaces(lang, tenant)`, which is called from `src/i18n/request.ts:16` — the next-intl server request config. This is the only entry point.

### Does any caller rely on tenant in the key for separation?

**No.** The `tenant` parameter is:

1. Part of the `unstable_cache` key (automatic from function arguments).
2. Sent as `X-Base-Site` header (ignored by backend).
3. Not used anywhere else in the cache function body or by any consumer.

No caller distinguishes translations by tenant. Dropping tenant from the key would not break any existing behavior — the backend returns identical data regardless of tenant.

### Cache invalidation surface

The `/api/revalidate` route at `app/api/revalidate/route.ts` supports `revalidateTag('translations')` to wipe all cached translations. The tag comments at line 23 also list a granular tag format `translations:{NS}:{lang}:{tenant}` — but `unstable_cache` at `translationsCache.ts:90` uses only the static tag `'translations'`. The granular tags in the comment are aspirational/unused; the actual invalidation is all-or-nothing via `revalidateTag('translations')`.

### Tests asserting against the current shape

No test files exist for `translationsCache.ts`. Grep for `translationsCache` in test files returned zero matches. The change is test-risk-free.

### Blast radius of dropping tenant from the key

**Minimal.** The fix involves:

1. Remove `tenant` from the `fetchNamespaceTranslations` parameter list.
2. Remove `X-Base-Site` header from the fetch call.
3. Update `loadAllNamespaces` to not pass `tenant`.
4. Update `request.ts` to not pass `tenant` to `loadAllNamespaces`.

Or, if `tenant` must remain in the function signature for other reasons (e.g., diagnostic logging), add it to the `unstable_cache` wrapper's exclusion — but `unstable_cache` doesn't support excluding specific args from the key. The cleaner approach is to remove the parameter entirely.

**Structural surprise:** The only structural coupling to `tenant` in the translations path is the `X-Base-Site` header on the fetch call. The backend explicitly ignores it. Removing it is safe.

---

## Files touched

None — read-only audit.

## Tests

Not applicable — no code changes.

## Cleanup performed

None needed (read-only session).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no new entries; the three existing entries are confirmed accurate (with minor line-number corrections noted in the body)

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — read-only session
- Part 4a (simplicity): N/A — read-only session
- Part 4b (adjacent observations): confirmed — see "For Mastermind" section
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only session)
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Item 1 cross-repo dependency is the critical path.** The web fix for privacy/terms is trivial (~5 lines per page), but the per-locale markdown files must be authored/translated and committed to `memento-tech/oglasino-platform` first. Only `.en.md` files exist today. Igor needs to author or commission SR/RU/CNR translations of both documents before the web engineer brief can land. The legal-drafts chat (referenced in `state.md` under "Privacy Policy + Terms (drafts)" — status `drafted`, pending lawyer review) is the natural upstream. Sequence: lawyer review → per-locale markdown files committed → web engineer brief.

- **Item 1 CNR decision needed.** Does Montenegrin (`cnr`) get its own legal documents, or alias to Serbian? Conventions Part 9 says CNR aliases to SR. If that extends to legal content, the web fix maps `cnr` → `sr` and no separate CNR files are needed. If legal counsel says CNR needs distinct content, two additional files are required.

- **Item 1 issues.md entry minor corrections.** The entry says `privacy.md` / `terms.md`; actual filenames are `privacy-policy.en.md` / `terms-of-use.en.md`. Line `:14` should be `:15` (TODO) / `:21-23` (URL). These are cosmetic; the substantive claim is correct.

- **Item 2 fix is self-contained in web.** No cross-repo dependency. The fix touches `ProductList.tsx` and optionally `productsSearchService.ts` (client-side). Applies to all seven ProductList-using surfaces uniformly. Option A (AbortController) is the richer fix but requires service signature change; Option B (sequence counter) is simpler and sufficient for the UX problem. Recommend Option A for consistency with the autocomplete pattern already in the codebase.

- **Item 3 fix is self-contained in web.** No cross-repo dependency. Four files touched (`translationsCache.ts`, `internalRequest.ts`, `request.ts`, and optionally diagnostic logging adjustment). Eliminates 132 redundant cache entries. No tests to update. The granular tag format `translations:{NS}:{lang}:{tenant}` mentioned in `/api/revalidate` route comments (line 23) would need to drop the `:{tenant}` segment — but since that tag format is aspirational/unused (the actual `unstable_cache` uses only the static `'translations'` tag), the comment update is cosmetic.

### Adjacent observations (Part 4b)

1. **MarkdownViewer's `force-cache` + 1-hour revalidate could serve stale legal content after a locale fix ships.** At `MarkdownViewer.tsx:13-15`, `fetch(url, { cache: 'force-cache', next: { revalidate: 3600 } })` means newly-committed per-locale markdown files won't appear for up to an hour after the web deploy that adds the locale-suffixed URLs. Low severity — the 1-hour window is bounded and the content was already stale (English in all locales). File path: `src/components/server/MarkdownViewer.tsx:13-15`. I did not fix this because it is out of scope.

2. **The `revalidate` route's tag comment (line 22-30) lists `translations:{NS}:{lang}:{tenant}` as an available tag, but `unstable_cache` uses only the static `'translations'` tag.** The granular tag format is documented but non-functional. Low severity — misleading comment, no runtime impact. File path: `app/api/revalidate/route.ts:23`. I did not fix this because it is out of scope.
