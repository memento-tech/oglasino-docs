# Session summary

**Repo:** oglasino-backend
**Branch:** dev (unchanged — no git ops this session)
**Date:** 2026-05-29
**Task:** Confirm the seeded rate-limit (429) translation key — grep the ERRORS-namespace seed SQL, the `RateLimitFilter` 429 body, and the `SystemErrorCode.RATE_LIMITED` constant; verdict whether all three agree on the single key mobile should expect. (READ-ONLY)

## Implemented

- Nothing implemented — this was a read-only audit. No code, no DB writes, no git ops.
- Three independent checks run and cross-verified; all three converge on a single key string.

### Findings

**1. Seed SQL (ERRORS namespace).** The seeded key is **`system.rate_limited`** in the `ERRORS` namespace, present in all four locale seed files:

| Locale | File | Line | Value |
| ------ | ---- | ---- | ----- |
| EN  | `src/main/resources/data/translations/0001-data-web-translations-EN.sql`  | 603 | "You're making requests too quickly. Please wait a moment and try again." |
| RS  | `src/main/resources/data/translations/0001-data-web-translations-RS.sql`  | 601 | "Šaljete previše zahteva. Molimo sačekajte trenutak i pokušajte ponovo." |
| RU  | `src/main/resources/data/translations/0001-data-web-translations-RU.sql`  | 599 | "Vy delaete zaprosy slishkom bystro. Pozhaluysta, podozhdite nemnogo i poprobuyte snova." |
| CNR | `src/main/resources/data/translations/0001-data-web-translations-CNR.sql` | 599 | "Šaljete zahtjeve prebrzo. Molimo sačekajte trenutak i pokušajte ponovo." |

- `product.system.rate_limited` — **does NOT exist** in any seed file.
- `system.rate_limited` — **exists** in all four locales.

**2. `RateLimitFilter` 429 wire body.** `RateLimitFilter.java:30-35` (under `security/filter/`) builds the `RATE_LIMITED_BODY` constant. It is **not** a hardcoded literal key — it is assembled from the enum:

```java
private static final String RATE_LIMITED_BODY =
    "{\"errors\":[{\"field\":null,\"code\":\""
        + SystemErrorCode.RATE_LIMITED.name()
        + "\",\"translationKey\":\""
        + SystemErrorCode.RATE_LIMITED.getTranslationKey()
        + "\"}]}";
```

On a real Spring 429 (`RateLimitFilter.java:73-77`) the exact body written on the wire is:

```json
{"errors":[{"field":null,"code":"RATE_LIMITED","translationKey":"system.rate_limited"}]}
```

So the wire `translationKey` is **`system.rate_limited`**. Because the body is derived from `SystemErrorCode.RATE_LIMITED.getTranslationKey()` at class-load, it is guaranteed identical to step 3 by construction — the two cannot drift.

**3. `SystemErrorCode.RATE_LIMITED` constant.** `SystemErrorCode.java:11`:

```java
RATE_LIMITED("system.rate_limited", HttpStatus.TOO_MANY_REQUESTS),
```

`translationKey` = **`system.rate_limited`**. The error-code split (2026-05-29) placed `RATE_LIMITED` on `SystemErrorCode` (implements `ErrorCode`), confirmed.

### Verdict (one line)

All three sources agree. The single key mobile should emit / expect on the wire is **`system.rate_limited`** (ERRORS namespace, seeded in EN/RS/RU/CNR). The 429 envelope also carries `code: "RATE_LIMITED"`; mobile may key off either, but the resolving translation key is `system.rate_limited`.

## Files touched

- None. Read-only audit.

## Tests

- None run. No code changed; nothing to test. `./mvnw test` / `spotless:check` not applicable to a read-only grep audit.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change

(All read-only. No backend change this session. See "For Mastermind" for one docs-staleness flag that lives in the docs repo, not in these four files.)

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code touched.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): one observation flagged in "For Mastermind".
- Part 6 (translations): confirmed — verified the seed conforms to the post-split key; no keys added.
- Part 7 (error contract): confirmed — the 429 body matches the Part 7 wire shape (`{errors:[{field, code}]}`, `field: null` for object-level), with an additional `translationKey` field carried alongside `code`.
- Other parts touched: none.

## Known gaps / TODOs

- None. The audit's three questions are answered with exact strings and line numbers.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **D2 web-bug escalation — NOT a bug.** The brief asked me to flag web's `system.rate_limited` 429-synth constant as a web-bug escalation candidate *if it disagrees with the seed*. It does not disagree: web's `system.rate_limited` **matches** the live seed and the live `SystemErrorCode`/`RateLimitFilter` truth exactly. There is no resolving-key mismatch on the web side. No escalation needed.

- **Stale spec reference (docs-repo, low).** The only stale string among the three brief candidates is `product.system.rate_limited`, described in the brief as living in the "spec error-code table (pre-error-code-split)." That key does not exist in the backend seed or code — it is the pre-split name. If that spec table still lists `product.system.rate_limited`, it is stale and should be updated to `system.rate_limited`. This lives in `oglasino-docs` (the feature/spec table), not in this repo and not in the four config files, so it is a Docs/QA edit, not a backend change. Flagging for routing. Severity low (documentation accuracy; no runtime impact — the live key is correct everywhere in code).

- **Adjacent observation (Part 4b, informational, low).** The brief framed `RATE_LIMITED_BODY` as a "hardcoded 429 body." In reality it is assembled at class-load from `SystemErrorCode.RATE_LIMITED.name()` + `.getTranslationKey()` (`RateLimitFilter.java:30-35`). This is a strengthening, not a defect: the wire `code` and `translationKey` are guaranteed to track the enum, so the filter body can never silently diverge from `SystemErrorCode`. I did not change anything — noting it only so the "hardcoded body" phrasing in any future brief is corrected. No fix required.

- Nothing else flagged.
