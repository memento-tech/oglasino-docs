# Audit — oglasino-expo — contact/support feature

**Repo:** oglasino-expo
**Branch:** dev
**Date:** 2026-06-11
**Mode:** read-only (Phase 2 audit). No code changes.
**Method:** every citation below confirmed with both `view` (Read) and `rg`.

Scope: the five questions in `.agent/brief.md`. The aim is to name the cheapest existing template a Contact screen can be built on, and to surface any cross-repo seam (notably translations) before engineering starts.

---

## 1. Simple content screen pattern

**Closest template: the privacy / terms `MarkdownViewer` screens.** They are the leanest static-content screens in the app and a Contact screen should be modeled on them.

### Screen shell

`app/(portal)/(public)/privacy.tsx:8-18` and `app/(portal)/(public)/terms.tsx:8-18` are near-identical. Shape:

```tsx
export default function PrivacyScreen() {
  const lang = useBootStore((s) => s.language)?.code;          // privacy.tsx:9
  return (
    <ScrollView className="flex-1">                            // privacy.tsx:12
      <BackToHomeButton />                                     // privacy.tsx:13
      <MarkdownViewer url={legalDocUrl('privacy', lang)} />    // privacy.tsx:14
      <Footer />                                               // privacy.tsx:15
    </ScrollView>
  );
}
```

- **Scroll wrapper:** a plain `ScrollView className="flex-1"` from `react-native` (`privacy.tsx:6,12`). No explicit `SafeAreaView` at the screen level (see safe-area note below).
- **Back affordance:** `<BackToHomeButton />` (`src/components/BackToHomeButton.tsx:9-21`) — a `Pressable` that calls `router.push('/')`, label from the `BUTTONS` namespace key `back.home.label` (`BackToHomeButton.tsx:10,18`).
- **Footer:** `<Footer />` (`src/components/navigation/Footer.tsx:12`).

`app/(portal)/(public)/about.tsx:42-134` is the richer variant of the same family — same `ScrollView` + `BackToHomeButton` + `Footer` skeleton, but with translated text blocks (`useTranslations(TranslationNamespace.ABOUT_PAGE)`, `about.tsx:45`), a `ScrollToTop` button (`about.tsx:50`), and a conditional CTA gated on auth (`{!user && ...}`, `about.tsx:118`). About is the template if Contact needs authored copy + a button; privacy/terms is the template if Contact is just rendered content.

### Navigation registration

**File-based routing — no explicit `<Stack.Screen>` registration needed.** The route group `app/(portal)/(public)/` has a layout `_layout.tsx:12` that is simply:

```tsx
return <Stack screenOptions={{ headerShown: false }} />;
```

Dropping `contact.tsx` into `app/(portal)/(public)/` auto-registers `/contact`. (The layout also pins `setPortalScope('portal')`, `_layout.tsx:8-10`.) No header is rendered by the navigator — screens supply their own top affordance via `BackToHomeButton`.

The footer link list is **data-driven**, not file-driven. Two arrays feed `Footer.tsx`:
- `src/lib/navigation/companyNavigations.tsx` — `about` → `/about`, `pricing` → `/pricing`, `privacy` → `/privacy`, `terms` → `/terms` (each `{ labelKey, route }`, rendered at `Footer.tsx:45-52` under the `PAGING` namespace).
- `src/lib/navigation/helpNavigations.tsx` — currently a single entry, `blog.free.zone.label` → `/blog/free-zone` (rendered `Footer.tsx:53-60`).

To surface a Contact link in the footer, add one `{ labelKey, route: '/contact' }` row to **`helpNavigations.tsx`** (semantically the "help/support" column) or `companyNavigations.tsx`. Both label sets resolve through `tPaging` (`PAGING` namespace), so the label key must be seeded in `PAGING`.

### Safe-area / scroll wrapper

No screen in this family wraps itself in `SafeAreaView`; safe-area is handled higher up (root layout / navigator chrome), and the content screens rely on a bare `ScrollView`. `MarkdownViewer` adds its own inner `ScrollView` with `contentContainerStyle` width 90% / `paddingBottom: 24` (`src/components/MarkdownViewer.tsx:44-49`) — i.e. nesting a content `ScrollView` inside the screen `ScrollView` is the established (if slightly redundant) pattern. A Contact screen following privacy/terms inherits this exactly.

**Template verdict:** for a "tap these addresses" Contact screen, clone `privacy.tsx` (ScrollView + BackToHomeButton + body + Footer) and replace `<MarkdownViewer>` with a small contact body. For a form, see §4.

---

## 2. Mailto / external link

**API confirmed: `Linking.openURL(...)` with `Linking` imported from `react-native`.** This is the single mechanism the app uses for every outbound jump. There is no `expo-linking` `openURL` usage and no `WebBrowser` for these flows.

Existing call sites (all `import { Linking } from 'react-native'`):

| Call site | What it opens | Scheme |
| --- | --- | --- |
| `src/components/product/CallUserButton.tsx:65` | `tel:${phoneNumber}` | **`tel:`** |
| `src/components/dialog/dialogs/LoginDialog.tsx:83` | web `/forgot-password` (the "Reset password" affordance) | `https:` |
| `src/components/pricingPage/SupportButton.tsx:12` | `https://ko-fi.com/oglasino` | `https:` |
| `src/components/init/SoftUpdateModal.tsx:20` | `getStoreUrl()` (store listing) | `https:` |
| `src/components/init/HardUpdateScreen.tsx:19` | `getStoreUrl()` (store listing) | `https:` |
| `src/components/dialog/dialogs/AppVersionConfigurationDialog.tsx:23` | `https://memento-tech.com` | `https:` |
| `src/components/messages/Message.tsx:42` | `run.url!` (sanitized in-message link) | `https:` |
| `src/lib/permissions/useMediaPermissionDeniedToast.ts:25` | `Linking.openSettings()` (OS settings) | — |

**`mailto:` is the cheap path and is fully supported by this same API, but it is not used anywhere today.** The only `mailto:` occurrence in the repo is a comment in the in-message link sanitizer listing it among schemes to *reject* for chat links (`src/components/messages/utils.ts:31`) — that is a chat-message safety allowlist, unrelated to an app-initiated support link. So:

- A `support@oglasino.com` / `privacy@oglasino.com` tappable address would be a new first use of `Linking.openURL('mailto:support@oglasino.com')`. Same import, same call shape as `CallUserButton`'s `tel:` (`CallUserButton.tsx:6,65`) — that `tel:` site is the closest precedent for a non-`https` scheme.
- **Platform note (worth flagging to the brief):** `tel:` is the only non-http scheme exercised today, and `tel:` is reliably handled by the OS dialer. `mailto:` opens a mail composer **only if a mail client is configured** — on a bare simulator (no Mail account) `Linking.openURL('mailto:…')` can reject. The web "tap-to-email" assumption translates, but the no-mail-client path should degrade gracefully (the addresses should also be visible/selectable as text, not exposed solely behind the tap). This is the iOS/Android gotcha the brief's question 2 is implicitly asking about.

Note the contact addresses themselves are decided in `oglasino-docs/decisions.md:2346`: `privacy@oglasino.com` (privacy), `support@oglasino.com` (general support / appeals) — and that entry records both mailboxes as **not yet operational** (pre-launch action item). That is an upstream/ops dependency, not a mobile blocker, but a mailto link will dead-end until the mailboxes exist.

---

## 3. Auth state + email

**Selector:** `const user = useAuthStore((s) => s.user);` — the canonical pattern, used across the app (e.g. `about.tsx:46`, `FollowUserButton.tsx:26`, `BottomBar.tsx:31`, `owner/user.tsx:29`, `ReportButton.tsx:37`).

**State field:** `user: AuthUserDTO | null` on the auth store (`src/lib/store/authStore.ts:101`). `null` ⇒ logged out. The store persists `user` across launches via `partialize: (state) => ({ user: state.user })` (`authStore.ts:322`).

**Email field:** `AuthUserDTO.email: string` (`src/lib/types/user/AuthUserDTO.ts:8`). So a screen reads the current user's email as `user?.email`. The DTO also carries `displayName` (`AuthUserDTO.ts:7`) if the form wants a name.

**Prepopulate-and-lock (same question as web):** the data is available — read `user?.email` from the selector above and, when non-null, render it into the email field and disable editing. There is no existing mobile screen that *does* prepopulate-and-lock an email today, so there is no in-repo precedent to copy for the lock UI itself; the building blocks (selector + `email` field + the `Input` component used elsewhere) are all present. For a logged-out visitor `user` is `null`, so the field must fall back to free entry. (The about screen demonstrates the auth-gated-render pattern with `{!user && ...}` at `about.tsx:118`, which is the inverse condition and a good reference for branching on auth state.)

---

## 4. Form + submit (only if we go full-form on mobile)

**Cleanest model: `src/components/dialog/dialogs/ReportDialog.tsx`.** It is the tightest example of a user-authored-text submission that POSTs and surfaces backend errors via `parseServiceError` (the Φ4 contract).

Why it's the best fit for a Contact form:
- It collects free text (a `multiline` `TextInput`, `ReportDialog.tsx:144-153`) plus a selected option — structurally identical to "subject + message".
- Client-side structural validation first (required + min-length), set as inline messages from the `VALIDATION` namespace before any network call (`ReportDialog.tsx:58-72`).
- Submits through a thin service: `sendReport(...)` → `BACKEND_API.post('/secure/report/add', …)` (`src/lib/services/reportService.ts:11`), which returns business outcomes for known statuses (`406` = already reported) and **throws on every other failure so the structured error stays reachable** (`reportService.ts:24-35`).
- The catch block reads the backend error via `parseServiceError(err).errors` and renders codes through their `translationKey` against the `ERRORS` namespace, falling back to a generic `ERRORS` toast (`ReportDialog.tsx:91-102`). Codes never become messages — this is the Part 7 / Φ4 contract applied correctly.
- `loading` / `success` / `errorMessage` local state, submit disabled while pending (`ReportDialog.tsx:48-50,171-177`).

`parseServiceError` itself: `src/lib/utils/parseServiceError.ts:40`, returning `{ errors, byField }` (note `field: null` object-level errors are excluded from `byField` — see `preValidateOutcome.ts:19`). Other consumers if a second reference helps: `verificationService.ts:67`, `updateSubmitOutcome.ts:76`, `preValidateOutcome.ts:56`, `UploadedProductDialog.tsx:148`.

**Important seam for a Contact form (flag):** `ReportDialog` posts to `/secure/report/add` — there is **no existing backend contact/support endpoint consumed by mobile** (grep for `contact` in `src/` returns only the `contact_seller_clicked` analytics plumbing on the product buttons, not a support form: `productEvents.ts`, `CallUserButton.tsx`, `StartMessageButton.tsx`, `FavoriteButton.tsx`). If the feature goes full-form, a backend route must exist first and be adopted, not invented mobile-side (CLAUDE.md: no new mobile-specific backend routes without explicit instruction; default reuse web's route). The error-contract plumbing to consume it is already in place; the route is the open question.

`ReportDialog` is a *dialog*, not a *screen*. The form fields/validation/submit pattern transfers verbatim; only the shell changes (screen `ScrollView` + `BackToHomeButton` + `Footer` per §1 instead of `DialogTitleDescription` + dialog chrome).

---

## 5. Translation keys

**Mobile consumes the exact same backend-seeded keys web does**, per namespace, via `GET /public/translations?namespace=${namespace}&lang=${lang}` (`src/i18n/fetchNamespace.ts:32`). The response (`TranslationDTO[]`) is flattened into an i18next resource bundle. Screens read keys through `useTranslations(TranslationNamespace.X)` (`src/i18n/useTranslations.ts:3`).

**Which namespaces mobile fetches:** the boot freshness gate iterates **every** value of the mobile `TranslationNamespace` enum — `for (const ns of Object.values(TranslationNamespace))` (`src/lib/store/bootStore.ts:610`; also passed as the refetch set at `bootStore.ts:700`). So mobile fetches the full set declared in `src/i18n/types.ts:6-39`:

`COMMON, COMMON_SYSTEM, ERRORS, VALIDATION, PAGING, BUTTONS, INPUT, DIALOG, HEADER, FOOTER, NAVIGATION, INTRO, EXTRA_PRODUCTS, COOKIES, MESSAGES_PAGE, DASHBOARD_PAGES, ABOUT_PAGE, FREE_ZONE_PAGE, PRICING_PAGE, METADATA.`

(This is a subset of the backend `TranslationNamespace` — mobile's enum omits e.g. `ADMIN_PAGES`, `BACKEND_TRANSLATIONS`. Mobile only declares what it renders.)

**Where would a Contact screen's strings live? — open question, not resolvable mobile-side.** There is **no `CONTACT` / `SUPPORT_PAGE` namespace** in the mobile enum, and none in the conventions Part 6 namespace list either. Options, by cost:

- If Contact is mostly chrome (a back label, a heading, "email us at…"), the strings split naturally across **existing** namespaces already fetched: heading/body text into `COMMON`, a footer/nav link label into `PAGING` (where `companyNavigations`/`helpNavigations` labels resolve, `Footer.tsx:50,58`), any button labels into `BUTTONS`, any validation into `VALIDATION`/`ERRORS`. No new namespace needed → cheapest path.
- If it's a full form with its own page-scoped copy, web parity would suggest a page namespace — but **inventing a namespace is forbidden** to a mobile engineer (conventions Part 6 Rule 1: agents do not invent namespaces; stop and ask Igor). A new `SUPPORT_PAGE`/`CONTACT_PAGE` namespace would require a backend `TranslationNamespace` enum addition + SQL seed (Backend's job) **and** a matching addition to `src/i18n/types.ts` so the boot loop fetches it.

**Recommendation for the spec:** decide up-front whether Contact reuses `COMMON`/`PAGING`/`BUTTONS` (no new namespace, mailto-only screen — preferred for the cheap path) or warrants a dedicated page namespace (backend-led, full-form path). Mobile cannot choose this alone.

---

## Cross-repo seams / flags for Mastermind

1. **No backend contact/support endpoint is consumed by mobile today** (§4). The full-form path is blocked on a backend route; the cheap mailto path has no backend dependency (only the mailboxes themselves, which `decisions.md:2346` records as not yet operational).
2. **`mailto:` is unused today** (§2). It's the same `Linking.openURL` API mobile already uses for `tel:`, but the "no mail client configured" rejection path (esp. iOS simulator) means addresses should also render as plain selectable text, not be reachable only via the tap.
3. **Translation namespace is undecided** (§5). Mobile cannot invent a namespace; if the spec wants page-scoped Contact copy, that's a backend-led namespace addition mirrored into `src/i18n/types.ts`. The cheap path reuses `COMMON`/`PAGING`/`BUTTONS` and needs no new namespace.
4. **Prepopulate-and-lock email has no in-repo precedent** (§3). The data (`user?.email`) and the `Input` primitive exist, but no current mobile screen renders a locked/disabled prepopulated email; the lock UI would be net-new (small).

---

## Recommended Contact screen shape (synthesis, for the spec author)

- **Cheapest viable:** a static screen at `app/(portal)/(public)/contact.tsx` cloned from `privacy.tsx` (ScrollView → BackToHomeButton → body → Footer), body showing `support@oglasino.com` / `privacy@oglasino.com` as tappable `Linking.openURL('mailto:…')` rows (precedent: `CallUserButton.tsx:65`) **and** as selectable text fallback; footer link added via a `helpNavigations.tsx` row; strings drawn from existing `COMMON`/`PAGING`/`BUTTONS`. No backend work, no new namespace.
- **Full form (only if briefed):** same screen shell, body modeled on `ReportDialog.tsx`'s field + validation + `parseServiceError` submit pattern, posting to a backend support endpoint that must exist and be adopted (not invented), with email prepopulated from `user?.email` when logged in.
