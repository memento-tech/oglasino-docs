# Audit — keyword-stuffing ratio multipliers

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-27
**Task:** Inventory the surface for lifting `KeywordStuffingDetector.Tuning` ratio values to config.

---

## 1. Call sites and current values

### KeywordStuffingAnalyzer (name-side)

**File:** `src/main/java/com/memento/tech/oglasino/moderation/analyzer/KeywordStuffingAnalyzer.java`
**Tuning construction:** lines 46–53
**Hardcoded constants:** lines 29–33

| Field | Constant name | Value |
|---|---|---|
| `minEligibleWords` | `NAME_MIN_ELIGIBLE_WORDS` | `2` |
| `minOccurrenceRatio` | `NAME_MIN_OCCURRENCE_RATIO` | `0.30` |
| `baseRatio` | `NAME_BASE_RATIO` | `0.80` |
| `ratioDecayPerWord` | `NAME_RATIO_DECAY_PER_WORD` | `0.10` |
| `ratioFloor` | `NAME_RATIO_FLOOR` | `0.30` |

The first two `Tuning` fields (`minWordLength`, `consecutiveThreshold`) are already read from config via `configurationService.getRequiredIntConfig(MIN_WORD_LENGTH_KEY)` and `configurationService.getRequiredIntConfig(FREQUENCY_THRESHOLD_KEY)` at lines 40–41. The remaining five fields are the hardcoded constants above.

### SpammyDescriptionAnalyzer (description-side)

**File:** `src/main/java/com/memento/tech/oglasino/moderation/analyzer/SpammyDescriptionAnalyzer.java`
**Tuning construction:** lines 72–79
**Hardcoded constants:** lines 40–44

| Field | Constant name | Value |
|---|---|---|
| `minEligibleWords` | `DESCRIPTION_MIN_ELIGIBLE_WORDS` | `3` |
| `minOccurrenceRatio` | `DESCRIPTION_MIN_OCCURRENCE_RATIO` | `0.12` |
| `baseRatio` | `DESCRIPTION_BASE_RATIO` | `0.40` |
| `ratioDecayPerWord` | `DESCRIPTION_RATIO_DECAY_PER_WORD` | `0.01` |
| `ratioFloor` | `DESCRIPTION_RATIO_FLOOR` | `0.12` |

Same as name-side: `minWordLength` and `consecutiveThreshold` are already from config (reuses `KeywordStuffingAnalyzer.MIN_WORD_LENGTH_KEY` and `FREQUENCY_THRESHOLD_KEY` at lines 52–55).

### Do the two analyzers use the same values?

**No — they intentionally differ.** The description side is more permissive across all five fields. This is documented in the Javadoc at `SpammyDescriptionAnalyzer.java:28–29` and comments at lines 38–39: "long-form text legitimately repeats words more often."

| Tuning field | Name-side | Description-side |
|---|---|---|
| `minEligibleWords` | 2 | 3 |
| `minOccurrenceRatio` | 0.30 | 0.12 |
| `baseRatio` | 0.80 | 0.40 |
| `ratioDecayPerWord` | 0.10 | 0.01 |
| `ratioFloor` | 0.30 | 0.12 |

---

## 2. `Tuning` record definition

**File:** `src/main/java/com/memento/tech/oglasino/moderation/analyzer/KeywordStuffingAnalyzer.java`
**Lines:** 65–72

```java
record Tuning(
    int minWordLength,
    int consecutiveThreshold,
    int minEligibleWords,
    double minOccurrenceRatio,
    double baseRatio,
    double ratioDecayPerWord,
    double ratioFloor) {}
```

Nested inside the package-private `KeywordStuffingDetector` utility class (line 61). Plain Java record — no constructor logic, no validation, no defaults. All seven fields are positional.

The record has seven fields total. Two (`minWordLength`, `consecutiveThreshold`) are already config-driven. The `issues.md` entry names four (`baseRatio`, `ratioDecayPerWord`, `ratioFloor`, `minOccurrenceRatio`). That leaves one unlisted field: `minEligibleWords`. The brief scope says "four values" matching the `issues.md` entry, but there are actually **five** hardcoded values per analyzer (see section 1). The fix brief should decide whether `minEligibleWords` is also lifted — it is the same kind of tuning parameter.

---

## 3. `ConfigurationService` pattern survey

### Method signatures

From `ConfigurationService.java` (interface):

| Method | Return | Behavior on missing key |
|---|---|---|
| `getConfig(String)` | `String` | Returns `""` (empty string) |
| `getConfig(String, String)` | `String` | Returns the default argument |
| `getBooleanConfig(String)` | `boolean` | Silent default `false` |
| `getDoubleConfig(String)` | `double` | Silent default `0` |
| `getIntConfig(String)` | `int` | Silent default `0` |
| `getRequiredConfig(String)` | `String` | Throws `IllegalStateException` |
| `getRequiredIntConfig(String)` | `int` | Throws `IllegalStateException` |
| `getRequiredDoubleConfig(String)` | `double` | Throws `IllegalStateException` |

### Convention: `getRequired*` (fail-fast) is the moderation standard

Every existing moderation analyzer uses the `getRequired*` family. Examples from call sites:

1. **`KeywordStuffingAnalyzer.java:40`** — `configurationService.getRequiredIntConfig(MIN_WORD_LENGTH_KEY)`. Fails fast on missing key.
2. **`SpammyDescriptionAnalyzer.java:51`** — `configurationService.getRequiredIntConfig(REPEATING_CHARS_CONFIG_KEY)`. Same pattern.
3. **`ContentValidationConfig.java:82`** — `configurationService.getRequiredDoubleConfig(GIBBERISH_SHORT_TEXT_KEY_PREFIX + lang)`. Double variant.
4. **`DefaultProductService.java:394`** — `configurationService.getRequiredDoubleConfig(MASSIVE_CHANGE_SIMILARITY_KEY)`. Same pattern for `validation.*` keys.
5. **`ContentValidationConfig.java:136`** — `configurationService.getRequiredConfig(configKey)` for string-valued config (banned words, regex).

The silent-default family (`getBooleanConfig`, `getDoubleConfig`) is used only outside moderation — e.g., `FirebaseAuthFilter.java:154` for `postman.testing.active` and `UserDeletionScheduledJobs.java:119` for the reconciliation toggle.

**Convention for new moderation keys: `getRequiredDoubleConfig`**, matching the existing pattern for `double`-typed moderation thresholds (gibberish entropy, massive-change similarity). The new ratio values are all `double`.

### Key naming convention

Dotted path, all lowercase, underscores within segments: `validation.<analyzer_group>.<specific_param>`. Examples:
- `validation.keyword_stuffing.min_word_length`
- `validation.keyword_stuffing.frequency_threshold`
- `validation.massive_change.similarity_threshold`
- `validation.repeating_chars.description_threshold`
- `validation.gibberish.short_text.entropy_threshold.en`

---

## 4. `data-configuration.sql` seed file

**Path:** `src/main/resources/data/configuration/data-configuration.sql`
**Total rows:** 90 (IDs 1–90)
**Last ID used:** 90 (`catalog.checksum.me`)

### Existing `validation.*` key pattern

IDs 27–56 hold all `validation.*` keys. The block structure:

| ID range | Key pattern |
|---|---|
| 27–29 | `validation.banned_words.{en,sr,ru}` |
| 30–32 | `validation.regex.promo.{en,sr,ru}` |
| 33–35 | `validation.regex.contacts.{en,sr,ru}` |
| 36–38 | `validation.stopwords.{en,sr,ru}` |
| 42–44 | `validation.repeating_chars.threshold`, `validation.excessive_punctuation.threshold`, `validation.all_caps.min_length` |
| 45–46 | `validation.keyword_stuffing.min_word_length`, `validation.keyword_stuffing.frequency_threshold` |
| 47–48 | `validation.massive_change.*` |
| 49–55 | `validation.gibberish.*` |
| 56 | `validation.repeating_chars.description_threshold` |

**Gap:** IDs 39–41 are unused (IDs 23–26 are also unused placeholder rows).

### Next-available IDs

The `validation.*` namespace ends at ID 56. The next namespace starts at ID 57 (`user.deletion.*`). Per conventions Part 6 Rule 3: append at the end of the namespace group. The `validation.keyword_stuffing.*` sub-group is at IDs 45–46.

Options:
- **Use the gap IDs 39–41** to stay within the existing `validation.*` block. Five new keys would need IDs 39–41 plus two more — but the gap only has three slots, so this doesn't cleanly fit 5 (or even 4) plus the need to not collide.
- **Append after ID 56** (the current end of the `validation.*` block) with IDs starting from a safe gap. But IDs 57–65 are taken by `user.deletion.*`.
- **Use IDs 39–41 for three keys, then the remaining keys go at the very end** — but this splits the sub-group, which is worse than the gap.

**Recommendation:** Use the three unused IDs 39, 40, 41 for three of the keys, then use another gap or append at the end of the file (ID 91+) for the remaining keys. However, the cleanest approach is: if there are exactly 4 new keys (matching `issues.md`), use IDs 39–41 plus one more (e.g., reuse one of 23–26, which are placeholder rows with key `'2'`/`'3'`/`'4'`/`'5'` and empty values — these appear to be dead data). If there are 5 keys (including `minEligibleWords`), use 39–41 plus two of 23–26.

**Caution on IDs 23–26:** These rows exist with keys `'2'`, `'3'`, `'4'`, `'5'` and empty values. They appear to be test/placeholder data. The `ON CONFLICT (id) DO NOTHING` clause means reusing these IDs requires first confirming they aren't referenced elsewhere. Since the seed is `ON CONFLICT (id) DO NOTHING`, reusing the IDs with new keys would only work on a fresh DB; existing DBs with the old rows would keep `key='2'` etc. **This is a problem.** The fix brief should either:
- Delete/update the placeholder rows in a migration (but pre-launch, V1 schema fold applies), or
- Append the new rows at the end of the file with IDs 91+.

**Safest option for the fix brief: append with IDs 91–94 (or 91–95 if 5 keys).** This avoids collisions with existing rows.

---

## 5. `auditRequiredConfig`

**File:** `src/main/java/com/memento/tech/oglasino/moderation/ContentValidationConfig.java`
**Method:** `auditRequiredConfig(ApplicationReadyEvent)` at line 106

### Does it currently enumerate moderation analyzer keys?

Yes. It builds a list from two sources (method `requiredKeys()` at line 121):

1. **Per-language prefixes** (`PER_LANGUAGE_PREFIXES`, lines 49–56): six prefixes × 3 languages = 18 keys. Covers `banned_words`, `regex.promo`, `regex.contacts`, `stopwords`, and the two gibberish entropy thresholds.
2. **Global required keys** (`GLOBAL_REQUIRED_KEYS`, lines 58–68): nine keys including `validation.keyword_stuffing.min_word_length` and `validation.keyword_stuffing.frequency_threshold`.

### Would new required keys need to be added there?

**Yes.** The four (or five) new keys are of the `getRequiredDoubleConfig` family — same contract as the existing keys in `GLOBAL_REQUIRED_KEYS`. They must be added to `GLOBAL_REQUIRED_KEYS` so the boot-time audit catches missing seeds. Without this, a missing seed would only surface at the first moderation call (runtime throw), not at boot.

Note: `ConfigurationSeedTest.java` (line 38, `REQUIRED_KEYS` list) also maintains a parallel list of keys expected in the seed file. New keys must be added there too, or the seed-coverage test will not catch a missing row.

---

## 6. Test files

### `KeywordStuffingAnalyzerTest.java`
**File:** `src/test/java/com/memento/tech/oglasino/moderation/analyzer/KeywordStuffingAnalyzerTest.java`
**Tests:** 7 test methods

**How `Tuning` is constructed:** Tests do NOT construct `Tuning` directly. They call `analyzer.analyze(...)` which internally constructs `Tuning` using the hardcoded constants and mocked `configurationService` values. Tests mock `configurationService.getRequiredIntConfig()` for `MIN_WORD_LENGTH_KEY` and `FREQUENCY_THRESHOLD_KEY` (lines 115–126). The five ratio constants are not visible or controllable from tests — they use the `private static final` values baked into the analyzer.

**Impact of the fix:** After lifting ratios to config, tests would need to mock the new `getRequiredDoubleConfig` calls for the four/five ratio keys. The `stubThresholds` helper (line 115) would need to be expanded. Alternatively, tests could remain as-is if the analyzer provides a fallback — but the `getRequired*` convention means no fallback, so tests MUST mock the new keys.

### `SpammyDescriptionAnalyzerTest.java`
**File:** `src/test/java/com/memento/tech/oglasino/moderation/analyzer/SpammyDescriptionAnalyzerTest.java`
**Tests:** 8 test methods

**Same pattern:** Tests call `analyzer.analyze(...)` and mock the three existing config keys. The five description-side ratio constants are baked in. After the fix, the `stubDefaults` helper (line 129) would need to mock the new ratio keys too.

### `ContentModerationGoldenSetTest.java`
**File:** `src/test/java/com/memento/tech/oglasino/moderation/ContentModerationGoldenSetTest.java`
**Tests:** 1 parameterized test method with 34 golden cases

**How config is handled:** Uses a mock `ConfigurationService` with `seededIntFor()` (line 251) to return integer config values. The `KeywordStuffingAnalyzer` and `SpammyDescriptionAnalyzer` instances are wired with this mock (lines 92–96). After the fix, `seededIntFor` would need to handle the new double keys (or a parallel `seededDoubleFor` would be needed, since the mock currently only stubs `getRequiredIntConfig`).

### `LocalContentModeratorTest.java`
**File:** `src/test/java/com/memento/tech/oglasino/moderation/impl/LocalContentModeratorTest.java`
Both analyzers are `@Mock`ed here (lines 44, 48), so this test is unaffected — it doesn't exercise the internal `Tuning` construction.

### `ConfigurationSeedTest.java`
**File:** `src/test/java/com/memento/tech/oglasino/moderation/ConfigurationSeedTest.java`
**Impact:** `REQUIRED_KEYS` list (line 38) must include the new keys to ensure the seed file coverage test catches missing rows.

---

## 7. Proposed key names

Following the existing `validation.keyword_stuffing.*` convention established by `validation.keyword_stuffing.min_word_length` and `validation.keyword_stuffing.frequency_threshold`:

### For name-side (KeywordStuffingAnalyzer):

| Tuning field | Proposed key |
|---|---|
| `baseRatio` | `validation.keyword_stuffing.name.base_ratio` |
| `ratioDecayPerWord` | `validation.keyword_stuffing.name.ratio_decay_per_word` |
| `ratioFloor` | `validation.keyword_stuffing.name.ratio_floor` |
| `minOccurrenceRatio` | `validation.keyword_stuffing.name.min_occurrence_ratio` |

### For description-side (SpammyDescriptionAnalyzer):

| Tuning field | Proposed key |
|---|---|
| `baseRatio` | `validation.keyword_stuffing.description.base_ratio` |
| `ratioDecayPerWord` | `validation.keyword_stuffing.description.ratio_decay_per_word` |
| `ratioFloor` | `validation.keyword_stuffing.description.ratio_floor` |
| `minOccurrenceRatio` | `validation.keyword_stuffing.description.min_occurrence_ratio` |

### Note on `minEligibleWords`

The `issues.md` entry names four fields. There are actually five hardcoded values per analyzer. `minEligibleWords` is structurally identical to the other four (a tuning knob that an admin might want to adjust). Proposed key if included:

| Tuning field | Proposed key (name) | Proposed key (description) |
|---|---|---|
| `minEligibleWords` | `validation.keyword_stuffing.name.min_eligible_words` | `validation.keyword_stuffing.description.min_eligible_words` |

### Naming rationale

- Uses `name.` and `description.` segments to distinguish the two sides, matching the existing `validation.repeating_chars.threshold` (name-side) vs `validation.repeating_chars.description_threshold` (description-side) precedent but with a cleaner sub-path style.
- Keeps the `validation.keyword_stuffing.*` prefix so the keys sort together in the admin UI and in `data-configuration.sql`.
- Uses `snake_case` within segments, matching every existing key in the file.

### Total new key count

- **4 per side × 2 sides = 8 keys** if the brief scope is the `issues.md` four fields only.
- **5 per side × 2 sides = 10 keys** if `minEligibleWords` is also lifted.

The fix brief should make the call on `minEligibleWords`. If it stays hardcoded, document why in the session summary's Part 4a "Considered and rejected" section.
