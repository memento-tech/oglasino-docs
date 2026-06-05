# Read-only audit — web Privacy / Terms link + render pattern

**Repo:** oglasino-web
**Branch:** `dev` (unchanged; nothing switched, committed, or pushed)
**Date:** 2026-06-01
**Type:** READ-ONLY investigation. No code changed.
**Purpose:** Establish exactly how web sources, links to, and renders the Privacy Policy and Terms of Use pages, so the mobile (`oglasino-expo`) stale-link fix can target the correct destination (mobile Ψ batch, issues.md 2026-05-31).

---

## TL;DR

- The public, canonical URLs are **`https://oglasino.com/{tenant}-{lang}/privacy`** and **`https://oglasino.com/{tenant}-{lang}/terms`** — the locale segment is a single combined `tenant-lang` slug (e.g. `rs-sr`, `me-en`), not a bare language code.
- Both pages render **remote English-only markdown** fetched at request time from GitHub raw URLs under `memento-tech/oglasino-platform` (`privacy-policy.en.md` / `terms-of-use.en.md`). The `.en.md` suffix is hardcoded; content is **not** locale-aware yet (blocked on lawyer review — issues.md 2026-05-27).
- Web links to them from the **footer only**, via a locale-prefixing `<Link>`, using bare app-relative routes `/privacy` and `/terms`.

---

## Q1 — Route / URL for each page

Both pages live under the public portal route group, behind the `[locale]` segment:

- **Privacy:** `app/[locale]/(portal)/(public)/privacy/page.tsx` — `PrivacyPage` (default export, async RSC).
- **Terms:** `app/[locale]/(portal)/(public)/terms/page.tsx` — `TermsPage` (default export, async RSC).

The `(portal)` and `(public)` segments are Next.js route groups (parenthesised → no URL contribution). The only URL-bearing segments are `[locale]`, then `privacy` / `terms`. So the on-site paths are `/{locale}/privacy` and `/{locale}/terms`.

## Q2 — Content source

Each page fetches **remote markdown from a GitHub raw URL** and renders it through a shared `MarkdownViewer` server component. Content is not bundled and not stored in the repo.

- Privacy URL (`app/[locale]/(portal)/(public)/privacy/page.tsx:10-11`):
  ```
  https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/privacy-policy.en.md
  ```
  Passed to `<MarkdownViewer url={PRIVACY_MARKDOWN_URL} />` at `privacy/page.tsx:22`.

- Terms URL (`app/[locale]/(portal)/(public)/terms/page.tsx:10-11`):
  ```
  https://raw.githubusercontent.com/memento-tech/oglasino-platform/refs/heads/main/terms-of-use.en.md
  ```
  Passed to `<MarkdownViewer url={TERMS_MARKDOWN_URL} />` at `terms/page.tsx:22`.

- Fetch + render mechanics (`src/components/server/MarkdownViewer.tsx:7-31`): `fetch(url, { cache: 'force-cache', next: { revalidate: 3600 } })` (ISR, 1-hour revalidate), text rendered via `ReactMarkdown` with `remarkGfm` + `rehypeRaw`. On any non-ok response or throw it renders a localized error string (`ERRORS` namespace → `markdown.field.load`, `MarkdownViewer.tsx:18,22`) — it does **not** fall back to alternate content.

The page body itself is otherwise empty save for an `sr-only` `<h1>` pulled from `METADATA` (`privacy.title` / `terms.title`) — consistent with the SEO-foundation note that JSON-LD was removed from privacy/terms and both render as "now empty" SEO-wise.

## Q3 — Locale handling

**Content is hardcoded to English. It is NOT locale-aware.** The `.en.md` suffix is a literal in both URL constants. An identical interim comment sits above each page component (`privacy/page.tsx:13-15`, `terms/page.tsx:13-15`):

```
// Hardcoded English: per-locale legal content is pending lawyer review.
// See issues.md 2026-05-27 — "Per-locale legal markdown content (privacy + terms)"
// for the swap-to-locale-aware task.
```

The English-only state is also reflected in metadata: non-EN locales are flagged `robots: { index: false, follow: true }` so search engines don't index English legal text under SR/RU/CNR URLs (`src/metadata/generatePrivacyPageMetadata.ts:13,21-23`, and the mirror `generateTermsPageMetadata.ts`). Only `rs-en` / `rsmoto-en` / `me-en` are left indexable.

Per the brief's out-of-scope note: the per-locale legal-content state is already tracked in **issues.md 2026-05-27** ("Per-locale legal markdown content (privacy + terms)", status `open`, blocked on lawyer review of Serbian translations). That entry also records the intended future swap shape and the required `privacy-policy.sr.md` / `terms-of-use.sr.md` files. Not investigated further here.

## Q4 — Canonical public-facing URL

The locale segment is a **single combined `tenant-lang` slug**, confirmed from the routing config (`src/i18n/routing.ts:4-17`):

```
locales: rs-sr, rs-en, rs-ru, rsmoto-sr, rsmoto-en, rsmoto-ru, me-sr, me-cnr, me-en, me-ru
defaultLocale: 'rs-sr'
localePrefix: 'always'   // every URL carries the locale segment
localeDetection: false
localeCookie: false
```

So the canonical reader-facing URLs are:

```
https://oglasino.com/{tenant}-{lang}/privacy
https://oglasino.com/{tenant}-{lang}/terms
```

Concrete examples: `https://oglasino.com/rs-sr/privacy`, `https://oglasino.com/me-en/terms`. Because `localePrefix: 'always'`, there is **no** unprefixed `/privacy` URL on the live site — the prefix is mandatory. The default tenant-locale is `rs-sr`.

The canonical URL is built in metadata as `` `${basicMetadata.baseUrl}/${locale}${pagePath}` `` with `pagePath = '/privacy'` (`src/metadata/generatePrivacyPageMetadata.ts:12-13`; terms mirror). So the host is sourced from `basicMetadata.baseUrl` (via `getBasicMetada`), not hardcoded in the page files.

> Caveat on host: I confirmed the path shape and locale segment from the routing config and the metadata builder. The apex host `oglasino.com` is the project's known production domain (per state.md / SEO foundation references); the live origin string resolves from `basicMetadata.baseUrl` (`generalMetadata.ts`), which I did not open this session. If mobile needs the exact origin, confirm `baseUrl` there — but the **path** portion (`/{tenant}-{lang}/privacy` | `/terms`) is authoritative from the code above. The `tenant-lang` slug shape is also enforced at the edge by `proxy.ts` (the cookie-wins-over-URL locale authority).

## Q5 — Footer / link surfaces

Web links to these pages from **one surface: the footer.** No cookie-consent-banner or settings link to privacy/terms was found.

- Link data (`src/lib/data/companyNavigations.tsx:5-21`): a `companyNavigations` array with three `kind: 'link'` items — `about.label → /about`, `privacy.label → /privacy`, `terms.label → /terms`. The `route` values are **bare app-relative paths** (`/privacy`, `/terms`), with no locale segment baked in.
- Render (`src/components/server/layout/Footer.tsx:17-25`): the footer maps `companyNavigations` and renders each `kind: 'link'` via the locale-aware `<Link href={item.route}>` imported from `@/src/i18n/navigation`. Labels resolve from the `FOOTER` namespace.
- Locale prefixing (`src/i18n/navigation-client.tsx:12-25`): the custom `Link` prepends the active routing locale to any `/`-leading href via `prefixWithLocale(href, locale)` → `/${locale}${href}`. So footer `/privacy` becomes `/{currentLocale}/privacy` at render (e.g. `/rs-sr/privacy`).

**Net href shape the user clicks:** an internal, locale-prefixed app path — `/{tenant}-{lang}/privacy` and `/{tenant}-{lang}/terms` — which is exactly the canonical public URL path from Q4.

---

## Notes for the mobile-target decision (Igor's call, not mine)

- The "current correct target" for a mobile link to web's Privacy/Terms is `https://{web-origin}/{tenant-lang}/privacy` (or `/terms`), with `{tenant-lang}` matching the user's selected base site + language — e.g. `rs-sr`. There is no unprefixed path; the locale segment is mandatory.
- The *content* mobile would land on is English-only today regardless of the locale segment chosen (Q3). That's a platform-wide state, not a web-link bug.
- I did not inspect what the mobile app currently points at — that's the separate expo brief.

---

## Conventions check

- This was a read-only audit; no code, tests, or config touched. Part 4 (cleanliness): N/A — nothing written to source. Part 4a / 4b: N/A. Part 6 (translations): N/A.
- Config-file impact: **none required.** No new finding beyond what issues.md 2026-05-27 already records; I did not draft any edit to `conventions.md`, `decisions.md`, `state.md`, or `issues.md`.

## For Mastermind

- Nothing flagged beyond the report. The English-only legal-content state is already in issues.md 2026-05-27 (open); no new issue is warranted.
- One small thing worth confirming before mobile wires a link: the exact production **origin** (apex `oglasino.com` vs a tenant subdomain). The path/locale shape is authoritative from code; the host string is not constant-defined in these page files.
