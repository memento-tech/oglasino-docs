# Session summary

**Repo:** oglasino-web
**Branch:** stage
**Date:** 2026-05-30
**Task:** READ-ONLY audit of the web client's image pipeline across 10 areas, code as ground truth, flag every code-vs-spec divergence — output to `.agent/audit-image-pipeline.md`.

## Implemented

- Produced `.agent/audit-image-pipeline.md`: a code-level inventory of the web image pipeline under the 10 brief headings, with exact `file:line` references, real signatures/shapes, and a divergence summary table up top.
- All findings verified against the actual source (`src/lib/images/*`, `src/lib/stores/viewTokens.ts`, `src/lib/config/api.ts`, consumers, entity services) — not against the spec.
- Surfaced 7 code-vs-spec divergences (D1–D7), the two headline ones being that the spec's "Deferred items" (view-token Zustand store, comprehensive retry policy) are **already implemented** in code.

## Files touched

- `.agent/audit-image-pipeline.md` (new, audit deliverable)
- `.agent/2026-05-30-oglasino-web-image-pipeline-1.md` (new, this summary)
- `.agent/last-session.md` (overwritten with a copy of this summary)

No source files modified — read-only audit.

## Tests

- None run. Read-only audit; no code changed, so `lint` / `tsc` / `test` are not applicable to touched paths.

## Cleanup performed

- none needed (no code written).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change — but see "For Mastermind": the `image-pipeline` spec's "Deferred items" and "Translation keys" sections are stale vs. code; that's a *spec/feature-doc* correction, owned by the Mastermind→Docs/QA path, not a config-file edit. No config-file dependency from this session.
- issues.md: no change (no new out-of-scope bug filed; one low-severity adjacent observation noted in "For Mastermind" for Mastermind to triage)

## Obsoleted by this session

- Nothing in code. The audit documents that several `image-pipeline.md` spec sections ("Deferred items": view-token Zustand store + comprehensive retry policy; "Translation keys": `TODO(phase 7)` framing) are now stale relative to the shipped code. The spec is in `oglasino-docs/` — I cannot edit it; flagged for Mastermind/Docs/QA.

## Conventions check

- Part 4 (cleanliness): confirmed — no code added.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one low-severity item flagged in "For Mastermind".
- Part 6 (translations): N/A for changes; audit *documents* the live key set and notes the existing `.label` parent/child-collision handling (Rule 2) in `errorMapping.ts` is correct.
- Part 7 (error contract): confirmed — image pipeline uses the `{ error: { code, ... } }` envelope (distinct from the product `{ errors: [{field, code}] }` shape); client keys i18n off `error.code` via `errorMapping.ts`.
- Part 11 (trust boundaries): confirmed in audit — client sends only processed content-types + full backend-issued keys; all auth/membership decisions are server/Worker-side. No client value drives a trust decision.

## Known gaps / TODOs

- Worker internals and backend internals were explicitly out of scope (separate audits); the audit reports only what the web client sends/consumes.
- I did not exhaustively enumerate every `publicImageUrl` consumer's exact line — named the components; metadata generators listed by file.
- `oglasino-expo` deliberately not read, per brief.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **Spec-correction recommendations (for the canonical `features/image-pipeline.md`, via Docs/QA):**
  1. **"Deferred items → View-token Zustand store"** is stale. `src/lib/stores/viewTokens.ts` is a complete store (60s pre-expiry buffer, in-flight dedupe, `invalidate`, `clear` at logout) and `MessageImages.tsx` consumes it — it does **not** use per-component `useState` for the token. Recommend moving this from "Deferred" to the implemented Display flow.
  2. **"Deferred items → Comprehensive retry policy"** is stale. `uploadImages.ts` implements 5xx backoff `[1s,4s]`, immediate 429 with `retryAfterSec`, and `MessageImages` does 401-invalidate-and-retry-once. Recommend updating.
  3. **"Translation keys / Pending registration (68 rows) / TODO(phase 7)"** is stale. `errorMapping.ts` has 13 live keys wired through `safeT` + inline-English fallbacks; no `TODO(phase 7)` comments remain. Whether the 68 backend SQL seed rows actually exist is a *backend* question (out of my repo) — recommend a backend confirmation before the spec claims "registered."
  4. **Upload scope enum**: spec says `product|profile|chat|report`; code's `UploadScope` adds a live `review`. Recommend adding `review` to the spec's `/upload-tokens` field table.
  5. Minor: spec variant table omits the `original` variant that `variants.ts` exposes; spec hero row reads "watermark" but code correctly gates it behind `NEXT_PUBLIC_WATERMARK_ENABLED` (default off).

- **Adjacent observation (Part 4b), low severity:** `uploadImages.cleanupOrphanImages` uses `console.warn` (`src/lib/images/uploadImages.ts:408`) for partial-failure logging. Pre-existing, intentional (fire-and-forget orphan cleanup), and arguably fine, but it's an ad-hoc `console.*` rather than a project logger. Not fixed (out of scope, read-only). Mastermind's call whether to file in `issues.md`.

- **Note on the brief's "only file" instruction:** the brief says the audit is the only file to create. I also wrote the two mandatory Part-5 session-summary files (`...-image-pipeline-1.md` + `last-session.md`), which are required every session by conventions Part 5 and CLAUDE.md and are process overhead distinct from the audit deliverable. No other files created.

- **Closure gate:** no pending config-file drafts. The spec corrections above target `oglasino-docs/features/image-pipeline.md` (a feature doc, not one of the four config files) and are routed to Mastermind/Docs/QA. No unstated config-file dependency.
