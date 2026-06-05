# Diagnostic — PreviewProductDialog: false banned/inactive on Listing tab + Details-tab scroll dead-zone

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-06-01
**Feature slug:** product-update-parity
**Type:** READ-ONLY diagnostic (Phase 2). No code changed, nothing staged.
**Task:** Diagnose two on-device bugs in `PreviewProductDialog` (opened only from the update screen, `[productId].tsx handlePreview` → `{ productDetails }`): (Q1) the Listing tab shows the product as banned/inactive though it is active; (Q2) the Details tab scrolls only over the top region, the rest is unreachable. Root cause + file:line + minimal-fix location for each. Do not fix.

---

## Q1 — false banned/inactive on the Listing tab

### The chain

1. **Listing tab** renders `<PreviewProductCard productOverview={previewProductData} />` (`PreviewProductDialog.tsx:209`), which is a thin wrapper around `<ProductCard>` (`PreviewProductCard.tsx:9`).
2. **The banned/inactive visual lives in `ProductCard`** (`ProductCard.tsx:79-93`). It is a full-card dimming overlay (`absolute … h-full w-full bg-background/50`) plus one or two red badges, gated on:

   ```tsx
   // ProductCard.tsx:79-80
   (productOverview.productState !== ProductState.ACTIVE ||
    productOverview.moderationState === ModerationState.BANNED)
   ```

   - Badge 1 always renders inside the overlay and prints `productOverview.productState` (`:84`).
   - Badge 2 ("BANNED") renders only when `moderationState === BANNED` (`:86-90`).

3. **The DTO that drives it is `previewProductData`** (`PreviewProductDialog.tsx:121-140`), built as a `ProductOverviewDTO`.

### What the field is getting

`ProductOverviewDTO` declares **`productState?`** and **`moderationState?`** as optional (`ProductOverviewDTO.ts:7-8`). `previewProductData` (`PreviewProductDialog.tsx:121-140`) **never assigns either one** — it maps `id, ownerId, baseSiteOverview, name, cityLabelKey, price, currency, free, topImageKey, favorite, condition` only. So both fields are **`undefined`** on the card.

Evaluate the overlay gate with `productState === undefined`:

```
undefined !== ProductState.ACTIVE   →  true   →  overlay shows, every time
```

So the dimming overlay and badge 1 render unconditionally. Badge 1 prints `undefined`, i.e. an **empty red pill** (`<Text>{undefined}</Text>`), and the card is greyed out — exactly the "shown as banned/inactive even though it's active" report. (The BANNED badge itself stays hidden, since `moderationState` is `undefined`, not `BANNED`.)

### What it should be

The source object `productDetails` (typed `UpdateProductRequestDTO`) **does carry the real state** — `productState?: ProductState` and `moderationState?: ModerationState` (`UpdateProductRequestDTO.ts:6-7`). The preview is simply **not mapping them through**. The card's notion of "active" is `productState === ProductState.ACTIVE && moderationState !== ModerationState.BANNED`; that is the same gate the real product cards use — there is no second/wrong field, the field is just absent.

### Root cause (Q1)

`previewProductData` (`PreviewProductDialog.tsx:121-140`) omits `productState` and `moderationState`, leaving them `undefined`. `ProductCard.tsx:79` treats `undefined !== ACTIVE` as "not active" and paints the banned/inactive overlay with an empty red badge.

- **Field:** `productState` (primary) and `moderationState` (secondary) on the overview DTO.
- **Value it gets:** `undefined`. **Value it should be:** `productDetails.productState` (`ACTIVE` for an active product) / `productDetails.moderationState` (`APPROVED`).

### Minimal fix location (do NOT fix here)

In `previewProductData` (`PreviewProductDialog.tsx:121-140`), map the two fields from `productDetails`, defaulting to active so a not-yet-saved/undefined state previews as active:

```tsx
productState: productDetails.productState ?? ProductState.ACTIVE,
moderationState: productDetails.moderationState ?? ModerationState.APPROVED,
```

(`ProductState` and `ModerationState` are already imported elsewhere in the tree; both fields exist on `UpdateProductRequestDTO`.) No change to `ProductCard` is needed.

---

## Q2 — Details-tab scroll dead-zone

### Scroll-container structure (outer → inner)

| Layer | Where | Orientation |
|---|---|---|
| **Outer ScrollView** | `DialogWrapper.tsx:67-76` — wraps every dialog; `contentContainerStyle` `padding:24, paddingBottom:40, flexGrow:1`. Dialog frame is `max-h-[85%] … overflow-hidden` (`DialogWrapper.tsx:63`). | vertical |
| **Inner ScrollView** | `PreviewProductDialog.tsx:181` — the dialog's **own** root `<ScrollView contentContainerStyle={{ gap: 10 }}>`. No `nestedScrollEnabled`, no height/flex. | vertical |
| Details-tab content (inside inner) | `ImagesCarousel` (`:215`), then `ProductUserDetails`, `ProductDetails`, `ProductFunctions`, `ProductSpec` (`:219-240`). | — |
| `ImagesCarousel` internal | `ImagesCarousel.tsx:114` — a **horizontal** `FlatList`, item height 500px (`IMAGE_HEIGHT`, `:10/:97`). | horizontal |

The detail sub-components (`ProductDetails`, `ProductSpec`, `ProductFunctions`, `ProductUserDetails`) contain **no** `ScrollView`/`FlatList` of their own — confirmed by grep. So the only scrollers in the Details tab are the two stacked **vertical** ScrollViews plus the carousel's horizontal one.

### Why scroll dies past the top region

The Details tab nests a **second vertical `ScrollView` (`PreviewProductDialog.tsx:181`) directly inside `DialogWrapper`'s vertical `ScrollView` (`DialogWrapper.tsx:67`)** — same orientation, one inside the other. The Details content is tall (≈500px carousel + four stacked sections), well past the 85%-height dialog viewport.

With two same-orientation ScrollViews stacked and the inner one declaring **no `nestedScrollEnabled` and no bounded height**, the inner ScrollView owns the vertical pan but its scrollable extent is pinned to the parent's viewport rather than its own content height. The result on device: the first screenful (the carousel + the start of `ProductUserDetails`) is what the inner viewport shows, the outer ScrollView has nothing extra to advance, and everything below the first viewport is clipped and unreachable — the reported dead zone. The **Listing tab does not exhibit it** because its content (one `ProductCard` + the Close button) fits within the viewport and never needs to scroll past it.

Ruled out:
- **`ImagesCarousel`'s horizontal `FlatList` (`:114`) is not the cause.** It is horizontal (`horizontal` prop), so it claims the *horizontal* pan and lets vertical gestures pass to the parent; it is also `scrollEnabled={data.length > 1}` and `bounces={false}`. It sits in the top region that *does* scroll.
- **No `flex:1` / fixed-height child** consumes the space — the sub-components are plain `View`s; the dead-zone is the ScrollView-in-ScrollView, not a greedy child.
- **No content rendered outside the scroller** — all five sections are inside the inner ScrollView.

> Note on the sibling dialogs: `FiltersDialog.tsx:167-168` also nests its own root ScrollView but adds **`nestedScrollEnabled`**, which is the prop that makes a nested same-orientation ScrollView scroll on Android. `PreviewProductDialog` (`:181`) omits it. `AddUpdateProductDialog.tsx:187` nests one too without the prop, but its per-step content is short enough to fit, so it doesn't surface the bug. The reliable differentiator here is the missing `nestedScrollEnabled` combined with tall content.

### Root cause (Q2)

`PreviewProductDialog` wraps its body in its **own** root `<ScrollView>` (`PreviewProductDialog.tsx:181`) even though it is already mounted inside `DialogWrapper`'s `<ScrollView>` (`DialogWrapper.tsx:67`). Two stacked vertical ScrollViews + tall Details content + no `nestedScrollEnabled` = the inner ScrollView captures the gesture but can't scroll past the first viewport, so content below is unreachable.

### Minimal fix location (do NOT fix here)

At `PreviewProductDialog.tsx:181` (and its matching close at `:251`). Two viable minimal fixes:

- **Preferred — remove the redundant scroller:** change the root `<ScrollView … contentContainerStyle={{ gap: 10 }}>` to a plain `<View className="gap-3">` (or similar) and let `DialogWrapper`'s ScrollView do the single, unconflicted vertical scroll. This eliminates the nesting entirely and matches the "dialog content + DialogWrapper provides the scroll" model.
- **Alternative — match `FiltersDialog`:** add `nestedScrollEnabled` to the inner ScrollView (`:181`). Smaller diff, but keeps two stacked scrollers and the gesture handoff.

Recommend the first: removing the inner ScrollView removes the conflict at the source.

---

## Session summary (conventions Part 5)

### Implemented
Nothing. Read-only diagnostic per the brief. No source files changed, nothing staged.

### Files touched
None (source). Output written to `.agent/diagnose-preview-dialog.md` and copied to `.agent/last-session.md`.

### Files read (evidence)
- `src/components/dialog/dialogs/PreviewProductDialog.tsx`
- `src/components/dashboard/components/PreviewProductCard.tsx`
- `src/components/product/ProductCard.tsx`
- `src/lib/types/product/ProductOverviewDTO.ts`, `ProductState.ts`, `ModerationState.ts`, `UpdateProductRequestDTO.ts`
- `src/components/dialog/components/DialogWrapper.tsx`
- `src/components/ImagesCarousel.tsx`
- `src/components/product/ProductDetails.tsx`, `ProductSpec.tsx`, `ProductFunctions.tsx`, `ProductUserDetails.tsx` (grep for nested scrollers — none)
- Sibling dialogs `FiltersDialog.tsx`, `product-creation/AddUpdateProductDialog.tsx` (nested-ScrollView convention check)

### Tests
None run — no code changed.

### Cleanup performed
None needed (read-only).

### Obsoleted by this session
Nothing.

### Conventions check
- Part 4 (cleanliness): N/A — no code written.
- Part 4a (simplicity): both recommended fixes are minimal; the preferred Q2 fix *removes* a redundant component rather than adding props.
- Part 4b (adjacent observations): while tracing the overlay I noticed `ProductCard.tsx:84` renders the raw enum value (`productState`) as the badge text rather than a translated label — out of scope for this diagnostic, flagged for Mastermind below, not acted on.
- Challenge gate: the brief's framing matched the code; no Brief-vs-reality conflict. The brief's Q2 hypothesis list named the actual cause (nested vertical scrollable); confirmed.

### Config-file impact
No change. This is a diagnostic; it does not adopt or close a feature, so no edit to `state.md`'s Expo backlog table or any of the four config files is needed.

### For Mastermind
- **Q1 fix is one mapping line** in `previewProductData` (map `productState`/`moderationState` from `productDetails`, default ACTIVE/APPROVED). Both fields already exist on `UpdateProductRequestDTO`; no card change.
- **Q2 fix is one structural line**: drop `PreviewProductDialog`'s redundant root `<ScrollView>` (line 181) in favor of `DialogWrapper`'s, or add `nestedScrollEnabled`. Recommend the former.
- **Adjacent (not in scope):** `ProductCard.tsx:84/:88` print the raw `ProductState`/`ModerationState` enum string as badge text (e.g. literal "INACTIVE"), not a translated label. If the real product cards are expected to show localized state labels, this is a separate parity item worth a backlog row.
