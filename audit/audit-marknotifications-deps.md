# Audit — `markNotificationsAsSeen` `useEffect` exhaustive-deps (read-only)

**Repo:** oglasino-web
**Branch:** dev
**Date:** 2026-06-04
**Mode:** READ-ONLY — no code changed, no deps, no build, nothing staged.
**Tracked issue:** the `markNotificationsAsSeen` `useEffect` at
`app/[locale]/(portal)/(protected)/notifications/page.tsx:23` carries a pre-existing
`react-hooks/exhaustive-deps` warning.

**Bottom line:** Recommend **(b) suppress** with
`// eslint-disable-next-line react-hooks/exhaustive-deps` plus a one-line rationale. The
only dep eslint actually wants added is `markNotificationsAsSeen`, an **unstable** function
identity recreated on every render. Adding it re-fires the effect on every render → a
redundant Firestore read each render plus render-thrash risk during the write-settle cycle.
This is the same class of bug as the FilterManager deps-array history. Current firing
(`[user]`) is correct and should be preserved.

---

## 1. The effect, quoted

`app/[locale]/(portal)/(protected)/notifications/page.tsx:19-23`:

```tsx
useEffect(() => {
  if (user) {
    markNotificationsAsSeen();
  }
}, [user]);
```

`user` and `markNotificationsAsSeen` originate at lines 13 and 16-17 of the same file:

```tsx
const { user } = useAuthStore();                       // line 13
...
const { notifications, loadMore, hasMore, markNotificationsAsSeen } =
  useNotificationsStore(router);                       // lines 16-17
```

**What it does / should fire on:** when the authenticated `user` resolves to a truthy value,
mark every unseen notification for that user as seen. Intent is "the user has opened the
notifications page while signed in → clear the unseen state." `markNotificationsAsSeen`
queries the user's `userNotifications` Firestore subcollection for `seen == false`, bulk-writes
`seen: true` on each match, and zeroes the local `unseenCount`. See §4.

---

## 2. The warning, precisely

Actual `npx eslint` output for this file (not inferred):

```
23:6  warning  React Hook useEffect has a missing dependency: 'markNotificationsAsSeen'.
               Either include it or remove the dependency array
               react-hooks/exhaustive-deps

✖ 1 problem (0 errors, 1 warning)
```

- **Currently in the deps array:** `[user]`.
- **Missing dep eslint reports:** `markNotificationsAsSeen` — **one** dep, **one** warning.

**Brief-vs-reality note (matters for the fix):** the brief describes the report as "a missing
dep **plus a `useState`-derived `user` dep**." The live lint shows only the single missing
`markNotificationsAsSeen` dep. `user` is **already in** the deps array and is **not** flagged.
Also, `user` is not `useState`-derived — it comes from the Zustand `useAuthStore`
(`page.tsx:13`), not a `useState` hook. So there is no second warning to resolve, and the
"unstable `user`" concern the brief implies does not apply (see §5). The fix only has to
decide what to do about `markNotificationsAsSeen`.

---

## 3. Firing semantics — the key question

**What firing SHOULD be:** once when `user` resolves (the page is opened by a signed-in user;
or the user signs in while the page is mounted). It is a "clear unseen on view" side effect.
Re-running it after the first success is *idempotent* (see §4) but not *free* — each run is a
Firestore read.

Candidate missing dep, evaluated:

| Candidate dep | Stable identity? | If ADDED, effect re-fires… | Verdict |
|---|---|---|---|
| `markNotificationsAsSeen` | **No** — recreated every render (`useNotifications.ts:110`, plain inline `async` fn, not memoized) | on **every render** of `NotificationsPage` | **(c) harmful** |
| `user` (already present) | Yes between renders; new object only on auth hydration/token refresh | only when auth state changes | correct — leave as-is |

**Why adding `markNotificationsAsSeen` is harmful, step by step:**

1. `useNotificationsStore(router)` runs on every render of `NotificationsPage`;
   `markNotificationsAsSeen` is a fresh closure each time → new reference each render.
2. With it in the deps, the effect re-runs on **every** render (any cause — parent re-render,
   any `useAuthStore` change, the realtime snapshot updating `notifications`).
3. Each re-run issues a `getDocs` query against Firestore (a network read) even though the
   first run already marked everything seen.
4. Worse, there is a feedback path: the first run writes `seen: true`; the page's realtime
   `onSnapshot` subscription (`useNotifications.ts:38-80`) fires on those writes → updates
   `notifications`/`unseenCount` → re-render → new `markNotificationsAsSeen` ref → effect
   re-fires → another `getDocs`. It settles (subsequent queries return `snap.empty` and
   early-return before any write), but it is render-thrash plus redundant reads, exactly the
   unstable-identity-in-deps footgun behind the FilterManager deps bugs in `issues.md`.

So option (a) "just add the dep" is **not** harmless here. It is harmful.

---

## 4. What `markNotificationsAsSeen` actually does on each call

`src/notifications/hooks/useNotifications.ts:110-125`:

```ts
const markNotificationsAsSeen = async () => {
  if (!firebaseUid) return;

  const q = query(
    collection(db, 'notifications', firebaseUid, 'userNotifications'),
    where('seen', '==', false)
  );

  const snap = await getDocs(q);

  if (snap.empty) return;

  await Promise.all(snap.docs.map((docSnap) => updateDoc(docSnap.ref, { seen: true })));

  setUnseenCount(0);
};
```

- **Network/Firestore every call?** A `getDocs` read **every** call. Writes (`updateDoc`)
  **only** when there is at least one unseen doc.
- **Idempotent?** Yes on writes — the `where('seen','==',false)` filter + `if (snap.empty) return`
  guard mean a second call after the first success writes nothing and skips `setUnseenCount`.
  It is **not** read-free: each call still costs one query.
- **Visible effect of re-firing?** None user-visible after the first run (already seen; nothing
  to write; `unseenCount` already 0). The cost of re-firing is purely redundant Firestore reads
  (billable/quota) and render cycles — invisible but wasteful, and unbounded if tied to render
  frequency via the deps array.

---

## 5. The `user` dep specifically

- `user` is `AuthUserDTO | null` held in Zustand store state (`useAuthStore.ts:127`,
  hydrated via `set({ user })`). The page reads it at `page.tsx:13`.
- **Identity stability:** stable *between ordinary re-renders* — its reference changes **only**
  when the store calls `set({ user: ... })`. That happens on: login hydration (null → object),
  logout (object → null, `useAuthStore.ts:263`), and `refreshUser` setting a fresh `backendUser`
  on token rotation (`useAuthStore.ts:286`). It does **not** change on unrelated re-renders.
- **Consequence:** keeping `[user]` makes the effect fire when auth resolves and again on each
  token refresh (a new `backendUser` object). The token-refresh re-fires are infrequent
  (~hourly) and land on the idempotent early-return path (§4) — **harmless-but-redundant**, not
  harmful. This is the current behavior and is acceptable.
- The brief's worry that an unstable `user` would cause repeated firing does **not** hold: `user`
  is not the unstable identity here. `markNotificationsAsSeen` is.

---

## 6. Recommended resolution — **(b) suppress**, with rationale comment

Pick **(b)**. Keep the deps array as `[user]`; silence the single warning with the in-repo
intentional-omission pattern:

```tsx
useEffect(() => {
  if (user) {
    markNotificationsAsSeen();
  }
  // Fire only when auth resolves. markNotificationsAsSeen is recreated every render
  // (unstable identity); adding it would re-fire the effect — and a redundant Firestore
  // read — on every render. It is idempotent, so [user] firing is the correct semantics.
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, [user]);
```

**Why (b) over (a) and (c):**

- **(a) add the dep — rejected.** The only missing dep is the unstable
  `markNotificationsAsSeen`; adding it re-fires every render with redundant reads and
  render-thrash risk (§3). Not "provably harmless" — the bar the brief sets for (a).
- **(c) restructure — rejected as not the clean option.** The natural restructure is to wrap
  `markNotificationsAsSeen` in `useCallback([firebaseUid])` inside `useNotifications.ts`, which
  would stabilize its identity and let it be added safely. But (i) it changes a shared hook for
  a one-line page concern, widening blast radius; (ii) the hook's other returned callbacks
  (`loadMore`, `useNotifications.ts:85`) are **not** memoized, so memoizing only this one
  introduces a parallel pattern inconsistent with the file (Part 4a — match surrounding style);
  (iii) a `firebaseUid` ref-guard on the page is more machinery than the intent needs. The
  effect's intent — "fire when auth resolves" — is already exactly `[user]`.
- **(b) is the minimal, in-pattern fix.** It matches the established repo precedent for
  intentional-omission effects: `LogInDialog.tsx:65`, `RegisterDialog.tsx:77`,
  `LoginOptionsDialog.tsx:41`, `ConsentBanner.tsx:75`, `BackendCachePanel.tsx:73` all suppress
  `react-hooks/exhaustive-deps` with a rationale comment (ConsentBanner's "Mount-only: …" is the
  closest stylistic precedent). Preserves current correct firing; adds zero behavioral risk.

**Exact change shape:** add the `// eslint-disable-next-line react-hooks/exhaustive-deps` line
(with the rationale comment above it) immediately before the `}, [user]);` closing line at
`page.tsx:23`. No deps-array change. No hook change. One- to four-line comment addition only.

### Auth-hydration timing interaction — flagged explicitly

Yes, this effect interacts with auth-hydration timing (the app's known-sensitive area):

- The effect is **correctly gated** on `user` truthiness, and `user` is hydrated by the
  `onIdTokenChanged` listener (UseTokenRefresh), not by this page. So the mark-as-seen is
  correctly deferred until auth resolves — there is no "fired before auth hydrated" hole. The
  `if (user)` guard plus the `if (!firebaseUid) return` guard inside `markNotificationsAsSeen`
  (`useNotifications.ts:111`) double-protect the unauthenticated path.
- `user`'s reference changes on each token refresh (`refreshUser` → `set({ user: backendUser })`),
  so the effect re-fires per token rotation. Idempotent early-return makes this harmless (§5).
- Option (b) does not alter any of this timing — it only silences the lint while preserving the
  current, auth-correct firing. This is the conservative choice the brief asks for.

---

## Adjacent observations (Part 4b — flagged, not fixed; read-only audit)

1. **`const { user } = useAuthStore();` (`page.tsx:13`) subscribes to the whole store, no
   selector.** Every `useAuthStore` state change (e.g. `loading`, `isAdmin*`, `accountBanned`,
   `deletionInFlight`) re-renders `NotificationsPage`, even though it only needs `user`.
   Contrast with `useNotifications.ts:25` which correctly uses a selector
   (`useAuthStore((s) => s.user)`). Severity: **low** (perf only; correctness unaffected). This
   is also what makes the unstable-identity deps trap in §3 more frequently hit. Out of scope —
   not fixed. A selector (`useAuthStore((s) => s.user)`) would be the one-line cleanup if a
   future brief touches this page.

2. **`logout` reads `get().user.id` without a null guard (`useAuthStore.ts:252`).** In the
   `catch` for `detachPushToken`, `get().user.id` will throw if `user` is already null. Not in
   this audit's scope and not in the notifications path; noting it because it was in view while
   tracing `user`. Severity: **low/medium** (narrow timing window). Out of scope — not fixed.

---

## Conventions check

- **Part 4 (cleanliness):** N/A — read-only audit, no code touched. Audit file is the only
  artifact written.
- **Part 4a (simplicity):** the recommendation is the *simpler* of the candidates — suppress
  (zero new abstraction) over restructure (a `useCallback` that would introduce a memoization
  pattern the hook doesn't otherwise use). Documented in §6.
- **Part 4b (adjacent observations):** two flagged above.
- **Part 7 / Part 11 (error contract / trust boundaries):** N/A — no DTO, no wire shape, no
  moderation/authorization decision in this effect.

## Config-file impact

- conventions.md: no change.
- decisions.md: no change.
- state.md: no change.
- issues.md: no change required by this audit. (If Mastermind wants the existing pre-existing-
  lint backlog entry — `issues.md` 2026-05-16, "211 warnings on dev" — annotated with this
  specific resolution, that is a Docs/QA call, not an engineer write. Flagged, not drafted.)
