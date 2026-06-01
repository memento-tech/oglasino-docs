# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-30
**Task:** Read-only audit — confirm against `dev` (not the spec) that `allowPreferenceCookies` is fully removed from backend code/schema and that five COOKIES translation keys were deleted from the seed SQL; report the current COOKIES namespace shape, the `/auth/firebase-sync` request/response shape, and which user-preference fields are still server-enforced.

## Implemented

- Read-only audit, no code. No files changed, no migrations, no mutating commands.
- All findings proven by grep with commands and hit counts shown; absence claims proven with zero-hit greps; presence claims cited with file:line.

## Files touched

- none (read-only)

## Tests

- Ran: none (read-only audit; no mutating runs per brief)
- Result: N/A

---

## Section 1 — `allowPreferenceCookies` end-state in backend code

**Verdict: CONFIRMED — the field is absent end-to-end (column, all DTOs, entity, every propagator/mapper/service). The `/auth/firebase-sync` request body is `{ displayName?: string }` only.**

### Whole-tree sweep (both spellings, case-insensitive)

```
$ grep -rin "allowPreferenceCookies"   src/main   → 0 hits (exit 1)
$ grep -rin "allow_preference_cookies" src/main   → 0 hits (exit 1)
$ grep -rin "preferencecookie\|preference_cookie" src/main → 0 hits (exit 1)
$ grep -rin "allowPreferenceCookies\|allow_preference_cookies" src → 0 hits (exit 1)
```

**Zero hits across the entire `src` tree (main + test) for both spellings.** Stated explicitly: there are no hits.

### Per item requested

1. **`users` table schema — absent.** `src/main/resources/db/migration/V1__init_schema.sql` is the only migration file (`V1__init_schema.sql`; no `V2+`). The `users` table carries four preference columns and no `allow_preference_cookies`:
   - `allow_emails boolean NOT NULL` — `V1__init_schema.sql:588`
   - `allow_notifications boolean NOT NULL` — `V1__init_schema.sql:589`
   - `allow_phone_calling boolean NOT NULL` — `V1__init_schema.sql:590`
   - `allow_promo_emails boolean NOT NULL` — `V1__init_schema.sql:591`
   - `grep -in "preference" V1__init_schema.sql` → no `preference` column anywhere.

2. **`AuthUserDTO` (response DTO) — absent.** `src/main/java/com/memento/tech/oglasino/dto/AuthUserDTO.java`. Preference fields present: `allowNotifications:15`, `allowEmails:16`, `allowPromoEmails:17`, `allowPhoneCalling:18`. No `allowPreferenceCookies`.

3. **`UpdateUserDTO` (`/secure/user/update` request DTO) — absent.** `src/main/java/com/memento/tech/oglasino/dto/UpdateUserDTO.java`. Preference fields present: `allowNotifications:22`, `allowEmails:23`, `allowPromoEmails:24`, `allowPhoneCalling:25`. No `allowPreferenceCookies`.

4. **`LoginRequest` (`/auth/firebase-sync` request DTO) — absent.** `src/main/java/com/memento/tech/oglasino/dto/LoginRequest.java` carries exactly one field: `displayName:9` (with `@Size`/`@Pattern`). No `allowPreferenceCookies`, no other fields. **Wire body confirmed `{ displayName?: string }` only.**

5. **`User` entity — absent.** `src/main/java/com/memento/tech/oglasino/entity/User.java`. Preference fields present: `allowNotifications:112`, `allowEmails:113`, `allowPromoEmails:114`, `allowPhoneCalling:115`. No `allowPreferenceCookies` field or column mapping.

6. **Propagators / mappers / services — absent.** The whole-tree grep above returns zero hits, so no converter, facade, or service references either spelling. (For completeness, the live preference-copying converters are `AuthUserConverter`, `UpdateUserConverter`, `EntityUserInfoConverter`; none reference a preference-cookie field.)

---

## Section 2 — The five COOKIES translation keys (B4, backend half)

**Verdict: CONFIRMED — all five keys are absent in all four locale files; the nine "likely still exist" keys are present in all four.**

The four locale files exist with the exact names in the brief:
`0001-data-web-translations-{EN,RS,RU,CNR}.sql` (in `src/main/resources/data/translations/`).

### The five allegedly-deleted keys — proof of absence

```
$ grep -c "'COOKIES', '<key>'" 0001-data-web-translations-<EN|RS|RU|CNR>.sql
```

| Key | EN | RS | RU | CNR | Result |
|-----|----|----|----|-----|--------|
| `required.label`          | 0 | 0 | 0 | 0 | **absent in all four** |
| `required.description`    | 0 | 0 | 0 | 0 | **absent in all four** |
| `required.sub.description`| 0 | 0 | 0 | 0 | **absent in all four** |
| `config.label`            | 0 | 0 | 0 | 0 | **absent in all four** |
| `config.description`      | 0 | 0 | 0 | 0 | **absent in all four** |

All five return zero rows in every seed file.

### Inverse — the nine "likely still exist" keys — proof of presence

| Key | EN | RS | RU | CNR | Result |
|-----|----|----|----|-----|--------|
| `notifications.label`       | 1 | 1 | 1 | 1 | **present in all four** |
| `notifications.description` | 1 | 1 | 1 | 1 | **present in all four** |
| `notifications.warning`     | 1 | 1 | 1 | 1 | **present in all four** |
| `email.label`               | 1 | 1 | 1 | 1 | **present in all four** |
| `email.description`         | 1 | 1 | 1 | 1 | **present in all four** |
| `email.warning`             | 1 | 1 | 1 | 1 | **present in all four** |
| `email.promo.label`         | 1 | 1 | 1 | 1 | **present in all four** |
| `email.promo.description`   | 1 | 1 | 1 | 1 | **present in all four** |
| `email.promo.warning`       | 1 | 1 | 1 | 1 | **present in all four** |

file:line (EN, for reference): `notifications.*` `0001-...-EN.sql:1221-1223`, `email.*` `:1224-1226`, `email.promo.*` `:1227-1229`. RS `:1218-1226`, RU `:1219-1227`, CNR `:1216-1224`.

---

## Section 3 — Current COOKIES namespace shape (consolidated)

**Verdict: CONFIRMED — the COOKIES namespace is exactly 33 keys, identical across all four locales (no key missing from any locale). The 24 consent-mode-v2 keys carry a "pending native-translator review" placeholder comment in RS/RU/CNR; EN is the real source.**

### Count reconciliation

A case-insensitive `grep -ic "'COOKIES'"` reports EN=34, RU=34, RS=33, CNR=33. **This is a grep artifact, not a namespace difference.** The 34th match in EN and RU is the NAVIGATION row `owner.account.cookies.label` whose display *value* is the literal `'Cookies'` (matches `'cookies'` case-insensitively); RS/CNR render that value as `'Kolačići'`, so they don't match. The actual COOKIES *namespace* (4th column = `'COOKIES'`) is **33 distinct keys in every file**, byte-identical key sets.

### The 33 COOKIES keys (present in all four locales; EN ids 4000–4032 / RS 6100–6132 / RU 8200–8232 / CNR 1900–1932)

Pre-existing (9 — real translations, NOT placeholder):
`notifications.label`, `notifications.description`, `notifications.warning`, `email.label`, `email.description`, `email.warning`, `email.promo.label`, `email.promo.description`, `email.promo.warning`

Consent-mode-v2 additions (24 — placeholder-flagged in RS/RU/CNR):
`banner.title`, `banner.body`, `banner.accept_all.label`, `banner.reject_all.label`, `banner.customize.label`, `banner.save.label`, `banner.back.label`, `banner.close.label`, `category.necessary.label`, `category.necessary.description`, `category.preference.label`, `category.preference.description`, `category.analytics.label`, `category.analytics.description`, `category.marketing.note`, `footer.link.label`, `settings.preference.label`, `settings.preference.description`, `settings.analytics.label`, `settings.analytics.description`, `settings.necessary.label`, `settings.necessary.description`, `settings.necessary.sub.description`, `page.title`

9 + 24 = 33. **No COOKIES key is missing from any of the four locales.**

### The +1 NAVIGATION key (web "Cookies" footer/menu entry)

`owner.account.cookies.label` — namespace `NAVIGATION`, present in all four: EN `:145` (`Cookies`), RS (`Kolačići`), RU (`Cookies`), CNR (`Kolačići`). This is the "24 + 1 NAVIGATION" key the consent-mode-v2 feature seeded.

### Placeholder-pending-native-review flags

- **EN:** no placeholder comment in the COOKIES section — EN values are the real source-of-truth copy.
- **RS / RU / CNR:** the 24 consent-mode-v2 keys sit under a `-- ... PLACEHOLDER -- pending native-translator review (2026-05-19 User Deletion precedent)` comment. In RS: comment at `0001-...-RS.sql:1228-1229` (covers ids 6109–6132 = `banner.*` through `page.title`), repeated at `:1250`. Same shape in RU (`:1230`, `:1252`) and CNR (`:1227`, `:1249`; CNR note: "Distinct ijekavian copy"; RU note: "Latin transliteration; 'cookies' kept verbatim"). The 9 pre-existing keys (`notifications.*`/`email.*`) are NOT under a placeholder comment — they carry real localized copy.

This matches `state.md`'s Consent Mode v2 open item: "native-translator review of placeholder RS / RU / CNR copy across 25 keys" (24 COOKIES + 1 NAVIGATION).

---

## Section 4 — `/auth/firebase-sync` request/response shape (informational, for F)

**Verdict: CONFIRMED on both points.**

1. **Response DTO carries `wasRegister`.** `AuthUserDTO.java:38` (`private boolean wasRegister;`), getter/setter `:168-174`. It is set server-side in `AuthController.firebaseSync` at `AuthController.java:110` (`dto.setWasRegister(syncResult.wasRegister())`) — derived from `SyncResult` (backend create-vs-match state), explicitly NOT from any client field. The DTO comment (`:32-38`) documents this. The F discriminator is present and trustworthy.

2. **Request body is `displayName`-only; `idToken` is NOT read from the body.** The `/auth/firebase-sync` handler binds `@RequestBody @Valid LoginRequest` (`AuthController.java:53-54`), and `LoginRequest` has only `displayName`. The token is read from the **Authorization header**, not the body: `firebaseAuthService.resolveFirebaseToken(servletRequest)` (`AuthController.java:61`), whose implementation reads `request.getHeader("Authorization")` and strips the `Bearer ` prefix (`DefaultFirebaseAuthService.java:152-165`). **X5 answer: the request does not accept or read `idToken` in the body — header only.**

---

## Section 5 — Trust boundary note (Part 11): which preference fields are real

Endpoint `POST /api/secure/user/update` → `UserUpdateController.updateUserData` (`UserUpdateController.java:27-36`) → `DefaultUserFacade.updateCurrentUserData` (`DefaultUserFacade.java:108-149`).

**Trust boundary — clean.** Identity is derived from `SecurityContextHolder` via `currentUserService.getCurrentUserIdStrict()` (`DefaultUserFacade.java:110`). The client-supplied `UpdateUserDTO.id` only selects the row; a mismatch against the authenticated id is **rejected** (`:111-115`, `if (!currentUserId.equals(user.getId())) return false;`). No client-supplied "before" value is trusted. Acceptable per Part 11 ("a foreign key the server validates against its own data").

Per-field status (read on GET = `getCurrentUserUpdateData` modelMapper; written on POST = the setters in `updateCurrentUserData`):

| Field | Entity | Column | In update DTO | GET returns | POST persists | Verdict |
|-------|--------|--------|---------------|-------------|---------------|---------|
| `allowNotifications` | `User.java:112` | `allow_notifications` (`V1:589`) | `UpdateUserDTO.java:22` | yes | **yes** `DefaultUserFacade.java:135` | **LIVE — server stores & enforces** |
| `allowEmails` | `User.java:113` | `allow_emails` (`V1:588`) | `:23` | yes | **yes** `:136` | **LIVE** |
| `allowPromoEmails` | `User.java:114` | `allow_promo_emails` (`V1:591`) | `:24` | yes | **yes** `:137` | **LIVE** |
| `allowPhoneCalling` | `User.java:115` | `allow_phone_calling` (`V1:590`) | `:25` | yes (`UpdateUserConverter.java:50`) | **NO** — not set in `updateCurrentUserData` | **PARTIAL / write-path ghost** — see adjacent observation #1 |

**Explicit answer to the `allowNotifications` question:** YES — the backend has a live `allowNotifications` field on the entity (`User.java:112`), with a DB column (`allow_notifications`, `V1:589`), on the update DTO (`UpdateUserDTO.java:22`); it is read on GET and **stored on POST** (`DefaultUserFacade.java:135`). The backend field is live. The mobile dead `allowNotifications` toggle would be wiring an existing real field — not a ghost. (Wire-vs-remove is Igor's product call; the backend half is real.)

---

## Cleanup performed

- none needed (read-only audit)

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change
- **Config-file impact: none (read-only audit).** Two adjacent observations are drafted in "For Mastermind" for triage; whether either becomes an `issues.md` entry is Mastermind's call (Docs/QA applies).

## Obsoleted by this session

- nothing (read-only audit)

## Conventions check

- Part 4 (cleanliness): confirmed — no code written
- Part 4a (simplicity): N/A — no code written (structured evidence below: nothing in all three categories)
- Part 4b (adjacent observations): two flagged in "For Mastermind"
- Part 6 (translations): N/A this session (read-only verification of existing seeds; no keys added)
- Part 7 (error contract): N/A this session
- Part 11 (trust boundaries): confirmed — `/secure/user/update` identity derives from `SecurityContextHolder`; client `id` validated against it (Section 5)

## Known gaps / TODOs

- none — all five brief sections answered with file:line evidence. Section 4 (informational, for chat F) was cheap and is answered in full.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit, no code.
  - Considered and rejected: nothing.
  - Simplified or removed: nothing.

- **One-line verdict per section:**
  - Section 1 (`allowPreferenceCookies` removed): **CONFIRMED** — zero hits both spellings across all of `src`; absent from schema, `AuthUserDTO`, `UpdateUserDTO`, `LoginRequest`, `User` entity, and all converters/services. Wire body is `{ displayName? }` only.
  - Section 2 (five COOKIES keys deleted): **CONFIRMED** — all five absent in all four locales; the nine `notifications.*`/`email.*`/`email.promo.*` keys present in all four.
  - Section 3 (current COOKIES shape): **CONFIRMED** — 33 keys, identical across all four locales, none missing anywhere; 24 consent-mode-v2 keys placeholder-flagged in RS/RU/CNR; +1 NAVIGATION key (`owner.account.cookies.label`) present in all four.
  - Section 4 (`/auth/firebase-sync` shape): **CONFIRMED** — `wasRegister` present on `AuthUserDTO` and set server-side; request body is `displayName`-only; `idToken` read from the Authorization header, never from the body (X5 answered).
  - Section 5 (server-enforced preference fields): **MIXED** — `allowNotifications`, `allowEmails`, `allowPromoEmails` are fully live (read + write); `allowPhoneCalling` is read/returned but its write path is missing (see #1). Trust boundary clean.

- **Adjacent observation #1 (Part 4b) — MEDIUM:** `allowPhoneCalling` is accepted on the `/secure/user/update` request DTO (`UpdateUserDTO.java:25`) and returned on GET (`UpdateUserConverter.java:50`), but `DefaultUserFacade.updateCurrentUserData` (`DefaultUserFacade.java:135-137`) sets only `allowNotifications`/`allowEmails`/`allowPromoEmails` — it never calls `setAllowPhoneCalling`. A user toggling "allow phone calling" via this endpoint has the new value **silently dropped**; the stored value stays at its registration default. User-facing (a real preference toggle that does nothing), hence medium. I did not fix this — out of scope for a read-only audit. Note for the mobile chat: `allowPhoneCalling` is a write-path ghost today; do not wire a mobile toggle to it expecting persistence until the backend setter is added.

- **Adjacent observation #2 (Part 4b) — LOW:** `UserUpdateController`'s handler methods are declared `private` (`UserUpdateController.java:23, 28, 39, 46`). Spring MVC currently maps them via reflection so they function, but `private` handler methods are fragile (they break under CGLIB-proxied controllers and are an unusual pattern vs the rest of the codebase). Cosmetic/structural only; pre-existing; not fixed (out of scope).

- **Suggested next step:** the mobile chat is cleared to strip `allowPreferenceCookies` and the five dead COOKIES keys — backend reality matches the consent-mode-v2 claim exactly. The only caveat to pass along is observation #1 (`allowPhoneCalling` write-path gap), which affects the separate decision about mobile preference toggles, not the cookie-cleanup that this audit unblocks.
