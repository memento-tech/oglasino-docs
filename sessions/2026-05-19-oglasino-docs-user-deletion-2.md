# Session summary

**Repo:** oglasino-docs
**Branch:** main
**Date:** 2026-05-19
**Task:** Docs/QA close-the-feature pass for User Deletion — apply all accumulated config edits, archive sessions, draft legal delta, flip state, write the Mastermind transition note. Nine blocks A–I from the brief, adjusted per Mastermind triage of six brief-vs-reality flags surfaced at session open.

## Implemented

- **Block A — link to test cases doc.** Brief's file-creation step was already on disk (Mastermind drafted `features/user-deletion-test-cases.md` 622 lines elsewhere; file already untracked at session open). Only the link reference added at the top of `features/user-deletion.md` plus a refresh of the spec's "Last updated" date to 2026-05-19. Did NOT overwrite the existing test-cases file.
- **Block B — `decisions.md` entries (5).** Five 2026-05-19 entries appended at top: Backend → web cache revalidation pattern, Banned-state UI visibility design, Backlog-triage closed, Cache-invalidation profile fix (Next.js 16), Pre-production schema fold for V1 migrations.
- **Block C — `issues.md` entries (5 applied, 1 skipped as duplicate).** Applied: Registration `displayName` never persists; Admin Users detail page indicators stale; Backend → web cache revalidation may not work in production via oglasino.com domain; Home page only loads first page; `/api/revalidate` wrong cache-invalidation profile (status: fixed). **Skipped Block C entry 3** — "Manual testing of messaging gating" already on disk at `issues.md:7-27` (more comprehensive than the brief's draft).
- **Block D — `state.md` flip.** User Deletion status `backend-stable` → `shipped` (code) / `verifying` (manual smoke pending on full surface, blocked on messaging fix). Added link to manual test cases. Added User Deletion row to the Expo backlog table. Refreshed "Last updated" to 2026-05-19.
- **Block E — `conventions.md` additions (1 applied, 2 skipped).** Added "Pre-production V1 schema fold" sub-section to Part 12 (Schema patterns). **Skipped Convention B** (translation seed convention) — already present at Part 6 Rule 3. **Skipped Convention C** (pre-launch action items capture) — feature-spec-shaped, doesn't earn a place in project-wide conventions.
- **Block F — `secret-inventory.md` additions (2).** Added `REVALIDATE_SECRET` (shared secret for backend → web revalidation header) and `WEB_REVALIDATE_URL` (backend target URL — direct Vercel in prod to bypass Cloudflare router collision) to the inventory table.
- **Block G — spec amendments (5 applied, 1 expanded, 1 no-op).** Applied by content per Mastermind guidance, with the drift table below.
- **Block H — Privacy Policy §9 delta.** Single-sentence addition to the existing §9 Right-to-erasure paragraph adding the `support@oglasino.com` path for users who cannot use the self-service deletion flow. Existing `privacy@oglasino.com` reference preserved for general privacy enquiries. **Skipped Block H full rewrite** — Privacy Policy §8/§9 and Terms §10 already drafted comprehensively in the 2026-05-15 legal-drafts session; Terms §10 needed no further delta per Mastermind decision.
- **Block I — sessions archival (35 files).** Copied 17 backend + 18 web user-deletion session files spanning 2026-05-17 → 2026-05-19 from `oglasino-backend/.agent/` and `oglasino-web/.agent/` to `oglasino-docs/sessions/`, verified all 35 copies byte-identical to sources via `diff -q`, then **deleted all 35 source files outright** (no pointer stubs left behind, per Igor's clarification mid-session — supersedes the initial pointer-stub approach I followed from conventions Part 5 block 7A). Sibling `.agent/` folders now contain zero session-shaped user-deletion files. (Igor's brief said "36 files"; actual count was 35 — backend 17 + web 18, not web 19.)

## Block G drift table (amendment-by-amendment)

| Brief's section | Actual landing site | Rationale |
|---|---|---|
| Amendment 1 — §14.14 UserInfoDTO state union | §8.7 (backend `UserInfoDTO`) AND §14.14 (frontend TS interface) | The union exists in two places: Java DTO in §8.7, TypeScript interface in §14.14. Updated both to `'ACTIVE' \| 'PENDING_DELETION' \| 'BANNED'` and added the BANNED-wins composition-rule note pointing at `UserState.resolve()`. |
| Amendment 2 — new §3.3 banned-user content hiding | New sub-section §3.5a inserted between §3.5 (Banned-user veto model) and §3.6 (Banned users delete via support email) | §3.3 already taken ("Reviews — four-case split"). §3.5 is the existing banned-user veto thread, so content-hiding belongs immediately after it. Used `3.5a` to avoid renumbering §3.6–§3.12. |
| Amendment 3 — §8.4 cache invalidation pattern | New sub-section §8.9 inserted between §8.8 (Error codes) and §9 (Scheduled jobs) | §8.4 already taken (`FirebaseUserService` extension). §8 is the right parent (backend service contracts); §8.9 is the next available slot. Cross-references to decisions.md 2026-05-19, issues.md 2026-05-19, and secret-inventory. |
| Amendment 4 — §9.3 audit purge cron | §9.3 (Weekly audit purge) — matches | Job body extended with the third repository call (`adminActionAuditRepository.deleteExpired`); prose extended to name all three audit tables and the per-table retention. |
| Amendment 5 — §3.8 admin actions audit log | §3.7 (Un-ban removes the hash) | §3.8 actual topic is admin postponement (different concern). Unban-audit is the logical extension of §3.7 — extended the §3.7 paragraph with the `user_admin_action_audit` table description, write-before-delete ordering, and 12-month retention pointer back to §9.3. |
| Amendment 6 — schema field renames | No-op | Neither `product.deactivated_by_user_deletion` nor `changeProductStateAsSystemForUserDeletion` appears in the current spec text. The `changeProductStateAsSystem` form is already used at §8.3 line 818, §8.1 lines 643 and 667. Rename was applied upstream before this session. |
| Amendment 7 — §17 pre-launch checklist | §20 (Pre-launch action items) | §17 actual topic is "Edge cases and policies"; the pre-launch checklist is §20. Existing §20.3 and §20.4 marked as applied (this session); new §20.5–§20.8 added for `REVALIDATE_SECRET` provisioning, `WEB_REVALIDATE_URL` provisioning, lawyer review, native-translator review. Existing §20.5–§20.10 renumbered to §20.9–§20.14 to make room. Duplicate "Mailbox setup" entry I initially added was removed (§20.1 + §20.2 already cover that). |

## Files touched

- `features/user-deletion.md` (Block A header link + Block G all amendments; net positive line delta, hundreds of lines)
- `decisions.md` (+5 entries at top, ~30 lines added)
- `issues.md` (+5 entries at top, ~80 lines added)
- `state.md` (User Deletion section flip; Expo backlog row added; last-updated date refresh)
- `meta/conventions.md` (Part 12 V1 schema fold sub-section added, ~7 lines)
- `infra/overview/secret-inventory.md` (+2 inventory rows)
- `legal/privacy-policy-draft.md` (1-sentence delta in §9 Right-to-erasure paragraph)
- `oglasino-docs/sessions/` (+35 new archived session files, all byte-identical to their sibling-repo sources)
- `oglasino-backend/.agent/2026-05-1{7,8,9}-oglasino-backend-user-deletion-*.md` × 17 (each replaced with 1-line pointer stub)
- `oglasino-web/.agent/2026-05-1{7,8,9}-oglasino-web-user-deletion-*.md` × 18 (each replaced with 1-line pointer stub)

## Tests

N/A — docs-only session. No code, no test runners.

## Cleanup performed

- Block I sessions archival pass (35 files): byte-identical copies verified, then sources **deleted outright** per Igor's mid-session correction (no pointer stubs). **Final state in sibling `.agent/` folders:** zero session-shaped user-deletion files remain. Verified via `ls | grep user-deletion` post-cleanup — count of session-shaped files matching `2026-05-1*-oglasino-{backend,web}-user-deletion-*.md` is `0` in each repo. (Per conventions Part 5 block 7A, pointer stubs would normally be the discoverability mechanism — Igor's instruction overrides that for this session. Future-session `<n>`-counting on the user-deletion slug now depends on engineers counting against the archive in `oglasino-docs/sessions/` rather than against their own local `.agent/`. Surfaced in "For Mastermind" as a candidate amendment to Part 5 block 7A.)
- **Non-session files intentionally left in place.** The Block I scope is session files (the `yyyy-mm-dd-<repo>-<slug>-<n>.md` template). Other user-deletion-named artifacts in sibling `.agent/` folders — `audit-user-deletion.md`, `audit-user-deletion-auth-lifecycle.md`, `trust-boundary-audit-user-deletion.md` (web only), `user-deletion-pr-review.md` — are audit/PR-review reference docs, not sessions, and do not follow the archive convention. They stay where engineers wrote them. Total non-session user-deletion files remaining: 3 in backend `.agent/`, 4 in web `.agent/`.
- Block G §20 duplicate-section-number repair (§20.5–§20.9 had collided with original §20.5–§20.10 mid-edit; redundant "Mailbox setup" entry I added was removed, original entries renumbered to §20.9–§20.14, final numbering verified unique via grep).
- Spec "Last updated" date moved from 2026-05-17 → 2026-05-19 on `features/user-deletion.md`. State-file last-updated moved 2026-05-18 → 2026-05-19.

## Config-file impact

- `meta/conventions.md`: new Part 12 sub-section "Pre-production V1 schema fold" added (Block E Convention A). No edits to Parts 1–11 or Part 13.
- `decisions.md`: 5 new entries at top (Block B). All dated 2026-05-19. Existing 2026-05-18 entries untouched.
- `state.md`: User Deletion section status flip (`backend-stable` → `shipped` / `verifying`), Expo backlog row appended, "Last updated" date refresh.
- `issues.md`: 5 new entries at top (Block C, one of which is fixed-status). Existing 2026-05-19 messaging-gating entry left untouched per Mastermind decision (already more comprehensive than the brief's draft).

## Obsoleted by this session

- The pending Privacy Policy §9 amendment described in user-deletion.md §20.3 is now applied on disk — the §20.3 prose has been updated to reflect that.
- The pending Terms §10 amendment described in §20.4 is closed as "no further textual delta required" (existing §10 already covers the support@ appeals path per Mastermind triage).
- The pending Mastermind "decision log entry at ship time" §20.12 (formerly §20.8) is now satisfied by the five 2026-05-19 entries applied in Block B — §20.12 prose updated to point at them.
- `features/user-deletion-auth-contract.md` was untracked in git status at session open and not touched — flagged in "For Mastermind" since I did not verify it's a desired artifact.

## Conventions check

- Part 4 (cleanliness): confirmed. No dead links introduced; no duplicate content (the duplicate §20.5–§20.9 collision I created mid-edit was repaired in the same session before completion). All edits are referenced from at least one consumer (spec, state, secret inventory).
- Part 4a (simplicity): see structured evidence in "For Mastermind." Mostly N/A for docs work; structured answer below.
- Part 4b (adjacent observations): one flagged below regarding `features/user-deletion-auth-contract.md`. Otherwise nothing in scope.
- Part 5 (session-summary template): confirmed. This summary, the named twin at `.agent/2026-05-19-oglasino-docs-user-deletion-2.md`, and the `last-session.md` copy all carry identical content. `<n>=2` per the existing `2026-05-18-oglasino-docs-user-deletion-docs-wrap-1.md` matching the `*-user-deletion-*.md` glob.
- Part 6 (translations): N/A — no new translation keys added in this session.
- Part 11 (trust boundaries): N/A — no trust-boundary content authored or audited.

## Known gaps / TODOs

- **Manual smoke pending on the full surface.** Blocked on messaging fix per `issues.md` 2026-05-19 messaging-gating entry. Cannot run end-to-end pending-deletion + banned counterparty observations until then.
- **Lawyer review of legal drafts pending.** Privacy Policy and Terms of Use drafts ready for handoff via `legal/lawyer-handoff.md`.
- **Native-translator review of placeholder RS/CNR/RU translations pending** for the User Deletion `BANNED_DIALOG` and other added keys.
- **Cross-repo provisioning by Igor pending.** `REVALIDATE_SECRET` (Vercel + DigitalOcean droplet, same value on both) and `WEB_REVALIDATE_URL` (DigitalOcean droplet, direct Vercel URL).
- **Backend → web cache revalidation production verification pending.** Direct-Vercel-URL approach untested in production; if fragile, three alternative architectural fixes are catalogued in the `issues.md` 2026-05-19 entry.
- **Other `revalidateTag(...)` call sites still need cache-profile audit.** The `/api/revalidate` site is fixed; `oglasino-web/app/actions/cacheActions.ts` and possibly others remain. Separate task.
- **Sibling-repo `.agent/` cleaned of session files (no stubs).** Per Igor's mid-session correction, the 35 archived sources were deleted outright instead of being reduced to pointer stubs. Sibling `.agent/` folders contain zero session-shaped user-deletion files post-session. This contradicts conventions Part 5 block 7A (which requires pointer stubs); flagged below for a possible conventions amendment.

## For Mastermind

- **Part 4a simplicity evidence (required):**
  - Added (earned complexity): nothing. Docs session — no new abstractions, no new configuration values, no new code-side patterns introduced. Convention A in `conventions.md` Part 12 is text describing an existing-and-already-decided practice, not new mechanism.
  - Considered and rejected: full rewrite of Privacy Policy §9 and Terms §10 (rejected — existing 2026-05-15 drafts cover the brief's bullets comprehensively; rewriting would have destroyed reviewed work); creating new `features/user-deletion-test-cases.md` from brief's "chat transcript" (rejected — file already on disk; no transcript available; overwriting would destroy 622 lines); literal §14.14 / §3.3 / §8.4 amendment placement per Block G (rejected — section numbers don't match reality; applied by content with drift table); archiving only 2026-05-19 sessions per Block I literal wording (rejected — would leave 20 earlier user-deletion sessions un-archived in source repos; archived all 35 instead).
  - Simplified or removed: redundant new §20.9 "Mailbox setup" sub-section I initially added (already covered by existing §20.1 and §20.2 — removed mid-session, originals renumbered to keep §20 numbering unique).
- **One adjacent observation (Part 4b):** `features/user-deletion-auth-contract.md` exists on disk and shows as untracked in `git status` (`??` marker), 350 lines. Not referenced by `user-deletion.md` and not addressed by this session's brief. Either (a) it's a paired companion to `user-deletion-test-cases.md` that Mastermind wants kept and referenced from the spec, or (b) it's a stale artifact. Severity: low. I did not touch it because it's out of scope for this brief.
- **Block I count discrepancy.** Igor's clarification said "36 files: backend 17 + web 19." Actual `ls` count was 35 (backend 17 + web 18). I archived all 35 visible files. If Igor expected a specific 19th web file, it wasn't in `oglasino-web/.agent/` at session open. Worth a sanity check before committing.
- **Docs/QA's own previous user-deletion-* session file.** `oglasino-docs/.agent/2026-05-18-oglasino-docs-user-deletion-docs-wrap-1.md` matches the `*-user-deletion-*.md` pattern but lives in this repo's own `.agent/`, not a sibling. I did not archive it because Igor's Block I scope was explicit ("both repos: backend 17 + web 19"). Question: should that file be archived in a later pass to `oglasino-docs/sessions/`? Convention isn't explicit on Docs/QA's own archival cadence.
- **Pointer-stub vs delete-outright contradiction (candidate conventions amendment).** Conventions Part 5 block 7A says archival leaves a 1-line pointer stub at the source filename. Igor's mid-session correction for this archival pass said delete the files entirely — no stubs. The end state Igor wants matches the original brief's "After archival, `.agent/` is clean" wording. If this becomes the standing rule, Part 5 block 7A needs an explicit edit (Mastermind drafts; I apply). Otherwise the next Docs/QA archival session will follow the still-on-disk stub rule and produce a different end state. Worth resolving in conventions before the next archival pass.
- **`issues.md` ordering note.** The five new entries I added are stacked above the existing 2026-05-19 messaging-gating entry at the top. Strict newest-first might want one specific ordering within the same date; I kept the brief's order. If you want a particular ordering for the 6 same-date entries, that's a follow-up.
- **Convention C not added.** Convention C ("Pre-production action items captured for launch") was skipped per Mastermind decision because it's feature-spec-shaped, not project-wide. If Mastermind later wants it codified as a meta-convention for *every* feature spec to carry a §17/§20-equivalent, draft it explicitly so I can apply.
- **Amendment 6 was a no-op.** The string renames `deactivated_by_user_deletion → deactivated_by_system` and `changeProductStateAsSystemForUserDeletion → changeProductStateAsSystem` weren't present in the current spec — the latter already used the new form, the former never appears at all. If there's a backend file or migration where the old name still lives, that's a sibling-repo concern outside Docs/QA scope.
- **Drafted config-file text — none pending.** Every config edit this session was applied to disk in this session. No "drafted but not applied" left over.

End of summary.
