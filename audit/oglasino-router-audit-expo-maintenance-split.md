# Audit — Expo Maintenance Split / Mobile-Aware Routing

**Repo:** oglasino-router
**Branch:** stage
**Date:** 2026-05-29
**Type:** Read-only audit. No code changes.
**Ground truth:** `src/index.ts` (213 lines) as it stands on disk this session, plus `wrangler.toml`. No prior report was read (the brief forbids it; an older `.agent/audit-worker-maintenance-split.md` exists and was intentionally not opened).

All line numbers below refer to `src/index.ts` as read this session.

---

## 0. One-paragraph orientation

The worker is a single bare `fetch` handler (`src/index.ts:41-142`) plus two module-level helpers (`maintenanceResponse` 144-173, `forwardToOrigin` 175-212) and two module constants (`MAINTENANCE_JSON` 32-37, `NOINDEX_HEADER` 39). It routes **by hostname** (apex/www/API), reads **three** KV flags inside one fail-open `try/catch`, optionally runs a backend `/health` probe, applies the maintenance gate, then forwards to the appropriate origin. There is no path-based origin routing today and no `/api/mobile/*` concept anywhere in the file.

---

## 1. KV reads

**All KV reads happen in exactly one place** — the `Promise.all` at `src/index.ts:84-88`, which sits **inside the fail-open `try` block** (`try` opens at line 83, `catch` at 92).

| KV key (literal) | Line | Local it sets | What the value gates | Inside fail-open try/catch? |
|---|---|---|---|---|
| `"maintenance.active"` | 85 | `maintenanceActive` | Whether the maintenance gate is armed at all (line 116). | Yes |
| `"admin.bypass.disabled"` | 86 | `adminBypassDisabled` | Whether admin/API requests are *also* blocked during maintenance (full lockdown) vs allowed through (line 117). | Yes |
| `"use.backend.check"` | 87 | `useBackendCheck` | Whether the optional `/health` liveness probe runs (line 101). | Yes |

Exact read calls, quoted verbatim:

```ts
const [maintRaw, bypassRaw, backendCheckRaw] = await Promise.all([
  env.CONFIG.get("maintenance.active", { cacheTtl: 30 }),
  env.CONFIG.get("admin.bypass.disabled", { cacheTtl: 30 }),
  env.CONFIG.get("use.backend.check", { cacheTtl: 30 }),
]);
maintenanceActive = maintRaw === "true";
adminBypassDisabled = bypassRaw === "true";
useBackendCheck = backendCheckRaw === "true";
```

**Value semantics:** each flag is the strict string comparison `raw === "true"`. Any other value (including `"false"`, `null` for a missing key, `"True"`, `"1"`, whitespace) resolves to `false`. There is no trimming or case-folding.

**Fail-open behavior (lines 92-98):** if any of the three `.get` calls throws, the `catch` resets **all three** locals to `false`:

```ts
} catch (_err) {
  // Fail open on KV errors — better to serve traffic than to lock everyone out.
  // All three flags fall back to false (equivalent to "no KV entry").
  maintenanceActive = false;
  adminBypassDisabled = false;
  useBackendCheck = false;
}
```

The three locals are also pre-initialized to `false` at declaration (lines 80-82), so the `catch` re-assignment is belt-and-suspenders. Net effect: a KV outage = "no maintenance, no lockdown, no probe" = serve everyone. This is the deliberate fail-open posture documented in CLAUDE.md.

**Confirmation of current state of the three named keys:** all three (`maintenance.active`, `admin.bypass.disabled`, `use.backend.check`) exist in the code today, are read together in the single `Promise.all`, each with `{ cacheTtl: 30 }`, and each coerced via `=== "true"`. There is **no fourth KV read** and no other `CONFIG.get` anywhere in the file. The `CONFIG` binding is declared in `Env` at line 22 and bound in `wrangler.toml` (`[[kv_namespaces]] binding = "CONFIG"`, id `0b9a66b9…` for stage/default, `dafb45ca…` for production).

---

## 2. The maintenance gate logic

The decision runs in three steps after the host checks.

### Step A — optional probe can *raise* maintenance (lines 100-110)

```ts
// Optional: backend liveness probe can force maintenance on
if (!maintenanceActive && useBackendCheck) {
  try {
    const probe = await fetch(`${env.BACKEND_ORIGIN}/health`, {
      cf: { cacheTtl: 30, cacheEverything: true },
    });
    if (!probe.ok) maintenanceActive = true;
  } catch (_err) {
    maintenanceActive = true;
  }
}
```

The probe only runs when maintenance is **not already** active **and** `use.backend.check` is true. A non-`ok` response (HTTP status outside 200–299) or a thrown fetch (network error / timeout / DNS) sets `maintenanceActive = true`. So a dead backend *fails closed into maintenance* — note this is the opposite posture from the KV fail-open, and it is intentional per the header comment block (`src/index.ts:15-18`).

### Step B — the block decision (lines 112-121)

```ts
if (maintenanceActive) {
  const shouldBlock = adminBypassDisabled || !isAdminRequest;
  if (shouldBlock) {
    return maintenanceResponse(isApi, path, url.search, env);
  }
}
```

Truth table (matches the matrix in the header comment, lines 10-13):

| `maintenanceActive` | `adminBypassDisabled` | `isAdminRequest` | `shouldBlock` | Outcome |
|---|---|---|---|---|
| false | — | — | (gate skipped) | Forward to origin |
| true | false | true | false | **Allowed through** (admin/API bypass) → forward |
| true | false | false | true | Maintenance response |
| true | true | true | true | Maintenance response (full lockdown — admin blocked too) |
| true | true | false | true | Maintenance response |

So `admin.bypass.disabled = false` means "bypass is **enabled**" (admin/API allowed during maintenance); `= true` means "bypass is **disabled**" (everyone blocked). The double-negative naming is worth noting for the redesign.

### Step C — fall-through to forwarding (lines 123-140)

If not blocked, the request forwards: API host → `BACKEND_ORIGIN`, otherwise → `FRONTEND_ORIGIN`, with `isStage` controlling the noindex header.

### The maintenance response itself (`maintenanceResponse`, lines 144-173)

Two distinct shapes depending on `isApi`:

**API branch (lines 150-159)** — synchronous, body is the module constant `MAINTENANCE_JSON`:

```ts
return new Response(MAINTENANCE_JSON, {
  status: 503,
  headers: {
    "Content-Type": "application/json; charset=utf-8",
    "Retry-After": "120",
    "Cache-Control": "no-store",
    "X-Oglasino-Maintenance": "true",
  },
});
```

`MAINTENANCE_JSON` (lines 32-37, quoted verbatim):

```ts
const MAINTENANCE_JSON = JSON.stringify({
  status: "maintenance",
  message:
    "Oglasino is undergoing maintenance. Please try again in a few minutes.",
  retryAfter: 120,
});
```

So an API client receives: **HTTP 503**, `Content-Type: application/json; charset=utf-8`, `Retry-After: 120`, `Cache-Control: no-store`, `X-Oglasino-Maintenance: true`, and the JSON body `{"status":"maintenance","message":"…","retryAfter":120}`.

**Non-API (frontend) branch (lines 162-172)** — fetches the maintenance page from `MAINTENANCE_ORIGIN` and re-wraps it as a 503:

```ts
return fetch(`${env.MAINTENANCE_ORIGIN}${path}${search}`).then((upstream) => {
  const headers = new Headers(upstream.headers);
  headers.set("X-Oglasino-Maintenance", "true");
  headers.set("Cache-Control", "no-store");
  headers.set("Retry-After", "120");
  return new Response(upstream.body, {
    status: 503,
    statusText: "Service Unavailable",
    headers,
  });
});
```

So a browser receives: **HTTP 503 "Service Unavailable"**, the upstream maintenance page's HTML body and headers, with `X-Oglasino-Maintenance: true`, `Cache-Control: no-store`, and `Retry-After: 120` forced on. `MAINTENANCE_ORIGIN` is `https://oglasino-maintenance.pages.dev` for both stage and production (`wrangler.toml`).

---

## 3. Request routing

**Routing is by hostname, not by path.** Sequence in the `fetch` handler:

1. **www → apex 301** (lines 52-57): only when `env.WWW_HOST` is non-empty (production only; stage has `WWW_HOST = ""`). Redirects `https://${APEX_HOST}${path}${search}` with 301.
2. **Host classification** (lines 59-60): `isApi = host === env.API_HOST`; `isFrontend = host === env.APEX_HOST`.
3. **Unknown host → 404** (lines 62-64): `new Response("Not found", { status: 404 })`.
4. **Origin selection** (lines 125-140): `isApi` → `forwardToOrigin(env.BACKEND_ORIGIN, …)`, otherwise → `forwardToOrigin(env.FRONTEND_ORIGIN, …)`.

**Is there path-based branching today?** Not for origin selection. The only place `path` influences control flow is the `isAdminRequest` computation (lines 68-77), which gates *maintenance bypass*, not *which origin* the request goes to. API vs page is decided purely by hostname (`API_HOST` = `api-stage.oglasino.com` / `api.oglasino.com`; `APEX_HOST` = `stage.oglasino.com` / `oglasino.com`).

**How would the worker distinguish a request bound for `/api/mobile/*`?** Everything needed is already on the request object — reporting, not designing:

- `url.pathname` is parsed at line 49 into the local `path` (`const url = new URL(request.url); … const path = url.pathname;`). A `/api/mobile/*` request would have `path === "/api/mobile/…"`. Note `path.startsWith("/api/")` is already evaluated (line 73) inside `isAdminRequest`, so a `/api/mobile/` prefix test would be the same idiom.
- `host` (line 48) tells API-subdomain from frontend. On the API subdomain the pathname may or may not carry an `/api` prefix — the code already hedges by treating *both* `isApi` (host match) and `path.startsWith("/api/")` (path match) as API traffic (lines 72-73), implying API requests can arrive either on the API host or under an `/api/` path on the frontend host.
- `request.headers` is fully available (it's used in `forwardToOrigin` at lines 184-185 to read `Host`). A mobile client could send a custom header (e.g. a device-id or client-type header) that the worker could read, but **no such header is read today**.

I am not proposing where the branch goes — see §7b for the natural seam.

---

## 4. The admin bypass

`isAdminRequest` is computed once (lines 66-77):

```ts
const isAdminRequest =
  // The admin page itself and anything under it (e.g. /rs-sr/admin, /en-us/admin/users)
  /^\/[a-z]{2}-[a-z]{2}\/admin(\/|$)/i.test(path) ||
  // All API traffic (admin talks to the backend)
  isApi ||
  path.startsWith("/api/") ||
  // Next.js static assets and chunks
  path.startsWith("/_next/") ||
  // Common static files the browser will request
  path === "/favicon.ico";
```

So a request counts as "admin infrastructure" if **any** of:
1. The locale-prefixed admin route regex matches (below), **or**
2. it's on the API host (`isApi`), **or**
3. its path starts with `/api/`, **or**
4. its path starts with `/_next/` (Next.js assets/chunks), **or**
5. its path is exactly `/favicon.ico`.

**The admin-request regex (quoted verbatim, line 70):**

```ts
/^\/[a-z]{2}-[a-z]{2}\/admin(\/|$)/i
```

It matches a leading locale segment of the exact form `xx-xx` (two ASCII letters, hyphen, two ASCII letters — case-insensitive via the `/i` flag) followed by `/admin`, then either a `/` or end-of-string. Matches `/rs-sr/admin`, `/en-us/admin/users`, `/ME-CNR/admin`; does **not** match `/admin` (no locale prefix), `/rs/admin` (single-segment locale), or `/rs-sr/administrator` (the `(\/|$)` boundary prevents the `administrator` false-positive). CLAUDE.md flags this regex as safety-critical and test-covered: if the locale format changes (segment lengths, separator), the regex must change in lockstep.

**Interaction with the gate:** `isAdminRequest` only matters when `maintenanceActive` is true. The block expression is `shouldBlock = adminBypassDisabled || !isAdminRequest` (line 117). Reading it plainly:

- If `adminBypassDisabled` is **true**, `shouldBlock` is always true regardless of `isAdminRequest` → full lockdown, admin included.
- If `adminBypassDisabled` is **false**, `shouldBlock` collapses to `!isAdminRequest` → admin infrastructure (admin pages, all API, Next assets, favicon) passes through; everything else gets the maintenance response.

---

## 5. Caching primitives in use

- **Cache API (`caches.default`)?** **No.** The string `caches` does not appear anywhere in the file.
- **In-memory module-scope mutable state?** **No.** The only module-scope values are the two `const`s `MAINTENANCE_JSON` (32-37) and `NOINDEX_HEADER` (39) — immutable string literals computed once at module load, used as response payloads/headers. They are not a cache and hold no per-request or cross-request state.
- **KV with TTL?** **Yes — this is the `cacheTtl = 30` the docs reference.** It appears in two distinct spots:

  1. **The three KV reads (lines 85-87):** each `env.CONFIG.get(key, { cacheTtl: 30 })`. This is Cloudflare KV's edge-cache TTL: the value of each flag is cached at the edge for 30 seconds, so a flag flip can take up to ~30s to propagate to a given edge location. Quoted: `env.CONFIG.get("maintenance.active", { cacheTtl: 30 })` (and the two siblings).
  2. **The backend health probe (lines 103-105):** `fetch(\`${env.BACKEND_ORIGIN}/health\`, { cf: { cacheTtl: 30, cacheEverything: true } })`. This is the Cloudflare *fetch* cache (`cf.cacheTtl`), not KV — it caches the `/health` probe response at the edge for 30s with `cacheEverything: true`. Effect: the liveness probe is not actually hit on every request; the probe result is reused for up to 30s.

  So `cacheTtl = 30` shows up **four times total** (three KV gets + one fetch `cf` option), all hardcoded to `30`, all serving the same "≤30s staleness" budget. It caches: (a) the three maintenance flag values, and (b) the `/health` probe result.

---

## 6. Origin forwarding shape

`forwardToOrigin` (lines 175-212):

- **Call shape:** `fetch(forwarded)` where `forwarded` is a `new Request(targetUrl, { method, headers, body, redirect: "manual" })` (lines 191-198). **`redirect: "manual"` is present** (line 195) — upstream 3xx responses pass through unchanged rather than being followed. CLAUDE.md flags this as a care area; do not change to `follow`.
- **Target URL:** `${originBase}${path}${search}` (line 182), where `originBase` is whichever origin the caller passed — `BACKEND_ORIGIN` for API host, `FRONTEND_ORIGIN` otherwise.
- **Origin URLs/hosts (from `wrangler.toml`):**
  - Stage: `FRONTEND_ORIGIN = https://oglasino-web-stage.vercel.app` (commented "placeholder until Phase 1E"), `BACKEND_ORIGIN = https://api-origin-stage.oglasino.com`.
  - Production: `FRONTEND_ORIGIN = https://oglasino-web-prod.vercel.app`, `BACKEND_ORIGIN = https://api-origin.oglasino.com`.
- **Header rewriting (lines 184-189):** copies all original headers, and if a `Host` header exists sets `X-Forwarded-Host` (to the original host) and `X-Forwarded-Proto: https`.
- **Stage noindex enforcement (lines 200-209):** when `addNoIndex` (i.e. `isStage`) is true, the upstream response is re-wrapped with `X-Robots-Tag` forced to `NOINDEX_HEADER` = `"noindex, nofollow, noarchive, nosnippet"` (line 39). Otherwise the upstream response returns unchanged (line 211). CLAUDE.md flags this as a care area.

**Is there an existing health-probe / liveness call to the backend?** **Yes.** Lines 101-110, already covered in §2 Step A: `fetch(\`${env.BACKEND_ORIGIN}/health\`, { cf: { cacheTtl: 30, cacheEverything: true } })`, run conditionally when `!maintenanceActive && useBackendCheck`. It is the only outbound call besides the origin forward and the maintenance-page fetch. **Evidence: this directly contradicts the brief's framing that a backend liveness probe is being newly "added" — see §8.**

---

## 7. Seams

### 7a — What shape could the worker return to a native mobile client that the Expo boot path could read as "maintenance"

The idiomatic shape **already exists** for API traffic: the `isApi` branch of `maintenanceResponse` (lines 150-159) returns **HTTP 503** with a JSON body `{"status":"maintenance","message":"…","retryAfter":120}` plus the explicit marker header `X-Oglasino-Maintenance: true` and `Retry-After: 120`. A native client can branch on any of three orthogonal signals — the 503 status, the `X-Oglasino-Maintenance` header, or `body.status === "maintenance"` — the most robust being the dedicated header since it is unambiguous and survives body changes. (Whether the Expo boot path *should* read maintenance from the worker vs. a separate backend endpoint is a cross-repo design question, not decided here.)

### 7b — Where a `/api/mobile/*` branch would naturally sit in the current control flow

The current API/frontend split is a single hostname decision at lines 59-60 and again at the forward step (lines 125-140). A path-aware mobile branch would naturally sit alongside that forward decision (after the maintenance gate, lines 123-140) — the point where `isApi` already selects `BACKEND_ORIGIN`, since `url.pathname` (the `path` local) is already in scope and `path.startsWith("/api/")` is an established idiom in the file. The maintenance gate (§2) sits *upstream* of that point, so any mobile-specific maintenance shaping would instead key off `path`/`isApi` inside `maintenanceResponse` (which already takes `isApi` and `path` as args).

### 7c — What a backend liveness probe would call and what counts as "down"

It already calls `${BACKEND_ORIGIN}/health` (line 103). "Down" today is defined by lines 106-109 as **either**: (a) `!probe.ok` — any HTTP status outside the 200–299 range (so a 500, 502, 503, 404, etc. all count as down), **or** (b) the `fetch` throwing — network error, DNS failure, connection refused, or timeout. **There is no explicit timeout configured** on the probe `fetch`; it relies on the Cloudflare Workers default subrequest behavior, and the `cf: { cacheTtl: 30, cacheEverything: true }` means a probe result is reused for up to 30s rather than re-probed per request. So "down" = non-2xx **or** thrown, with timeout governed implicitly by the platform rather than by an explicit `AbortSignal`/timeout in the code.

---

## 8. Brief vs reality (factual, for the design phase)

The brief frames the work as *"splitting the single maintenance flag into two dependency flags and adding mobile-aware routing with a backend liveness probe."* Against the code on disk:

1. **The worker is not on a "single maintenance flag."** It already reads **three** flags (`maintenance.active`, `admin.bypass.disabled`, `use.backend.check`) — §1.
2. **The backend liveness probe is not new — it already exists and is wired into the gate.** §2 Step A and §6 document the live `${BACKEND_ORIGIN}/health` probe, its fail-closed-into-maintenance behavior, its `use.backend.check` gate, and its 30s fetch cache.
3. **Mobile-aware routing genuinely does not exist** — there is no `/api/mobile/*` concept, no path-based origin routing, and no mobile-client header read (§3).

This is reported, not resolved — flagged for Mastermind in the session summary so the design starts from the audited reality rather than the brief's framing. No code was changed.

---

## 9. Inventory completeness

Every outbound effect and decision point in `src/index.ts` is accounted for above: host classification (§3), the three KV reads (§1), the health probe (§2/§6/§7c), the maintenance gate and both response shapes (§2/§7a), origin forwarding with `redirect: "manual"` and stage noindex (§6), and the caching primitives (§5). No `caches.default`, no module-scope mutable state, no second KV namespace, no additional outbound fetch. `wrangler.toml` bindings (`CONFIG` KV, the origin/host vars, `ENVIRONMENT`) match what the code consumes.
