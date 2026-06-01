# Audit — version-checksums-per-language — oglasino-expo

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-29
**Mode:** read-only (Phase 2 audit). No code changed. Findings are code facts; each is marked **[read]** (directly read from the file at the cited line) or **[inferred]**.

Central question (per the brief): is the boot-redesign claim — that the per-(namespace, language) `/versions` upgrade is a contained client-side swap isolated into a small number of helpers — true, and what exactly changes? **Short answer: the claim is *substantially* true but *not complete*. The staleness rule, the key shape, and the response type are genuinely isolated into three helpers, but the gate body inlines one direct per-namespace-checksum-as-string read (`bootStore.ts:408`, feeding `:413` and `:477`) that the three-point isolation contract does not enumerate. The swap is "3 helpers + 1 gate-body line," not "3 helpers." Detail in §5.**

---

## 1. The freshness module

The freshness logic is split across four files, all under `src/lib/store/` and `src/lib/init/`:

- **`src/lib/store/bootFreshness.ts`** — the three isolated helpers (the swap surface). **[read]**
- **`src/lib/store/checksumStorage.ts`** — AsyncStorage read/write helpers for checksums and per-NS payloads. **[read]**
- **`src/lib/init/versionsService.ts`** — `fetchVersions()`, the `GET /api/public/versions` client. **[read]**
- **`src/lib/store/bootStore.ts`** — the boot state machine; the freshness gate body lives in the `runFreshnessGate` action. **[read]**

**The gate this lives in:** **Gate 4 (freshness)** of the `bootStore` gate state machine — `runFreshnessGate` at `bootStore.ts:346-522`. The machine is `booting → Gate 1 maintenance → Gate 2 version → Gate 3 base-site → Gate 4 freshness → ready` (`BootStatus` union at `bootStore.ts:70-76`; gate chain at `:120, :164, :205, :234, :286, :307`). **[read]**

Gate 4's overall shape (`runFreshnessGate`, `bootStore.ts:346-522`): **[read]**

- **Resolve active state** (`:350-358`): reads `selectedBaseSite` + `language` from the store (Gate 3 populated them); `activeLang = language.code` (`:358`). Null here → `toMaintenance` (programming-error guard).
- **Enter `updating`** (`:363`): `toUpdating()` shows the splash while the gate works.
- **One `/versions` call** (`:367-373`): `fetchVersions()` wrapped in `withGateTimeout`; timeout/throw → `toMaintenance`.
- **Catalog staleness** (`:379-397`): reads local catalog checksum, compares to `versions.catalog[selectedBaseSite.code]`; if stale, refetches the full `BaseSiteDTO`, re-stores DTO + checksum (co-storage), refreshes the in-memory slot.
- **Per-namespace staleness loop** (`:402-479`): iterates `Object.values(TranslationNamespace)` (the **local** 20-entry enum, not the backend keyspace). Per namespace: reads local checksum + cached payload, reads backend checksum, decides staleness, refetches stale slices via `fetchNamespace(ns, activeLang)`, and on success persists payload + checksum together.
- **i18n registration** (`:481-516`): first boot configures the singleton via `i18n.use(ICU).use(initReactI18next).init(...)`; subsequent passes use `addResourceBundle(...)`; then **always** `await i18n.changeLanguage(activeLang)`.
- **Advance boot state** (`:521`): `set({ status: 'ready' })` — the one legitimate place the gate writes `ready`.

The gate reaches the staleness rule, the key shapes, and the response type only through `bootFreshness` / `checksumStorage` / the typed `VersionsResponse` — **with one exception** documented in §5.

---

## 2. The three claimed swap points

### 2a. The `VersionsResponse` type — **[read]** `bootFreshness.ts:32-35`

```ts
export type VersionsResponse = {
  translations: Record<TranslationNamespace, string>;
  catalog: Record<string, string>;
};
```

The per-namespace value of `translations` is a single **`string`**, exactly as the brief expects. `catalog` is keyed by base-site code → string (per-base-site, not language-scoped). Confirmed: the transitional shape is per-NS-collapsed.

### 2b. The checksum-key helper — **[read]** `bootFreshness.ts:47-49`

```ts
export function translationsChecksumKey(ns: TranslationNamespace, lang: string): string {
  return `translations_checksum_${ns}`;
}
```

**Critical answer: YES — it ALREADY takes `lang`** as its second parameter, and **deliberately ignores it today** (documented at `:42-46`). The return string is per-NS only (`translations_checksum_${ns}`). Because `lang` is already in the signature and already passed at every call site, the per-language swap is a one-line body change here (`...${ns}_${lang}`) with **zero call-site churn**. The claim holds for this helper.

### 2c. The staleness helper — **[read]** `bootFreshness.ts:74-97`

```ts
export function isNamespaceStaleForActiveLanguage(
  ns: TranslationNamespace,
  activeLang: string,
  backendChecksums: Record<TranslationNamespace, string>,
  localChecksum: string | null,
  hasCachedPayload: boolean
): boolean
```

- Reads the backend checksum as `backendChecksums[ns]` — a **bare string** indexed by namespace only (`:85`).
- Reads the local checksum from the `localChecksum` parameter (a `string | null`), normalizing `null → ""` (`:95`).
- `activeLang` is **accepted but unused** in the comparison today (`:66-67` comment); the future swap changes the internal read to `backendChecksums[ns]?.[activeLang]` without touching the call site.

Staleness logic (transitional, `:81-96`): stale if (a) no cached payload (`:81-83`, the OR-clause for fresh-language/install), or (b) backend `""` not-yet-computed sentinel (`:91-93`), or (c) `backendChecksum !== local` (`:96`). A **`undefined`** backend checksum returns **not-stale** (`:87-89`) — the helper defers the missing-checksum decision to the gate (DECISION 4). The claim holds for this helper.

---

## 3. Storage keys — **[read]** `checksumStorage.ts`

| Thing | Key shape | Where |
| --- | --- | --- |
| Translation **payload** | `translations_<NS>_<lang>` | `translationsPayloadKey`, `:31-33` |
| Translation **checksum** | `translations_checksum_<NS>` (no lang suffix yet) | via `translationsChecksumKey(ns, lang)`, `:39, :47` |
| Catalog **checksum** | `base_site_catalog_checksum` (not language-scoped) | `CATALOG_CHECKSUM_KEY`, `:29` |

Both expectations from the brief are confirmed: the payload is **already** per-NS-per-lang (`translations_<NS>_<lang>`, shipped in boot-redesign), and the checksum is currently per-NS only (`translations_checksum_<NS>`).

**Co-storage rule** (brief asks: are payload and checksum written in the same operation?): **[read]** They are written **back-to-back in the same critical section of the gate**, not in a single atomic storage call. In the loop the gate calls `setStoredNamespacePayload(ns, activeLang, fresh)` (`bootStore.ts:476`) immediately followed by `setStoredNamespaceChecksum(ns, activeLang, backendChecksum)` (`:477`). Both are awaited sequentially; there is no single transactional write. The module header (`checksumStorage.ts:19-22`) states the invariant as "a checksum is always written in the same logical operation as the payload it describes." Precision: it is the *same logical operation in the gate*, two separate `AsyncStorage.setItem` calls. A crash between `:476` and `:477` would leave a payload without its matching checksum — next boot would normalize the missing local checksum to `""` and refetch (fail-toward-fetch), so the asymmetry is self-healing, not corrupting. **[inferred]** (consequence reasoning).

---

## 4. The language model

- **There is no `LanguageCode` union/enum.** Language is modelled as `LanguageDTO = { code: string; active: boolean }` (`src/lib/types/catalog/LanguageDTO.ts:1-4`). `code` is an **unconstrained `string`**. **[read]**
- **Active-language source the gate uses:** `bootStore.language` (a `LanguageDTO`), set in Gate 3 (`runBaseSiteGate`, `bootStore.ts:228-233`) from the stored language (if still in the site's `allowedLanguages`) else `selectedBaseSite.defaultLanguage`. Gate 4 reads `activeLang = language.code` (`:358`) and uses it verbatim as: the AsyncStorage key segment (`:405-406, :476-477`), the `lang` query param to `fetchNamespace` (`:444`), and the i18n `lng` (`:494, :516`). **[read]**
- **CNR (Montenegrin):** Because mobile has **no enumerated language type**, there is no set of codes to check against — mobile accepts whatever `code` the backend's base-site `allowedLanguages` carries. So the brief's framing ("does the type include CNR?") does not apply: there is **no type to disagree with the backend's sr/en/ru/cnr**. **[read]**
- **CNR aliasing/normalization:** **None anywhere on the mobile translation/boot path.** A repo-wide search for `cnr` / `montenegr` / aliasing in `src` returns zero hits in the boot or translation code (the only `me` matches are an unrelated TLD fragment in `productValidator.ts` and base-site code fixtures in tests). The language `code` flows through **verbatim** — no SR-aliasing before it is used as a storage-key segment or a lookup key. **[read]**

**Seam note (factual, not a design proposal):** test fixtures use language codes `'rs-sr'`, `'en'`, `'ru'` (`bootStore.test.ts:211-213`). Whatever exact string the base-site `allowedLanguages[].code` carries is what mobile will use as the per-language checksum lookup key after the swap (`versions.translations[ns][activeLang]`). The same code string **already** round-trips today as the `lang` query param on `fetchNamespace` (`/public/translations?namespace=…&lang=<code>`, `fetchNamespace.ts:31-33`), so the per-language `/versions` keying only needs to match the convention that the translation fetch already relies on. **[inferred]** (consequence reasoning) — flagged because a backend that keys `/versions` by `cnr` while mobile sends `rs-sr` (or vice versa) would silently mis-match and over- or under-refetch.

---

## 5. The gate body's coupling to the three helpers (the isolation question)

**This is the central finding.** Every place the gate body touches a per-namespace **or** catalog checksum *value* directly:

| # | Location | What it touches | In per-language swap scope? |
| --- | --- | --- | --- |
| A | `bootStore.ts:380` | `versions.catalog[selectedBaseSite.code]` — catalog checksum, read directly | **No** — catalog is per-base-site, not language-scoped (now or after the swap, per `bootFreshness.ts:30-31`). |
| B | `bootStore.ts:381-384` | catalog staleness comparison, **inlined in the gate** (no helper) | **No** — same reason as A. |
| C | `bootStore.ts:408` | `const backendChecksum: string \| undefined = versions.translations[ns]` — **direct read of the per-NS checksum as a bare string, outside the staleness helper** | **YES — this is the leak.** |
| D | `bootStore.ts:413` | `if (backendChecksum === undefined)` — DECISION 4 missing-checksum gate decision, consumes the value read at C | **YES** — depends on C's value. |
| E | `bootStore.ts:431` | passes the whole `versions.translations` map into `isNamespaceStaleForActiveLanguage` | No — value handled inside the helper. |
| F | `bootStore.ts:477` | `setStoredNamespaceChecksum(ns, activeLang, backendChecksum)` — persists the value read at C | **YES** — persists C's value. |

**Verdict: isolation is REAL for the staleness *rule*, the key *shape*, and the response *type*, but NOT complete.** The gate body reads `versions.translations[ns]` itself at `:408`, treats it as a `string`, and reuses that value for the missing-checksum gate decision (`:413`) and for what it persists (`:477`). The three-point isolation contract documented in `bootFreshness.ts:8-20` (1: response type, 2: key helper, 3: staleness helper internals) **does not enumerate this fourth touch point.**

Concretely, after the type swap makes `versions.translations[ns]` a per-language map:

- `:408`'s annotation `string | undefined` becomes wrong; the value is now a map object. The line must change to `versions.translations[ns]?.[activeLang]` to recover the per-language checksum string.
- `:413`'s `=== undefined` check changes meaning: it would test "namespace entirely absent" rather than "no checksum for the active language." Without the `:408` fix, a namespace present for *some* languages but not `activeLang` would pass the undefined check and fall through.
- **`:477` is the sharpest consequence:** without the `:408` fix it would persist the **entire per-language map object** as the stored checksum (`setStoredNamespaceChecksum` takes a `string`), corrupting the stored checksum so every subsequent boot mis-compares.

**Net:** the swap is **3 helpers + 1 gate-body region** (`bootStore.ts:408`, which transitively fixes `:413` and `:477`). The isolation claim undersells by one line. It is still a small, single-file, well-contained change — the brief's "cheap" scoping survives — but a plan that touches *only* the three named helpers and leaves `:408` alone would ship a latent corruption-on-persist bug. Name the leak: **`bootStore.ts:408`** (with downstream `:413`, `:477`).

(Asymmetry worth noting for completeness, not a swap blocker: catalog staleness (A/B) is inlined directly in the gate with no helper, whereas namespace staleness is extracted into `isNamespaceStaleForActiveLanguage`. Catalog is out of the per-language swap's scope, so this asymmetry does not affect this feature — but it means "freshness is fully helper-isolated" is not literally true even setting aside the §5 leak.)

---

## 6. Test coverage

**Framework:** Vitest (`import { describe, expect, it } from 'vitest'`). **[read]**

**How `/versions` is mocked:** `vi.mock('../init/versionsService', () => ({ fetchVersions: mockFetchVersions }))` where `mockFetchVersions` is a `vi.hoisted` mock (`bootStore.test.ts:102-104`). The default fixture (`:192-195`) returns `{ translations: {…per-NS strings…}, catalog: {…} }`. `fetchNamespace` is mocked identically (`:106-107`). `checksumStorage` is fully mocked (`:114-126`) so gate tests drive local-checksum/payload state directly. **The real `bootFreshness` staleness helper is intentionally NOT mocked** (`:110-113`) — gate tests exercise real staleness. **[read]**

| Test file | Asserts against | Post-swap expectation |
| --- | --- | --- |
| `bootFreshness.test.ts` — `translationsChecksumKey` block (`:14-32`) | **helper internals** — explicitly asserts the key is per-NS and the **lang arg is ignored** (`:21-30`) | **Must change.** The "same NS, different lang → same key" assertions (`:25-30`) flip to per-language keys. |
| `bootFreshness.test.ts` — `isNamespaceStaleForActiveLanguage` block (`:34-74`) | **helper internals** — 8 cases over a per-NS-string `backendChecksums` map | **Must change.** Fixtures (`backend()` helper, `:38-39`) become per-language maps; new assertions for the `activeLang` dimension. |
| `bootFreshness.test.ts` — `VersionsResponse` | compile-time only (`:10-12`); `tsc --noEmit` is its test | **Implicitly changes** with the type. |
| `bootStore.test.ts` — Gate 4 block (`:944-1199`) | **gate body behavior** — zero-fetch returning user, full first-boot refetch, partial-stale counts, missing-checksum branches, maintenance routing, INVARIANT 3 | **Mostly stay green** on behavior, **but** the `mockFetchVersions` fixtures everywhere build `translations: { NS: 'string' }`; those fixtures must be reshaped to per-language maps, and the §5 `:408`/`:413`/`:477` change will need its own assertions (e.g. "persists the per-language checksum string, not the map"). |

So: the **helper-internal tests** (`bootFreshness.test.ts`) need new/updated assertions; the **gate-body tests** assert behavior that should stay green provided the gate's external contract is preserved, but their `/versions` fixtures must be reshaped to the per-language wire shape, and the §5 leak warrants a dedicated gate-body assertion. **[read]** + **[inferred]** (the "must change" calls are consequence reasoning from the assertions as written).

---

## 7. Trust boundary check (conventions Part 11) — verdict

**Clean.** No value from `/versions` is used in any moderation, authorization, or state-transition decision. The checksum values drive only refetch-or-not and (in pathological cases) maintenance routing:

- A malformed/hostile `/versions` can cause at worst **over-refetch** (bogus checksums → fetch every namespace), **under-refetch** (matching-but-wrong checksums → serve cached strings), or **route to maintenance** (missing checksum + no cached payload, `bootStore.ts:421-425`). The gate fails toward fetch / toward maintenance, never toward a privileged action.
- The translation *payload* fetched as a consequence is registered into i18n as display strings — a hostile backend could inject arbitrary UI text, but that is the **identical trust posture as the existing `fetchNamespace` path** and is not worsened by `/versions`; `/versions` only decides *whether* to call `fetchNamespace`, which already trusted the backend for string content.
- Catalog staleness (`:379-397`) likewise only triggers a `fetchBaseSiteByCode` refetch; the refetched `BaseSiteDTO` is the authoritative source either way.

No concern to name. The per-language upgrade does not introduce a new trust surface — it changes the *shape* of an already-trusted, non-security-bearing value.

---

## Inventory of factual vs inferred

- **[read] / factual:** all file paths, function names, signatures, line numbers, the type shapes (§2), key shapes (§3), the absence of a `LanguageCode` type and of CNR aliasing (§4), the six touch points in §5, the test structure and mocking (§6).
- **[inferred] / consequence reasoning:** the co-storage crash-window self-heal (§3); the seam risk if backend/mobile language-code strings diverge (§4); the precise post-swap edits required at `bootStore.ts:408/413/477` and the corruption-on-persist consequence at `:477` (§5); the "must change" test calls (§6). These are derived from reading the transitional code against the stated future shape, not from any spec (no feature/future doc was read, per the brief).
