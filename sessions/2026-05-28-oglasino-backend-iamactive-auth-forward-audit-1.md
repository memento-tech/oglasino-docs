# Session summary

**Repo:** oglasino-backend
**Branch:** dev
**Date:** 2026-05-28
**Task:** View-count increment — backend iamActive audit (READ-ONLY): does the #8 web fix (auth now forwarded on `getProductDetails` portal branch and `getUserForId`) change what the backend returns for `iamActive` on the embedded product owner?

## Implemented

- Read-only audit. No code changes, no temp logs added, no `psql`. Output is the answers below + a yes/no verdict on hypothesis (c).

## Files touched

- None (read-only).

## Tests

- Ran: `./mvnw spotless:check` — BUILD SUCCESS (622 files clean, 0 needs changes).
- Ran: `./mvnw test` — 663 passed, 0 failed, 0 errored, 0 skipped. BUILD SUCCESS.

---

# Answers

## Q1 — How is `iamActive` computed? Quote the computation.

**`iamActive` is set by `DefaultUserFacade.applyCallerContext(UserInfoDTO)` at `src/main/java/com/memento/tech/oglasino/facade/impl/DefaultUserFacade.java:88-98`.** Verbatim:

```java
private UserInfoDTO applyCallerContext(UserInfoDTO dto) {
  Long currentUserId = currentUserService.getCurrentUserId().orElse(null);
  if (currentUserId != null && dto.getId() != null) {
    dto.setIamActive(currentUserId.equals(dto.getId()));
    dto.setFollowingCurrent(followService.isFollowing(currentUserId, dto.getId()));
  } else {
    dto.setIamActive(false);
    dto.setFollowingCurrent(false);
  }
  return dto;
}
```

- `currentUserId` comes from `DefaultCurrentUserService.getCurrentUserId()` (`security/service/impl/DefaultCurrentUserService.java:32-38`), which reads the `OglasinoAuthentication` placed in `SecurityContextHolder` by `FirebaseAuthFilter`. No auth in context → `Optional.empty()` → `null` here.
- `dto.getId()` is the **queried user's** id — i.e. the owner whose record was loaded. For `/api/public/user?id=<ownerId>` this is exactly `<ownerId>`.
- **Meaning:** `iamActive` is true iff the authenticated viewer **is the same user as** the DTO being returned. It is **not** an "owner is active / not banned" signal. The DTO javadoc spells this out at `dto/UserInfoDTO.java:6-25` ("true iff the queried user equals the authenticated caller"), and the facade docstring at `facade/impl/DefaultUserFacade.java:80-87` repeats it: the field is caller-context-dependent and is set AFTER the cache lookup so the cached entry never carries one viewer's verdict to another.

The cache (`redisUserInfo`, populated by `EntityUserInfoConverter` at `converter/EntityUserInfoConverter.java:37-72`) deliberately leaves `iamActive` at the field default `false` (`dto/UserInfoDTO.java:36` — `private boolean iamActive;`, no initialiser). Class javadoc at lines 9-19 and converter comment at line 64 both confirm this is intentional.

### Where does `iamActive` actually ship from?

Two endpoints return `UserInfoDTO` through the facade:

1. **`GET /api/public/user?id=<id>`** — `UserDataController.getUser` (`controller/UserDataController.java:19-24`) → `UserFacade.getUserData(id)` → `applyCallerContext`. **This is the endpoint `getUserForId` calls in the product page flow.**
2. **`GET /api/auth/userForFirebaseId/...`** — `AuthController.getUserForFirebaseId` (`controller/AuthController.java:120+`). Not the product-page flow.

The **product details portal payload** (`GET /api/public/product/search?productId=<id>` → `ProductSearchController.getProductDetails` at `controller/ProductSearchController.java:38-49` → `DefaultProductsSearchFacade.getProductDetailsForId` → `ProductDetailsDTO`) does **not** carry an embedded owner `UserInfoDTO`. `ProductDetailsDTO` (`dto/ProductDetailsDTO.java`) extends `ProductOverviewDTO` (`dto/ProductOverviewDTO.java`) which has `private Long ownerId;` and nothing else owner-shaped. No `iamActive` field anywhere on the product-details payload. **So `iamActive` on the "embedded owner" in the web product page flow comes exclusively from the separate `/public/user?id=<ownerId>` call** — there is only one iamActive source to audit for this defect.

---

## Q2 — Effect of auth now being forwarded.

Before #8 the web's `getUserForId` axios call carried `skipAuth: true` and `FirebaseAuthFilter` saw no token → `SecurityContextHolder` empty → `currentUserId == null` → **always the `else` branch → `iamActive = false`**, regardless of who the caller actually was.

After #8 the call forwards the viewer's Firebase ID token. `FirebaseAuthFilter` decodes it, loads `authData` from `redisUserAuth`, and places an `OglasinoAuthentication` carrying the viewer's `userId` in `SecurityContextHolder` (`security/filter/FirebaseAuthFilter.java:71-114`). Now `currentUserService.getCurrentUserId()` returns `Optional.of(viewerUserId)` and the `if` branch evaluates `viewerUserId.equals(ownerId)`.

### Per-viewer-case table

For a request to `/api/public/user?id=<ownerId>` where the DTO returned is the **owner** of the product page being viewed:

| Viewer case | Before #8 (`skipAuth`) | After #8 (auth forwarded) | Changed? |
| --- | --- | --- | --- |
| Logged-out / anonymous (no Bearer) | `currentUserId == null` → else branch → **`iamActive = false`** | No token forwarded by web for logged-out viewers either; `FirebaseAuthFilter` sees no token → `SecurityContextHolder` empty → else branch → **`iamActive = false`** | **No change.** |
| Logged-in non-owner | `currentUserId == null` → else → **`iamActive = false`** | `currentUserId == viewerId`, `dto.getId() == ownerId`, `viewerId != ownerId` → if-branch evaluates `false` → **`iamActive = false`** | **No change.** |
| Logged-in owner viewing own product | `currentUserId == null` → else → **`iamActive = false`** | `currentUserId == ownerId == dto.getId()` → if-branch evaluates `true` → **`iamActive = true`** | **Yes — flipped from `false` to `true`.** |

Edge cases worth recording for completeness:

- **PENDING_DELETION viewer** (`FirebaseAuthFilter.java:93-97`): filter clears the context and continues anonymously. Same as logged-out → `iamActive = false`.
- **Banned/disabled viewer** (`FirebaseAuthFilter.java:83-86`): filter short-circuits with 403 USER_BANNED before the controller is reached. No DTO returned.
- **Invalid token on `/api/public/**`** (`FirebaseAuthFilter.java:115-129`): `SecurityContextHolder.clearContext()`, suppression branch fires, request proceeds anonymously → `iamActive = false`.
- The third-row "owner viewing own product" case is **the only viewer case whose `iamActive` value moved** as a result of #8 dropping `skipAuth`.

---

## Q3 — Is `iamActive` semantically a view-count gate?

**No.** `iamActive` is a "viewer-equals-queried-user" signal. The DTO javadoc (`dto/UserInfoDTO.java:6-25`) and the facade docstring (`facade/impl/DefaultUserFacade.java:80-87`) both name it that way. Its only on-server consumer is the cache-correctness contract (the field is intentionally null-zero in the cached representation so cache hits don't leak one caller's verdict to another).

The web is overloading it as "viewer is the owner of this product, so don't count their view." That overload happens to be correct **for the owner-on-own-product case** — `iamActive` being true on the owner DTO and the owner viewing their own product page are coextensive there (the web fetches `/public/user?id=<product.ownerId>`, so `dto.getId() == ownerId` always, and `iamActive == true` exactly when `viewerId == ownerId`). It is the only case it covers, however. `iamActive` cannot represent "owner is banned/deleted/inactive" or any other property of the owner — that information lives in `UserInfoDTO.state` (an enum field, see `UserInfoDTO.java:42-50`), which is unrelated to `iamActive`.

If Mastermind wants the gate-the-self-view behaviour to survive without depending on `iamActive`, the cleanest framing per the prior backend audit (Q3 in `sessions/2026-05-28-oglasino-backend-view-counter-check-1.md`) is "drop the web gate; let the server's owner-exclusion at `DefaultProductSeenService.java:49` decide." That decision is out of scope for this audit.

---

## Verdict on hypothesis (c)

**No.** The backend does **not** return `iamActive = true` for the normal viewing population on the product-details payload.

- For **anonymous viewers** (the bulk of traffic on a public product page) the value is `false` both before and after #8. The web's gate `if (!owner.iamActive) markAsSeen(...)` fires; `markAsSeen` is correctly called.
- For **logged-in non-owners** (the rest of the bulk) the value is also `false` after #8. Before #8 it was likewise `false` (because the call was `skipAuth`). The gate still fires; `markAsSeen` is still called.
- The **only** case where `iamActive` flipped is the **logged-in owner viewing their own product** — a rare case, and the one case where suppressing `markAsSeen` is the intended behaviour (and is independently enforced server-side at `DefaultProductSeenService.java:49`).

Therefore hypothesis (c) does not explain why `/seen` is not reaching the backend for the normal viewing population. The auth-forwarding change introduced by #8 did not move `iamActive` for that population, and the web gate remains in the same state for them.

Hypotheses (a) `after()` callback not executing and (b) the fetch inside `after()` being dropped silently are the surviving candidates — both web-side, beyond this audit's scope. The recommended next step in `issues.md` (read-only diagnostic-instrumentation pass) is the right move; this audit eliminates (c) from the list.

---

## Cleanup performed

- None needed (read-only audit, zero file edits, no temp logs added).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change
- issues.md: no change required from this audit. The hypothesis-(c) elimination is a finding for Mastermind to fold into the next step on the 2026-05-28 view-count entry; the entry already lists hypotheses (a)/(b)/(c)/(d) in priority order, so Docs/QA could optionally strike (c) when Mastermind says so. Drafted text in "For Mastermind" below; not authored here.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A this session (read-only — no code touched, no logs added).
- Part 4a (simplicity) / Part 4b (adjacent observations): see "For Mastermind".
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): confirmed — `iamActive` is derived server-side from `SecurityContextHolder` (via `CurrentUserService`) compared to the DTO's owner id (read from the DB / cache, not a client-supplied claim). No client-supplied "I am the owner" signal is trusted. The #8 auth-forwarding change strengthens — not weakens — the trust boundary for this field, because previously `iamActive` was always `false` regardless of who called, which masked the real signal.

## Known gaps / TODOs

- None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only).
  - Considered and rejected: nothing (read-only).
  - Simplified or removed: nothing (read-only).

- **Verdict, copy-pasteable for the bug chat / next brief:**
  > Hypothesis (c) is ruled out by the backend audit (`oglasino-backend-iamactive-auth-forward-audit-1`, 2026-05-28). `iamActive` is `currentUserId.equals(dto.getId())` (false for anonymous viewers and logged-in non-owners; true only when the logged-in viewer **is** the owner of the queried user record). The #8 auth-forwarding change only flips the value for the logged-in owner on their own product page — the very case the gate is intended to skip. For the normal viewing population (anonymous + logged-in non-owners) `iamActive` remains `false`; the web gate `if (!owner.iamActive)` still fires; `markAsSeen` should still be called. The break is upstream of the gate. Surviving hypotheses to pursue: (a) `after()` callback not executing, (b) fetch inside `after()` being dropped silently, (d) URL/base-URL specific to this call.

- **Drafted update for `issues.md` (Docs/QA to apply if Mastermind agrees):** In the 2026-05-28 "Product view count always 0 across the portal" entry, the "Open hypotheses" list could mark hypothesis (c) as eliminated by this audit and bump (a)/(b) to first-priority. Suggested edit, inline beneath the existing (c) bullet:
  > **(c) eliminated.** Read-only backend audit `oglasino-backend-iamactive-auth-forward-audit-1` (2026-05-28) confirms `iamActive` is `currentUserId.equals(dto.getId())`, so for anonymous viewers and logged-in non-owners (the normal viewing population) the value remained `false` after #8 — the only viewer case whose value moved is the logged-in owner on their own product, which is the case the gate intentionally skips. The break is upstream of the gate; pursue (a) and (b) first.
  >
  > Recommended-first-step paragraph stays as-is; the diagnostic-instrumentation pass targets (a) and (b) directly.

- **Adjacent observation (Part 4b — low severity, no action requested).** `applyCallerContext` performs a `followService.isFollowing(currentUserId, dto.getId())` call on every authenticated request that returns a `UserInfoDTO` (`facade/impl/DefaultUserFacade.java:92`). On product pages that means one DB/Redis follow-lookup per page view for logged-in viewers — paid even when the consumer of `isFollowingCurrent` does nothing with it (e.g. when the viewer's own bookkeeping doesn't need the field). Not relevant to the view-counter defect; flagging because the auth-forwarding change in #8 turned this into a real per-request cost where previously it was a no-op (`currentUserId == null` short-circuited the else branch). File: `src/main/java/com/memento/tech/oglasino/facade/impl/DefaultUserFacade.java:88-98`. Severity low — `followService.isFollowing` is a single Redis/DB hit, well within the per-page budget.

- (or: nothing else flagged.)
