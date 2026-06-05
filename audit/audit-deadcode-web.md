# Audit — Dead / Unused Code (oglasino-web)

**Repo:** oglasino-web · **Branch:** dev · **Date:** 2026-06-05 · **Mode:** READ-ONLY (inventory, no fixes)

## Method

No dead-code analyzer is installed in the repo. For this audit I ran, read-only:

- `npx tsc --noEmit --noUnusedLocals --noUnusedParameters` (CLI flag override — no tsconfig edit) for dead locals/params.
- `npx knip@5` (runs from the npx cache; no repo files written, no config added) for unused files / exports / types / deps — knip has **no config here**, so it over-reports (vitest tests treated as unused files, namespace-imported icons treated as unused, locally-used-but-exported symbols treated as unused). **Every knip candidate below was re-verified by hand with `grep`** (exact import paths, in-file usage counts, namespace-lookup mechanism). The verification overturned several knip calls (see "knip false positives" notes).

Tiering follows the brief: **Tier 1** = exported/declared, zero importers/usages, not a Next.js special file, not dynamically referenced. **Tier 2** = looks dead but has a catch (test-only, dynamic/string reference, internal-only use behind an unnecessary `export`, build-time string path, vendored kit). **Tier 3** = intentionally retained (comment / issues.md / decisions.md).

---

## TIER 1 — safe to delete

Each verified: zero real importers, zero non-definition usages, not a Next.js special file, not reachable dynamically.

### Orphan files (no importer anywhere)

| # | File | What it is | Evidence it's unused |
|---|------|-----------|----------------------|
| 1 | `src/lib/service/nextCalls/favoritesService.ts` | Next-call service module | `grep -rn "favoritesService"` across `src`/`app` → no importer (the live favorites service is `reactCalls`, a different file). |
| 2 | `src/components/popups/components/FilterSelector.tsx` | Filter-selector component | No exact importer of `…/components/FilterSelector`. The live components are `MultiOptionFilterSelector` / `SingleOptionFilterSelector` / `RangeFilterSelector` / `DateFilterSelector` (substring matches only — this base file is mounted by none of them). |
| 3 | `src/lib/types/configuration/RegexData.ts` | Type module | `grep -rln "RegexData"` (excluding self) → zero importers. |
| 4 | `src/lib/types/KeyLabelPair.ts` | `KeyLabelPair` type module | Duplicated: `src/lib/types/catalog/RegionAndCityDTO.ts:1` declares its **own** inline `export type KeyLabelPair` and uses that (`:7,:8`); it does not import this file. No other importer. (Also a Part 4b duplicate-type smell.) |
| 5 | `src/components/shadcn/ui/sonner.tsx` | Vendored shadcn `Toaster` wrapper | No importer. `app/[locale]/layout.tsx:14` and `src/lib/config/toast.tsx:9` import `Toaster`/`toast` from the **`sonner` npm package** directly, never this wrapper. |
| 6 | `src/components/shadcn/ui/textarea.tsx` | Vendored shadcn `Textarea` | No importer. The project's own `src/components/server/Textarea.tsx` does **not** import it; it's the live textarea. |
| 7 | `src/components/icons/dynamic/OtherIcon.tsx` | Dynamic SVG icon | **Not** re-exported from `src/components/icons/dynamic/index.ts`, and zero string references anywhere. `DynamicIcon` only resolves names present on the `index.ts` namespace (`Icons[name]`), so this file is unreachable. |
| 8 | `src/components/icons/dynamic/PressureRangeFilterIcon.tsx` | Dynamic SVG icon | Same as #7 — absent from `index.ts`, no string ref. |
| 9 | `src/components/icons/dynamic/WeightFilterIcon.tsx` | Dynamic SVG icon | Same as #7 — absent from `index.ts`, no string ref. |

> **Catch to note for #7–#9:** `index.ts` is `// AUTO-GENERATED` (by `scripts/convert-icons.js`). These three files exist on disk but were not emitted into the generated index. Deleting them is safe today; if the icon generator is re-run from a source set that still lists them, they could reappear. Worth confirming with the icon source/manifest before deleting, but as the tree stands they are unreachable dead files.

### Dead exported symbols (declared, zero usages incl. own file)

| # | Symbol | Location | What it is | Evidence |
|---|--------|----------|-----------|----------|
| 10 | `getColorForLetter` | `src/components/client/buttons/AuthUserProfileButton.tsx:34` | Avatar-color helper | Only occurrence in the file is the definition; never imported. It is a **duplicate** of the live `getColorForLetter` in `src/components/server/OglasinoAvatar.tsx` (which uses its own copy). This button's copy is dead. |
| 11 | `useConfig` | `src/configuration/ConfigProvider.tsx:18` | React context hook | Never called. Only in-file occurrences are the declaration and its own `throw` message. `ConfigProvider` itself **is** mounted (`app/layout.tsx`), but nothing consumes the hook. |
| 12 | `pathnameWithoutLocale` | `src/lib/utils/utils.ts:10` | Path helper | Definition only; zero references. Sibling of the already-removed `isAllowedPath` path-util family. |
| 13 | `storage` | `src/lib/config/firebaseClient.ts:34` | `getStorage(app)` Firebase Storage instance | Never imported (the `storage` hits elsewhere are unrelated local identifiers in `appPromo/storage.ts` etc.). Consistent with the W1 removal of the web Firestore/storage path (issues.md 2026-06-02). Removing it also orphans the `getStorage` import on line 10. |
| 14 | `getDashboardProductDetails` | `src/lib/service/nextCalls/productsSearchService.ts:68` | Thin wrapper over `getProductDetails(…, 'owner')` | No importer of the **nextCalls** version. `app/[locale]/owner/products/[productId]/page.tsx:34` imports the same-named function from **reactCalls**, not this one. Not used internally either. |
| 15 | `getAdminProductDetails` | `src/lib/service/nextCalls/productsSearchService.ts:72` | Thin wrapper over `getProductDetails(…, 'admin')` | No importer of the nextCalls version. `app/[locale]/admin/products/product/[productId]/page.tsx:13` imports the **reactCalls** same-named function. Not used internally. |
| 16 | `letItStay` | `src/lib/data/productCardStyles.ts:1` | Style-map object (small/medium/large) | Definition only; zero references. It is an abandoned earlier twin of the live `productCardStyles` in the same file. ⚠️ The name `letItStay` reads like a deliberate "keep me" marker — but there is **no comment or decisions.md entry** retaining it, so per the brief it is not Tier 3. Recommend Igor confirm before deletion. |

### Dead exported types

| # | Type | Location | Evidence |
|---|------|----------|----------|
| 17 | `ReportDialogProps` | `src/components/popups/dialogs/ReportDialog.tsx:25` | Definition only; not imported and **not even used as the component's props** — `ReportDialog` destructures inline (`isOpen`, `onClose`, … which the type doesn't list). Stale/out-of-sync type. |
| 18 | `CardStylings` (plural) | `src/lib/types/ui/CardStyling.ts:1` | Definition only. The five product cards import the **singular** sibling `CardStyling` from the same file; `CardStylings` (plural wrapper) is consumed by nothing. File stays (singular is live); only this type is dead. |

### Dead local var

| # | Item | Location | Evidence |
|---|------|----------|----------|
| 19 | unused map param `v` | `app/[locale]/(portal)/(public)/blog/free-zone/page.tsx:99` | `Array.from({ length: 9 }).map((v, i) => …)` — `v` unused (placeholder array). tsc `TS6133`. Trivial. |

---

## TIER 2 — looks dead, has a catch (do NOT delete)

### Dynamically referenced (the big one)

- **The entire dynamic-icon family** (~135 re-exports in `src/components/icons/dynamic/index.ts` plus each icon file's `default` export — the bulk of knip's 332 "unused exports"). **Catch:** consumed via namespace + string lookup in `src/components/server/DynamicIcon.tsx` — `import * as Icons` then `Icons[name]`, where `name` is a backend-driven icon key (category/filter `labelKey`). knip cannot see member access on a namespace import. **Not dead.** (The three icons in Tier 1 #7–#9 are the exception: they're not in `index.ts`, so the namespace never carries them.)

### Internal-only use behind an unnecessary `export` (symbol is live; only the `export` keyword is redundant)

These are flagged by knip because nothing **imports** them, but each is used **within its own file**. Not dead code.

| Symbol | Location | Internal use |
|--------|----------|-------------|
| `serializeJsonLd` | `src/components/server/seo/JsonLd.tsx:11` | Used by `JsonLd` internally; also covered by `JsonLd.test.ts` (the security-fix escaping function — Brief 3). |
| `getMessagingInstance` | `src/notifications/lib/messaging.ts:8` | Called internally at `:22`. |
| `ensureUserInFirestore` | `src/lib/service/reactCalls/authService.ts:43` | Called internally 4× (`:170,180,185,190`) by login/register flows (does avatar sync — still live despite the W1 Firestore token-write removal). |
| `getProductDetails`, `getProducts` | `src/lib/service/nextCalls/productsSearchService.ts:25,76` | Base helpers behind the live `getPortal*/getDashboard*/getAdmin*` wrappers in the same file. |
| `getCookie`, `setCookie`, `updateCookie` | `src/lib/service/oglasinoCookies.ts:30,43,56` | Internal helpers behind the live `get/update/clear/readGlobalCookie*` API consumed across the app. |
| `CONDITION_TO_SCHEMA_ORG`, `AVAILABILITY_TO_SCHEMA_ORG` | `src/metadata/schemaOrgMappings.ts:17,27` | Read by the live `conditionToSchemaOrg` / `availabilityToSchemaOrg` in the same file. |
| `errorCodeToTranslationKey`, `safeT`, `ProcessingError` | `src/lib/images/errorMapping.ts:28,115`, `processImage.ts:45` | Used internally (and in tests). |
| ~35 exported types | (list below) | Used locally as the props / return / store type of their own module; only the `export` is unnecessary. |

The ~35 over-exported types: `PaginatedTableProps`, `SelectOption`, `CategoryToggles`, `ClockFn`, `FilteredProductListProps` (×2 — `FilteredProductList.tsx:10`, `SelectableFilterProductListWrapper.tsx:22`; both components are live), `RatingProps`, `UserDetailsShareProduct`, `MetaDataProductData`, `ClearAndWarmupResult`, `StartReindexResult`, `ResolveReportRequest`, `ApproveDisapproveReview`, `ErrorBoundary`, `FetchOptions`, `ApiResponse`, `FetchBackendApi`, `NotificationToastData`, `CompanyNavigation`, `ImageErrorTranslationKey`, `StageInfo`, `PreparePickerFilesOptions`, `ProcessingErrorCode`, `ProcessOptions`, `UploadScope`, `ImageVariant`, `MetadataProductSnapshot`, `ConfirmResetResult`, `AuthState`, `BaseSiteStore`, `ConsentStore`, `SelectedRangeData`, `LinkifyToken`, `ProductMetadata`, `HreflangMode`, `TenantLocale`. (`UploadFileStage` in `uploadImages.ts:54` is genuinely imported elsewhere — fully live.)

### Test-only (production-dead, kept alive solely by a unit test — flag the pair)

| Symbol | Location | Catch |
|--------|----------|-------|
| `deleteOgConsent` | `src/lib/consent/cookie.ts:47` | No production caller; only `cookie.test.ts` imports/exercises it. Either it's a still-wanted part of the consent API or both it and its test block are removable — Igor's call. |
| `stepForField` | `src/lib/utils/productStepMapping.ts:23` | Only `productStepMapping.test.ts` consumes it (the live consumer is the sibling map, not this fn). |

### Vendored shadcn UI kit — unused sub-exports (~45)

`badgeVariants`, `BreadcrumbLink/Page/Ellipsis`, `CardHeader/Footer/Title/Action/Description`, `CommandDialog/Shortcut/Separator`, `DialogClose/Overlay/Portal/Trigger`, `Drawer*`, `DropdownMenu*` (Portal/Checkbox/Radio*/Shortcut/Sub*), `PaginationEllipsis`, `PopoverAnchor`, `SelectScroll*Button/Separator`, `Sheet*`, `Sidebar*`, `TableCaption`, `tabsListVariants`, etc. **Catch:** these live in `src/components/shadcn/ui/*` — a vendored component kit kept whole by convention (the parts are generated as a set; the used members import fine). Trimmable in principle, but this is a library-completeness pattern, not accidental dead code. Recommend leaving unless Igor wants a deliberate shadcn prune. (`sonner.tsx` and `textarea.tsx` are the exception — whole files with **no** consumed member → Tier 1 #5/#6.)

### Build-time / tooling string references (knip can't see them)

| Item | Catch |
|------|-------|
| `public/firebase-messaging-sw.template.js` | Consumed by `scripts/build-firebase-sw.mjs` (the `build:sw` / `predev` / `prebuild` script) via a string path, not an import. Live build template. (Also currently Modified in git — actively worked on.) |
| `scripts/convert-icons.js`, `scripts/generate-logo.js` | Standalone dev tooling run manually (not wired into `package.json` scripts). `convert-icons.js` generates the `dynamic/index.ts` icon barrel. Intentional tooling, not app dead code. |

### ~30 `.test.ts` files flagged "unused files"

knip isn't configured for vitest, so it lists every test file as an unused file (`*.test.ts` under `src/`). They are run by `vitest run` (`npm test`). **Not dead.** Caveat: a test bound to a Tier-1-dead module would itself go dead with it — but none of the Tier 1 items above has a dedicated test except the two test-only items already called out (`deleteOgConsent`, `stepForField`).

### Unused function parameters (signature-positional)

`generateCatalogPageMetadata` params `searchParams`, `baseSite` (`src/metadata/generateCatalogPageMetadata.ts:11,12`) — unused, but they're positional params 1–2 while params 3–7 are used; removing them shifts the call signature. Low-value cleanup, left as-is. (tsc `TS6133`.)

### Dependencies knip flagged that are actually used (false positives — keep)

- `sharp` — Next.js's runtime image-optimization dependency; used by `next/image` in production, never imported in app code.
- `tailwind-scrollbar-hide` — Tailwind plugin; provides the `scrollbar-hide` utility used in `app/globals.css`.
- `typescript-eslint` (devDep) — referenced by `eslint.config.mjs` (flat config).

---

## TIER 3 — intentionally kept (excluded from deletion)

| Item | Why retained |
|------|--------------|
| `src/components/icons/FacebookIcon.tsx` | Orphaned Facebook sign-in scaffolding, deliberately retained for future Facebook login. issues.md 2026-06-01 "Orphaned Facebook sign-in scaffolding"; **wontfix 2026-06-04 (Igor)**. knip lists it as an unused file; it is intentional. |

---

## Brief-specific confirmations

- **`isAllowedPath` stragglers** — `grep -rn "isAllowedPath" src app` → **none**. Confirmed fully gone.
- **FilterManager.tsx exhaustive-deps warnings** — known/out-of-scope; **not** flagged here.
- **Notification `shown`-style dead type member (web side)** — **not present**. `grep -rn "shown" src/notifications` returns only an unrelated local `shownIds: Set<string>` toast-dedup in `notificationManager.ts:10,49,51,68`; the web `AppNotification` type (`src/notifications/types/AppNotification.ts`) declares no `shown` member. This matches issues.md (2026-06-02 carry-forward, closed 2026-06-04): the dead `shown` member is an **expo-only** residue, out of scope for this repo. Nothing to delete web-side.
- **Unused i18n / translation keys** — **skipped (not cheaply determinable)**, per the brief's escape hatch. Web has no local message catalog; keys are seeded from the backend SQL and consumed dynamically (`t(key)`, plus backend-driven icon/`labelKey` strings), so a static web-side "unused key" set can't be derived without the backend seed + runtime key set. Out of cheap reach.
- **Dead CSS modules / style files** — none. The project uses Tailwind + a single `app/globals.css`; there are no `*.module.css` files. The one custom utility checked (`scrollbar-hide`) is in use.

---

## Dependencies genuinely unused (adjacent — not core dead *code*, lower priority)

Verified zero references across `src`/`app`:

- `@radix-ui/react-accordion` — no accordion component exists in `shadcn/ui`, 0 refs.
- `@tanstack/react-virtual` — no `useVirtualizer`/virtualization usage, 0 refs.
- `react-syntax-highlighter` — 0 refs.
- devDep `@types/file-saver` — `file-saver`/`saveAs` 0 refs.
- devDep `@types/js-cookie` — `js-cookie` 0 refs.

(Listed for completeness; dependency removal is a `package.json` change outside this read-only audit's remit and the brief's code-focus. Flagged for Mastermind to route if wanted.)

---

## Summary counts

- **Tier 1 (safe to delete):** 19 items — 9 orphan files, 7 dead exported symbols, 2 dead exported types, 1 dead local var.
- **Tier 2 (looks dead, has a catch):** the dynamic-icon family (~135 + defaults), ~45 vendored shadcn sub-exports, ~35 over-exported-but-locally-used types, 8 internal-only over-exported symbols, 2 test-only symbols, ~30 vitest test files, 3 build/tooling files, 2 unused params, 3 false-positive deps.
- **Tier 3 (intentionally kept):** 1 (`FacebookIcon.tsx`).
- **Adjacent:** 5 genuinely-unused npm dependencies (3 deps + 2 `@types`).
