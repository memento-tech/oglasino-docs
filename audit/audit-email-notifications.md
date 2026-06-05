# Audit — email-notifications (oglasino-router)

**Repo:** oglasino-router
**Branch:** stage (unchanged — read-only audit, no edits)
**Date:** 2026-06-02
**Ground truth:** `src/index.ts` (read in full), cross-checked against `tests/router.test.ts` and `wrangler.toml`.
**Question:** Does `oglasino.com/[locale]/verify?oobCode=...` (a normal page on our own apex domain, NOT a Firebase-hosted action page) route cleanly to the web origin through the worker, with no special handling?

**Answer up front:** Yes. A `/[locale]/verify?oobCode=...` request on the apex host is ordinary frontend traffic. It is not caught by the `/api/*` backend branch, the admin-locale regex, the mobile branch, or any other special case, and it forwards to `FRONTEND_ORIGIN` (Vercel) with the `oobCode` query string passed through verbatim. The only caveat is maintenance (Section 2) and the host the link must use (Section 3).

---

## Section 1 — Routing for a new `/verify` page

Traced for `GET https://oglasino.com/rs-sr/verify?oobCode=ABC123` (production apex), step by step through `fetch` (`src/index.ts:65`–`196`):

| Step | Code | Result for `/rs-sr/verify?oobCode=ABC123` |
| --- | --- | --- |
| `host` | `src/index.ts:72` | `oglasino.com` |
| `path` | `src/index.ts:73` | `/rs-sr/verify` |
| www→apex 301 | `src/index.ts:76` | **skipped** — host is apex, not `WWW_HOST` |
| `isApi` | `src/index.ts:83` | `false` — host ≠ `API_HOST` (`api.oglasino.com`) |
| `isFrontend` | `src/index.ts:84` | `true` — host = `APEX_HOST` |
| 404 guard | `src/index.ts:86` | **skipped** — `isFrontend` is true |
| `isMobile` | `src/index.ts:92` | `false` — path does not start with `/api/mobile/` |
| `isAdminRequest` | `src/index.ts:96`–`105` | **`false`** — see breakdown below |
| mobile branch | `src/index.ts:139` | **skipped** — not mobile |
| `webDown` gate | `src/index.ts:172`–`178` | passes through when no maintenance flag is set (see Section 2) |
| `isApi` forward | `src/index.ts:180` | **skipped** — not API |
| frontend forward | `src/index.ts:189` | **`forwardToOrigin(FRONTEND_ORIGIN, "/rs-sr/verify", "?oobCode=ABC123", request, isStage)`** |

**`isAdminRequest` breakdown** (`src/index.ts:96`–`105`) — every clause is false for `/rs-sr/verify`:

- Admin-locale regex `/^\/[a-z]{2}-[a-z]{2}\/admin(\/|$)/i` (`src/index.ts:98`): requires the literal segment `/admin` immediately after the locale. `/rs-sr/verify` has `verify`, not `admin` → **no match**. (The regex would only matter if the verify page lived under `/[locale]/admin/...`, which it does not.)
- `isApi` → false; `path.startsWith("/api/")` → false; `path.startsWith("/_next/")` → false; `path === "/favicon.ico"` → false.

So `/verify` is classified as plain non-admin frontend traffic and, absent maintenance, lands at the frontend forward (`src/index.ts:189`).

### `oobCode` query param survives forwarding — confirmed

`forwardToOrigin` (`src/index.ts:236`–`273`) builds the target URL at `src/index.ts:243`:

```ts
const targetUrl = `${originBase}${path}${search}`;
```

`search` is `url.search` (`src/index.ts:189`), i.e. the full query string `?oobCode=ABC123` exactly as received. The worker never parses, re-encodes, strips, or rewrites the query string anywhere on the frontend path — it concatenates it verbatim. So `oobCode` (and any additional params such as Firebase's `mode`, `apiKey`, `continueUrl`, `lang`) reach Vercel unchanged.

This is **test-backed**: `tests/router.test.ts:820` ("query string preserved when forwarding") asserts `https://oglasino.com/search?q=hello&page=2` forwards to `https://oglasino-web-prod.vercel.app/search?q=hello&page=2`, and `tests/router.test.ts:114` asserts `/page?x=1` forwards with the query intact. `oobCode` is just another query param and follows the same path.

**One precision note on the brief's framing.** The brief attributes query-string passthrough to `redirect: "manual"`. These are two independent mechanisms:

- Query-string passthrough is governed by the `${path}${search}` URL construction at `src/index.ts:243` — the query survives because the worker forwards `url.search` literally.
- `redirect: "manual"` (`src/index.ts:256`) governs the *response* direction: it makes upstream 3xx responses from Vercel pass back to the client unchanged instead of the worker following them. It does not touch the request query string.

Both are favorable for `/verify`: the `oobCode` reaches Vercel verbatim, and if the verify page itself responds with a redirect (e.g. to a "verified" success page), that redirect passes through to the browser unchanged. No special handling is needed for either.

---

## Section 2 — Maintenance interaction

During maintenance, `/verify` is gated **exactly like any other non-admin frontend page** — this is expected behavior, not a bug, but the spec should acknowledge it.

Trace with `maintenance.web.active="true"` OR `maintenance.backend.active="true"` (`src/index.ts:172`–`178`):

- `webDown = maintenanceWebActive || maintenanceBackendActive` → `true`.
- `shouldBlock = adminBypassDisabled || !isAdminRequest`. For `/verify`, `isAdminRequest` is `false`, so `!isAdminRequest` is `true` → `shouldBlock` is `true` **regardless of the admin-bypass flag**.
- Result: `maintenanceResponse(isApi=false, "/rs-sr/verify", "?oobCode=ABC123", env, isStage)` (`src/index.ts:176`).

`maintenanceResponse` on the non-API branch (`src/index.ts:218`–`233`) fetches `MAINTENANCE_ORIGIN` with the same path+search and returns **HTTP 503** with `X-Oglasino-Maintenance: true`, `Cache-Control: no-store`, `Retry-After: 120` (and `X-Robots-Tag: noindex…` on stage). The user clicking a verification link mid-maintenance therefore sees the maintenance page, **not** the verify page; the `oobCode` is never delivered to Vercel on that request.

**Expected, per the maintenance matrix.** `/verify` is non-admin frontend traffic, and the matrix blocks all non-admin web traffic when web is down. There is no router-side carve-out for `/verify`, and per the brief this audit does not propose one.

Downstream UX implication worth recording in the feature spec (out of router scope, flagged for the seam analysis): Firebase action codes (`oobCode`) are time-limited (typically up to a few days, depending on action type). A click during maintenance returns 503; the user must retry after maintenance ends, and if the code expires in the interim the verification fails and a fresh email is needed. This is a product/spec consideration, not a worker change — the worker behaves correctly by treating `/verify` like every other page.

---

## Section 3 — Seams

**What the worker assumes about the web origin serving `/verify`:** that `FRONTEND_ORIGIN` (Vercel) serves `/[locale]/verify` as an ordinary page. The worker does nothing special — it forwards `method`, headers (adding `X-Forwarded-Host` / `X-Forwarded-Proto`), body, path, and the full query string, with `redirect: "manual"` so any verify-page redirect passes through. All `oobCode` handling lives on Vercel/the web app; the worker is a transparent pass-through for this route.

**What the email-link shape must avoid / satisfy:**

1. **Use the apex host.** Production: `https://oglasino.com/...`; stage: `https://stage.oglasino.com/...` (`APEX_HOST` per `wrangler.toml:20`/`61`).
   - `www.oglasino.com` would 301-redirect to apex preserving path+query (`src/index.ts:76`) — harmless, but an avoidable extra hop.
   - **`api.oglasino.com` must NOT be used.** On the API host `isFrontend` is false and `isApi` is true, so the request would forward to `BACKEND_ORIGIN` (the droplet), not Vercel — the verify page would not be served. This is the one host that breaks the route.
2. **Avoid the worker's special-cased path prefixes.** A path of the shape `/[locale]/verify` is clear of all of them, but for completeness the reserved prefixes are: `/api/` and `/api/mobile/` (backend / mobile branches), `/_next/` and `/favicon.ico` (treated as admin-infra during maintenance), and `/[a-z]{2}-[a-z]{2}/admin(/|$)` (admin-locale regex). Keep `verify` as its own path segment under the locale — do not nest it under `/admin` and do not prefix it with `/api`.
3. **Locale segment format is unconstrained for this route.** The `[a-z]{2}-[a-z]{2}` shape only matters for the admin regex; a normal `/[locale]/verify` forwards to the frontend regardless of how the locale segment is shaped. (Today's locales are `rs-sr`/`en-us`-style two-part segments, but even a non-matching segment would still forward to Vercel — it just wouldn't be an admin request.)

No new bindings, routes, or worker changes are required to support the branded verification link. The route works today as ordinary frontend traffic.

---

## Closeout

**Code changes:** none — read-only audit.
**Files touched:** this file only (`.agent/audit-email-notifications.md`), per the brief's "no new files except the audit output." I did not write the usual `.agent/yyyy-mm-dd-...-<slug>-<n>.md` + `last-session.md` session-summary pair this session because the brief explicitly constrained output to the single audit file (consistent with conventions Part 10, Phase 2, which names `.agent/audit-<slug>.md` as the audit deliverable). Flagging the deviation here so it is visible.
**Lint / tests:** not run — no code changed. Existing routing behavior is covered by `tests/router.test.ts` (frontend forward `:114`, admin-path forward `:128`, query-string preservation `:820`), which I read to corroborate the trace but did not modify.

### Conventions check

- Part 4 (cleanliness): N/A — no code touched.
- Part 4a (simplicity) / 4b (adjacent observations): see "For Mastermind" below.
- Critical care areas (maintenance matrix, fail-open KV, admin regex, stage noindex, `redirect: "manual"`): observed only, none altered.

### For Mastermind

**Brief vs reality** (one minor framing clarification, not a contradiction):

1. **Query passthrough mechanism**
   - Brief says: "the worker uses `redirect: 'manual'` forwarding — confirm query strings pass through."
   - Code says / I observed: query passthrough comes from the `${originBase}${path}${search}` construction at `src/index.ts:243`, which forwards `url.search` verbatim. `redirect: "manual"` (`src/index.ts:256`) is a separate mechanism that governs *response* 3xx passthrough, not the request query string.
   - Why this matters: both behaviors are favorable for `/verify`, but they are independent. The `oobCode` survives because of the URL construction; `redirect: "manual"` additionally ensures a verify-page redirect reaches the browser unchanged. Conflating them could mislead a future reader into thinking removing `redirect: "manual"` would affect query passthrough (it would not).
   - Recommended resolution: in the feature spec, state both: (a) query string forwarded verbatim, and (b) upstream redirects pass through unchanged.

**Adjacent observation (Part 4b)** — for the spec/seam analysis, not the router:

- One-line: a verification click during maintenance returns 503; if the `oobCode` expires before maintenance ends, the link fails and a fresh email is required.
- File path: behavior originates at `src/index.ts:172`–`178` (correct, by the matrix) — the consideration is product-side, not in the worker.
- Severity guess: low (UX edge case during maintenance windows; the worker is behaving correctly).
- I did not act on this because it is out of scope and would not be a worker change.

**Part 4a simplicity evidence:**
- Added (earned complexity): nothing — no code.
- Considered and rejected: nothing — no code.
- Simplified or removed: nothing — no code.

**Config-file impact:** none required. This audit surfaces no change to `conventions.md`, `decisions.md`, `state.md`, or `issues.md`. (The Section 2 expiry note and the Section 1 framing clarification are inputs for the Mastermind seam analysis / feature spec, not config-file edits — and per Part 3, I would draft rather than write them regardless.)
