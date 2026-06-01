# Consent Mode — Mobile (consent UI v1 + consent-field cleanup)

**Status:** `shipped` (code-complete and merged-ready on `new-expo-dev`; on-device runtime verification — string resolution + privacy-link navigation against a seeded build — is carried by the Ψ pass, like this branch's other `verifying` items)
**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Mastermind chat:** 2026-05-30 (merged chats C + G)
**Spec authority:** this document, for mobile. The web reference is [consent-mode-v2.md](consent-mode-v2.md); where this doc and the web spec differ, this doc governs mobile (mobile has no cookies, no SSR, no Google Consent Mode).

---

## Summary

Two pieces of work, merged into one chat because they touch the same screen.

**Part C — cleanup.** The mobile settings screen (`app/owner/dashboard/user.tsx`) still carries consent leftovers from before the web Consent Mode v2 work: a dead `allowPreferenceCookies` field, two cookie-label toggle sections whose translation keys the backend deleted (so they render blank/raw today), and some dead types and small bugs. Remove all of it.

**Part G — consent UI v1.** Mobile has no consent surface at all. Build one: a single "Allow analytics" on/off toggle, stored on the device, that the future analytics feature (chat F) reads before turning analytics on. This is deliberately *not* a port of the web cookie-consent system — see § Why mobile is different.

Part C runs first (cleans the screen), then Part G builds on the cleaned screen.

---

## Why mobile is different from web

The web consent system models four Google Consent Mode signals (`analytics_storage`, `ad_storage`, `ad_user_data`, `ad_personalization`) plus a `preference` axis, all stored in a browser cookie. Almost none of that applies to a phone:

- **No cookies.** Mobile stores data in AsyncStorage, not cookies. The entire `og_consent` cookie mechanism, the SSR `<head>` snippet, and the legacy-cookie migration have no mobile analog.
- **No Google Consent Mode.** Web gates analytics *declaratively* — it tells Google "analytics denied" and Google honors it; there is no `if (granted) startAnalytics()` code branch. Mobile analytics is just `setAnalyticsCollectionEnabled(true/false)`, a plain on/off switch. There is no declarative layer to delegate to.
- **Mobile's stored preferences don't need a consent gate.** Mobile already saves base-site, language, theme, card-size, auth state, push timing, and the translation cache to AsyncStorage with no gate. These are either necessary for the app to function or preferences the user set themselves — "strictly necessary" / "functional" data under GDPR, which does not require consent. Web gates its *cookie* persistence of UI preferences; mobile has no equivalent requirement.

The one thing that legally needs consent is **analytics** — data collected to measure the user rather than to serve them. That, and only that, is what mobile's consent toggle gates.

**The consequence for the gating shape:** web applies its imperative `if (granted)` gate to *cookie persistence*, and lets Google handle analytics declaratively. Mobile has no declarative path, so mobile applies that same imperative `if (granted)` shape directly to **analytics SDK init**. Mobile mirrors web's *gating pattern* but points it at the analytics signal — the opposite axis from where web applies it. This is the single most important thing to get right.

---

## Consent model

One axis. Two states. Stored on the device.

```ts
// src/lib/types/consent/ConsentDecision.ts  (new — mobile-native, NOT the web ConsentData)
export type ConsentDecision = {
  analytics: boolean;     // the only user-controlled signal
  decidedAt: number;      // unix seconds; 0 = no decision yet
  version: 1;             // future-proofing, mirrors web's versioned shape
};
```

- `analytics: false` is the default before any decision (analytics off until the user opts in).
- `decidedAt: 0` is the "no decision yet" sentinel — distinguishes "never asked" (show the first-run prompt) from "user chose off" (don't re-prompt). Mirrors web's `decidedAt: 0` convention.
- This is a **new mobile-native type**. Do **not** resurrect or reuse the dead `ConsentData` / `GlobalCookie` types (Part C deletes them) — those carry web cookie signals that mean nothing on a device.

### Default state

Before the user decides: `{ analytics: false, decidedAt: 0, version: 1 }`. Analytics off, no decision recorded, first-run prompt pending.

### After a decision

- User opts in → `{ analytics: true, decidedAt: <now>, version: 1 }`.
- User opts out → `{ analytics: false, decidedAt: <now>, version: 1 }`.

Both record a real `decidedAt`, so the first-run prompt won't show again.

---

## Storage

Follow the mobile house style: a dedicated per-concern module with its own key constant calling AsyncStorage directly. (Confirmed pattern: `authStorage.ts`, `softUpdateDismissal.ts`. Do **not** use the generic `userPreferenceStorage` wrapper — it has zero callers and is not the convention.)

src/lib/storage/consentStorage.ts

- key constant: 'consent_decision'
- getConsentDecision(): Promise<ConsentDecision | null>   // null = absent
- setConsentDecision(d: ConsentDecision): Promise<void>

The read returns `null` when the key is absent (first launch ever). The caller treats `null` as "no decision yet" — same meaning as `decidedAt: 0`.

A reactive store is also needed so the toggle UI and the gate read the same live value:

src/lib/store/useConsentStore.ts  (new)

- state: decision: ConsentDecision | null, hydrated: boolean
- hydrate(): reads consentStorage into the store, sets hydrated = true
- setDecision(next): writes consentStorage AND updates the store (write-through)

This mirrors the existing auth-store pattern (Zustand + AsyncStorage). Hydrate on app start so the gate has a value to read.

---

## The gate (the signal chat F reads)

One exported, synchronous, default-deny helper:

```ts
// src/lib/consent/analyticsGate.ts  (new)
export function isAnalyticsConsentGranted(): boolean {
  try {
    return useConsentStore.getState().decision?.analytics === true;
  } catch {
    return false;
  }
}
```

- Returns `true` only when the stored decision has `analytics === true`.
- Absent decision, not-yet-hydrated, `null`, or any throw → `false`. Default-deny.
- This mirrors web's `isPreferenceConsentGranted()` shape exactly, but reads the analytics signal.

**Chat F's contract:** F's analytics SDK init reads this gate and only initializes analytics when it returns true (combined with F's own ATT check on iOS — see § Boundary with chat F). The init mounts as a new child of `AppInit` (`src/components/init/AppInit.tsx`), which runs only once `bootStatus === 'ready'`, alongside `PushNotificationsInit`. This spec defines the gate; F consumes it. F must also re-check the gate when the user changes the toggle (so flipping analytics off stops collection without an app restart) — the store is reactive, so F subscribes to it.

---

## UI surfaces

### First-run prompt

Shown once, when there's no stored decision (`null` / `decidedAt: 0`). It is a one-time overlay rendered at the **first `ready` render**, gated on `hydrated && decision === null`. It mounts as a `ready`-gated sibling in `app/_layout.tsx` (alongside the other visible overlays — `SoftUpdateModal`, the boot `LoadingOverlay`, `DialogManager`), **not** as an `AppInit` child (that headless slot is where chat F's analytics init goes). It asks the user to allow or decline analytics, with a link to the privacy policy (`/privacy`, already exists, fetches markdown over the network).

It is placed at `ready`, **not** at `intro-picker` / in `BaseSiteSelector`, for two reasons: (a) `/privacy` only resolves once the portal `<Stack>` is mounted, which the boot-redesign invariant restricts to `ready`/`updating` — at `intro-picker` there is no navigator and the privacy link would be dead; (b) an existing user with a stored base site boots straight to `ready` and never enters `intro-picker`, so an `intro-picker`-slotted prompt would skip every existing user with no decision, whereas the `ready` placement reaches them. (Mastermind amended the placement from the original `intro-picker` slot after implementation surfaced this collision — see [decisions.md](../decisions.md) 2026-05-31 entry.)

The prompt must let the user proceed either way — opt in or opt out — and records the decision so it never shows again. Unlike web's first-run banner, it does **not** need to block the whole app; declining is a first-class choice, not a dismissal to avoid.

### Settings toggle

A single "Allow analytics" switch the user can change any time. Two options for where it lives:

- **Option A (simplest):** add it to the existing settings screen `user.tsx`, after Part C has cleaned that screen up.
- **Option B (cleaner long-term):** a new dedicated screen `app/owner/dashboard/privacy.tsx`, linked from the account section of the sidebar (`dashboardNavigations.tsx`). No such slot exists today; this creates one.

**Recommendation: Option A for v1** — it's the smallest change and the screen is already being edited by Part C. A dedicated privacy screen can come later if the surface grows. The implementing engineer picks A or B in the brief; A is the default.

Immediate-write semantics (matching web): flipping the toggle calls `setDecision(...)` right away — no Save button. The toggle's value reads live from `useConsentStore`.

### What the consent UI should disclose

The prompt and/or settings screen should make clear, in plain language, that analytics is optional and off unless enabled. Per the privacy discussion in this chat: the consent UI gates analytics, but the app also stores functional data (base-site, language, theme, etc.) that doesn't require consent. The privacy policy is the place that discloses the full stored-data list — see § Privacy-policy dependency.

---

## Translations

New strings go in the **COOKIES** namespace. This namespace is already in mobile's translation fetch list (the boot loop fetches all 20 namespaces, COOKIES included), so **mobile needs zero fetch or namespace changes** — seed the strings backend-side and mobile picks them up automatically.

Strings needed (final EN copy authored in the brief; RS/RU/CNR seeded as placeholders pending native-translator review, per the existing project precedent):

COOKIES namespace (new mobile consent keys):
mobile.consent.prompt.title
mobile.consent.prompt.body
mobile.consent.allow.label
mobile.consent.decline.label
mobile.consent.settings.label          (the settings toggle label)
mobile.consent.settings.description
mobile.consent.privacy.link.label

(Exact key list finalized in the backend seed brief. Keys are namespaced `mobile.consent.*` to avoid colliding with the web consent keys already in COOKIES.)

The backend must emit a translation checksum for the COOKIES namespace covering the new keys (mobile's boot freshness check expects a checksum for every known namespace).

---

## Part C — cleanup scope (runs first)

All in `app/owner/dashboard/user.tsx` unless noted. Confirmed against `new-expo-dev` (lines current as of the 2026-05-30 audit; re-confirm at implementation time).

1. **Remove `allowPreferenceCookies` end-to-end.** Backend deleted this field completely (confirmed); mobile sends it (silently ignored) and reads it (always undefined). Remove from:
   - `src/lib/types/user/AuthUserDTO.ts` (the field declaration)
   - `src/lib/types/user/UpdateUserDTO.ts` (the field declaration)
   - `user.tsx`: the `useState`, the response-seeding line, the change-detection comparison, the save-body field, and the Switch (the whole "Preference cookies" toggle block).

2. **Delete the two dead cookie-label sections.** The "Strictly necessary" section and the "Preference cookies" section reference five COOKIES keys the backend deleted (`required.label`, `required.description`, `required.sub.description`, `config.label`, `config.description`). These render blank/raw today — a live visible regression. Deleting both sections removes all five references and fixes the regression. (The "Strictly necessary" section was a display-only copy of a web pattern; it gates nothing on mobile.)

3. **Delete dead types.** `src/lib/types/cookie/ConsentData.ts` and `src/lib/types/cookie/GlobalCookie.ts` — a closed dead chain (ConsentData imported only by GlobalCookie; GlobalCookie imported by nothing). Confirmed zero live consumers. Delete both.

4. **Fix the promo-emails double-set bug (B2).** In the response-seeding effect, `setAllowPromoEmails(...)` is immediately overwritten with `allowPhoneCalling`'s value, so the promo toggle shows the wrong value on load. Remove the erroneous overwrite line so promo-emails seeds from its own value.

5. **Remove the placeholder text (B17).** The hardcoded `"DODAJ ZA BASE SITE"` scaffold label is a rendered, untranslated dev placeholder. Remove it.

6. **Remove the debug log (B16).** The `console.error(error)` in the seeding catch block violates the no-ad-hoc-logging convention. Remove it (or route through the project logger if one fits).

7. **Surface the save-failure path to the user.** The save function's catch re-throws on a fire-and-forget handler, so a backend save error currently shows the user nothing. Since this function is being edited anyway (fields removed from the save body), add a user-facing error message on the failure path. Small in-file fix.

**Explicitly left alone:** the `allowNotifications` toggle. It is non-functional today (never seeded, never sent), but a future notifications feature will own wiring it. Do **not** wire it, remove it, or fix its seeding here. This leaves one deliberately-dead toggle on the screen after cleanup — a known, intentional gap, recorded here so it is not mistaken for an oversight.

---

## Boundary with chat F (analytics)

- **G owns:** the consent decision (storage, store, the `isAnalyticsConsentGranted()` gate), and the consent UI (first-run prompt + settings toggle).
- **F owns:** the analytics SDK install and init, and the iOS App Tracking Transparency (ATT) prompt. ATT is Apple's separate IDFA gate; it is **not** the same as GDPR analytics consent. F's init gates on both: `(ATT allows on iOS, or Android) AND isAnalyticsConsentGranted()`.
- G does not install any analytics SDK, does not touch ATT, and does not wire `firebaseAnalytics.ts` (which is dead web-SDK code F will replace with a native init).

---

## Privacy-policy dependency (not code)

The consent UI gates analytics, which is the only stored data that legally requires consent. The app also stores functional/preference data (base-site, language, theme, card-size, auth, push timing, translation cache) without a gate — this is permitted as "strictly necessary"/"functional" data, but **disclosure is separate from consent**. The queued privacy-policy review (lawyer pass, already on the docs backlog) should confirm the privacy policy discloses the full stored-data list. This spec does not author privacy-policy copy; it flags the dependency. (Note: not legal advice — the lawyer review is the authority.)

---

## Trust boundary (Part 11)

No trust decision changes. Part C removes a field the backend already ignores. Part G's consent decision is device-local and read only client-side to gate the client's own analytics init — it is never sent to the server or used in any authorization/moderation decision.

---

## Implementation order — Phase 5 brief sequence

1. **Part C cleanup** (one session). Removes `allowPreferenceCookies`, the two dead label sections, the dead types, the B2/B17/B16 fixes, the save-failure message. Leaves `allowNotifications` alone. Result: a clean settings screen. `tsc`/lint/tests green.
2. **Part G build** (one to two sessions, on the cleaned screen). New `ConsentDecision` type, `consentStorage`, `useConsentStore` (hydrate on boot), `isAnalyticsConsentGranted()` gate, the first-run prompt, the settings toggle. References the COOKIES strings.
3. **Backend translation seed** (one backend brief). Seed the new `mobile.consent.*` COOKIES keys in all four locales — EN final, RS/RU/CNR placeholders for native review — plus the COOKIES checksum. (This is the only cross-repo touch; it's additive seeding.)
4. **Docs cleanup.** Set this spec to `shipped`; record the deliberate `allowNotifications` gap and the F-facing gate contract in `decisions.md`.

Brief 1 and brief 3 are independent and can run in parallel; brief 2 depends on brief 1 (clean screen) and brief 3 (strings) being available.

---

## Definition of done

- `allowPreferenceCookies` gone from both mobile DTOs and `user.tsx`; the two dead cookie-label sections deleted; the broken-label regression fixed.
- Dead `ConsentData` / `GlobalCookie` types deleted.
- B2 (promo-emails double-set), B17 (placeholder text), B16 (`console.error`) fixed; save-failure path shows the user an error.
- `allowNotifications` toggle untouched (deliberate, recorded).
- One "Allow analytics" consent decision stored on-device via `consentStorage` + `useConsentStore`; `isAnalyticsConsentGranted()` exported and default-deny.
- First-run prompt shows once when no decision exists; settings toggle changes the decision with immediate write.
- New `mobile.consent.*` strings seeded in COOKIES across all four locales (EN final, others placeholder); COOKIES checksum emitted.
- `tsc --noEmit`, lint, tests green per conventions.
- Spec set to `shipped`; `decisions.md` records the F-facing gate contract and the deliberate `allowNotifications` gap.

---

## Items logged elsewhere (out of this chat's scope)

- **Backend:** `allowPhoneCalling` is accepted on the update request and returned on GET but never written (`DefaultUserFacade.updateCurrentUserData` doesn't call `setAllowPhoneCalling`) — the value is silently dropped on save. Route to a backend chat. Does not block this chat (we don't touch that toggle).
- **Web:** `allowPreferenceCookies` still declared as dead type fields on web's `AuthUserDTO`/`UpdateUserDTO` (contradicts the web spec's "removed end-to-end"); and the web consent migration shim was never built (superseded by cookies-closing). Both are web-side docs/cleanup items for Docs/QA. Do not affect mobile.

---

## Session log

- **2026-05-30 — Brief 1 (Part C cleanup), `oglasino-expo-consent-mode-mobile-1`.** Removed `allowPreferenceCookies` end-to-end (both DTOs + the five `user.tsx` sites), deleted the two dead cookie-label sections (clearing the five backend-deleted COOKIES-key references → broken-label regression gone) and the dead `ConsentData`/`GlobalCookie` types, fixed B2 (promo double-set) / B17 (placeholder) / B16 (`console.error`), and surfaced the save-failure path via the existing toast pattern. `allowNotifications` left untouched (deliberate). `tsc` clean; 313 tests pass.
- **2026-05-31 — Brief 2 (Part G build), `oglasino-expo-consent-mode-mobile-2`.** Built the `ConsentDecision` type, `consentStorage`, `useConsentStore` (hydrated via an unconditionally-mounted `ConsentInit`), the `isAnalyticsConsentGranted()` default-deny gate, and the settings toggle (immediate write-through). Stopped on task 6 (first-run prompt): the spec's `intro-picker` slot collides with the boot-redesign portal-mount invariant (dead privacy link) and skips existing users — surfaced to Mastermind, who amended the placement to `ready`. 321 tests pass (+8).
- **2026-05-31 — Brief 2b (Part G task 6), `oglasino-expo-consent-mode-mobile-3`.** Built the first-run prompt at the amended `ready` placement (one-time `Modal` overlay, gated `hydrated && decision === null`, both choices first-class, resolving `/privacy` link), mounted as a `ready`-gated sibling in `app/_layout.tsx`. Confirmed it reaches existing users (stored-site boot → `ready`). The brief-2 `var` flag was stale — no `var` present. 325 tests pass (+4).
- **2026-05-31 — Brief 3 (backend COOKIES seed), `oglasino-backend-consent-mode-mobile-1`.** Seeded the seven `mobile.consent.*` keys in COOKIES across all four locales (EN final; RS/RU/CNR placeholder, pending native-translator review), COOKIES 33 → 40 keys. The COOKIES checksum is auto-derived at runtime — nothing to hand-edit.
- **2026-05-31 — Docs close-out, `oglasino-docs-consent-mode-mobile-1`.** Corrected § UI surfaces to the `ready` placement; set status to `shipped` (Ψ-pending caveat); recorded the F-facing gate contract, the deliberate `allowNotifications` gap, and the placement amendment in [decisions.md](../decisions.md); updated the `state.md` entry and added a Risk Watch row for the seven `mobile.consent.*` native-review placeholders; archived the four build session summaries.
