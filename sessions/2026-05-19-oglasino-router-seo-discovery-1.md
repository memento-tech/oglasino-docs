# Session summary

**Repo:** oglasino-router
**Branch:** stage
**Date:** 2026-05-19
**Task:** Audit the Cloudflare Worker for SEO-relevant behavior. Code is ground truth.

## Findings

Code references are to `src/index.ts` unless otherwise stated.

### 1. The stage `noindex` header

1. **Header emitted on stage for all forwarded responses**
   - `src/index.ts:33` (constant), `:117` (`isStage` gate), `:174` (`addNoIndex` param), `:195–202` (emit)
   - Status: `correct`
   - Severity: `n/a`
   - Note: `isStage = env.ENVIRONMENT === "stage"`; production `wrangler.toml` sets `ENVIRONMENT = "production"`, so no leak path. Header value is forced regardless of upstream, which is the intent.

2. **Header name and value match spec**
   - `src/index.ts:33`, `:197`
   - Status: `correct`
   - Severity: `n/a`
   - Note: `X-Robots-Tag: noindex, nofollow, noarchive, nosnippet` — matches `CLAUDE.md` care-area.

3. **Maintenance responses on stage do NOT emit `X-Robots-Tag`**
   - `src/index.ts:110–114` (maintenance branch returns before `forwardToOrigin`), `:138–167` (`maintenanceResponse` — no noindex header set)
   - Status: `partial`
   - Severity: `medium`
   - Note: When stage is in maintenance, the 503 (API JSON or HTML from `MAINTENANCE_ORIGIN`) is returned without `X-Robots-Tag`. The 503 status itself discourages indexing, but the stage-wide noindex contract is not uniformly enforced. `MAINTENANCE_ORIGIN` is a public Pages site (`oglasino-maintenance.pages.dev`).

### 2. Maintenance mode behavior for crawlers

4. **API maintenance returns 503 + `Retry-After: 120` + `no-store`**
   - `src/index.ts:144–153`
   - Status: `correct`
   - Severity: `n/a`
   - Note: SEO-correct shape — crawlers will retry without de-indexing.

5. **Frontend maintenance returns 503 + `Retry-After: 120` + `no-store`**
   - `src/index.ts:156–166`
   - Status: `correct`
   - Severity: `n/a`
   - Note: Upstream body from `MAINTENANCE_ORIGIN` is preserved, status forced to 503, `Retry-After` and `Cache-Control: no-store` set on every response. Good SEO behavior.

6. **Full lockdown (`maintenance.active=true`, `admin.bypass.disabled=true`) returns 503 to crawlers**
   - `src/index.ts:106–115`
   - Status: `correct`
   - Severity: `n/a`
   - Note: `shouldBlock = adminBypassDisabled || !isAdminRequest` — when bypass is disabled everyone (crawlers included) gets 503 + `Retry-After`. Crawlers will retry, not de-index, which is the correct behavior even in lockdown.

7. **`MAINTENANCE_ORIGIN` fetch follows redirects (no `redirect: "manual"`)**
   - `src/index.ts:156`
   - Status: `partial`
   - Severity: `low`
   - Note: The fetch to the maintenance Pages site uses default `redirect: "follow"`. Not a current bug — `oglasino-maintenance.pages.dev` returns 200s — but inconsistent with the rest of the file's pattern and a latent surprise if the maintenance origin ever starts redirecting.

### 3. Admin / owner prefix handling

8. **Locale-prefixed admin paths are recognized for bypass routing only, NOT for noindex emission**
   - `src/index.ts:62–71` (regex `/^\/[a-z]{2}-[a-z]{2}\/admin(\/|$)/i`)
   - Status: `partial`
   - Severity: `high`
   - Note: The regex is used solely to classify admin traffic for maintenance-bypass purposes. No `X-Robots-Tag: noindex` is emitted for admin paths in production. Edge does not protect admin surfaces from indexation; that responsibility is fully delegated to the web layer's `robots.txt` and/or `<meta name="robots">`. If the web layer drops that, admin pages become crawlable from the edge's perspective.

9. **`/owner/*` paths have zero edge-level handling**
   - `src/index.ts:62–71` (regex only matches `/xx-xx/admin`)
   - Status: `missing`
   - Severity: `high`
   - Note: No regex covers `/owner/*` (or `/xx-xx/owner/*`). No edge noindex, no maintenance classification. Entirely web-layer-managed. The brief explicitly asks about `/owner/*`; the code has nothing for it.

10. **Bare `/admin` (no locale prefix) is not recognized**
    - `src/index.ts:64`
    - Status: `partial`
    - Severity: `medium`
    - Note: The regex requires a `xx-xx` prefix. A request to bare `/admin` or `/admin/foo` would NOT count as `isAdminRequest`, so it would be blocked in maintenance even with bypass enabled. Acceptable if the web app always uses locale-prefixed admin routes; brittle if it doesn't. Not strictly an SEO finding but a regex coverage observation noted because the brief asks about admin handling.

### 4. Redirect semantics at the edge

11. **Origin redirects preserved via `redirect: "manual"`**
    - `src/index.ts:185–192`
    - Status: `correct`
    - Severity: `n/a`
    - Note: 301/302 from Vercel or backend pass through with original `Location`. Crawlers see the real redirect chain.

12. **Stage noindex wrap preserves redirect status and Location**
    - `src/index.ts:195–202`
    - Status: `correct`
    - Severity: `n/a`
    - Note: When upstream is 3xx, the wrapped response copies `upstream.headers` (including `Location`) and `upstream.status`, then adds `X-Robots-Tag`. Redirect semantics survive the noindex wrap.

13. **`www → apex` 301 emitted at the edge for production**
    - `src/index.ts:46–51`
    - Status: `correct`
    - Severity: `n/a`
    - Note: `https://www.oglasino.com/...` → `https://oglasino.com/...` (301), preserving `path` and `search`. Standard apex canonicalization; only active when `WWW_HOST` is set (production). Stage has `WWW_HOST=""`.

### 5. Bot-vs-human routing

14. **No User-Agent inspection anywhere in the worker**
    - `src/index.ts` (whole file)
    - Status: `correct`
    - Severity: `n/a`
    - Note: No UA-based branching, no Googlebot/Bingbot special-case, no cloaking. Crawlers and humans receive identical responses for the same URL — the SEO-safe default.

### 6. Locale / baseSite routing

15. **Worker is locale-agnostic; routing is host-based only**
    - `src/index.ts:42–58` (host classification), `:119–134` (origin forwarding)
    - Status: `n/a` (delegated)
    - Severity: `n/a`
    - Note: Worker recognizes only `APEX_HOST`, `WWW_HOST`, `API_HOST`. All locale paths (`/rs-sr/...`, `/en-us/...`, `/me-cnr/...`, `/rsmoto-*/...`) forward to the same Vercel origin. Locale resolution, canonical URLs, and hreflang are entirely the web layer's responsibility — the worker does nothing to help or hurt at the edge.

16. **No separate `oglasino.me` domain configured for Montenegro market**
    - `wrangler.toml:14–68`
    - Status: `missing` (intentional or not — flagged for triage)
    - Severity: `medium`
    - Note: Both Serbia and Montenegro markets share the production apex `oglasino.com`. There is no edge mapping for a Montenegrin TLD. If the SEO plan includes a `.me` ccTLD for the ME market, it is not present in the worker config.

### 7. Headers added at the edge

17. **No `Cache-Control` added or stripped on normal forwarded traffic**
    - `src/index.ts:169–206`
    - Status: `correct`
    - Severity: `n/a`
    - Note: Cache headers pass through from origin unchanged. Crawl pressure depends entirely on Vercel/backend headers; the worker neither helps nor hurts.

18. **No `X-Frame-Options` / CSP set or modified at the edge**
    - `src/index.ts:169–206`
    - Status: `correct`
    - Severity: `n/a`
    - Note: Social-card and OG-image crawlers (Facebook, Twitter, LinkedIn, Slack) are unaffected by edge-level frame or CSP rules — none exist. Pass-through behavior.

19. **Request-side `X-Forwarded-Host` / `X-Forwarded-Proto` set**
    - `src/index.ts:178–183`
    - Status: `correct`
    - Severity: `n/a`
    - Note: Origin sees the public host. Useful for origin-side canonical/redirect logic; not an SEO concern at the edge itself.

20. **Maintenance response adds `X-Oglasino-Maintenance: true`**
    - `src/index.ts:151`, `:158`
    - Status: `correct`
    - Severity: `n/a`
    - Note: Custom header for observability. Not crawler-visible in any meaningful way; documented for completeness.

### 8. `rsmoto-*` locales and Montenegrin handling

21. **Admin regex excludes all three `rsmoto-*` web locales**
    - `src/index.ts:64`
    - Status: `broken` (for admin-bypass during maintenance — not strictly SEO)
    - Severity: `medium`
    - Note: `/rsmoto-sr/admin`, `/rsmoto-en/admin`, etc. are 8-2 segments and do not match `[a-z]{2}-[a-z]{2}`. In maintenance with admin bypass enabled, these routes would be blocked. Brief flags this as known. SEO impact is indirect — if the worker were ever used to emit per-route noindex, these locales would also be excluded; today the worker emits none, so SEO impact is zero.

22. **No edge-level robots / canonical / hreflang for `me/cnr` or `rsmoto-cnr`**
    - `src/index.ts` (whole file)
    - Status: `missing`
    - Severity: `high`
    - Note: Conventions Part 9 says Montenegrin (me/cnr) aliases to SR. The worker does nothing to canonicalize CNR → SR at the edge. If the web layer serves SR content under both `/rs-sr/...` and `/me-cnr/...` (or `/rsmoto-sr/...` and `/rsmoto-cnr/...`) without canonical tags, the result is duplicate content. Edge can't see this — it forwards both. Flag as a cross-repo seam.

23. **No edge-level differentiation between Serbia and Montenegro markets**
    - `src/index.ts:42–58`, `wrangler.toml`
    - Status: `missing`
    - Severity: `medium`
    - Note: Single apex `oglasino.com`. Geo-targeting, hreflang, and ccTLD strategy are all web-layer or DNS-layer concerns today. Documented here so triage can decide whether the worker should grow a role.

## Files touched

- none (read-only)

## Tests

- n/a (read-only audit, no code changes)

## Cleanup performed

- none needed

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed
- Part 4a (simplicity): see structured evidence in "For Mastermind"
- Part 4b (adjacent observations): see "For Mastermind"
- Part 6 (translations): N/A this session
- Other parts touched: Part 8 (architectural defaults — the worker as edge boundary) — confirmed

## Known gaps / TODOs

- none

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Cross-repo seams uncovered by this audit (the brief asks for these explicitly):**

  1. **Admin / owner noindex is entirely web-layer.** The worker does not emit `X-Robots-Tag: noindex` for `/xx-xx/admin/*` or any `/owner/*` path in production. Indexation protection for these surfaces depends entirely on `oglasino-web` `robots.txt` and/or `<meta name="robots">`. If the web layer drops those, the edge has no fallback. Decision needed: should the worker emit `X-Robots-Tag` for admin/owner paths in production as belt-and-suspenders, or remain delegate-only?

  2. **Locale canonicals / hreflang are entirely web-layer.** Worker forwards `/rs-sr/...`, `/me-cnr/...`, `/rsmoto-*/...` to the same Vercel origin without canonical rewrites or hreflang injection. Conventions Part 9 says CNR aliases to SR — if `oglasino-web` is not emitting a canonical from CNR pages to SR pages (or vice versa), duplicate content risk. This is a web audit question, not a router question — but the seam runs through the edge.

  3. **Stage maintenance pages have no `X-Robots-Tag: noindex`.** When stage is in maintenance, the response is 503 from `oglasino-maintenance.pages.dev` with no noindex header. The 503 status is the primary SEO signal; absent a stronger reason, this is fine. But it breaks the otherwise-uniform "everything on stage is noindexed" property. If that property is load-bearing, `maintenanceResponse` should also set `X-Robots-Tag` when `env.ENVIRONMENT === "stage"`.

  4. **Montenegro market has no edge presence.** No `.me` ccTLD configured, no edge canonical for `me/cnr` → SR, no per-market routing. If the Serbia + Montenegro SEO push requires distinct edge handling for ME, it does not exist today and would need a router brief.

  5. **`rsmoto-*` locales are invisible to the admin regex.** Already a known issue per the brief. Re-confirmed at `src/index.ts:64`. SEO impact today is zero (worker emits no per-locale headers) but it's a latent footgun if the worker ever grows per-locale logic.

- **Adjacent observations (Part 4b):**
  - `src/index.ts:156` — `fetch(MAINTENANCE_ORIGIN ...)` uses default `redirect: "follow"`, inconsistent with the explicit `redirect: "manual"` at line 189. Severity: low. I did not fix this because it is out of scope.
  - `src/index.ts:64` — admin regex requires locale prefix; bare `/admin` would not be classified as admin during maintenance. Severity: medium. I did not fix this because it is out of scope.
  - `src/index.ts:90–104` — the optional backend liveness probe runs an additional KV lookup and a fetch on every request when enabled. Not an SEO issue per se, but adds per-request latency that could feed into Core Web Vitals if the probe is slow. Severity: low. I did not fix this because it is out of scope.

- **Config-file drafts:** none required for this audit. Findings are pre-triage; Mastermind decides which become features, which become `issues.md` entries, and which are wontfix.
