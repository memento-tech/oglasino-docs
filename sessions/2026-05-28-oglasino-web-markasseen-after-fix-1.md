# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-28
**Task:** Fix brief — view-count increment never fires (Fix B: `after()`) + local log-on-failure

## Implemented

- Wrapped the gated `markAsSeen(productDetails.id)` call in `after(() => ...)` from `next/server` in the product RSC, so the increment is scheduled to run after the response is sent rather than as an orphan Promise during render. Owner-exclusion gate (`!owner.iamActive`) preserved exactly as-is; `after()` is invoked only inside the gated branch.
- Imported `after` from `next/server` alongside the existing `notFound, redirect` import from `next/navigation`.
- Added a local non-success inspection in `markAsSeen`: after awaiting `FETCH_BACKEND_API.get(...)`, the resolved envelope's `success` boolean is checked, and on `false` `logServiceWarn('product.next.markAsSeen', response)` records the status + errorCode. Keeps the existing `try/catch` for genuine throws; does not re-throw — fire-and-forget posture preserved.
- Auth posture and shared `FETCH_BACKEND_API.request()` swallow behaviour deliberately untouched per brief scope and prior audits.

## Files touched

- `app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx` (+2 / -1)
- `src/lib/service/nextCalls/productService.ts` (+5 / -2)

## Tests

- Ran: `npx eslint <touched paths>`, `npx tsc --noEmit`, `npm test`
- Result: lint clean (no output), tsc clean (no output), vitest `247 passed (247)` across 22 test files
- New tests added: none — no DOM test environment exists in this repo (see [issues.md](../../oglasino-docs/issues.md) 2026-05-27 entry "No DOM test environment in `oglasino-web`") and `markAsSeen` has no existing service-layer test file. The change is small and validated by reasoning + stage smoke per the brief's Definition of Done.

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: 1 new entry drafted in "For Mastermind" below (the consolidated view-count bug entry the brief asks for). Not written here — Docs/QA applies per Part 3.

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no commented-out code, no unused imports, no `console.log`, no `TODO`/`FIXME` added.
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): re-flagged in "For Mastermind" (same shapes the audits already flagged; nothing new this session)
- Part 6 (translations): N/A this session — no translation keys touched.
- Part 7 (error contract): N/A this session — the `/public/product/seen/<id>` endpoint is not a validation surface.
- Part 11 (trust boundaries): trust posture unchanged. The view increment is not a moderation / authorization / state-transition decision. Owner-exclusion remains server-derivable (auth forwarded on the call) plus the web-side `!owner.iamActive` gate. No client-supplied identity is trusted.

## Known gaps / TODOs

- none

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): `after(() => markAsSeen(...))` wrapper — earns its place because it is the Next 15+ supported mechanism for reliably running side effects after an RSC response, replacing an unreliable orphan Promise. Single concrete problem (silent 0% delivery), no introduced abstraction.
  - Considered and rejected:
    - **`await markAsSeen(...)` (Fix A)** — adds RTT to every non-owner product-page render; rejected per brief.
    - **Client-side `ProductSeenTracker` (Fix C)** — largest change; rejected per brief (also shifts counted population — bots/no-JS out).
    - **Touching `FETCH_BACKEND_API.request()` swallow behaviour** — broader concern out of scope per brief; deliberately not changed.
    - **Adding `skipAuth: true` to `markAsSeen`** — would break server-side owner-exclusion identification; brief explicitly says do not add. Not added.
    - **Adding a test for `markAsSeen`** — no existing test file for it; repo lacks a DOM testing environment; adding service-layer-only test scaffolding for one function would be premature. Rejected.
  - Simplified or removed: nothing.

- **Drafted `issues.md` entry (consolidated view-count bug — please pass to Docs/QA):**

  ```markdown
  ## 2026-05-28 — Product view count always 0 across the portal (increment never fired + display guard hid the zero)

  **Severity:** medium
  **Status:** fixed
  **Found in:** `oglasino-web/app/[locale]/(portal)/(public)/product/[productId]/[productName]/page.tsx`, `oglasino-web/src/lib/service/nextCalls/productService.ts`, `oglasino-web/src/components/client/NumberOfViews.tsx`.
  **Detail:** Visible view count was `0` on every product page across the portal. Two cooperating root causes:

  1. **Increment never reached the backend.** `markAsSeen(productDetails.id)` was invoked as an un-awaited async call inside the product page's RSC render (`page.tsx:95`). The Promise's first hop is `await cookies()` inside `FETCH_BACKEND_API.request` (`src/lib/config/fetchApi.ts:39`); Next 15 / React 19 do not guarantee that fire-and-forget side effects started during render flush before the request lifecycle ends. The fetch was effectively never dispatched. Read-only audit: [`.agent/2026-05-28-oglasino-web-markasseen-never-fires-audit-1.md`](sessions/2026-05-28-oglasino-web-markasseen-never-fires-audit-1.md) (or the in-repo `.agent/` original until archived).
  2. **Display guard hid any real zero.** `NumberOfViews` rendered nothing when `viewsCount <= 0`, masking new products and any backend / fetch failure path. Surfaced in [`.agent/2026-05-28-oglasino-web-views-not-displaying-audit-1.md`](sessions/2026-05-28-oglasino-web-views-not-displaying-audit-1.md).
  3. **All failures were silent.** `FETCH_BACKEND_API.request()` swallows errors into a resolved `{ status: 0, errorCode: 'connection.timeout' }` envelope, so `markAsSeen`'s `try/catch` never fired and `logServiceError` was never called. A 0% delivery rate produced 0 log lines — nothing in the operational surface to falsify the assumption that the call was firing.

  Backend was confirmed clean by a parallel read-only audit (`oglasino-backend/.agent/2026-05-28-oglasino-backend-view-counter-check-1`): the counter chain works correctly when `/public/product/seen/<id>` is actually called.

  > **Fix:**
  > - **Increment reliability:** `markAsSeen(productDetails.id)` is now wrapped in `after(() => ...)` from `next/server` (session `oglasino-web-markasseen-after-fix-1`, 2026-05-28). The owner-exclusion gate `!owner.iamActive` is preserved exactly as-is; the call is only scheduled when the gate is satisfied. `after()` is the Next 15+ supported mechanism for reliably running side effects after the response is sent.
  > - **Local log-on-failure:** `markAsSeen` now inspects the resolved envelope's `success` boolean and calls `logServiceWarn('product.next.markAsSeen', response)` on a non-success result. The existing `try/catch` is kept for genuine throws; nothing is re-thrown. A future silent failure will leave a trace. Local to `markAsSeen`; the shared `FETCH_BACKEND_API.request()` swallow is deliberately untouched (separate, wider concern).
  > - **Display guard relaxation:** the `< 0` guard plus `null`-on-failure path in `NumberOfViews` was already landed in the prior session (see the views-display audit and the session that adopted its recommendation). New products and real zeros now render visibly; only network/backend failure paths render nothing.
  > - **Auth posture:** unchanged. `markAsSeen` still forwards cookies + Firebase token (no `skipAuth: true`), so the backend can identify the viewer for owner-exclusion. The web-side `!owner.iamActive` gate remains in place as the first line of defence.
  > - **Shared `FETCH_BACKEND_API.request()` error-swallow:** untouched. Flagged across three audits (this one, `markasseen-never-fires-audit`, `skipauth-footprint-audit`). Separate concern.
  ```

- **Re-flagged adjacent observations (not new — re-stated only because they were called out in the audit pair this brief closes):**
  - **`FETCH_BACKEND_API.request()` swallows all errors into a resolved `{ status: 0, errorCode: 'connection.timeout' }` envelope (`src/lib/config/fetchApi.ts:72-79`) — severity: medium.** This silent-failure shape is what allowed the present bug to persist undetected. The local log-on-failure added in this session makes `markAsSeen` specifically observable, but every other `/public/*` server-side call in the repo is still subject to the same silence. Fix paths: either (a) thread an `opt-in-throw-on-failure` flag through `request()` for callers that want to react to failures, or (b) repeat the local-inspect pattern at every important caller. (a) scales better. Out of scope here; flagged across three audits now.
  - **`markAsSeen` does not pass `skipAuth: true` despite hitting a `/public/*` endpoint — severity: low.** Intentionally so — auth is forwarded so the backend can identify the viewer for owner-exclusion. The web-side gate doubles up. The brief explicitly says do not add `skipAuth`. Re-flagging only because the audit pair noted it; no action requested.

- (nothing else flagged)
