# Version Checksums Per Language

Upgrades translation namespace checksums on `GET /api/public/versions` from **per-namespace-collapsed-across-languages** (current) to **per-(namespace, language)**. The change is primarily backend. The mobile client takes a contained swap. Web is not affected.

This spec supersedes `future/version-checksums-per-language.md`. Where the two disagree, this spec wins. It is reconciled against the Phase 2 audits (`oglasino-backend` and `oglasino-expo`, both 2026-05-29) and the Phase 3 seam resolutions.

## 1. Why

The mobile boot freshness gate (`expo-boot-redesign.md` Part 4, Gate 4) compares per-namespace checksums against a local cache and refetches only stale slices. Today the checksum is computed across all languages for a namespace, so a Russian-only translation edit bumps the namespace's single checksum and forces every user on every language to refetch that namespace — even users whose language did not change. The waste is bounded but real and grows with the number of languages. Per-(namespace, language) checksums eliminate it: a Russian-only change bumps only Russian's checksum for that namespace, and a Serbian user refetches nothing.

## 2. Current contract (what this replaces)

`GET /api/public/versions` returns `VersionsResponseDTO` — a record `(Map<String,String> translations, Map<String,String> catalog)`. `translations` is keyed by namespace name only; the value is a single checksum string. `catalog` is keyed by base-site code.

```json
{
  "translations": { "COMMON": "<16-hex>", "ERRORS": "<16-hex>", "...": "..." },
  "catalog": { "rs": "<16-hex>", "rsmoto": "<16-hex>", "me": "<16-hex>" }
}
```

The translation checksum is computed across all languages for the namespace. `VersionChecksumService.computeTranslationChecksum` gathers every row for a namespace via `translationRepository.findByTranslationNamespace`, sorts by language code then translation key, hashes the `translationKey + "|" + translationValue` lines joined by newline with SHA-256, and takes the first 16 hex characters. The language code is used for ordering but is **not** part of the hashed material. Any key/value change in any language bumps the one checksum.

## 3. New contract

`GET /api/public/versions` returns translation checksums keyed by namespace **and** language. `translations` becomes a nested map: namespace → (language code → checksum).

```json
{
  "translations": {
    "COMMON": { "sr": "<16-hex>", "cnr": "<16-hex>", "en": "<16-hex>", "ru": "<16-hex>" },
    "ERRORS": { "sr": "<16-hex>", "cnr": "<16-hex>", "en": "<16-hex>", "ru": "<16-hex>" },
    "...": { "...": "..." }
  },
  "catalog": { "rs": "<16-hex>", "rsmoto": "<16-hex>", "me": "<16-hex>" }
}
```

`catalog` is unchanged — catalog data is not language-specific.

The DTO shape is a **nested map**, `Map<String, Map<String, String>>` for `translations`. It is not a flat composite key (`"<NS>.<lang>"`). The mobile client codes against the nested shape.

### What changes a checksum

A namespace's checksum for a language is computed from only that language's rows in that namespace. A change to a Russian key/value in `COMMON` bumps `translations.COMMON.ru` and nothing else.

### Languages

Four languages have their own stored translation rows: `sr` (Serbian), `cnr` (Montenegrin), `en` (English), `ru` (Russian). **CNR is a real fourth language with its own rows**, not a runtime alias of SR. The `cnr → SR` collapse that exists in `moderation/SupportedLanguage` is moderation-only and is not on the checksum or translation-serving path. The per-language space is four, not three.

The language code strings used as keys in the `translations` map are the exact codes the existing `GET /api/public/translations?lang=<code>` endpoint already accepts and the language table already stores: `sr`, `cnr`, `en`, `ru`. No new code convention is introduced. The mobile client reads the per-language checksum using the same `language.code` string it already sends as the `lang` query parameter to `fetchNamespace`, so the two sides match by reusing the proven convention.

### Namespaces

The backend returns all 22 namespaces, unfiltered, exactly as today (`ADMIN_PAGES` and `BACKEND_TRANSLATIONS` included). Mobile intersects the response against its local 20-entry `TranslationNamespace` enum, as today. This spec does not change which namespaces are returned — only the shape of each namespace's value.

## 4. Empty / absent language handling — posture B

The backend emits a checksum for **every (namespace, language) pair the four-language set defines**. A pair with zero rows gets the checksum of the empty content set — the same value the current all-languages hash produces for an empty input (`e3b0c44298fc1c14`, the first 16 hex of SHA-256 over the empty string). No `(namespace, language)` key is ever omitted from a present namespace, and no empty-string sentinel is used for translation checksums.

This posture is chosen so the mobile client's missing-checksum semantics do not change. Today a `undefined` translation checksum means exactly one thing — the namespace is absent from the response, a backend error that routes to maintenance. Under posture B, `translations[ns][activeLang]` is a real checksum string for every real pair, so that single meaning is preserved and the mobile gate's existing maintenance routing is untouched.

Cost of posture B: a genuinely-empty `(namespace, language)` pair has no local cache on a fresh install, so the empty-content checksum mismatches "nothing cached," and the client fetches that namespace once. The fetch returns an empty map, the client stores the empty payload plus the empty-content checksum, registers nothing into i18n, and proceeds. On every later boot the checksum matches and nothing is fetched. The cost is one harmless no-op fetch per genuinely-empty pair per install, self-healing after the first boot. This is a bounded, one-time cost, not a maintenance trigger.

(Posture A — omit absent pairs — was considered and rejected. It would force the mobile gate to distinguish "namespace object missing" from "language key missing inside a present namespace," splitting the single `undefined` meaning into two and enlarging the mobile change. Posture B keeps the mobile change at the size described in Part 6.)

## 5. Backend implementation

This is the substantive work.

### Computation

For each namespace `NS` and each of the four languages `L`: gather only `L`'s rows in `NS`, sort by key (the language dimension is now fixed, so the sort key is the translation key), hash the `key|value` lines with the same SHA-256-first-16-hex algorithm used today, and emit under `translations[NS][L]`. A pair with zero rows hashes the empty set, yielding `e3b0c44298fc1c14` (posture B). The hash algorithm, the `key|value` line format, and the 16-hex truncation are unchanged — only the scope of the hashed material narrows from all-languages to one language. The existing per-language row source `loadTranslationsFromDb(namespace, langCode)` (already used to build the per-language `redisTranslations` payload cache) is a ready source for the per-language rows.

### Checksum storage (configuration table)

Checksum values live in the `configuration` table, read through the in-memory `configurationCache` (Redis is not on the checksum delivery path). `VersionChecksumService.persistChecksum` throws if the config key is absent — this is a deliberate safety net and is kept. The per-namespace keys (`translations.checksum.<NS>`, 22 rows) become per-(namespace, language) keys: `translations.checksum.<NS>.<lang>`, **88 rows** (22 namespaces × 4 languages). These rows are pre-seeded in the seed SQL before `VersionChecksumService` runs — a seed-file edit per the Part 12 pre-launch V1-fold convention, not a Flyway migration. The throw-if-missing invariant is not relaxed; the rows must exist before boot recompute.

The catalog checksum keys (`catalog.checksum.<code>`) are unchanged.

### Caching note

The `redisTranslations` payload cache is **already** keyed per `(namespace, language)` (`"<NS>:<lang>"`). Only the checksum layer is namespace-only today. This feature brings the checksum layer in line with the payload layer; the payload cache needs no change.

### Invalidation — boot

`VersionChecksumService.onAppReady` (`@Order(3)`) recomputes and compares per namespace today, with an "any language missing from Redis" short-circuit that rebuilds all languages of a namespace on any single-language miss. The boot path becomes per-(namespace, language): compute, compare, and persist each pair independently. The all-or-nothing language rebuild can become per-pair.

### Invalidation — admin edit (core scope, not optional)

`DefaultTranslationService.updateTranslation` currently calls `rebuildTranslationCacheAsync(namespace, language)`, which rebuilds the single edited `(namespace, language)` Redis payload but then recomputes the **namespace-wide** checksum. This must change to recompute and persist only the edited `(namespace, language)` checksum. This is the point of the feature: leaving the namespace-wide recompute in place means a Russian edit still recomputes Serbian's checksum path, partially defeating the change. Tightening to per-pair also removes the existing race when two languages of the same namespace are edited concurrently.

### Read path (controller)

`VersionController.getVersions` assembles the nested response by iterating namespaces × the live language set (from `languageRepository.findAll()`, already available to the service). The `Cache-Control: max-age=300, public, stale-while-revalidate=86400` header is unchanged.

## 6. Mobile client swap

The mobile change is small and confined to two files: `src/lib/store/bootFreshness.ts` (the three isolated helpers) and `src/lib/store/bootStore.ts` (one gate-body region). The boot-redesign feature isolated the first three of these for exactly this swap. The audit found a fourth touch point the isolation contract did not enumerate; it is included below.

### The four edit points

1. **`VersionsResponse` type** (`bootFreshness.ts`). Before: `translations: Record<TranslationNamespace, string>`. After: `translations: Record<TranslationNamespace, Partial<Record<string, string>>>` — namespace → (language code → checksum). Partial is defensive; under posture B every real pair is present, but the type does not need to assert completeness. One type, one file.

2. **The checksum-key helper** (`translationsChecksumKey`, `bootFreshness.ts`). It already takes `(ns, lang)` and ignores `lang` today, returning `translations_checksum_<NS>`. The body changes to return `translations_checksum_<NS>_<lang>`. Every call site already passes both arguments, so no call site changes shape.

3. **The staleness helper** (`isNamespaceStaleForActiveLanguage`, `bootFreshness.ts`). Its internal read changes from `backendChecksums[ns]` (a string) to `backendChecksums[ns]?.[activeLang]` (a string under posture B). The `activeLang` parameter, accepted but unused today, becomes load-bearing. The gate's call to the helper does not change.

4. **The gate-body checksum read** (`bootStore.ts`, around the per-namespace staleness loop). The gate reads `versions.translations[ns]` directly as a bare string, uses that value for the missing-checksum decision, and persists it via `setStoredNamespaceChecksum`. After the type swap, that direct read must index `[activeLang]` to recover the per-language checksum string. Without this edit, the persist step would store the entire per-language map object as the checksum string, corrupting the stored checksum so every later boot mis-compares. This is the fourth edit point the original three-helper isolation contract did not list. It is one region: the direct read, the missing-checksum branch that consumes it, and the persist call.

### What stays the same

The fetch primitive (`fetchNamespace(ns, lang)`), the `addResourceBundle` register-into-i18n call, the gate's overall shape (call `/versions`, compare per-NS, refetch stale slices, register, advance), the single backend-down rule (`/versions` times out → maintenance), the silent-failure posture, the translation payload storage key (`translations_<NS>_<lang>`, already per-language), and the 20-namespace intersection against the local enum.

### Test fixtures

The `bootStore.test.ts` and `bootFreshness.test.ts` `/versions` fixtures build `translations: { NS: 'string' }` today and must be reshaped to the nested per-language map. The `bootFreshness.test.ts` helper-internal tests change their assertions (the checksum-key "lang ignored" assertions flip to per-language keys; the staleness-helper fixtures become per-language maps with new `activeLang`-dimension cases). The `bootStore.test.ts` Gate-4 behavioral tests should stay green on behavior once fixtures are reshaped, plus one new assertion covering edit point 4 — that the gate persists the per-language checksum **string**, not the map object. One test fixture uses the misleading language code `rs-sr`; correct it to a real code (`sr`) when reshaping, since the per-language lookup key is the real code the backend serves.

### Staleness rule after the swap

The transitional two-condition rule (checksum mismatch OR no cached payload) collapses to effectively one condition: a namespace is stale for the active language if the per-(NS, lang) checksum differs from the locally-stored per-(NS, lang) checksum, or no cached payload exists (in which case no local checksum exists either, so the comparison fails by missing data — same outcome). The OR-clause stays in the code but its language-switch trickery is gone.

## 7. Trust boundary (conventions Part 11)

Clean, both sides. `getVersions` is parameterless; no client-supplied value reaches the computation, the cache/config key, or the response — everything is server-side stored data and server config. On mobile, no `/versions` value feeds any moderation, authorization, or state-transition decision; the worst a malformed response can do is over-refetch, under-refetch, or route to maintenance, and the gate fails toward fetch or toward maintenance, never toward a privileged action. The per-language upgrade changes the shape of an already-trusted, non-security-bearing value and introduces no new trust surface.

## 8. Build order

Backend first, then mobile, then a coordinated single deploy. The `/v2` parallel-endpoint fallback in the original future doc is rejected — pre-launch, no real users, the coordinated cost is near-zero, and a parallel endpoint is permanent debt.

1. **Backend** (`oglasino-backend`, `dev` branch) — implement Part 5. Pre-seed the 88 config keys. Validate by hand with `curl` against `/api/public/versions` returning the nested shape for known data; confirm a single-language admin edit bumps only that language's checksum.
2. **Mobile** (`oglasino-expo`, `new-expo-dev` branch) — implement Part 6, the four edit points plus fixture reshape.
3. **Coordinated deploy** — backend and mobile ship together. Web is unaffected.

## 9. Definition of done

- `GET /api/public/versions` returns the Part 3 nested shape with per-(namespace, language) checksums for all four languages.
- A translation change in language `L` bumps only `translations[NS][L]`, never another language's checksum for that namespace.
- The admin-edit invalidation recomputes only the edited `(namespace, language)` checksum.
- The 88 per-(NS, lang) config rows are pre-seeded; boot does not throw on a missing checksum key.
- Mobile's four edit points (Part 6) land; the gate's external shape is unchanged.
- A Serbian user does not refetch Serbian when a Russian-only translation changes — the observable signal the upgrade worked.
- A genuinely-empty `(NS, lang)` pair fetches at most once per install (posture B), then matches forever; it never routes to maintenance.
- `decisions.md` entry at feature close recording: the per-NS-collapsed contract is replaced; the `/v2` fallback was rejected for a coordinated deploy; this feature closes the roadmap pointer the boot-redesign feature added.
- `state.md` flips this feature from `planned` to `shipped` when both backend and mobile are live.
