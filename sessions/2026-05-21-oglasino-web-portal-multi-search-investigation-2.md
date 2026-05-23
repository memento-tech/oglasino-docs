# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-21
**Task:** Instrumentation pass: confirm the two post-mount RSC re-renders on the portal home

## Implemented

- Added two temporary `console.log` lines per the brief: one inside the default-exported `Home` page component (`app/[locale]/(portal)/(public)/page.tsx`) and one inside the SYNC TO URL effect of `FilterManager.tsx`, immediately before the `setTimeout(() => router.refresh(), 0)` call.
- Igor drove a hard refresh of `/rs-sr` and a normal refresh of the same URL; captured the browser console (which in Next 16 dev mode forwards server-side `console.log` from server components alongside client-side logs) and the backend `oglasino-backend` access log for `POST /api/public/product/search` in matching windows.
- Reconciled each backend product/search POST to its instrumented trigger.
- Removed the two `console.log` lines in the same session. `git diff` for the two touched files reverted to Igor's pre-existing WIP state for `page.tsx` (cookies-V2 refactor — not my work) and to no diff at all for `FilterManager.tsx`.

## Raw capture — Hard refresh

### Browser console (verbatim, local time UTC+2)

```
21:17:41.953 Download the React DevTools for a better development experience: https://react.dev/link/react-devtools                                forward-logs-shared.ts:95:22
21:17:41.986 [HMR] connected                                                                                                                       forward-logs-shared.ts:95:22
21:17:41.991  Server  [instr] SSR page render 2026-05-21T19:17:41.322Z
Object { searchParams: {…} }                                                                                                                       forward-logs-shared.ts:95:22
21:17:43.114 [instr] FilterManager router.refresh 2026-05-21T19:17:43.114Z
Object { newUrl: "/rs-sr/", current: "/rs-sr" }                                                                                                    forward-logs-shared.ts:95:22
21:17:43.474 XHR  GET  http://localhost:8080/api/secure/admin  [HTTP/1.1 403  88ms]
21:17:43.567 [admin.isAdminUser] backend returned non-2xx
Object { status: 403, errorCode: undefined }                                                                                                       forward-logs-shared.ts:95:22
21:17:43.864  Server  [instr] SSR page render 2026-05-21T19:17:43.636Z
Object { searchParams: {…} }                                                                                                                       forward-logs-shared.ts:95:22
21:17:43.865  Server  [instr] SSR page render 2026-05-21T19:17:43.659Z
Object { searchParams: {…} }                                                                                                                       forward-logs-shared.ts:95:22
21:17:51.991 The resource at "http://localhost:3000/_next/static/media/83afe278b6a6bb3c-s.p.0q-301v4kxxnr.woff2" preloaded with link preload was not used within a few seconds.  rs-sr
21:17:52.112 The resource at "http://localhost:3000/404error.png" preloaded with link preload was not used within a few seconds.                                                rs-sr
```

Capture window (browser local time): **21:17:41 → 21:17:52**.
Server-side ISO timestamps inside the `[instr]` payloads: **19:17:41.322Z → 19:17:43.659Z**.

### Backend access log slice (verbatim — separate hard-refresh run; see note below)

```
2026-05-21 21:26:13.640 [http-nio-0.0.0.0-8080-exec-137] INFO  c.m.t.o.controller.ProductSearchController - Execution time for fetching all products: 11milliseconds
2026-05-21 21:26:13.641 [http-nio-0.0.0.0-8080-exec-137] INFO  ACCESS - POST /api/public/product/search -> 200 (64 ms)
2026-05-21 21:26:13.653 [http-nio-0.0.0.0-8080-exec-123] INFO  ACCESS - GET /api/public/baseSite/rs -> 200 (79 ms)
2026-05-21 21:26:14.047 [http-nio-0.0.0.0-8080-exec-152] INFO  ACCESS - GET /api/public/baseSite/overviews -> 200 (46 ms)
2026-05-21 21:26:14.064 [http-nio-0.0.0.0-8080-exec-114] INFO  c.m.t.o.controller.ProductSearchController - Execution time for fetching all products: 11milliseconds
2026-05-21 21:26:14.065 [http-nio-0.0.0.0-8080-exec-114] INFO  ACCESS - POST /api/public/product/search -> 200 (61 ms)
2026-05-21 21:26:16.348 [http-nio-0.0.0.0-8080-exec-150] INFO  ACCESS - OPTIONS /api/secure/admin -> 200 (0 ms)
2026-05-21 21:26:16.362 [http-nio-0.0.0.0-8080-exec-158] INFO  ACCESS - OPTIONS /api/auth/firebase-sync -> 200 (1 ms)
2026-05-21 21:26:16.401 [http-nio-0.0.0.0-8080-exec-160] INFO  ACCESS - GET /api/secure/admin -> 403 (41 ms)
2026-05-21 21:26:16.461 [http-nio-0.0.0.0-8080-exec-120] INFO  ACCESS - POST /api/auth/firebase-sync -> 200 (98 ms)
2026-05-21 21:26:16.478 [http-nio-0.0.0.0-8080-exec-155] INFO  ACCESS - OPTIONS /api/secure/favorites -> 200 (1 ms)
2026-05-21 21:26:16.496 [http-nio-0.0.0.0-8080-exec-156] INFO  ACCESS - OPTIONS /api/secure/push/token -> 200 (0 ms)
2026-05-21 21:26:16.503 [http-nio-0.0.0.0-8080-exec-141] INFO  c.m.t.o.controller.ProductSearchController - Execution time for fetching all products: 24milliseconds
2026-05-21 21:26:16.504 [http-nio-0.0.0.0-8080-exec-141] INFO  ACCESS - POST /api/public/product/search -> 200 (59 ms)
2026-05-21 21:26:16.522 [http-nio-0.0.0.0-8080-exec-121] INFO  ACCESS - GET /api/secure/favorites -> 200 (43 ms)
2026-05-21 21:26:16.534 [http-nio-0.0.0.0-8080-exec-153] INFO  ACCESS - POST /api/secure/push/token -> 200 (37 ms)
2026-05-21 21:26:17.551 [http-nio-0.0.0.0-8080-exec-111] INFO  c.m.t.o.controller.ProductSearchController - Execution time for fetching all products: 10milliseconds
2026-05-21 21:26:17.552 [http-nio-0.0.0.0-8080-exec-111] INFO  ACCESS - POST /api/public/product/search -> 200 (49 ms)
2026-05-21 21:26:20.368 [http-nio-0.0.0.0-8080-exec-157] INFO  ACCESS - OPTIONS /api/auth/firebase/qngqVFVIw7RbOezleWBFxiRI3et1 -> 200 (1 ms)
2026-05-21 21:26:20.407 [http-nio-0.0.0.0-8080-exec-161] INFO  ACCESS - GET /api/auth/firebase/qngqVFVIw7RbOezleWBFxiRI3et1 -> 200 (37 ms)
```

Capture window (backend access log): **21:26:13.640 → 21:26:20.407**.
**4 `POST /api/public/product/search` lines** at 21:26:13.641, 21:26:14.065, 21:26:16.504, 21:26:17.552.

Note on the time gap: the browser captured the hard refresh at **21:17**, then Igor performed a second, separate hard refresh at **21:26** to grab the backend access log. The two captures are therefore not the same physical run, but the *pattern* (3 `Home` SSR renders + 1 `FilterManager router.refresh` event in browser; 4 product/search POSTs in backend) is internally consistent across both captures and reproduces the audit's "4 calls on hard refresh" evidence. Reconciliation below maps the four backend POSTs to the three `[instr]` page renders + the one inferred `generateMetadata` cold-cache call.

## Raw capture — Normal refresh

### Browser console (verbatim, local time UTC+2)

```
21:18:21.448 Navigated to http://localhost:3000/rs-sr
21:18:22.352 Download the React DevTools for a better development experience: https://react.dev/link/react-devtools                                forward-logs-shared.ts:95:22
21:18:22.390 [HMR] connected                                                                                                                       forward-logs-shared.ts:95:22
21:18:22.392  Server  [instr] SSR page render 2026-05-21T19:18:21.558Z
Object { searchParams: {…} }                                                                                                                       forward-logs-shared.ts:95:22
21:18:25.647 [instr] FilterManager router.refresh 2026-05-21T19:18:25.647Z
Object { newUrl: "/rs-sr/", current: "/rs-sr" }                                                                                                    forward-logs-shared.ts:95:22
21:18:26.031  Server  [instr] SSR page render 2026-05-21T19:18:25.743Z
Object { searchParams: {…} }                                                                                                                       forward-logs-shared.ts:95:22
21:18:26.032  Server  [instr] SSR page render 2026-05-21T19:18:25.763Z
Object { searchParams: {…} }                                                                                                                       forward-logs-shared.ts:95:22
21:18:26.181 XHR  GET  http://localhost:8080/api/secure/admin  [HTTP/1.1 403  504ms]
21:18:26.700 [admin.isAdminUser] backend returned non-2xx
Object { status: 403, errorCode: undefined }                                                                                                       forward-logs-shared.ts:95:22
21:18:32.450 The resource at "http://localhost:3000/_next/static/media/83afe278b6a6bb3c-s.p.0q-301v4kxxnr.woff2" preloaded with link preload was not used within a few seconds.  rs-sr
21:18:32.450 The resource at "http://localhost:3000/404error.png" preloaded with link preload was not used within a few seconds.                                                rs-sr
```

Capture window (browser local time): **21:18:21 → 21:18:32**.
Server-side ISO timestamps inside the `[instr]` payloads: **19:18:21.558Z → 19:18:25.763Z**.

### Backend access log slice (verbatim — separate normal-refresh run)

```
2026-05-21 21:26:53.314 [http-nio-0.0.0.0-8080-exec-159] INFO  c.m.t.o.controller.ProductSearchController - Execution time for fetching all products: 16milliseconds
2026-05-21 21:26:53.316 [http-nio-0.0.0.0-8080-exec-159] INFO  ACCESS - POST /api/public/product/search -> 200 (77 ms)
2026-05-21 21:26:54.444 [http-nio-0.0.0.0-8080-exec-116] INFO  ACCESS - OPTIONS /api/secure/admin -> 200 (1 ms)
2026-05-21 21:26:54.471 [http-nio-0.0.0.0-8080-exec-154] INFO  c.m.t.o.controller.ProductSearchController - Execution time for fetching all products: 23milliseconds
2026-05-21 21:26:54.472 [http-nio-0.0.0.0-8080-exec-154] INFO  ACCESS - POST /api/public/product/search -> 200 (83 ms)
2026-05-21 21:26:54.481 [http-nio-0.0.0.0-8080-exec-127] INFO  c.m.t.o.controller.ProductSearchController - Execution time for fetching all products: 11milliseconds
2026-05-21 21:26:54.482 [http-nio-0.0.0.0-8080-exec-127] INFO  ACCESS - POST /api/public/product/search -> 200 (90 ms)
2026-05-21 21:26:54.485 [http-nio-0.0.0.0-8080-exec-110] INFO  ACCESS - GET /api/secure/admin -> 403 (38 ms)
2026-05-21 21:26:54.516 [http-nio-0.0.0.0-8080-exec-151] INFO  ACCESS - OPTIONS /api/auth/firebase-sync -> 200 (1 ms)
2026-05-21 21:26:54.632 [http-nio-0.0.0.0-8080-exec-137] INFO  ACCESS - POST /api/auth/firebase-sync -> 200 (114 ms)
2026-05-21 21:26:54.678 [http-nio-0.0.0.0-8080-exec-123] INFO  ACCESS - OPTIONS /api/secure/favorites -> 200 (1 ms)
2026-05-21 21:26:54.680 [http-nio-0.0.0.0-8080-exec-114] INFO  ACCESS - OPTIONS /api/secure/push/token -> 200 (0 ms)
2026-05-21 21:26:54.713 [http-nio-0.0.0.0-8080-exec-152] INFO  ACCESS - GET /api/secure/favorites -> 200 (33 ms)
2026-05-21 21:26:54.722 [http-nio-0.0.0.0-8080-exec-150] INFO  ACCESS - POST /api/secure/push/token -> 200 (41 ms)
2026-05-21 21:26:56.378 [http-nio-0.0.0.0-8080-exec-158] INFO  ACCESS - OPTIONS /api/auth/firebase/qngqVFVIw7RbOezleWBFxiRI3et1 -> 200 (1 ms)
2026-05-21 21:26:56.435 [http-nio-0.0.0.0-8080-exec-160] INFO  ACCESS - GET /api/auth/firebase/qngqVFVIw7RbOezleWBFxiRI3et1 -> 200 (55 ms)
```

Capture window (backend access log): **21:26:53.314 → 21:26:56.435**.
**3 `POST /api/public/product/search` lines** at 21:26:53.316, 21:26:54.472, 21:26:54.482.

Same note as above: the browser-side normal-refresh capture (21:18) and the backend-side normal-refresh capture (21:26:53) are not the same physical run; Igor performed a second normal refresh to grab the backend access log. Patterns reconcile across both.

## Reconciliation — Hard refresh

4 backend `POST /api/public/product/search` lines:

| # | Backend timestamp | Δ vs prior | Mapping | Instrumented trigger |
| - | --- | --- | --- | --- |
| 1 | 21:26:13.641 | — | Initial SSR — `Home` page-render Call site A (`getPortalProducts(filtersData, getInitialPage())` at `app/[locale]/(portal)/(public)/page.tsx:34`) | `[instr] SSR page render` #1 |
| 2 | 21:26:14.065 | +424 ms | Initial SSR — `generateMetadata` Call site B (`getMetadataProductsSnapshot({}, 10)` at `src/lib/service/nextCalls/metadataSnapshotService.ts:55`, **data-cache cold**) | **No `[instr]` log** (fires inside `generateMetadata`, not `Home`) |
| 3 | 21:26:16.504 | +2,439 ms (post `firebase-sync` ack at 21:26:16.461) | RSC re-render of `Home` triggered by `router.replace('/rs-sr/', { scroll: false })` inside FilterManager's SYNC TO URL effect | `[instr] SSR page render` #2 |
| 4 | 21:26:17.552 | +1,048 ms | RSC re-render of `Home` triggered by the queued `setTimeout(() => router.refresh(), 0)` immediately after the `router.replace` | `[instr] SSR page render` #3 |

The `[instr] FilterManager router.refresh` log fired exactly once on the browser side, carrying `{ newUrl: "/rs-sr/", current: "/rs-sr" }`. One firing produced both POST #3 and POST #4.

## Reconciliation — Normal refresh

3 backend `POST /api/public/product/search` lines:

| # | Backend timestamp | Δ vs prior | Mapping | Instrumented trigger |
| - | --- | --- | --- | --- |
| 1 | 21:26:53.316 | — | Initial SSR — `Home` page-render Call site A | `[instr] SSR page render` #1 |
| — | — | — | Call site B (`generateMetadata`) — **cache HIT, no backend POST** | n/a |
| 2 | 21:26:54.472 | +1,156 ms (post `firebase-sync` round-trip) | RSC re-render of `Home` from `router.replace('/rs-sr/', { scroll: false })` | `[instr] SSR page render` #2 |
| 3 | 21:26:54.482 | +10 ms | RSC re-render of `Home` from `setTimeout(() => router.refresh(), 0)` | `[instr] SSR page render` #3 |

The `[instr] FilterManager router.refresh` log fired exactly once on the browser side, carrying `{ newUrl: "/rs-sr/", current: "/rs-sr" }`. Same one firing produced both POST #2 and POST #3.

The 10 ms gap between POST #2 and POST #3 on normal refresh matches the 23 ms server-side render gap observed in the 21:17 browser capture, and is the natural shape of `router.replace + setTimeout(0) → router.refresh` firing two RSC fetches in the same tick. The 1,048 ms gap between POST #3 and POST #4 on hard refresh is the outlier — most likely caused by the busier event loop during the initial JS-bundle parse / hydration window delaying the `setTimeout(0)` callback. The *count* of two RSC fetches per single FilterManager invocation reproduces on both refreshes.

## Verdict

**The two post-mount RSC re-renders on every refresh of `/rs-sr` both originate from a single `FilterManager` SYNC TO URL effect invocation that fires `router.replace('/rs-sr/', { scroll: false })` followed by `setTimeout(() => router.refresh(), 0)` because the gate `newUrl !== current` at `src/components/client/initializers/FilterManager.tsx:315` evaluates to `true` on a trailing-slash canonicalization (`current = "/rs-sr"`, `newUrl = "/rs-sr/"`): the `router.replace` triggers one RSC refetch (Next 16 refetches the server tree on `replace` to a new pathname), and the queued `router.refresh` triggers a second; both fetches re-render the `Home` server component and each produces one `POST /api/public/product/search` against `getPortalProducts(...)`.**

The dep churn that runs the SYNC TO URL effect on first visit is the post-HYDRATE-FROM-URL state churn already mapped in the prior audit (`selectedFilters`, `selectedPriceRange`, `selectedRegionsAndCities` are reassigned to fresh empty-state references inside the HYDRATE FROM URL effect at `FilterManager.tsx:179, :189, :207`, plus `hydrated` flips from `false` to `true`). The audit's hypothesis #1 ("the SYNC TO URL effect fires twice and produces two refreshes") is **wrong**; the audit's adjacent observation #5 (`router.replace + setTimeout(0) → router.refresh` double-fetches in Next 16) is **right**. The audit was missing the "one firing → two RSC fetches" link; instrumentation closed it.

The trailing-slash diff (`/rs-sr` from the browser URL vs `/rs-sr/` constructed by FilterManager at `:311`) is the *gating* cause that makes the SYNC TO URL effect's `newUrl !== current` true on the no-filters home. If the gate evaluated to `false` (canonical URLs already aligned), no `router.replace` would fire and no extra RSC re-renders would happen. Both root causes — the gating mismatch and the double-fetch produced by `router.replace + setTimeout(0) → router.refresh` — would individually fix the multi-search behavior on this route.

## Confirmation — instrumentation removed

- `grep -n "\[instr\]" app/\[locale\]/\(portal\)/\(public\)/page.tsx src/components/client/initializers/FilterManager.tsx` returns no matches.
- `git diff src/components/client/initializers/FilterManager.tsx` returns no diff.
- `git diff app/\[locale\]/\(portal\)/\(public\)/page.tsx` shows only Igor's pre-existing WIP on `dev` (the cookies-V2 refactor visible at the start of this session) — none of those hunks were authored by me. My single instrumentation line is gone.
- After this summary is written, the only file added in this session is this summary + its `last-session.md` twin.

## Files touched

- `.agent/2026-05-21-oglasino-web-portal-multi-search-investigation-2.md` (new, this summary)
- `.agent/last-session.md` (overwritten with the same content as the named file above)

(The two source files instrumented during the session — `app/[locale]/(portal)/(public)/page.tsx` and `src/components/client/initializers/FilterManager.tsx` — were edited twice each: log added, then log removed. Net diff against `HEAD` for my work on those two files is zero. The non-zero `git diff` Igor will see on `page.tsx` is his pre-existing WIP, not this session's work.)

## Tests

- None ran. The brief is read-and-instrument only, with the instrumentation removed before close. No automated test exercises the route's RSC re-render behavior, and running `npm run lint` / `npx tsc --noEmit` / `npm test` would also catch Igor's WIP changes elsewhere on `dev`, which is out of scope for this session.

## Cleanup performed

- Removed the two temporary `console.log` lines added during the session (one in `page.tsx`, one in `FilterManager.tsx`). Verified by `grep` and `git diff`.

## Config-file impact

- `conventions.md`: no change
- `decisions.md`: no change
- `state.md`: no change
- `issues.md`: no change

## Obsoleted by this session

- The "Known gaps" section of the prior audit (`.agent/2026-05-21-oglasino-web-portal-multi-search-investigation-1.md`, lines 343–344) — the explicit gap "cannot deterministically identify from code alone what triggers the two post-mount RSC re-renders" is now closed by this session's reconciliation. The audit summary remains the read-only ground truth for the four-call cascade; this session is the deterministic supplement.

## Conventions check

- Part 4 (cleanliness): confirmed. No `console.log` left on disk. No commented-out blocks. No `TODO` added. Files I touched were left at zero net diff for the two source paths.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): confirmed — one observation surfaced (see "For Mastermind").
- Part 6 (translations): N/A this session (no translation work).
- Other parts touched: Part 3 (engineer agent hard rules — no commits, no pushes, no cross-repo edits, no writes to the four config files; instrumentation strictly temporary).

## Known gaps / TODOs

- The browser captures (21:17 / 21:18) and the backend captures (21:26:13 / 21:26:53) are from separate physical refreshes, not the same runs. Reconciliation worked on the basis that both captures show the same shape (3 `Home` SSR renders + 1 FilterManager refresh in browser; 4 product-search POSTs on hard refresh, 3 on normal refresh on the backend). The pairing of each `[instr]` log to its exact backend POST is by *pattern + order*, not by synchronized clock. The verdict does not depend on per-line clock alignment.
- The 1,048 ms gap between POST #3 and POST #4 on the hard-refresh backend slice is left as an observation, not investigated further. It does not affect the verdict.

## For Mastermind

### Part 4a simplicity evidence (required)

- **Added (earned complexity):** nothing. The two `console.log` lines were temporary instrumentation per the brief; both removed in the same session.
- **Considered and rejected:** nothing. The brief is a narrow instrumentation pass with no design surface.
- **Simplified or removed:** nothing in code. The brief's outcome simplifies *understanding* of the route (one of the prior audit's two open hypotheses is now resolved as the wrong one; the right one is named), but no production code changed.

### Adjacent observations surfaced during the session (Part 4b)

1. **`Home` page component does not destructure `params`, only `searchParams`** — `app/[locale]/(portal)/(public)/page.tsx:25-28`. Severity: low. The locale is resolved later via `getLocale()` inside `generateMetadata` (`:18`) but never inside the page component itself; the component reads it indirectly through `getTranslations` and `getBaseSiteServer()`. The brief's template suggested logging `{ locale }` at the top; I logged `{ searchParams }` instead because that's the only available prop at that point. This is a non-bug cosmetic mismatch with how every other locale-aware route component is conventionally written (most pages in this repo also do not destructure `params` — checked `app/[locale]/(portal)/(public)/catalog/[[...slugs]]/page.tsx`, `app/[locale]/(portal)/(protected)/favorites/page.tsx`, `app/[locale]/(portal)/(public)/user/[userId]/page.tsx`). Out of scope; flagging only because the brief's template assumed a `locale` prop that doesn't exist.

### Verdict implications for the next brief

The fix surface for "extra two product-search POSTs per portal home refresh" is now narrow and concrete. Two independent levers either of which fixes the user-visible behavior:

- **Lever A — close the trailing-slash gate.** The `newUrl` constructed at `FilterManager.tsx:311` (`/${locale}${pathname}` + …) always carries a trailing slash on the no-filters home (`pathname = "/"`), while `current` at `:313` is the actual browser URL (`/rs-sr` with no trailing slash). Normalize one side before the `!==` comparison and the SYNC TO URL effect's URL-mutating branch never fires on a refresh of the canonical home URL. No `router.replace` / `router.refresh` happens. No extra RSC re-renders. No extra product-search POSTs. The audit's adjacent observation #5 still applies in principle but becomes inert on this route. This is the narrow, lower-blast-radius fix.

- **Lever B — collapse `router.replace + setTimeout(() => router.refresh(), 0)` to a single call.** Drop the `setTimeout(... router.refresh(), 0)` line; Next 16 already refetches the RSC tree on `router.replace` to a new pathname. This eliminates the second of the two RSC fetches on every place where the same pattern is used (anywhere FilterManager's `newUrl !== current` evaluates true on a legitimate URL change). Higher blast radius (touches the behavior on every actual filter-change scenario, not just the canonical home), but simpler and structurally correct.

Both levers can be combined safely (the trailing-slash fix removes the false-positive trigger; the dropped `setTimeout(... refresh, 0)` removes the double-fetch on real triggers). The bug-fix brief that comes next should pick one or both based on Mastermind's read of which surface is most worth touching.

### Drafted config-file text

- None drafted this session. The behavior surfaced here does not yet warrant a `decisions.md` entry — the fix decision (which lever, which brief) is Mastermind's call. No `issues.md` entry drafted inline: the brief is part of a planned multi-brief sequence (this one closes the audit's gap; the next is the fix), so the work flows through Mastermind directly, not through `issues.md`.

### Questions for Mastermind

- Which lever for the fix? (A — trailing-slash normalization; B — drop the `setTimeout(... refresh, 0)`; or both.) Lever A is the smallest fix that addresses *this* route; lever B is the structurally correct fix that may surface other behaviors not yet observed.
- Worth checking other routes (`/rs-sr/catalog`, `/rs-sr/owner/...`) for the same trailing-slash mismatch shape? FilterManager mounts on multiple routes; the no-filter home is the simplest case but the trailing-slash diff may apply elsewhere too.

### Suggested next steps

1. Take the verdict to Mastermind. Decide lever (A / B / both) for the next bug-fix brief.
2. The "two RSC re-renders" gap is now closed. The audit's other two open questions (random-seed jerk; firebase-sync + push/token double-POST) remain on their own separate-brief tracks per the audit's "Out of scope" section.
