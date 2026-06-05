# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev (unchanged — no commit/push/checkout)
**Date:** 2026-06-04
**Slug / order:** deep-links / 2
**Task:** Implement `app/+native-intent.ts` exporting `redirectSystemPath` — strip a leading routing-locale segment from inbound deep-link paths so web's `https://oglasino.com/{locale}/…` links resolve against the locale-less app route tree. JS-only; no native/config edits.

## Brief vs reality (raised before writing code; resolved by Igor)

1. **Prescribed locale regex matched only 5 of the 10 enumerated locales.**
   - Brief said: detect with `/^[a-z]{2}-[a-z]{2}$/i` ("two letters, hyphen, two letters"), *"do this, don't deviate"* (lines 32, 47), and test that **all 10** locales strip (line 73).
   - Reality: that regex matches only `rs-sr rs-en rs-ru me-sr me-en me-ru`. It **fails** `rsmoto-sr`, `rsmoto-en`, `rsmoto-ru` (6-letter base-site) and `me-cnr` (3-letter language) — verified by running the regex against the brief's own 10-locale list.
   - Why it mattered: shipped as prescribed, inbound universal links for the entire moto vertical + Montenegrin-Cyrillic (5/10) would keep their locale prefix, miss every route, and land on `+not-found`; the required all-10 test could not pass.
   - **Resolution (Igor chose):** broaden to `/^[a-z]{2,}-[a-z]{2,}$/i`. Stays shape-based (no hardcoded list — the brief's stated intent, drift-resistant), covers all 10, and is false-positive-free: no first-segment app route contains a hyphen (`free-zone`, `account-verification`, `not-ready` are always deeper than position 0, which is the only position tested).

## Implemented

- **`src/lib/navigation/redirectSystemPath.ts`** (new) — pure `redirectSystemPath({ path, initial }): string`. Strips a leading locale segment of shape `{2,}-{2,}` (case-insensitive), preserves query/fragment verbatim, discards the stripped locale (no `bootStore` write / language side-effect), passes everything else through unchanged, never throws.
- **`app/+native-intent.ts`** (new) — expo-router entry point; re-exports `redirectSystemPath` from the `src/` module via the `@/` alias. Logic lives under `src/` because vitest's config only scans `src/**` (see Tests).
- **`src/lib/navigation/redirectSystemPath.test.ts`** (new) — 35 cases (see Tests).

### Design choices (Part 4a)

- **String parsing, not `new URL`.** The brief's suggested `new URL(path, base)` shape is labeled "illustrative, not prescriptive." I used plain string operations instead because **React Native 0.81 ships a non-spec-compliant regex-based global `URL`** (verified directly in `node_modules/react-native/Libraries/Blob/URL.js`: hand-rolled getters, `validateBaseUrl` only accepts `https?`/`ftp`, no URL polyfill installed). `new URL` would behave one way in the node test environment and differently on-device. String slicing is deterministic across both, and it preserves the query string **verbatim** (no re-encode/reorder) — which the filter-deep-linking dependency requires. The function: split off `?`/`#` suffix → strip optional `scheme://authority` prefix → test first path segment → rejoin.
- Considered and rejected: `new URL` (RN polyfill divergence, above); a one-file `src/lib/linking/` directory (placed in existing `src/lib/navigation/` instead — no new dir for one file).
- Simplified/removed: nothing — net-new code.

## Files touched

- `app/+native-intent.ts` (new)
- `src/lib/navigation/redirectSystemPath.ts` (new)
- `src/lib/navigation/redirectSystemPath.test.ts` (new)
- `.agent/2026-06-04-oglasino-expo-deep-links-2.md` (this summary)
- `.agent/last-session.md` (exact copy)

No source/config/native files modified. The pre-existing uncommitted diffs (`LoginDialog.tsx`, `utils.ts`, `utils.test.ts`) are not mine — untouched.

## Tests

- New `redirectSystemPath.test.ts`: **35 cases pass** — universal product/user/catalog/static links strip; query string preserved verbatim (`?priceFrom=…&priceTo=…&condition=…`); fragment preserved; all 10 locales parameterized (and case-insensitive); locale-less universal + bare paths unchanged; custom-scheme (`oglasino://`) and Expo Go (`exp://…/--/`) pass through; bare locale-prefixed path strips; locale-only path → `/`; non-first locale-shaped segment NOT stripped; hyphenated route segment not treated as locale; empty/`/`/malformed safe, no throw.
- **Full suite: 40 files / 457 tests pass.** `npx tsc --noEmit` exit 0. `npx eslint` on the three new files: clean (0 warnings/errors) — lint baseline (88 warnings / 1 pre-existing `DashboardSidebar` error) held; I added neither.

## Cleanup performed

- none needed (net-new code; no debug logging, no dead code, no commented-out blocks, no TODO/FIXME).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- issues.md: no change
- state.md: no change written. The deep-linking feature is **not yet shipped** by this session — `+native-intent` (the keystone JS hook) is in place and tested, but the feature still depends on native config (`app.config.ts` associatedDomains/intentFilters), web-hosted `.well-known` files, and the release SHA-256 (all cross-repo / out of scope). So **no Expo-backlog row should be removed**. If Docs/QA tracks a deep-linking row, the keystone-hook line item can be marked done; the feature row stays open pending the native + cross-repo blockers. (Closure gate: no implicit config-file dependency from this session beyond this note.)

## Obsoleted by this session

- Nothing.

## Conventions check

- **Part 4 (cleanliness):** clean — no console/debug logging, no dead code, no commented-out blocks, no unused imports, no TODO/FIXME. tsc + lint + test green for touched paths.
- **Part 4a (simplicity):** see "Design choices" above — earned complexity (string parser) justified by the RN `URL` divergence + verbatim-query requirement; rejected `new URL` and a new directory.
- **Part 4b (adjacent observations):** none new. (The pre-existing prod-hardcoded share host in `getNormalizedProductUrl`, already in issues.md, is adjacent to a future universal-link rollout but out of scope here.)
- **Part 6 (translations):** N/A — no user-facing strings; the hook is pre-routing path rewriting.
- **Hard rules:** no commit/push/checkout; no `app.config.ts`/`eas.json`/native edits; no builds/`eas`/prebuild; no cross-repo edits. expo-router support and the RN `URL` behavior were verified directly against `node_modules`, not assumed.

## Known gaps / TODOs

- Universal-link end-to-end cannot be exercised yet: needs production-profile native config (`app.config.ts` — forbidden without an explicit brief), web-hosted `apple-app-site-association` + `assetlinks.json`, and the release SHA-256 from EAS. On-device behavior of the hook under RN's `URL`/Linking is a Ψ item (the string-based impl sidesteps the `URL` polyfill, but the inbound URL shape `Linking.getLinkingURL()` actually hands the hook on iOS vs Android is only confirmable at runtime — see For Mastermind).

## For Mastermind

- **expo-router signature matched (quoted, `expo-router@6.0.24`, `node_modules/expo-router/build/types.d.ts:41-44`):**
  `redirectSystemPath?: (event: { path: string; initial: boolean }) => Promise<string> | string;`
  I export a named (not default) `redirectSystemPath` returning a synchronous `string` (a valid subtype). expo-router reads `nativeLinking?.redirectSystemPath` off the module namespace (`getLinkingConfig.js`), invoked in `getInitialURL` (`initial: true`) and in `subscribe`'s URL listener (`initial: false`) — confirmed in the installed build.
- **Full URL vs bare path — both, and it's runtime-determined.** From the installed source, `path` is whatever `getInitialURL()` / `Linking.addEventListener('url')` yields, i.e. a **full URL with scheme** on device: `https://oglasino.com/{locale}/…` (universal links), `oglasino://…` (custom scheme), `exp://…/--/…` (Expo Go). The hook also accepts bare paths. The impl normalizes both via string parsing and returns a **bare path** only when it strips a locale (always a web URL); otherwise it returns the input untouched. **Ψ check:** the exact string `Linking.getLinkingURL()` (iOS) / the Android `getInitialURLWithTimeout` path hands the hook for a verified universal link can only be confirmed on-device once native config + `.well-known` land — flag as a runtime verification item.
- **Route-tree segments of locale shape:** none at first position. First-segment routes: `product user catalog about pricing privacy terms blog favorites messages notifications owner` + index — zero hyphens. Hyphenated segments (`free-zone`, `account-verification`, `not-ready`) are always deeper; the hook only tests position 0. No counterexample found.
- **Forward-compatible with filter deep-linking:** confirmed — no query param is special-cased or stripped. The query string (and fragment) is carried as a verbatim substring slice of the original `path`, so `?priceFrom=…&priceTo=…&condition=…` survives byte-for-byte (no re-encode/reorder). Explicit test asserts this.
- **Regex widening decision (Igor, this session):** `/^[a-z]{2,}-[a-z]{2,}$/i` replaces the brief's `/^[a-z]{2}-[a-z]{2}$/i` so the impl covers `rsmoto-*` and `me-cnr`. Still shape-based, no hardcoded list. If a future first-position route ever introduces a hyphenated `{2,}-{2,}` segment, this would false-positive — none exists today; worth a note if the route tree grows one.
- **Part 4a simplicity evidence:** Added (earned) — a ~20-line string-based path parser, justified by RN's non-spec-compliant global `URL` + the verbatim-query requirement. Considered/rejected — `new URL` (polyfill divergence), a new `src/lib/linking/` dir (reused `navigation/`). Simplified/removed — nothing (net-new).
- **Cross-repo blockers (unchanged from audit-deep-links-2):** web must host `apple-app-site-association` + `assetlinks.json`; Android needs the release SHA-256 (EAS/Igor); native config edit (`app.config.ts`) needs an explicit brief. The keystone JS hook no longer blocks them — it's done and tested.
