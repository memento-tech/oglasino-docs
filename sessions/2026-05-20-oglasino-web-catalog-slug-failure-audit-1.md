# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-20
**Task:** Audit catalog slug failure modes: current state + recommended fix (read-only; output an audit summary).

## Implemented

- Read-only audit. No code touched.

## Files touched

- (none — read-only)

## Tests

- Not run (read-only).

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (the two 2026-05-14 catalog-slug entries are the inputs to this audit, not amended by it; once a fix lands, Docs/QA will flip them)

## Obsoleted by this session

- nothing (read-only audit)

## Conventions check

- Part 4 (cleanliness): confirmed (no code changes)
- Part 4a (simplicity): N/A — read-only audit; no abstractions added, considered, or removed
- Part 4b (adjacent observations): one adjacent observation flagged in "For Mastermind"
- Part 6 (translations): N/A this session (audit names missing translation keys for the proposed fix; seeding them is Backend's job once a fix is briefed)
- Other parts touched: Part 8 (Architectural defaults) — server-as-trust-boundary informs the fix shape (server resolves, server decides 404)

## Known gaps / TODOs

- none

---

# Audit — Catalog slug failure modes

## 1. Current state

Three observably-different outcomes today for what is logically the same condition: a catalog slug that does not resolve against `baseSite.catalog.categories`. The three paths share the same resolver (`getCategoriesFromPathSlugs`) but diverge in how each "miss" is handled.

### 1a. First-slug miss → server redirect to home

**File:** `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx:89-94`

```tsx
const { slugs = [] } = await params;
const categories = getCategoriesFromPathSlugs(baseSite.catalog, slugs);

if (!categories) {
  redirect(`/${baseSite.code}-${oglasinoLocale}`);
}
```

`getCategoriesFromPathSlugs` (lines 28-47 of the same file) returns `undefined` when `topCategory` cannot be found:

```tsx
const topCategory = catalog.categories?.find((cat) => cat.route === '/' + slugs[0]);
if (!topCategory) return undefined;
```

- **What "not found" means here:** `undefined` returned from the helper. The condition `if (!categories)` catches it.
- **What happens on miss:** `redirect()` from `next/navigation` is called with the localized base-site home URL. Next.js issues an HTTP 307 (temporary redirect) by default. Status is **307**, not 404.
- **`notFound` import status:** the file does **not** currently import `notFound` from `next/navigation`. Only `redirect` is imported (line 19).
- **Empty-slugs edge:** when `slugs = []` (i.e., the bare `/catalog` URL), `slugs[0]` is `undefined`, the `.find` cannot match, `topCategory` is `undefined`, and the helper returns `undefined` — so the bare `/catalog` URL also currently 307s to home. This is not in the brief's intended-behavior list (the brief talks about *bad* slugs, not the empty case); see "Open questions" §5.

### 1b. Sub/final-slug miss → silent fallback to parent

**File:** same file, lines 38-44

```tsx
if (slugs[1]) {
  subCategory = topCategory.subcategories?.find((cat) => cat.route === '/' + slugs[1]);
}

if (slugs[2]) {
  finalCategory = subCategory?.subcategories?.find((cat) => cat.route === '/' + slugs[2]);
}

return { topCategory, subCategory, finalCategory };
```

- **What "not found" means here:** `subCategory` or `finalCategory` is left as the initial `undefined` (declared on lines 35-36). The miss is **not** signaled — the helper returns an object with the resolved levels and `undefined` for the missed levels.
- **What happens on miss:** the page renders. The downstream consumers (`getPortalProducts`, line 98-106; the `getBottomCategoryName` helper, lines 114-124; `SelectableFilterProductListWrapper`, line 136-146) all chain on `?.id`, so they degrade to the deepest resolved level. `/catalog/cars/bogus` is functionally identical to `/catalog/cars`.
- **Why this matters:** there is no error signal upstream of render. The page returns 200, the user sees the parent category's products, the breadcrumb chain (resolved separately in `ProductBreadcrumbs`, see §1c) shows only the resolved level. To a tester this reads as "the sub-filter is broken" rather than "this slug doesn't exist."

### 1c. Empty breadcrumb chain on `/catalog` → client redirect to `/error`

**File:** `src/components/client/product/ProductBreadcrumbs.tsx:55-61`

```tsx
useEffect(() => {
  if (!baseSite) return;

  if (pathname.startsWith('/catalog') && categoryBreadcrumbData.length === 0) {
    window.location.href = '/error';
  }
}, [pathname, categoryBreadcrumbData, baseSite]);
```

- **What "empty chain" means here:** `categoryBreadcrumbData` is computed in the `useMemo` at lines 19-53. For a `/catalog/...` pathname it calls `resolveCategoryChain(pathname)` (line 49), which walks `pathname.split('/')` against `baseSite.catalog.categories`. The loop **`break`s on the first miss** (line 34: `if (!found) break`). If the very first segment misses, the chain is empty (`length === 0`); if a later segment misses, the chain is truncated but **non-empty**, so this guard does not fire on sub-slug misses.
- **What `/error` resolves to:** there is **no `/error` route** in the app tree (`grep` confirms: only `app/error.tsx`, `app/global-error.tsx`, `app/not-found.tsx`; no `app/error/page.tsx` or `app/[locale]/error/page.tsx`). `window.location.href = '/error'` triggers a full page navigation; Next.js has no matching segment, so it falls through to `app/not-found.tsx`. The user ends up at the 404 surface — but via a full page reload through a phantom URL, with HTTP **200** for `/error` followed by Next's 404-rendering (the response code depends on how the deployment hosts the rendered not-found at an unmatched route; for App Router it is 404, but the path the browser ends on is `/error`, not the original `/catalog/...`).
- **Reachable only from `/catalog`:** the `pathname.startsWith('/catalog')` guard (line 58) scopes the fallback. `ProductBreadcrumbs` is also rendered from the product detail page (`app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx:89`), but on that route `pathname.startsWith('/catalog')` is false, so the `/error` redirect cannot fire there. The fix can change this branch without regressing the product surface.
- **Race condition observable today:** because `baseSite` comes from `useBaseSiteStore` (client-side Zustand store), the `useEffect` runs after hydration. On the first paint where `baseSite` is still being populated, the effect early-exits (line 56); on the second paint, the effect re-evaluates. The fallback only fires once `baseSite` is truthy *and* the chain is empty. This is correct, just worth noting because the brief asks what "empty chain" means in code.

### 1d. Resolver shape and signaling — the unified view

`getCategoriesFromPathSlugs` is the single resolver. It signals "miss" in **two different ways** within the same function:

| Position | Signal on miss              | Caught by                         |
| -------- | --------------------------- | --------------------------------- |
| slug[0]  | returns `undefined`         | `if (!categories) redirect(...)`  |
| slug[1]  | leaves `subCategory` as `undefined` inside the returned object | nobody — silent fallback to parent |
| slug[2]  | leaves `finalCategory` as `undefined` inside the returned object | nobody — silent fallback to parent |

The first-slug path uses the unfindable-helper signal; the deeper paths do not. That asymmetry is the root cause of both `issues.md` entries.

### 1e. Catalog route is a Server Component with request-time data fetching

`app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx` is an `async` default export (line 80). It awaits `getBaseSiteServer()` at line 84 and `getPortalProducts` at line 98. The miss is detectable **before** any render — `getCategoriesFromPathSlugs` runs at line 90, immediately after the data is in hand and before any downstream awaits or JSX. This is the ideal point to call `notFound()` from `next/navigation`: same place the existing `redirect()` is called, same await context. No bubble-up needed.

`generateMetadata` (lines 49-76) runs in the same Server Component context and already short-circuits with `return {}` on a miss (lines 63-65). Once `notFound()` lands in the page body, metadata still needs to handle the miss without crashing; returning empty metadata as it does today is fine, but consider whether the metadata for the not-found surface should be set explicitly (see "Open questions" §3).

---

## 2. Existing not-found surface

### 2a. File location and content

**File:** `app/not-found.tsx` (root-level, not locale-scoped).

```tsx
import { TranslationNamespaceEnum } from '@/src/translations/types/TranslationNamespaceEnum';
import { getTranslations } from 'next-intl/server';
import Link from 'next/link';

export default async function NotFound() {
  const t = await getTranslations(TranslationNamespaceEnum.BUTTONS);
  return (
    <div className="mt-12 flex min-h-[30vh] w-full flex-col items-center justify-center">
      <img src="/404error.png" alt="404 error" className="..." />
      <Link href="/" ...>
        {t('back.home.label')}
      </Link>
    </div>
  );
}
```

- Renders a 404 illustration and a "back home" button (translated via the `BUTTONS` namespace, key `back.home.label`).
- **No heading text and no body copy** — the visual is the `/public/404error.png` image alone plus the CTA button.
- **There is no `app/[locale]/not-found.tsx`.** Confirmed by `find app -name not-found.tsx` → only the root file exists.

### 2b. How it behaves when triggered from a deeply-nested locale route

This is the one item the brief asks to flag if not fully determinable from code. Behavior from code reading:

- Next.js App Router renders the **closest `not-found.tsx` ancestor**. From `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx`, the closest ancestor is `app/not-found.tsx` (root). Result: the root not-found page renders.
- **Layouts wrapping the not-found render:** only `app/layout.tsx` (the root). The `[locale]/layout.tsx` is **bypassed** because it sits below the not-found's segment. Concretely, the `NextIntlClientProvider`, `BaseSiteInit`, `AuthInit`, `DrawerDialogManager`, `Toaster`, and the locale-setting `setRequestLocale(locale)` call all live in `[locale]/layout.tsx` and **do not apply** to the not-found render.
- **Practical consequence:** `getTranslations(TranslationNamespaceEnum.BUTTONS)` in `not-found.tsx` runs without a `setRequestLocale` having been called for that render. next-intl's server `getTranslations` reads request-scoped locale via its request config; whether that resolves correctly without the per-segment `setRequestLocale` depends on how `src/i18n/request.ts` (or wherever the next-intl config lives) detects locale. The translation may resolve correctly via headers/cookie fallback, or it may fall back to the default locale. **This needs a runtime check** — open `/catalog/bogus-slug` after the fix and confirm the "Back home" button text renders in the URL's locale, not always EN.
- **`notFound()` already works from a locale route in practice** — `app/[locale]/layout.tsx` calls it at lines 25 and 33 today on bad-locale and missing-baseSite cases respectively. Whatever happens for those two paths is what the catalog fix would inherit. Verifying it renders cleanly under a non-EN locale is the runtime check that matters.

### 2c. The `/error` page (referenced by `ProductBreadcrumbs.tsx`)

- **There is no `/error` route.** As noted in §1c, `grep` confirms no `app/error/page.tsx`, no `app/[locale]/error/page.tsx`, no folder named `error/`. The only files containing "error" in the app tree are `app/error.tsx` (route error boundary) and `app/global-error.tsx` (top-level error boundary). Neither is a route.
- `app/error.tsx` is a **route error boundary** — it renders only when a Server/Client Component on a sibling/descendant route throws. It is not navigable.
- `app/global-error.tsx` is the **top-level error boundary** — it renders only when a root error escapes the layout chain. Also not navigable.
- **So `window.location.href = '/error'` today lands the user at Next's unmatched-route handling**, which renders `app/not-found.tsx`. The user sees the 404 surface, but the URL bar reads `/error` (or rather `/{baseSite}-{locale}/error` after locale middleware), which is a worse UX than landing on the not-found surface at the original URL.

### 2d. Difference between not-found and error surfaces

| Surface                | Triggered by                            | URL preserved | HTTP status              | Translated copy        |
| ---------------------- | --------------------------------------- | ------------- | ------------------------ | ---------------------- |
| `app/not-found.tsx`    | `notFound()` call, or unmatched route   | yes (when `notFound()` is called); no (when client-side nav to `/error`) | 404 from server-rendered miss | yes (`BUTTONS.back.home.label`) |
| `app/error.tsx`        | Server/Client throw in a sibling route  | yes           | 500 (the throw is genuine) | no — hardcoded EN copy (`"Something went wrong"`) |
| `app/global-error.tsx` | Root-level throw                        | yes           | 500                      | no — hardcoded EN copy |

Today's `window.location.href = '/error'` is functionally landing the user on the not-found surface, but via the worst possible route (full reload, phantom URL, wrong-status response from the user's perspective). The fix should make this a direct `notFound()` call at the server.

---

## 3. Proposed fix

### 3a. `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx`

**One change to the resolver and one change to the guard.** The resolver becomes strict — every supplied slug must resolve, else the resolver returns `undefined`. The guard at line 92 swaps `redirect()` for `notFound()`.

Concretely, in `getCategoriesFromPathSlugs` (lines 28-47):

- If `slugs[1]` is supplied **and** `subCategory` is `undefined` after the lookup → return `undefined`.
- If `slugs[2]` is supplied **and** `finalCategory` is `undefined` after the lookup → return `undefined`.
- Otherwise return the resolved triple as today.

In the page body (lines 89-94):

- Replace `redirect(\`/${baseSite.code}-${oglasinoLocale}\`);` with `notFound();`.
- Add `import { notFound } from 'next/navigation';` (the file already imports `redirect` from the same module on line 19 — swap it).
- The `redirect` import can be removed if no other caller in this file needs it (none does — confirmed by reading the file end-to-end). Cleaner to drop unused imports.

In `generateMetadata` (lines 49-76):

- The strict resolver will return `undefined` for all three miss cases. The existing `if (!categories) return {};` already covers this — no change needed there. Worth confirming the metadata-on-404 outcome is acceptable (see "Open questions" §3).

Result: all three resolver miss cases (first, sub, final slug) take the same `notFound()` path. The page never renders a partial result.

### 3b. `src/components/client/product/ProductBreadcrumbs.tsx`

**Delete the `useEffect` at lines 55-61.** Once §3a lands, the catalog page never renders with a slug whose top-level segment is unresolvable — `notFound()` short-circuits before `ProductBreadcrumbs` mounts. The `categoryBreadcrumbData.length === 0` condition on `/catalog` cannot occur in practice.

If a residual concern exists (e.g., a stale cached `baseSite` snapshot in the Zustand store from a prior session, so the client-side `resolveCategoryChain` still produces an empty chain even though the server resolved successfully), the safer client behavior is to **render an empty `OglasinoBreadcrumbs`** (the component already accepts an empty array — line 65 passes it through unchanged), not to navigate. The current "redirect to phantom `/error`" path is strictly worse than rendering an empty breadcrumb bar on a transient client-state mismatch.

Recommended:

- Remove the `useEffect` (lines 55-61) and the `useEffect` import if no other effect remains (there is none in this file — confirmed).
- Result: `ProductBreadcrumbs` becomes purely declarative — it renders whatever chain the memo produces, including the empty case, and lets the server be the only authority on "this catalog URL is bad."

### 3c. Existing not-found surface

**No change required** — `app/not-found.tsx` is the surface the fix routes to. The 404 illustration and "back home" CTA are already in place.

Two optional polish items, scoped out of this brief by §"Out of scope":

- **Locale-scoped not-found.** Adding `app/[locale]/not-found.tsx` would let the `[locale]/layout.tsx` (and its `NextIntlClientProvider` + `setRequestLocale`) wrap the not-found render, removing the dependency on next-intl's default-locale fallback. The current root `not-found.tsx` would stay as the unmatched-locale fallback.
- **"Category not found" heading copy.** The brief calls the intended behavior a "category not found" message. Today's `not-found.tsx` has no heading text — only the 404 image and a button. A new key in `ERRORS` (e.g., `not.found.title` and `not.found.body`) could give the page actual prose. This is a content question for Mastermind, not an audit recommendation.

---

## 4. HTTP status outcome

| Path                    | Today                                                                | After proposed fix             |
| ----------------------- | -------------------------------------------------------------------- | ------------------------------ |
| Bad first slug          | 307 redirect to home                                                 | **404** via `notFound()`       |
| Bad sub/final slug      | 200 with parent's product list                                       | **404** via `notFound()`       |
| Empty breadcrumb chain  | 200 + client-side full reload to `/error` → renders not-found at the phantom URL | unreachable (server short-circuits before render); 404 already issued upstream |

**All three paths reach a clean 404.** The proposed fix does not have to fall back to a less-than-404 outcome for any case — the breadcrumb-chain fallback becomes unreachable rather than client-handled, which is the better shape.

One caveat on the empty-`slugs` case (the bare `/catalog` URL with no slugs at all, §1a): the strict resolver would return `undefined` for `slugs = []` too (because `slugs[0]` is `undefined`, `.find` cannot match, `topCategory` is `undefined`). Today the bare `/catalog` 307s to home — under the proposed fix it would 404. This may or may not be desired. See "Open questions" §5.

---

## 5. Risks and regressions

### 5a. Anything in the codebase relying on the silent-parent-fallback behavior?

`grep`-checked. The sub/final-slug silent fallback's only consumers are inside `[[...slugs]]/page.tsx` itself — `getPortalProducts` (line 98), `getBottomCategoryName` (lines 114-124), and the JSX (`SelectableFilterProductListWrapper` at line 136, the no-products-empty-state at line 148-166). All of these chain on `?.id` and degrade quietly. **Nothing outside this file depends on the silent fallback.** No upstream caller of `getCategoriesFromPathSlugs` exists either — it is a local helper.

### 5b. SEO / sitemap

- **Sitemap.** A bad slug landing on a 404 is correct SEO behavior — better than a soft-redirect to home (search engines can be misled into thinking the slug *is* the home URL). Sitemap is not enumerated from slugs; it is generated from the catalog category tree, so a non-existent slug was never sitemap-ed in the first place.
- **`generateMetadata` already returns `{}`** for a miss today, so search engines don't get OG/title data for bad slugs. After the fix, the 404 page's default metadata applies (no `title` defined in `app/not-found.tsx`, which means the document title falls back to whatever the root layout sets). Worth a minor decision — see "Open questions" §3.

### 5c. Cached responses (Next.js data cache / Vercel edge cache)

- The catalog page does not use `force-static`, `revalidate`, or any explicit cache directives at the route level. It is fully dynamic per request (the page awaits `searchParams`, `params`, `cookies()` — all of which mark the route as dynamic).
- `notFound()` from a dynamic Server Component produces a 404 response that is **not** cached by Next.js's data cache. No risk of a bad-slug 404 being cached and replayed against a later-valid slug.
- The Cloudflare router worker does not have catalog-route-specific caching rules (per `oglasino-router` CLAUDE.md and conventions Part 8); it forwards transparently. No risk there.

### 5d. The `/error` URL appearing in browser history

Today, when the breadcrumb fallback fires, the URL bar momentarily reads `/{baseSite}-{locale}/error` before the 404 surface renders. That URL is now in the browser's history stack — pressing back returns the user to `/error` again, not to the original `/catalog/...`. After the fix, the URL stays at the original `/catalog/...` and the 404 renders in-place. **This is a UX win, but worth calling out** because automated tests or bookmarks that exercise the old `/error` URL will behave differently (the URL was never a real route, so this should be a non-issue, but flagging for completeness).

### 5e. The QA topics that cross-reference these `issues.md` entries

`category-navigation` and `catalog-page` topics in `app/[locale]/design/topics.ts` carry "Known issue" pitfalls that pin the current (broken) behavior. The QA Preparation feature spec calls out this dependency explicitly — when the fix lands, the topic pitfalls must flip. This is a **follow-up task for the Docs/QA agent**, not a regression risk in the fix itself. Tracked in `state.md`'s QA Preparation entry ("Tasks remaining" already lists topic-flip work that will need to absorb this).

### 5f. Strict resolver also catches the bare `/catalog` URL

Already noted in §4. The bare `/catalog` URL currently 307s to home; under the strict resolver it would 404. Decision needed — see "Open questions" §5.

### 5g. Header search box, category dialog, and direct URL all hit the same resolver

These three entry points (per the `category-navigation` QA topic and the catalog page's audit) all build URLs that flow through `[[...slugs]]/page.tsx`. They all benefit from the unified 404 behavior. **No additional sites to fix.**

---

## 6. Open questions for Igor

1. **Bare `/catalog` URL with no slugs.** Should `/catalog` (zero slugs) 404 (strict resolver), 200 with all-products (treat as "browse all"), or 307 to home as today? The brief says "a bad slug at any level should render a 'category not found' error" — bare `/catalog` is not a bad slug per se, it's no slug. Three reasonable answers: (a) keep the current 307-to-home behavior with an explicit `if (slugs.length === 0) redirect(...)` guard before the resolver; (b) treat as a 404 like any other miss; (c) render an "all categories" page. The fix shape changes slightly depending on the answer. Recommendation: keep today's 307-to-home with an explicit `slugs.length === 0` guard at the top of the page body, so the fix narrows in scope to "bad slug = 404."

2. **Locale-scoped not-found.** The current root `app/not-found.tsx` is bypassed by `[locale]/layout.tsx`'s `NextIntlClientProvider` and `setRequestLocale`. Whether the existing key `BUTTONS.back.home.label` renders in the URL's locale on a bad-slug 404 depends on next-intl's default-locale-fallback configuration. Two options: (a) accept the current root-only not-found and runtime-verify the locale fallback works; (b) add `app/[locale]/not-found.tsx` to inherit the locale layout's providers. Option (b) is the safer fix; option (a) is the smaller fix. Brief says "scoped to `[[...slugs]]/page.tsx`, `ProductBreadcrumbs.tsx`, and the not-found / error surfaces" — both options stay in scope. Recommendation: (b) if a runtime check shows EN-fallback under non-EN URLs; (a) otherwise.

3. **Metadata on the 404 surface.** `generateMetadata` returns `{}` on a miss today. After the fix, a bad-slug request still calls `generateMetadata` first, then hits `notFound()`. The empty metadata applies to the 404 render, which means no `<title>`, no OG tags. Acceptable, or should the 404 surface get its own metadata via a new `ERRORS.not.found.title` key? Open question, not a blocker. Recommendation: out of scope for this fix — file a separate small task if Mastermind wants explicit 404 metadata.

4. **"Category not found" prose.** The current `not-found.tsx` has no heading text — only the 404 image and a "Back home" button. The brief mentions "category not found" as the intended outcome but does not specify whether to add prose. Recommendation: leave the surface as-is for this fix (smallest change that closes both `issues.md` entries); if heading copy is wanted, file a separate task with a new `ERRORS` namespace key.

5. **Whether to keep `redirect` import in the catalog page after the swap.** The file imports `redirect` from `next/navigation` (line 19) and uses it only on the line being swapped to `notFound()`. After the swap, `redirect` becomes unused. Drop it (cleanliness Part 4) unless §6.1 keeps a redirect for the bare-`/catalog` case. Mention here so the engineer agent receiving the fix brief knows the import audit applies.

---

## For Mastermind

- **Part 4b adjacent observation (one item):** `ProductBreadcrumbs.tsx`'s `useEffect` (lines 55-61) issues a `window.location.href` navigation as a "safety net" — the comment-style pattern of "if data is inconsistent, hard-reload the user out of this page" is worth flagging beyond the current bug. It exists in this one component today. If similar guards land in other client components over time, the codebase accumulates phantom-URL navigation paths. Severity low (this one is being removed by the proposed fix; the observation is about the pattern, not the code line). No second occurrence found on a quick `grep` for `window.location.href.*=` across `src/` — only this one and a couple in `AppInit`-style reload-after-error paths that look intentional. Flagging here so Mastermind can decide whether to formalize a "no `window.location.href` for control-flow" convention if more instances appear.

- **No `decisions.md` / `state.md` / `issues.md` / `conventions.md` drafts produced by this audit.** This is a read-only audit; the next engineer session that implements the fix will produce the cleanup drafts (issues.md status flips on the two 2026-05-14 entries, and QA topic pitfall flips on `category-navigation` and `catalog-page`).

- (No further notes flagged.)
