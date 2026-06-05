# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-04
**Task:** Brief 3 — backend-security-hardening: close the JsonLd `<script>`-context escape (stored XSS) by making `JsonLd` escape the serialized JSON-LD as its single output path. Gates Brief 4 (backend filter removal).

## Implemented

- **Fixed the sink (Option B — escape inside `JsonLd`).** Added `serializeJsonLd(item)` in `src/components/server/seo/JsonLd.tsx`, the single output path for every caller. It runs `JSON.stringify` then escapes all three script-context-significant characters to JSON unicode escapes: `<`→`<`, `>`→`>`, `&`→`&`. The `dangerouslySetInnerHTML` now consumes `serializeJsonLd(item)` instead of raw `JSON.stringify(item)`. A backend string containing `</script><script>…</script>` can no longer break out of the JSON-LD script element.
- **Chose Option B over Option A deliberately.** Investigated whether the framework offers a delivery that auto-escapes the script context. It does not: neither `next/script` with `type="application/ld+json"` nor rendering the JSON as a React text child escapes `</script>` — Next.js's own JSON-LD guidance uses exactly this `dangerouslySetInnerHTML` + manual `.replace(/</g,'<')` pattern. So Option A offered nothing cleaner; Option B is the framework-blessed, bypass-proof fix. (See "Brief vs reality" re: the actual Next version.)
- **Escaped once, inside the component, on the final serialized string** — not at call sites. `JsonLd` maps over an array (or wraps a single object into one), so each item is serialized-and-escaped independently. Any future page that uses `JsonLd` is safe regardless of what data it passes.
- **Verified with the real attack string** via a new test file: a render-based test (real `<script>` HTML via `react-dom/server`, no new deps) plus a pure-serializer test. Asserts no literal `</script>` survives in the script body, the escaped form is present, and the body still `JSON.parse`s back to the exact input (SEO/structured data intact).

## Files touched

- src/components/server/seo/JsonLd.tsx (+15 / -2)
- src/components/server/seo/JsonLd.test.ts (+109 / -0, new)

## Tests

- Ran: `npx vitest run src/components/server/seo/JsonLd.test.ts`
- Result: 6 passed, 0 failed
- New tests added: `JsonLd.test.ts` — `serializeJsonLd` (breakout payload escaped; bare `<`/`>`/`&` escaped; valid-JSON round-trip) and `JsonLd` rendered output (breakout payload produces no literal `</script>` in body; array items escaped independently — exactly 2 real `</script>` closers; real product-feeder shape with `name`+`description` containing `<` serializes safely).
- `npx tsc --noEmit`: clean.
- `npx eslint` on both touched files: 0 errors, 1 warning — pre-existing (see Conventions check / For Mastermind).
- `npx prettier --check` on both files: clean.

## Cleanup performed

- none needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change. (Mastermind/Docs/QA may want to note the gate is cleared so Brief 4 can run — that is sequencing, drafted in "For Mastermind", not a config edit I make.)
- issues.md: no change written by me. The high/open `2026-06-03 — Web: JSON-LD <script> injection` entry is now addressed in code; the resolution note is DRAFTED in "For Mastermind" per the brief (flip is batched for feature close, not written here).

## Obsoleted by this session

- Nothing. No code was made dead. The raw `JSON.stringify(item)` injection was replaced in place by `serializeJsonLd(item)`; no other code referenced the old inline form. The three feeder modules (`generateProductPageStructuredData.ts`, `generateUserPageStructuredData.ts`, `generateCatalogPageStructuredData.ts`) are deliberately unchanged — the fix is at the sink, not the call sites.

## Conventions check

- Part 4 (cleanliness): confirmed. No commented-out code, no debug logging, no unused symbols added. One pre-existing lint warning left in place and flagged (Part 4b).
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session — no translation keys touched.
- Other parts touched: Part 11 (trust boundaries) — confirmed: the fix treats all `JsonLd` input as untrusted and encodes it at the output boundary (the script context), which is the correct place; it does not change what data the three pages pass. Part 8 ("errors are codes" / output encoding) — consistent.

## Known gaps / TODOs

- none. The "Smoke (after ship)" steps in the brief (stage product/user/catalog with a `</script>…` payload, view-source for `<`, JSON-LD validator) are owed to Igor post-ship — they require a running stage environment and are not code.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): `serializeJsonLd` — one small exported pure function. Earns its place: it is the single bypass-proof output path the brief requires (so no future caller can reintroduce the XSS) and it is the unit under test. One concrete caller today (`JsonLd`); exported so the security property can be unit-tested directly without a DOM.
  - Considered and rejected: (1) Option A / a framework `<Script>`-based delivery — rejected because no Next/React path auto-escapes `</script>` (verified), so it would add indirection without removing the manual-escape need. (2) Adding `@testing-library/react` + `jsdom` for the component test — rejected; used `react-dom/server.renderToStaticMarkup` + `React.createElement`, which renders the real component in the existing node/vitest environment with zero new dependencies and no JSX-transform-in-test config. (3) Escaping at each call site — rejected; the brief's whole point is sink-level safety.
  - Simplified or removed: nothing.
- **Brief vs reality (non-blocking — did NOT stop work, the fix is unaffected):**
  1. **Next/React version.** My CLAUDE.md and `conventions.md` Part 9 say "Next.js 15 / React 18." `package.json` is `next ^16.2.6` / `react ^19.2.6`. This does not change the fix (Option B is correct on both, and the newer version still has no auto-escaping script delivery). Flagging only so the stack reference can be reconciled by whoever owns it.
  2. **"Seven pages" count.** The audit and brief say `JsonLd` is rendered on "seven pages." Independent grep finds **8 call sites**: `app/page.tsx` (intro), `(public)/page.tsx` (home), catalog, user, product, free-zone, about, pricing. The three raw-content feeders (product/user/catalog) are exactly as the audit listed and verified at the cited lines; the other 5 are translation-key-fed (trusted). The miscount is immaterial because the fix is at the sink and covers all callers, present and future. No new raw-content caller has appeared since the audit.
  3. **No second script-context sink.** Re-confirmed: repo-wide grep for `dangerouslySetInnerHTML` / `application/ld+json` / `<script` finds only `JsonLd` and the two trusted cookie/theme snippets in `app/layout.tsx` (sanitized/allowlisted per the audit). `MarkdownViewer` untouched (out of scope, operator-controlled). The `<script>` hit in `linkify.test.ts` is test data.
- **Part 4b adjacent observation (1, low):**
  - `src/components/server/seo/JsonLd.tsx:26` carries `// eslint-disable-next-line react/no-danger`, which ESLint now reports as an **unused disable directive** (the active config does not report `react/no-danger`). This warning is **pre-existing** — the same directive was on line 13 of `HEAD` before my change; I only moved it down as the file grew. Severity low (a warning, not an error; lint still passes). I left it rather than deleting it, because `eslint-config-next` can enable `react/no-danger` under stricter configs and removing it could surface an error in CI. Recommend a deliberate decision (remove the directive, or keep it and accept the warning) rather than a silent delete. Out of scope for this brief.
- **DRAFT — issues.md resolution note (do NOT let me write it; for Docs/QA, batched for feature close per the brief):**
  > **Update to `2026-06-03 — Web: JSON-LD `<script>` injection` (high):** Status `open` → `fixed`. Resolved in `oglasino-web` `dev` 2026-06-04 (session `oglasino-web-backend-security-hardening-1`). `src/components/server/seo/JsonLd.tsx` now escapes the serialized JSON-LD for the HTML script context as its single output path (`serializeJsonLd`: `<`→`<`, `>`→`>`, `&`→`&`), so a `</script>…` payload in any backend/user field can no longer break out of the JSON-LD `<script>`. Output remains valid, parseable JSON-LD (round-trip test). Covers all `JsonLd` callers (product/user/catalog raw feeders + the trusted ones). Verified by `JsonLd.test.ts` (6 tests, render + serializer, real attack string). This **clears the gate on Brief 4** (backend `InputSanitizationFilter` removal).
- **Sequencing note (per the brief):** Part 3 verification is **green** — the breakout is proven closed against the literal `</script><script>alert(1)</script>` payload in real rendered HTML, and structured data round-trips. Part 1's "other caller" sweep found no caller the audit missed (8 call sites, same 3 raw feeders, no new sink). Per the brief, Brief 4 (backend filter removal) is gated on this being **confirmed in** — code is on disk on `dev`, not committed (engineer agents do not commit). On-stage smoke (the brief's "Smoke (after ship)" steps) remains owed to Igor before/with the prod rollout, but the web side is now safe on its own.
