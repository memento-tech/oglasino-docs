# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-03
**Task:** backend-security-hardening web output-encoding check — confirm the web client encodes backend-supplied strings at render so the backend can stop HTML-encoding input. READ-ONLY audit.

## Implemented

- Read-only audit only; no code changed. Produced `.agent/audit-web-output-encoding.md` (Parts 1–3 + bottom line).
- Grepped the repo for `dangerouslySetInnerHTML` (3 hits) and the `rehype-raw` markdown surface (1), classified each as backend/user-fed vs trusted, and `view`-read every cited `file:line` to cross-check the grep (Risk Watch rule).
- **Loud finding:** `JsonLd` (`src/components/server/seo/JsonLd.tsx:14`) injects structured data into a `<script type="application/ld+json">` via `JSON.stringify`, which does NOT escape `</script>`/`<`/`>`. Product/user/catalog pages feed it raw `product.name`, `product.description`, `user.displayName`, `product.title`. This is a stored-XSS sink whose safety currently depends on the backend HTML-encoding those fields — the exact filter the feature wants to remove.
- Confirmed the safe surfaces: layout consent + theme `<script>` snippets are cookie-derived and sanitized/allowlisted; the messaging auto-linkify escapes the body and constructs only `<a>` wrappers with scheme-restricted hrefs; the default JSX text path (product name/description, displayName, message body) escapes by default.
- Confirmed `MarkdownViewer` uses `rehype-raw` with NO sanitizer, but its source is operator-controlled GitHub markdown (privacy/terms), never touches the backend — fragile but independent of the backend question.

## Files touched

- `.agent/audit-web-output-encoding.md` (new, deliverable)
- `.agent/2026-06-03-oglasino-web-web-output-encoding-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten with this summary)
- No source/test/config files touched (read-only brief).

## Tests

- Not run. Read-only audit; no code changed, no touched code paths.

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (finding feeds the Phase-3 seam analysis for M1 per the brief; no state.md edit owed by this audit)
- issues.md: no change. NOTE: the JsonLd JSON-LD `<script>` injection is a genuine latent web-side XSS finding (currently masked by backend input-encoding). Per the brief this is the feature's Phase-3 input, not a stray out-of-scope bug, so it is reported in the audit rather than drafted as an issues.md entry. If Mastermind decides it should also live in issues.md independently of the feature, the draft text is in "For Mastermind" below.

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed; deliverable + summary only.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one adjacent observation flagged in "For Mastermind" (MarkdownViewer unsanitized rehype-raw).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 11 (trust boundaries) — the finding is a trust-boundary point: web treats backend `<`-encoding as load-bearing for the JSON-LD script context; confirmed. Standing tool-reliability rule (state.md Risk Watch) — applied: every cited line grep + view cross-checked.

## Known gaps / TODOs

- I did not exhaustively trace the home/intro/about/pricing/free-zone JsonLd payloads field-by-field; spot-read confirmed they are translation-key-fed (trusted). The three pages carrying raw user content (product/user/catalog) were fully traced — those are the ones that matter.
- I did not attempt to run an actual XSS payload through the stack (read-only). The sink is asserted from code inspection of `JSON.stringify`-into-`<script>`, a well-established vector.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — no code written.
  - Considered and rejected: nothing — read-only audit, no implementation choices.
  - Simplified or removed: nothing.

- **Headline for Phase 3 / M1 seam analysis:** the backend input-sanitization filter is NOT purely redundant with web output-encoding. Web has one `<script>`-context sink (`JsonLd`, JSON-LD) where JSX's default escaping does not apply and `JSON.stringify` does not escape `</script>`. Today the backend's `<`-encoding of `product.name`/`product.description`/`displayName`/`product.title` is what keeps the product/user/catalog public pages safe. **Removing the backend filter without a paired web fix opens stored XSS on public, unauthenticated pages.** The web fix is small (escape the serialized JSON for the script context, e.g. `JSON.stringify(item).replace(/</g, '\\u003c')`, or deliver structured data without a raw `<script>`). Sequencing: web fix must land before or together with the backend removal — recommend a web brief paired with (and gating) the backend filter-removal brief.

- **Adjacent observation (Part 4b) — out of scope, not fixed:**
  - `MarkdownViewer` (`src/components/server/MarkdownViewer.tsx:27`) renders fetched markdown with `rehype-raw` and **no** `rehype-sanitize`/sanitizer — raw HTML in the markdown renders live. Severity: low **today** (source is the operator's own fixed GitHub files, privacy/terms, never user/DB/backend content), but **high the moment** that source ever becomes user- or DB-authored (e.g. per-locale legal content from a CMS, or any future markdown-from-backend feature). File: `src/components/server/MarkdownViewer.tsx`. I did not fix this — out of scope for this audit, and independent of the backend filter question. Candidate for an `issues.md` entry if Mastermind wants it tracked.

- **Optional issues.md draft (only if Mastermind wants the JsonLd finding tracked outside the feature):**
  > ## 2026-06-03 — Web: JSON-LD `<script>` injection — backend strings via `JSON.stringify` not escaped for script context
  > **Repo:** `oglasino-web` · **Severity:** high · **Status:** open
  > **Found in:** `src/components/server/seo/JsonLd.tsx:14` (`dangerouslySetInnerHTML={{ __html: JSON.stringify(item) }}`); fed by `generateProductPageStructuredData.ts:61-62,116`, `generateUserPageStructuredData.ts:19,46`, `generateCatalogPageStructuredData.ts:44`.
  > **Detail:** Raw `product.name`/`product.description`/`user.displayName`/`product.title` are injected into a `<script type="application/ld+json">` via `JSON.stringify`, which does not escape `</script>`/`<`/`>`. A value containing `</script><script>…</script>` breaks out and executes — stored XSS on public pages. Currently masked by the backend's input HTML-encoding; will be exposed if/when the backend-security-hardening feature removes `InputSanitizationFilter`. Fix web-side: escape serialized JSON for the script context (`.replace(/</g,'\\u003c')`) or change the delivery. Surfaced by the backend-security-hardening web output-encoding audit (`.agent/audit-web-output-encoding.md`).

- **Closure gate:** no config-file edit is required by this session. The two drafts above (issues.md entry for JsonLd, and the MarkdownViewer observation) are Mastermind's call — neither is a config-file dependency I am leaving implicit; both are explicitly surfaced here for triage. Nothing else flagged.
