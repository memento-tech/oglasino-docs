# Audit — image pipeline, mobile current state (`oglasino-expo`)

**Type:** read-only audit. No code changed.
**Date:** 2026-05-30
**Branch:** `new-expo-dev` (as checked out; not switched)
**Feature:** [image-pipeline](../../oglasino-docs/features/image-pipeline.md) — `web-stable`, contract frozen.
**Method:** every claim verified against current code (file:line). The pre-existing `jobs/image_pipeline/IMAGE-PIPELINE-RN-AUDIT.md` (2026-05-08) was consulted as a hint only; per-claim staleness verdicts are at the end of each section.

---

## ⚠️ Headline — the brief's premise has flipped

**The token-based image pipeline is already implemented on mobile.** This is not a greenfield surface to scope; a complete implementation matching the frozen contract is already **committed** on `new-expo-dev` — landed via `memento-tech/feature/image-pipeline-v2` (PR #1) across phase-1–10 commits (`e807dbb` sync → `51ebe3b` upload rewrite → `52a6fc1` view-token store → `606ee34` processing UX → docs phases 8–10), with the squash commit `016da95` labelled *"changes by claude for image-pipeline, not fully tested."* This work landed **outside** the current Mastermind/Docs-QA orchestration (the older `jobs/image_pipeline/` flow), which is why `state.md` and the Expo backlog table still record image-pipeline mobile status as `not-started` (stale — correction drafted in the session summary). The legacy `cloudflareService.ts` (multipart / retired `/direct-upload` endpoints) is **gone**, replaced by:

- `src/lib/images/` — `processImage.ts`, `uploadImages.ts`, `uploadPrimitive.ts`, `variants.ts`, `errorMapping.ts`, `preparePickerAssets.ts`, `uploadInput.ts` (each with a co-located `*.test.ts`).
- `src/lib/services/imageTokensService.ts` — token issuance + delete.
- `src/lib/stores/viewTokens.ts`, `src/lib/stores/uploadProgress.ts` (each with `*.test.ts`).
- `src/components/images/ImageStatusOverlay.tsx` — processing-stage UX.
- `app/__smoke__/upload.tsx` — a real-device upload-primitive smoke harness (self-marked for deletion).

No retired endpoint (`/secure/direct-upload`, `/direct-upload-batch`, `/view-token`) is referenced anywhere in `src/` (grep clean). The display layer is on `expo-image`, not RN's built-in `Image`.

**Consequence for scoping:** the next phase is **not** "build the pipeline." It is verification, gap-closing, and cleanup of an existing implementation. The single most useful action for whoever owns the implementation chat is to treat the prior `IMAGE-PIPELINE-RN-AUDIT.md` as fully obsolete and start from this document. See §10 for the genuine remaining work.

---

## 1. Existing image handling — what's there today

### Upload — one shared, token-based orchestrator

`uploadImages(files, options)` in `src/lib/images/uploadImages.ts:80` is the single entry point used by every surface. Flow:

1. **Issue tokens** — `fetchUploadTokens` (`src/lib/services/imageTokensService.ts:52`) → `POST /secure/images/upload-tokens`, **15 s** per-call timeout override (`imageTokensService.ts:7,56`). Request `{scope, count, contentTypes, chatId?}` (`:18`); response `{tokens:[{token, key, uploadUrl, expiresAt}]}` (`:11,25`).
2. **Process each file** — `processImageForUpload` (`src/lib/images/processImage.ts:81`): resize longest side → 2400 px, JPEG re-encode at quality 0.85 with a 0.75 fallback; PNG path is single-pass lossless (`outputFormat: 'png-passthrough'`).
3. **PUT raw bytes** to the token's `uploadUrl` — `uploadBytes` (`src/lib/images/uploadPrimitive.ts:23`) via `expo-file-system/legacy` `FileSystemUploadType.BINARY_CONTENT` (raw bytes, **not** multipart). Headers `x-upload-token` + `Content-Type` set to the processed content type (`uploadImages.ts:252-255`).
4. **Retry** — `putWithRetry` (`uploadImages.ts:229`): 401 `TOKEN_EXPIRED` → single token re-issue; 429 → respect retry-after; 5xx → backoff `[1s, 4s]`; network → 1 retry.
5. **Fail-fast batch** — `Promise.allSettled` (`uploadImages.ts:134`); on any failure the already-uploaded keys are pruned via `cleanupOrphanImages` → `deleteImageKey` → `DELETE /secure/images/{key}` (`imageTokensService.ts:82-84`, path segment-encoded at `encodeKeyPath:100`).

`uploadImages` returns `string[]` of **full image keys** (not URLs). Services forward those keys to the backend.

Per-surface entry points:
- **Product:** `uploadProduct` (`src/lib/services/productService.ts:42`), scope `'product'` (`:67`) → `POST /secure/products/create|update` with `imageKeys`.
- **Review:** `reviewProduct` (`src/lib/services/reviewService.ts:70`), scope `'product'` (`:98`, reviews share the products prefix) → `POST /secure/review/product` (`:114`).
- **Profile (manual):** `app/owner/dashboard/user.tsx:135` → `uploadImages([toUploadInput(avatarFile)], { scope: 'profile' })`.
- **Profile (OAuth first-login):** `src/lib/services/authService.ts:79` → `uploadImages([...], { scope: 'profile', outputFormat: 'png-passthrough' })` inside `ensureUserInFirestore`.
- **Chat:** `src/components/messages/MessageInput.tsx:114` → `uploadImages(images.map(toUploadInput), { scope: 'chat', chatId, signal, onProgress })`.

`ImageScope = 'product' | 'profile' | 'chat' | 'report'` (`imageTokensService.ts:9`).

### Display — `expo-image` + CDN URL builder

Display uses **`expo-image`** (`import { Image } from 'expo-image'`) across ~16 components (ImagesImport, MessageImages, OglasinoAvatar, ProductTopImage, ProductReview, ZoomableImage, ImagesCarousel, …). RN's built-in `Image` is not used on these surfaces.

URL construction in `src/lib/images/variants.ts`:
- `publicImageUrl(key, variant)` (`:42`) → `${CDN}/cdn-cgi/image/<params>/<key>`; variants `card` (400×300), `hero` (1600×1200, optional watermark `draw`), `original` passthrough.
- `privateImageUrl(key, viewToken)` (`:49`) → `${CDN}/<key>?token=<viewToken>` for chat images.
- CDN base read lazily from `process.env.EXPO_PUBLIC_CDN_URL` (`getCdnBase`, `:9`), throws if unset.

### Token flow vs older mechanism

**Fully token-based, matching the frozen contract.** Public reads use Cloudflare image-resizing variant URLs; private chat reads use `POST /secure/images/view-tokens` (via the store, §7) + tokenized URL. **No Firebase Storage, no multipart, no retired endpoints.**

**Stale-audit reconciliation (§2 of the 2026-05-08 audit):**
| Prior claim | Verdict |
|---|---|
| `cloudflareService.ts` calls `/direct-upload`, `/direct-upload-batch`, `/view-token` | **CHANGED** — file deleted; endpoints are now `/secure/images/upload-tokens`, `/secure/images/view-tokens`, `DELETE /secure/images/{key}`. |
| PUT uses FormData `{uri,name,type}` | **CHANGED** — raw bytes via `BINARY_CONTENT` (`uploadPrimitive.ts:32`). |
| Display uses built-in RN `Image`; `expo-image` present-but-unused | **CHANGED** — `expo-image` is the primary display lib. |
| Keys stored as bare UUIDs; `window.location.origin` bug in `productService` | **CHANGED** — full keys with prefix; `productService.ts` rewritten, no `window` reference. |

---

## 2. The image surfaces — do they exist?

| Surface | Exists today | File(s) |
|---|---|---|
| Product create/edit + upload | **Yes** | UI `src/components/ImagesImport.tsx`; persist `productService.uploadProduct` (`productService.ts:42`). |
| Profile / avatar upload | **Yes** | UI `src/components/dashboard/components/AvatarUpload.tsx`; wired `app/owner/dashboard/user.tsx:135` (manual) + `authService.ts:79` (OAuth). |
| Chat attachment send + display | **Yes** | Send `src/components/messages/MessageInput.tsx` (`pickImages:62`, `send:97`); display `src/components/messages/MessageImages.tsx`. |
| **Review-with-images** | **Yes — IN SCOPE** | UI `src/components/dialog/dialogs/ProductReviewDialog.tsx` renders `ImagesImport` (`:1,192`) then calls `reviewProduct({images, imageKeys, …})` (`:97`). The `review` scope is a **live mobile upload surface**, not a no-op. |

**Admin:** `app/admin/` is gone (admin-removal chat α shipped). No remaining admin image surface. **CONFIRMED.**

**Orphan / dead-code flag:** `src/components/dialog/components/ProductReviewImageImport.tsx` exists but is **never imported** anywhere (grep: only doc-comment mentions in `ImageStatusOverlay.tsx:10` and `uploadProgress.ts:2`). The live review dialog uses `ImagesImport` instead. The orphan carries a `//TODO TODO` (`:21`) and hardcoded Serbian strings. **Deletion candidate** if the review surface is touched in the implementation chat (raised in §10 / session summary, not actioned here).

---

## 3. Image picking + OS-level format — the HEIC question

**Picker:** `expo-image-picker` `~17.0.10` (`package.json`). The only picker; `react-native-image-picker` is absent.

**Call-site options:**
| Surface | file:line | Options |
|---|---|---|
| Product camera | `ImagesImport.tsx:100` | `{ quality: 0.8 }` |
| Product gallery | `ImagesImport.tsx:110` | `{ quality: 0.8, allowsMultipleSelection: true }` |
| Avatar gallery | `AvatarUpload.tsx:46` | `{ mediaTypes: ['images'], allowsEditing: true, quality: 0.8 }` |
| Chat gallery | `MessageInput.tsx:65` | `{ allowsMultipleSelection: true, quality: 0.8, selectionLimit: 5, mediaTypes: MediaTypeOptions.Images }` |

No `format` option at any call site. (Minor inconsistency: `MessageInput` uses the deprecated `ImagePicker.MediaTypeOptions.Images` enum while others use the newer `'images'` string form — pre-existing nit.)

**Does the picker yield JPEG, or HEIC on iOS?** It does **not** reliably transcode HEIC. `quality` only compresses the saved output; without `allowsEditing`, iOS frequently hands back the original HEIC URI/mimeType. **The code already assumes this** — every picker result is passed through `preparePickerAssets` (`src/lib/images/preparePickerAssets.ts`), which detects HEIC by mimeType (`image/heic`/`image/heif`) **and** by uri/filename extension (`isHeicAsset:92`) and converts via `expo-image-manipulator` `manipulateAsync(uri, [], { compress: 0.95, format: SaveFormat.JPEG })` (`:65`), with a pass-through fast path when no HEIC is present (`:37`).

**Therefore: the web `heic2any` step is unnecessary/irrelevant on mobile.** Mobile already does HEIC→JPEG natively at the picker boundary via `expo-image-manipulator`. The `heic2any` name appears only in a comment describing web (`preparePickerAssets.ts:10-11`).

**Return shape:** `ImagePickerAsset`, aliased `OglasinoImage = ImagePickerAsset` (`src/lib/types/ui/OglasinoImage.ts:3`). **URI-based** (`file://` / `content://`), not base64 or Blob. `toUploadInput` (`src/lib/images/uploadInput.ts:12`) maps it to `ProcessImageInput { uri, width, height, fileName?, mimeType?, fileSize? }` (`processImage.ts:41`), deliberately dropping base64/exif/duration.

**Stale-audit reconciliation:** `quality:0.8` and `AvatarUpload allowsEditing:true` **CONFIRMED**. The 2026-05-08 audit's entire §4.1 "we need to *pick* a HEIC strategy / `expo-image-manipulator` not installed" is **CHANGED** — the strategy is implemented and the library is installed (§4).

---

## 4. Image-processing libraries already present

All image deps are **first-party Expo native modules already bundled in the existing dev build** — using them adds **no new native module** beyond the rebuild the dev build already entails.

| Package | Version | Used? | Where (proof) | Native? |
|---|---|---|---|---|
| `expo-image` | `~3.0.11` | **Yes** | `import { Image } from 'expo-image'` ×16 (e.g. `ImagesCarousel.tsx:5`) | Native (Expo, bundled) |
| `expo-image-manipulator` | `~14.0.8` | **Yes** | resize/encode `processImage.ts:2`; HEIC `preparePickerAssets.ts:1` | Native (Expo, bundled) |
| `expo-image-picker` | `~17.0.10` | **Yes** | pickers + types (`ImagesImport.tsx:1`, etc.) | Native (Expo, bundled) |
| `expo-file-system` | `~19.0.21` | **Yes** | file size (new `File` API, `processImage.ts:1`); upload (`/legacy`, `uploadPrimitive.ts:1`) | Native (Expo, bundled) |

**Absent (confirmed via grep):** `react-native-image-resizer`, `react-native-compressor`, `react-native-blob-util`, any standalone HEIC lib, `sharp`/`jimp`. No third-party native image module is present or needed.

**Native-module verdict:** the entire resize/compress/transcode/upload path is covered by first-party Expo modules already in the dev build. **No candidate library introduces a new native module.** This is the answer to the brief's Risk-Watch concern: the image pipeline does **not** add native-rebuild surface of its own.

**Stale-audit reconciliation:** prior audit said `expo-image-manipulator` was **not** installed and `expo-image` was unused — both **CHANGED**.

---

## 5. HTTP + auth plumbing

**HTTP client:** single axios instance `BACKEND_API`, created by `createApiInstance(BACKEND_API_URL)` in `src/lib/config/api.ts:152` (the only non-test `axios.create`, `:32`). Base URL `process.env.EXPO_PUBLIC_API_URL` (`:22`, throws if missing `:24`). Global timeout **8000 ms** (`:34`); image token calls override to 15 s (`imageTokensService.ts:7,56`).

**Firebase ID token attachment:** **CONFIRMED** — request interceptor (`api.ts:41-52`) reads `auth.currentUser`, sets `Authorization: Bearer <getIdToken()>` (`:48-49`). Any `/secure/images/*` call through `BACKEND_API` inherits this — and `imageTokensService` already does. The 401 response path does a single-flight force-refresh + retry (`api.ts:121`). `authStore` does not mint/store tokens; it only subscribes via `onIdTokenChanged`.

**Tenant / locale headers:** **CONFIRMED** — same interceptor sets `X-Base-Site` (`selectedBaseSite.code`) and `X-Lang` (`language.code`) (`api.ts:44-46`), read fresh per request from **`useBootStore.getState()`** (`:44`). No `getTenantLocale` helper — bootStore is the source (Φ3 boot-redesign). The backend doesn't require these on image endpoints, but image calls will carry them automatically.

**Raw PUT to the Worker (raw bytes + `x-upload-token`):** **a working pattern already exists** — `uploadBytes` in `src/lib/images/uploadPrimitive.ts` via `expo-file-system/legacy` `uploadAsync`/`createUploadTask`, `httpMethod:'PUT'` (`:31`), `uploadType: BINARY_CONTENT` (`:32`), body = `file://` URI (not Blob), headers passed straight through. The image PUT goes to `token.uploadUrl` directly, **bypassing** the `BACKEND_API` interceptor by design (carries `x-upload-token`, not `Authorization`). No generic Blob-to-arbitrary-URL pattern exists outside this path; `uploadBytes` is the only raw-binary mechanism. This resolves the 2026-05-08 audit's open risk §7.1 (Content-Type / BINARY mode) — it is implemented and smoke-harnessed (`app/__smoke__/upload.tsx`).

---

## 6. The Φ4 error-contract seam — CRITICAL

Read `src/lib/utils/parseServiceError.ts` in full (87 lines).

**What it does with an image-style `{"error":{code,message,details,retryable}}` body:** returns the empty neutral result. It only reads the plural `response.data.errors` array; for a singular `error` envelope `raw` is `undefined`, `!Array.isArray(raw)` is true, and it returns `{ errors: [], byField: {} }` (`:41,45-46`). The module docstring states this explicitly: *"imageTokensService reads a singular `data.error.code` envelope and keeps its own `ImageTokenError`"* (`:37-38`).

**Would routing image errors through `parseServiceError` silently swallow them?** **YES** — the `code` would never surface, nothing would throw, the screen would see an empty error set. So image errors must **not** go through `parseServiceError`. **The implementation already knows this** and keeps a parallel seam:

- `src/lib/images/errorMapping.ts` — `ERROR_CODE_TO_KEY` map (`:113`, ~25 Worker/processing codes → `ERRORS`-namespace keys), `errorCodeToTranslationKey(code)` (`:167`, defaults `image.upload.failed`), plus stage→key (`:83`) and English-fallback (`:196`) switches.
- `imageTokensService.ts` — `ImageTokenError` class (`:29`) carrying `code`/`status`/`retryAfterSec`; `imageTokenErrorFromResponse` (`:104`) / `imageTokenErrorFromError` (`:114`) read the singular `body.error.code`. `parseErrorBody` in `uploadImages.ts:378` does the same for the Worker PUT body.

**Is a code-keyed image error mapper net-new?** **No — it already exists** (`errorMapping.ts` + `ImageTokenError`). Separately, `src/lib/utils/authErrors.ts` `mapAuthError` keys off Firebase auth codes (different domain). For contrast, the Part-7 `{errors:[...]}` mapper is consumed by product/report/review flows (`preValidateOutcome.ts:56`, `updateSubmitOutcome.ts:72`, `UploadedProductDialog.tsx:129`, `ReportDialog.tsx:95`) — confirming image errors live on a deliberately separate seam.

---

## 7. View-token state + stores

**Store conventions:** Zustand v5 (`zustand ^5.0.11`). Two locations: `src/lib/store/` (singular — `authStore`, `bootStore`, chat/favorites/filter stores) and `src/lib/stores/` (plural — image-pipeline `viewTokens.ts`, `uploadProgress.ts`). Created via `create<State>()(...)`, persistence via `persist` + `createJSONStorage(() => AsyncStorage)` + `partialize` (`authStore.ts:1-4,256-264`). Reset methods (`clear()`/`reset()`) are wired into logout.

**A view-token store already exists and mirrors web** — `src/lib/stores/viewTokens.ts`:
- `useViewTokenStore` keyed per `chatId`: `tokens: Record<string, {token, expiresAt}>` (`:22-25,44`).
- Pre-expiry buffer: `PREEXPIRY_BUFFER_MS = 60_000` (`:35`); cache hit only if `expiresAt > now + buffer` (`:50`).
- In-flight dedupe: `inFlight: Record<string, Promise<string>>` (`:29,57-58,77-79`).
- `invalidate(chatId)` (`:83`) and `clear()` (`:87`).
- Source: `POST /secure/images/view-tokens` `{scope:'chat', chatId}`, 15 s timeout (`:92-102`).
- **`clear()` is called at logout** — `authStore.ts:153`, alongside the chat/user/favorites/notification/filter store clears.

**Existing chat-image consumer:** `src/components/messages/MessageImages.tsx` already consumes the store — `useViewTokenStore.getState().getToken(chatId)` then `imageKeys.map(key => privateImageUrl(key, token))` (`:27-42`); on `<Image onError>` it calls `invalidate(chatId)` and bumps a capped `retryAttempt` (`:49-53`). Local `useState` holds only the resolved URL array + retry counter, not the token.

**Stale-audit reconciliation:** the 2026-05-08 audit's "deferred view-token Zustand store" / "MessageImages uses per-component `useState` for the token" is **CHANGED** — the store exists and `MessageImages` uses it.

---

## 8. Translation key resolution

**Library:** `i18next ^25.10.10` + `react-i18next ^16.6.6` + `i18next-icu ^2.4.3` (ICU). **Init relocated by the boot-redesign** — `src/i18n/i18n.ts` no longer exists; init now lives in `src/lib/store/bootStore.ts` (`i18n.use(ICU).use(initReactI18next).init({...})`, `bootStore.ts:534-549`), with bundles registered incrementally via `i18n.addResourceBundle(...)` (`:550-553`) gated on checksum staleness. Config: `lng = stored active language`, `fallbackLng: 'sr'`, `ns: Object.values(TranslationNamespace)` (all namespaces), `defaultNS: COMMON_SYSTEM`.

**Fetch path:** `fetchNamespace(namespace, lang)` → `BACKEND_API.get('/public/translations?namespace=${ns}&lang=${lang}')` (`src/i18n/fetchNamespace.ts:31-33`), flattened then nested by dotted key. **Wire shape CONFIRMED** identical to the prior audit; only the orchestration moved (checksum-driven boot gate, not module-init).

**ERRORS / INPUT namespaces wired:** **CONFIRMED.** `TranslationNamespace` (`src/i18n/types.ts:6-38`) contains `ERRORS = 'ERRORS'` (`:10`) and `INPUT = 'INPUT'` (`:16`) among ~20 entries, all registered via `ns: Object.values(...)` (`bootStore.ts:541`) and the per-namespace fetch loop (`:451`). Resolver is the namespace-parameterized `useTranslations(namespace)` (`src/i18n/useTranslations.ts:3`), wrapping `useTranslation`. So the implementation can rely on backend-seeded `INPUT`/`ERRORS` keys (including the four that differ from spec — `image.processing.converting_heic`, `…complete.label`, `…uploading.label`, `…uploading.with.size`) without adding namespaces.

**Note (not blocking):** the local enum lists ~20 namespaces while the backend `/versions` set is described as 22 — staleness is driven by the local enum; a backend namespace absent from the local enum is simply not fetched. Does not affect ERRORS/INPUT.

---

## 9. Config / env

**Convention:** `EXPO_PUBLIC_*` vars read directly via `process.env.*` (Expo inlines them at bundle time from the active `.env.<profile>`). A non-public `APP_ENV` selects bundle id / GoogleServices / scheme / name in `app.config.ts:4`. Firebase config + EAS projectId are re-exposed under `extra.*` (`app.config.ts:71-85`) for runtime read via `expo-constants`. The CDN/API base is **not** in `extra` — read straight from `process.env`.

**Env-file reality (correcting the brief's premise):** `.env.development`, `.env.preview`, `.env.production` are **still present on disk** and remain the source of truth. `git status` shows `D .env.development`/`.env.production` because they were removed from the **git index** and are now git-ignored (`.gitignore:33-36`: `.env.*` with `!.env.example`), with a committed `.env.example` template. The mechanism (`.env.<profile>` + `process.env.EXPO_PUBLIC_*`) is unchanged — only their tracking status changed.

**CDN / image base env var:** **already exists and is fully wired** — `EXPO_PUBLIC_CDN_URL` (renamed from the prior audit's `EXPO_PUBLIC_WORKER_URL`, which no longer exists anywhere):
- `.env.development` / `.env.preview` = `https://cdn-stage.oglasino.com`; `.env.production` = `https://cdn.oglasino.com` (now a real value, no longer empty).
- Read in `variants.ts:10` (lazy `getCdnBase`, throws if unset). Exercised by `variants.test.ts`.
- Companion flag `EXPO_PUBLIC_WATERMARK_ENABLED=false` in all three env files.

**So `EXPO_PUBLIC_CDN_URL` is NOT net-new.** The upload PUT target is not built from an env base at all — it comes from the backend token response `uploadUrl` (absolute URL, `imageTokensService.ts:14`). Token issuance routes through `BACKEND_API` (base `EXPO_PUBLIC_API_URL`), not a CDN base.

**Stale-audit reconciliation:** prior audit §1/§5.2/§7.7 (`EXPO_PUBLIC_WORKER_URL`, empty prod env, "add `EXPO_PUBLIC_CDN_URL`") are all **CHANGED/STALE** — the var was renamed, prod is populated, and the env files moved to gitignored-but-present.

---

## 10. Seams + scoping summary

- **What copies across nearly unchanged (already done):** the entire portable set the brief anticipated — variant URL builders (`variants.ts`), error mapping (`errorMapping.ts`), the view-token store (`viewTokens.ts`) and upload-progress store (`uploadProgress.ts`), token DTOs/service (`imageTokensService.ts`). These already exist and mirror web's structure.
- **Genuine RN rewrites (already done):** the processing core (`processImage.ts` via `expo-image-manipulator` instead of canvas), HEIC handling (`preparePickerAssets.ts` instead of `heic2any`), and the raw-byte PUT (`uploadPrimitive.ts` via `expo-file-system` BINARY_CONTENT instead of `fetch`+Blob). All present and unit-tested.
- **Single biggest unknown / risk:** **not a code gap — a verification gap.** This implementation has unit tests and a smoke harness, but its on-device correctness against the staging Worker (`cdn-stage.oglasino.com`) is unverified here. It is **committed** on `new-expo-dev` (not working-tree-pending) but, per the headline, landed outside the Mastermind/Docs-QA orchestration — so `state.md` and the Expo backlog still record it as `not-started`, and no Phase-2 seam analysis or canonical-spec adoption ever ran against it. The risk is twofold: (a) the implementation chat assumes greenfield (per the stale prior audit) and rebuilds what exists; (b) governance/docs drift — the work was never reconciled against the frozen contract by this orchestration. The real risk is that the implementation chat assumes greenfield (per the stale prior audit) and rebuilds what exists. The orphan `ProductReviewImageImport.tsx` (dead code, §2) and the self-marked-for-deletion `app/__smoke__/upload.tsx` are cleanup loose ends.
- **Native-module impact:** **none from the image pipeline.** Every library used (`expo-image`, `expo-image-manipulator`, `expo-image-picker`, `expo-file-system`) is a first-party Expo native module already bundled in the dev build. The pipeline adds no native-rebuild surface of its own (it still rides the pending iOS+Android rebuild tracked in Risk Watch, but contributes nothing new to it).

---

## Net staleness verdict on the prior `IMAGE-PIPELINE-RN-AUDIT.md` (2026-05-08)

**Obsolete in its entirety as a description of current state.** It predates the full image-pipeline implementation, the Φ3 boot-redesign (i18n + tenant/locale moved to `bootStore`), the Φ4 error contract (`parseServiceError`), and the cloud-setup env rework. Its §6 "Decisions needed" are effectively all resolved in code (HEIC→manipulator, single-library processing, BINARY_CONTENT upload, dev build, shared backend translations). It remains useful only as a historical record of the migration's design intent. **Start the implementation chat from this document, not from it.**
