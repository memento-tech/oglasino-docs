# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-28
**Task:** Audit `skipAuth` usage footprint (informs #8 fix choice). READ-ONLY.

## Implemented

- Read-only audit of every `skipAuth` call site in `oglasino-web`, the wrappers that honor the flag, the four required attributes per call site, and the dynamic-rendering posture of the two RSC consumer routes for `getUserForId` (`user/[userId]/page.tsx`, `product/[productId]/[productName]/page.tsx`). No code changes.

## Files touched

- (none)

## Files read

- `src/lib/config/fetchApi.ts` (the `skipAuth` doc-comment at lines 15–20 and gate at 38–45)
- `src/lib/config/api.ts` (the client-side axios `BACKEND_API` — confirms `skipAuth` is not part of its surface)
- `src/lib/service/nextCalls/userService.ts` (`getUserForId`, the bug-#8 site)
- `src/lib/service/nextCalls/productsSearchService.ts` (`getProductDetails`, `getProducts`)
- `src/lib/service/nextCalls/metadataSnapshotService.ts` (`getMetadataProductsSnapshot`)
- `src/lib/service/nextCalls/productService.ts` (`markAsSeen` — note: no `skipAuth`, runs in the product page render path)
- `app/actions/getBaseSiteServer.ts` (`getBaseSiteServer`, `getAllBaseSitesOverviews`)
- `app/sitemap.ts` (three skipAuth call sites: overviews, search, count)
- `app/[locale]/(portal)/(public)/user/[userId]/page.tsx` (RSC consumer of `getUserForId`)
- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` (RSC consumer of `getUserForId`)
- `src/lib/types/user/UserInfoDTO.ts`, `src/lib/types/catalog/BaseSiteDTO.ts`, `src/lib/types/catalog/BaseSiteOverviewDTO.ts`, `src/lib/types/product/ProductDetailsDTO.ts`, `src/lib/types/product/ProductOverviewDTO.ts` (to classify viewer-dependent fields)
- `src/components/client/buttons/FavoriteButton.tsx`, `src/components/client/product/ProductList.tsx`, `src/lib/store/useFavoritesStore.ts` (to verify whether `ProductOverviewDTO.favorite` is client-overridden post-mount)
- Prior audit `2026-05-28-oglasino-web-isfollowing-seed-audit-1.md` (already-known data-path findings, kept consistent)
- Repo-wide grep: `skipAuth`, `BACKEND_API|FETCH_BACKEND_API`, `getMetadataProductsSnapshot`, `getBaseSiteServer|getAllBaseSitesOverviews|getPortalProductDetails|getUserForId`, `useFavoritesStore`

## Audit

### Section 1 — The mechanism

**`src/lib/config/fetchApi.ts` doc-comment (lines 15–20), verbatim:**

> When true, do NOT call cookies() or attach the Firebase token to the request. Use this for /public/* endpoints — calling cookies() opts the route into dynamic rendering and forwards per-user cookies to the backend, which can defeat caching. Default: false (auth attached).

**`src/lib/config/fetchApi.ts` inline comment at the implementation (lines 32–35), verbatim:**

> Only read cookies and the Firebase token for authenticated requests. Calling cookies() in a Server Component opts the route into dynamic rendering — we don't want that on /public/* fetches because it disables the Data Cache for the surrounding route.

**What `skipAuth: true` strips (lines 36–45):**

```
let cookieHeader = '';
let firebaseToken: string | undefined;
if (!skipAuth && typeof window === 'undefined') {
  const cookieStore = await cookies();
  firebaseToken = cookieStore.get('firebase_token')?.value;
  const allCookies = cookieStore.getAll();
  if (allCookies.length > 0) {
    cookieHeader = allCookies.map((c) => `${c.name}=${c.value}`).join('; ');
  }
}
```

With `skipAuth: true`, both `cookieHeader` and `firebaseToken` remain empty; the outgoing `fetch` carries neither `Cookie:` nor `Authorization: Bearer ...`. The `typeof window === 'undefined'` gate means client-side calls (which can't read Next's `cookies()` anyway) never enter this branch — `skipAuth` is a no-op there. Server-side without `skipAuth` calls `await cookies()`, which is Next 15's dynamic API for cookie access and opts the surrounding route out of static rendering.

**Strict-Next-15 nuance, for accuracy.** `cookies()` opts the route out of **static rendering / the Full Route Cache**. Individual `fetch()` calls within a dynamic route can still hit the **Data Cache** if their inputs are stable. The doc-comment phrasing "disables the Data Cache for the surrounding route" conflates the two; what's strictly true is that, with `skipAuth: false`, the per-request `Cookie:` + `Authorization:` headers vary the per-fetch cache key, so the Data Cache fragments per viewer — that's the practical cache loss, distinct from the route-level static-rendering loss. Both forms of cost are relevant to Fix A; section 4 separates them.

**Which wrapper(s) honor `skipAuth`.**

- `FETCH_BACKEND_API` (RSC-side, `src/lib/config/fetchApi.ts:89-98`) — wraps `request()` which destructures and gates on `skipAuth` (line 30). Honors the flag.
- `BACKEND_API` (client-side axios, `src/lib/config/api.ts:76`) — has no `skipAuth` surface. The axios request interceptor at lines 94–139 always attaches `Authorization: Bearer <cached-token>` when `auth.currentUser` exists, and the instance is constructed with `withCredentials: true` (line 18) so the browser sends cookies on cross-origin requests as a property of the browser environment, not as something `skipAuth` could gate. Client-side calls do not have the Data Cache concern (no `cookies()` API on the client, no Next route-level dynamic-rendering switch).

**Practical conclusion.** `skipAuth` is an RSC-only escape hatch on `FETCH_BACKEND_API`. The footprint we care about for the #8 decision lives in server-side `next/server-actions` and `app/` server files.

### Section 2 — Every call site

Eight occurrences of `skipAuth: true` across six files. All on `FETCH_BACKEND_API`. All RSC-side.

| # | File:line — function | Endpoint | RSC vs client | Viewer-dependent fields? | `revalidate` / tags today? |
|---|----------------------|----------|---------------|--------------------------|---------------------------|
| 1 | `app/sitemap.ts:33` — `getAllBaseSitesForSitemap` | `GET /public/baseSite/overviews` | RSC | No — `BaseSiteOverviewDTO[]` is `id/code/domain/labelKey/flagImageKey/defaultLanguage/allowedLanguages`. Catalog metadata. | No per-fetch `next` block; relies on the file-level `export const revalidate = 86400` covering the whole sitemap render |
| 2 | `app/sitemap.ts:68` — `getProductsPage` | `POST /public/product/search` | RSC | Yes-but-unused — `ProductOverviewDTO.favorite` is viewer-dependent, but the sitemap only reads `product.id` and `product.name` (line 175–179). Field is harmlessly defaulted. | Yes — `next: { revalidate }` set (file-level 86400) |
| 3 | `app/sitemap.ts:84` — `getProductCountForBaseSite` | `GET /public/product/count` | RSC | No — `{ count: number }` only. Per-baseSite, viewer-independent. | Yes — `next: { revalidate }` set (file-level 86400) |
| 4 | `app/actions/getBaseSiteServer.ts:21` — `getBaseSiteServer` | `GET /public/baseSite/{tenant}` | RSC (server action, called from layouts + pages) | No — `BaseSiteDTO` is catalog/currencies/languages/regions. Viewer-independent. | Yes — `revalidate: 86400` + tags `['base-site', 'base-site:${tenant}']` |
| 5 | `app/actions/getBaseSiteServer.ts:48` — `getAllBaseSitesOverviews` | `GET /public/baseSite/overviews` | RSC (server action) | No — same DTO as row 1. | Yes — `revalidate: 86400` + tags `['base-site', 'base-site:overviews']` |
| 6 | `src/lib/service/nextCalls/userService.ts:15` — `getUserForId` | `GET /public/user?id=<id>` | RSC | **Yes — bug #8.** `UserInfoDTO.isFollowingCurrent` and `UserInfoDTO.iamActive` are viewer-dependent; with `skipAuth: true` the backend has no viewer identity and defaults them. `iamActive` is client-recomputed in `UserDetails.tsx:55-61`; `isFollowingCurrent` is **not** corrected client-side. | Yes — `revalidate: 300` + tags `['user', 'user:${userId}']` |
| 7 | `src/lib/service/nextCalls/productsSearchService.ts:40` — `getProductDetails` (only when `portalScope === 'portal'`) | `GET /public/products?productId=<id>` | RSC | **Yes — latent twin of #8.** `ProductDetailsDTO` extends `ProductOverviewDTO` (which carries `favorite: boolean`) and embeds `owner: UserInfoDTO` (which carries `iamActive` and `isFollowingCurrent`). `favorite` is client-overridden via `useFavoritesStore` on the buttons that consume it (`FavoriteButton.tsx:37-38`, `ProductList.tsx:105`), so its skipAuth-stripped seed is invisible to users. `owner.isFollowingCurrent` rendered through `UserDetails` → `FollowUserButton` IS user-visible and is bug #8 manifested on the product page. | Yes — `revalidate: 60` + tags `['product', 'product:${productId}']` |
| 8 | `src/lib/service/nextCalls/metadataSnapshotService.ts:60` — `getMetadataProductsSnapshot` | `POST /public/product/search` | RSC (called from `generateMetadata` / structured-data helpers on the home and catalog pages) | No (as consumed) — `ProductOverviewDTO.favorite` is viewer-dependent but the snapshot mapper at lines 73–89 reads only `id/name/topImageKey/price/currency`. Drops the viewer-dependent surface. | Yes — `revalidate: 3600` + tags `buildSnapshotTags(filters)` (`metadata-snapshot-home` or `metadata-snapshot-catalog[-${id}]`) |

### Section 3 — The `getUserForId` call specifically

**`getUserForId` itself.** Confirmed `src/lib/service/nextCalls/userService.ts:13-20`:

```ts
const res = await FETCH_BACKEND_API.get<UserInfoDTO | null>('/public/user?id=' + userId, {
  headers: { 'X-Base-Site': tenant, 'X-Lang': lang },
  skipAuth: true,
  next: {
    revalidate: 300,
    tags: ['user', `user:${userId}`],
  },
});
```

`skipAuth: true`, `revalidate: 300`, tags `['user', 'user:${userId}']`. Inline comment at line 12: "Public profile data cached for 5 minutes; purge via revalidateTag(`user:${userId}`)."

#### Consumer route 1 — `app/[locale]/(portal)/(public)/user/[userId]/page.tsx`

- Calls `getUserForId(userId)` in both `generateMetadata` (line 32) and `UserPage` render (line 49).
- **Reads `cookies()` directly** at line 81: `const cookieStore = await cookies();` followed by `readGlobalCookieForSsr(cookieStore)`. This unconditionally opts the route into dynamic rendering.
- Calls `getPortalProducts({ ownerId: userId }, getInitialPage())` at line 77. `getPortalProducts` → `getProducts` (`src/lib/service/nextCalls/productsSearchService.ts:75-109`). `getProducts` does **not** pass `skipAuth` on any branch (portal or otherwise) and does not set `next: { revalidate }`. Inside `fetchApi`'s `request()` this triggers `await cookies()` — a second route-level dynamic opt-out.
- Also calls `getBaseSiteServer()` (skipAuth: true, cacheable) and `markAsSeen` is **not** called on this route.

**Verdict for user page.** Already dynamic for at least two independent reasons (explicit `cookies()` read at line 81; `getPortalProducts` reads `cookies()` transitively). Dropping `skipAuth` on `getUserForId` adds **zero** route-level cache cost. The Data Cache benefit on the `getUserForId` fetch itself would shift from "one entry shared across viewers (5-min revalidate)" to "one entry per viewer keyed by Authorization/Cookie headers" — that fetch-level fragmentation is the only real cost, and it exists irrespective of the route being already dynamic.

#### Consumer route 2 — `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`

- Calls `getUserForId(productDetails.ownerId)` at line 91, after `getPortalProductDetails(productId)` at lines 38/63.
- Does **not** read `cookies()` directly.
- Does **not** call `getPortalProducts`.
- **Calls `markAsSeen(productDetails.id)` at line 94**, gated on `!owner.iamActive`. `markAsSeen` (`src/lib/service/nextCalls/productService.ts:6-12`) calls `FETCH_BACKEND_API.get('/public/product/seen/' + productId)` with **no `skipAuth: true`** — so it enters the `await cookies()` branch of `fetchApi.ts:38-45`. Since `owner.iamActive` is itself viewer-dependent and stripped by `getUserForId`'s `skipAuth: true` (the field is defaulted, almost certainly `false` for the no-viewer case), the `!owner.iamActive` gate evaluates true for every render → `markAsSeen` runs on every render → `cookies()` is read on every render → the route is already dynamic in practice.
- `getBaseSiteServer()` and `getPortalProductDetails(...)` are both skipAuth-true and don't dynamize the route on their own.

**Verdict for product page.** Already dynamic transitively, via `markAsSeen`'s missing `skipAuth: true`. The status is slightly weaker than the user page's (where the dynamic opt-out is explicit and unconditional), but in practice it fires on every render because of the `iamActive` self-reinforcement: skipAuth strips `iamActive` → it reads as false-y → `markAsSeen` fires → cookies() is read → route is dynamic. Dropping `skipAuth` on `getUserForId` adds **zero** route-level cache cost. The fetch-level Data Cache fragmentation cost is identical to the user-page case.

### Section 4 — Verdict for the #8 decision

#### Per-call-site verdict

| # | Site | Verdict |
|---|------|---------|
| 1 | `getAllBaseSitesForSitemap` (sitemap) | Correctly viewer-independent, leave alone |
| 2 | `getProductsPage` (sitemap) | Returns viewer-dependent `favorite`, but unused by sitemap mapper. Leave alone — no user-visible bug, and removing skipAuth would not improve sitemap behavior. |
| 3 | `getProductCountForBaseSite` (sitemap) | Correctly viewer-independent, leave alone |
| 4 | `getBaseSiteServer` | Correctly viewer-independent, leave alone |
| 5 | `getAllBaseSitesOverviews` | Correctly viewer-independent, leave alone |
| 6 | `getUserForId` | **Viewer-dependent, same bug class as #8.** Fix A drops `skipAuth: true` here. |
| 7 | `getProductDetails (portal)` | **Viewer-dependent, same bug class as #8** for `owner.isFollowingCurrent` (the `favorite` field on the product itself is client-overridden, so it's not user-visible). Latent twin — flagged in "For Mastermind." |
| 8 | `getMetadataProductsSnapshot` | Returns viewer-dependent `favorite`, but the snapshot mapper drops it before any consumer sees it. Effectively viewer-independent as consumed. Leave alone. |

#### Is Fix A's cache cost real or zero — on the two `getUserForId` consumer routes?

**Route-level (static-rendering / full-route-cache) cost: ZERO.** Both consumer routes already opt out of static rendering today, by independent mechanisms:

- `user/[userId]/page.tsx` reads `cookies()` directly (line 81) and also calls `getPortalProducts` which reads `cookies()` transitively.
- `product/[productId]/[productName]/page.tsx` calls `markAsSeen` (no `skipAuth`) on every render in practice, which reads `cookies()` transitively.

Dropping `skipAuth: true` from `getUserForId` does not change either route's dynamic-vs-static posture — they are both already dynamic.

**Fetch-level (Data Cache) cost: NON-ZERO, but small.** Today the per-userId `getUserForId` fetch is shared across all viewers (one Data Cache entry per userId, 5-min revalidate). Post-Fix-A the fetch carries `Authorization: Bearer <user-token>` + `Cookie: <user-cookies>`, so the cache key effectively becomes (userId, viewer). For a popular user profile, that's a hit-rate reduction. The 5-minute window still holds per (userId, viewer), so a user who refreshes their own /user/X within 5 min still gets a Data Cache hit; cross-viewer sharing is lost.

**Net answer to the brief's question.** Fix A's *route-level* cache cost is **zero** on both consumer routes. Fix A's *fetch-level* cache cost is **real but small** (per-viewer Data Cache fragmentation, no full revalidate-window loss). The doc-comment in `fetchApi.ts` is conservative — it elides the route-vs-fetch distinction — but the routes it was protecting are already dynamic for unrelated reasons, so the protection is moot here.

## Cleanup performed

- none needed (read-only audit, no code changes)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — read-only audit, no commented code, no unused imports, no console.log, no TODO/FIXME added.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): flagged in "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: none

## Known gaps / TODOs

- The strict Next 15 distinction between "Full Route Cache" (static rendering) and "Data Cache" (per-fetch) is glossed in the `fetchApi.ts` doc-comment. This audit makes the distinction explicit in Sections 1 and 4 but does not propose rewording the comment — out of scope.
- The exact backend default for `iamActive` when no viewer identity is attached is inferred (almost certainly `false`) but not verified by reading backend code (which is out of scope per the brief). The "markAsSeen runs on every render" claim depends on this default — if backend defaults `iamActive` to true on the no-auth path, the product page is dynamic only for authenticated viewers. Either way the static-cache benefit is, at best, anonymous-viewer-only, and Fix A's relative cost on that subset is still small.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only audit; no code added).
  - Considered and rejected: nothing (no implementation decisions made).
  - Simplified or removed: nothing (no code removed).

- **Latent twin to #8 — `getPortalProductDetails`'s `owner.isFollowingCurrent` (severity: medium).**
  `ProductDetailsDTO` embeds `owner: UserInfoDTO`. Because `getProductDetails` passes `skipAuth: true` on the `portal` branch (`src/lib/service/nextCalls/productsSearchService.ts:40`), the owner's `isFollowingCurrent` is also non-authoritative — and the product detail page renders the owner's `FollowUserButton` via `UserDetails` (`product/[productId]/[productName]/page.tsx:151-157`). This is the same bug class as #8 on a second call site. Whatever fix is chosen for `getUserForId` (A/B/C from the prior `isfollowing-seed-audit`) should be applied uniformly to `getPortalProductDetails` so the product page and user page don't drift apart. I did not fix this because it is out of scope of this audit.

- **Adjacent observation — `markAsSeen` should arguably pass `skipAuth: true` (severity: low).**
  `src/lib/service/nextCalls/productService.ts:6-12` calls `FETCH_BACKEND_API.get('/public/product/seen/' + productId)` without `skipAuth`. The endpoint is `/public/...` and (judging from its name and the fire-and-forget call pattern in the product page) doesn't need viewer identity. As-is, it forwards per-user cookies + Bearer token to a public endpoint and forces the product page route into dynamic rendering. If the intent is "fire-and-forget anonymous view counter," setting `skipAuth: true` would be the consistent choice. Worth noting that today's `iamActive`-via-skipAuth feedback loop is what guarantees the product page stays dynamic — fixing `getUserForId` (Fix A) flips `iamActive` to its real value for authenticated viewers, which means `!owner.iamActive` becomes false on every page where the viewer is the owner, and `markAsSeen` stops firing on those renders. That's intended behavior (an owner shouldn't count as a viewer of their own product), but it does mean the product page's "already dynamic" status weakens after Fix A: it remains dynamic for non-owner authenticated viewers via `markAsSeen`, and becomes statically-rendering-eligible for the owner-as-viewer case. Not a blocker for Fix A; just a knock-on to be aware of. I did not fix this because it is out of scope.

- **Adjacent observation — inconsistency between `getProductDetails` and `getProducts` (severity: low).**
  `getProductDetails` switches `skipAuth` based on portal scope (line 39); `getProducts` (same file, lines 75–109) does not — it never passes `skipAuth` for any scope, including portal. That asymmetry means `getPortalProducts(...)` always reads cookies and dynamizes its caller's route, even though `POST /public/product/search` is the same public endpoint the sitemap uses with `skipAuth: true`. If `getProducts` were brought into line with `getProductDetails`'s scoped pattern (skipAuth for portal, no skipAuth for owner/admin), the user page would lose one of its two "already dynamic" mechanisms — but only one; the explicit `cookies()` read at line 81 would still hold. Worth flagging because this asymmetry is part of why the audit's "already dynamic" verdict is robust today. I did not fix this because it is out of scope.

- **Adjacent observation — doc-comment phrasing in `fetchApi.ts` (severity: very low).**
  The comments at lines 15–20 and 32–35 say `cookies()` "disables the Data Cache for the surrounding route." Strictly, `cookies()` opts the route out of *static rendering* (Full Route Cache); individual `fetch()` calls within a dynamic route can still hit the Data Cache (the Data Cache is per-fetch, not per-route). The practical effect the comment is gesturing at — per-user header fragmentation of the Data Cache when `Authorization`/`Cookie` headers vary — is real, but it's a fetch-level concern, not a route-level one. A more precise wording would help future readers reason about cases like Fix A. I did not fix this because it is out of scope and the comment's practical guidance is still sound.

- **Recommendation for the #8 decision (not a fix, just framing).**
  Given the audit findings, Fix A's *route-level* cache cost is zero on both consumer routes today, and its *fetch-level* cost is bounded (per-viewer fragmentation of the 5-min Data Cache, no full revalidate-window loss). The other candidate fixes from the prior `isfollowing-seed-audit` (B: drop the field from the public endpoint + add a small `/secure/follow/status` call; C: client-side recompute via a follows-store) trade slightly different cache postures against round-trip and complexity costs. Fix A is the cheapest in lines-of-code and least-invasive, and its cache concern is smaller than the doc-comment's framing suggests. The unresolved question for Mastermind is whether the per-viewer Data Cache fragmentation is acceptable for the user and product pages' actual traffic profile — that's a product/operational call, not a code-shape call. Whatever choice is made should also be applied to `getPortalProductDetails` (latent twin above).

- (nothing else flagged)
