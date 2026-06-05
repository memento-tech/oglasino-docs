# Audit — oglasino-web: what happens on product-create SUCCESS (read-only)

**Repo:** oglasino-web
**Branch:** `stage`
**Date:** 2026-05-30
**Mode:** READ-ONLY. No code changed, no git ops, no runtime.
**Companion to:** [`audit-create-flow.md`](audit-create-flow.md) (the prior audit — captured the pre-validate gate + step-4 FAILURE UX; this one captures the SUCCESS path it left open).

All file:line references are to code on `stage` as read this session.

---

## TL;DR (the one answer mobile needs)

On a successful create, web does **not** close the dialog and does **not** navigate anywhere. The same step-4 dialog re-renders in place as a **success confirmation screen**: a "<name> created" heading, a two-line blurb, the new product's **absolute public URL** shown in a bordered box with a **copy-to-clipboard** icon, and two buttons — **"View link"** (opens that URL in a **new browser tab**, dialog stays open behind it) and **"Close"** (closes the dialog, then **hard-reloads the page the user was on**). The user is never auto-taken to the product; they opt in via "View link" or just close. The page reload on close is the only thing that surfaces the new product into whatever list was behind the dialog — there is no targeted refetch and no router navigation.

**`onFinish` is never supplied anywhere on web.** It is a dead, threaded-but-unwired extension hook. So the no-`onFinish` branch *is* the web behavior, not an Expo-only artifact.

---

## Section 1 — The create-success handler

File: `src/components/popups/components/UploadedProductDialog.tsx`, success branch at **lines 64–83** (inside the `useEffect` IIFE that runs once via `triggeredRef`).

### Actual order of operations (corrected vs the prior audit)

The prior audit (`audit-create-flow.md:189`) summarized the order as "`setUploadSuccess(true)`, build product URL, `track`, `onFinish()`." Reading the real code, the order is different — **track is first, `setUploadSuccess` is third**:

1. **`const parsedPrice = Number(productData.price)`** (line 65) — local, for the analytics payload only.
2. **`track('product_create_completed', { ... })`** (lines 66–80) — GA4 event. Payload:
   - `product_id: result.data.id`
   - `top_category_id` / `sub_category_id` / `final_category_id` — each included only if that category id is defined
   - `price: parsedPrice` — only if `Number.isFinite(parsedPrice)`
   - `currency: productData.currency.code` — only if present
   - `image_count: productData.imagesData?.length ?? 0`
3. **`setProductUrl(getNormalizedProductUrl(result.data.id, result.data.name, true, locale))`** (line 81) — builds and stores the URL string in component state.
4. **`setUploadSuccess(true)`** (line 82) — flips the view to the success screen.
5. **`onFinish?.()`** (line 83) — optional-chained; **no-op on web** (see Section 2).
6. Then, common to all branches: **`setLoading(false)`** (line 99), and `finally { progress.finish() }` (line 115) resets the upload-progress store.

`result.data` is `NewProductResponseDTO = { id: number; name: string }` (per the prior audit's verbatim shape, `audit-create-flow.md:183`).

### The product URL — what it's built from and what it's used for

- **Built from:** `getNormalizedProductUrl(id, name, withPrefix=true, locale)` at `src/lib/utils/utils.ts:115`.
  - Returns `` `https://oglasino.com/${locale}/product/${productId}/${normalizeProductName(productName)}` `` (line 121).
  - With `withPrefix=true` it is an **absolute production URL** with a hardcoded `https://oglasino.com` host and a `/${locale}` segment.
  - `normalizeProductName` (line 128) slugifies the returned `name`: lowercase, NFD diacritic strip, `đ→d`, non-`[a-z0-9]`→hyphen, collapse/trim hyphens. So the slug is derived from the **returned** `name`, not a backend-supplied slug field.
- **Used for (three things, all in the success screen — never a navigation):**
  1. **Displayed** as plain text in a bordered box — `<span className="text-xs">{productUrl}</span>` (line 164).
  2. **Copied to clipboard** — the copy icon's `handleCopy` (lines 120–126) calls `navigator.clipboard.writeText(productUrl)` and shows a 2s "copied" state.
  3. **Opened in a new tab** — the "View link" button calls `window.open(productUrl, '_blank')` (line 173).
- It is **never** passed to a router / `router.push`. The dialog performs **no Next.js navigation** on success.

### What `onFinish()` actually does at the call site

Traced up the chain:

- `UploadedProductDialog` declares `onFinish?: () => void` (line 30) and calls `onFinish?.()` on success (line 83).
- Its parent `CreateNewProductDialog` (`src/components/popups/dialogs/CreateNewProductDialog.tsx`) declares `onFinish?: () => void` (line 32) and passes it straight through to `UploadedProductDialog` (line 186). It does nothing else with it.
- `CreateNewProductDialog` is rendered only by `DrawerDialogManager` (`src/components/popups/DialogManager.tsx:79`) as `<SpecificDialog isOpen onClose={closeDialog} {...dialogProps} />`. So `onFinish` would have to arrive inside `dialogProps`.
- `dialogProps` is whatever the second arg of `openDialog(id, props)` was (`src/components/popups/store/useDialogStore.tsx:14`).
- **Every** site that opens this dialog calls it with **no props** (see Section 2). So `onFinish` is `undefined` at every web invocation, and **`onFinish?.()` is always a no-op on web.**

So: at the call site, `onFinish` does **nothing** — there is no web handler that closes the dialog, navigates, refreshes a list, or resets wizard state via `onFinish`. The close/reload behavior lives entirely in the success screen's Close button, not in `onFinish`.

### Is there a success SCREEN, or does it just close?

There **is** a dedicated in-place success screen (it does not close on success). Rendered by the `{uploadSuccess && ( ... )}` block, **lines 152–188**:

- `<h2>` — `tDialog('new.product.success.title', { productName: productData.name })` (line 155).
- `<p>` — `tDialog('new.product.success.description.1')` + `' '` + `tDialog('new.product.success.description.2')` (lines 160–161).
- Bordered box: the absolute `productUrl` text + a `CopyIcon` whose click runs `handleCopy` (lines 163–168).
- Two buttons (lines 170–186):
  - **"View link"** — `tDialog('new.product.success.view.link')` → `onClick={() => productUrl && window.open(productUrl, '_blank')}` (lines 171–175).
  - **"Close"** — `tDialog('button.close.label')` → `onClick={() => { onClose(); if (!onFinish) { window.location.reload(); } }}` (lines 176–185).

---

## Section 2 — The no-`onFinish` case on web

**Web NEVER runs `UploadedProductDialog` / the wizard WITH an `onFinish` prop.** This is the inverse of the brief's hypothesis.

Evidence:

- A repo-wide search for `onFinish` returns only its definition in `UploadedProductDialog.tsx` and the pass-through in `CreateNewProductDialog.tsx`. **No component anywhere supplies it.**
- All three open sites call `openDialog(DialogId.CREATE_NEW_PRODUCT_DIALOG)` with **no second argument** (so `dialogProps = {}`):
  1. `src/components/client/buttons/AuthAddNewProductButton.tsx:32`
  2. `src/components/client/JoinFreeZoneButton.tsx:21`
  3. `src/components/popups/dialogs/UserBasicDataSelectorDialog.tsx:88`

So the no-`onFinish` path is the **only** live path on web. Concretely it produces:

- **At success:** `onFinish?.()` (line 83) is a no-op.
- **On "Close":** `onClose()` closes the dialog (`closeDialog()` clears `currentDialogId` + `dialogProps`), then — because `!onFinish` is always true — **`window.location.reload()`** hard-reloads whatever page the user was on when they opened the wizard.

**Implication for mobile:** the no-`onFinish` branch is not an Expo-only artifact with no web counterpart — it is exactly what web does. But the web counterpart's mechanism is a full-page reload, which has no RN equivalent. Mobile should derive its post-success behavior from the **success-handler intent** (confirmation screen with link → on close, reset/navigate so the new product shows up), not from `onFinish`, since `onFinish` carries no behavior on web.

---

## Section 3 — What "success" leaves on screen (the behavior mobile reproduces)

**Plain answer:** After a successful create, the user is looking at a **"product created" confirmation screen inside the same dialog** — not the product page, not a freshly-refreshed list. The screen shows the new product's canonical public URL with a **copy** button and a **"View link"** button.

From there:

- If they tap **"View link"** → the product's public page opens in a **new browser tab** (`window.open(..., '_blank')`); the confirmation dialog stays open behind it. They are not navigated in the current tab.
- If they tap **"Close"** → the dialog closes and the **current page hard-reloads** (`window.location.reload()`). The reload is the only mechanism that makes the new product appear in whatever list was behind the dialog (e.g. an owner products list). There is no targeted refetch, no optimistic insert, no router navigation to the product.
- Web never auto-navigates the user to the new product.

### Exact keys / copy / link target

Copy text is **backend-seeded** (DIALOG namespace, per conventions Part 6) — not present in this repo, so only the keys are reportable:

| Element | Translation key (DIALOG namespace) | Notes |
| --- | --- | --- |
| Success heading | `new.product.success.title` | interpolates `{ productName }` = `productData.name` |
| Blurb line 1 | `new.product.success.description.1` | rendered before line 2, space-joined |
| Blurb line 2 | `new.product.success.description.2` | |
| "View link" button | `new.product.success.view.link` | → `window.open(productUrl, '_blank')` |
| "Close" button | `button.close.label` | → `onClose()` + `window.location.reload()` |

**Link target** (the displayed/copied/opened URL):
`` `https://oglasino.com/${locale}/product/${id}/${slug}` `` where `slug = normalizeProductName(name)`, `id`/`name` from `result.data`.

---

## For Mastermind — web-only mechanics with no direct Expo equivalent

Name these so the RN translation is deliberate:

1. **`window.location.reload()` (line 182) — full-page hard reload on Close.** No RN equivalent. This is precisely the gap the Expo step-4 `// TODO` describes ("navigate back or reset state instead of reload"). Mobile equivalent: navigate back / reset wizard state and refetch the owner's product list — there is no web-side targeted refetch to copy, the reload *is* the refresh.
2. **`window.open(productUrl, '_blank')` (line 173) — opens a NEW browser tab to the absolute public URL.** RN equivalent is a deliberate choice: in-app navigation to the product screen by id (router push) and/or `Linking.openURL` for the external page — not a 1:1.
3. **Absolute production URL `https://oglasino.com/${locale}/product/...`** (hardcoded host in `getNormalizedProductUrl`, `utils.ts:121`) — a web canonical/share URL, not an app route. Mobile's "view" should route to its in-app product screen by id; the **copy/share string** can still be this public URL.
4. **`navigator.clipboard.writeText` (line 123)** — web clipboard API. RN equivalent: `expo-clipboard`.
5. **The dead `onFinish` hook.** There is no real web "`onFinish` success path" to mirror — the prop is threaded through two components but supplied by zero callers, so its only observable effect is the inverse `if (!onFinish) reload()` guard. Mobile should ignore `onFinish` as a contract and build the success behavior from the intent in Section 1/3.

### Adjacent observation (conventions Part 4b)

- **Dead `onFinish` prop across the create wizard.** `onFinish?: () => void` is declared on `UploadedProductDialog` (`UploadedProductDialog.tsx:30`) and `CreateNewProductDialog` (`CreateNewProductDialog.tsx:32`) and threaded through, but **no opener anywhere supplies it** — all three `openDialog(CREATE_NEW_PRODUCT_DIALOG)` calls pass no props. The only behavioral consequence is the always-true `if (!onFinish)` reload guard. **Severity: low** (structural / could mislead a future reader into thinking a finish-callback path exists). Not fixed — out of scope and this session is read-only.

---

## Config-file impact

- **None.** Read-only audit; no `conventions.md` / `decisions.md` / `state.md` / `issues.md` change required.
