# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-04
**Task:** READ-ONLY audit — Deep Linking (Universal/App Links: URL shape, hosting, consumer surfaces). Output `.agent/audit-deep-links.md`.

## Implemented

- Read-only audit, no code changes. Answered all 8 brief questions against source on disk and wrote findings to `.agent/audit-deep-links.md`.
- Confirmed the canonical product share URL shape `https://oglasino.com/{locale}/product/{id}/{slug}` (single builder `getNormalizedProductUrl`, apex hardcoded), and the user (`/{locale}/user/{id}`, no slug) and catalog (`/{locale}/catalog[/slug…]`) canonical shapes.
- Established the 10-locale routing set is a hand-maintained constant duplicated in `routing.ts` and `localeMapping.ts`, also runtime-derivable from base-site DTOs; locale segment is mandatory and invalid values `notFound()`.
- Verified `appDeepLink.ts` targets the **custom scheme** `oglasino://` (disabled), `.well-known` is unserved today, `proxy.ts`'s matcher excludes dotted paths (so it won't redirect `.well-known`), the footer app-store badges are link-less SVGs, and everything web emits uses apex (not www).

## Files touched

- `.agent/audit-deep-links.md` (new, audit output)
- `.agent/2026-06-04-oglasino-web-deep-links-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten copy)

No source/config files edited.

## Tests

- None run — read-only audit, no code paths changed. (lint/tsc/test N/A; nothing to verify.)
- New tests added: none.

## Cleanup performed

- none needed (no code touched).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (four adjacent observations flagged in "For Mastermind" as candidates Mastermind may route into issues.md; not drafted here)

## Obsoleted by this session

- nothing

## Conventions check

- Part 4 (cleanliness): confirmed — no files edited beyond `.agent/` audit + summary outputs.
- Part 4a (simplicity): see structured evidence in "For Mastermind" (nothing added — audit only).
- Part 4b (adjacent observations): four flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (apex/www, worker as edge boundary) — informed the hosting recommendation; Part 11 (trust boundaries) — confirmed N/A for public share URLs.

## Known gaps / TODOs

- `base.url` METADATA translation value is backend-seeded and not visible in this repo; the audit assumes it equals `https://oglasino.com` and flags the divergence risk if it does not. Confirmation owed by Igor/Backend.
- The dotfile/extensionless `Content-Type` behaviour of Next 16.2.6 serving `public/.well-known/apple-app-site-association` was reasoned about, not empirically tested (read-only). The audit recommends a route handler or the worker to avoid the question entirely.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Hosting recommendation:** serve `apple-app-site-association` + `assetlinks.json` from the **Cloudflare router worker** (it fronts every request, can set exact `application/json`, and sidesteps Next's extensionless/dotfile uncertainties and apex-vs-www serving). If web must serve them, use a Route Handler (`app/.well-known/…/route.ts`), not a `public/` static file. Cross-repo (`oglasino-router`) decision — flagged, not actioned.
- **Reconcile with scaffold:** `appDeepLink.ts` is custom-scheme (`oglasino://`) and disabled; Universal/App Links are `https://`/OS-claimed. Recommend **extending** `appDeepLink.ts` as the single deep-link home rather than adding a parallel util.
- **Adjacent flags (Part 4b), all out-of-scope, not fixed:**
  1. (medium) Host-source divergence — product/robots/sitemap hardcode `https://oglasino.com`; user/catalog derive host from `base.url` translation (`generateUserPageMetadata.ts:28`, `generateCatalogPageMetadata.ts:38`). Silent canonical-host split if `base.url` ≠ apex.
  2. (medium) Locale-set duplicated in `routing.ts:5-15` and `localeMapping.ts:14-25` with no parity test — desync risk when a locale is added.
  3. (low) `NEXT_PUBLIC_SITE_URL` env var is unused (zero refs in src/app) — misleads anyone hunting the canonical host.
  4. (low) `getNormalizedProductUrl(id,name,true,undefined)` would render `https://oglasino.com/undefined/product/…`; no live call site does this, but deep-link work adds new call sites.
- **Config-file dependency:** none. No edit to the four config files is required by this session; the observations above are candidates Mastermind may choose to log, not a pending draft owed by me.
