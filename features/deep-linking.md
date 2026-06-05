# Deep Linking (Universal Links / App Links)

**Status:** code-complete across repos, pending on-device verification (Ψ) and Android signing-cert fingerprints.
**Repos:** oglasino-router (serves association files), oglasino-expo (app-side). Backend N/A. Web deferred (consumer surfaces only).
**Date:** 2026-06-04

---

## 1. What this is

Shareable `https://oglasino.com/...` links that open the mobile app on the matching screen (iOS Universal Links, Android App Links) instead of the browser. Custom-scheme links (`oglasino://...`) already worked; this feature adds the verified-`https` kind — the links users actually share (SMS, email, social, "open in app").

Two distinct mechanisms hide under "deep linking":
- **Custom-scheme** (`oglasino://`) — app-private, already working, web not involved. Unchanged by this feature.
- **Universal/App Links** (`https://`) — OS-verified via association files hosted on the domain. The subject of this spec. A joint router + app effort.

## 2. Openable surface (v1)

Public paths only: **product, user, catalog**, plus static pages (about, pricing, privacy, terms, blog/free-zone). Secured routes (`/messages`, `/owner/*`) are **deferred** — they resolve mechanically but would land a logged-out user on home, and the post-login return-path UX is not built. Filter deep-linking applies to **home and catalog** (the only filtered surfaces).

## 3. Tiers & identifiers

Two tiers, every artifact parameterized:

| | Production | Preview |
|---|---|---|
| Host | `oglasino.com` | `stage.oglasino.com` |
| iOS appID | `44PHQVN8PB.com.oglasino` | `44PHQVN8PB.com.oglasino.preview` |
| Android package | `com.oglasino` | `com.oglasino.preview` |

Apple Team ID `44PHQVN8PB` is constant across tiers (identifies the Apple account; public, not a secret). Development tier has no universal links (custom-scheme only). Apex-only — no `www` (the router 301s www→apex; verifiers don't follow redirects).

## 4. Architecture

### 4.1 Association files — served by the router worker

`oglasino-router` (`src/index.ts`) serves both files directly, inline, short-circuited **before** the maintenance gate, origin forward, and KV reads, tier-correct per `env.ENVIRONMENT`:

- `GET /.well-known/apple-app-site-association` → 200, `application/json`, AASA body
- `GET /.well-known/assetlinks.json` → 200, `application/json`, assetlinks body

Not routed through `forwardToOrigin` (no origin redirect, no stage `X-Robots-Tag`). A documented exception to the maintenance matrix: these paths are served even during a maintenance window, because OS re-verification during a 503 can de-verify the domain association.

**Why router, not web:** origin-forwarding let a maintenance window 503 the files, passed origin redirects through (verifiers don't follow them), and left Next's dotfile/extensionless serving uncertain. Serving from the worker eliminates all three and is KV-outage-immune. (Ownership shift web→router; supersedes `seo-foundation.md` §11's "web serves the well-known files" assumption.)

AASA `components` use a `/*/` leading wildcard to match the locale segment (AASA globs can't enumerate the locale set; non-locale first segments aren't valid web URLs, so the over-claim is harmless). Patterns cover `/*/product/*`, `/*/user/*`, `/*/catalog`, `/*/catalog/*`, and the static pages.

### 4.2 App-side — `+native-intent` + boot Gate 3

**`app/+native-intent.ts` → `redirectSystemPath`** (pure, unit-tested): strips a leading compound-locale segment (shape `/^[a-z]{2,}-[a-z]{2,}$/i`, first-segment only) so web's locale-prefixed URLs resolve against the locale-less app route tree. It also **parses** the segment (`{baseSite}-{language}`, first-hyphen split) and **stashes** `(baseSite, language, path-with-query)` on a read-and-clear module-level side-channel. The locale travels beside the path, never in it. The query string is preserved verbatim (load-bearing for filters).

**Native config** (`app.config.ts`, tier-branched): `ios.associatedDomains` (`applinks:<host>`) and `android.intentFilters` (`autoVerify`, `https`, apex host). Inert until a prod/preview prebuild+build.

### 4.3 The compound locale — strict validation, base-site switch, link-language filter resolution

The locale is base-site + language, both load-bearing. Boot Gate 3 reads the side-channel and:

**Validates strictly.** Valid iff base-site exists in the available set AND language is in that base-site's `allowedLanguages`. **Invalid → the whole link is untrusted → normal boot on the stored site** (no switch, no filters, no crash). Rationale: a bad locale (e.g. `rs-cnr`, a near-miss between languages with different filter labels) poisons trust in which language the filter words are in. Overviews fetched only when the base-site differs (same-site links validate language against the already-loaded DTO). Any fetch failure → open normally on the stored site, never maintenance.

**Switches base-site** (when valid + differing). The entire data context — catalog, categories, filters, regions, cities, currencies — is one `BaseSiteDTO`; switching it (via the existing `pickBaseSite` primitive's load path, folded into Gate 3) cascades for free. The switch completes **before the portal mounts** (Design A) so the single feed fetch and mount-once filter hydration run once, against the switched site.

**Resolves filters in the link's language, displays in the user's preferred language.** Web emits filter/region/city slugs in the link's language; they only match if resolved in it. The app resolves them with a **fixed link-language translator** over a transient in-memory label load — without switching or persisting the active display language. Resolved filters are language-independent objects, so they survive with the display in the user's preferred language. **The user's preferred language is never changed or persisted-over.** A loading curtain (`status` stays `'booting'`) holds across the switch — no flash; the portal mounts once, switched, in the preferred language, filters applied.

### 4.4 Filter deep-linking (home + catalog)

A shared URL's filter query string populates and applies filters on open. Mobile mirrors web's **SSR** `parseFiltersFromQueryParams` (slug-vs-slug via `toQueryParam`, accent-robust) — explicitly NOT web's lossy client-store hydrate. Hydration is synchronous on first render (a `useState` initializer) so the list's mount-once load reads populated state → a single fetch. **Strict on the locale, lenient on filters:** within a valid link, individual filter values that don't resolve are skipped (not errors); apply what resolves.

**Filter URL contract** (what web emits; mobile parses against it; values percent-encoded on the wire — decode before splitting): `search_text` (slug), `f_<filterKey>` (OPTION: comma-joined option slugs; RANGE/DATE: `from:<prefix><n><suffix>`/`to:...` comma-joined), `from`/`to` (raw numeric), `free` (`true`), `currency` (code), `order` (code), `regions`/`cities` (comma-joined label slugs). Category travels in the path, not the query. `randomSeed` is not in the URL (each open reseeds).

## 5. Critical path & current state

| Piece | Repo | State |
|---|---|---|
| `.well-known` direct-serve, tier-correct | router | code-complete; both Android SHAs are placeholders |
| `+native-intent` (strip + parse + stash) | expo | code-complete, tested |
| native config (associatedDomains/intentFilters) | expo | code-complete, inert until prebuild+build |
| filter hydration (home + catalog) | expo | code-complete, tested |
| base-site switch + link-language resolve (Gate 3) | expo | code-complete, tested |
| backend | backend | N/A (default-deny would 403 `.well-known`; confirmed) |
| web consumer surfaces (badges, open-in-app) | web | deferred until app is published |

**Remaining (owner: Igor — not agent work):**
1. Generate preview Android keystore → swap stage SHA placeholder.
2. Play Console setup → swap prod SHA placeholder (use Google's Play-signing key; list upload + signing keys).
3. Prebuild + build prod and preview (on-disk build is dev-tier).
4. Commit the `new-expo-dev` work (four stacked uncommitted sessions).
5. On-device Ψ pass (§6).
6. Web confirmation: web always emits base-site half as a real code, language half within that site's allowed set, filter slugs in the link's language.

## 6. On-device verification (Ψ)

Cannot be exercised in the node/vitest harness:
1. Verified `https://<apex>/{locale}/...` opens the app; `+native-intent` strips the locale and routes correctly. (AASA CDN caching — budget propagation time / reinstall during testing.)
2. Side-channel ordering: `redirectSystemPath` for the initial URL runs before Gate 3's stored-site read. Cold-start an `me-*` link on an `rs`-stored device → switch happens.
3. **i18n resource-bundle survival (highest-risk):** `ensureLanguageLabelsLoaded`'s `addResourceBundle(linkLang)` survives boot i18n init. A cross-language link's filters must actually resolve on device — if the bundle was wiped, filters silently won't match. Test a real `me-en` link on an `rs`-stored device and confirm filters populate.
4. Single feed fetch on a switching deep-link cold-start.
5. No transient-language flash; preferred language intact (persisted preference unchanged) after a cross-language link.
6. Android verification on both tiers once real SHA-256 fingerprints are in place.

## 7. Decisions of record

See [decisions.md](../decisions.md) (2026-06-04 — "Deep Linking (Universal/App Links): product & architecture decisions"): public-only paths; apex-only; two-tier; router-serves-`.well-known` (ownership shift); compound locale drives strict-validated base-site switch with link-language filter resolution that never persists over the preferred language (supersedes the earlier "discard the locale" position); no display-language auto-switch; strict-on-locale / lenient-on-filters.

## 8. Related issues

See [issues.md](../issues.md) (2026-06-04): prod Android SHA swap (after Play Console); preview Android SHA swap (after keystore); in-app `PortalConfigDialog` feed-stale latent bug; deferred web consumer surfaces.

## 9. Session log

- **2026-06-04** — Phase 4 canonical spec authored (Docs/QA, from Mastermind draft). Feature is code-complete across router + expo on `new-expo-dev` (uncommitted), pending the prod/preview build, the two Android SHA-256 swaps, and the on-device Ψ pass. Paired `decisions.md` (one entry) and `issues.md` (four entries) applied the same session; `state.md` active-feature block + Expo-backlog row added; `seo-foundation.md` §11 reconciled for the web→router ownership shift.
