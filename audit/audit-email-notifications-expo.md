# Audit — email-notifications (oglasino-expo)

**Repo:** oglasino-expo
**Branch:** new-expo-dev (read-only; no code changed, no commits)
**Date:** 2026-06-02
**Scope:** Does the mobile login path enforce email verification, and what does an unverified login attempt look like on mobile today. Code is ground truth.

## TL;DR

Mobile does **not** enforce email verification anywhere. It never reads `emailVerified`, never calls `sendEmailVerification`, and has no verify UI (only an empty placeholder screen). Email registration logs the user straight in. The only post-sign-in rejection mobile honors is the backend `USER_BANNED` / `EMAIL_BANNED` 403 — and that handling lives on only **one** of the two sign-in code paths. There is **no inbound deep-link wiring** of any kind, so a web `/verify` URL cannot accidentally open or interfere with the app. The verification gate is therefore entirely a backend + web concern today; mobile's only seam is the `/auth/firebase-sync` POST.

---

## Section 1 — Mobile login + verification gate

### The email login path (trace)

1. `src/components/dialog/dialogs/LoginDialog.tsx:61` `loginInternal` — structural validation only (`loginValidationFailures`), then calls the store action.
2. `src/lib/store/authStore.ts:94` `login({email, password})` → `loginUserFirebase(email, password)`; on success `set({ user: backendUser })`; on throw `set({ error: mapAuthError(err) })`.
3. `src/lib/services/authService.ts:165` `loginUserFirebase` → `signInWithEmailAndPassword(auth, email, password)` then `buildUserSession(res.user)`.
4. `authService.ts:157` `buildUserSession` → `ensureUserInFirestore(firebaseUser)` + `syncUserToBackend(firebaseUser)`.
5. `authService.ts:143` `syncUserToBackend` → `POST /auth/firebase-sync` → returns `AuthUserDTO`.
6. Back in `LoginDialog.tsx:72`, a `useEffect` watching `loading` closes the dialog once `user` is set. User is in.

### Is there any `emailVerified` check after sign-in?

**No.** Repo-wide grep for `emailVerified`, `sendEmailVerification`, and `reload()` across `src/` and `app/` returns **zero hits**. The `FirebaseUser` returned by `signInWithEmailAndPassword` carries `.emailVerified`, but mobile never reads it. An unverified email user who can authenticate against Firebase is **logged straight in** — no "verify your email" state, no block, no gate.

### Where is `emailVerified` read, and from where?

Nowhere. It is neither read from the authoritative Firebase user record nor from a client-trusted value — it is simply **not consumed**. There is consequently no trust-boundary (conventions Part 11) decision touching `emailVerified` on mobile today.

### The one rejection mobile *does* honor — and a path asymmetry

The only post-sign-in account-state gate is the banned check, and it exists on only one of the two sync paths:

- **`initAuthListener` (the `onIdTokenChanged` auto-login path), `authStore.ts:267-272`:** catches the sync error and, if `isErrorWithCode(err, 'USER_BANNED')` or `'EMAIL_BANNED'`, sets `accountBanned: true` and calls `auth.signOut()`. Any **other** error code falls through to `logServiceError` and **leaves the Firebase session alive** (no sign-out).
- **The explicit `login()` action, `authStore.ts:100-104`:** has **no** code-based branch at all. A backend rejection at `/auth/firebase-sync` is an axios error (no `auth/*` code), so `mapAuthError` falls through to its `err.message ?? 'Login failed. Please try again.'` default (`authErrors.ts:33-34`) — surfaced as a raw/generic string, not a translated state.

Note also that `signInWithEmailAndPassword` itself fires `onIdTokenChanged`, so a successful email login triggers **two** `syncUserToBackend` calls — once directly via `buildUserSession`, once via `initAuthListener`. Both hit `/auth/firebase-sync`. This matters for Section 4: whichever path a future "email not verified" rejection lands on determines whether the user is signed back out or left in a half-authenticated state.

---

## Section 2 — Mobile registration

### The email registration path (trace)

1. `src/components/dialog/dialogs/RegisterDialog.tsx:68` `registerInternal` — structural validation only, then the store action.
2. `authStore.ts:111` `register({email, password, displayName})` → `registerUserFirebase(email, password)`; sets `backendUser.displayName = displayName`; `set({ user: backendUser })`; fires `trackAuthEvent`.
3. `authService.ts:171` `registerUserFirebase` → `createUserWithEmailAndPassword(auth, email, password)` then `buildUserSession(res.user)` — same Firestore-ensure + `/auth/firebase-sync` as login.
4. `RegisterDialog.tsx:29-33` `useEffect` closes the dialog the moment `user` is truthy.

### Where does the user land? Does mobile call `sendEmailVerification`?

After signup the user is **immediately logged in** — `user` is set, the register dialog closes, the user is dropped into the authenticated app. There is **no `sendEmailVerification` call anywhere** (grep: zero hits). Mobile sends no verification email and shows no "check your inbox" state.

### The placeholder verify screen

`app/owner/dashboard/account-verification.tsx` exists but is a **stub** — it renders only a `<Text>AccountVerificationScreen</Text>`, no logic. It is reachable: `src/components/user/UserMenu.tsx:177` renders a "verify.account" button that calls `navigateToValidDashboard('/dashboard/account-verification')`. So there is a live entry point to an empty screen. Whatever mobile-side verification UX (if any) gets designed later, this is the existing — currently inert — slot. (Flagged in Section 4 / For Mastermind.)

---

## Section 3 — The email → browser → app round-trip

### Is there any inbound deep-link wiring today?

**No.** Confirmed by grep across `src/` and `app/`:

- No `Linking.getInitialURL`, no `Linking.addEventListener`, no `useURL` / `createURL` (expo-linking).
- No expo-router linking config, no `intentFilters` (Android) and no `associatedDomains` / `CFBundleURLTypes` app-link entries (iOS) in `app.config.ts`.
- Every `Linking.openURL` call in the codebase is **outbound** (e.g. `tel:` in `CallUserButton.tsx:65`, external sites in `SupportButton.tsx`, `HardUpdateScreen.tsx`, `SoftUpdateModal.tsx`, message link runs in `Message.tsx:45`). None register the app as a handler for incoming URLs.

### Could a `/verify` URL accidentally match a deep-link handler?

**No — for two independent reasons.**

1. **No handler exists.** There is no inbound deep-link surface to match against in the first place.
2. **Scheme mismatch even if one existed.** The app's custom scheme (`app.config.ts:40`) is `oglasino` in production (`oglasino-preview` / `oglasino-dev` in the other tiers). The verification link is `https://oglasino.com/[locale]/verify` — an `https://` URL on the `oglasino.com` domain. Opening the app for that URL would require iOS **Universal Links** (`associatedDomains: applinks:oglasino.com`) or Android **App Links** (`intentFilters` with `autoVerify` on the `https` host) — **neither is configured**. A custom-scheme URL would have to look like `oglasino://verify`, which a web `https://oglasino.com/...` link never produces.

**Conclusion:** the email → browser → app round-trip is unobstructed. The `/verify` link opens in the browser exactly as the feature intends; nothing in the current app intercepts it, assumes a different flow, or would mis-handle it. After verifying in the browser, the user returns and logs in normally.

---

## Section 4 — Seams

**What mobile assumes about the backend gate.** Mobile treats `/auth/firebase-sync` as the single backend chokepoint and currently special-cases only `USER_BANNED` / `EMAIL_BANNED` (and only on the `initAuthListener` path). *Assumption:* the backend will enforce the verification gate at `firebase-sync` and reject an unverified user there. *To confirm before building:* (a) the exact error code + envelope the backend emits for "email not verified" (so mobile can branch on a code, per the error contract, rather than rendering `err.message`); and (b) that the gate lands on the `firebase-sync` response — given the two-sync asymmetry in Section 1, an unhandled non-banned 403 today leaves a live Firebase session (no sign-out) on the listener path and a raw error string on the explicit-login path. A verification gate will need handling on **both** paths (likely a sign-out + a translated "verify your email" surface), not just the `initAuthListener` banned branch.

**What mobile assumes about the web verify page.** Mobile assumes `oglasino.com/[locale]/verify` is the *sole* verification surface and that mobile needs no verify UI of its own (it has only the inert `account-verification.tsx` stub). *To confirm:* that completing verification on the web page flips backend/Firebase state such that a fresh mobile login then succeeds with **no** client-side `emailVerified` read required on mobile — i.e. the gate is fully server-evaluated and mobile stays a pure consumer.

---

## Out-of-scope flags (Part 4b)

- **`account-verification.tsx` is a reachable empty stub** (`app/owner/dashboard/account-verification.tsx`, entry at `UserMenu.tsx:177`). Severity low. Either it's the intended slot for a future mobile verification-status screen, or it's dead UI to remove. Not touched (read-only audit; out of scope).
- **`UserInfoDTO.isVerified`** (`src/lib/types/user/UserInfoDTO.ts:11`, rendered as a badge in `ProductUserDetails.tsx:141`) is a **public-profile "verified seller" badge on another user**, unrelated to email-verification of the current user. Noted only so the two `verified` concepts aren't conflated during spec/seam design. Severity n/a.
