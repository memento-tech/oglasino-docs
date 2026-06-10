# Audit — oglasino-router — Q11 legal-document fact verification

**Session type:** READ-ONLY AUDIT. No source/config files modified.
**Date:** 2026-06-10
**Scope verified against:** `src/index.ts`, `wrangler.toml`, `package.json`, `.github/workflows/` — each claim confirmed by direct read AND `rg`.

---

## Q11(a) — Personal data + logging

**Verdict: The worker reads NO personal data and logs NOTHING. No `console.log`. No Logpush/observability config in-repo. Retention config is NOT FOUND in-repo (Cloudflare account-level, outside this repo).**

### Evidence

**No logging of any kind.** `rg "console\.|logpush|tail_consumer|observability|logger|\.log\("` over `src/` and `wrangler.toml` → **no matches**. The worker emits no logs and declares no log sink.

**No personal-data reads.** `rg -i "cf-connecting-ip|x-real-ip|x-forwarded-for|request\.cf|cookie|authorization|user-?id|x-forwarded"` over `src/` returns only the worker's *own outbound* additions, never reads:

```
src/index.ts:345  newHeaders.set("X-Forwarded-Host", originalHost);
src/index.ts:346  newHeaders.set("X-Forwarded-Proto", "https");
```

The only inbound header the worker ever *reads* is `Host` (not personal data):

```
src/index.ts:343  const originalHost = originalRequest.headers.get("Host") || "";
```

**Pass-through, not inspection.** The worker copies the *entire* incoming header set and the body verbatim into the forwarded request, but never parses, inspects, branches on, or stores their contents:

```
src/index.ts:342  const newHeaders = new Headers(originalRequest.headers);
src/index.ts:352  body: originalRequest.body,
```

So any client IP header, cookie, or auth token the client sent is *relayed* to the origin (Vercel / droplet) unchanged — the worker neither reads nor records it. Routing decisions key only on `url.hostname` and `url.pathname` (`src/index.ts:144-189`), never on identity.

**No observability config.** `wrangler.toml` has no `[observability]`, `logpush`, or `tail_consumers` block (full file read; `rg` confirms none). `.github/workflows/` (`ci.yml`, `deploy.yml`) contain no logpush/observability/analytics references.

**Caveat (platform-level, NOT FOUND in-repo):** Cloudflare's edge may inject `CF-Connecting-IP` on the forwarded request and may retain edge HTTP logs per account settings. Neither is configured or referenced in this repo — that is Cloudflare account/dashboard state, outside the worker's code. Marked **NOT FOUND in-repo**; do not assert worker behavior for it.

**Plain answer:** The router does not read, log, or store client IPs, user identifiers, headers, or cookies — it relays the full request to origin untouched and writes no logs; any edge-level IP logging is Cloudflare-platform config that does not appear in this repo.

---

## Q11(b) — Proxy scope

**Verdict: The worker handles exactly the apex, www, and API hosts per env; for traffic that reaches it, it forwards the full request (method + all headers + body) to origin, i.e. Cloudflare terminates TLS and the worker processes full request content in transit. WHICH traffic is attached to the worker (orange-cloud routes) is NOT FOUND in-repo — configured in the Cloudflare Dashboard, not `wrangler.toml`.**

### Evidence

**Hostnames handled** (`wrangler.toml` [vars]):

| Env | APEX_HOST | WWW_HOST | API_HOST |
| --- | --- | --- | --- |
| stage (`wrangler.toml:20-22`) | `stage.oglasino.com` | `""` (none) | `api-stage.oglasino.com` |
| production (`wrangler.toml:61-63`) | `oglasino.com` | `www.oglasino.com` | `api.oglasino.com` |

**Host dispatch** (`src/index.ts`):
- `www` → 301 redirect to apex, only when `WWW_HOST` is set (`:149-154`).
- `.well-known/apple-app-site-association` and `/.well-known/assetlinks.json` → served *directly* by the worker, never forwarded (`:161-178`).
- `isApi = host === env.API_HOST` (`:180`), `isFrontend = host === env.APEX_HOST` (`:181`).
- Any other host → `404 "Not found"` (`:183-185`).

**Full-content proxying for matched traffic** (`forwardToOrigin`, `src/index.ts:333-370`):

```
src/index.ts:349  const forwarded = new Request(targetUrl, {
src/index.ts:350    method: originalRequest.method,
src/index.ts:351    headers: newHeaders,         // all inbound headers
src/index.ts:352    body: originalRequest.body,  // full body
src/index.ts:353    redirect: "manual",
src/index.ts:356  const upstream = await fetch(forwarded);
```

Frontend forwards to `FRONTEND_ORIGIN` (Vercel), API forwards to `BACKEND_ORIGIN` (droplet) — `src/index.ts:277-292`. For a Cloudflare Worker on a Custom Domain this means TLS is terminated at the edge and the worker handles cleartext request content before re-forwarding.

**Route attachment is NOT in this repo.** `wrangler.toml:1-9` states routes are configured via *Cloudflare Dashboard → Triggers → Custom Domains*, NOT in the file; `rg "routes"` confirms no `routes`/`route` key exists in `wrangler.toml`. Therefore whether *all* web/API traffic is orange-clouded through the worker (vs. some paths bypassing it) **cannot be verified from repo source** → **NOT FOUND in-repo**. Project convention (conventions.md Part 8: "The Cloudflare router worker is the edge boundary … Backend liveness and frontend availability never bypass the worker") asserts full coverage, but that is a doc claim, not code evidence.

**Plain answer:** The worker proxies the apex, www, and API hostnames (stage and prod), forwarding each request's full method/headers/body to Vercel or the droplet with TLS terminated at Cloudflare's edge — but the actual route/orange-cloud attachment that determines whether 100% of traffic flows through it lives in the Cloudflare Dashboard, not this repo, so that completeness cannot be confirmed from source.

---

## Q11(c) — State (bindings)

**Verdict: One binding only — the `CONFIG` KV namespace, which the worker only READS, and only four boolean maintenance flags (no user data, no writes). No D1, no Durable Objects, no R2, no queues. ASSETLINKS/AASA are inline source constants (NOT bindings) holding only static, non-personal app-identity values.**

### Evidence

**Sole binding = `CONFIG` KV.** `rg -i "kv_namespaces|d1_database|durable_object|r2_bucket|queues|hyperdrive|KVNamespace|D1Database|DurableObject"` over `src/` + `wrangler.toml` returns *only*:

```
wrangler.toml:25  [[kv_namespaces]]            (default/stage, id 0b9a66b9…)
wrangler.toml:47  [[env.stage.kv_namespaces]]  (id 0b9a66b9…)
wrangler.toml:66  [[env.production.kv_namespaces]] (id dafb45ca…)
src/index.ts:54   CONFIG: KVNamespace;
```

No D1, Durable Object, R2, or queue binding exists anywhere.

**KV is read-only and stores only operational flags.** The worker calls `CONFIG.get` for exactly four keys and never `.put`/`.delete` (`rg "CONFIG\.get|CONFIG\.put"` → only gets):

```
src/index.ts:214  env.CONFIG.get("maintenance.web.active",  { cacheTtl: 30 }),
src/index.ts:215  env.CONFIG.get("maintenance.backend.active", { cacheTtl: 30 }),
src/index.ts:216  env.CONFIG.get("admin.bypass.disabled",   { cacheTtl: 30 }),
src/index.ts:217  env.CONFIG.get("use.backend.check",       { cacheTtl: 30 }),
```

Each is a `"true"`/`"false"` maintenance/feature flag (header comment `src/index.ts:7-12`). None is keyed by, or contains, user data.

**ASSETLINKS / AASA are constants, not bindings.** They are hardcoded `JSON.stringify` literals in source (`src/index.ts:85-136`), served directly. Contents are static, non-personal app-identity values: iOS app IDs + shared Team ID `44PHQVN8PB` (`:100, :109`), Android package names `com.oglasino` / `com.oglasino.preview` (`:121, :132`), and Android sha256 cert fingerprints — a tracked placeholder `REPLACE_AFTER_PLAY_CONSOLE_SETUP` for prod (`:122`) and the stage EAS-keystore fingerprint (`:133`). A cert fingerprint is a public app-signing identifier, not personal data.

**Plain answer:** The worker's only stateful binding is the read-only `CONFIG` KV namespace holding four boolean maintenance flags — no user data, no writes — and the app-association payloads are static, non-personal constants baked into the source, not a binding.

---

## Adjacent findings

1. **Prod Android assetlinks fingerprint is still a placeholder.** `src/index.ts:122` ships `sha256_cert_fingerprints: ["REPLACE_AFTER_PLAY_CONSOLE_SETUP"]` for `com.oglasino`. File path: `src/index.ts:122`. Severity: **medium** — not a privacy/legal issue (no personal data), but Android App Links verification for the production app will fail until the real Play Console fingerprint is registered. Out of audit scope; flagged only. *Not fixed — out of scope (read-only audit, and this is a tracked placeholder per the header comment at `src/index.ts:74-78`).*

2. **Route/orange-cloud completeness is unverifiable from this repo** (see Q11(b)). For the Privacy Policy's "all traffic proxied through Cloudflare" claim, the compliance reviewer needs the Cloudflare Dashboard Custom Domains config, which is not in `oglasino-router`. File path: `wrangler.toml:1-9` (documents the omission). Severity: **low** (documentation gap, not a defect). *Not fixed — out of scope and not a code issue.*
