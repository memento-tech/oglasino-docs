# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-05-30
**Task:** Brief D1 — Docs/QA close-out: product-validation mobile adoption (Chat A). Append the Chat A `decisions.md` entry; update `state.md` (product-validation = code-complete/pending-Ψ, NOT mobile-stable; Risk Watch); correct verified spec drifts in `features/product-validation.md`; archive this chat's session artifacts. NO `issues.md` writes.

## Implemented

- **decisions.md** — prepended a 2026-05-30 "Chat A" closing entry (newest at top, above the same-day image-pipeline entry) recording the rebuild + the six banked decisions: (1) rebuild onto the frozen contract with `toUpdateWirePayload` allow-list narrowing (closes the Part 11 trust-boundary violation), (2) MIME-undefined-passes RN translation, (3) `__system` option A (`findSystemError` scan; `parseServiceError` unchanged), (4) system/user keys rendered as sent (`system.rate_limited` live), (5) create-success UX RN translation (in-app View link + `triggerDashboardReload`; dead `onFinish` removed), (6) product-URL standardization. Pending-Ψ posture stated; no `mobile-stable`, no pipeline-done claim. Factual-vs-inferred note included.
- **state.md** — product-validation active-feature block → `web-stable` / mobile **code-complete** on `new-expo-dev`, pending Ψ (explicit "do not promote to `mobile-stable` until Ψ passes"); Expo-backlog row → mobile `in-progress`, adopted in `oglasino-expo-product-validation-1..-8`; two Risk Watch rows extended (uncommitted-`new-expo-dev` row now enumerates the product-validation rebuild; iOS+Android-rebuild row now notes product Ψ shares the rebuild dependency because create exercises the image-upload pipeline); `Last updated` → 2026-05-30; new session-log line.
- **features/product-validation.md** — corrected six verified spec drifts (see Config-file impact for the per-drift verdict). Drift 7 skipped (spec doesn't track namespace placement). Drift 4 (stale rate-limit key) extended to two further in-spec occurrences beyond the table row (a step-2 screenshot comment and §Platform adoption Part B) for internal consistency — both backed by the same cited reality, and Part B is the mobile intake material so leaving it wrong would mislead the mobile chat.
- **Archival** — 22 chat-A artifacts copied to `sessions/` (verified byte-identical) and sources deleted from sibling `.agent/` folders.

## Files touched

- decisions.md (1 new entry prepended)
- state.md (active-feature block, Expo-backlog row, 2 Risk Watch rows, Last-updated date, session-log line)
- features/product-validation.md (6 drift corrections across ~8 edits)
- sessions/ (+22 archived files; 1 in-repo rename for collision disambiguation)
- Sibling `.agent/` deletions (22 sources): `oglasino-expo` (8 product-validation summaries + `audit-expo-product-validation.md` + `audit-expo-readiness-product-validation.md` + `audit-create-update-flow.md`), `oglasino-web` (3 create-flow audit summaries + `audit-create-flow.md` + `audit-create-success-path.md` + `audit-create-update-flow.md`), `oglasino-backend` (`product-validation-verify-1`, `rate-limit-key-confirm-1`, `errors-seed-coverage-1`, `copy-done-label-seed-1`, `dialog-chrome-keys-seed-audit-1`)

## Tests

- N/A — markdown-only docs repo, no test suite. Verification was grep-based: confirmed zero remaining bare `/secure/products` and zero `product.system.rate_limited` in the spec; confirmed all 22 archived copies byte-identical (`cmp`) before deleting sources; confirmed no `sessions/` filename collisions.

## Cleanup performed

- Resolved a cross-repo audit filename collision: web and expo both carried `audit-create-update-flow.md` (distinct content — web reference vs mobile delta). Renamed the archived copies to the repo-disambiguated form per the boot-redesign / version-checksums precedent: `audit-oglasino-web-create-update-flow.md` and `audit-oglasino-expo-create-update-flow.md`.
- All 22 source files deleted from sibling `.agent/` folders after verified archival (conventions Part 3 cross-repo exception). Confirmed no product-validation / create-flow artifacts remain in any sibling `.agent/`.

## Config-file impact

- conventions.md: no change.
- decisions.md: **1 new entry** — "2026-05-30 — Chat A: mobile product create/edit/validation rebuilt onto the frozen `product-validation` contract (code-complete, pending Ψ)".
- state.md: product-validation active-feature block updated; Expo-backlog product-validation row updated (`not-started` → `in-progress`); 2 Risk Watch rows extended; Last-updated date; 1 session-log line.
- issues.md: **no change** (per Igor — issues wait for the post-smoke pass). The `M issues.md` in `git status` is pre-existing uncommitted state from a prior session, not this session's edit.

### Per-drift verdict (features/product-validation.md)

1. **`free` removed — CORRECTED.** §Request DTOs and §Known gaps both said the stale `private boolean free` was still present; backend verification (`oglasino-backend-product-validation-verify-1`) confirmed field+getter+setter fully gone. Both sections amended to "removed"; the issues.md closure left for Igor.
2. **Enum attribution — CORRECTED.** "42 constants total" on `ProductErrorCode` → "spans 42 codes across `ProductErrorCode` (36) + `SystemErrorCode` + `UserErrorCode`," with the 6 non-product codes named (cited reality: same backend verification).
3. **Path prefix — CORRECTED.** Five bare `/secure/products` references normalized to `/api/secure/products` (lines were in §Create flow ×2, §Update flow ×2, §Platform adoption update-flow logic). The §Endpoints + §Platform-adoption-Part-A paths already carried `/api`. The `/secure/images/upload-tokens` path left untouched (not a products path; no cited reality).
4. **Stale rate-limit key — CORRECTED.** `product.system.rate_limited` → `system.rate_limited` in the error-code table (cited: `oglasino-backend-rate-limit-key-confirm-1` — `system.rate_limited` seeded in all four locales, `product.system.rate_limited` exists nowhere). Extended to the two other in-spec occurrences for consistency.
5. **Orphan-cleanup timing — CORRECTED.** §Step 4 → On failure now states cleanup runs on ANY non-success create result (validation + transport), with the retry-re-uploads-from-memory consequence and the one exception (upload-throw-before-POST). Cited: web `audit-create-update-flow.md` ("Orphan cleanup fires first if `newKeys.length > 0`" on the 400/422 branch).
6. **Filter-field step matching — CORRECTED.** Step-3 mapping now lists all three matched forms (exact `filters`, exact `filter`, any `filter.`-prefixed). Cited: web audit (`filters/filter*→3`).
7. **`button.close.label` DIALOG-vs-BUTTONS — SKIPPED (correct per brief).** The spec does not track namespace placement of `button.close.label` (grep returned nothing), so per the brief's "record only if your spec tracks namespace placement," no edit.

## Obsoleted by this session

- The two superseded web audits (`audit-create-flow.md`, `audit-create-success-path.md`) and the superseded pre-Φ mobile audit (`audit-expo-readiness-product-validation.md`) are now archived to `sessions/`, so future readers no longer encounter them as live/current in engineer `.agent/` folders. Their canonical replacements (`audit-oglasino-web-create-update-flow.md`, `audit-expo-product-validation.md`) are archived alongside and remain discoverable.
- The pre-correction spec statements (free field "still present," "42 on ProductErrorCode," bare `/secure/products`, `product.system.rate_limited`, error-branch-only cleanup, `filter.`-only matching) are obsolete and were deleted in this session.

## Conventions check

- Part 1 (doc style): confirmed — relative links only, kebab-case archived filenames preserved on straight copies, repo-disambiguation followed the established precedent.
- Part 3 (config-file writes): confirmed — every substantive edit traces to the upstream brief (Mastermind/Igor); drift extensions (#4 to two more sites) are stale-reference consistency fixes backed by the same cited reality. issues.md left untouched per Igor.
- Part 3 (cross-repo `.agent/` exception): confirmed — only session-archival copies + post-archival source deletes in sibling `.agent/`; no other cross-repo writes; no source/test/docs touched.
- Part 4 (cleanliness): confirmed — no dead links introduced; collision resolved; sources cleaned; no duplicate of the already-current image-pipeline row.
- Part 4a (simplicity): see "For Mastermind."
- Part 4b (adjacent observations): flagged in "For Mastermind."
- Part 5 (session summary): this file + `last-session.md` twin; `<n>=1` (first oglasino-docs product-validation session).
- Part 6 (translations): confirmed — corrected stale key references to the live seeded `system.rate_limited`; no keys invented.
- Part 7 (error contract): confirmed — drift corrections preserve the `{field, code, translationKey}` wire shape and the 403→422→400 routing.
- Closure gate: no pending upstream config-file drafts left un-applied; the one new substantive need surfaced (other `product.system.*`/`product.user.*` table keys, below) is flagged for Mastermind, not silently applied.

## Known gaps / TODOs

- Deferred to Igor's post-smoke pass (explicitly out of scope, NOT done this session): the `copy.done.label` RU/CNR native-translator review; the `finish.label` dead-key/rename question; the un-run `new.product.success.*` / DIALOG locale-completeness sweeps; the image-pipeline V6/V9/V4 gaps (image-pipeline chat's issues); the Ω cleanup leftovers (`Array<...>` lint sweep, scroll-to-error on-device verification, base-URL consolidation across share helpers); **all `issues.md` entries** for the above.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — docs-only edits, no abstractions.
  - Considered and rejected: rewriting the entire cross-cutting error-code table to `system.*`/`user.*` (rejected — no cited reality for keys other than `system.rate_limited`; see flag below).
  - Simplified or removed: collapsed the cross-repo `audit-create-update-flow.md` collision to two clearly-disambiguated archived names.

- **Brief-vs-reality #1 — image-pipeline Expo-backlog row was already corrected.** Brief §2 said state.md "currently records image-pipeline mobile as `not-started`" and asked me to update it to implemented-with-gaps. Reality: the 2026-05-30 image-pipeline close-out already set that row to mobile `in-progress` (adopted in `oglasino-expo-image-pipeline-1..-4`, smoke pending). I left it as-is — applying the brief's edit would have duplicated or regressed a more-current record. No action needed; flagged so the brief author knows the row is already correct.

- **Brief-vs-reality #2 — no discrete "URL-parity brief" summary exists to archive, and the URL standardization is not shown landed in the artifacts.** Brief §1 banks "Product URL standardized … via the URL parity brief" and §0 lists a "URL-parity brief" among the code-complete set. But `oglasino-expo/.agent/` held product-validation summaries `-1..-8` only (`-1` Phase-2 audit, `-2..-5`+`-7` = A1–A5, `-6` = Ω-A cleanup, `-8` = mobile delta audit) — no separate URL-parity summary. The latest summaries actually show the URL still divergent: A5 (`-7`) reuses `getNormalizedProductUrl` which hardcodes `https://www.oglasino.rs`, and the delta audit (`-8`) flags the `.rs`/`.com`/raw-slug divergence as an **open** parity gap (severity low–medium). I recorded the decision as Igor banked it but softened the decisions.md bullet to "banked direction — landed-in-code status not verifiable from the archived artifacts," and adjusted the factual-vs-inferred note accordingly. **Recommended resolution:** reconcile the URL-parity work against the actual `new-expo-dev` code in the post-smoke pass; if a URL-parity session summary exists elsewhere, route it to a future Docs/QA session for archival.

- **Adjacent observation (Part 4b, medium) — the rest of the cross-cutting error-code table likely carries the same pre-error-code-split staleness.** After fixing the rate-limit row, the table still lists `product.system.not_authenticated`, `product.system.access_denied`, `product.system.internal_error` (cross-cutting block) and `product.system.not_owner` / `product.user.setup_incomplete` (product-level block). The 2026-05-29 error-code-split renamed `product.system.*` → `system.*` and `product.user.*` → `user.*`, so these are probably stale too — but I have **no cited reality** for those exact live key strings (the backend rate-limit audit only confirmed `system.rate_limited`; the verify audit confirmed the codes' home enums, not their translationKey strings). Per the no-invented-facts rule I did NOT change them. **Recommended:** a short backend grep to confirm the live `SystemErrorCode`/`UserErrorCode` translationKeys, then a follow-up Docs/QA edit to normalize the remaining table rows. This matters because §Platform adoption is the mobile chat's intake material.

- **Config-file dependency closure:** all of the brief's drafted config edits (decisions entry, state updates, spec drifts) are applied on disk. The two items above are flagged for Mastermind, not applied. No unstated config dependency remains.
