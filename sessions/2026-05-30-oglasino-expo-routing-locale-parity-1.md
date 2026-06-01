# Session summary

**Repo:** oglasino-expo
**Branch:** new-expo-dev
**Date:** 2026-05-30
**Task:** Audit (read-only) — how is the routing locale resolved in mobile, and is it web-compatible for the prefixed/shared product URL?

> ⚠️ **Brief-swap notice (read first).** At session start the harness delivered
> the *routing-locale audit* brief (read-only) via `.agent/brief.md`, and this
> session executed it in full. **By end of session, `.agent/brief.md` on disk had
> been replaced with a different brief — "image pipeline conformance fixes
> (V6 + V9)" (code changes).** `.agent/` is untracked so git shows no diff. I did
> NOT start the image-pipeline work — it requires code changes and I will not act
> under brief ambiguity. Igor to confirm which brief is live next.

## Implemented

- Read-only audit only — **no code changed**. Findings written to
  `.agent/audit-routing-locale-parity.md`.
- Traced mobile's locale source of truth to `useBootStore`
  (`src/lib/store/bootStore.ts`), holding base site + language as **two separate
  DTO slots** (`selectedBaseSite.code` like `rs`, `language.code` like `sr`) —
  the same codes the axios interceptor sends as `X-Base-Site` / `X-Lang`
  (`src/lib/config/api.ts:44-46`). **No compound `rs-sr` value is stored.**
- Verified the web format (`useRoutingLocale()` → `rs-sr`, `{baseSite}-{language}`)
  is **composable 1:1** from mobile's two fields with no casing/separator/ordering
  mismatch, because both clients consume the same backend DTOs.
- Found two divergences and one **live bug** at the prefixed-URL call sites
  (see "For Mastermind").
- Confirmed web requires the locale segment: invalid/missing → `notFound()` 404
  (`oglasino-web/app/[locale]/layout.tsx:26-27`); `proxy.ts` redirects valid-shape
  wrong-combo or cookie-mismatched locales but 404s malformed ones.

## Files touched

- None (read-only). Deliverables written under `.agent/`:
  - `.agent/audit-routing-locale-parity.md` (new — the audit)
  - `.agent/2026-05-30-oglasino-expo-routing-locale-parity-1.md` (this summary)
  - `.agent/last-session.md` (copy of this summary)

## Tests

- None run (read-only audit; no code touched).

## Cleanup performed

- none needed (no code changed).

## Config-file impact

- conventions.md: no change
- decisions.md: no change
- state.md: no change (this is a Phase-2 audit; any feature-row/backlog updates
  are Mastermind/Docs-QA's to draft once a parity feature is scoped)
- issues.md: no change. NOTE: the `ShareProductButton` `useLocale()` →
  `[object Object]` bug is a genuine user-facing defect surfaced by this audit;
  it is in-scope for the parity feature this audit feeds, so I did NOT draft a
  standalone issues.md entry. If Mastermind decides the parity feature is not
  imminent, this warrants its own `issues.md` entry — flagged below.

## Obsoleted by this session

- nothing.

## Conventions check

- Part 4 (cleanliness): confirmed — no code changed, nothing to clean.
- Part 4a (simplicity): see structured evidence in "For Mastermind".
- Part 4b (adjacent observations): two flagged in "For Mastermind".
- Part 6 (translations): N/A this session.
- Other parts touched: Part 8 (architectural defaults — routes/URLs reusable
  across web/mobile) is the lens of the whole audit; Part 9 (locales — me/cnr
  aliasing) addressed in the audit's Item 2.

## Known gaps / TODOs

- none (audit is complete per the brief's definition of done).

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing (read-only).
  - Considered and rejected: nothing (read-only).
  - Simplified or removed: nothing (read-only).

- **Audit verdict (the deliverable):** mobile has NO web-format locale value; it
  must be **composed** as `` `${selectedBaseSite.code}-${language.code}` `` from
  `useBootStore` → yields `rs-sr` with zero format mismatch. Both prefixed-URL
  call sites currently bypass this source. Full detail + file:line in
  `.agent/audit-routing-locale-parity.md`.

- **Adjacent observation #1 (HIGH, user-facing) — `ShareProductButton` shares a
  broken URL.** `src/components/product/ShareProductButton.tsx:23,29`. It reads
  `useLocale()` from `@react-navigation/native`, which returns
  `{ direction: 'ltr' }` (text direction), not a locale — so the shared message
  is `https://www.oglasino.com/[object Object]/product/{id}/{raw name}`. That URL
  **404s on web** (no hyphen in the segment) and the name is un-normalized. I did
  NOT fix this — it is the core thing the parity brief this audit feeds will
  address. If that brief is not imminent, this deserves its own `issues.md`
  entry (severity high: any user pressing Share today produces a dead link).

- **Adjacent observation #2 (MEDIUM) — create-dialog URL diverges from web.**
  `UploadedProductDialog.tsx:109` via `utils.ts:114` emits
  `https://www.oglasino.rs/product/{id}/{slug}` — no locale segment, domain
  `www.oglasino.rs` vs web's bare `oglasino.com`. Copied link won't match web's
  canonical and (with no `[locale]` segment) 404s on web. Out of scope for a
  read-only audit; it is exactly what the parity brief targets.

- **Brief vs reality (informational, not blocking).** The brief framed BOTH call
  sites as building the prefixed URL "the `getNormalizedProductUrl(..., true)`
  call" in `UploadedProductDialog.tsx` and `ShareProductButton.tsx`. Reality:
  only the dialog uses the helper; `ShareProductButton` builds the URL **inline**
  and **does not call `getNormalizedProductUrl` at all**. This is a finding the
  audit was meant to surface, so no stop-and-challenge was warranted — reported
  in the audit. The parity brief should plan for **two different code shapes**,
  not one helper call repeated.

- **UNDETERMINED handed up:** whether `www.oglasino.com` 301s to apex
  `oglasino.com` is a Cloudflare-router/DNS concern, not visible in the web app
  repo. Recommend mobile emit bare `oglasino.com` (matching web canonical) so the
  copied link never depends on a www→apex redirect.

- **Process flag (action needed):** `.agent/brief.md` on disk changed during this
  session (routing-locale audit → image-pipeline V6/V9 fixes). Please confirm
  which brief is the live task before I proceed; the image-pipeline brief needs
  code changes and I've held off.

- **Config-file impact:** none required from this audit (stated above). No drafted
  config-file text.
