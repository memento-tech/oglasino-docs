# Trust-boundary audit — user deletion (web half, §16 completion)

**Date:** 2026-05-18
**Branch:** feature/user-deletion
**Repo:** oglasino-web
**Counterpart:** spec §16.1–§16.7 (backend half) — this document closes §16 from the frontend side.

Per conventions Part 11: every value used in a moderation, authorization, or state-transition decision must be derived from the authenticated identity (`SecurityContextHolder` / verified Firebase ID token), read from server data, or otherwise unforgeable by the client. Client-supplied "before" / "previous" values are never trusted.

The seven user-facing surfaces this feature added are audited below.

---

## 1. Danger Zone delete-account submission

**Surface.** `DeleteAccountConfirmationDialog.tsx` → reauth → forced `getIdToken(true)` → `POST /api/secure/user/me/delete` with empty `{}` body and explicit `Authorization: Bearer <freshToken>` header.

**Values that gate the state transition.**

| Value                       | Source                                                                                   | Trust verdict       |
| --------------------------- | ---------------------------------------------------------------------------------------- | ------------------- |
| User identity (target)      | Backend reads `OglasinoAuthentication.userId` from verified Firebase ID token            | ✓ Server-derived    |
| Reauth freshness            | Backend reads `auth_time` claim from cryptographically verified ID token                 | ✓ Server-derived    |
| Lock status                 | Backend reads `user_deletion_locks` table                                                | ✓ Server-only data  |
| Current deletion status     | Backend reads `user.deletion_status` column                                              | ✓ Server-only data  |

**Frontend cannot lie about identity.** Request body is `{}` — there is no `userId` field. The backend ignores anything in the body for identity purposes (§10.2). Even if a malicious client sends `{userId: 999}`, the controller derives identity from the auth context.

**Frontend cannot fake a fresh `auth_time`.** The fresh token is minted by Firebase after `reauthenticateWithCredential` / `reauthenticateWithPopup`. The client can pass any string in `Authorization`, but the backend's `FirebaseAuthFilter` verifies the token cryptographically — a forged or replayed token fails verification entirely.

**Verdict.** ✓ Clean.

---

## 2. Post-deletion sessionStorage trigger

**Surface.** `AccountStateDialogsInit.tsx` reads `sessionStorage.getItem('account-just-deleted')` on root-layout mount and opens the post-deletion dialog with the stored timestamp interpolated as the deletion date.

**Threat.** Can a client write `account-just-deleted` to their own sessionStorage to trigger the dialog without actually deleting?

**Analysis.** Yes, technically. SessionStorage is client-controlled. A user who runs `sessionStorage.setItem('account-just-deleted', '2026-12-31T00:00:00Z')` in DevTools and then reloads the page will see the dialog.

**Does any state transition follow?** No. The dialog is informational only. Reading the key:
- Does not call any backend endpoint.
- Does not flip any frontend state that gates authorization, authentication, or moderation.
- Does not affect the user's session — they remain signed in (assuming they were before).

The deletion was already committed server-side before this key was set (the deletion-confirmation dialog sets it only after a 200 response from `/api/secure/user/me/delete`). The dialog is a confirmation surface, not a state gate.

**Verdict.** ✓ Clean. The sessionStorage value is "what date should I render in the informational dialog" — not a state transition input.

---

## 3. Restoration `X-Account-Restored` response header

**Surface.** `src/lib/config/api.ts` response interceptor reads `response.headers['x-account-restored']` on every successful response and calls `useAuthStore.setRestored(true)` when it equals `'true'`. `AccountStateDialogsInit.tsx` watches the flag and opens the restoration dialog.

**Threat.** Can a client fake this header to trigger the restoration dialog?

**Analysis.** Clients send *request* headers; servers send *response* headers. The interceptor reads the response object from `axios.post(...)` — that object is populated from the network response, which comes from the backend. There is no client-side write path to `response.headers`. (A client *can* shim `axios` itself in DevTools, but at that point they're already in their own browser and changing what their own copy of the code does — that's not a trust-boundary issue; nothing crosses to the server.)

The header is set by the backend's `FirebaseAuthFilter` (§10) only when the user's `deletionStatus = 'PENDING_DELETION'` and the filter invokes `cancelDeletionOnLogin`. That decision is made server-side from server data.

**Does the dialog gate any state transition?** No. The auth-store `restored` flag is a UI-only flag — it controls whether the restoration informational dialog opens, then clears. It is not consulted by `SessionGuard`, `useAuthResolved`, or any authorization path.

**Verdict.** ✓ Clean.

---

## 4. Ban-notice sessionStorage trigger

**Surface.** `AccountStateDialogsInit.tsx` reads `sessionStorage.getItem('account-banned')` on root-layout mount and opens the ban-notice dialog.

**Threat.** Same shape as #2 — can a client self-set the key?

**Analysis.** Yes, technically. A user setting `sessionStorage.setItem('account-banned', '1')` and reloading would see the dialog.

**Does any state transition follow?** No. The dialog renders static translated content (the four `BANNED_DIALOG` keys). No backend call, no auth-state change, no authorization gate. If a user sets the key but is not actually banned, they'll see the dialog, dismiss it, and continue using the app normally — because the keys that *actually* enforce a ban (the backend's `User.disabled` column, `banned_user_audit` hashes, Firebase `setDisabled`) are server-only and the auth filter rejects the user's requests regardless of what their sessionStorage says.

**Verdict.** ✓ Clean. The key is a UI hint, not an authorization signal.

---

## 5. Global 403 + `USER_BANNED` interceptor

**Surface.** `src/lib/config/api.ts` error interceptor: on `error.response?.status === 403` and `errors[0].code === 'USER_BANNED'`, signs the user out and sets `sessionStorage.setItem('account-banned', '1')`.

**Threat.** Can a client self-induce the sign-out?

**Analysis.** The signout is triggered by a *server response* (status 403 with a specific error code body). A client cannot make their own browser receive a 403 from the backend without the backend actually returning one. They could mock `axios` locally, but again that's the user changing their own code — no security boundary crossed.

The backend returns 403 + `USER_BANNED` only when `authData.disabled == true` (§10) or when `firebase-sync` matches a banned email hash (§10.1). Both gates are server-only.

**Note on the `auth.signOut()` consequence.** The signout is a Firebase client-SDK call; the user's local session ends. The backend already considers them banned (that's why the 403 happened in the first place), so this is a UX reconciliation, not a trust transition.

**Verdict.** ✓ Clean.

---

## 6. Profile-page badge rendering (`ScheduledForDeletionBadge`)

**Surface.** `UserDetails.tsx` renders `<ScheduledForDeletionBadge />` when `userDetails.state === 'PENDING_DELETION'`. The chat header in `Messages.tsx` does the same when `activeChat.withUser.state === 'PENDING_DELETION'`.

**Threat.** Can the client lie about another user's deletion state?

**Analysis.** No. `userDetails: UserInfoDTO` is fetched server-side via `getUserForId(userId)` → backend `/public/user/{id}`. The `state` field is populated from `User.deletion_status` server-side. The client only renders what it receives.

Could a malicious client modify their own copy of the rendered DOM to hide the badge? Yes, locally — but that affects only their own view. The backend's behavior (hiding listings, blocking messaging to/from the user) is enforced server-side regardless of what the local client renders.

**Verdict.** ✓ Clean.

---

## 7. `CallUserButton` gate

**Surface.** `ProductFunctions.tsx` (Phase 2 of this brief) passes `callingAllowed={(owner.allowPhoneCalling || false) && owner.state === 'ACTIVE'}` to `<CallUserButton>`. The button renders in its disabled state when `callingAllowed` is false.

**Threat.** Can the client bypass the gate to fetch the phone number anyway?

**Analysis.** Both flags (`allowPhoneCalling`, `state`) come from the server-fetched `UserInfoDTO`. A client that rendered the button despite a falsy `callingAllowed` and clicked it would call `getUserPhoneNumber(userId)` → backend `/secure/user/phoneNumber?userId=...`.

Per spec §14.7: "The endpoint should now also reject `PENDING_DELETION` users with a generic null response or 403. The frontend's `CallUserButton` is also gated client-side... (frontend gate is UX-only; backend check is load-bearing)."

So:
- Frontend gate: UX only. Hides the call affordance from honest users.
- Backend gate: load-bearing. Rejects the request server-side regardless of what the client tries.

A client that fully bypasses the frontend gate and hits the endpoint directly gets the same `null` / 403 the backend would return for a banned or deleting user.

**Verdict.** ✓ Clean.

---

## Summary

Seven surfaces audited. Zero CRITICAL findings.

Two informational-only dialog triggers (#2 post-deletion sessionStorage, #4 ban-notice sessionStorage) are technically client-writable, but writing them only opens an informational dialog in the writer's own browser — no state transition follows, no other user is affected, no backend gate yields to the local state.

The two response-driven surfaces (#3 X-Account-Restored header, #5 USER_BANNED interceptor) are server-derived signals that the client cannot self-induce.

The three rendering surfaces (#6 badge, #7 button gate, plus the chat header from Brief A) are gated on server-supplied fields; the client cannot lie about another user's state, and the backend re-checks every authorization-relevant call independently.

The deletion submission (#1) carries no client-trusted identity field — the backend reads the user from the verified Firebase ID token's claims, and the reauth freshness check reads the cryptographically signed `auth_time` claim.

**Verdict — web half of §16 is clean. Feature was designed against Part 11 from the start.**
