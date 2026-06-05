# Audit — Deep Linking (Universal/App Links edge layer) — oglasino-router

**Type:** READ-ONLY audit. No code, config, KV, or `wrangler` changes made.
**Date:** 2026-06-04 · **Branch:** stage (unchanged)
**Subject paths:** `https://oglasino.com/.well-known/apple-app-site-association`, `https://oglasino.com/.well-known/assetlinks.json`
**Evidence base:** `src/index.ts` (read in full, 274 lines), `wrangler.toml` (read in full). No existing `.well-known`/AASA/assetlinks reference anywhere in `src/`, `tests/`, or `wrangler.toml` (grep: zero matches).

---

## 1. Current request flow for an arbitrary path

Trace of `GET https://oglasino.com/.well-known/apple-app-site-association` against production env vars (`APEX_HOST=oglasino.com`, `WWW_HOST=www.oglasino.com`, `API_HOST=api.oglasino.com`):

1. `url` parsed; `host="oglasino.com"`, `path="/.well-known/apple-app-site-association"` (`src/index.ts:71-73`).
2. www→apex 301 check (`:76`): `host === env.WWW_HOST` → `"oglasino.com" === "www.oglasino.com"` → false. **Not redirected** (on the apex host). See Q3/Q8 for the www host.
3. `isApi = host === API_HOST` → false; `isFrontend = host === APEX_HOST` → **true** (`:83-84`). Host-validity gate at `:86` passes (not a 404).
4. `isMobile = path.startsWith("/api/mobile/")` → **false** (`:92`).
5. `isAdminRequest` (`:96-105`): admin regex no-match, `isApi` false, not `/api/`, not `/_next/`, not `/favicon.ico` → **false**. (Detail in Q6.)
6. All four KV flags read (`:114-133`), fail-open.
7. `isMobile` branch skipped (`:139`).
8. **Maintenance composition** (`:172-178`): `webDown = maintenance.web.active || maintenance.backend.active`. If either is true → `shouldBlock = adminBypassDisabled || !isAdminRequest`; since `isAdminRequest=false`, `!isAdminRequest=true`, so `shouldBlock=true` **regardless of the admin bypass** → returns `maintenanceResponse(isApi=false, …)` (503 maintenance page). See Q5.
9. If maintenance is **not** active: `isApi` false (`:180`) → falls to `forwardToOrigin(env.FRONTEND_ORIGIN, path, …)` (`:189-195`).

**Answer:** On the apex host with no maintenance, the request is **origin-forwarded to `FRONTEND_ORIGIN` (Vercel frontend)** verbatim. Whether the file is actually served is then entirely up to the Vercel/web origin (cross-repo — see For Mastermind). With maintenance active, it is intercepted by the **maintenance gate** and never reaches the origin.

---

## 2. `.well-known` handling today

**None.** There is no path-specific handling for `/.well-known/*` (no ACME challenge handling, no AASA/assetlinks short-circuit, nothing). Confirmed by reading `src/index.ts` end-to-end and by grep across `src/`, `tests/`, `wrangler.toml` (zero matches for `well-known`, `app-site-association`, `assetlinks`, `universal link`, `deep link`). A `.well-known` path is treated as an ordinary frontend path and forwarded to `FRONTEND_ORIGIN`.

---

## 3. Redirect behavior

The worker contains **exactly one redirect**, and it does not follow or add redirects on the forward path:

- **www→apex 301** (`src/index.ts:76-81`): `if (env.WWW_HOST && host === env.WWW_HOST) return Response.redirect(\`https://${env.APEX_HOST}${path}${url.search}\`, 301)`. This fires for **every path including `/.well-known/*`** when the request arrives on `www.oglasino.com`. **This is the single biggest risk for this feature.** Universal Links / App Links association fetches do **not** follow redirects — iOS fetches `apple-app-site-association` with no redirect-following, and Android's verifier requires the file be served without redirects. A `.well-known` fetch hitting `www.oglasino.com` would receive a 301 and fail verification. On the **apex** host this redirect does not fire.
- **No locale redirect, no trailing-slash redirect, no http→https redirect** exists in the worker. None apply to `.well-known`.
- **Forward path uses `redirect: "manual"`** (`:256`): upstream 3xx responses pass through **unchanged** — the worker neither follows nor rewrites them. So the worker itself never *adds* a redirect to an apex `.well-known` response. **However**, if the Vercel origin emits a 301/302 for `/.well-known/...` (e.g. an i18n/trailing-slash rule on the web side), the worker faithfully passes it through, and that would break verification. That redirect would originate cross-repo (web), not here.

**Answer:** The worker adds no redirect on the apex host and follows none. The www host is 301'd unconditionally (risk if the OS/crawler ever fetches the www host). Any origin-emitted 3xx on apex passes through transparently (cross-repo risk).

---

## 4. Header injection

- **Production (`ENVIRONMENT="production"` → `isStage=false`):** `forwardToOrigin` takes the early-return at `:272` (`addNoIndex` false) and returns the **upstream response object unchanged** — body, status, and **all headers including `Content-Type` pass through untouched**. The worker injects **nothing** on a forwarded production response and **cannot break `Content-Type: application/json`** on this path.
- **Stage (`isStage=true`):** `forwardToOrigin` clones headers and sets `X-Robots-Tag: noindex, nofollow, noarchive, nosnippet` (`:262-269`). It sets **only** `X-Robots-Tag`; it does **not** touch `Content-Type`, so the JSON content-type is preserved. The `noindex` header *would* attach to a stage `.well-known` response (cosmetic; see below).
- **Request headers** `X-Forwarded-Host` / `X-Forwarded-Proto` are added to the **outbound request to the origin** (`:247-250`), not to the client-facing response — no effect on response content-type.

**Answer:** Nothing the worker injects can break `Content-Type: application/json`. On production no headers are added at all. On stage only `X-Robots-Tag` is added; the `noindex` header *would* attach to a `.well-known` response on stage — cosmetically undesirable for an association file, and the brief notes Google Search fetches `assetlinks.json` during indexing for app-action eligibility, so a `noindex` on that response is worth avoiding — but Universal Links are production-only (Q9), and production adds no such header.

---

## 5. Maintenance gating

**Yes — this is a real break.** With `maintenance.web.active=true` **or** `maintenance.backend.active=true`, `webDown` is true (`src/index.ts:172`). Because `isAdminRequest=false` for `/.well-known/*` (Q6), `shouldBlock = adminBypassDisabled || !isAdminRequest` evaluates to `true` **irrespective of the admin bypass** (`:174`). The request returns `maintenanceResponse(isApi=false, …)` → a 503 from `MAINTENANCE_ORIGIN` with `X-Oglasino-Maintenance: true` (`:218-233`), **not** the association file.

**Impact:** During any web- or backend-maintenance window, both `apple-app-site-association` and `assetlinks.json` would return 503. iOS caches AASA but re-fetches periodically and on app update; Android re-verifies on install/update. A 503 during a re-verification window can de-verify the domain association and silently kill deep-link opening until the next successful fetch.

**Recommendation:** `.well-known/{apple-app-site-association,assetlinks.json}` should **bypass the maintenance gate**. The cleanest way is the worker-serves-directly short-circuit (Q7) placed *before* the maintenance composition — that exempts these paths from maintenance by construction without weakening the matrix for any other path. (Note: this would be a deliberate, documented exception to the maintenance matrix — flag for Mastermind, do not implement under a read-only brief.)

---

## 6. Admin regex

The admin regex is `/^\/[a-z]{2}-[a-z]{2}\/admin(\/|$)/i` (`src/index.ts:98`). It requires the path to begin with `/<two-letters>-<two-letters>/admin`. `/.well-known/...` begins with `/.` (a literal dot where the regex expects `[a-z]`), so it **does not match**. None of the other `isAdminRequest` clauses match either (`isApi` false, not `/api/`, not `/_next/`, not `/favicon.ico`). **Confirmed: the admin regex does not match or interfere with `/.well-known/*`.** (The practical consequence is the *opposite* of helpful — because `.well-known` is *not* classified as admin infrastructure, it gets blocked during maintenance; see Q5.)

---

## 7. Can the worker serve these two files directly?

**Yes, cleanly — and it is the recommended approach.** The two files are static (bundle IDs, team IDs, SHA-256 cert fingerprints); they don't change between deploys, so an inline static string (or a bundled asset) is appropriate and adds no runtime dependency.

**Where the short-circuit must sit.** A match on `path === "/.well-known/apple-app-site-association" || path === "/.well-known/assetlinks.json"` returning a `200` with an inline body and `Content-Type: application/json` must run **before all of**:

- the **maintenance gate** (`src/index.ts:172-178`) — otherwise a maintenance window 503s the file (Q5);
- the **origin forward** (`:180-195`) — so the worker answers authoritatively and never depends on Vercel for these paths;
- the **KV flag reads** (`:107-133`) — the short-circuit needs none of them, and skipping them makes the response immune to a KV outage (consistent with, and stronger than, the fail-open posture).

**The one ordering subtlety — the www 301.** The www→apex redirect is the *first* thing the handler does (`:76-81`). There are two viable placements:

- **Option A — before the www redirect (`:76`), gated on `host === APEX_HOST || host === WWW_HOST`:** serves the file directly on *both* hosts with no redirect. Required if the association file must be reachable on `www.oglasino.com` without a redirect.
- **Option B — after the host-validity gate (`:88`) but before the KV reads (`:107`):** simpler, apex-only. A www fetch would still be 301'd to apex first; since the OS does not follow that redirect, **www would not work** under Option B. Acceptable only if the app's Associated Domains / intent filters are scoped to the apex host exclusively.

Given iOS/Android verifiers don't follow redirects, **Option A is the safer insertion point** if there's any chance the OS or a crawler fetches the www host. The short-circuit should set `Content-Type: application/json` (note: `apple-app-site-association` has *no* file extension and still must be `application/json`), `200`, and ideally a sane `Cache-Control`. It would *not* go through `forwardToOrigin`, so the stage `noindex` header would not auto-attach — add it explicitly only if these are to be served on stage (not required for prod-only v1).

**Constraints recap:** runs before maintenance + forward + KV; needs no bindings; must hard-set the content-type and a 200; decide apex-only vs apex+www by where it's inserted relative to `:76`. *(Code intentionally not written — read-only brief.)*

---

## 8. www vs apex

They are handled **differently** (production only). `www.oglasino.com` is **unconditionally 301-redirected to the apex** for every path (`src/index.ts:76-81`), including `/.well-known/*`. `oglasino.com` (apex) is the host that actually forwards to the frontend / serves content. Stage has `WWW_HOST=""`, so the www branch is dead on stage (no www variant).

**Implication for this feature:** if the app's Associated Domains (`applinks:`) or Android App Links host patterns include `www.oglasino.com`, the association fetch on www will hit a 301 and fail verification under today's code. Either (a) scope the app's domains to the apex only, or (b) serve the files directly on both hosts via the Option A short-circuit (Q7). This is a decision the app/web side must align on — noted, not acted on.

---

## 9. Per-environment

This worker serves **both stage and production**, selected by `wrangler.toml` `[env.*]` (`wrangler.toml:35-67`):

- **Production** (`[env.production]`): `APEX_HOST=oglasino.com`, `WWW_HOST=www.oglasino.com`, `API_HOST=api.oglasino.com`.
- **Stage** (default + `[env.stage]`): `APEX_HOST=stage.oglasino.com`, `WWW_HOST=""`, `API_HOST=api-stage.oglasino.com`.

Universal/App Links are bound to `oglasino.com`, which only the **production** env fronts. **Confirmed: the host that must serve the association files (`oglasino.com` / `www.oglasino.com`) is fronted exclusively by the production environment.** Stage (`stage.oglasino.com`) is not a universal-links domain and is effectively out of scope for v1 — though if any `.well-known` short-circuit is added, it will live in the same `src/index.ts` and run on both envs, so it should be host-agnostic by construction (it keys off `path`, not env).

---

## Care areas touched

| Documented care area | Touched? | Note |
|---|---|---|
| **Maintenance matrix** | **Yes (risk + likely change)** | `.well-known` is blocked with a 503 during web/backend maintenance (Q5). A direct-serve short-circuit before the gate is a deliberate, documented exception to the matrix — must update the matrix comment block in lockstep if implemented. |
| **`redirect: "manual"` forwarding** | Yes (relevant, no change needed) | Why the worker doesn't *add* redirects on apex (Q3). Keep as-is. |
| **Stage `noindex` (`addNoIndex`)** | Yes (cosmetic only) | Would attach `X-Robots-Tag` to a stage `.well-known` response; not on production (Q4). Don't weaken the header — just don't route the direct-serve response through `forwardToOrigin`. |
| **Admin regex** | Yes (verification only) | Confirmed no match/interference with `/.well-known/*` (Q6). No change. |
| **Fail-open KV reads** | Yes (favorably) | A direct-serve short-circuit placed before the KV reads makes association files immune to KV outages — strictly stronger than fail-open. No change to the existing fail-open logic. |

---

## For Mastermind

### Recommendation: worker-serves-directly vs origin-forwards

**Serve the two files directly from the worker (Option A short-circuit, Q7).** Rationale:

1. **Maintenance immunity.** Today these paths 503 during any web/backend maintenance window (Q5). Serving them before the maintenance gate is the only way to guarantee the OS/crawler always gets the file — and re-verification during a maintenance window can silently de-verify the domain association.
2. **Redirect immunity.** Direct-serve removes dependence on the Vercel origin's redirect/trailing-slash/i18n rules (any origin-emitted 3xx passes through the worker transparently under `redirect: "manual"` — Q3), which is exactly what would silently break verification.
3. **Static content, zero new deps.** Bundle IDs / cert fingerprints don't change between deploys; an inline string fits the "no new dependency / no config for one value" simplicity rule.
4. **KV-outage immunity** as a bonus (Q7).

The cost is that the static JSON lives in `src/index.ts` and must be updated on a deploy when an app's team/bundle/fingerprint changes (rare). That is an acceptable trade vs. the silent-break risks of origin-forwarding.

### Redirect / maintenance risks found (highest first)

1. **HIGH — maintenance 503s the association files.** `src/index.ts:172-178`. During web/backend maintenance, `/.well-known/*` returns the 503 maintenance page, not the file. Recommend a documented maintenance-gate exemption via the direct-serve short-circuit.
2. **HIGH — www host 301s every path including `.well-known`.** `src/index.ts:76-81`. If the app's associated domains include `www.oglasino.com`, verification fails (verifiers don't follow redirects). Decide apex-only vs apex+www; if apex+www, the short-circuit must precede the www redirect (Q7 Option A).
3. **MEDIUM (cross-repo) — origin-emitted redirects pass through.** If `oglasino-web` (Vercel) has any trailing-slash or i18n redirect that catches `/.well-known/...`, the worker forwards it unchanged and it breaks verification. Direct-serve eliminates this dependency. (Cross-repo — see below.)
4. **LOW (cosmetic, stage-only) — `X-Robots-Tag: noindex` on stage `.well-known`.** `src/index.ts:262-269`. Not a prod concern (prod adds no header); just don't route a direct-serve response through `forwardToOrigin`.

### Cross-repo, noticed but NOT acted on

- **`oglasino-web` currently owns `.well-known` serving (per `oglasino-docs/features/seo-foundation.md:560,569` and `571`).** `seo-foundation.md:571` already states the router "exempts `/.well-known/*` paths from any rewriting or redirect logic." That design assumes web serves the files and the router merely stays out of the way — but today the router does **not** stay out of the way (maintenance gate + www 301 + pass-through of origin 3xx). If Mastermind adopts worker-serves-directly, that's a change of ownership from web→router and the seo-foundation plan and `decisions.md` should record it. I did not edit any docs (read-only, and the four config files are Docs/QA-owned).
- **`oglasino-docs/features/password-reset.md:150,190`** notes there is no `.well-known`/app-association infra in web today and that AASA/assetlinks ownership is an open cross-repo contract Igor brokers. This audit's direct-serve recommendation is a concrete answer to that open question, scoped to the router. Cross-repo coordination (app Associated Domains / intent filter host scope) is required and out of this repo's scope.

### Config-file impact

If worker-serves-directly is adopted: (a) the **maintenance matrix comment block** in `src/index.ts:1-43` must gain a documented `.well-known` exemption clause, and (b) `decisions.md` should record the web→router ownership shift for the association files, and (c) `seo-foundation.md` §11.14 should be reconciled with the new router responsibility. These are draft pointers for Docs/QA — **no config files were edited in this session.**

---

## Conventions check (Part 4)

- **Read-only:** confirmed. No edits to `src/index.ts`, `wrangler.toml`, `tests/**`, `package.json`, `tsconfig.json`, or any docs/config file.
- **No `wrangler` commands, no KV reads/writes, no deploys** run.
- **Cleanliness:** N/A — no code changed. The only file written this session is this audit (`.agent/audit-deep-links.md`), per the brief's Output instruction, plus the session summary.
- **No cross-repo edits.** Cross-repo observations are reported above, not acted on.
