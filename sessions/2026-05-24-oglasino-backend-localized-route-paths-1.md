# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-24
**Task:** Read-only audit of `page.*.url` translation keys — inventory, consumers, divergence from on-disk routes, and feasibility of "abandon" vs "finish" localized URLs

This is a read-only audit session. No code changes. No SQL edits.

## Implemented

- Read-only audit written to `.agent/audit-localized-route-paths.md` answering all five questions from the brief
- Cross-referenced backend translation seed SQL files (4 locale files) with web repo on-disk routes and consumer code

## Files touched

(none — read-only audit)

## Tests

N/A — read-only audit

## Cleanup performed

None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code changes
- Part 4a (simplicity): N/A — no complexity added or removed
- Part 4b (adjacent observations): one finding flagged in "For Mastermind" (the `page.catalog.url` copy-paste bug)
- Part 6 (translations): N/A this session — no translation keys added or modified
- Other parts touched: none

## Known gaps / TODOs

None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing
  - Considered and rejected: nothing
  - Simplified or removed: nothing

- **Five answers in concise form** (full detail in `.agent/audit-localized-route-paths.md`):

  **Q1:** 7 `page.*.url` keys exist in the `METADATA` namespace, seeded identically across all four locales (EN, RS, RU, CNR). The values are Serbian path segments (`/o-nama`, `/cenovnik`, `/uslovi`, `/politika-privatnosti`, `/dzabe-zona`) for 5 keys, `/user` for one, and a copy-paste duplicate `/dzabe-zona` for `page.catalog.url`. No adjacent `route.*` or `path.*` keys found.

  **Q2:** 6 of 7 keys diverge from their on-disk routes. The on-disk routes use English: `/about`, `/pricing`, `/terms`, `/privacy`, `/blog/free-zone`, `/catalog`. Only `page.user.url` (`/user`) matches. The divergence means every canonical URL and hreflang sibling emitted by the metadata generators points to a URL that 404s.

  **Q3:** 11 consumers across 11 files, all in `src/metadata/`. Five metadata generators use `t('page.X.url')` for canonical URL + hreflang pathBuilder. Five structured-data generators use it for JSON-LD `url`. One user-page metadata generator uses it for canonical only (hreflang hardcodes `/user/${user.id}`). `page.catalog.url` has zero consumers. No navigation, UI, or sitemap consumers exist.

  **Q4:** "Abandon localized URLs" is trivially realistic. 10 one-line replacements in web metadata files (swap `t('page.X.url')` for the hardcoded English on-disk path) + delete 28 SQL rows from backend seed files. No downstream consumer breaks. No sitemap, navigation, or UI component uses these keys. Estimated effort: 1-2 hours.

  **Q5:** "Finish localized URLs" is feasible but expensive and risky. next-intl supports a `pathnames` config in `defineRouting()`, but the project's routing is heavily customized (custom `proxy.ts` middleware, compound routing locales like `rs-sr`, custom navigation wrappers from the cookies-closing feature). Adopting `pathnames` would require 50+ locale-path mapping entries, reconciling proxy.ts with next-intl's pathname rewriting, reworking the navigation wrappers, and testing 100+ URL paths across 10 locales. Estimated effort: 1-2 weeks minimum, with high regression risk in recently-stabilized routing code.

- **Engineer's overall read:** Path 1 ("abandon") is cheaper by roughly 10-20x in engineering hours and carries near-zero regression risk. Path 2 ("finish") would reopen the routing surface that was stabilized across 3+ weeks of cookies-closing work and carries meaningful regression risk. The SEO value of localized URLs for a pre-launch site with zero Google authority is close to zero — localized URLs can always be added later (with 301 redirects from English paths) if post-launch analytics show a need.

- **Adjacent finding outside the question list:**
  - `page.catalog.url` has value `/dzabe-zona` — copy-paste of `page.free.zone.url`. Dead key with zero consumers. Inert bug. If Path 1 is chosen, delete it in the same pass. File: all four backend translation seed SQL files (see audit Q1 for line numbers). Severity: low (dead code).
