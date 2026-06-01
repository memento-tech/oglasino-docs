# Session summary

**Repo:** oglasino-backend
**Branch:** dev (Igor's working branch; not switched)
**Date:** 2026-05-28
**Task:** Read-only check — does an authenticated single-user fetch, a follow-status endpoint, or a "my follow IDs" endpoint exist? Plus how `isFollowingCurrent` is populated today and where the follow relation lives.

## The three facts

### 1. Authenticated single-user fetch returning `UserInfoDTO` with `isFollowingCurrent` populated from the viewer

**No dedicated `/secure/user?id=<id>` endpoint exists.**

The single-user-by-id route is `GET /api/public/user?id=<id>` (`controller/UserDataController.java:19-24`, calls `userFacade.getUserData(id)`). It returns `UserInfoDTO`.

However — and this is the load-bearing detail for Fix A — that endpoint **is already auth-aware on the server**. `FirebaseAuthFilter` is registered globally in `security/config/SecurityConfig.java:84` and runs on every path; SecurityConfig only declares `/api/public/**` as `permitAll`, which means **the filter still populates `SecurityContextHolder` when a Bearer token is presented** on a public path. `UserDataController` calls `userFacade.getUserData(id)` which calls `DefaultUserFacade.applyCallerContext(dto)` (`facade/impl/DefaultUserFacade.java:88-98`), and that method reads the current user from `currentUserService.getCurrentUserId().orElse(null)` and sets `dto.setFollowingCurrent(followService.isFollowing(currentUserId, dto.getId()))` when a viewer exists.

**Practical consequence for Fix A:** if the web client simply stops using `skipAuth: true` for `getUserForId`, the existing endpoint will populate `isFollowingCurrent` correctly with no backend change. No `/secure/user?id=<id>` exists, but no new endpoint is strictly required to enable Fix A.

(Tangentially related: `GET /api/auth/firebase/{firebaseId}` at `controller/AuthController.java:119-126` also routes through `userFacade.getUserDataByFirebaseUid` → `applyCallerContext`. It looks up by Firebase UID, not by numeric id, and is the "fetch self" path used by the auth lifecycle. It is not the right shape for "fetch arbitrary user X for viewer V.")

### 2. Follow-status endpoint (`GET /secure/follow/status?userId=<id>` or equivalent)

**No such endpoint exists.**

The only follow-related controller is `controller/FollowController.java`. It exposes exactly one route:

- `POST /api/secure/follow/{targetUserId}` — **toggles** follow state and returns `{ "following": <new boolean> }` post-toggle (`FollowController.java:19-22`).

There is no read-only "is the viewer following this target" GET. Fix B would require adding one (the underlying query already exists — see "where the relation is stored" below — so the endpoint would be a thin wrapper around `followService.isFollowing(viewerId, targetId)`).

### 3. Lightweight "my follow IDs" endpoint

**No such endpoint exists.**

The only "my follows" endpoint is `POST /api/secure/user/follows` (`controller/UserController.java:41-44`), which calls `userFacade.getMyFollowings(paging)` and returns a paginated `UserFollowingsDTO` containing a list of full `UserInfoDTO` objects (`facade/impl/DefaultUserFacade.java:53-68`). That is the wrong shape for Fix C — it's the paginated full-DTO list the brief already discounts.

No endpoint returns just the set of followed user IDs for the authenticated viewer. Fix C would require adding one. The underlying repository would need a new projection (current repo methods at `repository/UserFollowRepository.java` return `User` entities or booleans — no `findFollowingIdsByFollowerId(Long)` exists today).

## Also report

### How `isFollowingCurrent` is populated on `/public/user` today, including the `skipAuth` case

`DefaultUserFacade.applyCallerContext(UserInfoDTO dto)` at `facade/impl/DefaultUserFacade.java:88-98`:

```java
Long currentUserId = currentUserService.getCurrentUserId().orElse(null);
if (currentUserId != null && dto.getId() != null) {
  dto.setIamActive(currentUserId.equals(dto.getId()));
  dto.setFollowingCurrent(followService.isFollowing(currentUserId, dto.getId()));
} else {
  dto.setIamActive(false);
  dto.setFollowingCurrent(false);
}
```

So:

- **With auth forwarded:** `isFollowingCurrent` is computed correctly against the authenticated viewer.
- **With `skipAuth: true` (web's current behavior):** the security context is empty, `currentUserId` is `null`, the else branch fires, and `isFollowingCurrent` is set to a hardcoded **`false`** — the non-authoritative seed the issue is about. The cached `UserInfoDTO` produced by `EntityUserInfoConverter` also defaults the field to `false` (see `converter/EntityUserInfoConverter.java:63-65`); `applyCallerContext` is the post-cache overwriter.

This means the backend already has the right machinery; the bug is entirely on the web side (the `skipAuth: true` flag on `getUserForId`).

### Where the follow relationship is stored and the cheapest "does V follow T" query

**Entity:** `entity/UserFollow.java` — `@Entity` mapped to table `user_follows`. Columns `follower_id` and `following_id` (both `@ManyToOne` to `User`), unique constraint `(follower_id, following_id)`, indexes on each column independently.

**Repository:** `repository/UserFollowRepository.java`.

**Cheapest "does viewer V follow target T" query:** `userFollowRepository.existsByFollowerIdAndFollowingId(viewerId, targetId)` (line 15 of the repository) — a Spring Data derived `EXISTS` that hits the unique constraint and the `idx_user_follows_follower` index. Sub-millisecond. This is the exact call `FollowService.isFollowing` already uses (`service/impl/DefaultFollowService.java:20-22`).

For Fix C's "give me my follow IDs," there is no existing query — would need a new projection-style method, e.g.

```java
@Query("SELECT uf.following.id FROM UserFollow uf WHERE uf.follower.id = :userId")
List<Long> findFollowingIdsByFollowerId(@Param("userId") Long userId);
```

The data shape supports it cleanly (single-column scan against `idx_user_follows_follower`); no schema change required.

## Decision-enabling summary for Mastermind

| Fix | Backend dependency status |
|---|---|
| **Fix A** — split web's `getUserForId` so authenticated callers don't use `skipAuth` | **No backend change required.** `/api/public/user?id=<id>` already populates `isFollowingCurrent` correctly when a Bearer token is forwarded. |
| **Fix B** — `GET /secure/follow/status?userId=<id>` returning `{ following: boolean }` | **New endpoint needed.** Trivial implementation: thin wrapper around the existing `followService.isFollowing(viewerId, targetId)` (which itself wraps the existing `existsByFollowerIdAndFollowingId` index lookup). |
| **Fix C** — lightweight "my follow IDs" endpoint | **New endpoint + new repository method needed.** No existing query returns just the ID set. New `@Query` projection is ~3 lines; new endpoint wraps it. |

## Files touched

- None (read-only session).

## Tests

- None run (read-only check, no code changes).

## Cleanup performed

- None needed.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change. The existing `2026-05-17` `UserInfoDTO.isFollowingCurrent` entry is the home of this work; Mastermind decides whether the Fix A/B/C selection it now has the facts to make warrants an update to that entry.

## Obsoleted by this session

- Nothing.

## Conventions check

- Part 4 (cleanliness): N/A — no code changed.
- Part 4a (simplicity): N/A — no code changed.
- Part 4b (adjacent observations): nothing surfaced that wasn't already in scope of the brief.
- Part 6 (translations): N/A this session.
- Part 11 (trust boundaries): confirmed indirectly — the existing `applyCallerContext` machinery reads identity from `SecurityContextHolder` (the trust boundary per Part 11), not from request payload, so Fix A's "drop `skipAuth`" path keeps the trust boundary intact. No client-supplied "is following" value is or would be honoured.

## Known gaps / TODOs

- None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing.
  - Considered and rejected: nothing — read-only fact-finding.
  - Simplified or removed: nothing.

- **Brief vs reality:** nothing flagged. The brief asked three yes/no questions plus two factual reports; the code answered all five cleanly with no contradictions to push back on. The one nuance worth highlighting (and which materially affects the Fix A/B/C choice): `/api/public/user?id=<id>` is **already auth-aware on the server** — the `skipAuth` problem is entirely on the web side, not a backend gap. This may compress Fix A to "remove `skipAuth: true` from the web call site" with zero backend work, depending on how the web audit framed it.

- **Suggested next step:** route this fact set to Mastermind to pick A vs B vs C. If the answer is A, the next brief is web-side and this repo is done. If B or C, a small backend brief follows; the data shape (table, index, repo method) is all in place — both are <30 lines plus tests.

- No drafted config-file text.
