# Session summary — oglasino-expo — deeplink-basesite-switch (1)

**Date:** 2026-06-04 · **Repo:** oglasino-expo · **Branch:** `new-expo-dev` (not changed)
**Type:** READ-ONLY audit. No source edited, no builds, no `eas`, no git ops.
**Deliverable:** `.agent/audit-deeplink-basesite-switch.md` (the audit; full evidence there).

## Task (one sentence)
Audit whether runtime base-site switching on a deep-link cold-start is feasible, what cascade it drags, how it sequences against the cold-start filter hydration, and where the valid/invalid base-site fallback branches — to make the implementation brief writable (or prove it needs a bigger conversation).

## Findings (headline)
- **Switch primitive exists:** `bootStore.pickBaseSite(code)` (`src/lib/store/bootStore.ts:307`), reachable post-boot via `PortalConfigDialog`. Not a from-scratch job.
- **Cascade is one DTO:** `BaseSiteDTO` embeds catalog (categories/topFilters/orderTypes), regions→cities, currencies, allowedLanguages. One `fetchBaseSiteByCode` loads the whole context; no React Query, no multi-resource invalidation.
- **Two mount-once gaps the switch does NOT drive:** filter hydration (`useState` initializer, once per mount) and the product feed (`ProductList` mount-once load, keyed on language + `fetchPage`, never `selectedBaseSite`). The portal Stack stays mounted through a switch (`'ready'→'updating'→'ready'`), so neither re-runs.
- **Sequencing:** a base-site-switching link is a **separate, boot-integrated cold-start path**. The switch must complete **before the portal mounts** → fold into **boot Gate 3** (Design A). The in-place `PortalConfigDialog` flow is not reusable. No existing readiness signal holds the portal off during a switch.
- **Fallback:** validate against `baseSiteOverviews`, which is **empty for a returning user** → needs an overviews fetch added to the cold-start path. Branch lives in Gate 3, not `+native-intent`. Side-channel: a module-level stash written by `redirectSystemPath` (it already receives an unused `initial` flag), read-and-cleared by the boot consumer.
- **Size: MEDIUM (heavy end)** — switch primitive + one-DTO cascade exist (not large); but hydration doesn't wait and the feed isn't base-site-keyed (not small); and the fix edits boot Gate 3 (cold-start state).
- **Language facts gathered (not decided):** labels exist only in the active display language (Gate 4 loads `(ns, activeLang)` only; resolution uses active `t()`), so option (b) "switch display to link language" is cheaper-and-correct than option (a) "keep display, resolve in link language" (which needs an extra label-set + a second translator). A base-site switch can force a language change anyway when the link/current language isn't in the new site's allowed set (`cnr` is `me`-only).

## Brief vs reality
Nothing to challenge. The brief's framing was accurate; the audit confirmed and sharpened it (notably: the cascade is a single DTO, not a fan-out; a switch primitive already exists; the gap is the two mount-once mechanisms + the boot-flow seam). No code contradicted the brief.

## Obsoleted by this session
Nothing. No prior audit or doc is superseded; this is the first audit for this slug.

## Cleanup performed
None needed (read-only; no source touched).

## Conventions check
- Part 4 (cleanliness): N/A — no code edited; no lint/tsc/test required.
- Part 4a (simplicity) / 4b (adjacent observations): one adjacent finding (in-app `PortalConfigDialog` base-site switch likely leaves the feed stale when language is unchanged — `ProductList` has no `selectedBaseSite` refetch trigger) recorded in the audit's "For Mastermind" rather than acted on. Recommended as an `issues.md` row (Docs/QA writes it).
- Hard rules: honored — no commit/push/branch-change, no edits to config files or other repos, stayed on `new-expo-dev`, no builds/`eas`/prebuild.

## Config-file impact
None. This audit adopts/removes no feature, so no `state.md` Expo-backlog row changes. No edit to any of the four governed docs is needed or made. (Closure gate: no implicit config-file dependency outstanding.)

## For Mastermind
See the audit's "For Mastermind" section for the full set. Key items:
1. **Boot-flow change, flag loudly:** the clean implementation edits boot **Gate 3** to read the deep-link side-channel and switch before the portal mounts — a cold-start boot-state change; the three `bootStore` invariants must be honored.
2. **Validation cost:** returning users hold no available-set; deep-link base-site validation adds an overviews fetch to cold start (decide: always vs only-when-differs).
3. **Latent existing bug (independent of this feature):** in-app `PortalConfigDialog` base-site switch doesn't refresh the feed unless language also changes — candidate `issues.md` entry. Could not confirm on-device (code-reading inference).
4. **Cross-repo (web):** confirm web emits the `{base-site}-{language}` segment with base-site = a real `code` and language ∈ that site's allowed set, and slugs filter/region/city values in the link's language.
5. **Could not verify:** on-device whether `redirectSystemPath` for the initial URL always runs before Gate 3's async storage read resolves (Design A's ordering dependency) — require an on-device cold-start check (`me-*` link, stored `rs` site) in the implementation brief.
