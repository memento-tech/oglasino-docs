# Diagnostic (Phase 2, narrowed) — Why `ImagesCarousel` paints but `ImagesImport` renders blank

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Feature slug:** product-update-parity
**Type:** READ-ONLY diagnostic. No code changed, nothing staged.
**Task (verbatim narrowing):** Same app, same CDN, same product, same keys — `ImagesCarousel` renders the product's images correctly on device, `ImagesImport` (update screen) draws image boxes but they render blank. Diff the two components and name whether it's URL-construction or render-props/styling.

> **This overturns the prior session's leading hypothesis.** Diagnostic `-1` (Q2) guessed cause **(a)**: `imageKeys` not arriving from the backend → `imagesData === []`. The new on-device evidence kills that: `ImagesImport` *draws boxes* (the main 350px frame + thumbnail strip only render when `images`/`visibleImage` are populated), and `ImagesCarousel` shows the very same product's images. So `imageKeys` **is** arriving and `imagesData` **is** populated. The defect is entirely render-side, inside `ImagesImport`.

---

## 1. What each component receives, and the key shape it consumes

| | `ImagesCarousel` | `ImagesImport` |
|---|---|---|
| Prop | `images?: string[]` (`ImagesCarousel.tsx:16`) — **already-built URL strings** | `images: ImageData[]` of `{ key?, file? }` (`ImagesImport.tsx:26`, `ImageData`) |
| Who builds the URL | **The caller.** Item is used verbatim as the URI. | **The component itself**, via `publicImageUrl`. |
| Live caller | Portal product page: `images={productDetails.imageKeys?.map((key) => publicImageUrl(key, 'hero')) ?? []}` (`app/(portal)/(public)/product/[...productData].tsx:219-220`) | Update screen: `images={productDetails.imagesData ?? []}` (`[productId].tsx:317`), where `imagesData = (imageKeys ?? []).map((key) => ({ key }))` |
| Key string right before URL build | the raw `imageKey` `K` → fed to `publicImageUrl(K, 'hero')` by the caller | the **same** raw `imageKey` `K` (as `img.key`) → fed to `publicImageUrl(K, 'card')` (`ImagesImport.tsx:130`, `:174`) |

**The key is byte-identical in both paths.** Both ultimately call the same `publicImageUrl` (`src/lib/images/variants.ts:42`) with the same `K`. No prefix handling, no path-vs-bare-key divergence, no double prefix.

## 2. The URL each builds for the same key `K`

`publicImageUrl(key, variant)` = `` `${CDN}/cdn-cgi/image/${VARIANT_PARAMS[variant]}/${key}` `` (`variants.ts:42-47`).

```
ImagesCarousel (portal, 'hero'):
  ${CDN}/cdn-cgi/image/width=1600,height=1200,fit=scale-down,format=auto,quality=85/K

ImagesImport (main image + thumbs, 'card'):
  ${CDN}/cdn-cgi/image/width=400,height=300,fit=cover,format=auto,quality=85/K
```

Side by side, the **only** difference is the variant params (`hero` vs `card`). Same host, same `/cdn-cgi/image/` transform path, same trailing `/K`. The `'card'` variant is **not** suspect: it is used successfully elsewhere on device — `ProductTopImage` (`ProductTopImage.tsx:45`, the dashboard/list cards) and the avatars (`OglasinoAvatar.tsx:44`) both render `publicImageUrl(key, 'card')`. So a `'card'` URL for a valid key paints fine in this app.

→ **URL construction is not the cause.** Both URLs are valid; the `'card'` variant renders elsewhere.

### 2a. Cloudflare resize-transform check

| | Goes through `/cdn-cgi/image/`? | Variant | Exact transform params |
|---|---|---|---|
| `ImagesCarousel` (portal, working) | **Yes** | `hero` | `width=1600,height=1200,fit=scale-down,format=auto,quality=85` |
| `ImagesImport` (blank) | **Yes** | `card` | `width=400,height=300,fit=cover,format=auto,quality=85` |

**Both go through Cloudflare's `/cdn-cgi/image/...` resize transform — neither points at the raw key.** This is structural, not coincidental: `publicImageUrl` (`variants.ts:42-47`) emits a bare `${base}/${key}` **only** for `variant === 'original'`; for every other variant (`card`, `hero`) it always wraps in `/cdn-cgi/image/${params}/${key}`. Neither callsite here passes `'original'`, so **both** rely on the resize transform. There is no "one resizes, the other doesn't" asymmetry.

Cloudflare image resizing is therefore **relied upon and proven working on device**:
- `ImagesCarousel`'s *working* URL is itself a `/cdn-cgi/image/` (hero) URL — so the resize path is live and functioning.
- The specific `card` transform that `ImagesImport` uses also resolves successfully elsewhere on device: `ProductTopImage.tsx:45` (dashboard/list cards) and `OglasinoAvatar.tsx:44` both call `publicImageUrl(key, 'card')` → the same `width=400,height=300,fit=cover,…` transform → and they paint.

→ **Cloudflare resize is ruled out as the cause.** Same transform path for both; both variants (incl. `card`) are confirmed working through `/cdn-cgi/image/` on device. The blank is downstream of a valid, resized URL — it is the render-side sizing (§3).

## 3. Which renderer, and the `source`/prop/sizing setup

Both use `expo-image`'s `Image`. `source` shape is identical (`source={{ uri }}`). The decisive divergence is **how the image is sized**:

| | `ImagesCarousel` (works) | `ImagesImport` (blank) |
|---|---|---|
| Dimensions via | **inline `style`** — `style={{ width: containerWidth, height: IMAGE_HEIGHT }}` (`ImagesCarousel.tsx:95-98`), concrete px from `onLayout` | **NativeWind `className`** — main: `className="h-[350px] w-full"` (`ImagesImport.tsx:133`); thumb: `className="h-full w-full"` (`:176`) |
| `source` | `{{ uri: item }}` (`:94`) | `{{ uri: img.key ? publicImageUrl(img.key,'card') : img.file?.uri }}` (`:128-132`, `:173-175`) |
| `contentFit` | `"contain"` | `"contain"` (main), `"cover"` (thumb) |
| `onError` | **yes** — `markFailed(item)` → shows a "failed to load" fallback (`:103`) | **none** on either Image |
| `recyclingKey` | `item` | thumb only (`:179`) |

**Why `className` sizing fails for `expo-image` in this project — two compounding reasons:**

1. **`expo-image` is not wired into NativeWind here.** The project registers `cssInterop` **only** for the icon component (`src/components/ui/icon.tsx:13`) and **never** for `expo-image`'s `Image`. Consistent with that, **every** working `expo-image` in the codebase sizes via inline `style`, never via `className` for layout: `ProductTopImage` `style={{ width:'100%', height:'100%' }}` (`:47`), `OglasinoAvatar` `style={{ width:size, height:size }}` (`:62-64`), `UserAvatar` `style={{ width:size, height:size }}` (`:18`), `ZoomableImage` `style={{ width, height }}` (`:165`), and `ImagesCarousel` itself (`:95`). `ImagesImport` is the **lone** product-image surface that puts its dimensions in `className`. With `className` dims not applying, the `Image` lays out at 0×0 → blank box.

2. **The thumbnail also has no height anchor even if `className` did apply.** Thumb `Image` is `h-full w-full` (`:176`) inside `<Pressable className="mx-1 w-20 ...">` (`:169`) — the Pressable sets width (`w-20` = 80px) but **no height**; its parent strip is `h-20` (`:154`) but the Pressable doesn't inherit that as a concrete height. `h-full` = 100% of an unsized parent = 0 → zero-height thumbnails regardless.

(`ImagesCarousel` survives even though it *also* has a `className` on its `Image` — but only `bg-background` (`:100`), cosmetic; its **dimensions live in `style`**, so it paints regardless of whether `className` is wired.)

## 4. Verdict

**(b) Same construction, valid URL, different render.** The two components build the same key into the same `publicImageUrl` path; the only URL delta is the `hero` vs `card` variant, and `card` is proven to render elsewhere on device. `ImagesImport` renders blank because it **sizes its `expo-image` via NativeWind `className` instead of inline `style`** — and in this project `expo-image` is not `cssInterop`-registered (only the icon is), so those `className` dimensions don't apply; the images collapse to zero size. The thumbnail compounds this with a `h-full` that has no height anchor.

### Both URLs for the same key (recap)
```
works (carousel/hero): ${CDN}/cdn-cgi/image/width=1600,height=1200,fit=scale-down,format=auto,quality=85/K
blank (import/card):    ${CDN}/cdn-cgi/image/width=400,height=300,fit=cover,format=auto,quality=85/K   ← URL fine; render is the problem
```

### Minimal fix location (do NOT fix here — read-only)
`src/components/ImagesImport.tsx`, the two `<Image>` instances:
- **Main image (`:127-136`)** — move dimensions to inline `style`, e.g. `style={{ width: '100%', height: 350 }}` (drop the `h-[350px] w-full` from `className`), matching `ProductTopImage`/`ImagesCarousel`.
- **Thumbnail (`:172-181`)** — give the `Image` a concrete `style={{ width: 80, height: 80 }}` (or set an explicit height on the `w-20` `Pressable`), so it no longer relies on `h-full`/`w-full` against an unsized parent.

No change to URL building, `publicImageUrl`, `hydrateImagesData`, or the backend is required. The optional robustness add — an `onError` fallback like `ImagesCarousel` has — would surface a true load failure instead of a silent blank, but it is **not** the cause here.

---

## Session summary (conventions Part 5)

**Repo:** oglasino-expo · **Branch:** new-expo-dev · **Date:** 2026-06-01
**Task:** Read-only narrowed diagnostic — why `ImagesCarousel` paints the same product's images but `ImagesImport` renders blank.

### Implemented
- Nothing. Read-only diagnostic; output is this document.

### Files touched
- None. **Read:** `src/components/ImagesCarousel.tsx`, `src/components/ImagesImport.tsx`, `src/lib/images/variants.ts`, `app/(portal)/(public)/product/[...productData].tsx`, `src/components/ui/icon.tsx`, `src/components/product/ProductTopImage.tsx`, `src/components/user/{OglasinoAvatar,UserAvatar}.tsx`, `src/components/{ZoomableImage,FullScreenImageViewer,messages/MessageImages}.tsx`, plus the prior `diagnose-product-update-parity-mobile.md`.

### Tests
- Not run — no code changed.

### Cleanup performed
- None needed (read-only).

### Obsoleted by this session
- Supersedes diagnostic `-1`'s Q2 leading hypothesis (cause **(a)**, "`imageKeys` not arriving"). New on-device evidence (boxes render) shows `imageKeys`/`imagesData` **are** populated; the defect is render-side sizing in `ImagesImport`, not the backend DTO. The Q2 "confirm `ProductForUpdateDTO.imageKeys`" backend seam is **no longer the blocker** for the blank-images symptom.

### Conventions check
- Part 4 (cleanliness): N/A to this session (no code changed). **Adjacent finding (Part 4b):** `ImagesImport.tsx:40` has a stray `console.log(images)` — debug logging that violates Part 4; flagged below, out of scope to remove in a read-only pass.
- Part 4a (simplicity): N/A — nothing added.
- Part 8 (routes reusable): confirmed — both image paths reuse `publicImageUrl`/the same CDN transform; no mobile-specific route.

### Config-file impact
- conventions.md / decisions.md / issues.md: no change (Docs/QA is sole writer).
- state.md: no change; no Expo backlog row flips from a read-only diagnostic.

### For Mastermind
- **Root cause (high, fix is mobile-local):** `ImagesImport` sizes its `expo-image` via NativeWind `className` (`h-[350px] w-full` / `h-full w-full`), but `expo-image` is not `cssInterop`-registered in this project (only `ui/icon.tsx` is) and every other `expo-image` sizes via inline `style`. Result: zero-size → blank. Fix = inline `style` dimensions on the two `<Image>`s in `ImagesImport.tsx` (`:127-136`, `:172-181`); thumbnail additionally needs a concrete height anchor. No backend/URL change.
- **Correction to feed forward:** the earlier "backend `ProductForUpdateDTO.imageKeys` may be dropped" concern is decoupled from this symptom — images arrive; they just don't paint. (Still fine to confirm the DTO independently, but it is not what blanks the update-screen images.)
- **Part 4b adjacent (low):** stray `console.log(images)` at `ImagesImport.tsx:40` — remove when the file is next touched.
- **Config-file dependency (closure gate):** none. No drafted config edits.
