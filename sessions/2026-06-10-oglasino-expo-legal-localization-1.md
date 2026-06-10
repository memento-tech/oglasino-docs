# Session — Legal localization audit (oglasino-expo)

**Date:** 2026-06-10
**Slug:** legal-localization · session 1
**Branch:** `dev`
**Type:** READ-ONLY audit. No code changed, nothing installed, no commit.

## Task

Read-only audit for the legal-localization feature (mobile side), ahead of the spec. Locate the two in-app legal screens and their hardcoded GitHub raw markdown URLs, identify the active-language source available to those screens, confirm the `.en.md` URLs match the expected filenames on `refs/heads/main`, characterize the fetch/caching behavior, and confirm the trust boundary. Findings written to `.agent/audit-legal-localization.md`.

## What I did

- Read the two legal screens (`app/(portal)/(public)/privacy.tsx`, `app/(portal)/(public)/terms.tsx`), the shared `src/components/MarkdownViewer.tsx`, `src/lib/store/bootStore.ts`, `src/lib/types/catalog/LanguageDTO.ts`, and `src/lib/navigation/deepLinkLocale.ts`.
- Verified every file:line claim with both Read and an independent `rg`.
- Wrote the full audit to `.agent/audit-legal-localization.md`.

## Key findings (summary; full detail in the audit file)

1. **Two screens only.** Privacy = `app/(portal)/(public)/privacy.tsx:12`; Terms = `app/(portal)/(public)/terms.tsx:12`. Each passes a fully-inlined hardcoded URL to `MarkdownViewer` — no shared constant/helper.
   - `…/refs/heads/main/privacy-policy.en.md`
   - `…/refs/heads/main/terms-of-use.en.md`
2. **Active language** = `useBootStore((s) => s.language)?.code`, a bare lowercase code (`'sr'`/`'cnr'`/`'en'`/`'ru'`), `LanguageDTO | null`, not compound. `cnr` is a real distinct code (Montenegrin, `me`-only), never collapsed to `sr` — so the brief's `sr`/`cnr` grouping is correct and necessary. Neither legal screen reads the language today (both are static, no `bootStore` import).
3. **URLs match exactly** on `refs/heads/main` (code-string confirmation; did not live-fetch the likely-private repo).
4. **Fetch at render, no caching.** `MarkdownViewer` does `fetch(url)` in a `useEffect` keyed on `[url]`; no AsyncStorage/checksum/memo. The `[url]` key means a language-derived URL re-fetches automatically on switch.
5. **Trust boundary: none** — public static anonymous GitHub raw content, no auth/credentials/user input, read-only render.

## Brief vs reality

No correctness conflicts — the brief matches the code on every checkable point. Two non-blocking implementer notes recorded in the audit: (a) neither screen reads the language today, so the `.sr.md` branch is new wiring per screen (or a shared helper); (b) `language` is nullable — guard so null/unknown falls to the `.en.md` default.

## Cleanup performed

None needed — read-only audit, no code touched.

## Obsoleted by this session

Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed, no debug logging, no TODOs added.
- Part 4a (simplicity) / Part 4b (adjacent observations): N/A — audit only; observations captured in the audit file's "Brief vs reality."
- Part 6 (translations): N/A this session — no translation keys added or touched. (Note: the legal markdown is fetched from GitHub, not the i18n namespace system, so the feature itself adds no backend translation keys.)
- Other parts touched: none.

## Config-file impact

No config-file edit needed. This is a pre-spec audit; there is no `features/legal-localization.md` and no Expo backlog row for it yet, so nothing in `state.md`/`issues.md`/`decisions.md`/`conventions.md` requires updating from this session. **For Mastermind:** if/when this feature is specced, the brief's mapping (`sr`/`cnr` → `.sr.md`, else `.en.md`) is confirmed implementable against the current mobile code — the only new work is wiring the active language into the two otherwise-static legal screens.

## For Mastermind

- The two legal screens are static and language-unaware; adopting the feature means each screen (or a new shared helper) must read `useBootStore((s) => s.language)?.code` and select `.sr.md` vs `.en.md`. Recommend a single small helper (e.g. `legalMarkdownUrl(doc, langCode)`) over duplicating the branch in both screens.
- Implementer must confirm `privacy-policy.sr.md` and `terms-of-use.sr.md` exist on `refs/heads/main` of `memento-tech/oglasino-platform` before shipping — this audit confirmed only the existing `.en.md` code strings, not live `.sr.md` file presence.
- No backend/web contract dependency: this is pure client-side URL selection against static GitHub content. Per the waiting rule, mobile can proceed whenever the spec lands (no `web-stable` gate, since there's no backend wire shape involved).
