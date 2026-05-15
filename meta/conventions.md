# Conventions

The shared rulebook for everyone working on Oglasino — Igor, Mastermind, the engineer agents, and the docs agent. This file governs both documentation style and engineering work.

When a rule here conflicts with a request, the agent stops and asks Igor. Conventions change only via an entry in [`../decisions.md`](../decisions.md).

---

## Part 1 — Documentation style

### Markdown

GitHub-flavored markdown. ATX headings (`#`, `##`, `###`) only — never underlined headings. For URLs that repeat throughout a page, use reference-style links.

### File naming

- kebab-case
- lowercase
- `.md` extension
- The filename is the document's slug. No spaces, no underscores, no capitals.

### Page titles

Every doc starts with `# <Title Case Title>` matching the file's purpose.

### Diagrams

Mermaid in fenced ` ```mermaid ` blocks. Renders natively on GitHub.

### Images

Screenshots and other images are referenced as if the file already exists, even when it doesn't yet. Igor supplies the actual files after the doc is written.

- **Filename:** kebab-case, lowercase, descriptive of content — `product-creation-step-2.png`, not `screenshot1.png`. The filename should tell a reader what the image shows without opening it.
- **Location:** images live in an `assets/` folder next to the doc that references them. A doc at `features/product-validation.md` references `assets/product-creation-step-2.png`.
- **Reference format:** standard markdown image syntax with descriptive alt text: `![Product creation wizard, step 2](assets/product-creation-step-2.png)`.
- **Accompanying comment:** immediately above each image reference, add an HTML comment describing what the image should show. This is a note for whoever supplies the file later, not reader-facing content. Scale the comment to the image: one line for a simple screenshot, several for a complex one.

Example:

​```markdown

<!-- Screenshot: the create-product wizard on step 2, with the name and
     description fields filled and a validation error showing under the
     name field. Error state must be visible. -->

![Product creation wizard, step 2 with validation error](assets/product-creation-step-2.png)
​```

### Sensitive data

**Never commit credentials, API keys, or secret values to this repo.** Document secret _names_ in [`../infra/overview/secret-inventory.md`](../infra/overview/secret-inventory.md); values live in GH Secrets / Vercel env / Firebase / wherever the secret is consumed.

### Status indicators

Use the same syntax as [`../master-plan.md`](../master-plan.md) for any checklists added later:

- `[ ]` = not started
- `[~]` = in progress
- `[x]` = complete
- `[!]` = blocked (add a note explaining what's blocking)

### Cross-references

Relative links between files (e.g., `[Firebase projects](../infra/firebase/projects.md)`). Never link by absolute URL to this repo's GitHub view — relative paths survive forks, renames, and the eventual Confluence migration.

### Long-term goal: one source of truth

Eventually all developer documentation across Oglasino consolidates here. Backend's `docs/`, web's `docs/`, mobile's `docs/` — all migrate into `oglasino-docs/`. The future migration to Jira or Confluence will export from this repo alone.

**Until that consolidation happens:**

- **Repo-internal "how to work in this repo" docs** (getting-started, local-dev, troubleshooting) may remain in `<repo>/docs/` so developers find them next to the code.
- **All new feature specs, cross-cutting docs, and shared references** are authored _here_ in `oglasino-docs/` from day one.
- **No agent creates new documentation files in `<repo>/docs/`.** New work goes into `oglasino-docs/`. Existing files in `<repo>/docs/` can be edited or deleted, but the folder doesn't grow.
- **The feature spec wins.** Where a sibling-repo doc (`oglasino-backend/docs/`, `oglasino-web/docs/`, `oglasino-expo/docs/`) overlaps with a feature spec in `oglasino-docs/features/`, the feature spec is authoritative. On any disagreement, defer to the feature spec.

---

## Part 2 — Repo layout

Local working tree:

```
~/code/oglasino/
  oglasino-backend/      Spring Boot, Hibernate, Postgres, Firebase Admin
  oglasino-web/          Next.js 15 App Router, React TS, Tailwind, Zustand
  oglasino-expo/         Expo, React Native
  oglasino-docs/         this repo — shared brain, specs, conventions
```

Engineers in each code repo have **read access** to `../oglasino-docs/` as a sibling directory. They never write to docs from their working repo. Docs are written by the Docs/QA agent in its own repo.

---

## Part 3 — How agents work

### The five agents

- **Mastermind** — runs in Claude Desktop. Plans, decides, reviews engineer session summaries, challenges bad ideas. Never reads or writes code.
- **Backend Engineer** — runs as Claude Code in `oglasino-backend`. Spring/Java only.
- **Web Engineer** — runs as Claude Code in `oglasino-web`. Next.js/TS only.
- **Mobile Engineer** — runs as Claude Code in `oglasino-expo`. Expo/RN only. Waits for backend and web to stabilize before touching a feature.
- **Docs/QA** — runs as Claude Code in `oglasino-docs`. Maintains specs, session archives, QA topics, legal drafts.

### Branching

```
feature/<slug>      long-running feature branches
fix/<slug>          small bug fixes
chore/<slug>        non-feature housekeeping
```

The slug matches `oglasino-docs/features/<slug>.md` where applicable.

### Hard rules — every engineer, every session, no exceptions

- **No `git commit`.** Engineers stage changes on disk only. Igor commits.
- **No `git push`, `git merge`, `git rebase`, or `git checkout` to a different branch.** Engineers stay on the branch Igor has checked out.
- **No deploys.** `vercel deploy`, `firebase deploy`, `maven bootRun` against any non-local target, `gh release` — all forbidden.
- **No destructive DB operations.** Read-only `psql` against local DB only.
- **No Firestore security rule edits** without an explicit instruction in the brief.
- **No cross-repo edits.** Backend Engineer doesn't touch `oglasino-web`. Web Engineer doesn't touch `oglasino-backend`. If a task seems to require it, stop and tell Igor.
- **No new files in `<repo>/docs/`.** New documentation goes in `oglasino-docs/`.

### Mastermind hard rules

- Never reads or writes code.
- Never references an engineer's chat directly — only summary files.
- Never approves a session summary that lacks an explicit "Cleanup performed" entry.
- Never lets engineering work begin on a feature without a corresponding `features/<slug>.md` spec.

### Docs/QA hard rules

- Works only in `oglasino-docs`.
- Never edits `decisions.md` or `conventions.md` (those are Igor's, drafted with Mastermind).
- Never invents facts. If a session summary is ambiguous, asks Igor.

---

## Part 4 — Cleanliness — task is not done until

Non-negotiable. The session summary's "Cleanup performed" section lists what was done; "none needed" is a valid value but must be explicitly written.

- No commented-out code left behind. Git history is the just-in-case archive.
- No unused imports, variables, functions, classes, or files.
- No `console.log`, `System.out.println`, `print`, or other ad-hoc debug logging added during the task. Logger calls that fit the existing logging strategy are fine.
- No `TODO` or `FIXME` comments added without a matching entry in the session summary's "Known gaps."
- No new files created that aren't referenced by something.
- No formatter or linter violations introduced:
  - Backend: `./mvnw spotless:check` and `./mvnw test` for touched modules
  - Web: `npm run lint`, `npx tsc --noEmit`, and `npm test` for touched paths
  - Mobile: same as web, plus `npx expo-doctor` when relevant
- If a refactor obsoletes old code, the old code is deleted in the same session. Not left "for later."
- Engineers explicitly answer the question "what does this session obsolete?" in the "Obsoleted by this session" section of the summary. "Nothing" is valid but must be written.

## Obsoleted by this session

- [things this session made dead — stale tests, unreferenced code, contradictory docs, duplicate validators]
- [for each: deleted in this session / left for follow-up with reason / cannot delete because of X]
- (or: nothing)

## Conventions check

- Part 4 (cleanliness): [confirmed / one violation flagged in "For Mastermind"]
- Part 4a (simplicity) / Part 4b (adjacent observations): [confirmed / N/A / flagged in "For Mastermind"]
- Part 6 (translations): [confirmed / N/A this session / one violation flagged]
- Other parts touched: [list any other part of conventions that applied to this session's work, e.g. "Part 7 (error contract) — confirmed"]

---

## Part 4a — Simplicity

Complexity that earns its place stays. Complexity that doesn't, doesn't. The judgment is the engineer's, but bias toward simpler when in doubt. The session summary's "For Mastermind" section is where complexity decisions get explained.

Guidelines:

- **Abstractions earn their introduction.** A new interface, base class, helper function, or wrapper should solve a concrete problem visible today, not a hypothetical one. If introduced for one caller with a second caller plausibly coming, that's a defensible call — name the second caller in "For Mastermind." If introduced "in case we need it," remove it.
- **Configuration is for values that vary.** A value that differs across environments, locales, tenants, or experiments belongs in config. A value that has exactly one setting today and no foreseeable second setting belongs as a constant. The dividing line is "will someone plausibly change this without changing the code?" If yes, config. If no, constant.
- **Match the surrounding code's style.** If the file uses a pattern, use the same pattern. Introducing a parallel "better" pattern alongside the existing one creates two ways to do the same thing, which is worse than either way alone. If the existing pattern is genuinely wrong, flag it for refactor — don't drop a competing pattern next to it.
- **Defensive code where the contract is loose; trust where it's tight.** Validate inputs at trust boundaries (controllers, external APIs, deserialized JSON). Don't re-validate downstream of a layer that already did. If a parameter is `@NonNull`, don't null-check it.
- **Comments explain why.** Code is the "what." Comments are for surprises, edge cases, or business rules that aren't obvious from reading.

Engineers explain non-obvious choices in the session summary's "For Mastermind" section. "I added an interface here because the upcoming mobile work will be the second caller" is a fine explanation. So is "I hardcoded the limit because no one's asked it to change in three years." Both are reasoned. Both are fine. The absence of explanation is the problem, not the choice itself.

---

## Part 4b — Adjacent observations

Engineers don't fix bugs outside their brief's scope. They do flag them. If during a session an engineer notices:

- a bug that's one step outside the current scope
- code that contradicts conventions
- a stale comment, dead import, or commented-out block in a file they touched
- a test that no longer asserts what its name claims
- a TODO that has clearly been done
- a contract mismatch between two files
- anything that makes a future engineer's life harder

…they flag it. The flag goes in the session summary's "For Mastermind" section, with:

- one-line description
- file path
- severity guess (low / medium / high) — low if cosmetic, medium if it could mislead a future reader, high if it could cause user-facing bugs
- "I did not fix this because it is out of scope" or similar

Mastermind decides what to do with each flag. Some go into `issues.md`, some get rolled into the next brief, some get triaged as "wontfix."

The rule is not "fix everything you see." The rule is "see everything you can see."

---

## Part 5 — Session summary template

Every engineer writes this session summary to **two** files at the end of every session:

1. `.agent/yyyy-mm-dd-<repo>-<slug>-<n>.md` — the permanent, uniquely-named record.
2. `.agent/last-session.md` — an exact copy of the same content. This is the predictable path Igor and existing briefs use to grab the latest summary.

Both files have identical content. `last-session.md` is always a duplicate of the most recent named file.

**Determining `<n>` (the order number):**

- `<n>` is sequential per `(repo, slug)` pair — not per day. A feature with 10 sessions in one repo is numbered `-1` through `-10` in order, regardless of how the dates fall.
- The engineer determines `<n>` by listing its own `.agent/` folder for files matching `*-<slug>-*.md`, taking the highest existing order number, and adding one. If no such file exists, this is the first session for that slug and `<n>` is `1`.
- Numbering is independent per repo. `oglasino-backend`'s `validation-refactor` sessions are `oglasino-backend-validation-refactor-1`, `-2`, `-3`; `oglasino-web`'s are `oglasino-web-validation-refactor-1`, `-2`, `-3`, counted separately. The repo name in the filename prevents collision. An engineer cannot see another repo's `.agent/` folder, so per-repo counting is the only scheme that works.
- The date in the filename is informational. It does not drive the number.

**Archiving:** Docs/QA copies the named file to `oglasino-docs/sessions/`. The file is already correctly named, so the archive is a straight copy — no rename.

This naming rule applies to sessions going forward. Existing `last-session.md` files from before this rule are not backfilled — overwritten history cannot be recovered. The first session run under the new rule for any given `(repo, slug)` finds no matching `*-<slug>-*.md` files and correctly starts at `-1`.

```markdown
# Session summary

**Repo:** oglasino-backend
**Branch:** feature/validation-refactor
**Date:** 2026-05-13
**Task:** [one line, copied verbatim from the brief]

## Implemented

- [3–5 bullets, plain English, what changed and why]

## Files touched

- path/to/File.java (+45 / -12)
- path/to/Other.java (+8 / -3)

## Tests

- Ran: ./mvnw test :module:foo
- Result: 274 passed, 0 failed
- New tests added: PreValidateEndpointTest

## Cleanup performed

- Removed 3 commented-out blocks in ImageService
- Deleted unused ImageLegacyMapper.java
- (or: none needed)

## Known gaps / TODOs

- [things deliberately not done, with reason]
- (or: none)

## For Mastermind

- [questions, risks, suggested next steps]
- (or: nothing flagged)
```

The "For Mastermind" section is the only place an engineer is allowed to speak to Mastermind. Everything else is factual.

---

## Part 6 — Translations

Translations are the seam where backend and frontend agree on user-visible text.

### Rule 1 — Namespaces are fixed

## Translation namespaces

Translations live in a fixed set of namespaces, mirroring the `TranslationNamespace` enum in `oglasino-backend`. Agents do not invent new namespaces. If a translation seems to belong nowhere, the agent stops and asks Igor.

## CORE

- `COMMON` — generic shared strings used across the app
- `COMMON_SYSTEM` — system-wide UI strings (buttons, generic actions)
- `ERRORS` — anything red-on-screen the user needs to fix or acknowledge: validation failures, server failures, "something went wrong" messages, prescriptive input feedback. All new error-like keys go here.
- `VALIDATION` — **frozen.** Existing user-input validation keys. No new keys are added. Anything new that would have gone here goes to `ERRORS`. Existing keys will migrate to `ERRORS` post-launch.
- `PAGING` — pagination controls and labels

## UI

- `BUTTONS` — button labels not covered by `COMMON_SYSTEM`
- `INPUT` — form labels, placeholders, input help text
- `DIALOG` — modal and dialog strings

## LAYOUT

- `HEADER` — site header strings
- `FOOTER` — site footer strings
- `NAVIGATION` — primary and secondary navigation strings

## GLOBAL FEATURES

- `INTRO` — onboarding / first-visit strings
- `EXTRA_PRODUCTS` — related/suggested product strings
- `COOKIES` — cookie consent strings

## PAGES

- `MESSAGES_PAGE` — strings used only on the messages page
- `DASHBOARD_PAGES` — owner dashboard strings
- `ADMIN_PAGES` — admin-only strings
- `ABOUT_PAGE` — about page strings
- `FREE_ZONE_PAGE` — free zone page strings
- `PRICING_PAGE` — pricing page strings

## META

- `METADATA` — page titles, descriptions, OG tags

## BACKEND

- `BACKEND_TRANSLATIONS` — strings the backend emits directly to the user, bypassing the frontend's translation layer. Push notification bodies, email content, anything where the client has no opportunity to translate before display.

### Rule 2 — Unique keys, no parent/child collision

These two keys **cannot coexist**:

```
some.translation
some.translation.with.suffix
```

The first is treated as a leaf string; the second tries to nest under it. The translation library rejects this.

Resolution: append a suffix to the parent key. Preferred suffix is `.label`. Other suffixes (`.text`, `.title`) are acceptable when `.label` doesn't fit semantically. Agent picks the suffix that reads best, but must use one — never leave the parent as a leaf.

### Rule 3 — Backend agent appends to existing SQL file

Translations are seeded via the SQL file in the backend repo.

When adding translations:

1. Find the existing namespace group in the SQL file.
2. Append new rows at the **end of that group**, before the next namespace starts.
3. Use the next available ID. If the next ID would collide with an existing row, the agent stops and reports the collision. Does not silently overwrite.
4. Maintain alphabetical order within a namespace group when reasonable. Skip if the existing group isn't already alphabetized.

### Rule 4 — Translation key pattern for validation errors

Backend error codes map to translation keys via:

```
product.<field>.<code_lowercase>
```

Examples:

- `NAME_BANNED_WORDS` → `product.name.banned_words`
- `DESCRIPTION_SPAMMY` → `product.description.spammy`

Frontend never receives message text from backend. Backend sends `{field, code}`. Frontend translates.

---

## Part 7 — Error contract

### Wire shape

```json
{
  "errors": [{ "field": "name", "code": "NAME_BANNED_WORDS" }]
}
```

- `field` is `null` for object-level violations.
- First error per field wins on the client.

### HTTP status codes

| Status | When                                                          |
| ------ | ------------------------------------------------------------- |
| 200    | Success, or the pre-validate "violations are not errors" case |
| 400    | Jakarta constraint failed (required, length, type)            |
| 422    | Business rule failed after Jakarta passed                     |
| 403    | Auth-related (not owner, not logged in)                       |
| 429    | Rate limit exceeded                                           |
| 500    | Genuine server error — never used for validation              |

If validation logic produces a 500, that's a bug.

---

## Part 8 — Architectural defaults

- **Routes are reusable across web and mobile.** Mobile-specific routes need justification in the feature spec.
- **Server is the single source of truth for content moderation.** Client-side checks are structural only (length, required, type).
- **Errors are codes, never messages.** Backend never sends message text on validation failures.
- **Direct-to-R2 for image uploads.** Backend issues a push token, frontend uploads directly. Never proxy image bytes through backend.
- **Feature branches are long-lived but mergeable.** Avoid letting a feature branch drift more than 2 weeks behind `main`.

---

## Part 9 — Stack reference

| Layer   | Tech                                                                                                     |
| ------- | -------------------------------------------------------------------------------------------------------- |
| Backend | Spring Boot 4 / Spring 7, Java 21, Maven, JPA/Hibernate, PostgreSQL, Elasticsearch, Redis, Cloudflare R2 |
| Web     | Next.js 15 App Router, React 18 TypeScript, Zustand, Tailwind, next-intl, axios, Zod                     |
| Mobile  | Expo, React Native, TypeScript                                                                           |
| Auth    | Firebase Auth (JWT)                                                                                      |
| Hosting | Vercel (web), DigitalOcean (backend), Cloudflare (CDN + R2)                                              |
| Storage | Postgres (primary), Firestore (some realtime), R2 (images)                                               |
| Locales | EN, SR, RU. Montenegrin (me/cnr) aliases to SR.                                                          |

---

## Part 10 — Feature lifecycle

Every feature follows the same lifecycle. Mastermind runs the planning; engineers do the work; Igor commits and decides.

Phase 1 — Intake (Mastermind chat)
Igor opens a new Mastermind chat for the feature. He pastes any pre-existing material: reports, drafts, prompts, chat excerpts. Pre-existing material is input only. It is not authoritative. Mastermind discusses, asks questions, forms a working understanding. No spec, no briefs in this phase.

Phase 2 — Audit (engineers, read-only)
Mastermind drafts one audit brief per affected repo. Each engineer audits its own repo for the feature's current state — endpoints, fields, validators, trust boundaries, seams with other repos. Engineers do not read pre-existing reports. They read code. Output: .agent/audit-<feature-slug>.md in each repo. Igor pastes back to Mastermind.

Phase 3 — Seam analysis (Mastermind chat)
Mastermind reads the audits side by side. Identifies contradictions, mismatches, and trust boundary violations. Presents findings to Igor with recommended resolutions. Igor confirms or overrides.

Phase 4 — Canonical spec (Mastermind drafts; Igor commits)
Mastermind drafts oglasino-docs/features/<slug>.md reflecting the audited reality plus the resolved seams. Spec is what the feature will be after engineering work, not what it was.

Phase 5 — Engineering briefs (Mastermind drafts; engineers execute)
One brief per session. Order typically: backend, web, mobile, docs cleanup. Each session ends with .agent/last-session.md. Igor reviews, brings summary to Mastermind, Mastermind verdicts, next brief.

##Part 11 — Trust boundaries

The server is the trust boundary.
A value used in a moderation, authorization, or state-transition decision must be one of:

Derived from auth (user identity, roles, claims from the JWT).
Read from the server's database, Firestore, or other authoritative store.
A value the client cannot misrepresent (a foreign key the server validates against its own data).

Client-supplied "before" or "previous" values for change detection are forbidden. The server compares the incoming new value to its own stored version.
This rule exists because the backend was caught trusting client-supplied oldName for change-detection on the validation feature. The fix removed the field from the request DTO and moved comparison server-side.

Every audit must explicitly check trust boundaries for the feature being audited. Every spec must state, for each field on a request DTO, whether it is trusted from the client, derived server-side, or read from the DB.

## When in doubt

Stop and ask Igor.
