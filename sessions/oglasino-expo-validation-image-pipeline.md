# Validation — Image Pipeline Contract Conformance (mobile, `oglasino-expo`)

**Type:** READ-ONLY validation + test run. No code changed. No fixes staged.
**Date:** 2026-05-30
**Branch:** `new-expo-dev` (as checked out; not switched)
**Feature:** [image-pipeline](../../oglasino-docs/features/image-pipeline.md) — contract frozen.
**Method:** Ground truth in priority order — backend audit (`../oglasino-backend/.agent/audit-image-pipeline.md`), web audit (`../oglasino-web/.agent/audit-image-pipeline.md`), feature spec. Each contract point traced to current code (file:line). The prior current-state audit (`.agent/audit-image-pipeline.md`) was re-verified, not trusted. Tests were run.

---

## Verdict table

| Area | Verdict | One-line reason |
|---|---|---|
| **TESTS** | **PASS** | 109/109 image tests pass; `tsc --noEmit` clean; lint 0 errors (pre-existing warnings only). |
| **V1** Upload-token request | **PASS** | Endpoint, body (`chatId` omitted not null), processed content-types, positional pairing, `TOKEN_COUNT_MISMATCH` guard, full-prefixed-key return all conform. |
| **V2** Processing pipeline | **PASS** | Thresholds 10MB/8000/2400/5MB/0.85→0.75 all match; HEIC→JPEG at picker boundary before processImage; png-passthrough type consistent token↔PUT. |
| **V3** Raw-byte PUT | **PASS** | PUT to token's absolute `uploadUrl` via `expo-file-system` `BINARY_CONTENT`; `x-upload-token` + processed `Content-Type`; no `Authorization`, bypasses BACKEND_API. |
| **V4** Retry policy | **PARTIAL** | 401/5xx/network/batch branches match web exactly; **429 retries once after honoring retry-after, whereas web surfaces immediately.** Benign, no wire-contract break. |
| **V5** Error envelope seam (Φ4) | **PASS** | Singular `{error:{code}}` parsed via `ImageTokenError`/`parseErrorBody`/`errorMapping`; `parseServiceError` absent from every image path (grep exit 1); code map at parity with web. |
| **V6** Translation keys vs seeded | **FAIL** | `converting-heic` stage builds `image.processing.converting-heic` (hyphen) but seed is `converting_heic` (underscore); **actively emitted by AvatarUpload → silent English-only fallback for SR/RU/CNR.** |
| **V7** Four surfaces → one orchestrator | **PASS** | product/profile(manual+OAuth)/chat/review all call `uploadImages` with correct scope and forward keys to the correct field; review genuinely uploads. |
| **V8** View-token store | **PASS** | Per-chatId cache, 60s buffer, in-flight dedupe, invalidate/clear, correct POST body, logout wiring, MessageImages onError invalidate+capped-retry all present. |
| **V9** DELETE cleanup path | **PARTIAL** | `encodeKeyPath` + fire-and-forget correct; product/review clean up on save-failure, but **avatar and chat surfaces do not** (web does both). |
| **V10** Config / env | **PASS** | `EXPO_PUBLIC_CDN_URL` lazy+throwing, all three env files correct (stage/stage/prod); watermark gate present; issuance via BACKEND_API, PUT target from token. |

**Overall: NO — not fully contract-conformant.** The implementation is structurally complete and the core upload→download wire contract is conformant and smoke-ready. Three gaps must be closed before ship (none individually blocks on-device smoke):

1. **V6 (must-fix, low blast radius):** add a `STAGE_KEY_OVERRIDES` entry mapping `'converting-heic' → 'converting_heic'` (mirroring the existing `uploading`/`complete` `.label` overrides) so the HEIC-conversion label resolves to the seeded localized string instead of hard-coded English.
2. **V9 (should-fix):** wire `cleanupOrphanImages` into the avatar save-failure path (`app/owner/dashboard/user.tsx`) and the chat send-failure path (`useActiveChatStore.sendMessage`), matching web's per-surface orphan cleanup.
3. **V4 (decide):** the 429 single-retry diverges from web's "surface immediately." Confirm whether the one bounded retry-after retry is intended; if strict parity is required, drop it.

---

## Step 0 — Test / lint / typecheck results

**Image unit tests** — command:
```
npx vitest run src/lib/images/ src/lib/services/imageTokensService.test.ts \
  src/lib/stores/viewTokens.test.ts src/lib/stores/uploadProgress.test.ts
```
Result: **Test Files 7 passed (7), Tests 109 passed (109)** — 0 failed, 0 skipped. (Duration ~235ms.)
Files: `errorMapping.test.ts`, `processImage.test.ts`, `uploadImages.test.ts`, `variants.test.ts`, `imageTokensService.test.ts`, `viewTokens.test.ts`, `uploadProgress.test.ts`.

**Typecheck** — `npx tsc --noEmit`: **clean** (exit 0, no output).

**Lint** — `npm run lint` (`expo lint`): **0 errors, 75 warnings** — all pre-existing baseline. Image-path warnings are style-only nits: `productService.ts:47` and `reviewService.ts:78` (`@typescript-eslint/array-type` — `Array<T>` vs `T[]`), plus `import/first` in image test files. None are image-pipeline correctness issues.

No image test failed or was skipped. No top-line test finding.

---

## V1 — Upload-token request conformance · PASS

- Endpoint exactly `POST /secure/images/upload-tokens` via the `BACKEND_API` axios instance — `imageTokensService.ts:56-57`.
- Body `{scope, count, contentTypes, chatId?}` with `count === contentTypes.length` (both derived from the `files` array) — `uploadImages.ts:94,99-104`. `chatId` **omitted via conditional spread** `...(chatId !== undefined ? { chatId } : {})` — never sent as `null`. DTO types `chatId?: string` — `imageTokensService.ts:18-23`.
- `contentTypes` are the **processed/output** types, not picker mimeTypes — `uploadImages.ts:94` → `determineOutputContentType(f, outputFormat)` (`:357-365`) returns `'image/png'` (png-passthrough) or `'image/jpeg'`.
- Response `{tokens:[{token,key,uploadUrl,expiresAt}]}` consumed at `imageTokensService.ts:25-27,63`; entry shape `:11-16`.
- Positional pairing: `files.map((file, i) => uploadOneFile(file, tokens[i], ...))` — `uploadImages.ts:135`.
- Count-mismatch guard: `if (tokens.length !== files.length) throw new UploadError('TOKEN_COUNT_MISMATCH', ...)` — `uploadImages.ts:116-120`.
- Returns full prefixed keys: `uploadOneFile` returns `tokenEntry.key` (the backend-issued key field) — `:224`; `uploadImages` returns `succeededKeys` — `:138,143,167`.

**Adjacent (low):** `isPngInput` is duplicated byte-for-byte in `processImage.ts:228-232` and `uploadImages.ts:367-371`. Token-time `Content-Type` ↔ PUT-time `Content-Type` equality depends on both copies staying in sync; a future edit to one only could silently desync the Worker-rejected pairing. Flagged, not in scope to fix.

## V2 — Processing pipeline correctness · PASS

- HEIC→JPEG at the picker boundary, before processImage's dimension/size logic — `preparePickerAssets.ts:65-76` (`manipulateAsync([], {compress:0.95, format:JPEG})`, returns asset with `mimeType:'image/jpeg'`); detect via `isHeicAsset` (mimeType `image/heic|image/heif` OR `.heic/.heif` extension) `:92-98`; passthrough fast-path when no HEIC `:37-39`. processImage never decodes HEIC, so size/dimension logic always runs on a decodable JPEG.
- Thresholds all match the web audit: `MAX_INPUT_BYTES=10MB` (`processImage.ts:5`), `MAX_OUTPUT_BYTES=5MB` (`:6`), `MAX_DIMENSION=8000` (`:7`), `RESIZE_LONGEST_SIDE=2400` trigger **and** target (`:8,108-116`), `QUALITY_PRIMARY=0.85`/`QUALITY_FALLBACK=0.75` (`:15-16`) two-pass JPEG (`:161-178`).
- PNG: `usePngPath = outputFormat==='png-passthrough' && isPngInput(input)` (`:118`); PNG path is resize-only `encodePng` (single pass, `SaveFormat.PNG`, still enforces 5MB cap) (`:186-207`); `result.contentType = usePngPath ? 'image/png' : 'image/jpeg'` (`:134`). Deliberate per Decision Q2 (transparent-PNG avatars). **Note vs web:** web transcodes *non-alpha* PNG → JPEG and keeps *alpha* PNG; mobile's png-passthrough keeps **all** PNGs as PNG when `outputFormat==='png-passthrough'` (only the profile/avatar surfaces opt in). Consistent on the wire (token type === PUT type), so no Worker rejection — within contract.
- Token type === PUT type: `determineOutputContentType` (`uploadImages.ts:357-371`) and `processImage`'s decision share identical `isPngInput` logic; PUT sends `processed.contentType` (`:249-257`).

## V3 — Raw-byte PUT conformance · PASS

- PUT targets the token's absolute `uploadUrl` (from the response), not a CDN-base-built URL — `uploadImages.ts:249-251`; `uploadUrl` is a response field `imageTokensService.ts:11-14`.
- Raw bytes via `expo-file-system/legacy` `FileSystemUploadType.BINARY_CONTENT`, `httpMethod:'PUT'` — `uploadPrimitive.ts:30-34`; not multipart/FormData.
- Headers exactly `{'x-upload-token', 'Content-Type': processed.contentType}` — `uploadImages.ts:252-255`. `uploadBytes` builds headers solely from caller input (`uploadPrimitive.ts:23-88`); no `Authorization`, no axios/BACKEND_API in path (grep of `axios|BACKEND_API|Authorization|FormData|multipart` matched only a comment). Path deliberately bypasses the interceptor.

## V4 — Retry policy · PARTIAL

Per-branch trace (`uploadImages.ts` `putWithRetry`, `:229-323`):
- **401 `TOKEN_EXPIRED` → single re-issue + retry once:** `:283-299`, guard `tokenExpiredRetries < TOKEN_EXPIRED_MAX_RETRIES` (=1, `:66`); refetch single token `:290`; on refetch throw → `TOKEN_REISSUE_FAILED` `:292-297`. **Matches.**
- **5xx → backoff `[1s,4s]`, 2 retries:** `:310-315`, `RETRY_BACKOFF_MS=[1000,4000]` (`:64`), `serverErrorRetries < length`. **Matches.**
- **Network error → 1 retry:** `:258-272`, AbortError→`CANCELLED`; else one retry after `NETWORK_RETRY_DELAY_MS=500` then `NETWORK_ERROR`. **Matches.**
- **Fail-fast batch:** `Promise.allSettled` (`:134`), partition (`:140-147`), on any failure `void cleanupOrphanImages(succeededKeys)` (`:149-150`), surface first non-CANCELLED failure (`:154-160`). **Matches.**
- **429 → respect retry-after, surface immediately (no blind backoff):** `:301-307` does `rateLimitRetries < 1`, waits `retryAfterSec ?? readRetryAfterHeader ?? 5` seconds, then `continue` (**one retry**). **DIVERGES** from web's documented behavior (web surfaces 429 immediately with `retryAfterSec`, no retry — web audit §3/D2). Mobile respects retry-after and applies no blind/exponential backoff, but does not surface on the first 429.

Verdict PARTIAL: 4 of 5 branches conform exactly; the 429 branch behaviorally diverges from web's real behavior. Benign (server-dictated single wait, no wire-contract break, possibly intentional), but flagged for reconciliation.

## V5 — Error envelope seam (Φ4) · PASS

- Image errors parsed off the **singular** `{error:{code,...}}` envelope: `imageTokenErrorFromResponse` reads `body?.error?.code` (`imageTokensService.ts:108-111`); `imageTokenErrorFromError` reads `e?.response?.data?.error?.code`, maps `status===0`→`NETWORK_ERROR` (`:121`); `ImageTokenError` carries code/status/retryAfterSec (`:29-50`). PUT body: `parseErrorBody` reads `parsed?.error?.code` + `details.retryAfterSec` (`uploadImages.ts:378-393`).
- **`parseServiceError` absent from all image paths** — `grep -rn parseServiceError src/lib/images src/lib/services/imageTokensService.ts src/lib/stores` → **exit 1, zero matches**. (Defined at `src/lib/utils/parseServiceError.ts:40`; used only by non-image flows — ReportDialog, UploadedProductDialog, preValidateOutcome.) viewTokens.ts carries no error-code seam.
- `errorMapping.ts:113-165` routes Worker + backend + internal codes; internal `PROCESSING_FAILED`→`image.server.error` (`:146`), `TOKEN_REISSUE_FAILED`→`image.session.expired` (`:153`); `TOKEN_COUNT_MISMATCH` and `NETWORK_ERROR` route to the `image.upload.failed` catch-all (`:168-169`) — **identical to web** (web also leaves these two on the catch-all). **No code the web mapper handles is missing from mobile.** Full parity.

## V6 — Translation key resolution against SEEDED names · FAIL

Full list of mobile-requested `image.*` key strings (resolved against the backend-seeded names — backend audit §10):

**Stage labels (INPUT namespace), built `image.processing.${sub}` in `errorMapping.ts:67-68`:**
| Requested key string | Resolves? |
|---|---|
| `image.processing.validating` | ✓ seeded |
| `image.processing.converting-heic` | ✗ **MISSES** — seed is `image.processing.converting_heic` (underscore). **Live** (AvatarUpload.tsx:71). |
| `image.processing.resizing` | ✓ seeded |
| `image.processing.encoding` | ✓ seeded |
| `image.processing.uploading.label` (override, `:35-38`) | ✓ seeded |
| `image.processing.uploading.with.size` (`:53`) | ✓ seeded |
| `image.processing.complete.label` (override) | ✓ seeded |
| `image.processing.complete.with.sizes` (`:58`) | ✗ misses (not seeded) → English fallback; **same on web**, graceful |
| `image.processing.cancelled` | ✓ seeded |
| `image.processing.error` | ✓ seeded |
| `image.processing.default` (`:73`) | ✗ misses — intentional forward-compat probe before English fallback |

**Error keys (ERRORS namespace), `ERROR_CODE_TO_KEY` + fallthrough:**
| Requested key string | Resolves? |
|---|---|
| `image.upload.failed` | ✓ active seeded |
| `image.invalid` / `image.forbidden` / `image.bad.format` / `image.rate.limited` / `image.server.error` / `image.token.expired` / `image.session.expired` | ✓ reserved-7 seeded |
| `image.too.big` | ✓ **pre-existing seeded key** (backend audit §10 "not re-registered" list) — *resolves* (corrects a subagent miscall) |
| `image.not.owner` / `image.in.use` / `image.too.old` / `image.invalid.key` | ✗ miss (not seeded) → English fallback; **same on web**; only reachable on a user-triggered orphan DELETE |

**The single live defect:** `image.processing.converting-heic` (hyphen) is requested by `stageLabel` with no override (`errorMapping.ts:16-25,67-68`) and the `converting-heic` stage is **actively emitted** at `AvatarUpload.tsx:71` (`{ stage: 'converting-heic' as const }`) into `ImageStatusOverlay` (`:56,62`, `useTranslations(INPUT)`). The seeded name is the underscore form (`image.processing.converting_heic`, backend audit D3) — the underscore form is requested **nowhere** in mobile (grep confirms). Result: `safeT` returns null → `englishStageLabel` → hard-coded `"Converting HEIC…"`, so SR/RU/CNR users see English for this label even though a localized seed exists. This is exactly the "hard-coded spec key string = silent miss" class the brief flags as highest-value. `errorMapping.test.ts:161` bakes in the English-fallback behavior. Trivially fixed via an override entry. **Likely also affects web** (web uses the same hyphenated stage value) — flagged as an adjacent cross-repo observation for Mastermind, out of this repo's scope.

The other misses (`complete.with.sizes`, `default`, and the four DELETE-path keys) are benign: they match web, degrade gracefully to English, and either are forward-compat probes or only surface on rare explicit user-triggered DELETE.

## V7 — Four surfaces wire through the one orchestrator · PASS

- **Product:** `productService.ts:66-70` `uploadImages(files, {scope:'product'})`; keys → `imageKeys` via `toCreate/UpdateWirePayload` (`:88-90`, `productWirePayload.ts:28,55`).
- **Review:** `reviewService.ts:96-101` `uploadImages(files, {scope:'product'})` ("reviews share products prefix"); keys → `imageKeys` in payload (`:111`) → `POST /secure/review/product` (`:114`). The dialog **genuinely uploads** — `ProductReviewDialog.tsx:97-119` calls `reviewProduct({images, imageKeys:[]}, {onProgress})` with images from `ImagesImport` (`:192-198`); not the dead orphan `ProductReviewImageImport.tsx`.
- **Profile (manual):** `user.tsx:135-150` `uploadImages([toUploadInput(avatarFile)], {scope:'profile', outputFormat:'png-passthrough'})`; `keys[0]` → `profileImageKey` string → `updateUser` (`:163-168`).
- **Profile (OAuth first-login):** `authService.ts:79-83` `uploadImages([toUploadInput(photoFile)], {scope:'profile', outputFormat:'png-passthrough'})`; `keys[0]` → `profileImageKey` in FirestoreUser → `setDoc` (`:94,99`); upload error swallowed so user-create proceeds (`:84-86`).
- **Chat:** `MessageInput.tsx:114-129` `uploadImages(images.map(toUploadInput), {scope:'chat', chatId, signal, onProgress})`; keys → `ImagesMessageBlock.imageKeys` (`:144-145`) → `onSend({blocks})` → `useActiveChatStore` persists to Firestore `chats/{chatId}/messages` (`:367,398`). Keys are full `private/chats/{chatId}/<uuid>` prefixed keys. Display resolves private URLs via the view-token store — `MessageImages.tsx:31-33` (`getToken(chatId)` → `privateImageUrl`).

## V8 — View-token store conformance · PASS

`viewTokens.ts`: per-chatId cache `tokens: Record<string, CachedEntry>` (`:28`); `PREEXPIRY_BUFFER_MS=60_000` with hit-guard `cached.expiresAt > Date.now() + buffer` (`:35,49-50`); in-flight dedupe via `inFlight[chatId]` promise, pruned on success and failure so failures aren't cached (`:57-58,64,72,78`); `invalidate(chatId)` (`:83-85`); `clear()` resets both maps (`:87-89`). Source `BACKEND_API.post('/secure/images/view-tokens', {scope:'chat', chatId})` with finite-`expiresAt` validation (`:92-107`). **Logout wiring:** `useViewTokenStore.getState().clear()` inside `authStore.ts` `logout` (`:153`, import `:14`). Consumer `MessageImages.tsx:31-33` resolves URLs; `<Image onError>` → `handleImageError` invalidates + bumps `retryAttempt`, capped at one (`:49-53,66`) — the 401-invisible-on-`<Image>` workaround.

## V9 — DELETE cleanup path · PARTIAL

- `deleteImageKey` → `BACKEND_API.delete('/secure/images/${encodeKeyPath(key)}')` (`imageTokensService.ts:82-84`); `encodeKeyPath` = `key.split('/').map(encodeURIComponent).join('/')` — **slashes preserved** so backend `{*key}` matches (`:100-102`); failures `logServiceWarn`-logged, never re-thrown (`:85-87`). `cleanupOrphanImages` no-ops on empty, `Promise.allSettled` so failures can't reject (`uploadImages.ts:333-336`). Tests confirm segment-wise path + 404→no-throw (`imageTokensService.test.ts:151-160`).
- **Wired on save-failure (correct):** product create/update `productService.ts:86-94` (`catch { if (newKeys.length) void cleanupOrphanImages(newKeys); throw }`); review `reviewService.ts:113-120`; internal mid-batch failure `uploadImages.ts:150`.
- **NOT wired (gap):**
  - **Avatar** — `user.tsx:128-186`: avatar uploads to R2 (`:135-149`), `profileImageKey` set, then `updateUser` (`:163`). If `updateUser` returns falsy (`!result` handled at `:180`) or throws, the uploaded key is **never DELETE'd**. The only catch (`:151-159`) covers the upload itself.
  - **Chat** — `MessageInput.tsx:151` hands keys to `onSend` then immediately clears local state (`:152-153`); persistence is `useActiveChatStore.sendMessage` → `addDoc` (`:398`). On `addDoc`/`setDoc` failure the store catch (`:452-464`) only prunes the optimistic message — **no `cleanupOrphanImages`** (grep of `useActiveChatStore.ts` for cleanup/orphan/deleteImage → none).

Web cleans up both surfaces (avatar `user/page.tsx:241`, chat `useChatStore.ts:569`). Backend sweepers backstop (public/profiles daily ≤24h; private/chats only ≥30d on Sundays — so chat orphans linger longest), but the contract's Layer-1 per-surface client DELETE is missing on 2 of 4 surfaces.

## V10 — Config / env conformance · PASS

- `EXPO_PUBLIC_CDN_URL`: lazy `getCdnBase()` reads `process.env.EXPO_PUBLIC_CDN_URL`, throws if unset (`variants.ts:9-15`); sole CDN source for `publicImageUrl`/`privateImageUrl` (`:42-50`). Env files (gitignored but present on disk): `.env.development` = `https://cdn-stage.oglasino.com`, `.env.preview` = `https://cdn-stage.oglasino.com`, `.env.production` = `https://cdn.oglasino.com` — stage/stage/prod, correct.
- `EXPO_PUBLIC_WATERMARK_ENABLED`: `WATERMARK_ENABLED = process.env... === 'true'` (`variants.ts:1`); gates the hero `draw` param in `buildHeroParams` (`:19-34`), card variant ungated (`:38`); `=false` in all three env files.
- Issuance via `BACKEND_API` (base `EXPO_PUBLIC_API_URL`, `api.ts:22,33,152`); PUT target from token `uploadUrl` (`uploadImages.ts:249-254`) — no `getCdnBase`/`EXPO_PUBLIC_CDN_URL` in the upload path (grep clean). API base distinct from CDN base across all env files.

---

## What must be fixed before ship (not necessarily before smoke)

1. **V6 — `errorMapping.ts`:** add `'converting-heic': 'converting_heic'` to `STAGE_KEY_OVERRIDES` so the HEIC-conversion label resolves to the seeded key. (One line; highest-value finding.)
2. **V9 — avatar + chat orphan cleanup:** call `cleanupOrphanImages(newKeys)` on `updateUser` failure in `user.tsx`, and on `addDoc`/`setDoc` failure in `useActiveChatStore.sendMessage`.
3. **V4 — 429 policy:** decide whether the single retry-after retry is intended; if strict web parity is required, surface 429 immediately.

On-device behavior (the next gate — a real PUT landing in R2 against the staging Worker) is out of scope and not claimed here.

*End of validation. Read-only; no code changed; no fixes staged.*
