# Session summary

**Repo:** oglasino-expo
**Branch:** `new-expo-dev` (not changed; no commit/push)
**Date:** 2026-06-04
**Task:** Deep-link base-site switch + two-pass language resolve (boot Gate 3): `+native-intent` parses + stashes the compound locale via a read-and-clear side-channel; Gate 3 validates strictly, switches before the portal mounts, and resolves filters in the link's language while never persisting over the user's preferred language.

## Implemented

- **Side-channel from `redirectSystemPath`.** When it strips a leading compound-locale segment it now parses it (first-hyphen split, lowercased) and parks `{ baseSite, language, path }` on a module-level stash (`pendingDeepLink.ts`). `path` is the locale-less route **with its query string** — Gate 3 needs the query to re-resolve filters in the link language before the portal mounts. `redirectSystemPath` stays a pure leaf (imports only `parseCompoundLocale` + the stash; no store, no async) so its isolated unit test holds. The returned route is unchanged.
- **Strict locale validation (pure, unit-tested).** `deepLinkLocale.ts`: `parseCompoundLocale` (first-hyphen split, handles `rsmoto-sr`/`me-cnr`) and `isLocaleValid(baseSite, language, overviews)` — real iff the base-site exists in the available set AND the language is in **that** site's `allowedLanguages` (so the `rs-cnr` near-miss is rejected).
- **Gate 3 integration (Design A, in-place fold).** `runBaseSiteGate` reads the stash up front (read-and-clear). On the stored-site path, for a **differing** base-site it fetches overviews (differs-only), validates, fetches the switched DTO, sets the switched slots + the **preferred** language (current-if-allowed-else-default, same rule as `pickBaseSite`), persists the switched site + preferred language (**never the link language**), then resolves + releases. For a **same-site different-language link carrying filters** it resolves in the link language with no fetch. Same-site/same-language or no-filter links fall through to the unchanged path. Invalid locale / overviews-fetch failure / switched-DTO-fetch failure all **fall through to a normal boot on the stored site** — never maintenance.
- **Link-language resolve without a persisted language switch.** Instead of switching the active display language to the link's and reverting (the brief's literal two-pass), the deep-link path keeps the display in the user's preferred language throughout and resolves the link's filters with a **fixed link-language translator** (`getFixedT(linkLang, COMMON_SYSTEM)`) over a transient in-memory label load (`ensureLanguageLabelsLoaded`). The resolved objects are language-independent (DTO/option/region references), so they survive with the display in the preferred language. This realizes the brief's intent (resolve in link language; preferred language never persisted-over; no flash) with strictly less machinery and no revert window. See "For Mastermind".
- **Single feed fetch, no flash.** The whole resolve runs while boot status stays `'booting'` (the portal Stack mounts only on `'ready'`/`'updating'`), so the mount-once hook + feed load cannot run mid-resolve. Status flips to `'ready'` only after the preferred language is active and the resolved filters are parked. The hook (`useDeepLinkFilterHydration`) consumes the parked, already-resolved filters when present and otherwise parses the query as before — so the portal mounts once, switched, in the preferred language, filters applied.
- **Freshness gate refactor (no behaviour change).** Extracted the freshness work (versions → catalog-staleness refetch → stale-namespace refetch → i18n register/changeLanguage, everything up to but not including the `'ready'` release) into `loadAndRegisterActiveLanguage()`, reused by Gate 4 (`runFreshnessGate` = null-check + `toUpdating` + helper + `'ready'`) and by the deep-link path (which releases only after resolving filters). Failure routing (→ maintenance) is preserved verbatim.

## Files touched

- `src/lib/navigation/deepLinkLocale.ts` (NEW, 56) — `parseCompoundLocale` + `isLocaleValid`
- `src/lib/navigation/pendingDeepLink.ts` (NEW, 56) — inbound + resolved-filter read-and-clear stashes
- `src/lib/navigation/resolveDeepLinkFilters.ts` (NEW, 98) — query parse + path-scoped `availableFilters` + `hasFilterParams`
- `src/lib/navigation/redirectSystemPath.ts` (modified — parse + stash on strip; untracked on this branch)
- `src/lib/hooks/useDeepLinkFilterHydration.ts` (modified — consume parked resolved filters; untracked on this branch)
- `src/lib/store/bootStore.ts` (+169 / −13) — Gate 3 deep-link branch, `freshnessThenResolve`, `ensureLanguageLabelsLoaded`, `loadAndRegisterActiveLanguage` extraction
- `src/lib/navigation/deepLinkLocale.test.ts` (NEW, 101)
- `src/lib/navigation/pendingDeepLink.test.ts` (NEW, 50)
- `src/lib/navigation/resolveDeepLinkFilters.test.ts` (NEW, 117)
- `src/lib/navigation/redirectSystemPath.test.ts` (modified — added a side-channel `describe` block)

## Tests

- Ran: `npx vitest run` → **45 files, 512 passed, 0 failed**.
- Ran: `npx tsc --noEmit` → clean.
- Ran: `npx eslint` on all touched files → **0 errors, 0 new warnings** (2 pre-existing `import/no-named-as-default-member` warnings on `i18n.use`/`i18n.changeLanguage` remain — original freshness code I relocated but did not author; my new `getFixedT` use was switched to the named import to avoid adding a third).
- New tests: `deepLinkLocale.test.ts` (parse first-hyphen/lowercase/null edges; validation accept / unknown-site / `rs-cnr` near-miss / empty set), `resolveDeepLinkFilters.test.ts` (home vs catalog-scoped pool; link-language resolves where preferred-language does not; lenient region skip; fully-unresolvable → undefined; `hasFilterParams`), `pendingDeepLink.test.ts` (read-and-clear both stashes), plus a `redirectSystemPath` side-channel block (parks locale + path; lowercases; no-op on locale-less / custom-scheme).
- `npx expo-doctor`: **not run** — no dependency or native config change.

## Cleanup performed

- None needed. No commented-out code, debug logging, dead code, or unused imports introduced. The freshness extraction left no orphan (the old inline body moved wholesale into the helper).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change — see Closure gate in "For Mastermind"
- issues.md: no change (the one adjacent finding from the audit is already recorded there per the audit session; nothing new)

## Obsoleted by this session

- Nothing. No prior code, test, or doc is superseded. The audit (`-1`) remains the design record; this session implements it. `redirectSystemPath`'s former "the stripped locale is discarded" doc comment was rewritten in place to describe the side-channel — not obsoleted, corrected.

## Conventions check

- Part 4 (cleanliness): confirmed — lint/tsc/test green for touched paths; no debug logging; the transient-language path writes nothing to AsyncStorage and never overwrites the stored-preference language slot.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): N/A — no new out-of-scope issues surfaced beyond the audit's already-recorded feed-staleness finding.
- Part 6 (translations): N/A — no new translation keys; resolution reuses the backend-seeded COMMON_SYSTEM namespace.
- Other parts touched: Part 7 (error contract) — N/A (no new backend calls beyond the existing overviews/by-code primitives; failures degrade to open-normally, not surfaced errors).

## Known gaps / TODOs

- No `TODO`/`FIXME` left in code. The on-device Ψ items below cannot be exercised in the node/vitest harness (no RN renderer, no i18n init, no boot machine) and are flagged for Igor rather than left as code TODOs.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - *Added (earned complexity):*
    - `deepLinkLocale.ts` (parse + validate) — two pure leaf functions, concretely needed by `redirectSystemPath` (parse) and Gate 3 (validate); kept separate from the stash so the navigation module imports no store.
    - `pendingDeepLink.ts` two stashes — the side-channel the audit mandated; module-level (not a `bootStore` write) to keep `redirectSystemPath` pure and unit-testable.
    - `resolveDeepLinkFilters.ts` — bridges the stored path → category → `availableFilters` scoping to the existing `parseFiltersFromQueryParams`; earns its place by being unit-testable in the node harness (the Gate-3 orchestration itself is not).
    - `loadAndRegisterActiveLanguage` extraction — one new store method, justified by two callers (Gate 4 + the deep-link path) needing the freshness work without the `'ready'` release.
    - `ensureLanguageLabelsLoaded` — a thin transient label load; justified below (the brief's flagged unknown).
  - *Considered and rejected:*
    - Carrying the **resolved `HydratedFilters`** computed in Gate 3 via the path — kept (it is the brief's design); but I rejected resolving **in the hook** with a link-language `getFixedT` (would avoid the path-in-stash) because the portal mounts during the boot transient and the hook is mount-once: there is no point after i18n init and before the screen's first render to load link labels, so resolution must finish before `'ready'`, i.e. in the gate.
    - A literal **active-language switch + revert** two-pass — rejected in favour of a fixed link-language translator over a transient label load: it removes the revert step and the flash window entirely and never touches the persisted-language slot.
    - A `withGateTimeout` wrapper on the deep-link overviews/DTO fetches — rejected: an unverifiable deep-link base-site must open normally on the stored site, not escalate to maintenance.
  - *Simplified or removed:* the freshness `'ready'` release now lives in exactly one place per caller; no duplication of the freshness body despite two call sites.
- **Transient non-persisting language (the audit's "one unknown"):** a non-persisting active-language switch did **not** exist (`setLanguage` persists + re-runs Gate 4). I did **not** build one. Instead the deep-link path keeps the active/display language as the user's preferred language and resolves filter slugs with a **fixed translator bound to the link language** (`getFixedT(linkLang, COMMON_SYSTEM)`) over an in-memory-only label load (`ensureLanguageLabelsLoaded`, no AsyncStorage write, no active-language change). Net: the user's preferred language is never set to, nor persisted as, the link language. This is thinner than the anticipated transient `setLanguage` variant and avoids the revert/flash window. Flagging because it diverges from the brief's literal "set active language to the link's, then revert" mechanism while meeting every stated goal.
- **Boot invariants honored:** (1) no new effect — the only new render-time code is the hook's existing `useState` initializer reading a module stash (no machine-owned dep); Gate 3 advances by gate calls only. (2) machine writes `status` — the deep-link path sets `'ready'` exactly once at its terminus and otherwise routes via `toMaintenance`/`runFreshnessGate`; it deliberately stays in `'booting'` (does not call `toUpdating`) so the portal Stack does not mount before filters are resolved. (3) re-entry destroys nothing — the stash is read-and-cleared on first Gate-3 pass, so `reEnter` finds it null and takes the normal stored path against the (now persisted) switched site; no slot is nulled.
- **Ψ — on-device, cannot be verified in node (flag for Igor):**
  1. **Side-channel ordering:** confirm `redirectSystemPath` for the initial URL runs before Gate 3's stored-site read resolves. Cold-start an `me-*` link with a stored `rs` site and confirm the switch happens.
  2. **Single feed fetch** on a switching deep-link cold-start (one product request, against the switched site).
  3. **No transient-language flash:** the loading curtain holds (status stays `'booting'`) until preferred language + filters are ready; the user never sees a link-language render.
  4. **Preferred language intact:** after opening a cross-language link, the persisted `app_language` equals the user's original preference (the link language is never written).
  5. **i18n init ordering:** confirm `ensureLanguageLabelsLoaded`'s `addResourceBundle(linkLang)` survives — it runs **after** `loadAndRegisterActiveLanguage` (hence after first-boot `i18n.init`, which replaces the resource store). Verify a cross-language link's filters actually resolve on device.
- **Config-file impact (Closure gate):** none required. This brief implements an Expo-native deep-link capability; it adopts no web feature, so there is no `state.md` Expo-backlog row to remove and no governed-doc edit outstanding. If Mastermind tracks this feature in a backlog/state block, that edit is Docs/QA's to make — I am not aware of an existing row for it.
