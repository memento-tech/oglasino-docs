# Audit — image-pipeline docs reconciliation

**Repo:** oglasino-docs
**Date:** 2026-05-31
**Type:** read-only. No edits made to any file this session.

---

## Item 1 — Missing test-cases doc

**Does `features/image-pipeline-mobile-test-cases.md` exist on disk?** **NO.**

Confirmed by directory listing (`ls features/image-pipeline-mobile-test-cases.md` → "No such file or directory") and by a repo-wide `find` for any variant filename (`*image-pipeline*test*`, `*test*image-pipeline*`, and all `*image-pipeline*` files). The only `image-pipeline` files present are `features/image-pipeline.md`, the archived `sessions/*image-pipeline*.md` summaries/audits, and `.agent/handoffs/expo-image-pipeline.md`. **No test-cases doc exists under any name.**

**Every file that references it (4 files, 4 references):**

| File | Line | Context |
|---|---|---|
| `features/image-pipeline.md` | 590 | § Platform adoption — `(see [image-pipeline-mobile-test-cases.md](image-pipeline-mobile-test-cases.md) …)` |
| `state.md` | 261 | Expo-backlog table, Image-pipeline row — "on-device smoke (gate 2) pending per [features/image-pipeline-mobile-test-cases.md](…) before flip to `adopted`/`mobile-stable`" |
| `state.md` | 288 | Session-log line (2026-05-30) — "Created the on-device smoke test-case doc `features/image-pipeline-mobile-test-cases.md`." |
| `decisions.md` | 143 | 2026-05-30 entry, "Remaining gate" — "executed by Igor against the test-case doc at [`features/image-pipeline-mobile-test-cases.md`](…)" |

**Flag (contradiction, not just a missing file):** `state.md` line 288 asserts in its own session log that the doc *was created* ("Created the on-device smoke test-case doc …"), and `decisions.md` line 143 plus `state.md` line 261 both treat it as the live gate-2/Ψ artifact Igor smokes against. The file is not on disk. So three of the four references point at an artifact that the docs claim exists and is load-bearing for the `mobile-stable` flip, but which is absent. This is a real spec-vs-disk drift, not a forward-reference convention (the convention in Part 1 covers *images*, not linked `.md` docs). Whatever later edit brief addresses this should either create the doc or strike all four references — leaving it half-stated reproduces the H2 "drafted but never applied" pattern that `decisions.md`/conventions Part 3 call out.

---

## Item 2 — § Translation keys: active-key table verbatim

The section header framing (for your backend-seed diff) reads, verbatim:

> ### Seeded (68 rows)
>
> All 17 keys ARE seeded ×4 languages (EN/RS/RU/CNR) — **68 rows confirmed** in the backend audit §10. These are not pending; the earlier "pending registration / `TODO(phase 7)`" framing is stale. The seeded key names below are **authoritative** — they are the strings that actually resolve.

The **Active keys (10 keys)** table, verbatim from `features/image-pipeline.md` §Translation keys:

**Active keys (10 keys) — authoritative seeded names:**

| Key | Namespace | English |
|---|---|---|
| `image.upload.failed` | `ERRORS` | `Image upload failed: {filename} ({code})` |
| `image.processing.validating` | `INPUT` | `Checking…` |
| `image.processing.converting_heic` | `INPUT` | `Converting HEIC…` |
| `image.processing.resizing` | `INPUT` | `Resizing…` |
| `image.processing.encoding` | `INPUT` | `Compressing…` |
| `image.processing.cancelled` | `INPUT` | `Cancelled` |
| `image.processing.error` | `INPUT` | `Failed` |
| `image.processing.complete.label` | `INPUT` | `Done · {originalSize} → {processedSize}` |
| `image.processing.uploading.label` | `INPUT` | `Uploading…` |
| `image.processing.uploading.with.size` | `INPUT` | `Uploading {size}…` |

Counts as claimed by the spec: **17 keys total / 68 rows (×4 languages) / 10 active**. The remaining 7 are listed in the spec as **Phase 8 reserved keys** (`image.invalid`, `image.forbidden`, `image.bad.format`, `image.rate.limited`, `image.server.error`, `image.token.expired`, `image.session.expired`) — i.e. 10 active + 7 reserved = 17. The spec also notes four active keys carry seeded names that differ from the original spec strings: `converting-heic` → `converting_heic`; `.label` suffixes added to `complete`/`uploading` (Part 6 Rule 2 parent/child collision renames); and `image.uploading*` rescoped to `image.processing.uploading*`.

---

## Item 3 — § Platform adoption verbatim

Verbatim from `features/image-pipeline.md` § Platform adoption (the full section, mobile as-built state):

> ## Platform adoption
>
> Backend and web are complete. Mobile (`oglasino-expo`) is **implemented and validated** (on `new-expo-dev`); on-device smoke is the remaining gate before `mobile-stable` (see [image-pipeline-mobile-test-cases.md](image-pipeline-mobile-test-cases.md) and the [decisions.md](../decisions.md) 2026-05-30 entry).
>
> As-built mobile implementation:
>
> - **Token-based upload via one shared orchestrator** (`uploadImages`) across all four surfaces — product, profile, chat, and review.
> - **Display on `expo-image`.** HEIC→JPEG is handled natively at the picker boundary via `expo-image-manipulator` (web's `heic2any` is browser-only and is not used on mobile).
> - **Raw-byte PUT via `expo-file-system` `BINARY_CONTENT`** — not multipart, not fetch+Blob.
> - **Same contract as web:** same endpoints, same JWT structure, same full-prefixed-key contract. CORS does not apply on React Native (no Origin header).
> - **One intentional platform divergence:** on a 429, mobile honors `Retry-After` and retries once; web surfaces immediately. Deliberate — the better mobile UX (cross-reference the [decisions.md](../decisions.md) 2026-05-30 entry).
> - **No new native module** — all image libraries are first-party Expo modules already in the dev build.

**Cross-reference note for your expo Phase 2 diff:** the spec's top-line status is `web-stable` (line 5), and the Platform-adoption "No new native module" bullet (last line above) is worth diffing carefully against the Φ4 / consent-mobile reality recorded elsewhere in `state.md`, which repeatedly cites a *pending iOS+Android rebuild* needed to land the new `@react-native-community/netinfo` native module before any on-device smoke (including this feature's product-create-via-image-upload smoke) is meaningful. The image-pipeline spec itself claims no new native module for *its own* libraries — both can be true — but the practical smoke gate is the same blocked rebuild, so the as-built "already in the dev build" claim should be read against that.

---

## Summary

| Item | Finding |
|---|---|
| 1 | `features/image-pipeline-mobile-test-cases.md` does **NOT** exist on disk (no variant either). Referenced in 4 places: `features/image-pipeline.md:590`, `state.md:261`, `state.md:288`, `decisions.md:143`. `state.md:288` claims it was created — contradiction flagged. |
| 2 | 10-key active table extracted verbatim above; spec frames it as 17 keys / 68 rows / 10 active + 7 Phase-8 reserved. |
| 3 | § Platform adoption extracted verbatim above; status `web-stable`, mobile "implemented and validated" pending on-device smoke. |
