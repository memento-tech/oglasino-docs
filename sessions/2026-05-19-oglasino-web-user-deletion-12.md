# Session summary

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-05-19
**Task:** Web read-only audit: `/api/revalidate` route handler contract — verify header name, body shape, auth model, env-var name against the backend's recently-landed `DefaultWebRevalidationService` call.

## Implemented

- Read-only audit. No implementation.
- Located the route at `app/api/revalidate/route.ts` (Next.js 15 App Router convention, as the brief predicted) and confirmed the env-var template in `.env.local.example`.

## Files touched

- None.

## Tests

- None run.

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

- Part 4 (cleanliness): N/A — read-only audit, no production code touched.
- Part 4a (simplicity): see structured evidence in "For Mastermind" (all three categories: nothing).
- Part 4b (adjacent observations): no flag-worthy items beyond the divergences captured in "For Mastermind" (which are the audit's primary deliverable, not adjacent).
- Part 6 (translations): N/A.
- Other parts touched: none.

## Known gaps / TODOs

- None.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing — read-only audit.
  - Considered and rejected: nothing — read-only audit.
  - Simplified or removed: nothing — read-only audit.

### Deliverable — four answers

1. **Header name.** The web side validates `X-Revalidate-Secret` (lookup at `app/api/revalidate/route.ts:58`, lowercased per the Fetch API contract: `req.headers.get('x-revalidate-secret')`).
2. **Body shape.** The route accepts a union: `{ tag?: string }`, `{ tags?: string[] }`, `{ path?: string }`, `{ paths?: string[] }` (type at `route.ts:44-49`, merge logic at `route.ts:70-71`). Tag and path forms can be mixed in the same request. The backend's `{"tags": ["user:<id>"]}` payload is one of the supported shapes — exact match.
3. **Auth model.** Option A — shared secret header. Fail-closed: returns 503 `{error: "not_configured"}` if `REVALIDATE_SECRET` is unset server-side (`route.ts:53-56`); returns 401 `{error: "unauthorized"}` if the provided header value does not match (`route.ts:59-61`). GET is explicitly blocked with 405 (`route.ts:92-94`). No IP allowlist, no next-auth session check, no other auth gate.
4. **Env var name.** `REVALIDATE_SECRET` on the web side (read at `route.ts:52`; declared in the template at `.env.local.example:63` under the section header "Cache eviction (server-only, no NEXT_PUBLIC_ prefix)" at lines 56-62, with a `openssl rand -base64 32` generation hint).

### Divergences from the backend's current implementation

Two of the four answers diverge from what the brief said the backend currently uses; two match.

1. **Header name divergence.** Web expects `X-Revalidate-Secret`; backend brief says `DefaultWebRevalidationService.SECRET_HEADER` is currently `X-Revalidation-Secret`. The backend's header name has the extra suffix `-ion` (`Revalidat**ion**-Secret` vs `Revalidat**e**-Secret`). With the backend's current value, every backend-originated cache-invalidation request will land in the route's "header missing" branch (`provided !== expected` because `provided === null`) and return 401. The web side never invokes `revalidateTag` for the user-state-change tag — the cache stays stale until its 5-minute TTL expires.

   Proposed fix: one-line constant change in `DefaultWebRevalidationService` in `oglasino-backend`: rename `SECRET_HEADER` value from `X-Revalidation-Secret` to `X-Revalidate-Secret`. (Web could change instead, but the web side was first — committed 2026-05-06 per `jobs/rn_sync/FRONTEND-CHANGES-FOR-RN.md:75` — and the route already documents its name in published comments and the RN-sync doc, so backend conforming is the cheaper move.)

2. **Env-var-name divergence.** Web uses `REVALIDATE_SECRET`; backend uses `WEB_REVALIDATE_SHARED_SECRET`. The values must agree for the secret to match, so operators provisioning prod and stage today would need to set the same string under two different env-var names on two different platforms (Vercel and DigitalOcean). The brief explicitly flags this as something it wants aligned ("operators only provision one value").

   Proposed fix: change one side's env-var name. By the same precedence argument (web was first, web's name is already documented in `.env.local.example` and the RN-sync doc), the cleaner move is for the backend to rename `WEB_REVALIDATE_SHARED_SECRET` → `REVALIDATE_SECRET`. Mastermind's call — the alternative (web renames) is symmetric and equally one-line; the web rename would require updating `.env.local.example:63`, `route.ts:52`, `route.ts:54`, and the comment block at `route.ts:17`, plus the Vercel env-var name in any operator-facing runbook.

### Matches (no action)

- **Body shape.** Backend POSTs `{"tags": ["user:<id>"]}`. Web accepts this exact shape via the `tags?: string[]` field. The tag `user:{id}` is explicitly documented in the route's header comment as an available tag (`route.ts:32`), meaning the web side anticipated this use case when the endpoint was built.
- **Auth model.** Both sides agreed on Option A (shared secret).

### Adjacent observations (out of scope)

These are not divergences from the brief, but worth surfacing while I'm in the file. None are bug-grade — flagging at low severity for Mastermind's awareness:

- The route already runs in Next.js 16 cache-life-profile mode (`revalidateTag(tag, 'default')` at `route.ts:82`, with the comment block at `route.ts:80-81` explaining why). This is what the brief's out-of-scope note about "round-3 cache-invalidation fix" was pointing at. No action — already addressed.
- The `.env.local.example` for `REVALIDATE_SECRET` is empty by default (line 63 is `REVALIDATE_SECRET=`). Combined with the route's fail-closed `if (!expected)` check, this gives dev environments a clean local-disabled posture: backend's empty `WEB_REVALIDATE_SHARED_SECRET` won't even attempt the call (per the brief's Group 2 implementation note), and even if it did, the web side would return 503 rather than attempting an unauthenticated revalidate. The two empty-string defaults compose correctly for dev — confirming the brief's "Dev can stay empty" pre-launch reminder is correct.
- The route comment block at `route.ts:34-42` shows two example `curl` invocations that would be useful to keep in sync once the names are aligned. After Mastermind picks a side, whoever lands the rename should update the comment block too.

### Operator action items (relayed from brief — not engineer-actioned, just acknowledged)

These come from the brief's pre-launch reminders section. No engineer action this session; logging here so Igor sees them alongside the audit results:

- `WEB_REVALIDATE_URL` to be provisioned on the backend side (DO droplet env or backend deployment env) for prod and stage.
- `WEB_REVALIDATE_SHARED_SECRET` / `REVALIDATE_SECRET` (name TBD after Mastermind picks resolution above) to be provisioned with the **same value** on both Vercel (web) and DO (backend).
- Both names to be added to `oglasino-docs/infra/overview/secret-inventory.md` per conventions Part 1 sensitive-data rule.
- Dev stays empty per Group 2's clean-disable design.

### Summary

Two trivial one-line fixes resolve the audit findings. The web side is internally consistent and already documented; backend conforming to the web's existing names is the lower-cost move on both counts. The endpoint exists, accepts the right body shape, runs the right auth model, and is wired correctly to Next.js 16's tag-invalidation API — only the header name and env-var name need alignment.
