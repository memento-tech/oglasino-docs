# Audit — oglasino-web production bug sweep (read-only)

**Repo:** oglasino-web · **Branch:** `dev` · **Mode:** READ-ONLY (no code changed) · **Date:** 2026-06-04

Six tracked issues confirmed against the as-built code. Code is ground truth; where the
issue description drifted, the divergence is called out. Nothing was fixed.

Summary table:

| # | Item | Present? |
|---|------|----------|
| 1 | `Input` interpolated Tailwind width class invisible to JIT | **yes** (latent) |
| 2 | `resolveNotificationAction` has no unit test | **yes** |
| 3 | `router: any` parameter type | **yes** |
| 4 | In-app `/notifications` row click doesn't re-add locale | **yes** |
| 5 | SW focus-match never matches (new tab each tap) | **yes** |
| 6 | message-body trim divergence (web sends raw) | **yes** |

Items 3 and 4 are coupled (see Item 3 dependency note). Item 1 has a **sibling** in `Textarea.tsx`
(same bug, not in the brief) — flagged below.

---

## Item 1 — `Input` interpolated Tailwind width class invisible to JIT

**Present?** yes (latent — no current caller is bitten)

**As-built location:** `src/components/server/Input.tsx:62-65`

```tsx
<div
  className={`relative flex w-full flex-col pt-3 ${
    maxWidth > 0 ? `max-w-[${maxWidth}px]` : ''
  }`}>
```

`maxWidth?: number` prop declared at `:12`, defaulted `maxWidth = -1` at `:28`. The class is
built by string interpolation into an arbitrary value (`` max-w-[${maxWidth}px] ``). Tailwind's
JIT scans source text for complete class strings at build time; it cannot see a value assembled
at runtime, so no `max-w-[Npx]` utility is generated and a non-default `maxWidth` produces **no
width class**. The prop appears wired but silently does nothing.

**Caller survey (grep `maxWidth=`):** exactly one caller passes a non-default value —
`app/[locale]/owner/products/[productId]/page.tsx:489` passes `maxWidth={100}`. So this is **not
purely latent — it is actively biting that one call site** (the intended 100px cap is not applied;
the input falls back to `w-full`). Whether that's visually noticeable depends on the container,
but the prop is not doing what the author intended. No other production caller passes `maxWidth`.

**Recommended fix shape:** inline `style={{ maxWidth }}` on the wrapper `<div>` is the cleanest —
it's dynamic-value-friendly and needs no class allowlist. The file is currently **Tailwind-class-only
(no inline `style` anywhere)**, so a static class map (`const W = {100: 'max-w-[100px]', ...}`) would
better match the surrounding style — but a map only works for an enumerated set, and `maxWidth` is a
free `number`. Given the free-number prop, inline `style` is the pragmatic match; if the team prefers
to stay class-only, narrow the prop to a small union and use a static map. **Blast radius:** one
component, one live caller (`page.tsx:489`). Low.

**Adjacent (NOT in brief): `src/components/server/Textarea.tsx:52-54` has the identical bug** —
same `` max-w-[${maxWidth}px] `` interpolation, same `maxWidth?: number` / `= -1` shape. Any fix to
`Input` should be mirrored here or the footgun stays half-open. (No non-default caller found for
`Textarea` in this sweep, so it is latent — but it's the same defect.)

---

## Item 2 — `resolveNotificationAction` has no unit test

**Present?** yes (no spec exists for it)

**As-built location:** `src/notifications/lib/notificationActions.ts` — `resolveNotificationAction`
at `:7`.

**Branch behavior (enumerated):**
- `SAVED_PRODUCT` (`:15-23`): reads `data?.productId`. **Absent → returns `undefined`.** Present →
  returns a closure `() => router.push(getDashboardNormalizedProductUrl(productId))`
  (= `/owner/products/{productId}`, unprefixed — `utils.ts:124-126`).
- `NAVIGATION` (`:25-34`): reads `data?.navigate`. **Absent → returns `undefined`.** Present →
  returns a closure `() => router.push(stripRoutingLocale(navigateTo))`.
- `PRODUCT_EXPIRATION`, `PRODUCT_EXPIRED`, `default` (`:36-39`): return `undefined`.
- `MESSAGE`: intentionally **not** a case (comment `:11-14`: messages are push-tap only, never an
  in-app row) — falls through to `default` → `undefined`.

**Test runner:** vitest. `vitest.config.ts` present at repo root; `package.json:24` `"test":
"vitest run"`. Co-located `*.test.ts` convention confirmed — sibling specs live next to source, e.g.
`src/notifications/service/messageNotificationService.test.ts`, `src/components/server/seo/JsonLd.test.ts`,
`src/lib/images/processImage.test.ts`. **A new spec belongs at
`src/notifications/lib/notificationActions.test.ts`.**

**Purity / testability:** the function is **pure and trivially testable as-is.** It takes
`(notification, router)`, performs no React/router work at resolution time, and either returns
`undefined` or returns a *closure* that calls `router.push(...)` only when invoked. A plain mock
(`{ push: vi.fn() }`) suffices — no router runtime, no rendering.

**Recommended spec scope (do NOT write here — scope only):**
1. `SAVED_PRODUCT` + `data.productId` → returns a fn; invoking it calls `router.push('/owner/products/{id}')`.
2. `SAVED_PRODUCT` with `data` absent **and** with `data.productId` absent → `undefined` (both guard inputs).
3. `NAVIGATION` + `data.navigate` (unprefixed) → returns a fn; invoking calls `router.push(<path>)`.
4. `NAVIGATION` + `data.navigate` **locale-prefixed** → asserts `stripRoutingLocale` ran (pushed path is locale-less).
5. `NAVIGATION` with `data.navigate` absent → `undefined`.
6. `PRODUCT_EXPIRATION` → `undefined`; `PRODUCT_EXPIRED` → `undefined`; unknown category → `undefined`.
7. (optional, documents contract) `MESSAGE` → `undefined`.

---

## Item 3 — `router: any` parameter type

**Present?** yes

**As-built location:** `src/notifications/lib/notificationActions.ts:7` —
`export const resolveNotificationAction = (notification: AppNotification, router: any) => {`.
The `any` also propagates to the helpers that thread the router in: `notificationManager.show`
(`notificationManager.ts:46`, `router: any`) and `useNotificationsStore` (`useNotifications.ts:23`,
`router: any`).

**Callers and the router each actually passes:**
- `notifications/page.tsx:45` — passes the **plain** `next/navigation` `useRouter` (imported `:8`,
  `const router = useRouter()` `:14`). In-app row click.
- `notificationManager.ts:52` — called from `useNotifications.ts:72`
  (`notificationManager.showMany(incoming, router)`), where `router` is whatever was handed to
  `useNotificationsStore(router)`. That hook has **two** mount points:
  - `AuthNotificationButton.tsx:8,16` (the always-mounted header bell) passes the **next-intl-wrapped**
    `useRouter` from `@/src/i18n/navigation`.
  - `notifications/page.tsx:17` passes the **plain** router.

So `resolveNotificationAction` genuinely receives **two different router shapes** today: the wrapped
`WrappedRouter` (via the bell's toast path) and the plain `AppRouterInstance` (via the page). This is
why it is not a clean one-liner.

> Note: the **SW** handler (`public/firebase-messaging-sw.template.js`) does **not** call
> `resolveNotificationAction` at all — it inlines its own switch and `clients.openWindow`. Likewise
> the foreground **native-Notification** handler `ForegroundPushInit.tsx` inlines the switch and uses
> the wrapped router directly (`:4,9,49`). Neither routes through this function, so the brief's framing
> ("the SW handler / foreground push handler pass the router in") is slightly off: the only two callers
> of `resolveNotificationAction` are the page (plain) and the bell-driven toast (wrapped).

**Recommendation / dependency:** there is **no single honest type until the shapes are unified.** The
wrapped router's `push(href, options?)` is a structural superset of the plain one for the call this
function makes (`router.push(string)`), but typing the param as `WrappedRouter` would be a lie while
the page still passes the plain router. **Item 3's clean type is gated on Item 4** (switch the page to
the wrapped router). Once every caller passes the wrapped router, type the param `WrappedRouter`
(exported from `src/i18n/navigation-client.tsx:53`) and drop the three `any`s together.

---

## Item 4 — In-app `/notifications` row click doesn't re-add locale

**Present?** yes

**Plain vs wrapped, quoted:**
- In-app page uses the **plain** router: `notifications/page.tsx:8`
  `import { useRouter } from 'next/navigation';`, `:14` `const router = useRouter();`, passed into
  `resolveNotificationAction(n, router)` at `:45`.
- Foreground native-Notification handler uses the **wrapped** router:
  `ForegroundPushInit.tsx:4` `import { useRouter } from '@/src/i18n/navigation';`, `:9`
  `const router = useRouter();`, `:49` `router.push(url)`.
- SW handler re-adds the locale **manually** (it cannot import the wrapped router):
  `firebase-messaging-sw.template.js:43-53` (`currentLocaleFromClients`) + `:108-110`
  (`const url = locale ? \`/${locale}${...}\` : path`).

**Flow (the exact mismatch):** `resolveNotificationAction` calls `stripRoutingLocale(navigateTo)` for
NAVIGATION (`notificationActions.ts:32`) and `getDashboardNormalizedProductUrl(productId)` for
SAVED_PRODUCT (`:21`, returns the unprefixed `/owner/products/{id}`). Both produce a **locale-less**
path. The wrapped router's `push` then **re-adds** the current locale (`navigation-client.tsx:67-69`
→ `prefixWithLocale`, `:20-26`). The **plain** router does **not** — `next/navigation`'s `push` takes
the path verbatim. So on the in-app page the path is stripped but never re-prefixed; it works only
because the edge (`proxy.ts` + next-intl) resolves a locale from the un-prefixed path. The
comment at `notificationActions.ts:31` ("strip first so the wrapped router does not double-prefix")
confirms the function was written **for** the wrapped router — the page is feeding it the wrong one.

**Recommended fix shape:** change `notifications/page.tsx:8` to
`import { useRouter } from '@/src/i18n/navigation';` (matching `AuthNotificationButton` and
`ForegroundPushInit`). **Drop-in confirmation:** the wrapped `useRouter` returns `WrappedRouter` whose
`push(href: string, options?)` (`navigation-client.tsx:53-54`) is signature-compatible with the page's
only usages — `resolveNotificationAction`'s `router.push(<string>)` and the toast fallback
`router.push('/notifications')` (`notificationManager.ts:58`). No call-site change needed; the page
passes this same router into `useNotificationsStore(router)` (`:17`) which only forwards it. **Blast
radius:** the page's notification rows get correct locale prefixing; the toast fallback path also gets
corrected. Low. **Interaction with Item 3:** this unification makes *both* `resolveNotificationAction`
callers pass `WrappedRouter`, which is the precondition for typing the param (Item 3). Sequence Item 4
before Item 3.

---

## Item 5 — SW focus-match never matches (opens a new tab each tap)

**Present?** yes

**As-built location:** `public/firebase-messaging-sw.template.js` — `notificationclick` handler
`:72-123`; focus-match at **`:112-116`**:

```js
for (const client of clientList) {
  if (client.url === url && 'focus' in client) {
    return client.focus();
  }
}
```

**Why it never matches:** `client.url` is an **absolute** URL — origin + path + (any) query/hash, e.g.
`https://oglasino.com/rs-sr/notifications`. `url` is built **relative** at `:110`
(`const url = locale ? \`/${locale}${path...}\` : path;`) — e.g. `/rs-sr/notifications`. `===` between
an absolute and a relative string is never true, so the focus branch is dead and every background tap
hits `:118-120` `clients.openWindow(url)` → a new window/tab. (Even setting the absolute/relative issue
aside, an exact full-URL `===` would also miss on query/hash differences — pathname comparison is the
robust choice.)

**Recommended fix shape:** normalize both sides before comparing — resolve the target against the SW
origin and compare pathnames, e.g.
`const target = new URL(url, self.location.origin); ... if (new URL(client.url).pathname === target.pathname && 'focus' in client)`.
Optionally `client.navigate(target.href)` before `focus()` if the open window is on a different path.

**Template/placeholder safety:** this **is** a build-time template — `scripts/build-firebase-sw.mjs`
generates `public/firebase-messaging-sw.js` from it (header `:1-5`), substituting the
`__NEXT_PUBLIC_FIREBASE_*__` placeholders at `:11-15`. The focus-match lines (`:106-120`) contain **no
placeholders**, so editing them does not touch substitution. Edit the `.template.js` (not the generated
file). **Testing caveat:** SW changes require a hard refresh / SW update (unregister or
update-on-reload) to take effect — verify the new SW is active before retesting tap-to-focus.

---

## Item 6 — message-body trim divergence (web sends raw)

**Present?** yes

**Emitter:** `src/notifications/service/messageNotificationService.ts:13-18` —
`notifyMessageSent(chatId, messageText)` POSTs `{ chatId, messageText }` to
`/secure/notifications/message-sent`. **Call site (where `messageText` is assembled):**
`src/messages/store/useChatStore.ts:535-536`:

```ts
const textBlock = content.blocks.find((b): b is TextMessageBlock => b.type === 'text');
notifyMessageSent(resolvedChatId, textBlock?.text ?? '').catch(() => {});
```

**Raw vs trimmed:** web sends `textBlock?.text` **raw — no `.trim()`** (confirmed; the only transform
is `?? ''`). For an image-only message there is no text block, so `textBlock` is `undefined` and
`?? ''` sends `''` — the image-only empty-string contract holds (backend B6 frames the localized photo
body). This matches the issue log (2026-06-02 "web/mobile message-body trim divergence"): expo trims,
web doesn't.

**Recommended fix shape:** trim only the ping argument:
`notifyMessageSent(resolvedChatId, textBlock?.text?.trim() ?? '')`. With optional chaining the
image-only path still yields `''` (`undefined?.trim()` → `undefined` → `?? ''` → `''`), preserving the
empty-string contract. **Stored-content safety confirmed:** the Firestore write happens at
`useChatStore.ts:526` `batch.set(messageRef, messageRequest)` / `:528` `batch.commit()` — built from
`content`, committed **before** the ping at `:536`. Trimming the *argument* to `notifyMessageSent`
creates a new string passed only to the ping; it does not mutate `content`/`messageRequest`, so the
stored message is unaffected. The trim affects **only** the push-ping body. Cosmetic, low blast radius
(one line). Verify the `TextMessageBlock` import already in scope at that call site (it is — used in
the type guard on `:535`).

---

## Cross-item sequencing

- **Item 4 → Item 3.** Switch the page to the wrapped router (Item 4) first; that makes both
  `resolveNotificationAction` callers pass `WrappedRouter`, which is the precondition for typing the
  param and removing the three `any`s (Item 3). Doing Item 3 first would force a union or a lie.
- **Item 1 has a twin in `Textarea.tsx`** — fix both or the footgun stays half-open.
- Items 2, 5, 6 are independent and can land in any order.

## Notes / corrections to the issue descriptions

- **Item 1:** the issue log calls it "latent (no current caller relies on it)." Real code shows
  **one live caller** passing `maxWidth={100}` (`products/[productId]/page.tsx:489`), so it is mildly
  *active*, not purely latent.
- **Item 3/4:** the brief/issue framing that "the SW handler and foreground push handler pass the
  router into `resolveNotificationAction`" is **inaccurate** — neither does. The SW and
  `ForegroundPushInit` each inline their own switch. The only two callers of `resolveNotificationAction`
  are the in-app page (plain router) and the bell-driven toast (wrapped router). The locale-resolution
  contrast in Item 4 is still real; the routing of it just differs from the description.
