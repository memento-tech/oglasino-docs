# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-06-04
**Task:** Apply the Deep Linking (Universal/App Links) close-out: write the Phase-4 feature spec and apply the paired decisions.md / issues.md / state.md entries + the seo-foundation.md ownership-shift reconciliation.

## Implemented

- **Wrote `features/deep-linking.md`** — the Phase-4 canonical spec (§1 what-it-is, §2 openable surface, §3 tiers/identifiers, §4 architecture [router `.well-known` serve, `+native-intent`, compound-locale strict-validate + base-site switch + link-language filter resolution, filter URL contract], §5 critical path, §6 on-device Ψ, §7 decisions, §8 issues, §9 session log). §7/§8 cross-link the decisions/issues entries applied this same session.
- **`decisions.md`** — one new 2026-06-04 entry at top ("Deep Linking (Universal/App Links): product & architecture decisions"): public-only paths; apex-only; two-tier; router-serves-`.well-known` (web→router ownership shift); compound locale drives strict-validated base-site switch + link-language filter resolution that never persists over the preferred display language (supersedes the earlier "discard the locale" position); no display-language auto-switch; strict-on-locale/lenient-on-filters. Rejected-alternatives block included.
- **`issues.md`** — four new 2026-06-04 entries: prod `assetlinks.json` placeholder SHA-256 (medium/open); stage `assetlinks.json` placeholder SHA-256 (medium/open); in-app `PortalConfigDialog` base-site-switch feed-stale latent bug (medium/open, code-reading inference); web dead app-store badges + no "open in app" surface yet (low/open, deferred). The web-badge entry carries a cross-reference note tying it to the existing 2026-05-31 dead-badge entry to prevent duplicate tracking.
- **`state.md`** — new Deep Linking active-feature block (after JPA Fetch Tuning) + new Expo-backlog row; Last-updated line refreshed. Status recorded as code-complete / pending-Ψ + Android SHAs; "do not promote to `mobile-stable` until Ψ" guard included (per the status-flip-needs-evidence rule).
- **`features/seo-foundation.md`** §11.14 reconciled for the web→router ownership shift: superseded banner pointing at `deep-linking.md`; the "Server emits two well-known files / Web serves…" bullet corrected to router-serves; the `oglasino-web` cross-repo bullet reduced to deferred consumer-surfaces (no longer serves the files); the `oglasino-router` bullet rewritten from "exempts" to "serves directly."

## Files touched

- features/deep-linking.md (new, +~95 lines)
- decisions.md (+1 entry)
- issues.md (+4 entries +1 cross-ref note)
- state.md (+1 active block, +1 backlog row, Last-updated)
- features/seo-foundation.md (§11.14: 3 bullets + 1 banner)

## Tests

- N/A (docs-only repo, markdown).

## Cleanup performed

- Reconciled all three now-contradictory `.well-known`-ownership claims in `seo-foundation.md` §11.14 (not just the one the patch named), so the doc does not contradict itself after the web→router shift.
- Linked the new web-badge `issues.md` entry to the pre-existing 2026-05-31 dead-badge entry rather than letting two entries track the same badges independently.
- Confirmed `README.md` needs no change (it references `features/` generically, with no per-feature list).

## Config-file impact

- conventions.md: no change.
- decisions.md: new entry "2026-06-04 — Deep Linking (Universal/App Links): product & architecture decisions."
- state.md: new Deep Linking active-feature block + Expo-backlog row + Last-updated line.
- issues.md: 4 new 2026-06-04 entries authored (+1 cross-ref note added to the new web-badge entry).

## Obsoleted by this session

- The `seo-foundation.md` §11.14 deep-linking sketch is superseded by `features/deep-linking.md`. **Not deleted** — kept (banner-marked, defer-to-spec) because it sits inside SEO's deferred-items catalogue and still carries SEO context (smart app banner, `potentialAction`, ASO adjacency); only the now-wrong ownership/locale claims were corrected. Deleting the subsection would lose the SEO-acquisition framing; superseding-in-place is the cleaner call.
- Nothing else.

## Conventions check

- Part 4 (cleanliness): confirmed — seo-foundation self-contradiction resolved; issues cross-ref added; README revalidated (no change needed); spec §7/§8 cross-references resolve to real entries applied the same session (no dangling forward-refs).
- Part 4a (simplicity): see "For Mastermind."
- Part 4b (adjacent observations): one flagged below (issues-entry overlap, handled in-session).
- Part 6 (translations): N/A this session.
- Other parts touched: Part 3 (sole-writer of the four config files — authorized close-out, upstream draft = Igor's pasted patch); Part 10 (Phase-4 spec application).

## Known gaps / TODOs

- The feature itself is owner-work, not agent-work: preview/prod Android SHA-256 swaps, prod+preview prebuild/build, commit of the four stacked `new-expo-dev` sessions, and the on-device Ψ pass. All captured in `state.md` tasks-remaining and `features/deep-linking.md` §5/§6. No `mobile-stable`/`adopted` flip made (Ψ owed).
- No engineer session summaries to archive this session — the brief was a spec-application close-out, not an archival task. (The four stacked expo deep-linking sessions are still uncommitted in `oglasino-expo/.agent/`; archive them when Igor surfaces them.)

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): the Deep Linking active-feature block in `state.md` (a real, distinct in-flight feature with its own Ψ gate and owner tasks — earns a block like the other Expo features); the Expo-backlog row (this feature has no backend/web `web-stable` predecessor, so the row's "Web/Backend status" cell is annotated as a router+expo feature rather than left blank — one-off but accurate).
  - Considered and rejected: deleting the `seo-foundation.md` §11.14 subsection outright (rejected — keeps SEO context; superseded-in-place instead); a second new `issues.md` entry duplicating the 2026-05-31 dead-badge tracking (rejected — added a cross-ref note to the new entry instead).
  - Simplified or removed: collapsed the three divergent `.well-known`-ownership statements in `seo-foundation.md` into one consistent web→router story.
- **Judgment calls beyond the literal patch (flagged for visibility):** (1) the patch §4 drafted a replacement only for the `oglasino-router` "exempts" bullet; I also corrected the sibling "Web serves…" and `oglasino-web — serves .well-known` claims in the same subsection, because leaving them would contradict the drafted text — this is the Part 4 cleanliness mandate applied to the drafted change, not a new decision. (2) I added a cross-reference note linking the new web-badge issue to the existing 2026-05-31 badge entry. Both are conservative; flag if either should be reverted.
- **Brief-vs-reality resolved up front:** the spec's §7/§8 originally forward-referenced `decisions.md`/`issues.md` (2026-06-04) entries that did not exist on disk; I surfaced this before applying, Igor responded with the full close-out patch supplying those entries, and they were applied in this session so the references now resolve. No outstanding contradiction.
- Nothing else flagged.
