# Session summary

**Repo:** oglasino-web
**Branch:** feature/qa-preparation
**Date:** 2026-05-23
**Task:** Author a new QA topic entry `product-creation-flow` (`type: 'flow'`) — Stop 1: audit and structural draft.

## Implemented

This is Stop 1 of a two-stop session. No code on disk has changed. `topics.ts` will be appended in Stop 2. Below: the audit of the create-wizard surfaces, the structural draft of the topic, and the 2–3 option proposals for the judgment sections.

## Audit — create wizard surfaces

### The wizard is a dialog, not a route

The portal create-wizard is **not** routed. It is a Drawer dialog registered as `createNewProductDialog` and opened through the central `DialogManager` (`src/components/popups/DialogManager.tsx:42`). The wizard component is `src/components/popups/dialogs/CreateNewProductDialog.tsx`. There is no `/create`, no `/owner/products/new`, and no separate page surface.

This affects how the topic should describe the "route": the primary entry surface is the create button in the portal/owner header, not a URL the tester can deep-link to. The dialog cannot be reached by typing a URL.

### Entry points (the brief asked for an enumeration; here is the verified list)

Every entry routes through one of three button components, all gated on `useAuthStore.user` and on a `user.baseSite` + `user.regionAndCity` precondition.

1. **`AuthAddNewProductButton` (CirclePlus icon)** — `src/components/client/buttons/AuthAddNewProductButton.tsx`. Mounted in:
   - **Portal header (desktop ≥ xl)** — via `HeaderNavButtons.tsx:58`, only when the auth store has a user.
   - **Owner dashboard `TopNavigation`** — via `src/components/client/secure/TopNavigation.tsx:31`, present when `allowProductCreation` is true. The owner layout sets it true (default); the admin layout sets it false (`app/[locale]/admin/layout.tsx:35`). So the icon is present on every `/owner/*` page and absent on every `/admin/*` page.
   - **Mobile footer bar** — via `src/components/server/layout/MobileFooterNavigation.tsx:38`. Authed branch only; logged-out viewers see a different button (`AddNewProductButton`) that opens the login dialog instead of the wizard.
2. **`AuthAddNewProductButton` (regular-button variant)** — `useRegularButton={true}` is passed on `app/[locale]/owner/products/page.tsx:51`. It is mounted on the empty-state of the owner products page only — appears as a text button beside the "you have no products" message.
3. **`JoinFreeZoneButton`** — `src/components/client/JoinFreeZoneButton.tsx`. The two "Join Free Zone" buttons (hero + footer) on `/[locale]/blog/free-zone` open the wizard for an authed user (logged-out branch opens `LOGIN_OPTIONS_DIALOG` instead). Logged-in label reads `create.free`, signed-out label reads `join.button.label`.

### Pre-condition gate: `UserBasicDataSelectorDialog`

If a logged-in user clicks any of the three entries above while either `user.baseSite` or `user.regionAndCity` is null, `AuthAddNewProductButton.openNewProductDialog` (lines 22–32) opens `USER_BASIC_DATA_SELECTOR_DIALOG` instead of the wizard, with `shouldOpenDialog: true`. After the user fills the base-site + city + region selectors and saves, `UserBasicDataSelectorDialog.onUpdate` (lines 87–89) chains into `openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG)`. The wizard cannot open until base-site and region/city are set.

The `JoinFreeZoneButton` does NOT route through this precondition — it opens the wizard directly. So a free-zone-entry click against a user missing base-site/region produces the wizard with `selectedBaseSite=undefined`, which renders nothing (the `if (!productData) return <></>;` short-circuit at `CreateNewProductDialog.tsx:196`).

### Four step components, in order

`CreateNewProductDialog` holds `currentStep` (0-indexed) and `productData` (the wizard's working `NewProductRequestDTO`). The four steps render in this order (`CreateNewProductDialog.tsx:135–194`):

1. **Step 1 — Images**: `src/components/popups/components/ImageSelectionProductDialog.tsx`. Multi-image picker (max 5). Client-side image checks (`validateImagesData` at `productValidator.ts:106`) cover MIME (jpeg/png/webp), size (≤ 5 MB), and SHA-256 duplicate detection within the picked set. The error key is lifted into the parent so it survives a back-step round-trip (`CreateNewProductDialog.tsx:45` + comment block at lines 41–44). The single button is "Next".
2. **Step 2 — Basic info**: `src/components/popups/components/BasicInfoProductDialog.tsx`. Renders the three-level CategorySelector (top → sub → final), name input (max 80), description textarea (max 2000) with an inline "AI suggest" affordance, and (conditionally) price + currency. CitySelector is rendered disabled, pre-filled from `user.regionAndCity`. The button row is Back + Next.
3. **Step 3 — Meta data**: `src/components/popups/components/MetaDataProductDialog.tsx`. Hosts `MetaDataProduct` (`src/components/owner/product/MetaDataProduct.tsx`) which materialises the category-specific filters (baseSite top-filters + topCategory.filters + subCategory.filters + finalCategory.filters, minus the `age` filter — see `MetaDataProduct.tsx:56`). The reCAPTCHA invisible challenge fires on the Next click (`MetaDataProductDialog.tsx:30–40`); the failure shows `tValidation('suspicion')` inline and stays on step 3.
4. **Step 4 — Submit / Result**: `src/components/popups/components/UploadedProductDialog.tsx`. This step auto-submits on mount: it uploads images to R2 via `uploadImages('product', …)`, then POSTs `/secure/products/create` via `createNewProduct` (`src/lib/service/reactCalls/productService.ts:123`). On success it shows the canonical product URL with Copy + Open Link + Close buttons. On validation failure it lists the inline ERRORS-namespace keys, and the "Go back and fix" button jumps to the step that owns the first failing field via `resolveTargetStep(validationErrors)` from `productStepMapping.ts`. On transport error it shows the generic "create failed" copy and the same two buttons.

The wizard's progress bar at the top renders `(steps[currentStep].no * 100) / steps.length` and is hidden on step 4 (`CreateNewProductDialog.tsx:208`). The dialog cannot be dismissed by clicking outside (`closableOutside={false}`, line 206) and has no X-close button (`addCloseButton={false}`, line 207) — the only way out is the step's own Close button (step 4 success) or Exit button (step 4 failure), or the browser's back button / route nav.

### Pre-validate wiring (step 2 → step 3 advance gate)

`BasicInfoProductDialog.onNextInternal` (`BasicInfoProductDialog.tsx:78–143`) is the gate. It does Zod structural validation first (name + description blank/length, category cascade complete, price required if `topCategory && !topCategory.freeZone`, image check); on any structural failure it sets `productErrors` and stays on step 2. On structural pass, it calls `preValidateProductBasics(name, description)` from `productService.ts:230`. The call posts `{ name, description }` to `/secure/products/pre-validate`.

The PreValidateResult union (`productService.ts:29–32`) decides what happens next:

- **`{ type: 'clean' }`** — `onNextStep()` advances to step 3.
- **`{ type: 'validation', errors }`** — inline errors land on step 2 in the same `ERRORS`-namespace keys the server emits (`product.name.banned_words`, `product.description.spammy`, etc.). If `errors[SYSTEM_ERROR_KEY]` is present (rate-limit case), the form sets `isRateLimited=true`, disables the Next button, and clears the system error after a fixed 5-second back-off (`RATE_LIMIT_BACKOFF_MS = 5000`, `BasicInfoProductDialog.tsx:39`). Note: the retry-after header is not consumed; the 5-second value is a constant.
- **`{ type: 'error' }`** — pre-validate is a UX optimization, not a security boundary. The branch shows `notify.warning` with `tDialog('new.product.pre.validate.warning')` and **advances** to step 3 (`BasicInfoProductDialog.tsx:138–142`). Step 4 is the real gate.

### Image handling — Model C confirmed in code

`productData.imagesData` is a `Partial<NewProductRequestDTO>` field initialised undefined and held in wizard state across steps 1–3. The picker's `ImagesImport` writes `ImageData[]` entries that carry either a `File` (new image) or a `key` (existing R2 object key — irrelevant on create, only used on the update page). No `uploadImages` call exists anywhere in steps 1–3 — confirmed by grep and by reading every step component.

The upload happens in step 4 only (`UploadedProductDialog.tsx:51–61`): `createNewProduct(productData, { onProgress })` calls `extractAndUploadImages` (`productService.ts:67–112`) which posts `/secure/images/get-keys/{type}` and then PUTs each file directly to R2 via the returned push token. The per-file progress event maps a stage (`idle` → processing stages → `uploading` → `complete` | `error`) into `useUploadProgressStore` (`src/lib/stores/uploadProgress.ts`).

### Step-4 progressive status messages

The brief reads: "The step-4 submit flow with progressive status messages ('Uploading images...' → 'Creating product...' → done) per Model C."

In code, step 4's loading affordance is a single generic `<LoadingOverlay />` (`UploadedProductDialog.tsx:120–126`) that renders the COMMON-namespace `loading` key — i.e. just "Loading…" with an animated dot trail. The per-file upload progress states (`uploading`, `processing`, `complete`) live in `useUploadProgressStore` and are rendered by the picker components (`ImagesImport`, `AvatarUpload`, `ProductReviewImageImport`) — not on step 4. Step 1 is unmounted when step 4 is active, so the picker's per-file state indicators are not visible during submit.

**Net:** the tester does NOT see distinct "Uploading images…" / "Creating product…" prose during step 4. They see a single generic loading overlay until either the success surface or the failure surface appears. This is a brief-vs-reality observation; flagged in "For Mastermind."

### Free-zone-category branch on price

Client side (`BasicInfoProductDialog.tsx:228`): the price + currency block renders only when `productData.topCategory && !productData.topCategory.freeZone`. When the user picks a free-zone top category, the price input vanishes from the form and `validateProduct` skips the price check (`productValidator.ts:76–82`).

Server side (`oglasino-backend/.../DefaultProductService.populateCategories` lines 686–703): `destination.setFree(topCategory.isFreeZone())` followed by `if (topCategory.isFreeZone()) destination.setPrice(BigDecimal.ZERO)`. The server forces price to zero for free-zone categories regardless of what the client sent. For non-free-zone categories, the server preserves the client-supplied price (`DefaultProductService.createProduct` line 125) — there is **no** server-side `PRICE_REQUIRED` check on the create path (the `PRICE_REQUIRED` enforcement at line 825 lives in `updatePriceCurrency`, which is the update path only). Today the client's Zod check is the only thing stopping a non-free-zone product from being created with `price=null`. Flagged in "For Mastermind."

### Error-rendering surface

Each step component reads the `errors[]` array out of `ProductErrorResponse` via `parseProductValidationErrors` (`src/lib/utils/parseProductValidationErrors.ts`). Per-field codes map to `product.<field>.<code>` translation keys in the `ERRORS` namespace. Cross-cutting codes (rate limit, system errors) land under the `SYSTEM_ERROR_KEY = '__system'` slot.

The unified wire shape `{ errors: [{ field, code, translationKey }] }` is what every error-bearing status returns (400, 403, 422, 429, 500) — `productService.ts:37` defines `PARSEABLE_ERROR_STATUSES = new Set([400, 403, 422, 429, 500])`. The client trusts the `translationKey` from the response when present, falls back to the lowercase code form when absent (the legacy-feature compatibility path).

### Translation namespaces touched

- `DIALOG` — wizard step copy (`new.product.image.suggestion`, `new.product.basic.suggestion`, `new.product.meta.suggestion`, button labels `new.product.image.step.forward` / `…step.back`, `new.product.success.*`, `new.product.create.failed.*`, the AI-suggest tooltip, the pre-validate warning toast).
- `INPUT` — field labels (`product.name`, `product.description`, `price.label`, `currency.select.placeholder`, `currency.select.header`).
- `ERRORS` — all per-field validation errors (`product.<field>.<code>`) including blank/length client codes and every server-emitted moderation code; cross-cutting codes through `__system`.
- `VALIDATION` — `form.incomplete` (advisory under the form), `suspicion` (reCAPTCHA failure copy on step 3).
- `BUTTONS` — `add.new.product.label`, `add.new.product.tooltip` (the entry-point button itself).
- `COMMON` — `loading` (the step-4 LoadingOverlay).
- `FREE_ZONE_PAGE` — `join.button.label`, `create.free` (the JoinFreeZoneButton entries).
- `DIALOG` — `base.site.select.required.title`, `base.site.update.description.*`, `base.site.select.label`, `base.site.select.region.label`, `base.site.select.update.button.label`, `button.close.label` (the pre-condition `UserBasicDataSelectorDialog` only).

## Structural draft of the topic entry

The non-judgment sections — id, type, route, optionsControls, howToUse, whatToExpect — are drafted below. Title, overview, non-known-issue pitfalls, qaChecklist, relatedTopics, and images each carry 2–3 proposals further down for Mastermind selection.

### id

`'product-creation-flow'`

### type

`'flow'`

### route (primary surface only, per decisions.md §5)

`'Portal/owner header — "Add new product" (+) button'`

The dialog is not URL-addressable. The other surfaces (mobile footer bar, owner-products empty state, free-zone-page Join Free Zone buttons) are enumerated in `optionsControls`.

### optionsControls (structural — drafted directly)

- 'Add-new-product button (CirclePlus icon) — the primary entry. Visible to signed-in users in the desktop portal header (xl+), in the owner-dashboard TopNavigation on every /owner/* page (absent on admin pages), and in the mobile footer bar at the bottom of the screen on phones and tablets. Hidden when signed out and on every /admin/* page.'
- 'Add-new-product text button on /[locale]/owner/products — the regular-button variant of the same control, rendered beside the "you have no products yet" empty-state message. Same click behaviour as the icon: opens the wizard for a complete user, opens the base-site/region picker first for an incomplete user.'
- '"Join Free Zone" buttons on /[locale]/blog/free-zone — hero and footer. For a signed-in user, both buttons open the same wizard. For a signed-out user they open the login options dialog instead. The button label flips between "Join" and "Create free" based on sign-in state.'
- 'Base-site + city pre-condition picker (UserBasicDataSelectorDialog) — opens automatically instead of the wizard when the signed-in user does not yet have a base site or a region/city set. After saving, the picker chains directly into the wizard. The free-zone entry buttons skip this picker (see Pitfalls).'
- 'Step indicator at the top of the wizard — a four-segment progress bar that fills 25% / 50% / 75% / 100% as the user advances. Hidden on the success/failure step. The wizard cannot be closed by clicking outside the dialog and has no X-close button — the only ways out before submit are the wizard\'s own back-step buttons and the browser back or navigation.'
- 'Back and Next buttons inside the step body — every step from 2 onward has a Back button to return to the previous step; every step from 1 to 3 has a Next button to advance. Step 1 has Next only (no back-step). Step 4 has no Back or Next — only the Close (success) or Go-back-and-fix / Exit (failure) controls.'

### howToUse (structural — drafted directly)

- 'Sign in. Open any page where the Add-new-product button is mounted (portal home, owner dashboard, mobile footer, owner products list, or the Free Zone page) and click the button. If your account is missing a base site or a region/city, fill the base-site/region picker that opens first and save — the wizard then opens automatically. (Note: opening the wizard from the Free Zone buttons skips the base-site/region picker, see Pitfalls.)'
- 'Step 1 — Images. Add up to five product images via the picker. The picker filters by MIME type (JPEG, PNG, WEBP), rejects files over 5 MB, and rejects duplicates of any already-picked image. Inline error wording appears under the picker if any check fails. Click Next when you have at least one image you want to keep.'
- 'Step 2 — Basic info. Pick the category cascade (top, then subcategory, then final), type a product name (≤ 80 chars) and a description (≤ 2000 chars), and — if the top category is not a free-zone category — enter a price and pick a currency from the base site\'s allowed list. The city is shown for context but cannot be edited (it comes from your account). An AI-suggest icon next to the name input fills the description for you when clicked.'
- 'Click Next on step 2. The "Next" button shows a spinner while the server pre-validates your name and description against the moderation rules. If the text breaks a rule, inline errors appear under the offending field and the step stays put. If you hit the rate limit, a system error appears at the bottom of the form and the Next button is disabled for 5 seconds before you can retry. If the pre-validate call fails for a network or server reason, a warning toast appears and the wizard advances anyway — step 4 is the real gate.'
- 'Step 3 — Meta data. Pick the category-specific filter values that apply to your product. Which filters appear depends on the categories you picked on step 2 — vehicle categories may show year/condition/etc., real-estate categories may show size/rooms/etc. Each filter type has its own selector (single-pick, multi-pick, numeric range, year-date range). Click Next; the page silently runs a reCAPTCHA challenge in the background. A failure shows the "suspicious activity" message inline and you stay on step 3.'
- 'Step 4 — Submit. The wizard switches to a single loading screen and runs the submit in two phases: images upload to storage, then the create request fires. You do not click anything on this step — it runs automatically when you arrive. On success the screen shows the new product\'s public URL with Copy and Open Link buttons and a Close button. On a server-validation failure the screen shows the offending error bullets and a "Go back and fix" button that jumps to the step that owns the first failing field; the Exit button closes the wizard and discards the in-progress data.'

### whatToExpect (structural — drafted directly)

- 'The wizard is a modal dialog; you cannot reach it by typing a URL, and you cannot dismiss it by clicking outside or pressing an X — there is no X. To leave before submitting, use the Back buttons or the browser back / navigation; closing the tab discards everything you had typed.'
- 'After step 2\'s pre-validate passes, step 3 becomes reachable. If the server reports content violations (banned words, spam, hidden contacts, repeated characters, gibberish, keyword stuffing, etc.) the wizard stays on step 2 and renders the inline error against the offending field. If the pre-validate call itself fails (network drop, 5xx), the wizard surfaces a warning toast and lets you continue to step 3 anyway — the real moderation gate is on step 4.'
- 'The price field is conditional. When the selected top category is a free-zone category, the price input and currency selector disappear from step 2 entirely — the server forces the price to zero on save. When the top category is not free-zone, the price input renders and is required; a blank price keeps Next disabled (via inline error) until you fill it.'
- 'Submit on step 4 runs in two phases. Phase one uploads each image to storage; phase two posts the create request. Today these two phases share one generic "Loading…" overlay — the tester does not see distinct "Uploading images…" then "Creating product…" prose. If an image upload fails, an error toast appears and the failure surface renders. If the upload succeeds but the create request returns server-side errors, those errors render as bulleted inline messages and the "Go back and fix" button routes you to the step that owns the first failing field.'
- 'The success surface shows the new product\'s portal URL with a Copy icon (one-shot copy with a 2-second confirmation), a View link that opens the URL in a new tab, and a Close button. Closing without a parent `onFinish` handler reloads the current page — used so that, for example, the owner products list refetches after a create.'
- 'Image-upload failure does NOT auto-cleanup orphans — the wizard\'s store-layer cleans uploaded keys if the create POST then fails, but a network drop mid-upload leaves no R2 objects to clean up because the upload never completed.'
- 'The wizard cannot open until you have a base site and a region/city on your account. Clicking the regular Add-new-product button against an account missing either opens the base-site/region picker first; saving the picker chains directly into the wizard. The Free Zone page entries skip this gate — see Pitfalls.'

## Option proposals (Mastermind picks)

### Title — 3 options

- **A. Plain — "Product Creation Flow"** — symmetric with `Start Message Flow` and `Follow Flow` in voice; sticks to the schema slug.
- **B. Tester-friendlier — "Create-a-Product Wizard"** — names the surface as a wizard explicitly; reads more concrete to a tester who hasn't seen the dialog yet.
- **C. Action-led — "Posting a Product"** — leads with the user goal rather than the mechanism; pairs well with the `start-message-flow` precedent ("how a user opens a conversation").

### Overview — 3 versions (different emphasis)

**Version 1 — surface-led (emphasises that it's a dialog, not a route).**

> The dialog-based wizard a signed-in user runs to post a new product. The wizard is a four-step Drawer dialog opened from the "Add new product" (+) button — present in the portal header on desktop, in the owner-dashboard TopNavigation on every /owner/* page, and in the mobile footer bar at the bottom of the screen. The two "Join Free Zone" buttons on /blog/free-zone open the same wizard for signed-in users. The wizard is not URL-addressable: typing or pasting a URL cannot reach it, and there is no X-close — the only way out before submit is the Back buttons or the browser back navigation. The four steps are images → basic info → category-specific meta-data → submit; the user lands on a success surface with the new product's URL once the submit completes.

**Version 2 — flow-led (emphasises what happens to the data along the way).**

> How a signed-in user posts a new product through the four-step Drawer dialog opened from the "Add new product" (+) button. Step 1 picks up to five images and holds them in memory; step 2 takes name + description + category cascade + (when the category is not free-zone) price + currency, and the Next button on step 2 only advances after the server has cleared the name and description against the moderation rules; step 3 lets the user pick the category-specific filter values; step 4 runs automatically on arrival, uploading the images then posting the create request and showing the result. The wizard cannot be reached by URL and has no outside-click or X-close.

**Version 3 — prerequisite-led (emphasises what gates the wizard and where it lives).**

> The way a signed-in user posts a product on Oglasino. The wizard sits behind the "Add new product" (+) button in the portal header, the owner-dashboard top bar, the mobile footer, and the owner-products empty state, plus the two "Join Free Zone" buttons on /blog/free-zone. Before the wizard can open, the account must have a base site and a region/city — accounts missing either are routed through a one-step picker that chains into the wizard on save. The wizard itself is a four-step modal dialog (images → basic info → meta-data → submit), is not URL-addressable, and cannot be dismissed by clicking outside or by an X — closing the tab or backing out drops everything in progress.

### Pitfalls — 3 candidate cuts of non-known-issue pitfalls

The brief asks for variant emphasis. Known-issue pitfalls (carrying the cross-reference to `issues.md`) are listed in a separate block below the three cuts.

**Cut A — field-validation traps (mid-step failure modes).**

- 'The price + currency block is conditional. When the top category is a free-zone category the price input is hidden entirely on step 2 — there is nothing for the tester to fill, and the saved product has price zero. Easy to read "the price field disappeared" as a regression when it is the intended free-zone behaviour.'
- 'The pre-validate gate is server-side and lives at the Next button on step 2. Structural issues (blank field, too-long name) surface immediately on click; content moderation failures (banned words, hidden contacts, gibberish) surface after the call returns and look identical to structural errors — both render as inline ERRORS-namespace messages under the offending field. The two error sources are not distinguished in the UI.'
- 'If the pre-validate call itself fails for a transport reason (network drop, 5xx), the wizard does NOT block — it shows a warning toast and lets you advance to step 3 anyway. Step 4 is the real gate. A tester expecting "Next is gated; if the call breaks, Next stays disabled" will see the opposite.'
- 'Step 3\'s reCAPTCHA fires silently on the Next click. A tester who never trips it will not see it at all; a tester who looks suspicious to reCAPTCHA will see a single inline "suspicious activity" line on step 3 with no obvious "retry the challenge" affordance — the user has to back out, edit something, and come back.'
- 'Step 4 runs automatically when you land on it — there is no Submit button. A tester arriving at step 3\'s Next click and then waiting for "now I press submit" will miss that the submit is already happening.'

**Cut B — step-transition and entry-point traps (where the user can lose state or land in the wrong place).**

- 'The wizard cannot be dismissed by clicking outside the dialog and has no X-close button. The only ways to abandon a session before step 4 are the Back buttons (which preserve state on the previous step) and the browser back / navigation (which discards everything). A tester used to "click outside the modal to dismiss" will not get the expected escape hatch.'
- 'Closing the tab on any step (or refreshing the page) silently discards every field the user has filled. Image files, name, description, category cascade, filters — all of it is in memory only, never persisted as a draft. This is intentional but easy to read as data loss.'
- 'The Free Zone page entries open the wizard directly, skipping the base-site/region pre-condition. A logged-in user without a base site or region who clicks "Join Free Zone" lands on a wizard that renders nothing visibly (the dialog is open but the step body is empty) until they go back and set their base site / region elsewhere. The portal header and owner-dashboard entries route through the picker correctly; only the Free Zone buttons skip it.'
- 'The Add-new-product button is absent on every /admin/* page (`allowProductCreation={false}` is passed to TopNavigation in the admin layout). A tester signed in as an admin who navigates from /owner/products to /admin and then looks for the create button on the admin chrome will not find one — admins create products through the regular owner surfaces, not through admin chrome.'
- 'Step 1\'s image-error key is held by the wizard parent component, not by step 1. Going back from step 2 to step 1 will re-display whatever error was last shown on step 1, even if the picker has been edited since. The behaviour is intentional (the wizard wants the error to survive transitions) but reads as a stuck error if the user has already fixed the picker before coming back.'

**Cut C — image-handling and submit traps (where files vs network can confuse the tester).**

- 'Images are held in browser memory only across steps 1–3; no upload happens until step 4 lands. Refreshing the page, navigating away, or closing the tab between picking images on step 1 and arriving on step 4 discards every picked file — there is no draft on the server, no resumable upload.'
- 'Per-image upload progress (per-file processing → uploading → complete) is rendered inside the picker on step 1, but step 1 is unmounted by the time step 4 is active. So during the submit phase the tester sees a single generic "Loading…" overlay, NOT a per-image progress bar — even though the upload is running file by file underneath.'
- 'Image-upload failures (R2 push-token call failed, network drop mid-upload, R2 itself returned non-2xx) surface as a one-shot error toast plus the step-4 failure screen. No images have been persisted at this point — the server has not even seen the create request yet. There is nothing for the tester to clean up; the wizard\'s Exit button discards everything in memory.'
- 'When the upload succeeds but the create POST then returns 400 / 422 validation errors, the wizard cleans up the orphan R2 keys in the background (fire-and-forget). The tester sees the failure screen with the inline error list, but cannot tell from the UI whether the orphan cleanup actually ran — a network drop on the cleanup leaves images in R2 with no Product row pointing at them.'
- 'The success surface\'s Close button reloads the current page when no `onFinish` callback was passed by the opener. The owner-products entry passes `onFinish`, so closing there does NOT trigger a full reload — the products list re-fetches in place. The portal-header and Free-Zone entries do not pass `onFinish`, so closing those does trigger a full reload of whatever page was underneath the dialog. The reload is intentional (so the page reflects the new product) but reads as a session hiccup if the tester is not expecting it.'

**Known-issue pitfalls (independent of the three cuts; should appear in the final topic regardless).**

- 'Known issue: a malformed 429 response from the pre-validate call leaves the step-2 Next button silently disabled with no inline reason — the user sees that Next stopped working but cannot tell why. The defensive client-side synthesis of a `__system` rate-limit message was removed deliberately; if the upstream malformed-429 bug is observed in QA, the fix belongs at the backend, not the client. Tracked in issues.md (2026-05-14 entry — "Malformed 429 leaves create-wizard silently blocked"); until fixed, if a tester sees Next stop working without an inline message, suspect a malformed 429 from the backend and check the network tab before filing it as a wizard defect.'

### qaChecklist — 3 candidates (minimum-viable / standard / exhaustive)

**Candidate A — minimum viable (~10 items, golden paths + one of each failure mode).**

- 'Sign in as a user with a base site and a region/city set. Click the Add-new-product button in the portal header — verify the wizard opens at step 1.'
- 'On step 1, pick two valid images of different MIME (JPEG + PNG). Verify the picker accepts both and Next advances to step 2.'
- 'On step 2, pick a non-free-zone top category. Verify the price + currency block appears. Fill name (~30 chars), description (~200 chars), price (any positive number), and a currency. Click Next — verify the spinner shows briefly and the wizard advances to step 3.'
- 'On step 3, fill any required filters the chosen category exposes. Click Next — verify the wizard advances to step 4 without an inline suspicion message.'
- 'On step 4, wait. Verify the generic loading overlay shows, then the success surface appears with the product URL, a Copy button, an Open Link button, and a Close button. Click Open Link — verify it opens the new product page in a new tab.'
- 'Repeat the golden path with a free-zone top category — verify the price + currency block does NOT render on step 2, and the created product\'s page shows it as free.'
- 'On step 2, leave name empty and click Next — verify the inline ERRORS-namespace "required" message appears under the name field and the wizard stays on step 2.'
- 'On step 2, type a description longer than 2000 chars (use the character counter as a guide) — verify the inline "too long" error appears and Next is blocked until shortened.'
- 'Drive the wizard against a name containing a known banned word — verify the inline moderation error appears under the name field after Next, and the wizard stays on step 2.'
- 'Sign in as a user without a base site set, click Add-new-product — verify the base-site/region picker opens first, fill it, save, and verify the wizard then opens at step 1.'

**Candidate B — standard (~16 items, golden + per-entry + per-failure-mode + abandonment + free-zone parity).**

A through J from Candidate A, plus:

- 'Verify the mobile footer Add-new-product button opens the wizard on a mobile breakpoint.'
- 'Verify the owner-products empty state shows the regular Add-new-product button when the user has zero products.'
- 'Verify the Join-Free-Zone hero and footer buttons on /blog/free-zone open the same wizard when signed in.'
- 'On step 2, drive the wizard to trigger a 429 by clicking Next rapidly — verify the system-error band appears, the Next button is disabled, and re-enables after ~5 seconds.'
- 'On step 4, abandon the wizard mid-submit by closing the tab — reopen the portal and verify no draft was persisted and no orphan product page exists at /product/...'
- 'On step 4 in a failure case, click "Go back and fix" — verify the wizard jumps to the step that owns the first failing field.'

**Candidate C — exhaustive (~24 items, every surface + every conditional + every transition).**

Everything in B, plus:

- 'Verify the Add-new-product button is absent on every /admin/* page (the admin layout passes `allowProductCreation={false}`).'
- 'Verify the wizard cannot be dismissed by clicking outside the Drawer.'
- 'Verify the wizard has no X-close button on its header.'
- 'Verify that backing out from step 2 to step 1 redisplays the previously-shown image error if there was one, even if the picker has been edited.'
- 'Verify that the AI-suggest icon appears next to the name input only when the name has content, and that clicking it fills the description field with an AI-generated suggestion.'
- 'Verify the CitySelector on step 2 is rendered disabled and pre-filled from the user\'s profile.'
- 'On step 4 success, verify the Copy button copies the product URL to clipboard, shows a 2-second confirmation, then resets.'
- 'On step 4 success, verify clicking Close from the owner-products entry refetches the products list in place (no full reload).'
- 'On step 4 success, verify clicking Close from the portal-header entry triggers a full page reload.'
- 'Verify on a free-zone product that the resulting product page renders without a price and with the free-zone visual treatment.'
- 'Verify the reCAPTCHA challenge on step 3 fires silently on most testers; force the suspicion path (incognito + VPN) and verify the inline "suspicious activity" message appears and the wizard stays on step 3.'
- 'Verify image-upload failure on step 4 (kill the network tab partway) shows the upload-error toast plus the failure surface, and that the wizard\'s Exit closes without leaving an orphan product.'

### relatedTopics — 3 candidates

**Set A — narrowest (the page the wizard creates).**

- `{ topicId: 'product-page' }` — the public product detail page that the wizard creates.

**Set B — narrow-plus-onboarding (the page + where the new product surfaces).**

- `{ topicId: 'product-page' }` — the public product detail page.
- `{ topicId: 'home-page' }` — where the new product appears on landing.
- `{ topicId: 'free-zone-page' }` — one of the entry surfaces.

**Set C — full cross-reference (every adjacent topic the wizard touches).**

- `{ topicId: 'product-page' }` — the product the wizard creates.
- `{ topicId: 'home-page' }` — where the new product appears on landing.
- `{ topicId: 'catalog-page' }` — where the new product appears under its category.
- `{ topicId: 'free-zone-page' }` — one of the entry surfaces, also the source of the price-required branch in the wizard.
- `{ topicId: 'pricing-page' }` — explains the free-zone vs paid distinction the wizard enforces.

### images — 3 candidates (variant coverage)

Note on convention: each entry has the HTML-comment + `name` + `description` shape with `imageKey` unset. Igor supplies files after the topic ships.

**Set A — four slots, one per step.**

1. `product-creation-flow-step-1-images.png` — step 1, image picker open with three thumbnails picked.
2. `product-creation-flow-step-2-basic-info.png` — step 2, basic-info form with category cascade filled and price + currency visible (non-free-zone).
3. `product-creation-flow-step-3-meta-data.png` — step 3, category-specific filters rendered for a vehicle category.
4. `product-creation-flow-step-4-success.png` — step 4 success surface with product URL + Copy + Open Link.

**Set B — five slots, four steps + free-zone variant of step 2.**

A1–A4 plus:

5. `product-creation-flow-step-2-free-zone.png` — step 2 with a free-zone top category selected, illustrating the absent price + currency block.

**Set C — five slots, four steps + the failure surface.**

A1–A4 plus:

5. `product-creation-flow-step-4-failure.png` — step 4 failure surface with the inline error bullets visible and the "Go back and fix" / Exit button pair.

## Files touched

None. Stop 1 produces no on-disk code changes.

## Tests

Not run. Stop 1 does no code work; `npx tsc --noEmit` runs at Stop 2 after the `topics.ts` append.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

(Adjacent observations in "For Mastermind" are flagged for triage but are not drafted as new entries.)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed (no code touched).
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): three flagged in "For Mastermind".
- Part 6 (translations): confirmed (no translation keys added; the listing of namespaces touched by the wizard is observational only).
- Other parts touched: Part 5 (session-summary template) — applied for filename + sections + closure gate. Part 11 (trust boundaries) — applied for the audit-depth check on both endpoints.

## Known gaps / TODOs

- The `<n>` counting rule from conventions Part 5 prescribes `<n>=1` for this session because no `*-qa-preparation-*.md` file exists in this repo's `.agent/`. The prior oglasino-web qa-preparation sessions (1, 2, 3 from 2026-05-14 per state.md's session log) appear to have been archived by Docs/QA and the sources deleted with no archive-pointer stub left behind — so the literal-letter Part 5 count produces `1`, which collides semantically with the archived 2026-05-14 session. The brief explicitly prescribes this counting method ("list your own `.agent/` folder for `*-qa-preparation-*.md` and add one"), so I have followed it. Flagged in "For Mastermind" for awareness; if the disambiguation needs to be different, name it before Stop 2.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — Stop 1 produces a session summary only.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

### Trust-boundary check (full depth, per decisions.md §2)

**Endpoint 1 — `POST /api/secure/products/pre-validate`**

Chain of evidence:

- Client supply (web): `{ name: string, description: string }` only. No id, no userId, no baseSiteId, no category, no price. The wire shape is in `PreValidateProductRequestDTO.java` and matches what the client sends (`productService.ts:235`).
- Server derivation (`DashboardProductController.preValidate` at lines 127–146): the controller runs `contentModerator.moderate(request.getName(), ContentField.PRODUCT_NAME)` and `contentModerator.moderate(request.getDescription(), ContentField.PRODUCT_DESCRIPTION)`. The moderator's inputs are the text strings the client provided — there is nothing else for the server to derive, because the moderation task is "decide whether this text is acceptable." Authentication is enforced by the `/api/secure/*` path mapping plus `FirebaseAuthFilter` (per conventions Part 11). No moderation decision depends on a server-derived "before" value or on a client-supplied identity field — there is no "before" state because the wizard is creating, not updating.
- Identity layer: standard `SecurityContextHolder` populated by `FirebaseAuthFilter`. No path uses `OglasinoAuthentication` data inside the pre-validate handler.
- Authorization: a Firebase JWT is required to reach the handler; no further per-row authorization applies because no DB row exists yet.

**Conclusion: clean.** No `oldName`-style violation is possible — there is no client-supplied "previous" value to trust, because the only inputs are the moderation targets themselves.

**Endpoint 2 — `POST /api/secure/products/create`**

Chain of evidence:

- Client supply: `NewProductRequestDTO` (`oglasino-backend/.../dto/NewProductRequestDTO.java`): `name`, `description`, `price` (`BigDecimal`, nullable), `currency` (`CurrencyDTO`, NotNull), `topCategory` / `subCategory` / `finalCategory` (`CategoryDTO`, NotNull), `filters` (`List<SelectedFilterDTO>`), `imageKeys` (`Set<String>`). No `userId`, no `baseSiteId`, no `regionId`, no `cityId`.
- Server derivation (`DefaultProductService.createProduct` at lines 92–182):
  - **(a) userId** — derived from `currentUserService.getCurrentUserIdStrict()` at line 96, which reads from `SecurityContextHolder` (per CurrentUserService convention). The user entity is reloaded via `userService.getUserById(...)` and `product.setOwner(user)` is the line that ties the new product to the authenticated identity (line 108). **Clean.**
  - **(b) baseSiteId** — derived in `getBaseSite()` at lines 199–209. The method reads `currentUser.getBaseSite()` (from the loaded user entity, which is server-side) and `baseSiteContext.getCurrentBaseSite()` (from the request's `BaseSiteFilter`, also server-side, populated from the request host). If the request host's base-site domain matches the user's base-site domain, the request's base site wins; otherwise the user's home base site is used. Neither branch trusts a client-supplied base-site id. The DTO has no base-site field — there is no way for the client to assert a base site at all. **Clean.**
  - **(c) Category, filter options, and price** —
    - Categories: each of `topCategoryId`, `subCategoryId`, `finalCategoryId` is walked through the server's `BaseSite.catalog.categories` tree (lines 137–162). The DTO carries `CategoryDTO` (which has at minimum an `id`), but only the `id` is used — the server pulls the rest of the category data from its own catalog. A client trying to pass a category that does not belong to the resolved base site throws `ProductValidationException("topCategory" | "subCategory" | "finalCategory", ProductErrorCode.CATEGORY_NOT_FOUND)`. **Clean.**
    - Filter options: `populateFilters` and `validateFilterData` (lines 705–799) reject any filter that is not among the allowed set for the resolved categories (`FILTER_NOT_IN_CATEGORY`) and any option that does not belong to its filter (`FILTER_OPTION_NOT_IN_FILTER`). Each filter's selected option's id is verified against the server-derived `canonicalFilter.options`. **Clean.**
    - Currency: `populateCurrency` (lines 801–810) rejects any currency id that is not in `baseSite.allowedCurrencies`. **Clean.**
    - Price: the create path takes `request.getPrice()` directly at line 125 and stores it on the product. `populateCategories` (lines 686–703) then overwrites the price to `BigDecimal.ZERO` if `topCategory.isFreeZone()`. For non-free-zone categories there is **no** server-side check that the price is non-null or ≥ 1. The `PRICE_REQUIRED` enforcement at line 825 is inside `updatePriceCurrency`, which only the update path calls — the create path does not call it. So a client posting `{ price: null, topCategory: <non-free-zone> }` produces a product with `price = null` and `free = false`. **Concern (low severity).** The client's Zod check (`productValidator.ts:76–82`) is the only thing stopping this today; if a different client (mobile when it adopts) skips the same check, the server will accept the request.
- Identity layer / authorization: standard Firebase auth + `currentUserService`. The owner is bound to the authenticated user (line 108), not to a client-supplied id. `USER_SETUP_INCOMPLETE` fires (line 101) if the user has no base site / region / city.

**Conclusion: clean conditional on the price gap.** The (a) / (b) checks are clean. The (c) check is clean for category, filter, currency; the price branch on create has a gap.

The price gap is **a verification task that needs routing** — not a Known-issue pitfall in this topic (no `issues.md` entry exists today, per decisions.md §3 the chain belongs here in "For Mastermind" only). Recommended routing: a one-shot read-only audit brief in `oglasino-backend` that walks every call site of `populateCategories` and `updatePriceCurrency` and decides whether the create path should call the same `PRICE_REQUIRED` guard or whether it should accept null on the assumption that the client is the gate. The fix is one branch in `DefaultProductService.createProduct` or one extra Jakarta annotation on `NewProductRequestDTO.price`.

### Audit-vs-brief observations (process-level)

- **Brief framing: "progressive status messages ('Uploading images...' → 'Creating product...' → done) per Model C."** Code reality (per the audit section above): step 4 shows a single generic LoadingOverlay; per-file progress states live in `useUploadProgressStore` and are rendered only inside the picker (which is unmounted on step 4). The tester does NOT see distinct "Uploading images…" / "Creating product…" prose during submit. I have written the topic body to reflect what the tester actually sees, with a tester-friendly note in `whatToExpect` and a stronger framing in `pitfalls` Cut C bullet B. If the intent was that Model C *should* show progressive prose and the implementation is short of intent, this is a defect not currently in `issues.md` — let me know if you want me to draft an entry for it.

### Adjacent observations (conventions Part 4b)

1. **Price-required server-side gate is missing on create (medium).** `DefaultProductService.createProduct` does not call the `PRICE_REQUIRED` check that `updatePriceCurrency` enforces; for non-free-zone categories the create path trusts the client to send a non-null price. Today's client Zod check covers it; tomorrow's mobile client may not. File: `oglasino-backend/src/main/java/com/memento/tech/oglasino/service/impl/DefaultProductService.java:92–182`. I did not fix this because it is out of scope (no code changes this session, and the fix lives in the backend repo).
2. **Free-zone entries skip the base-site/region pre-condition (low).** `JoinFreeZoneButton.tsx:18–22` opens `CREATE_NEW_PRODUCT_DIALOG` directly when signed in, with no check on `user.baseSite` / `user.regionAndCity`. The portal-header and owner-products entries route through `UserBasicDataSelectorDialog` instead. The result is a wizard that renders an empty body if the user is missing base-site or region. File: `oglasino-web/src/components/client/JoinFreeZoneButton.tsx:18–22`. I did not fix this because it is out of scope.
3. **`__system` rate-limit back-off is a fixed 5 seconds regardless of `Retry-After`** (low/cosmetic). `BasicInfoProductDialog.tsx:39` defines `RATE_LIMIT_BACKOFF_MS = 5000` as a constant; the comment at lines 36–39 explains that the backend's `Retry-After` header is not reliably exposed through the axios interceptor. If the backend later starts emitting a different retry-after value, the wizard will diverge. File: `oglasino-web/src/components/popups/components/BasicInfoProductDialog.tsx:36–39`. I did not fix this because it is out of scope.

### Brief vs reality

No structural mismatch between the brief and the code worth stopping the session for. The deviations above (progressive status messages absent in code; price gate gap; free-zone-entry skip) are observations recorded for routing, not blockers to drafting the topic. The brief explicitly said to apply Mastermind's `Model C` framing — I have done so, while writing the tester-facing copy to match the implementation rather than the framing.

### `<n>` counting

Listed `.agent/` for `*-qa-preparation-*.md` — no matches. Followed the brief's prescription and conventions Part 5 to set `<n>=1`. Past oglasino-web qa-preparation sessions (1/2/3 from 2026-05-14 per state.md session log) have been archived by Docs/QA and the source files deleted. No archive-pointer stub is in `.agent/`. The literal Part-5 rule produces the same `-1` filename a fresh slug would. If the intent is to disambiguate against archives, this needs a Docs/QA pass before Stop 2 — not a blocker for Stop 1 review.

### Stop 2 — what's owed

After Mastermind selects from the option proposals above:

- Title — one of A / B / C (or other).
- Overview — one of V1 / V2 / V3 (or rewrite).
- Pitfalls — one of Cut A / B / C plus the standing Known-issue pitfall for the 2026-05-14 malformed-429 entry (and a decision on whether to also note the Model-C progressive-status gap as a pitfall in this topic body or leave it as a "For Mastermind" item only).
- qaChecklist — one of A / B / C (or a custom merge).
- relatedTopics — one of Set A / B / C.
- images — one of Set A / B / C.

Stop 2 folds the decisions in, appends the final topic entry to `qaTopics` in `app/[locale]/design/topics.ts`, runs `npx tsc --noEmit` from `oglasino-web`, and closes the session.
