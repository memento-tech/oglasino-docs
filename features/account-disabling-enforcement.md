# Account Disabling & Token Revocation Enforcement

End-to-end enforcement of account disabling and Firebase token revocation across backend and clients. This is a stub — when the feature is picked up, a Mastermind chat opens it for the full spec.

**Status:** `planned`
**Branch:** _(not yet created)_

---

## Why this is a feature, not a bug-fix

The backend change is small — effectively one line in `FirebaseAuthFilter`. But it forces cross-repo frontend work: a previously-fine user now starts getting `401`/`403`s mid-session, and both web and mobile clients need to handle that gracefully — detect the rejection, show an appropriate "your account has been disabled" state, stop retrying, force logout.

Shipping the backend half without the frontend half would leave disabled users hitting raw auth errors with no UI handling. The unit of work is the whole thing.

---

## Named pieces

### 1. `FirebaseAuthFilter` reads `authData.disabled()`

The `disabled` field is currently fetched from the DB into `AuthenticatedUserDTO`, serialized into Redis (`redisUserAuth`), and **never read anywhere on the request path**.

The admin "disable user" endpoint flips the DB column and calls Firebase admin SDK — but no in-process code consults the flag, so a disabled user keeps making authenticated requests until their Firebase ID token naturally expires (~1 hour).

Wiring the filter to check the flag closes the in-process loophole. The expected shape: after `verifyIdToken` succeeds and `authData` is loaded from cache, `FirebaseAuthFilter` checks `authData.disabled()` and short-circuits with `403` (or whichever status the spec settles on) when true.

### 2. `verifyIdToken(checkRevoked=true)` — design trade-off, deliberate

`FirebaseAuthFilter.verifyToken` currently calls `verifyIdToken` **without** `checkRevoked=true`, so a revoked Firebase token still passes verification until expiry.

Setting `checkRevoked=true` enforces revocation per-request — but it adds a Firebase admin-SDK round-trip on every authenticated request, with a real performance cost, and partially defeats the point of `redisUserAuth`.

This is a genuine trade-off requiring a [`decisions.md`](../decisions.md) entry when the feature lands, not a quick flag-flip. The trade-off space:

- Always `checkRevoked=true` — strongest correctness, real latency cost
- Never (status quo) — fastest, revocation only enforced via cache eviction
- Conditional — `checkRevoked=true` only on cache miss, or only on a subset of routes (admin/billing)

---

## Cross-repo work the backend change forces

Web and mobile both need rejection-handling for the new `401`/`403` state:

- **Detection** — the auth-aware fetch layer must distinguish "this specific request is unauthorized" from "the whole session is invalid" and route to the right UX.
- **UX** — show an "your account has been disabled" state, not the generic auth-failure path that prompts re-login.
- **Stop retrying** — silent retry loops on auth errors will hammer the backend; needs explicit short-circuit on this state.
- **Force logout** — clear the local auth state and route to the landing page.

The feature scope **includes** that work, not just the backend line.

---

## Auth-TTL interaction (load-bearing)

With `redisUserAuth` TTL now at 30min (raised 2026-05-14 — see [`decisions.md`](../decisions.md)), a disabled user's cache entry persists for up to 30min unless `saveUser` evicts it.

`saveUser` does evict — confirmed in the auth-TTL investigation (item 1 of [`decisions.md`](../decisions.md) 2026-05-14 entry) — so admin-triggered disable propagates quickly. **But this feature's correctness depends on that eviction continuing to fire.** The spec must:

- State the dependency explicitly.
- Flag it in any test plan: a regression on `saveUser`'s `@CacheEvict` for `redisUserAuth` would silently weaken this feature's effect from "immediate" to "up-to-30-min-stale."
- Note for the user-deletion feature: the deletion path must also evict `redisUserAuth` for the deleted user's `firebaseUid` — the same backstop applies.

---

## Source material

- [`investigation-userauth-ttl.md`](../sessions/investigation-userauth-ttl.md) — surfaced the unused `disabled` field and the `checkRevoked` gap during the 2026-05-14 connection-pool auth-TTL trace
- [`2026-05-14-oglasino-backend-userauth-ttl-1.md`](../sessions/2026-05-14-oglasino-backend-userauth-ttl-1.md), [`-2.md`](../sessions/2026-05-14-oglasino-backend-userauth-ttl-2.md), [`-3.md`](../sessions/2026-05-14-oglasino-backend-userauth-ttl-3.md) — the investigation + TTL patch + comment-cleanup arc
- [`decisions.md`](../decisions.md) 2026-05-14 entry — the `redisUserAuth` TTL raise and the constraint it establishes

---

## Open questions for spec time

- HTTP status — `401` vs `403` vs new `ACCOUNT_DISABLED` code on the existing error contract?
- Which decision goes in [`decisions.md`](../decisions.md): the `checkRevoked` trade-off (yes), the disabled-flag enforcement (probably yes since it's a new trust-boundary check)?
- Does the web/mobile rejection-handling reuse the existing `401` interceptor, or warrant a new code path?
- Audit: how many existing admin endpoints already enforce disabled-checking via DB lookup (and should those move to the cache-based check for consistency)?
