# Session Summary — oglasino-router — deep-links (1)

**Date:** 2026-06-04 · **Branch:** stage (unchanged) · **Type:** READ-ONLY audit

## Task
Audit how the router worker handles `/.well-known/apple-app-site-association` and `/.well-known/assetlinks.json` for the upcoming iOS Universal Links / Android App Links feature, and assess whether the worker should serve those files directly. Findings written to `.agent/audit-deep-links.md`.

## What I did
- Read `src/index.ts` in full (274 lines) and `wrangler.toml` in full; grepped `src/`, `tests/`, `wrangler.toml` for any existing `.well-known`/AASA/assetlinks handling (none exists).
- Traced the full request flow for a `.well-known` GET against production env vars.
- Answered all 9 brief questions with file:line evidence in `.agent/audit-deep-links.md`.

## Key findings
- **No `.well-known` handling today** — such a path is forwarded to `FRONTEND_ORIGIN` (Vercel) like any frontend path.
- **HIGH risk — maintenance gate 503s the files** (`src/index.ts:172-178`): `.well-known` is not classified as admin infrastructure, so it's blocked during web/backend maintenance regardless of the admin bypass.
- **HIGH risk — www→apex 301** (`src/index.ts:76-81`) fires for every path including `.well-known`; verifiers don't follow redirects, so a www-host fetch would fail verification.
- **Production adds no headers** on the forward path (early return at `:272`); cannot break `Content-Type: application/json`. Stage adds only `X-Robots-Tag` (cosmetic; prod-only feature anyway).
- **Admin regex does not match** `/.well-known/*` (confirmed) — which is *why* it gets caught by maintenance.
- **Recommendation: serve both files directly from the worker** via a static short-circuit placed *before* the KV reads, maintenance gate, and origin forward (and, if www must serve them, before the www 301). Gives maintenance/redirect/KV-outage immunity with zero new dependencies.
- This worker fronts **both stage and production**; only production fronts `oglasino.com` / `www.oglasino.com`, so the feature is production-only.

## Cleanup performed
None — read-only audit, no code touched.

## Obsoleted by this session
None.

## Known gaps
- Whether the app's Associated Domains / Android intent filters will include `www.oglasino.com` (apex-only vs apex+www) determines the short-circuit insertion point. Cross-repo decision; not resolved here.
- Whether `oglasino-web` (Vercel) has any redirect rule that would catch `/.well-known/...` was not verified (cross-repo).

## Conventions check
Read-only respected. No edits to `src/`, `tests/`, `wrangler.toml`, `package.json`, `tsconfig.json`, or any docs/config file. No `wrangler` commands, no KV access, no deploys. No commented-out code, no TODOs, no new unreferenced source files. The only files written are `.agent/audit-deep-links.md` (per brief) and this summary.

## Config-file impact
No config file edited. **If** worker-serves-directly is adopted, Docs/QA would need to: (a) reconcile `oglasino-docs/features/seo-foundation.md` §11.14 (currently assumes web serves the files and the router merely "exempts" them); (b) record the web→router ownership shift in `decisions.md`; (c) note that the maintenance-matrix comment block in `src/index.ts:1-43` would gain a documented `.well-known` exemption. Draft pointers only — see "For Mastermind" in the audit.

## For Mastermind
Full detail in `.agent/audit-deep-links.md` ("For Mastermind" section). Headlines:
1. Recommend **worker-serves-directly** for both association files.
2. Two HIGH risks under current code: maintenance 503s (`:172-178`) and www 301 (`:76-81`).
3. Cross-repo: `seo-foundation.md` currently assigns `.well-known` serving to web and assumes the router stays out of the way — but today it does not (maintenance gate + www 301 + pass-through of origin 3xx). Ownership decision needed. Not acted on (read-only, cross-repo).

### Brief vs reality
No contradictions. The brief's framing ("the worker is exactly the layer that can silently break that") is confirmed accurate by the code: the maintenance gate and the www 301 are concrete silent-break paths.
