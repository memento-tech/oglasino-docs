# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-28
**Task:** Read-only audit of oglasino-expo. Document the CURRENT cold-start boot implementation exactly as it exists, plus what's missing for a versioning-aware boot. No code changes. Output to .agent/audit-expo-boot-redesign.md.

## Implemented

- Read-only audit only. No source files touched.
- Authored `.agent/audit-expo-boot-redesign.md` covering all five parts of the brief: current boot flow end-to-end, storage inventory, version-endpoint consumption (none), update-check classification (`AppVersionConfigInit`), and the four known fragilities.
- Confirmed against code: `apiStore.ts` `globalThis`-pinned barrier; `authStore`↔chat-stores require-cycle triangle; always-mounted `<Stack>` + `<RequireBaseSite>` per-screen render gate (5 placements); live `[BOOT]` `console.log` in `api.ts:44-51` request interceptor.
- Surfaced seven adjacent observations (Part 4b) routed to the audit's bottom section for Mastermind triage.

## Files touched

- `.agent/audit-expo-boot-redesign.md` (new, +543 lines)
- `.agent/2026-05-28-oglasino-expo-boot-redesign-1.md` (new, this file)
- `.agent/last-session.md` (overwritten with a copy of this file)

No source files modified.

## Tests

- Not run. Read-only audit; no code changes.
- Lint / `tsc --noEmit` / `expo-doctor` not applicable per the same reason.

## Cleanup performed

- None needed. No code touched.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (the Φ3 cold-start boot-loading row in Risk Watch and the Catalog versioning backlog row remain accurate; this audit is the read-only Phase-2 deliverable for the resolving Mastermind chat and does not itself trigger config edits)
- issues.md: no change (the live `[BOOT]` instrumentation entry already exists; this audit confirms the location at `src/lib/config/api.ts:44-51`)

## Obsoleted by this session

- Nothing. Audit-only session.

## Conventions check

- Part 4 (cleanliness): N/A — no code changes.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): seven items flagged in the audit's "Adjacent observations" section and listed below in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 10 (feature lifecycle) — this audit is a Phase-2 read-only audit per the lifecycle definition; output landed at `.agent/audit-<feature-slug>.md` as that phase requires.

## Known gaps / TODOs

- None. The audit answers every numbered question in the brief. Two reads the Φ3 handoff explicitly named as "planned at chat close but not executed" were both completed in this session:
  - service-call comparison across the four bootstrap endpoints (Part 1.4),
  - response-interceptor inspection at `api.ts:60-145` (covered indirectly while documenting the always-mounted `<Stack>` and the live `[BOOT]` instrumentation — full response-interceptor body quoted in the audit's `api.ts:67-149` reference for the resolving chat).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: I considered duplicating the brief's text into the audit's section headings verbatim. Rejected — it reads as filler and the headings already match the brief's part numbers. The audit's structure mirrors the brief's parts 1–5 with sub-sections; cross-referencing the brief is enough.
  - Simplified or removed: nothing — read-only audit.

- **Adjacent observations (Part 4b), flagged in the audit's bottom section, listed here for triage:**
  1. `getAppConfiguration` returns `null` on non-200 / catch but is typed `Promise<ConfigMap>`. Low. `src/lib/services/configurationService.tsx:14, 17`. Not in scope.
  2. `initI18n(lang, tenant)` and `loadTranslations(lang, tenant)` accept `tenant` but ignore it. Cache helpers `getTranslations`/`setTranslations` (currently unused) key on `lang` alone — would collide across base sites if caching is re-enabled. Medium when caching is wired. `src/i18n/i18n.ts`, `src/i18n/loader.ts`, `src/i18n/storage.ts`. Not in scope.
  3. `AppVersionConfigInit` re-fires `checkVersion` on every pathname change. `checkingRef` guards overlap but does not debounce. Low. `src/components/internals/AppVersionConfigInit.tsx:114-116`. Not in scope.
  4. App-version update buttons open the placeholder URL `https://memento-tech.com`, not a store deeplink. Medium / pre-launch blocker but separate from boot redesign. `src/components/internals/AppVersionConfigInit.tsx:39-41`. Not in scope.
  5. `AppContext.bootstrap` re-writes `base_site` to AsyncStorage on every cold start (no-op overwrite via line 129). Low. `src/components/context/AppContext.tsx:109, 129`. Not in scope.
  6. Outer try/catch around `getStoredLanguage` / `setStoredLanguage` / `setCodes` / `setStoredBaseSite` / `initI18n` translates any failure in those calls into `status: 'maintenance'`. A namespace fetch hang would mislead the user into a "maintenance" overlay. Medium. `src/components/context/AppContext.tsx:89-148`. Not in scope — but the boot redesign will likely touch this codepath, so worth resolving in the same chat.
  7. `fetchBaseSites` failure returns `[]`, leaving `BaseSiteSelector` empty if the network is down at cold start. Medium UX. `src/lib/init/baseSitesService.ts:19-32`. Not in scope.

- **Questions / risks / suggested next steps for the resolving Mastermind chat:**
  - The brief's framing that `[BOOT] /public/translations?namespace=INTRO` appears as a boot-time signal is technically correct, but the translation burst is a **post-`setCodes` consequence**, not a parallel-with-`Promise.all` event. Translations fire from `initI18n` at `AppContext.tsx:130`, after `setCodes` at line 127. They are NOT among the four-parallel boot burst that hangs in the Φ3 issue; they only fire on the happy path once `bootstrap` proceeds past line 127. The four-parallel burst is `maintenance + config + baseSites + appVersion` (1–3 inside `Promise.all`, 4 from `AppVersionConfigInit`).
  - The redesign's "versioning-aware boot" will probably want a new GET `/public/versions` call inserted **before** the existing `Promise.all`, with the result driving conditional refetch of `config` / `baseSite/details` / `translations`. Storage on the client today (Part 2) is the full `BaseSiteDTO` blob plus the full `LanguageDTO` — there is no version field co-stored with either, so adopting `/public/versions` will require new AsyncStorage keys for the per-namespace and per-base-site version snapshots.

- **Config-file impact: no change.** Audit-only session.

- (nothing else flagged)
