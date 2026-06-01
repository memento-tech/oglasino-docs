# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-30
**Task:** READ-ONLY validation + test run of the image pipeline against the frozen contract — per-area PASS/FAIL/PARTIAL verdicts with line-anchored evidence (brief: `oglasino-expo validation audit: image pipeline contract conformance`).

## Implemented

- Ran the image-pipeline test suite (Step 0): **109/109 pass**, `tsc --noEmit` clean, lint 0 errors (pre-existing warnings only). No image test failed or skipped.
- Traced all 10 validation areas (V1–V10) to current code (file:line) against ground truth in priority order: backend audit → web audit → feature spec. Re-verified rather than trusting the prior current-state audit.
- Produced `.agent/validation-image-pipeline.md`: verdict table + per-area evidence + Step 0 results.
- Verdicts: **TESTS, V1, V2, V3, V5, V7, V8, V10 = PASS**; **V4, V9 = PARTIAL**; **V6 = FAIL**.
- Independently re-verified the two substantive findings (V6 converting-heic key, V9 avatar/chat cleanup gaps) by hand rather than relying on the fan-out agents, and corrected one agent miscall (`image.too.big` is a pre-existing seeded key → resolves).

## Findings (no fixes staged — findings only, per brief)

- **V6 (FAIL):** `errorMapping.ts:67-68` builds `image.processing.converting-heic` (hyphen) for the `converting-heic` stage; the backend seed is `converting_heic` (underscore). The stage is actively emitted at `AvatarUpload.tsx:71`, so the HEIC label silently falls back to hard-coded English for SR/RU/CNR. One-line fix (add a `STAGE_KEY_OVERRIDES` entry). Highest-value finding.
- **V9 (PARTIAL):** `cleanupOrphanImages` on entity-save-failure-after-upload is wired for product (`productService.ts:92`) and review (`reviewService.ts:118`) but NOT for the avatar (`user.tsx`, `updateUser` failure) or chat (`useActiveChatStore.sendMessage`, `addDoc` failure) surfaces. Web cleans up both. Sweepers backstop; chat orphans (private/chats, ≥30d Sunday sweep) linger longest.
- **V4 (PARTIAL):** 401/5xx/network/batch branches match web exactly; the 429 branch retries once after honoring retry-after (`uploadImages.ts:301-307`), whereas web surfaces 429 immediately. Benign, no wire-contract break; flagged to decide intent.

## Files touched

- `.agent/validation-image-pipeline.md` (new, +~160) — the validation audit.
- `.agent/2026-05-30-oglasino-expo-image-pipeline-3.md` (new) — this summary.
- `.agent/last-session.md` (overwritten) — exact copy of this summary.
- No source/test/config files changed (read-only validation).

## Tests

- Ran: `npx vitest run src/lib/images/ src/lib/services/imageTokensService.test.ts src/lib/stores/viewTokens.test.ts src/lib/stores/uploadProgress.test.ts`
- Result: 7 files passed, **109 passed / 0 failed / 0 skipped**.
- Also ran `npx tsc --noEmit` (clean) and `npm run lint` (0 errors, 75 pre-existing warnings).
- New tests added: none (read-only validation).

## Cleanup performed

- none needed (no code touched).

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: **change recommended (drafted in "For Mastermind").** The image-pipeline mobile work is implemented on `new-expo-dev` but `state.md` / the Expo backlog still record it as `not-started` (per the prior audit's headline). This validation confirms it is implemented-but-not-conformant (3 gaps). I do not edit `state.md` (Docs/QA is sole writer) — draft below.
- issues.md: **change recommended (drafted in "For Mastermind").** Three findings (V6 must-fix, V9 should-fix, V4 decide) are candidates for `issues.md` entries.

## Obsoleted by this session

- Nothing. This file is additive (a new validation artifact kept alongside, not replacing, `.agent/audit-image-pipeline.md` per the brief). No code, tests, or docs in this repo are made dead by a read-only audit.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched; no commented-out code, debug logging, or stray TODOs added.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): three flagged in "For Mastermind" (duplicate `isPngInput`, likely web converting-heic parallel, png-passthrough vs web alpha handling).
- Part 6 (translations): central to V6 — one mobile-requested key (`image.processing.converting-heic`) won't resolve against the seeded name; reported, not fixed (out of scope).
- Part 7 (error contract): confirmed via V5 — image errors stay on the singular `{error:{code}}` seam; `parseServiceError` absent from all image paths (grep exit 1).
- Part 11 (trust boundaries): N/A for client validation — keys are server-constructed; mobile sends back keys verbatim (V7).

## Known gaps / TODOs

- On-device behavior (a real PUT landing in R2 against the staging Worker) is out of scope per the brief — explicitly not claimed.
- The three findings are reported as findings, not patched (brief: "every gap is a finding, not a patch").

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only validation, no code added. (Used a 10-agent fan-out workflow to parallelize the V1–V10 traces, then hand-verified the two substantive findings; this is process, not code complexity.)
  - Considered and rejected: nothing — no implementation decisions in a validation pass.
  - Simplified or removed: nothing.

- **Overall verdict:** the image pipeline is structurally complete and the core upload→download wire contract is conformant and smoke-ready, but it is **NOT fully contract-conformant**. Three gaps, none individually blocking on-device smoke:
  1. **V6 (must-fix):** add `'converting-heic' → 'converting_heic'` to `STAGE_KEY_OVERRIDES` in `src/lib/images/errorMapping.ts`. Without it the HEIC stage label is English-only for non-EN locales despite a seeded translation existing.
  2. **V9 (should-fix):** wire `cleanupOrphanImages` into the avatar save-failure path (`app/owner/dashboard/user.tsx`) and chat send-failure path (`src/lib/store/useActiveChatStore.ts` `sendMessage`), matching web.
  3. **V4 (decide):** confirm whether the 429 single retry-after retry is intended or should surface immediately for strict web parity.

- **Adjacent observations (Part 4b):**
  - **Duplicate `isPngInput`** — `processImage.ts:228-232` and `uploadImages.ts:367-371` are byte-identical; token-time/PUT-time Content-Type equality (Worker-enforced) silently depends on them staying in sync. Severity low (could mislead a future editor into desyncing the pair). Out of scope — not fixed.
  - **Likely cross-repo (web) converting-heic parallel** — web appears to use the same hyphenated stage value, so the V6 miss may also affect `oglasino-web`. I cannot edit/verify other repos; flagging so Mastermind can task the web agent if confirmed. Severity medium (silent non-EN localization miss).
  - **png-passthrough vs web alpha handling** — mobile keeps all PNGs as PNG under `outputFormat:'png-passthrough'` (only profile/avatar surfaces opt in), whereas web transcodes non-alpha PNG → JPEG. Consistent on the wire (no Worker rejection), so within contract; noted as an intentional RN divergence (Decision Q2), not a defect.

- **Drafted config-file text (for Docs/QA to apply — I did not edit the files):**

  - **`issues.md`** — three new entries (2026-05-30, feature image-pipeline, repo oglasino-expo, branch new-expo-dev):
    - *[high] image-pipeline mobile: HEIC stage label won't localize.* `errorMapping.ts` requests `image.processing.converting-heic` (hyphen) but the seeded key is `image.processing.converting_heic` (underscore); stage actively emitted by `AvatarUpload.tsx:71`. Non-EN users see hard-coded English. Fix: add `STAGE_KEY_OVERRIDES['converting-heic'] = 'converting_heic'`. May also affect oglasino-web (same hyphenated stage value) — verify.
    - *[medium] image-pipeline mobile: orphan cleanup missing on avatar + chat save-failure.* `cleanupOrphanImages` not called when `updateUser` fails (`user.tsx`) or chat `addDoc` fails (`useActiveChatStore.sendMessage`); web cleans up both. Backend sweeper backstops (chat orphans linger ≥30d).
    - *[low] image-pipeline mobile: 429 retries once vs web's surface-immediately.* `uploadImages.ts:301-307` retries once after honoring retry-after; web surfaces 429 immediately. Decide intended behavior.

  - **`state.md`** — image-pipeline status: the mobile implementation exists on `new-expo-dev` (not `not-started` as currently recorded). Suggested: record image-pipeline mobile as `in-progress-mobile` (implemented, validated, 3 conformance gaps open before on-device smoke), and update the Expo backlog table row accordingly. (Confirms the correction the prior current-state audit already drafted.)

- **Closure gate:** the only config-file changes this session would require are the `issues.md` and `state.md` drafts above, both stated here for Docs/QA. No other implicit config-file dependency. No code committed (per hard rules — Igor commits).
