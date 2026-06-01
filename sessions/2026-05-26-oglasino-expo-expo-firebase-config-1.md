# Session summary

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-05-26
**Task:** Read-only audit of every Firebase-touching configuration surface in `oglasino-expo`. Inventory current state, compare against known-good Firebase project IDs, produce a verdict on staleness and environment separation.

## Implemented

- Inventoried all 16 Firebase configuration surfaces (5 Expo config files, 6 native config files/locations, 2 environment files, 3 JS/TS initialization sites).
- Compared every Firebase project ID, app ID, sender ID, API key, and storage bucket against the two known-good projects (`oglasino-stage-49abb` and `oglasino-prod-7e5db`).
- Discovered that **every Firebase value in the codebase points to `oglasino-dev`** — a third project (project number `507102976889`) that is neither stage nor prod.
- Identified 9 stale values (all high severity), 6 missing surfaces, and 5 adjacent observations.
- Assessed Φ1's device smoke as **invalid** unless the local backend also targets `oglasino-dev` — cross-project token rejection would cause 401s on every `syncUserToBackend` call.
- Full audit written to `.agent/audit-expo-firebase-config.md`.

## Files touched

- `.agent/audit-expo-firebase-config.md` (new, +~250 lines) — the audit output
- `.agent/2026-05-26-oglasino-expo-expo-firebase-config-1.md` (new) — this session summary
- `.agent/last-session.md` (overwrite) — copy of this session summary

No source code, config files, or `.env*` files were modified.

## Tests

- N/A — read-only audit, no code changes, no tests to run.

## Cleanup performed

- None needed (read-only audit).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: drafted Risk Watch entry below in "For Mastermind" — Φ1's device smoke may be testing against the wrong Firebase project
- issues.md: drafted new entry below in "For Mastermind" — `oglasino-expo` Firebase config points to third project `oglasino-dev`

## Obsoleted by this session

- Nothing. Read-only audit; nothing created or deleted.

## Conventions check

- Part 4 (cleanliness): N/A — no code changes.
- Part 4a (simplicity): see structured evidence in "For Mastermind."
- Part 4b (adjacent observations): five observations flagged in the audit document (A1–A5), summarized in "For Mastermind."
- Part 10 (feature lifecycle): this is a Phase 2 audit per conventions Part 10. Output feeds Phase 3 (seam analysis) at the next Mastermind session.
- Part 11 (trust boundaries): no Firebase Admin SDK found in mobile `package.json` — clean. No client-side reference to Firebase Admin credentials. The JS SDK `firebase` package is the correct client-side SDK.

## Known gaps / TODOs

- The audit cannot confirm whether `oglasino-dev` is a project Igor intentionally uses for isolated mobile development, or whether it's a leftover from an earlier phase. Igor's confirmation is needed.
- The audit cannot confirm whether `oglasino-stage-49abb` and `oglasino-prod-7e5db` have iOS/Android apps registered for `com.oglasino.dev` / `com.oglasino`. This determines whether native config files can be downloaded immediately or require app registration first.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code introduced.
  - Considered and rejected: nothing — no design decisions made.
  - Simplified or removed: nothing — no code changes.

- **HIGH SEVERITY — `oglasino-expo` talks to `oglasino-dev`, not stage or prod.**
  Every Firebase configuration surface in the repo — native config files, `.env.development`, and JS initialization code — points to Firebase project `oglasino-dev` (project number `507102976889`). This is a third project that is not `oglasino-stage-49abb` (stage) and not `oglasino-prod-7e5db` (prod). Mobile is silently operating in a completely separate Firebase universe from web and backend. Firebase Auth tokens, Firestore data (chats, notifications, users), FCM push tokens, and Analytics streams are all isolated to this third project.

- **HIGH SEVERITY — Φ1's device smoke may be invalid.**
  If Igor's local backend is configured for `oglasino-stage-49abb`, Firebase ID tokens from `oglasino-dev` will be rejected → 401 on every `syncUserToBackend` call. If the local backend is also on `oglasino-dev`, the smoke tests code paths but not cross-platform integration. Either way, Φ1's smoke needs to be re-run after the config fix.

- **Production builds are broken.** `.env.production` has all Firebase values empty. `google-services.prod.json` and `GoogleService-Info.prod.plist` don't exist but are referenced by `app.config.ts`. A production EAS build will fail.

- **Recommended fix scope: single brief.** All stale values, missing files, and env gaps are tightly coupled. The fix requires Igor to register iOS/Android apps in the stage and prod Firebase projects before the agent can place the native config files and update env vars. See audit Section 8 for the full recommendation.

- **Adjacent observations (Part 4b, from audit):**
  - A1: `firebaseAnalytics.ts:8` uses `typeof window !== 'undefined'` (web pattern, not RN) — low severity, out of scope.
  - A2: `firebaseClient.ts:38-52` dead `testConnection` function with `console.info`/`console.error` — low severity, Ω cleanup target.
  - A3: `.env.production:3` uses `http://oglasino.com/api` instead of `https://` — medium severity, iOS ATS would block this.
  - A4: App uses Firebase JS SDK, not `@react-native-firebase/*` — informational, affects analytics SDK story.
  - A5: `.env.development:10` `EXPO_PUBLIC_FIREBASE_APP_ID` is a web app ID (`web:` prefix), not a native app ID — low severity.

### Drafted config-file text

**For `issues.md` (new entry):**

```markdown
## 2026-05-26 — `oglasino-expo` Firebase config points to third project `oglasino-dev`, not stage or prod

**Severity:** high
**Status:** open
**Found in:** `.env.development`, `google-services.dev.json`, `GoogleService-Info.dev.plist`, `src/lib/client/firebaseClient.ts`
**Detail:** Every Firebase configuration surface in `oglasino-expo` targets Firebase project `oglasino-dev` (project number `507102976889`). The known-good projects are `oglasino-stage-49abb` (stage) and `oglasino-prod-7e5db` (prod). Mobile operates in a separate Firebase universe — Auth tokens, Firestore data, FCM tokens, and Analytics streams are all isolated. Additionally, `.env.production` has all Firebase values empty, and the production native config files (`google-services.prod.json`, `GoogleService-Info.prod.plist`) don't exist — production builds would fail. Fix requires: (1) Igor registers iOS/Android apps in stage and prod Firebase projects, (2) agent updates env files and places native config files, (3) Φ1 device smoke re-runs against stage. Full audit at `oglasino-expo/.agent/audit-expo-firebase-config.md`.
```

**For `state.md` Risk Watch (new row, if a Risk Watch section exists, or "For Mastermind" note if it doesn't):**

```markdown
- **Expo Firebase config mismatch (2026-05-26):** `oglasino-expo` targets `oglasino-dev`, not `oglasino-stage-49abb` or `oglasino-prod-7e5db`. Φ1's device smoke may be testing against the wrong Firebase project. Fix brief needed before mobile can be smoke-tested against stage or prod data. Audit at `oglasino-expo/.agent/audit-expo-firebase-config.md`.
```
