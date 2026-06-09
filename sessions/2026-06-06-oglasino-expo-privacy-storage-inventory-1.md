# Session summary

**Repo:** oglasino-expo
**Branch:** dev (brief said `new-expo-dev`; the repo is currently on `dev`. Read-only audit — no checkout performed.)
**Date:** 2026-06-06
**Task:** Enumerate everything the mobile app persists locally on the device (for an accurate Privacy Policy).

## Implemented

Read-only audit. No code changes. The inventory below is the core finding, derived from the code — not from the partial list in the brief.

---

## Local-storage inventory

### A. AsyncStorage — keys the app writes explicitly

| Key | Holds (plain terms) | Personal/identifying? | Cleared on logout? |
| --- | --- | --- | --- |
| `base_site` | The user's chosen base site / portal (city + portal selection, full `BaseSiteDTO`). | No — functional/preference. | **No** — persists. |
| `app_language` | The user's chosen UI language (`LanguageDTO`). | No — functional/preference. | **No** — persists. |
| `theme_choice` | The user's chosen theme (`light` / `dark` / `system`). | No — functional/preference. | **No** — persists. |
| `consent_decision` | The analytics-consent decision: `{ analytics: boolean, decidedAt: number, version: 1 }`. | No — a functional consent record (a boolean + timestamp, no identity). | **No** — persists. |
| `card-size-preferences` | Product-card size preference per portal scope (UI density choice). | No — functional/preference. | **No** — persists. |
| `dismissed_optional_update_version_data` | Soft-update "don't re-nag" memory: `{ version, lastChecked }` (which app version's update prompt was dismissed and when). | No — functional. | **No** — persists. |
| `base_site_catalog_checksum` | Freshness checksum for the cached catalog (cache-invalidation bookkeeping). | No — functional. | **No** — persists. |
| `translations_checksum_<NS>_<lang>` | Per-namespace, per-language checksum of cached translations (one key per namespace×language). | No — functional. | **No** — persists. |
| `translations_<NS>_<lang>` | Cached translation strings payload (one key per namespace×language). | No — functional (cached UI text). | **No** — persists. |
| `firestore_user_ensured:<uid>` | A `"1"` flag meaning "this user's Firestore `users/{uid}` doc was already ensured" — lets the app skip a per-login Firestore read. | **Yes (low) — the Firebase UID is embedded in the key name.** Value itself is just `"1"`. | **No** — not cleared on logout; persists per-uid. |
| `SPPL` | Timestamp (ISO string) of the last push-permission prompt — throttles re-prompting. | No — functional/timing. | **No** — persists. |
| `LHNR` | Dedup token for the last handled notification response (prevents double-handling a tapped notification). | No — functional. | **No** — persists. |
| `auth-storage` | Zustand-persisted copy of the logged-in user object (`AuthUserDTO`): `id`, `firebaseUid`, `displayName`, **`email`**, base site, region/city, `profileImageKey`, email/promo/phone-contact preferences, ban/deletion status. | **Yes (high) — name, email, user id, Firebase UID, location.** | **Yes** — `logout()` does `set({ user: null })`, and the persist `partialize` only stores `user`, so the persisted value becomes `{ user: null }`. (The key itself remains, now empty of identity.) |

### B. AsyncStorage — keys written by the Firebase Auth SDK (not by app code)

| Key (SDK-managed) | Holds | Personal/identifying? | Cleared on logout? |
| --- | --- | --- | --- |
| `firebase:authUser:<apiKey>:<appName>` (and related Firebase Auth entries) | The authenticated Firebase session: ID token, refresh token, uid, email, displayName, provider info. Written because `initializeAuth(app, { persistence: getReactNativePersistence(AsyncStorage) })` in `firebaseClient.ts:48` tells Firebase to persist the session to AsyncStorage. | **Yes (high) — auth tokens + email + uid.** | **Yes** — `logoutFirebase()` → `auth.signOut()` clears the Firebase-persisted auth user. Google sign-out (`GoogleSignin.signOut()`) is also called. |

### C. File system

| Location | Holds | Personal/identifying? | Cleared on logout? |
| --- | --- | --- | --- |
| `<cache>/user-profile-images/` (OS cache dir, via `expo-file-system` in `authService.ts:38`) | A downloaded copy of the user's Google profile photo, staged so it can be re-uploaded to R2 during account setup. | **Yes (low) — the user's own profile photo**, but transient and in the OS-reclaimable cache dir. | **No** explicit app deletion; left to OS cache eviction. Not tied to logout. |
| expo-image disk cache (`cachePolicy="memory-disk"` on ~10 `<Image>` sites) | Cached image bytes — product images, base-site images, message images, and **avatars** (`OglasinoAvatar`). | Mostly public catalog imagery; avatars are profile photos (own + other users'). Standard image disk cache, no identity metadata. | **No** — managed by expo-image / OS, not cleared on logout. |

### D. Firebase Analytics on-device storage

| Mechanism | Holds | Personal/identifying? | Cleared on logout? |
| --- | --- | --- | --- |
| `@react-native-firebase/analytics` local state | The Firebase **app-instance ID** (a pseudonymous device-scoped identifier) and queued/unsent analytics events, stored locally by the analytics SDK. Collection is **gated behind `consent_decision.analytics`** (consent mode). | **Yes (low) — pseudonymous app-instance ID** (not name/email). | **No** — device-scoped, not tied to logout. Disabled when consent is denied. |

### E. NOT persisted to disk (in-memory only) — listed so the policy does not over-claim

These Zustand stores hold user data only in RAM and are wiped on logout via `clearUserScopedStores()` (`authStore.ts:41`), and disappear on app kill. They are **not** written to AsyncStorage or disk:

- user cache (`userCache.ts`), favorites, chat list / active chat / chat-nav stores, portal-filter & dashboard-filter stores, view tokens, upload-progress, notification store.

### F. Mechanisms checked and confirmed absent

- **`expo-secure-store`** — the dependency is installed but **not used anywhere** in `src/` or `app/`. Nothing is stored in the device secure store / keychain.
- **Firestore offline disk persistence** — the app uses the Firebase JS SDK's `getFirestore(app)`, which on React Native uses an in-memory cache by default (no IndexedDB/disk). No Firestore documents are persisted to disk.
- **MMKV / SQLite / other KV stores** — none present.
- **Push token** — fetched fresh from Expo each registration and sent to the backend; **not** stored locally.

---

## Files touched

- None (read-only audit).

## Tests

- None run (no code changed).

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change (one low-severity observation flagged below for Mastermind triage, not authored into issues.md)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code written.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 10/Phase 2 (read-only audit posture) — confirmed; Part 11 (trust boundaries) — N/A, no trust decision touched.

## Known gaps / TODOs

- I did not enumerate the exact internal key names the Firebase Auth SDK writes (they are SDK-versioned, e.g. `firebase:authUser:<apiKey>:[DEFAULT]`); I described the category and what it holds rather than asserting an exact string, since the app does not reference them by name.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — no code written.
  - Considered and rejected: nothing — no code written.
  - Simplified or removed: nothing — no code written.

- **Confirmation against the brief's partial list:** the partial list (auth, base-site, language, theme, consent decision, translation cache, push timing) is all confirmed present. Mapping: auth → `auth-storage` + Firebase SDK keys; base-site → `base_site`; language → `app_language`; theme → `theme_choice`; consent → `consent_decision`; translation cache → `translations_<NS>_<lang>` + the two checksum keys; push timing → `SPPL` (+ `LHNR`).
  **Additions the brief's list was missing:** `card-size-preferences`, `dismissed_optional_update_version_data`, `firestore_user_ensured:<uid>`, the `<cache>/user-profile-images/` file cache, the expo-image disk cache, and the Firebase Analytics on-device state.

- **Privacy-relevant highlights for the policy author:**
  1. The two high-sensitivity stores (`auth-storage`, Firebase Auth SDK keys) hold name/email/uid/tokens and **are** cleared on logout.
  2. `firestore_user_ensured:<uid>` embeds the Firebase UID in the key and is **not** cleared on logout — a low-sensitivity residual identifier that survives sign-out. (Functionally harmless: it only gates a Firestore read.)
  3. The cached Google profile photo lives in the OS cache dir and is not explicitly deleted on logout.
  4. Firebase Analytics keeps a pseudonymous app-instance ID on-device; collection is consent-gated.

- **Part 4b adjacent observation (low severity, out of scope — not fixed):**
  - `firestore_user_ensured:<uid>` (`src/lib/storage/firestoreEnsuredStorage.ts`) and the `<cache>/user-profile-images/` directory (`src/lib/services/authService.ts:38`) are not included in `clearUserScopedStores()` on logout. This is almost certainly intentional (the ensured-flag is a cross-session optimization; the cache is OS-managed), and neither is high-sensitivity. Flagging only so the privacy policy describes them accurately as "may persist across logout." I did not change this because it is out of scope for a read-only audit.

- Nothing else flagged.
