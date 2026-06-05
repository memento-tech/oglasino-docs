# Audit — image-pipeline web reference (READ-ONLY)

**Repo:** oglasino-web
**Branch:** dev (not switched)
**Date:** 2026-05-31
**Task:** Read-only reference audit. Report web's ground truth for three image-pipeline items, each cited by file:line, so the mobile (oglasino-expo) image-pipeline polish chat can mirror web. No code changes.

All claims below are read from the code on disk. I did not read any pre-existing report about this feature.

---

## Item 1 — Progress-text wiring in the web upload dialog

### Where the stage→text mapping lives

The single mapping function is **`stageLabel`** (and its size-aware sibling **`processingMessage`**) in **`src/lib/images/errorMapping.ts`**:

- `stageLabel(tInputs, stage)` — `src/lib/images/errorMapping.ts:215`
- `processingMessage(tInputs, stage, info)` — `src/lib/images/errorMapping.ts:233`

The translation namespace passed in is always **`INPUT`** (`tInputs = useTranslations(TranslationNamespaceEnum.INPUT)`), confirmed at every call site (e.g. `MessageInput.tsx:42`, `ImagesImport.tsx:35`, `AvatarUpload.tsx:24`, `ProductReviewImageImport.tsx`).

### The exact key-construction rule

`stageLabel` builds the translation key like this (`src/lib/images/errorMapping.ts:216`):

```ts
const key = STAGE_KEY_OVERRIDES[stage] ?? `${STAGE_KEY_PREFIX}${stage}`;
```

- `STAGE_KEY_PREFIX = 'image.processing.'` — `errorMapping.ts:196`
- `STAGE_KEY_OVERRIDES` — `errorMapping.ts:206-209`:
  ```ts
  const STAGE_KEY_OVERRIDES: Record<string, string> = {
    uploading: 'image.processing.uploading.label',
    complete: 'image.processing.complete.label',
  };
  ```

So for any stage **not** in the override map, the key is the bare `image.processing.<stage>` — and the `<stage>` token is concatenated **verbatim**, including its hyphen.

### The exact stage → translation-key mapping web actually uses

| Internal stage value | String web passes to the translator | Source of the key choice |
| --- | --- | --- |
| `validating` | `image.processing.validating` | bare prefix |
| `converting-heic` | **`image.processing.converting-heic`** (HYPHEN) | bare prefix — **no override** |
| `resizing` | `image.processing.resizing` | bare prefix |
| `encoding` | `image.processing.encoding` | bare prefix |
| `uploading` | `image.processing.uploading.label` | `STAGE_KEY_OVERRIDES` (errorMapping.ts:207) |
| `complete` | `image.processing.complete.label` | `STAGE_KEY_OVERRIDES` (errorMapping.ts:208) |
| `idle` | `image.processing.idle` | bare prefix |
| `error` | `image.processing.error` | bare prefix |
| `cancelled` | `image.processing.cancelled` | bare prefix |
| (any unknown stage) | `image.processing.default`, then inline English | fallback chain, errorMapping.ts:222-225 |

`processingMessage` adds two **parameterized** variants on top of the table above, for the chat status line only (`errorMapping.ts:233-252`):

| Condition | Key | Params |
| --- | --- | --- |
| `stage === 'uploading'` and `info.processedSize` present | `image.processing.uploading.with.size` | `{ size }` |
| `stage === 'complete'` and both sizes present | `image.processing.complete.with.sizes` | `{ originalSize, processedSize }` |

Otherwise `processingMessage` delegates to `stageLabel` (`errorMapping.ts:251`).

### **The HEIC key — exact string web passes (brief's explicit ask)**

Web passes **`image.processing.converting-heic`** — **hyphen, not underscore**. There is **no normalization** of this stage: `converting-heic` is absent from `STAGE_KEY_OVERRIDES` (`errorMapping.ts:206-209`), so `stageLabel` falls to the bare-prefix branch and emits the hyphen form. The hyphenated stage string originates at the emitter `processImage.ts:123` (`onProgress?.('converting-heic')`) and at the two direct picker-time call sites that hardcode it: `ImagesImport.tsx:200` and `AvatarUpload.tsx:72`, both `stageLabel(tInputs, 'converting-heic')`.

> Cross-repo note for the mobile chat (factual, not a fix request): mobile's V6 fix added a `STAGE_KEY_OVERRIDES` entry `'converting-heic' → 'converting_heic'` so its lookup hits the **underscore** seed. **Web has no such override** — web reads the hyphen key. I cannot read the backend SQL seed from this repo, so I am not asserting what the seeded key string is; I am reporting only that web's override map omits `converting-heic` and therefore queries `image.processing.converting-heic`. If the seed is the underscore form, web's lookup misses and falls through to `image.processing.default` → inline English `englishStageLabel('converting-heic')` = `'Converting HEIC…'` (`errorMapping.ts:262-263`). Flagged in "For Mastermind."

### The four upload surfaces and how each renders progress

All four subscribe to the Zustand `useUploadProgressStore` (`src/lib/stores/uploadProgress.ts`) and render via `stageLabel`/`processingMessage`:

1. **Chat upload dialog — `src/messages/components/MessageInput.tsx`** (this is the closest match to "the web upload dialog"):
   - imports `processingMessage` — `MessageInput.tsx:7`
   - drives the store from the upload callback: `onProgress: (e) => progress.setState(e.fileIndex, { stage: e.stage, ...e.info })` — `MessageInput.tsx:132`
   - renders an `aria-live="polite"` status block, one row per file, with the stage text at `processingMessage(tInputs, state.stage, state)` — `MessageInput.tsx:209` (block at `:197-215`)
2. **Product images picker — `src/components/client/ImagesImport.tsx`**:
   - per-thumbnail overlay `ImageStatusOverlay` → `stageLabel(tInputs, state.stage)` — `ImagesImport.tsx:227` (overlay component `:215-230`, mounted `:184`)
   - picker-time HEIC label `stageLabel(tInputs, 'converting-heic')` — `ImagesImport.tsx:200`
3. **Avatar picker — `src/components/owner/client/AvatarUpload.tsx`**:
   - single-file overlay label `stageLabel(tInputs, fileState?.stage ?? 'idle')` — `AvatarUpload.tsx:73`; picker-time HEIC label `stageLabel(tInputs, 'converting-heic')` — `AvatarUpload.tsx:72`
4. **Review-image picker — `src/components/popups/components/ProductReviewImageImport.tsx`**:
   - overlay label `stageLabel(tInputs, state.stage)` — `ProductReviewImageImport.tsx:147`

### Where the stage values originate

Stages flow `processImageForUpload` → `uploadImages` → store → component:

- `processImage.ts` emits stages via `onProgress`: `'validating'` (`:104`), `'converting-heic'` (`:123`), `'resizing'` (`:158`), `'encoding'` (`:159`, `:168`), `'complete'` (`:182`). The `ProcessingStage` union is defined at `processImage.ts:30-35`.
- `uploadImages.ts` re-emits those (suppressing processImage's terminal `'complete'`, `:131`) and adds `'uploading'` (`:245`), `'error'` (`:256`,`:281`), `'cancelled'` (`:252`,`:278`), plus its own terminal `'complete'` via `emitUploadComplete` (`:285-302`). Its stage union `UploadFileStage` is at `uploadImages.ts:54`.
- The store's `UploadFileStage` adds `'idle'` — `uploadProgress.ts:14-20`.

---

## Item 2 — `image.processing.*` stage→key consumption and the override/normalization layer

### Does web have a stage-key override/normalization layer? **Yes.**

`STAGE_KEY_OVERRIDES` at **`src/lib/images/errorMapping.ts:206-209`** is exactly that map (internal stage name → explicit seeded key). It is consulted first in `stageLabel` (`errorMapping.ts:216`). It contains **two** entries only:

- `uploading → image.processing.uploading.label`
- `complete → image.processing.complete.label`

The documented reason (code comment, `errorMapping.ts:198-205`): those two stages also have parameterized sibling keys (`image.processing.uploading.with.size`, `image.processing.complete.with.sizes`), and the translations cache's `toNested()` cannot let a leaf coexist with a nested child of the same path — so the bare label is given a `.label` suffix to sit alongside the `.with.*` siblings. (This is the conventions Part 6 Rule 2 parent/child-collision pattern.)

**`converting-heic` is deliberately/incidentally NOT in this map** — see Item 1.

### Which keys web actually reads at render time

Via `stageLabel` (bare prefix unless overridden):
`image.processing.validating`, `image.processing.converting-heic`, `image.processing.resizing`, `image.processing.encoding`, `image.processing.idle`, `image.processing.error`, `image.processing.cancelled`, plus the two overridden leaves `image.processing.uploading.label` and `image.processing.complete.label`.

Via `processingMessage` (chat status line only, in addition to the above):
`image.processing.uploading.with.size`, `image.processing.complete.with.sizes`.

Fallback keys/strings when a key is missing (`safeT` returns `null` when next-intl echoes the key back — `errorMapping.ts:110-113`):
- `image.processing.default` — `errorMapping.ts:222`
- then inline English from `englishStageLabel(stage)` — `errorMapping.ts:254-279` (e.g. `validating → 'Checking…'`, `converting-heic → 'Converting HEIC…'`, `resizing → 'Resizing…'`, `encoding → 'Compressing…'`, `uploading → 'Uploading…'`, `complete → 'Done'`, default → `'Processing…'`).

---

## Item 3 — `isPngInput` definition

**Not found in oglasino-web.** No function, variable, or symbol named `isPngInput` exists anywhere in `src/` or `app/` (grep for `isPngInput`, `PngInput`, `isPng` returns zero matches).

The PNG-handling decision web does have is **inline**, not a named predicate — `pickTargetFormat` in `src/lib/images/processImage.ts:205-220`:

```ts
async function pickTargetFormat(file: File): Promise<'jpeg' | 'png' | 'webp'> {
  if (file.type === 'image/png') {
    let hasAlpha: boolean;
    try {
      hasAlpha = await decoder.detectPngTransparency(file);
    } catch {
      return 'png'; // defensive: keep PNG if alpha detection fails
    }
    return hasAlpha ? 'png' : 'jpeg';
  }
  if (file.type === 'image/webp') return 'webp';
  return 'jpeg'; // image/jpeg + HEIC-converted-to-jpeg
}
```

So web's "is this a PNG input" test is the bare `file.type === 'image/png'` check at `processImage.ts:206`, used to branch into transparency detection. The transparency probe itself is `detectPngTransparency` (`src/lib/images/imageDecoder.ts:39-65`) — it decodes to a canvas and scans the alpha byte of each pixel, returning `true` on the first non-`255` alpha.

There is **no consolidated `isPngInput` helper to mirror**. If mobile is consolidating its two duplicated `isPngInput` copies, web offers no single definition to match against; the nearest semantic equivalent is the inline `file.type === 'image/png'` guard in `pickTargetFormat`. (Whether mobile's `isPngInput` does more than a MIME check — e.g. extension sniffing — cannot be compared, since web has no such function.)

---

## Files read (no writes)

- `src/lib/images/errorMapping.ts`
- `src/lib/images/processImage.ts`
- `src/lib/images/uploadImages.ts`
- `src/lib/images/imageDecoder.ts`
- `src/lib/stores/uploadProgress.ts`
- `src/messages/components/MessageInput.tsx`
- `src/components/client/ImagesImport.tsx`
- `src/components/owner/client/AvatarUpload.tsx`
- `src/components/popups/components/ProductReviewImageImport.tsx` (stage-usage lines only)

## Cleanup performed

- None needed (read-only audit; no code touched).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change authored by me. One existing entry is corroborated — see "For Mastermind."

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changes, nothing added.
- Part 4a (simplicity): N/A — read-only audit, no abstractions added/considered/removed.
- Part 4b (adjacent observations): one observation flagged in "For Mastermind."
- Part 6 (translations): touched as subject matter — the `converting-heic` hyphen vs the seeded key is a Rule-2/Rule-4-adjacent contract question, reported factually.
- Part 7 (error contract): N/A this session (error-code→key mapping was read but not assessed).

## Known gaps / TODOs

- I cannot confirm the backend's seeded key string for the HEIC stage (`image.processing.converting-heic` vs `image.processing.converting_heic`) from this repo — that lives in the backend SQL seed. The audit reports only what web passes to the translator. Cross-checking against the seed is a backend/Mastermind step.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.
- **Adjacent observation (Part 4b) — HEIC stage label likely renders English-only for non-EN locales (corroborates issues.md 2026-05-30 cross-repo entry).**
  - Description: web's `STAGE_KEY_OVERRIDES` (`src/lib/images/errorMapping.ts:206-209`) covers only `uploading` and `complete`; `converting-heic` is absent, so `stageLabel` queries the hyphenated `image.processing.converting-heic`. Mobile's V6 fix established that the seeded key is the **underscore** form `image.processing.converting_heic`. If that is also the seed web reads, web's lookup misses and falls back to inline English `'Converting HEIC…'` for SR/RU/CNR — the exact bug the mobile V6 fix closed. The `converting-heic` stage IS reachable in web (emitted at `processImage.ts:123`, and rendered at picker time at `ImagesImport.tsx:200` and `AvatarUpload.tsx:72`).
  - File: `src/lib/images/errorMapping.ts:206-209` (the gap); emitters `src/lib/images/processImage.ts:123`, `src/components/client/ImagesImport.tsx:200`, `src/components/owner/client/AvatarUpload.tsx:72`.
  - Severity guess: medium (non-EN users see an English stage label; not a crash, not data loss).
  - I did not fix this — out of scope for a read-only audit. The fix, if confirmed against the backend seed, mirrors mobile's: add `'converting-heic': 'image.processing.converting_heic'` to web's `STAGE_KEY_OVERRIDES`. This needs a real web brief (and confirmation of the seeded key) before any change.
- This is a Phase-2 reference audit; the consuming chat is the mobile image-pipeline polish chat (oglasino-expo). Nothing here requires a config-file edit.
