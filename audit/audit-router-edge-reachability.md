# Audit — Router Edge Reachability (backend-security-hardening, Phase-2 edge addendum)

**Repo:** `oglasino-router` · **Branch:** `stage` · **Mode:** READ-ONLY (no code, no commit/push/deploy, no KV access)
**Date:** 2026-06-03
**Scope:** What the Cloudflare router worker (`src/index.ts`, 274 lines, the entire product) actually forwards to the backend origin, vs. blocks/handles at the edge. The backend audit listed paths as "publicly reachable" from Spring authorization rules alone; this settles which of those paths the *edge* actually routes to origin.

**Tool-reliability discipline (per brief + state.md Risk Watch):** every `file:line` below was opened with `view` (full-file Read of `src/index.ts`) **and** cross-checked with an independent `grep -n`. The two reads agree on every cited line. Where I note a disagreement or a gap, I say so explicitly. Cross-check command output is preserved in the session record.

---

## Part 1 — The worker's routing model: what reaches origin at all

### The forwarding decision: HOST-based pass-through, no path filtering

The worker classifies the request by **hostname**, not by path prefix. There is no allow-list of forwarded path prefixes and no block-list of named paths. Within an accepted host, **every** path is proxied to that host's origin (subject only to maintenance gating and the one mobile rewrite below).

Host classification (`src/index.ts:83-88`):

```ts
const isApi = host === env.API_HOST;
const isFrontend = host === env.APEX_HOST;

if (!isApi && !isFrontend) {
  return new Response("Not found", { status: 404 });
}
```

The terminal forwarding decision (`src/index.ts:180-195`):

```ts
if (isApi) {
  return forwardToOrigin(
    env.BACKEND_ORIGIN,
    path,
    url.search,
    request,
    isStage
  );
}
return forwardToOrigin(
  env.FRONTEND_ORIGIN,
  path,
  url.search,
  request,
  isStage
);
```

`forwardToOrigin` builds the upstream URL by concatenating the origin base with the **unmodified** path and query (`src/index.ts:243`):

```ts
const targetUrl = `${originBase}${path}${search}`;
```

There is no `switch`/`if` on `path` to gate forwarding. The independent grep for routing keywords (`internal|/api/secure|/api/public|/api/auth|/media|/health|/error|prometheus|/info|blocklist|allowlist|prefix`) returned **only** the two comment/probe lines (`:35`, `:146`) — zero path-prefix routing rules. Confirmed: **this is pass-through-everything, partitioned by host.**

So the model is:

- **Request to `API_HOST`** (`api.oglasino.com` prod / `api-stage.oglasino.com` stage) → every path forwarded to `BACKEND_ORIGIN`.
- **Request to `APEX_HOST`** (`oglasino.com` / `stage.oglasino.com`) → every path forwarded to `FRONTEND_ORIGIN` (Vercel).
- **Request to `WWW_HOST`** (prod only) → 301 to apex (`src/index.ts:76-81`).
- **Request to any other host** → edge `404 "Not found"` (`src/index.ts:87`), never reaches any origin.

The worker decides *which origin* by host; it never decides *whether to forward* by path (except mobile gating below). A backend path is therefore internet-reachable iff a client sends it to the **API host** — and an internet client fully controls which host it addresses.

### The origin hostname(s)

From `wrangler.toml` (`[vars]` stage `:18`; `[env.production.vars]` `:59`):

- **Backend origin:** `https://api-origin-stage.oglasino.com` (stage) / `https://api-origin.oglasino.com` (production). The top-of-file comment (`src/index.ts:3`) calls this the "droplet (via gray-cloud origin)".
- **Frontend origin:** `https://oglasino-web-stage.vercel.app` (stage placeholder, per the inline comment) / `https://oglasino-web-prod.vercel.app` (prod).
- **Maintenance origin:** `https://oglasino-maintenance.pages.dev` (both envs).

**Can the backend origin be reached without the worker?** Partially answerable from this repo. The worker forwards to the named `api-origin[-stage].oglasino.com` host. Whether that hostname is itself directly resolvable/reachable from the internet (i.e. orange-cloud proxied-and-locked-to-worker vs. gray-cloud/DNS-only origin that bypasses Cloudflare) is **a Cloudflare DNS/origin-config question I cannot settle from the worker source or `wrangler.toml`** — routes and DNS are configured in the Cloudflare Dashboard (`wrangler.toml:5-9` states routes live in the Dashboard, not this file). The comment "via gray-cloud origin" *suggests* a separate origin hostname exists precisely so the worker can reach it; if that hostname is publicly resolvable and not IP-allow-listed to Cloudflare, it is a worker bypass. **Flagged for seam analysis — needs an infra/DNS check, not a code read.** What I *can* state: the worker itself forwards to `api-origin[-stage].oglasino.com` and adds no origin authentication (see Part 3).

### Path rewriting

Exactly **one** rewrite exists. Mobile traffic (`src/index.ts:157-165`):

```ts
// Strip the /mobile segment: /api/mobile/<rest> → /api/<rest>.
const backendPath = "/api" + path.slice("/api/mobile".length);
return forwardToOrigin(
  env.BACKEND_ORIGIN,
  backendPath,
  url.search,
  request,
  isStage
);
```

`/api/mobile/<rest>` → backend sees `/api/<rest>`. The `/mobile` segment is worker-only; the backend never sees it. (Mobile is identified purely by path — `isMobile = path.startsWith("/api/mobile/")`, `src/index.ts:92` — and arrives on the apex host in prod or the API host in stage, per the `:90-91` comment.)

No other path is stripped, prefixed, or rewritten. All non-mobile forwards pass `path` verbatim (`:181-187`, `:189-195`, concatenated at `:243`). **Consequence for the audit:** every backend path below is seen by the origin exactly as the client sent it, *except* `/api/mobile/*`, which arrives at the backend as `/api/*`.

---

## Part 2 — Per-path reachability table (the deliverable)

**Governing fact (stated once):** the worker forwards *everything* on a per-host basis with no path filtering. On the **API host**, every path below is FORWARDED to `BACKEND_ORIGIN` verbatim. On the **apex host**, every path is FORWARDED to `FRONTEND_ORIGIN` (Vercel), which is not the backend. None of these paths is blocked, inspected, or special-cased at the edge (the sole exceptions: the `/api/mobile/*` rewrite, and maintenance-mode 503 gating). The deciding code is `src/index.ts:180-195` + `:243` for all rows.

| Path | Verdict (via API host) | Edge code that decides | Notes |
|---|---|---|---|
| `/api/secure/**` | **FORWARDED to origin** (verbatim) | `:180-188`, `:243` | No edge auth; Spring's `@PreAuthorize`/filters are the only gate. |
| `/api/public/**` | **FORWARDED to origin** (verbatim) | `:180-188`, `:243` | — |
| `/api/auth/**` | **FORWARDED to origin** (verbatim) | `:180-188`, `:243` | — |
| `/internal/**` | **FORWARDED to origin** (verbatim) | `:180-188`, `:243` | **No edge block.** The worker has no `/internal` rule; on the API host it pass-throughs to the backend. So `InternalTokenFilter` is the *only* gate — the edge adds **no** defense-in-depth here. (See "challenge" note below.) |
| `/actuator/**` (incl. `/health`, `/health/readiness`, `/health/liveness`, `/info`, `/prometheus`) | **FORWARDED to origin** (verbatim) | `:180-188`, `:243` | **LOAD-BEARING ROW: YES, reaches the backend.** No path filter. `/actuator/prometheus` and `/actuator/info` on `api.oglasino.com` are proxied straight to the droplet. |
| `/error` | **FORWARDED to origin** (verbatim) | `:180-188`, `:243` | — |
| `/health` (bare, non-actuator) | **FORWARDED to origin** (verbatim) | `:180-188`, `:243` | Not special-cased; forwarded like any path. |
| `/media/**` | **FORWARDED to origin by host — NOT served at edge** | `:180-195`, `:243` | The worker does **not** serve `/media` from CDN/R2. The `Env` interface (`src/index.ts:45-54`) binds only `CONFIG` (KV) and string vars — **no R2 bucket binding**. On the API host `/media/*` → backend (likely 404, no handler per brief); on apex → Vercel. Image serving is a *different* worker (`oglasino-image-worker`, referenced in `wrangler.toml:9`), not this one. |

**On the apex host**, every row above instead forwards to `FRONTEND_ORIGIN` (Vercel) — i.e. these backend paths do not reach the backend when addressed to `oglasino.com`; they hit Vercel and 404/handle there. Backend reachability requires the **API host**. (Quirk worth noting: during maintenance, `/api/*` on the apex host is classified `isAdminRequest` at `:99-101`, but the origin is still `FRONTEND_ORIGIN` because `isApi` is false at `:180` — so it never reaches the backend regardless. Not a reachability path to origin.)

**No "NOT ROUTED / undetermined" rows.** Every path matches the host-pass-through default; nothing falls through to an indeterminate state. The only true edge-terminal outcomes in the worker are: unknown-host `404` (`:87`), `www`→apex `301` (`:76-81`), and maintenance `503` (`:155`, `:176`, via `maintenanceResponse`). None of the audited paths is among these.

---

## Part 3 — Maintenance / liveness probe path (boot-loop cross-check)

**Exact origin path the probe hits** (`src/index.ts:143-152`):

```ts
if (!backendDown && useBackendCheck) {
  try {
    const probe = await fetch(
      `${env.BACKEND_ORIGIN}/actuator/health/readiness`,
      { cf: { cacheTtl: 30, cacheEverything: true } }
    );
    if (!probe.ok) backendDown = true;
  } catch (_err) {
    backendDown = true;
  }
}
```

- **Path:** `BACKEND_ORIGIN + /actuator/health/readiness`. Cross-checked: `grep -n actuator` returns exactly `:35` (comment) and `:146` (this fetch). No other actuator path is probed. This matches the maintenance-split feature's re-point (state.md: probe "re-pointed at dependency-aware `/actuator/health/readiness`").
- **Auth:** **none.** The probe is a bare `fetch(url, { cf: {...} })` — no `headers` argument, so it carries no internal token, no Authorization, nothing. It is an **unauthenticated GET** straight from the worker to `BACKEND_ORIGIN` (it bypasses the host-routing logic entirely — it's a direct origin fetch, not a forwarded client request). The `cf` option only sets a 30s edge cache.
- **Gating:** runs only when `use.backend.check` KV flag is true **and** the backend isn't already known-down, and only on the mobile path (`isMobile` branch, `:139-153`). It gates **mobile only** — never web.

**Boot-loop / default-deny implication for H1:** because the edge probe is an **unauthenticated GET to `/actuator/health/readiness`**, the H1 default-deny flip **must keep `/actuator/health/readiness` `permitAll`** or the edge backend-availability gate breaks (the probe would get 401/403, `probe.ok` is false → `backendDown=true` → all mobile traffic 503s even when the backend is healthy). This is the *same path* the brief flags for the docker healthcheck. **I can confirm the edge probe uses `/actuator/health/readiness`; I cannot confirm the docker healthcheck path** — that lives in `oglasino-backend` (cross-repo, not readable here). The brief's assumption that they're the same path is plausible and consistent with the maintenance-split design, but the docker side must be confirmed in the backend audit, not asserted from this repo. **If they are the same path, H1 keeps one `permitAll` and satisfies both consumers; if they differ, H1 must re-allow both.**

---

## Part 4 — Admin-bypass / locale-prefix interaction (note in passing)

The admin-request regex `/^\/[a-z]{2}-[a-z]{2}\/admin(\/|$)/i` (`src/index.ts:98`) and the `isAdminRequest` set (`:96-105`) feed **only** the maintenance-gating decision (`:174` `shouldBlock = adminBypassDisabled || !isAdminRequest`) — i.e. *who* gets through *during maintenance*. They do **not** change which origin a request is forwarded to (that is decided solely by `isApi`/`isFrontend` host at `:180-195`). No admin path is routed to a backend route outside what the host pass-through already allows; admin pages forward to Vercel, admin API calls forward to the backend as ordinary `/api/*` paths. **No reachability impact on the Part 2 table.**

---

## Bottom line for H1

**1. Does `/actuator/*` (esp. `/prometheus`, `/info`) reach the origin from the internet through the worker?**

**YES.** The worker applies no path filtering. Any path sent to the API host (`api.oglasino.com` / `api-stage.oglasino.com`), including `/actuator/prometheus` and `/actuator/info`, is forwarded verbatim to `BACKEND_ORIGIN` (`src/index.ts:180-188`, `:243`). The edge does not gate, strip, or block `/actuator/*`. **Backend `/actuator` lockdown is warranted — the "prometheus publicly scrapeable" finding stands; it is not mooted by the edge.** (Caveat carried up from Part 1: confirm in infra whether `api-origin[-stage].oglasino.com` is itself directly reachable, bypassing the worker — that would be an *additional*, separate exposure.)

**2. What is the minimum set of non-`/api/secure` paths the worker forwards to origin?**

**All of them — the edge narrows nothing.** The worker is pass-through-everything by host: on the API host, *every* path reaches the backend; on the apex host, *every* path reaches Vercel. There is no edge allow-list to shrink the surface. The only edge transformations are: the `/api/mobile/*` → `/api/*` rewrite (`:158`), the unknown-host `404` (`:87`), the `www`→apex `301` (`:76-81`), and maintenance `503` gating (`:155`/`:176`). **The real-world public backend surface H1 must account for is the *entire* Spring surface — identical to the Spring-rules-only surface the backend audit listed. The edge removes nothing from it.**

### Challenge the brief (which world are we in)

The brief asked us to determine whether the worker is **pass-through-everything** or **allow-list-by-prefix**. **It is pass-through-everything (partitioned by host).** Per the brief's own framing, this means **the backend's Spring-layer authorization is the only gate, and every backend audit finding stands at full severity.** The `/actuator` lockdown does **not** drop off the feature. Specifically:

- `/internal/**` is **not** edge-blocked — `InternalTokenFilter` is the sole defense (no defense-in-depth at the edge to lean on).
- `/actuator/prometheus` and `/actuator/info` are fully internet-reachable via the API host → backend lockdown required.
- The H1 default-deny re-allow list **must** include `/actuator/health/readiness` (unauthenticated edge probe + likely docker healthcheck), and per state.md also `/actuator/health/**`, `/error`, `OPTIONS`, and `/health` (or delete `/health`) — none of which the edge will protect if Spring stops permitting them.

---

## Notes / caveats

- **Origin-bypass question is unresolved here** and is the one item this audit cannot close from inside the repo: whether `api-origin[-stage].oglasino.com` is reachable without the worker. Needs a Cloudflare DNS/origin-lock check in seam analysis (Part 1).
- **Docker healthcheck path** is asserted-equal-to the edge probe path by the brief but lives in `oglasino-backend`; confirm there, not here.
- All verdicts above rest on the full current `src/index.ts` (read end-to-end) cross-checked by `grep`; the two reads agreed on every cited line. No fabricated-content disagreement was encountered this session.
