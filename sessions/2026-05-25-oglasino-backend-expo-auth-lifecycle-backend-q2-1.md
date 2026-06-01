# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-25
**Task:** Read-only verification of the exact JSON body shape returned by the backend when a banned user (`User.disabled = true`) makes an authenticated request, and when `firebase-sync` rejects a disabled or email-banned user.

## Implemented

- Read-only verification, no code changes.

## Files touched

- None.

## Tests

- N/A — read-only.

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — read-only session.
- Part 4a (simplicity): N/A — read-only session (see "For Mastermind" for structured evidence).
- Part 4b (adjacent observations): confirmed — no adjacent observations; the files read are well-structured.
- Part 6 (translations): N/A this session.
- Other parts touched: Part 7 (error contract) — confirmed, the three error responses all use the `{errors: [{field, code, translationKey}]}` envelope. Part 11 (trust boundaries) — confirmed, the response shape is itself part of the trust boundary contract; the shapes are server-derived constants, not influenced by client input.

## Known gaps / TODOs

- None.

## For Mastermind

### Part 4a simplicity evidence (required)

- Added (earned complexity): nothing.
- Considered and rejected: nothing.
- Simplified or removed: nothing.

### The three answers

---

#### Q1 — Mid-session USER_BANNED (from `FirebaseAuthFilter`)

**Code path:** `FirebaseAuthFilter.java:83-86` checks `authData.disabled()`. On `true`, calls `writeUserBanned(response)` at line 84, then returns (short-circuits the filter chain).

**Response construction:** `writeUserBanned` at `FirebaseAuthFilter.java:136-143` writes a **hardcoded inline JSON string** directly to `HttpServletResponse`. The string constant is defined at lines 32-34:

```java
private static final String USER_BANNED_BODY =
    "{\"errors\":[{\"field\":null,\"code\":\"USER_BANNED\","
        + "\"translationKey\":\"user.banned\"}]}";
```

**Literal JSON on the wire:**

```json
{"errors":[{"field":null,"code":"USER_BANNED","translationKey":"user.banned"}]}
```

- HTTP status: **403**
- Content-Type: `application/json`
- Charset: UTF-8
- `field`: present as JSON `null` (not absent)
- `code`: `"USER_BANNED"`
- `translationKey`: `"user.banned"`

---

#### Q2 — Sign-in USER_BANNED (from `firebase-sync` handler)

**Code path:** `AuthController.java:90-91` checks `user.isDisabled()`. On `true`, returns `errorResponse(HttpStatus.FORBIDDEN, "USER_BANNED", "user.banned")`.

**Response construction:** `AuthController.java:135-141` private helper method builds a `ResponseEntity<ProductErrorResponse>`:

```java
private ResponseEntity<ProductErrorResponse> errorResponse(
    HttpStatus status, String code, String translationKey) {
  return ResponseEntity.status(status)
      .body(
          new ProductErrorResponse(
              List.of(new ProductErrorResponse.FieldError(null, code, translationKey))));
}
```

`ProductErrorResponse` is a Java record at `ProductErrorResponse.java:15`: `record ProductErrorResponse(List<FieldError> errors)` with nested `record FieldError(String field, String code, String translationKey)`.

Jackson serializes this to the same wire shape as Q1.

**Literal JSON on the wire:**

```json
{"errors":[{"field":null,"code":"USER_BANNED","translationKey":"user.banned"}]}
```

- HTTP status: **403**
- Content-Type: `application/json` (Spring default for `@RestController`)
- `field`: present as JSON `null` (not absent)
- `code`: `"USER_BANNED"`
- `translationKey`: `"user.banned"`

**Shape identical to Q1?** Yes. Both produce the same JSON structure with the same field values. The only difference is the construction mechanism: Q1 uses a hardcoded string written directly to the servlet response; Q2 uses Spring's `ResponseEntity` + Jackson serialization of `ProductErrorResponse`. The wire output is identical.

---

#### Q3 — Sign-in EMAIL_BANNED (from `firebase-sync` handler)

**Code path:** `AuthController.java:69-76` checks `userAuditService.isEmailBanned(email)`. On `true`, attempts to delete the Firebase user (best-effort), then returns `errorResponse(HttpStatus.FORBIDDEN, "EMAIL_BANNED", "email.banned")`.

**Response construction:** Same `errorResponse` helper as Q2 (`AuthController.java:135-141`), same `ProductErrorResponse` record, different code and translationKey string values.

**Literal JSON on the wire:**

```json
{"errors":[{"field":null,"code":"EMAIL_BANNED","translationKey":"email.banned"}]}
```

- HTTP status: **403**
- Content-Type: `application/json` (Spring default)
- `field`: present as JSON `null` (not absent)
- `code`: `"EMAIL_BANNED"`
- `translationKey`: `"email.banned"`

**Shape identical to Q1/Q2?** Yes — same envelope structure (`{errors: [{field, code, translationKey}]}`), same `field: null` presence. Only the `code` and `translationKey` string values differ.

---

### Summary for mobile's interceptor

All three 403 paths produce the same envelope:

```json
{
  "errors": [
    {
      "field": null,
      "code": "<CODE>",
      "translationKey": "<KEY>"
    }
  ]
}
```

| Path | `code` | `translationKey` | Source |
|---|---|---|---|
| Mid-session (filter) | `USER_BANNED` | `user.banned` | `FirebaseAuthFilter.java:32-34` |
| Sign-in disabled | `USER_BANNED` | `user.banned` | `AuthController.java:91` |
| Sign-in email-banned | `EMAIL_BANNED` | `email.banned` | `AuthController.java:76` |

Mobile's Φ1 axios 403 interceptor logic `errorBody?.errors?.[0]?.code === 'USER_BANNED'` will work correctly against both USER_BANNED paths. To also catch EMAIL_BANNED, the interceptor should additionally check for `'EMAIL_BANNED'`.

Key structural facts for the interceptor:
- `errors` is always a **JSON array** (never a bare object).
- `field` is always present as **JSON `null`** (not absent from the object).
- `translationKey` is always present (never absent).
- The array always has exactly **one element** in these three cases.

---

### Brief vs reality — spec-vs-code drift on `translationKey`

**Drift found.** The spec at `user-deletion.md` §8.8 (line 956-957) documents:

| Code | Translation key (spec) |
|---|---|
| `USER_BANNED` | `errors.user.banned` |
| `EMAIL_BANNED` | `errors.email.banned` |

The same spec references these keys at line 144 (`translationKey: 'errors.user.banned'`) and line 360 (`translationKey: 'errors.email.banned'`).

**The code produces different keys:**

| Code | Translation key (code) | Source |
|---|---|---|
| `USER_BANNED` | `user.banned` | `FirebaseAuthFilter.java:34`, `AuthController.java:91` |
| `EMAIL_BANNED` | `email.banned` | `AuthController.java:76` |

The `errors.` prefix is in the spec but **not in the code**. The code emits `user.banned` and `email.banned`.

**Why this matters:** If mobile's interceptor uses `translationKey` to display a localized message (by looking up the key in its translation bundle), it must use `user.banned` / `email.banned`, not `errors.user.banned` / `errors.email.banned`. If web already consumes this in production, web's translation bundle presumably has the keys without the `errors.` prefix — worth a quick cross-check by Mastermind if there's any doubt.

**Why this does NOT block Φ1:** Mobile's interceptor matches on `code` (`USER_BANNED`), not `translationKey`. The `code` values match the spec exactly. The `translationKey` drift is a documentation inaccuracy, not a wire-contract problem for the interceptor's matching logic.

**Recommended resolution:** Update `user-deletion.md` §8.8, line 144, and line 360 to reflect the actual code values (`user.banned` / `email.banned`). This is a Docs/QA edit — draft below.

**Drafted config-file text:** N/A (this is a feature-spec correction, not a config-file edit). The correction targets `oglasino-docs/features/user-deletion.md`:

- §8.8 table, row `USER_BANNED`: change `errors.user.banned` → `user.banned`
- §8.8 table, row `EMAIL_BANNED`: change `errors.email.banned` → `email.banned`
- Line 144: change `translationKey: 'errors.user.banned'` → `translationKey: 'user.banned'`
- Line 360: change `translationKey: 'errors.email.banned'` → `translationKey: 'email.banned'`

Pass to Docs/QA for application.
