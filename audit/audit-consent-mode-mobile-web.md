# Audit — Consent Mode v2 as-built (web) for mobile consent-UI reference

**Repo:** oglasino-web
**Branch:** `dev` (Consent Mode v2 is merged into `dev`; `feature/consent-mode-v2` is its origin branch per `decisions.md`/`state.md`. The `src/lib/consent/` tree is present and complete on `dev` — audited there.)
**Date:** 2026-05-30
**Mode:** READ-ONLY. No files touched, no commits, no mutating test runs.
**Task:** Confirm what the web Consent Mode v2 system actually does — the consent decision-shape and the gating mechanism — so the mobile chat (chat G) can mirror it in a two-state AsyncStorage model. Code is authoritative; spec disagreements flagged.

> **Method note.** Findings below were extracted by a read-only parallel sweep (one agent per section) and then **every cited line was re-read directly from source** before being written here. All file:line citations are from direct reads.

---

## Section 1 — The consent data shape as built

**`ConsentData` lives in `src/lib/consent/types.ts:6-18`.** The built shape matches the spec exactly — all four GA Consent Mode v2 signals plus `preference`, `necessary`, `version`, `decidedAt`:

```ts
// src/lib/consent/types.ts:1
export type ConsentSignal = 'granted' | 'denied';

// src/lib/consent/types.ts:6-18
export type ConsentData = {
  necessary: 'granted';            // :7   locked literal

  analytics_storage: ConsentSignal; // :9  TOGGLEABLE (union)
  preference: ConsentSignal;        // :10 TOGGLEABLE (union)

  ad_storage: 'denied';            // :12  locked literal
  ad_user_data: 'denied';          // :13  locked literal
  ad_personalization: 'denied';    // :14  locked literal

  version: 1;                      // :16  locked literal
  decidedAt: number;               // :17  unix seconds
};
```

- **`DENIED_DEFAULTS`** (`types.ts:23-32`) is the first-time-visitor snapshot: everything `denied` except `necessary`, with **`decidedAt: 0`** as the "no decision yet" sentinel (distinguishes a default-denied snapshot from a deliberate reject-all, which carries a real timestamp — `types.ts:20-22`).
- **`isValidConsent`** runtime guard (`types.ts:38-51`) re-validates every field on read, so a hand-edited / corrupt cookie is rejected back to "no decision."

**Which signals are user-toggleable vs locked (confirmed against type literals AND the actual toggles):**

| Signal | Type | Toggleable? | Where surfaced |
|---|---|---|---|
| `necessary` | `'granted'` literal | **Locked on** | Banner `ConsentBanner.tsx:150` (`disabled checked`); settings `owner/cookies/page.tsx:45` (`disabled checked`) |
| `preference` | `ConsentSignal` union | **Toggleable** | Banner `ConsentBanner.tsx:162-167`; settings `owner/cookies/page.tsx:52-58` |
| `analytics_storage` | `ConsentSignal` union | **Toggleable** | Banner `ConsentBanner.tsx:179-184`; settings `owner/cookies/page.tsx:65-71` |
| `ad_storage` / `ad_user_data` / `ad_personalization` | `'denied'` literals | **Locked off** | No toggle anywhere; banner shows only a "we do not use marketing cookies" note (`ConsentBanner.tsx:187`) |

So the user controls **exactly two** axes: `preference` and `analytics_storage`. The three ad-* signals and `necessary` are compile-time-locked literals.

**Spec vs code:** No disagreement in this section. The built type is identical to the spec's `ConsentData`.

---

## Section 2 — The gating signal (the part mobile will mirror)

### 2.1 / 2.2 — The gating helper and its predicate

Helper is **`isPreferenceConsentGranted()`** — exactly the name and location the spec predicted.

```ts
// src/lib/consent/gating.ts:11-17
export function isPreferenceConsentGranted(): boolean {
  try {
    return useConsentStore.getState().consent?.preference === 'granted';
  } catch {
    return false;
  }
}
```

- **Signature:** `(): boolean` — synchronous, non-React.
- **Returns:** `true` only when `consent?.preference === 'granted'`. Absent consent (first-time visitor), `null`, or any throw → `false`. **Default-deny.**
- **Predicate (verbatim, `gating.ts:13`):** `useConsentStore.getState().consent?.preference === 'granted'`. It reads the **Zustand store**, not the cookie directly.

### 2.3 — Every call site (grep across `src/` + `app/`)

Production call sites (3 files, 4 sites):

| file:line | What it gates |
|---|---|
| `src/components/popups/dialogs/PortalConfigDialog.tsx:89` | `if (isSameTenant && isPreferenceConsentGranted()) updateGlobalCookie('lang', lang)` — gates the **language cookie write** on a same-tenant language switch |
| `src/lib/store/useCardSize.ts:20` | Gates **reading** persisted card sizes from `globalCookie` at store init |
| `src/lib/store/useCardSize.ts:37` | Gates **writing** card size to `globalCookie` in `setSize` |
| `src/lib/store/useTheme.ts:39` | Gates **writing** theme to `globalCookie` in `setTheme` |

Non-production: `src/lib/consent/gating.test.ts` (unit tests of the helper).

In every gated write, the **in-memory Zustand state updates unconditionally**; only the **durable cookie write** is wrapped in the gate. Examples: `useCardSize.ts:36` (`set(...)`) runs before the gated `updateGlobalCookie` at `:37-49`; `useTheme.ts:37` (`set(...)`) + `applyClass` run before the gated write at `:39-43`.

### 2.4 — Is there a separate analytics-specific gate? **No.**

This is the key answer for mobile. **Nothing in web gates a side-effect on `analytics_storage === 'granted'`.** Every occurrence of `analytics_storage === 'granted'` in non-test code is **UI-state derivation, not a gate**:

- `src/components/client/consent/consentDecisions.ts:58` — `analytics: consent.analytics_storage === 'granted'` inside `togglesFromConsent` (seeds banner toggle UI).
- `app/[locale]/owner/cookies/page.tsx:17` — `const analyticsOn = useConsentStore((s) => s.consent?.analytics_storage === 'granted')` (seeds settings toggle UI).

`analytics_storage` is otherwise only **written** (the decision builders) and **emitted** to Google — fed into the SSR default snippet (`app/layout.tsx:53`) and the `gtag('consent','update',...)` call (`src/lib/consent/sideEffects.ts:36`). Web never branches app behavior on it.

**Why:** web enforces analytics consent **declaratively via Google Consent Mode** — GA4 itself honors the `analytics_storage: denied` default and the later `update`, so web needs no imperative "if analytics granted then init GA" branch. The single **imperative** gate (`isPreferenceConsentGranted`) exists only for the **non-Google** persistence side-effects (language / theme / card-size cookies) that Google can't police.

**Net:** the two axes gate two different things — `preference` gates **local UI-preference persistence** (imperative), `analytics_storage` gates **GA** (declarative, no code gate). See "decision shape" + "gating pattern" in *For Mastermind* for the mobile translation.

**Spec vs code:** No disagreement. Code is internally consistent with the spec's gating model.

---

## Section 3 — How a consent decision is captured and applied

### 3.1 — `applyConsent`

```ts
// src/lib/consent/sideEffects.ts:43-46
export async function applyConsent(next: ConsentData): Promise<void> {
  useConsentStore.getState().setConsent(next);  // → writes cookie + updates store
  callGtagUpdate(next);                          // → fires gtag('consent','update',...)
}
```

`setConsent` (the store action) is what actually persists:

```ts
// src/lib/store/useConsentStore.ts:35-47
setConsent: (next) => {
  const prev = get().consent;
  writeOgConsent(next);          // :37 — writes the og_consent cookie
  set({ consent: next });        // :38 — updates in-memory store
  if (prev?.preference !== 'denied' && next.preference === 'denied') {
    clearGlobalCookie();         // :45 — extra side-effect: clears persisted UI prefs on transition INTO denied
  }
},
```

The fired side-effect (`sideEffects.ts:19-41`) sanitizes then calls:

```ts
// src/lib/consent/sideEffects.ts:35-40
gtag?.('consent', 'update', {
  analytics_storage: next.analytics_storage,
  ad_storage: next.ad_storage,
  ad_user_data: next.ad_user_data,
  ad_personalization: next.ad_personalization,
});
```

> **⚠ Spec vs code (flag #1).** The spec's "Client-side `gtag('consent','update',...)`" block (`features/consent-mode-v2.md` lines 239-243) sends `analytics_storage` **and `preference`**. The as-built code sends the **four Google signals** (`analytics_storage` + the three `ad_*`) and **does NOT send `preference`**. The code is **correct** — `preference` is a web-internal axis, not a Google Consent Mode signal, so it has no place in a `gtag` consent update. The spec snippet is wrong. Mobile takeaway: only the analytics axis maps to the analytics SDK; the preference axis is purely local.

### 3.2 — The Zustand consent store (`src/lib/store/useConsentStore.ts:6-51`)

**State:** `consent: ConsentData | null` (`:10`), `hydrated: boolean` (`:11`), `reopenRequested: boolean` (`:17`).
**Actions:** `hydrate()` (`:30-33`), `setConsent(next)` (`:35-47`), `requestReopen()` (`:49`), `clearReopen()` (`:50`).
**Hydrate path:** `hydrate()` calls `readOgConsent()` (the cookie read) and sets `{ consent: existing, hydrated: true }` (`:30-33`). `existing` is `null` when the cookie is absent — that `null` is the signal the banner uses to treat the visitor as first-time.

### 3.3 — The banner CTAs (`src/components/client/consent/ConsentBanner.tsx`)

All paths funnel through `persistAndClose` → `applyConsent` (`:78-82`), **immediately, no separate commit step**:

| CTA | Handler | Values written |
|---|---|---|
| **Accept all** | `:84` `buildAcceptAllConsent()` | `analytics_storage: granted`, `preference: granted`, ad-* denied (`consentDecisions.ts:12-23`) |
| **Reject all** | `:85` `buildRejectAllConsent()` | `analytics_storage: denied`, `preference: denied`, ad-* denied (`consentDecisions.ts:25-36`) |
| **Save my choices** | `:86-87` `buildCustomConsent({preference, analytics})` | reflects the two toggles, ad-* denied (`consentDecisions.ts:38-52`) |
| **Customize** | `:122` `setView('customize')` | **persists nothing** — view switch only |
| **Back** | `:191` `setView('default')` | **persists nothing** — view switch only |

`decidedAt` is stamped at write time by the decision builders (`consentDecisions.ts:10` default clock). First-time drawer is non-dismissible (`:90` blocks close when `mode === 'firstTime'`); the footer-reopened drawer is dismissible (`:98,:101-111`).

### 3.4 — The settings page `/owner/cookies` (`app/[locale]/owner/cookies/page.tsx`)

Three toggles: necessary (`:45`, disabled+checked), preference (`:52-58`), analytics (`:65-71`). **Immediate-write, no Save button:**

```ts
// owner/cookies/page.tsx:25-31
const handlePreferenceChange = (checked) =>
  void applyConsent(buildCustomConsent({ preference: checked, analytics: analyticsOn }));
const handleAnalyticsChange = (checked) =>
  void applyConsent(buildCustomConsent({ preference: preferenceOn, analytics: checked }));
```

Each toggle's `checked` is a live store selector (`:16-17`); both toggleable switches are `disabled={!hydrated}` (`:56,:69`) until `useConsentStore.hydrate()` resolves (`:19-23`). Flipping a toggle writes `og_consent` and fires `gtag update` synchronously — same "decision writes immediately" semantics as the banner.

**Spec vs code (flag #4, minor):** the spec is internally inconsistent about the settings page route — `§Scope` (line 31) and `§Implementation order` (line 405) say `/owner/user`; `§Logged-in user settings` (lines 328, 336) say `/owner/cookies`. **As-built it is `/owner/cookies`** (a dedicated page), matching the later spec section. No action needed beyond noting it.

---

## Section 4 — `allowPreferenceCookies` removal + `firebase-sync` body

### 4.1 — `allowPreferenceCookies` grep: **NOT zero (2 hits).**

> **⚠ Spec vs code (flag #2).** Spec `§Cross-repo seams` (line 417) and the brief expect `allowPreferenceCookies` removed **end-to-end / zero hits**. It survives in two web DTO type declarations:
> - `src/lib/types/user/AuthUserDTO.ts:13` — `allowPreferenceCookies?: boolean;`
> - `src/lib/types/user/UpdateUserDTO.ts:12` — `allowPreferenceCookies?: boolean;`
>
> These are **declarations only** — a full `src/`+`app/` grep finds **no read and no write** of the field anywhere else. The optional `?` means nothing populates it. So the **runtime wire is clean** (the firebase-sync body does not send it — see 4.2), but the field lingers as a **dead type member** on both DTOs. This is a documentation-vs-reality gap plus a cleanup item (see *For Mastermind*).

### 4.2 — `/auth/firebase-sync` request body

```ts
// src/lib/service/reactCalls/authService.ts:140-142
const response = await BACKEND_API.post<AuthUserDTO>('/auth/firebase-sync', {
  ...(displayName !== null ? { displayName } : {}),
});
```

The body is **`{ displayName?: string }` only** — `displayName` is included solely on the registration flow (when `nextRegisterDisplayName` was set, `authService.ts:136-137`), otherwise an empty `{}` is sent. This **matches the spec** (`§Cross-repo seams` line 419) and cross-checks the backend audit's Section 4. No `allowPreferenceCookies` (or any consent field) is sent. **Confirmed: consent is browser-local, no backend mirror on this wire.**

---

## Section 5 — What mobile canNOT reuse (cookie / browser / SSR-specific)

Explicit boundary for the mobile chat. Each item is web-only because its primitive is the cookie/SSR/DOM, with no AsyncStorage analog:

| Web-only part | file:line | Why web-only |
|---|---|---|
| **SSR consent-default snippet** — raw `<script dangerouslySetInnerHTML>` emitting `gtag('consent','default',{...})` + `window.__og_consent_loaded` | `app/layout.tsx:47-56` (snippet string), `:68` (injection) | Mobile has no server-rendered `<head>`, no `window.dataLayer`, no inline-script execution. The "default before any script" requirement is a browser-load-order problem that doesn't exist on RN. |
| **`readConsentForSsr`** — reads `og_consent` from the Next.js cookie store server-side | `src/lib/consent/ssr.ts:42-50` | No SSR pass and no server cookie store on mobile; consent is read directly from AsyncStorage at app start. |
| **`sanitizeForSnippet`** — off-union defense before string interpolation into the SSR snippet | `src/lib/consent/ssr.ts:24-40` | Exists only to harden the inline-`<script>` template; no template, no need. |
| **`og_consent` cookie read/write** — `document.cookie` regex read + `Secure; SameSite=Lax; Path=/; Max-Age` write | `src/lib/consent/cookie.ts:19-31` (read), `:33-45` (write), `:47-53` (delete) | `document.cookie` is the DOM cookie jar. Mobile swaps this primitive for AsyncStorage `get`/`set`/`remove`. **This is the one helper whose *shape* (read→validate→default-deny / write→serialize) mobile should mirror, while replacing the storage call.** |
| **`window.gtag`** consent update + GA4 wiring | `src/lib/consent/sideEffects.ts:34-40`; `app/layout.tsx:48-49,:70-81`; `src/components/client/initializers/GA4UserIdSync.tsx:16-21`; `src/lib/analytics/track.ts:11-20` | `window.gtag` is the GA4 **web** (gtag.js) global. Mobile uses the Firebase Analytics RN SDK and sets analytics-collection-enabled directly; there is no `gtag('consent','update')` call. |
| **`globalCookie` self-heal** (NOT the spec's migration shim — see note) | `src/components/client/initializers/AppInit.tsx:32-45`; store `clearGlobalCookie` on transition-into-denied at `useConsentStore.ts:44-45` | Operates on the `globalCookie` browser cookie; no analog. |

> **⚠ Spec vs code (flag #3 — migration shim).** The spec describes a one-time migration **`globalCookie.cookieConsent` → `og_consent`**, run both client-side in the banner mount (spec lines 130-148) and server-side in `readConsentForSsr` (spec lines 153-173, 230). **This migration is NOT built.** A `src/`+`app/` grep for `cookieConsent` returns **zero hits**; `readConsentForSsr` (`ssr.ts:42-50`) reads `og_consent` only and falls back to `DENIED_DEFAULTS` with **no legacy branch**; the store's `hydrate()` and the banner have **no legacy read** either. What *does* exist is a different, narrower thing: `AppInit.tsx:32-45` self-heals by **clearing** stale `globalCookie` preference fields (`lang`/`theme`/card sizes) when `og_consent.preference !== 'granted'` — a gating-cleanup, not a consent migration. Likely reconciled by the later `cookies-closing` feature, which removed the `cookieConsent` field from the `GlobalCookie` type (see `state.md` "Cookies closing"). **Mobile impact: none** — migration of a legacy cookie has no mobile analog regardless. Flagged for `decisions.md`/spec accuracy only.

---

## Config-file impact

- conventions.md: no change.
- decisions.md: no change required by me. *(Two spec-accuracy items surfaced — see "For Mastermind" flags #1, #2, #3 — drafted there for Mastermind/Docs-QA, not applied by me.)*
- state.md: no change.
- issues.md: no change. *(Adjacent observations drafted in "For Mastermind" for Mastermind triage; I do not write `issues.md`.)*

**Config-file impact: none (read-only audit).**

## Files touched

- None. Read-only audit. The only file created is this audit document (+ its session-summary twins per CLAUDE.md).

## Tests

- None run (read-only). Existing consent unit tests observed but not executed: `src/lib/consent/{gating,cookie,sideEffects,ssr}.test.ts`, `src/lib/store/useConsentStore.test.ts`, `src/components/client/consent/consentDecisions.test.ts`.

## Cleanup performed

- None needed (no code changed).

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code changed.
- Part 4a (simplicity) / Part 4b (adjacent observations): see structured evidence in "For Mastermind".
- Part 6 (translations): N/A this session (no keys added; observed only that consent copy lives in the `COOKIES` and `NAVIGATION` namespaces per spec).
- Part 7 (error contract): N/A — consent has no error wire.
- Part 11 (trust boundaries): confirmed — consent is browser-local, never sent to backend (Section 4.2); not used in any moderation/authorization/state-transition decision.

## Known gaps / TODOs

- None deliberately skipped. All five brief sections answered against source.

---

## For Mastermind

### Distilled "decision shape" (the single most useful output for the mobile model)

**Web gives the user TWO independent, separately-toggled consent axes: `preference` and `analytics_storage`** (both surfaced as distinct toggles in *both* the banner customize view and the `/owner/cookies` settings page; `necessary` + 3 `ad_*` are locked). They are **not** combined into one switch.

But the two axes do **different jobs**, and that is what decides mobile's toggle count:

- **`analytics_storage`** → gates **analytics** (GA4). On web this is enforced **declaratively** by Google Consent Mode (SSR default `denied` + `gtag` `update`), so there is **no imperative code gate** on it.
- **`preference`** → gates **persistence of local UI preferences** (language, theme, card-size cookies) via the imperative `isPreferenceConsentGranted()` gate. This is a *cookie-persistence* concern.

**Recommendation for mobile's two-state (analytics on/off) model:** mobile's single toggle should mirror **web's `analytics_storage` axis**, because mobile's stated gate is analytics-SDK init. Web's `preference` axis is about persisting UI prefs to the cookie store — if mobile persists UI prefs to AsyncStorage unconditionally (or has no such prefs to gate), it needs **only one** consent axis (analytics). If mobile later wants to gate AsyncStorage persistence of UI prefs the way web gates cookies, that would be a *second* axis mirroring `preference` — but that is not implied by "analytics on/off." **One toggle is faithful to the mobile brief; web being two-axis does not force mobile to two.**

### Distilled "gating pattern" (one sentence mobile will mirror)

> A synchronous boolean read of the persisted signal wraps each durable side-effect — `if (isPreferenceConsentGranted()) writeCookie(...)` — where the in-memory state updates **unconditionally** and only the **durable write** is skipped when consent is absent/denied (default-deny on absent).

**Critical translation note for mobile:** web does **not** use this imperative pattern for analytics — it delegates to Google Consent Mode. **Mobile has no Consent Mode equivalent**, so mobile *must* apply this exact imperative-gate shape to its **analytics-SDK init** (`if (analyticsConsentGranted()) initAnalytics()`), keyed on the analytics axis rather than the preference axis. So mobile mirrors web's *gating shape* (`isPreferenceConsentGranted`) but points it at the *analytics signal* — the opposite axis from where web applies the imperative gate. Miss this and mobile will either never gate analytics or gate the wrong thing.

### Spec-vs-code disagreements (all four, with file:line)

1. **`gtag` update payload** — spec sends `preference`; code sends the four Google signals only, no `preference`. Code is correct (`preference` is not a Google signal). `src/lib/consent/sideEffects.ts:35-40` vs `features/consent-mode-v2.md` lines 239-243. **Draft for Docs/QA:** correct the spec's client-side `gtag('consent','update',...)` snippet to drop `preference` and include the three `ad_*` signals.
2. **`allowPreferenceCookies` not removed end-to-end** — survives as dead optional type fields at `src/lib/types/user/AuthUserDTO.ts:13` and `src/lib/types/user/UpdateUserDTO.ts:12` (declared, never read/written; not on the wire). Contradicts spec `§Cross-repo seams` line 417 ("removed end-to-end"). **Draft for `issues.md`** (below).
3. **Migration shim not built** — spec's `globalCookie.cookieConsent → og_consent` migration (spec lines 130-173, 230) does not exist; zero `cookieConsent` references; `readConsentForSsr` (`ssr.ts:42-50`) has no legacy fallback. Likely superseded by `cookies-closing`. **Draft for Docs/QA:** mark the migration-shim section of the consent spec as superseded/removed, cross-referencing `cookies-closing`.
4. **Settings-page route inconsistency in the spec** — `/owner/user` (spec lines 31, 405) vs `/owner/cookies` (spec lines 328, 336); as-built is `/owner/cookies`. Minor. **Draft for Docs/QA:** make the spec consistently say `/owner/cookies`.

### Part 4a simplicity evidence (required)

- **Added (earned complexity):** nothing — read-only audit, no code added.
- **Considered and rejected:** nothing — no implementation choices were on the table.
- **Simplified or removed:** nothing — no code changed.

### Part 4b adjacent observations (drafted for Mastermind triage; I do not write `issues.md`)

1. **Stale store comment.** `src/lib/store/useConsentStore.ts:16-18` says of `reopenRequested`: *"No consumer is wired in this brief… so brief 4 doesn't have to touch this file again."* A consumer **is** now wired — `ConsentBanner.tsx:57` subscribes to `reopenRequested` and `:59` calls `clearReopen()`. **Severity: low** (misleading to a future reader). Out of scope — not fixed.
2. **Dead `allowPreferenceCookies` type fields.** `AuthUserDTO.ts:13`, `UpdateUserDTO.ts:12` — unreferenced optional booleans that contradict the spec's "removed end-to-end." **Severity: medium** (could mislead a future reader into thinking a consent mirror still exists; cross-check that the backend truly no longer emits the field before deleting the web type members). Out of scope — not fixed.
3. **Legacy-user re-prompt (consequence of flag #3).** With no `cookieConsent → og_consent` migration, any user still holding only the legacy `globalCookie.cookieConsent` is treated as a first-time visitor and re-prompted (rather than silently migrated). Given `cookies-closing` removed the `cookieConsent` field from the `GlobalCookie` type, this is likely intended/benign, but worth a one-line confirmation from Mastermind that re-prompting (not migrating) legacy users is the accepted behavior. **Severity: low.**

**Suggested `issues.md` entry (for Docs/QA to apply, if Mastermind agrees):**

> **2026-05-30 — Web `allowPreferenceCookies` type fields not removed end-to-end** — *Severity: medium, Status: open.* `src/lib/types/user/AuthUserDTO.ts:13` and `src/lib/types/user/UpdateUserDTO.ts:12` still declare `allowPreferenceCookies?: boolean`, though Consent Mode v2 spec `§Cross-repo seams` says the field was removed end-to-end and the firebase-sync wire confirms it is never sent. Dead type members; delete after confirming the backend `AuthUserDTO`/`UpdateUserDTO` no longer emit/accept the field. Surfaced by the consent-mode mobile-web audit.
