# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-24
**Task:** Read-only audit of `UniversalProductCard` SSR-emptiness — answering 5 questions about what the card needs from `useBaseSiteStore()`, whether SSR rendering is feasible, consumer blast radius, engineering cost, and intermediate options.

## Implemented

- Read-only audit. No code changes.
- Answered all five questions from the brief. Full detail in `.agent/audit-universal-product-card-ssr.md`.
- Key finding: the `useBaseSiteStore()` call in `UniversalProductCard.tsx` is **vestigial** — `baseSite` is destructured and used only as a null guard (line 39: `if (!baseSite) return <></>`), but is never referenced anywhere in the component's JSX or logic. The `productOverview.baseSiteOverview.labelKey` at line 87 reads from the prop, not the store.

## Files touched

(none — read-only audit)

## Tests

N/A — read-only audit.

## Cleanup performed

none needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change.

## Obsoleted by this session

nothing.

## Conventions check

- Part 4 (cleanliness): N/A — read-only session, no code changes.
- Part 4a (simplicity): N/A — no complexity added or removed.
- Part 4b (adjacent observations): N/A — no files touched; the observation below is the audit finding itself, not an adjacent finding.
- Part 6 (translations): N/A this session.
- Other parts touched: none.

## Known gaps / TODOs

- none.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing (read-only audit).

- **Five answers in concise form:**

  1. **Q1 — What does the card need from the store?** Nothing. `baseSite` is destructured at `UniversalProductCard.tsx:37` and used only as a null guard at line 39. It is never referenced in the JSX. Every piece of data the card renders comes from its `productOverview: ProductOverviewDTO` prop, `useTranslations` (works in SSR), and other props.

  2. **Q2 — Can baseSite be passed as a prop?** Moot. The card doesn't use `baseSite` for rendering. For completeness: `baseSite` is already fetched server-side via `getBaseSiteServer()` on every page that renders product cards (home, catalog, user page). Favorites is a `'use client'` page and would need restructuring, but it's a protected (noindex) route.

  3. **Q3 — Consumer blast radius?** Zero. Six direct consumers (`PortalProductCard`, `DashboardProductCard`, `AdminProductCard`, `PreviewProductCard`, via `ProductCarousel`, via `FavoriteProductList`). None passes `baseSite`, none relies on the empty-return SSR behavior, none needs a signature change. The fix is entirely internal to `UniversalProductCard`.

  4. **Q4 — Engineering cost?** 30 minutes to 1 hour. Delete 3 lines (import, destructure, guard). Verify `tsc`, `lint`, `test`, and manual `curl` of the dev server to confirm cards appear in HTML source. No consumer changes. No prop threading. No architectural decision.

  5. **Q5 — Smaller intermediate option?** No intermediate needed — the full fix *is* the smallest possible change. Any "degraded shell" or "placeholder anchor" approach would be more work than removing the vestigial guard. All child components (`ProductFavoriteButton`, `WithTooltip`, `ProductTopImage`) degrade gracefully in SSR.

- **Engineer's overall read:** This is a **15-minute fix, not a 2-day refactor.** The prior session's characterization of "significant architectural change" was based on the assumption that the store value was load-bearing. It isn't. The guard is vestigial — likely a survivor from an earlier version of the card that read `baseSite.code` or `baseSite.currency` for rendering, before `ProductOverviewDTO` was enriched to carry `baseSiteOverview.labelKey`, `currency`, etc. directly.

- **Anything the audit surfaced that wasn't in the question list:**
  - The `'use client'` directive on `UniversalProductCard` itself is still necessary post-fix because `WithTooltip`, `ProductTopImage`, and `ProductFavoriteButton` use hooks. The card will SSR as a client component (React renders it on the server, then hydrates on the client) — this is standard Next.js App Router behavior and produces the desired server-rendered HTML.
  - `ProductTopImage` uses `useEffect` + `useState` for lazy image loading (`src/components/client/product/ProductTopImage.tsx`). During SSR, the `<img>` still renders with the correct `src` and `alt`; the lazy-load effect enhances client-side only. No issue.
  - The favorites page (`app/[locale]/(portal)/(protected)/favorites/page.tsx`) is a `'use client'` page that fetches data in a `useEffect`. Product cards on this page were never SSR-visible regardless of the `baseSite` guard, and it's a protected (noindex) route, so it's not an SEO surface.

- **Drafted config-file text:** none needed.
