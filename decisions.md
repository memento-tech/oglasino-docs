# Decisions

Append-only log of decisions made across planning sessions. Igor commits entries. Mastermind drafts them. Engineers read but don't write.

Format: each decision has a date, a one-line summary, the reasoning, and the alternatives we considered. Newest at the top.

---

## 2026-05-14 — Conventions Part 4 corrected: backend is Maven, not Gradle

`meta/conventions.md` Part 4 (Cleanliness) listed the backend formatter and test commands as `./gradlew spotlessCheck` and `./gradlew test`. The backend repo is Maven. Engineers have been running `mvn spotless:check` and `mvn test` all along — the convention was wrong and the engineers' actual practice was right. Part 4's backend line is corrected to the Maven commands.

This surfaced during the product-filtering bug sweep: two backend sessions reported running `mvn` commands, and the engineer flagged the convention mismatch explicitly in its session summary.

**Consequence for queued work:** two backend briefs drafted but not yet run — the `oldName` trust-boundary fix and the `translationKey`-on-the-wire change — reference `./gradlew` in their hard rules and definition-of-done sections. Both briefs must have their commands corrected to Maven before they are handed to the backend engineer. The briefs are drafts in chat history, not committed artifacts, so this is a correction-at-hand-off, not a committed-file change.

**Open item:** the corrected line uses unscoped `mvn spotless:check` and `mvn test`. The original Gradle line implied "for touched modules" scoping. If the backend is multi-module and scoped runs are wanted, the exact `mvn -pl <module>` invocation needs Igor's confirmation — this entry does not guess at Maven module-scoping syntax. Unscoped commands are correct (the filtering-sweep backend sessions ran them and passed), just potentially broader than necessary.

**Reasoning:** a convention that contradicts actual practice is worse than no convention — it either gets ignored (eroding the conventions file's authority) or followed by a new agent and breaks. Same category as prior "convention didn't match reality" corrections. Cheap to fix, and it unblocks the two queued briefs from inheriting a wrong command.

**Alternatives considered:** guessing the Maven module-scoping syntax to preserve the "touched modules" intent (rejected — wrong syntax in a conventions file is the exact problem being fixed; better to ship the correct unscoped command and flag scoping as a follow-up).

---

## 2026-05-14 — Image convention and session-file naming added to conventions

Two additions to `meta/conventions.md`.

Part 1 gains an `### Images` subsection. Docs and any agent referencing images write the image reference as if the file exists, name it kebab-case and descriptive (`product-creation-step-2.png`, not `screenshot1.png`), and add an HTML comment above the reference describing what the image should show — scaled to image complexity. Images live in an `assets/` folder next to the referencing doc. Igor supplies the actual files after the doc is written.

Part 5's session-file rule changes. Previously every engineer wrote `.agent/last-session.md`, overwritten each session — so all session history except the latest was lost. Now the engineer writes the summary to `.agent/yyyy-mm-dd-<repo>-<slug>-<n>.md` (a permanent, uniquely-named record) and *also* to `.agent/last-session.md` (an exact copy, kept so Igor and existing briefs have a predictable path to the latest). `<n>` is sequential per `(repo, slug)` pair, not per day; the engineer counts existing `*-<slug>-*.md` files in its own `.agent/` folder and adds one. Per-repo counting is necessary because an engineer cannot see another repo's `.agent/` folder. The rule applies going forward; pre-rule `last-session.md` files are not backfilled because overwritten content cannot be recovered.

**Reasoning:** the image convention lets docs be written before screenshots exist, without placeholder cruft, and keeps the guidance for each missing file attached to the spot it's needed. The session-naming change fixes real data loss — every engineer session before this rule destroyed the previous session's summary. Numbered permanent files give the feature a full session history; the `last-session.md` copy preserves backward compatibility with every brief and habit that already points at that path.

**Alternatives considered:** for images — visible italic captions instead of HTML comments (rejected — the guidance is a note to the file-supplier, not reader-facing content; alt text already serves the reader). For session naming — per-day numbering (rejected — a slug's sessions don't map cleanly to days; per-slug sequential is the meaningful order). Dropping `last-session.md` entirely once named files exist (rejected — every existing brief and Igor's own workflow point at that path; keeping it as a duplicate costs nothing). Backfilling old session files (rejected — impossible; overwritten content is gone).

---

## 2026-05-13 — Simplicity guideline and adjacent-observations rule added to conventions

Two related additions to `meta/conventions.md`:

**Part 4a — Simplicity.** Guidelines (not bans) on when complexity earns its place. Engineers exercise judgment but explain non-obvious choices in the session summary's "For Mastermind" section. Configuration, abstractions, defensive code, and stylistic deviations are all available when they serve a concrete purpose — and visible when they do.

**Part 4b — Adjacent observations.** Engineers flag bugs and issues they notice outside their brief's scope. Flags include file path, severity guess, and an explicit "out of scope" acknowledgement. Mastermind triages flags into `issues.md`, follow-up briefs, or wontfix.

Session summary template gains one row in "Conventions check" covering both parts.

**Reasoning:** prior session output suggested engineers default to over-engineering (extra abstractions, configuration for non-varying values, defensive checks where the contract is tight). Parallel concern: engineers don't always flag adjacent issues, so bugs that should have been visible get rediscovered later. The two rules pull in opposite directions on attention — simplify what you write, broaden what you notice — which is the right balance for code quality.

**Alternatives considered:** a strict ban list ("no new abstractions without two existing call sites," etc.) — rejected because it kills good engineering instincts along with the bad. A "be simple" exhortation without checkable substance — rejected because vague rules don't change behaviour.

---

## 2026-05-13 — Translation namespaces reconciled with code; VALIDATION frozen

Conventions Part 6 Rule 1 listed 5 namespaces. The `TranslationNamespace` enum in `oglasino-backend` has 22. Conventions were stale. Reality is correct.

Conventions Part 6 Rule 1 is rewritten to list all 22 namespaces in the same grouping the enum uses (CORE, UI, LAYOUT, GLOBAL FEATURES, PAGES, META, BACKEND), with a one-line purpose for each. `INPUTS` (singular `INPUT` in code) is dropped from the convention — the namespace does not exist on disk.

**`VALIDATION` namespace is frozen.** No new keys are added to `VALIDATION`. Anything that would have gone there goes to `ERRORS` instead. Existing `VALIDATION` keys remain in place until a post-launch migration moves them all to `ERRORS`. Rationale: `ERRORS` and `VALIDATION` overlap in practice — both hold user-facing "this is wrong, fix it" strings. One namespace for everything red-on-screen is simpler than two.

Conventions Part 6 Rule 4 (the `product.<field>.<code_lowercase>` pattern) places keys in `ERRORS`. The queued `translationKey` backend brief produces keys for `ERRORS`.

**Reasoning:** the split between `VALIDATION` and `ERRORS` is conceptually defensible but breaks down in the product creation flow, where the same backend code can surface in either context. One bucket simplifies both authoring and consumption. Freezing now (rather than migrating now) avoids a multi-locale data migration during a feature push.

**Alternatives considered:** migrate all `VALIDATION` keys to `ERRORS` immediately (rejected — touches every locale in seed SQL and every consumer in web; scope explosion mid-feature). Keep both namespaces active and document the split (rejected — the split doesn't hold in practice, as the product validation case demonstrates). Drop `VALIDATION` from the enum entirely (rejected — existing keys still resolve through it; removing the enum value breaks live translations).

---

## 2026-05-13 — Session summary template gains "Obsoleted" and "Conventions check" sections

Two additions to the engineer session summary template in `meta/conventions.md` Part 5, between "Cleanup performed" and "Known gaps / TODOs":

**Obsoleted by this session.** What this session made dead but did not delete (stale tests, unreferenced code, contradictory docs). "Nothing" is a valid answer but must be written. Distinct from "Cleanup performed" — cleanup is what was deleted, obsoleted is what should have been deleted but was not, with reasons.

**Conventions check.** Explicit confirmation that the engineer checked Part 4 (cleanliness) and Part 6 (translations) against the work performed, plus any other part of conventions that applied. "N/A this session" is valid where a part does not apply (e.g., Part 6 on a session that touches no translations).

Conventions Part 4 gains a line referring to the new template section, tying the dead-code rule to its diagnostic question.

**Reasoning:** Part 4's existing rule ("if a refactor obsoletes old code, the old code is deleted in the same session") relies on the engineer noticing what's obsolete. The "Obsoleted" section forces the noticing. The "Conventions check" section catches brief-vs-conventions drift — the calibrated-challenge rule covers the same ground but operates at chat time; the template covers the same ground at session-end time. Two checkpoints, one issue.

**Alternatives considered:** trust the calibrated-challenge rule alone (rejected — already failed once in observable conditions). Add a generic "I followed conventions" confirmation (rejected — too vague to fail honestly; named parts force engagement). Move the dead-code rule out of Part 4 into Part 5 entirely (rejected — Part 4 is the rules, Part 5 is the deliverable; they reinforce each other).

---

## 2026-05-13 — "Obsoleted by this session" added to engineer session template

Engineers explicitly answer "what does this session obsolete?" at the end of every session. The answer goes in a new section of the session summary template, between "Cleanup performed" and "Known gaps / TODOs."

The two sections are distinct: "Cleanup performed" is what was deleted in this session; "Obsoleted by this session" is what is now dead but was not deleted (with reasons), or "nothing." A refactor session that leaves a stale test file is the typical case — the test isn't a cleanup target of the refactor itself, but it's now dead, and the next session shouldn't have to rediscover it.

Conventions Part 4 gains one line tying the section to the existing dead-code rule. Conventions Part 5 gains the template section.

**Reasoning:** the existing Part 4 rule ("if a refactor obsoletes old code, the old code is deleted in the same session") relies on the engineer noticing what's obsolete. Adding the diagnostic question to the template forces the noticing.

**Alternatives considered:** add the question to brief-authoring habits only, with no convention change (rejected — habits drift across many briefs; the template is the durable place). Combine obsoleted-tracking into "Cleanup performed" (rejected — mixing what-was-done with what-should-be-done loses the signal).

---

## 2026-05-13 — Engineers required to challenge briefs, with calibrated threshold

All four engineer `CLAUDE.md` files rewritten to include a "Challenging the brief" section. Engineers must read code before writing code, compare brief to actual code, and surface real mismatches in a "Brief vs reality" section before implementing.

The challenge threshold is deliberately calibrated to filter noise. Engineers challenge real bugs only: trust boundary violations, missing or wrong code references, hidden dependencies, regression risks, wire-shape mismatches, off-counts. Engineers do not challenge naming, ordering, formatting, plural vs singular labels, casing, comment phrasing, or "this could be more elegant" thoughts.

Trust boundary check is now an explicit engineer responsibility. If a brief tells an engineer to violate a trust boundary, the engineer flags it CRITICAL and refuses to implement.

**Reasoning:** the `oldName` bug existed in the code, was listed in the backend engineer's audit, but was never challenged. Engineers read code and Mastermind does not — engineers are the last line of defense on brief-vs-reality discrepancies. The first pass at this rule was too broad and surfaced cosmetic issues (singular vs plural namespace labels) at the same priority as real bugs. The calibrated version explicitly excludes cosmetic categories.

**Alternatives considered:** leaving the responsibility entirely with Mastermind (rejected — Mastermind cannot read code, cannot catch this class of bug alone). Making it advisory rather than mandatory (rejected — advisory rules degrade under deadline pressure). Letting engineers challenge anything (rejected — produced noise on the validation feature when an engineer surfaced a singular/plural labeling difference as if it were a bug).

---

## 2026-05-13 — Feature lifecycle and trust boundary principle codified

Two new sections added to `meta/conventions.md`: Part 10 (Feature lifecycle) and Part 11 (Trust boundaries).

Part 10 codifies:

- One Mastermind chat per feature. When the feature ships, the chat closes.
- Five phases per chat: Intake, Audit, Seam analysis, Canonical spec, Engineering briefs.
- Engineers audit code as ground truth, not pre-existing reports or drafts.
- Pre-existing material is input only, never authoritative.

Part 11 codifies the server-as-trust-boundary principle, with the `oldName` bug as the originating example. A value used in a moderation, authorization, or state-transition decision must be derived from auth, read from the server's database, or otherwise unforgeable by the client. Client-supplied "before" values for change detection are forbidden.

**Reasoning:** Mastermind chats had grown polluted across multiple features. Reusing chats caused context drift, stale assumptions, and process erosion. The `oldName` bug exposed a trust boundary violation that no current convention prohibited. Both gaps closed at the convention level so future agents inherit the rules.

**Alternatives considered:** keeping the Feature Audit Phase as Mastermind-only behavior (rejected — engineers also need to know audits are required and what an audit brief looks like). Adding trust boundaries as a one-line note in the security section (rejected — highest-cost class of bug seen so far, deserves its own Part).

---

## 2026-05-13 — Mastermind bootstrap rewritten for per-feature chats

`meta/mastermind-bootstrap.md` fully replaced. New version defines the per-feature chat lifecycle, the five phases, the strictness rules (corrected from over-strict failure mode), the trust boundary rule, and explicit scope limits on what a chat does not do.

Removed from the old bootstrap: long-running Mastermind assumption, cross-feature institutional memory expectation.

Added: phase discipline, "current code is ground truth" rule, chat closes when feature ships, scope-creep prevention.

**Reasoning:** the previous bootstrap was built for one long Mastermind chat covering many features. Context polluted, decisions drifted, the same agent reasoned over stale assumptions. The new bootstrap forces a clean planning surface per feature.

**Alternatives considered:** patching the existing bootstrap to add the lifecycle phases (rejected — the long-running-chat assumption was load-bearing in the old structure; a patch would leave contradictions in the file). Two bootstraps, one for "complex features" and one for "simple bug fixes" (rejected — adds a classification decision before every chat; not worth the complexity).

## 2026-05-13 — Mastermind operating rules formalized

Four operating upgrades applied to Mastermind: strictness, writing style, self-contained instructions, lead-with-a-recommendation. Captured in meta/mastermind-bootstrap.md. New Mastermind chats bootstrap with this file alongside conventions.md, state.md, and decisions.md.

Alternatives considered: keeping the rules implicit in conventions.md (rejected — operating-style rules are about how an agent communicates, not about project content; mixing them clutters both files).

## 2026-05-13 — Feature spec is authoritative over sibling-repo docs

When a doc in a sibling code repo (`oglasino-backend/docs/`, `oglasino-web/docs/`, `oglasino-expo/docs/`) overlaps with a feature spec in `oglasino-docs/features/`, the feature spec wins. Engineers reading the sibling-repo doc must defer to the feature spec on any disagreement.

Added to `meta/conventions.md` Part 1 as a standing rule.

**Reasoning:** the cross-repo consolidation tracked in `future/cross-repo-docs-consolidation.md` will eventually migrate the overlapping docs into `features/`. Until then, engineers need a single, obvious tiebreaker. Without this rule, every overlap is a judgment call.

**Alternatives considered:** scope the rule only to the three currently-overlapping files (rejected — future feature specs will hit the same overlap problem). Migrate the overlapping docs now (rejected — already deferred to post-launch per `future/cross-repo-docs-consolidation.md`).

## 2026-05-13 — Agent architecture and conventions

### Stack: Claude Code per repo, Mastermind in desktop app

Igor uses Claude Code (already installed, v2.1.140) in each of `oglasino-backend`, `oglasino-web`, `oglasino-expo`, `oglasino-docs`. Mastermind runs as a long-running Claude Desktop chat. Igor is the message bus.

**Alternatives considered:** custom Node service against Anthropic SDK (rejected — duplicates Claude Code infrastructure, costs extra tokens on top of Max x20); LangGraph (overkill for 4 agents); Cursor/Windsurf (don't give clean Mastermind/Engineer separation).

### Engineers do not commit

Engineers stage changes on disk only. Igor reviews working-directory diffs in IntelliJ/VS Code (file-color differentiation in the diff view) and runs `git add` / `git commit` himself.

**Alternatives considered:** engineer commits to feature branch only (rejected — Igor prefers diff color review over commit review for catching issues).

### Mastermind reviews summaries, not raw diffs

Engineers produce structured session summaries in `.agent/last-session.md`. Mastermind reads only summaries. If something looks off, Mastermind asks for the relevant code via Igor.

**Reasoning:** prevents Mastermind's context from filling up with every line of code change. Igor explicitly wanted long-running Mastermind conversations.

### Image pipeline create flow: Model C

Hold images in memory through the wizard. Upload to R2 only at step 4. Progressive status messages during step 4 ("Uploading images..." → "Creating product..." → done).

**Alternatives considered:** Model A (upload at step 1, current behavior — rejected, accepts orphan leak on wizard abandonment); Model A+ with pending/promote pattern (rejected for now, deferred to v2 — too much new infrastructure for 1-month timeline).

### Content moderation v2 parked

The policy-scoring engine proposal (Signal with score+confidence, weighted aggregation, ALLOW/WARN/REVIEW/BLOCK tiers) is correct in theory but premature. Parked in [`future/content-moderation-v2.md`](future/content-moderation-v2.md).

**Reasoning:** no production traffic yet means no real abuse data to calibrate weights. Current binary rule system ships and works. Revisit 2–3 months post-launch.

### Translation namespaces are fixed; no parent/child key collision

Translations live in fixed namespaces (ERRORS, INPUTS, COMMON_SYSTEM, DASHBOARD_PAGES, PAGE_SPECIFIC). Keys cannot have parent-child collision (e.g. `some.key` and `some.key.suffix` cannot coexist) — use `.label` suffix on the parent.

**Reasoning:** the translation library rejects parent-child collision. This is a hard library constraint, not a stylistic choice.

### Pre-validate endpoint added at step 2 of create wizard

New `POST /api/secure/products/pre-validate` runs content moderation on `name` + `description` only. Web wizard calls it before advancing from step 2 to step 3. Returns HTTP 200 in both clean and violation cases; clients check `errors.length`.

**Reasoning:** current create UX punishes users by collecting errors only at final submit. Pre-validate at step 2 catches content issues early. Update flow doesn't need this — it's single-page.

### Repo-internal docs stay; new docs go to `oglasino-docs/`

Existing `<repo>/docs/` folders remain (getting-started, deployment, troubleshooting). No new files added there. All new documentation authored in `oglasino-docs/`.

**Reasoning:** long-term goal is single source of truth in `oglasino-docs/` for eventual Jira/Confluence migration. Moving existing docs is busywork; growing new docs in code repos perpetuates the split.

### Docs/QA's first session is a repo sweep

Before any engineering brief runs, Docs/QA inventories `oglasino-docs/` and proposes a reorganization plan (what to move, what to keep, what new folders to add). Igor approves before further work.

**Reasoning:** the docs repo has pre-existing structure I haven't fully audited. Adding `features/`, `sessions/`, `future/`, `legal/` blindly might collide with existing folders.
