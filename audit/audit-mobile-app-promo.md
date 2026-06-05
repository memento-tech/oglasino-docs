# Audit — Mobile App Promo Dialog (oglasino-web)

**Type:** read-only audit. No code changes made.
**Date:** 2026-06-04
**Branch:** dev
**Feature:** a "we have mobile apps" promo dialog, web-only. Shows once, then no sooner than every 5 days, gated on a localStorage timestamp. Appears 5s **after** the consent banner is resolved on a first visit, and 5s after load on return visits.

Every machinery question is answered against the actual code below. Bottom line up front: **all the machinery exists, and there is an exact precedent** — `AccountStateDialogsInit` is an init component mounted in `AppInit` that reactively opens `InfoDialog` via the dialog store and clears its trigger flag. The promo can be built the same way, almost certainly **without a new dialog component or a new `DialogId`** (reuse `INFO_DIALOG`). See §14 and the "Recommended shape" at the end.

---

## DialogManager

### 1. How a dialog is registered and opened

The dialog system is a single global Zustand store — **no React context / provider**, so it is reachable from anywhere (React or not).

`src/components/popups/store/useDialogStore.tsx`:
```ts
type DialogStore = {
  currentDialogId: string | null;
  dialogProps: Record<string, any>;
  openDialog: (id: DialogId, props?: Record<string, any>) => void;
  closeDialog: () => void;
};
export const useDialogStore = create<DialogStore>((set) => ({
  currentDialogId: null,
  dialogProps: {},
  openDialog: (id, props = {}) => set({ currentDialogId: id, dialogProps: props }),
  closeDialog: () => set({ currentDialogId: null, dialogProps: {} }),
}));
```

- **API:** `openDialog(id: DialogId, props?: Record<string, any>)`. Only one dialog open at a time (single `currentDialogId`). Opening a second replaces the first.
- **Payload shape:** an arbitrary props bag spread onto the dialog component. There is no schema — each dialog defines the props it reads.
- **Two ways to call it:**
  - React: `const openDialog = useDialogStore((s) => s.openDialog)` (e.g. `AccountStateDialogsInit.tsx:37`).
  - Imperative / non-React: `useDialogStore.getState().openDialog(...)` / `.closeDialog()` (e.g. `AccountStateDialogsInit.tsx:64,99,125`).
- **DialogId enum lives in** `src/components/popups/dialogRegistry.ts:1-34` (a string enum, 32 members today; `INFO_DIALOG = 'infoDialog'` at `:7`).

### 2. Central registry / switch (DialogId → component)

`src/components/popups/DialogManager.tsx` (default export `DrawerDialogManager`):
- `DIALOGS: Record<string, React.ElementType>` maps the string id → component (`:38-71`; `infoDialog: InfoDialog` at `:45`).
- Render logic `:73-82`:
```tsx
const { currentDialogId, dialogProps, closeDialog } = useDialogStore();
if (!currentDialogId) return null;
const SpecificDialog = DIALOGS[currentDialogId];
if (!SpecificDialog) return null;
return <SpecificDialog isOpen={true} onClose={closeDialog} {...dialogProps} />;
```
So every dialog automatically receives `isOpen={true}` and `onClose={closeDialog}`, plus whatever was in the props bag.

`DrawerDialogManager` is mounted once at `app/[locale]/layout.tsx:64` (inside `NextIntlClientProvider`).

### 3. Pattern reference — a simple informational dialog

**`src/components/popups/dialogs/InfoDialog.tsx`** is the ideal reference — a generic informational dialog with a close button (and optional second action). Props (`:12-36`):
- `isOpen`, `onClose` (injected by the manager).
- `dialogTitle: ReactNode`, `dialogDescription: any` — the body can be arbitrary JSX (good: the two store badges can live here).
- `closeButtonLabel?: string` — defaults to `tDialog('button.close.label')` (`:64`).
- `autoDismissAfterMs?: number` — built-in auto-dismiss timer with cleanup (`:44-48`). Not needed for the promo (user closes it) but available.
- Optional `onContinue` second button (`:66-91`) — omit it and you get a pure close-only dialog.

It reads its strings via `useTranslations(TranslationNamespaceEnum.DIALOG)` (`:37`) and renders through the shared `DrawerDialog` shell (`:50-59`), passing `addCloseButton={false}` because it renders its own button row.

The shared shell **`src/components/popups/dialogs/DrawerDialog.tsx`** is responsive: `<Dialog>` (radix modal) on desktop (`≥768px`, `:58-111`) and `<Drawer>` (bottom sheet) on mobile (`:114-152`), switched by `useMediaQuery('(min-width: 768px)')` (`:56`). Any new dialog should render through `DrawerDialog` for consistency, or just reuse `InfoDialog`.

**How `InfoDialog` is opened today (the exact promo precedent)** — `AccountStateDialogsInit.tsx:51-69`:
```tsx
openDialog(DialogId.INFO_DIALOG, {
  dialogTitle: tDialog('banned.dialog.title'),
  dialogDescription: (<div className="flex flex-col gap-3">…</div>),
  closeButtonLabel: tButtons('banned.go.home.label'),
  onClose: () => { useDialogStore.getState().closeDialog(); router.replace(...); },
});
```
Note: a caller can override the injected `onClose` by putting `onClose` in the props bag (spread order in `DialogManager.tsx:81` puts `{...dialogProps}` last, so a props-bag `onClose` wins). The promo does not need to override it — the default `closeDialog` is fine.

### 4. Client-component boundary

Yes. `DialogManager.tsx:1` is `'use client'`; `InfoDialog.tsx:1`, `DrawerDialog.tsx:1`, and `AccountStateDialogsInit.tsx:1` are all `'use client'`. Any new dialog component or arming component must be `'use client'`.

---

## Consent decision signal

### 5. ConsentStore shape — "has the user decided yet?"

**`src/lib/store/useConsentStore.ts`:**
```ts
export interface ConsentStore {
  consent: ConsentData | null;   // null until hydrate() runs; null also = "no decision"
  hydrated: boolean;             // false until the first cookie-read attempt completes
  reopenRequested: boolean;
  hydrate: () => void;           // reads og_consent cookie → sets {consent, hydrated:true}
  setConsent: (next) => void;    // writes cookie + sets consent (the "user decided" path)
  requestReopen / clearReopen
}
```
- There is **no** `hasDecided` flag. The decision is **inferred**: a decision exists iff, after hydration, `consent !== null`.
- `ConsentData` (`src/lib/consent/types.ts:6-18`) carries **`decidedAt: number`**. A real decision has `decidedAt > 0`; the in-memory `DENIED_DEFAULTS` uses `decidedAt: 0` to mean "no decision yet" (`types.ts:20-32`). **Important:** `DENIED_DEFAULTS` is never written to the cookie — the banner's reject/accept/save builders stamp a real timestamp — so in practice any non-null `consent` in the store equals a real decision. Either signal works; **`hydrated && consent !== null` is the simplest, most direct read.** (`consent?.decidedAt > 0` is equivalent and slightly more defensive.)
- The cookie is `og_consent` (`src/lib/consent/cookie.ts:3`), read synchronously by `readOgConsent()` (`cookie.ts:19-31`).

### 6. Existing component that reacts to the consent decision

Two precedents:

- **`src/components/client/consent/ConsentBanner.tsx`** is the closest "react to the decision being made" component. It **imperatively subscribes** to the store (`useConsentStore.subscribe(...)`, `:47-61`) inside a mount-only effect, and detects the first-time→decided transition (`:51`) and reopen requests (`:57`). This is React 19's "react to external state changes" pattern — `setState` calls live in the subscription callback, not in the effect body. The arming component should mirror this subscription style.
- **`src/lib/consent/sideEffects.ts`** `applyConsent(next)` (`:43-46`) is the single funnel every decision flows through — it calls `setConsent(next)` then `callGtagUpdate(next)`. The banner's three handlers all route through it (`ConsentBanner.tsx:78-87`). This is where "the user just decided" actually happens; the store transition it causes is the signal the promo arms on.

### 7. Return-visit state on fresh load — sync or async hydration?

**Hydration is NOT synchronous at module load — it runs in a client effect after mount, but the read itself is synchronous.** The store initializes `{ consent: null, hydrated: false }` (`useConsentStore.ts:26-28`). `hydrate()` is only invoked from `ConsentBanner`'s mount effect (`ConsentBanner.tsx:66-68`): if `!state.hydrated`, it calls `state.hydrate()`, which does a synchronous `readOgConsent()` (document.cookie parse) and sets `{ consent, hydrated: true }`.

Consequences for the return-visit trigger:
- On any fresh load, there is a **brief window** where `hydrated === false` and `consent === null`, before `ConsentBanner` mounts and hydrates. The arming component must **not** read `consent` until `hydrated === true`.
- After hydration on a return visit (decision persisted on a prior visit), `consent` is immediately the persisted `ConsentData` (`decidedAt > 0`).
- **Ordering caveat:** the promo's arming component should not assume it hydrates the store itself. Either (a) subscribe and wait for `hydrated && consent !== null` (works regardless of who hydrates), or (b) call `useConsentStore.getState().hydrate()` defensively if `!hydrated` (idempotent — re-reads the same cookie). `ConsentBanner` already does (b); doubling it is harmless but (a) is cleaner since the arming component lives in the same `AppInit` subtree as nothing that guarantees order vs `ConsentBanner` (`ConsentBanner`'s mount point is **not** in `AppInit` — see §8/§9 caveat).

**This is the key design insight:** both triggers collapse to one condition. Arm a 5s timer the moment **`hydrated === true && consent !== null`** becomes true.
- First visit: that flips true when the banner resolves (`setConsent` via `applyConsent`) → timer starts 5s after the decision. ✔
- Return visit: that is already true right after `hydrate()` on load → timer starts ~immediately after load → fires 5s after load. ✔
One subscription, one derived condition, both required behaviors.

---

## App-level mount point

### 8. Where app-level client initializers mount

**`src/components/client/initializers/AppInit.tsx`** (`'use client'`) is the home for app-level side-effect components. It is mounted once at `app/[locale]/layout.tsx:48` (inside `TooltipProvider`, inside `NextIntlClientProvider`). What it renders today (`AppInit.tsx:47-62`):
- `<ChatsInit />`, `<PushInitializer />`, `<ChatsWatcher />`, `<InitFavoritesStore />`, `<UseTokenRefresh />`, `<GA4UserIdSync />`, **`<AccountStateDialogsInit />`** (the dialog-opening precedent), `<SyncCardSizeFromCookie />`, `<SyncThemeFromCookie />`, `<SyncThemeFromSystem />`, `<ScrollToTop />`, `<GA4ErrorListener />`.

AppInit itself also runs two `useEffect`s (locale sync `:24-26`; pre-consent globalCookie self-heal `:32-45`).

**This is where the new arming component (`AppPromoInit` or similar) should be added** — a sibling `<AppPromoInit />` in the `AppInit` return, modeled on `AccountStateDialogsInit`.

### 9. Does this mount point have access to `useConsentStore`?

**Yes, trivially.** `useConsentStore` is a plain global Zustand store with no provider (§5), so any client component anywhere can read it via the hook or `useConsentStore.getState()` / `.subscribe()`. AppInit and all its children qualify.

**Caveat worth flagging to Mastermind:** `ConsentBanner` is **not** mounted inside `AppInit`. Grep shows `ConsentBanner` is imported in `src/components/client/consent/` and `app/[locale]/owner/cookies/page.tsx`; the banner that hydrates the store mounts elsewhere in the tree, and the only code that calls `hydrate()` is `ConsentBanner` (`ConsentBanner.tsx:67`). The arming component must therefore **not** depend on hydration having already happened when it mounts — it must subscribe and wait, or hydrate defensively (§7). It does not need the banner as a sibling; the store is global.

---

## localStorage usage

### 10. Existing localStorage usage + SSR-safety pattern

There is **no shared localStorage wrapper/helper.** Raw `window.localStorage` is used directly. Only two non-`node_modules` files touch localStorage:
- **`src/components/client/buttons/FloatingButton.tsx`** — a dev-only draggable button persisting its position. This is the canonical pattern to copy. `loadPos()` (`:17-30`) and `savePos()` (`:32-38`):
  - **SSR guard:** `if (typeof window === 'undefined') return null;` (`:18`) before any access on the read path; reads/writes wrapped in `try/catch` to tolerate corrupt/unavailable/quota-exceeded storage.
  - The component holds the value in `useState<Pos | null>(null)` and only reads localStorage inside a mount `useEffect` (`:84-87`) — i.e. client-only, after mount, never during render.
- **`src/lib/consent/gating.ts`** — the match is only a comment mentioning "localStorage write"; it does not actually use localStorage (it reads the consent store). Not a precedent.

So the promo timestamp helper should: guard `typeof window === 'undefined'`, wrap in `try/catch`, store a number (epoch ms) — mirroring `FloatingButton`'s `loadPos`/`savePos` exactly. Suggested key namespace style: the repo uses `'oglasino:<feature>:<thing>'` (`FloatingButton.tsx:9` → `'oglasino:floating-button:pos'`). e.g. `'oglasino:app-promo:last-shown'`.

### 11. Precedent for *functional* (non-consent-gated) localStorage?

**Yes.** `FloatingButton`'s position write is **ungated** — it does not check `isPreferenceConsentGranted()` before writing. So there is precedent for functional client-persisted state outside the consent gate. (`FloatingButton` is dev-only, so it is a thin precedent, but it establishes the mechanism.)

**However — flag for Mastermind (consent-classification question, not a code blocker):** the consent system explicitly contemplates gating *persistence-style side effects* (`gating.ts:11-17` says `isPreferenceConsentGranted()` is "safe to call from any non-React code path that gates a persistence-style side effect (cookie write, **localStorage write**)"). The brief states the promo timestamp must be **functional (no consent gate)**. That is a defensible classification — a frequency-capping timestamp for a first-party UI promo is arguably "strictly necessary" UX state, not preference/analytics — but it is a **policy call** about the cookie/consent posture, not an engineering one. The spec should state explicitly that the promo timestamp is classified necessary/functional and is intentionally written without a preference-consent gate, so it is not later mistaken for a gating violation. Everything currently persisted by the consent flow itself routes through `og_consent` (a cookie) + the gate; this would be the first *product* feature to persist ungated localStorage.

---

## SVG assets

### 12. How SVGs are referenced today

SVGs are **inline React components**, one per file, under `src/components/icons/`. There is **no** `<img src="*.svg">` and no SVG-import-as-component loader configured — grep for `.svg` references in `src`/`app` returned nothing. Each icon is a function component returning a literal `<svg>…</svg>`.

Directly relevant: the two store badges **already exist as inline-SVG components**:
- `src/components/icons/AppleStoreGetIt.tsx` — `const AppleStoreGetIt = () => (<svg width="120" height="40" …>…</svg>)`, default export (`:1-82`).
- `src/components/icons/GooglePlayGetIt.tsx` — same shape, `<svg width="135" height="40" …>`, default export (`:1-78`).

They are currently rendered only in the footer — `src/components/server/layout/Footer.tsx:2-3` (imports), `:80-87`:
```tsx
<div className="flex gap-2">
  <div className="cursor-pointer xl:hover:scale-105"><GooglePlayGetIt /></div>
  <div className="cursor-pointer xl:hover:scale-105"><AppleStoreGetIt /></div>
</div>
```
Note the footer badges are **non-interactive** (no `<a>`/`href`/`onClick`) — see the open issues.md entry "Web: app-store/play-store footer badges are dead" (2026-06-04).

**Implication for the brief:** if Igor's two new badge SVGs are meant for the promo dialog, the established pattern is a new inline-SVG component under `src/components/icons/` (one per badge), imported into the dialog body. If the promo is acceptable reusing the **existing** `AppleStoreGetIt` / `GooglePlayGetIt`, no new asset components are needed at all — but the brief says "the two store-badge SVGs Igor will supply," implying new artwork, so the spec should clarify: new components vs. reuse the footer pair. Either way the dialog body links each badge to its store URL with an `<a href>` (the footer's missing affordance — and these store URLs do not exist yet per the issues.md "blocked on store listings" note, so the promo may ship with placeholder/disabled links or this is a dependency).

---

## Translations

### 13. Translation mechanism + namespace + new keys

- **Mechanism:** `next-intl`. Messages are **not** local JSON files in this repo — they are loaded per-locale via `src/i18n/request.ts` (`getRequestConfig` → `loadAllNamespaces(oglasinoLocale)`), and the underlying strings are **seeded from the backend SQL** (per conventions Part 6 Rule 3; adding keys to the SQL seed is the Backend agent's job, not web's — web only identifies the missing keys).
- **Consumption:** `useTranslations(TranslationNamespaceEnum.<NS>)`. The enum is `src/translations/types/TranslationNamespaceEnum.ts` (`DIALOG = 'DIALOG'` at `:12`).
- **Namespace for promo dialog copy:** **`DIALOG`** (conventions Part 6: "`DIALOG` — modal and dialog strings"). `InfoDialog` already reads `DIALOG` (`InfoDialog.tsx:37`); `AccountStateDialogsInit` reads `DIALOG` for titles/bodies and `BUTTONS` for button labels (`:34-35`). The promo close button can reuse the existing **`DIALOG` `button.close.label`** (`InfoDialog.tsx:64`, `DrawerDialog.tsx:104,135`) — no new button key needed if the default close label is acceptable.
- **Example of a dialog reading strings:** `AccountStateDialogsInit.tsx:53-62` (`tDialog('banned.dialog.title')`, body paragraphs), and `InfoDialog.tsx:64` (`tDialog('button.close.label')`).

**New keys we'd need (DIALOG namespace), to hand to Backend via Igor:**
| Key (DIALOG) | Purpose |
|---|---|
| `app.promo.title` (or `app_promo.title`) | dialog heading |
| `app.promo.body` | body copy ("get the Oglasino app", etc.) |
| (close button) | reuse existing `button.close.label` — **no new key** unless a bespoke label ("Maybe later" / "Not now") is wanted, in which case `app.promo.dismiss.label` |

Watch **Part 6 Rule 2 (no parent/child collision):** do not create both `app.promo` (leaf) and `app.promo.title` (nested). Use a suffix on any would-be-leaf parent (`.label`/`.title`/`.text`). The proposed keys above are all nested leaves under `app.promo.*`, so they are fine as long as no bare `app.promo` leaf is introduced. Final key naming/casing is the spec's call (Backend seeds it); web just needs the list. Native-translator review (RS/RU/CNR) will be owed on the new strings, as with other recent additions.

---

## 14. Anything that makes this harder than "mount, arm a timer, open a dialog"

Mostly reassuring, with a few real concerns:

- **No new dialog component or DialogId strictly required.** The cleanest build reuses `INFO_DIALOG` exactly as `AccountStateDialogsInit` does — pass `dialogTitle`, a `dialogDescription` JSX block containing the two badge SVGs, and (optionally) a custom `closeButtonLabel`. If the promo needs layout `InfoDialog` can't express (two side-by-side badge links with specific styling), a dedicated `AppPromoDialog` + new `DialogId.APP_PROMO_DIALOG` + a `DIALOGS` registry entry is a small, well-trodden addition. **Spec should pick one** (Part 4a: don't add a new dialog component "in case"). Recommendation: try `INFO_DIALOG` first.
- **`AppInit` (the locale layout) does NOT remount on same-segment navigation** — confirmed by the comment at `AccountStateDialogsInit.tsx:74-76` ("the locale layout does not remount on same-segment navigation"). Good: the arming component stays mounted across in-app navigation, so a single mount-effect + subscription is correct and won't re-arm on every route change. It **can** remount on a locale switch (`/sr` → `/en`), which would re-run the arming logic — the **localStorage 5-day gate is what prevents a re-show** in that case; rely on it, don't add navigation-specific guards.
- **React StrictMode double-fire (dev).** Effects mount→unmount→mount in dev. Mitigations, all standard and already in-repo: (a) the timer `useEffect` returns `clearTimeout` cleanup (like `InfoDialog.tsx:44-48` and `FloatingButton`); (b) **write the localStorage timestamp at the moment the dialog opens** and re-check the 5-day gate immediately before `openDialog`, so even a double-fire opens at most once and the second pass sees a fresh timestamp; (c) `openDialog` is idempotent (just sets store state). With (a)+(b) the double-fire is a non-issue.
- **Timer cleanup on unmount.** The 5s timer must be cleared if the component unmounts (locale switch / navigation away mid-countdown) to avoid `openDialog` firing after unmount. Standard `return () => clearTimeout(id)`.
- **Subscription style.** Use the `ConsentBanner` pattern (`useConsentStore.subscribe` inside a mount-only effect, `:47-76`) rather than a `useConsentStore((s)=>…)` selector that re-renders. The arming component returns `null` and should not re-render on unrelated store churn. Either works; the subscription is the established precedent for "do X once when the decision lands."
- **One-shot vs. gate.** "Shows once, then every 5 days" = the localStorage timestamp **is** the once-gate and the frequency-cap in one value. Logic: on the arm condition, read `lastShown`; if absent or `now - lastShown >= 5 days`, start 5s timer; on fire, re-check the gate, write `now`, `openDialog`. No separate "hasShownOnce" flag needed.
- **The 5-day constant.** Per Part 4a "configuration is for values that vary" — 5 days has one setting and no foreseeable second; make it a named module constant (`const PROMO_MIN_INTERVAL_MS = 5 * 24 * 60 * 60 * 1000`), not config. Same for the 5s delay (`const PROMO_DELAY_MS = 5_000`).
- **Dependency: store-listing URLs do not exist yet** (issues.md 2026-06-04 "blocked on store listings"). If the badges in the promo are meant to be tappable links to the App Store / Play Store, those URLs are a hard external dependency. The promo can ship with the badges as non-links (matching the current footer) or behind the same `APP_DEEP_LINK_ENABLED=false` posture — spec must decide. This is the only thing that could block a fully-functional promo.
- **Consent-classification policy call** (see §11) — the spec must explicitly bless the ungated localStorage timestamp as functional/necessary, so it is not later read as a consent-gate violation.

---

## Recommended shape (for the implementation brief)

1. **`src/components/client/initializers/AppPromoInit.tsx`** (`'use client'`, returns `null`) — modeled on `AccountStateDialogsInit`. Mount-only effect that:
   - subscribes to `useConsentStore`; defensively `hydrate()`s if `!hydrated`;
   - when `hydrated && consent !== null` first becomes true, checks the localStorage 5-day gate; if eligible, arms a `PROMO_DELAY_MS` (5s) timer;
   - on fire: re-check gate → write `now` to `'oglasino:app-promo:last-shown'` → `openDialog(DialogId.INFO_DIALOG, { dialogTitle, dialogDescription: <badges/>, … })`;
   - cleans up the timer and unsubscribes on unmount.
2. **Add `<AppPromoInit />`** as a sibling in `AppInit.tsx`'s return.
3. **localStorage helper** — a tiny `readLastShown()` / `writeLastShown()` pair (SSR-guarded + try/catch, copying `FloatingButton`), either inline in `AppPromoInit` or a small `src/lib/appPromo/storage.ts`.
4. **Badges** — reuse existing `AppleStoreGetIt`/`GooglePlayGetIt`, or new inline-SVG components under `src/components/icons/` from Igor's artwork (spec decides).
5. **Translations** — `DIALOG` namespace: `app.promo.title`, `app.promo.body` (+ optional `app.promo.dismiss.label`); reuse `button.close.label`. Web identifies; Backend seeds the SQL; native review owed.
6. **Constants** — `PROMO_DELAY_MS = 5_000`, `PROMO_MIN_INTERVAL_MS = 5 * 24 * 60 * 60 * 1000`.

No new dialog component or `DialogId` is needed if `INFO_DIALOG` suffices.
</content>
</invoke>
