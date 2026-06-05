# Documentation Centralization & Redesign Plan

A plan to make `oglasino-docs` the one place all Oglasino documentation lives **and** the
best version of itself — a coherent, well-structured reference that serves both AI agents and
humans. Leaving only a `README.md` in each code repo is the *outcome*; the *work* is designing
the right information architecture and filling it with the right docs.

**Author:** Docs/QA agent · **Date:** 2026-06-02 · **Status:** investigation / proposal — not yet executed.

> This is a **plan**, not an execution. Nothing has been moved, merged, deleted, or authored.
> It proposes changes to `meta/conventions.md` (Part 1, Part 2) and a `decisions.md` entry —
> substantive edits that require a Mastermind draft + Igor brief before they are applied
> (conventions Part 3). See [Conventions impact](#7-conventions-impact).

---

## 1. North star

Not "move files left to right." The goal is a documentation system where:

- **One concept = one canonical doc.** No duplicates, no drift. Everything else links to it.
- **Both audiences are first-class.** A human onboarding and an AI agent answering a question
  should each find the right doc fast and trust it.
- **Structure mirrors how people think about the system** — not how the repos happen to be split.
  Structural/architecture docs, feature docs, guides, reference, operations, and process each
  have a clear home.
- **The only doc in a code repo is its `README.md`**, and that README is a thin pointer back here.

What makes docs good for **AI** is mostly what makes them good for **humans**, plus a few extras:
a single explicit entry point and doc-map, consistent naming and status markers, one
authoritative source per topic (so retrieval doesn't surface three contradicting copies), and
tight cross-linking so a reader/agent can walk from any doc to its neighbors. This plan builds
for both at once.

This realizes the long-stated intent in [`conventions.md`](conventions.md) Part 1
("Long-term goal: one source of truth") and [`confluence-migration.md`](confluence-migration.md),
and goes one step further: the current convention *permits* repo-internal "how to work here"
docs to stay in `<repo>/docs/`. Igor's directive removes that allowance — a convention change.

## 2. Scope of this investigation

Per Igor's instruction, this swept **documentation only** — no source code in any repo, and not
this repo's `sessions/` archive. It covers every `.md`, `.mdx`, `.txt`, and report/spec/prompt
artifact outside `node_modules`, build output, `.git`, and `.agent/`, across all nine repos
(`backend`, `web`, `expo`, `router`, `firestore-rules`, `image-worker`, `landing`,
`maintenance`, `private`).

## 3. Current state — what's scattered where

| Repo | Doc surface found | Verdict |
|---|---|---|
| `oglasino-backend` | `docs/` (17 numbered guides + `README`), `jobs/image_pipeline/` (12 files), `jobs/user_delete_job/`, `firstPrompt.txt`, `output/extra_*.txt` | Heavy. Re-home `docs/`; triage `jobs/`; drop scratch `.txt`. |
| `oglasino-web` | `docs/` (7 numbered guides + `README`), `jobs/` (image_pipeline ×13, firestore_audit, rn_sync, user_delete_job, bugs, filtering audit), 2 `.txt` in `src/` | Heavy. Re-home `docs/`; triage `jobs/`; drop scratch. |
| `oglasino-router` | `docs/architecture.md`, `docs/operations.md` | Light. Re-home both. |
| `oglasino-image-worker` | `jobs/image_pipeline/` (7 files), `MIGRATION-NOTES.md` | Duplicate set. Drop dupes; re-home migration-notes. |
| `oglasino-expo`, `oglasino-firestore-rules`, `oglasino-landing`, `oglasino-maintenance` | `README.md` (+ `CLAUDE.md`) only | Already clean. |
| `oglasino-private` | secrets, AI prompts, `devops-blueprint-v4.md`, inbox notes | **Stays private.** Secret values never enter `oglasino-docs`. |

`CLAUDE.md` is agent instructions, not documentation — stays put, out of scope.

## 4. Key findings

1. **The same spec exists 4× and has already drifted.** `IMAGE-PIPELINE-SPEC.md` lives in
   backend, web, image-worker, and private — four copies, **four different checksums**. The
   "single" spec is silently diverging. A canonical [`features/image-pipeline.md`](../features/image-pipeline.md)
   already exists here. This is exactly the failure mode the redesign must prevent structurally.
2. **`USER_DELETION_SPEC.md` duplicated** across backend and web (drifted), while canonical
   [`features/user-deletion.md`](../features/user-deletion.md) already exists.
3. **Parallel numbered dev-doc trees** — `backend/docs/01..16` and `web/docs/01..07` mirror each
   other; no single cross-stack "how to develop locally."
4. **Some `docs/` topics already have canonical homes here** → migration is partly *merge*, not
   *move* (per-repo deployment ↔ [`infra/runbooks/`](../infra/runbooks/); env-vars ↔
   [`secret-inventory.md`](../infra/overview/secret-inventory.md); redis ↔ [`infra/redis.md`](../infra/redis.md)).
5. **Job-artifact sprawl** — `jobs/**` audits, reports, NEEDS-BACKEND/FRONTEND handoffs, prompts.
   Point-in-time working artifacts mostly superseded by shipped feature specs. Triage, don't bulk-move.
6. **Scratch `.txt` committed in code** (`firstPrompt.txt`, `output/extra_*.txt`,
   `favoritesText.txt`, `firestoreDatabase.txt`) — not documentation; delete.
7. **Drift the scatter hides:** `web/docs/README` says "Next.js 16"; [`conventions.md`](conventions.md)
   Part 9 says "Next.js 15". One is wrong.
8. **`oglasino-private` must be quarantined** — real secret values. Only secret *names* belong
   here, in [`secret-inventory.md`](../infra/overview/secret-inventory.md).
9. **`devops-blueprint`: v5 here, v4 in private** — confirm v4 dead, drop it.
10. **There is no single entry point and no domain glossary.** A newcomer (human or AI) has no
    "start here," no map of the system, no shared vocabulary. This is a structural gap, not a
    scattered file — see §6.

## 5. Target information architecture

Document **types** the system needs, and the top-level space each lives in. The existing tree
(`infra/`, `features/`, `future/`, `design/`, `legal/`, `meta/`, `sessions/`) is largely right;
the redesign adds an explicit **orientation layer**, a **structural/architecture layer**, a
**guides layer**, and a **reference layer**, and names what each is for.

| Doc type | Question it answers | Space |
|---|---|---|
| **Orientation** — index / "start here", system map, domain glossary | "Where do I begin? What does this word mean?" | root + `meta/` |
| **Structural / architecture** — system architecture, data model, request lifecycle, auth & trust boundaries, cross-repo contracts (error, translation seam, image pipeline), per-repo internal architecture | "How is it built and how do the pieces fit?" | `architecture/` *(new)* |
| **Feature** — one spec per feature: what the product does, acceptance criteria, status | "What does this feature do and is it done?" | `features/` *(exists)* |
| **Guides** — onboarding, local dev, per-repo "how to work here", troubleshooting | "How do I run/develop/debug this?" | `guides/` *(new)* |
| **Reference** — env/secret names, translation namespaces, stack, endpoints, schema | "What's the exact value/name/list?" | `infra/` + `reference/` *(new, as needed)* |
| **Operations** — infra inventory, runbooks, deploy | "How do I operate prod?" | `infra/` *(exists)* |
| **Process / meta** — conventions, agent workflow, migration plans, this plan | "How do we work and document?" | `meta/` *(exists)* |
| **Design / QA**, **Legal**, **Future** | QA topics, legal drafts, parked work | `design/` · `legal/` · `future/` *(exist)* |

Proposed new top-level directories:

```
oglasino-docs/
  index.md            # NEW — single "start here": what Oglasino is, the system map, links into every space
  architecture/       # NEW — structural docs: system overview, data model, request lifecycle,
                      #   auth & trust boundaries, cross-repo contracts, per-repo architecture
  guides/             # NEW — developer/onboarding "how to work here"
    backend/ web/ router/ expo/ + README (index)
  reference/          # NEW (if needed) — flat lookup docs: glossary, translation namespaces,
                      #   stack reference, endpoint catalogue
```

Each top-level directory remains a clean Confluence space ([`confluence-migration.md`](confluence-migration.md)).

**Conventions that make it work for AI + humans** (to fold into `meta/conventions.md` Part 1 via
the Mastermind draft):
- Every doc starts with a one-line **purpose** and a **status/authoritative** marker.
- One canonical doc per concept; duplicates become links.
- `index.md` and each space's `README` are kept as live maps.
- Cross-link liberally; relative links only (already required).

## 6. Docs that should exist but don't (authoring work)

Centralization isn't only relocation — these are **new docs to author/synthesize**, the part
that makes the repo *good* rather than merely *consolidated*. Because Docs/QA does not read code
and must not invent facts, the technical ones depend on engineer audits / Mastermind drafts as
source material; Docs/QA can own structure, orientation, and assembly.

| New doc | Why | Source of facts |
|---|---|---|
| `index.md` — start-here + system map | No single entry point exists today | Docs/QA can draft from existing docs |
| `reference/glossary.md` — domain & platform terms (base site, free zone, pre-validate, maintenance split, …) | Shared vocabulary for humans and AI; resolves ambiguous terms | Docs/QA assembles; Igor/Mastermind confirm |
| `architecture/system-overview.md` — the canonical architecture doc with a Mermaid map | Per-repo architecture docs exist but no whole-system one | Engineer audits / Mastermind |
| `architecture/cross-repo-contracts.md` — error contract, translation seam, image-pipeline contract, auth/trust boundary in one place | These are scattered across `conventions.md` parts and repo job docs | Mastermind (mostly already in conventions Parts 6–11 + feature specs) |
| `architecture/data-model.md`, `request-lifecycle.md` | Foundational structural docs newcomers always need | Engineer audits / Mastermind |
| Thin per-repo `README.md` rewrite | After re-homing, each repo README points here | Owning engineer agent |

## 7. Migration mapping (existing content → the new IA)

Actions: **move** (relocate, kebab-case), **merge** (fold into a canonical doc), **supersede**
(canonical exists; drop copy), **archive** (one-time artifact → `sessions/`/`future/`), **drop**
(not documentation), **quarantine** (stays in `oglasino-private`).

| Source | Action | Target |
|---|---|---|
| `backend/docs` getting-started, local-development, troubleshooting, pre-commit, db-reset | move | `guides/backend/` |
| `backend/docs` architecture, database/redis/elasticsearch overviews, migrations, seed | move | `guides/backend/` + reconcile with `architecture/` and [`infra/redis.md`](../infra/redis.md) |
| `backend/docs` environment-variables, deployment, pre-lunch-tasks, deploy-checklist | merge | [`secret-inventory.md`](../infra/overview/secret-inventory.md), [`infra/runbooks/`](../infra/runbooks/), [`master-plan.md`](../infra/master-plan.md) |
| `backend/docs/16-image-pipeline`, `*/jobs/image_pipeline/**` (all 4 repos) | supersede | [`features/image-pipeline.md`](../features/image-pipeline.md) (+ mobile test cases spec) |
| `web/docs/01..07` | move / merge | `guides/web/`; deployment+env-vars+pre-lunch → `infra/` |
| `router/docs/architecture.md`, `operations.md` | move | `guides/router/` + cross-check [`infra/cloudflare/workers.md`](../infra/cloudflare/workers.md) |
| `*/jobs/user_delete_job/USER_DELETION_SPEC.md` | supersede | [`features/user-deletion.md`](../features/user-deletion.md) |
| `web/jobs/{firestore_audit,rn_sync,bugs,audit-filtering-and-search}` | archive / drop | keep only if it records a decision not already in `decisions.md` |
| `image-worker/MIGRATION-NOTES.md` | move | `architecture/` or `infra/cloudflare/workers.md` (decide on read) |
| `*/firstPrompt.txt`, `*/jobs/**/*_PROMPT.txt`, `output/extra_*.txt`, `web/src/**/*.txt` | drop | not documentation — owning engineer deletes |
| `private/03-AI/oglasino-devops-blueprint-v4.md` | drop | superseded by [`v5`](../infra/oglasino-devops-blueprint-v5.md) |
| `private/01-secrets/**`, `private/05-RANDOM/**` | quarantine | stays private; names only in [`secret-inventory.md`](../infra/overview/secret-inventory.md) |

## 8. Conventions impact

When executed, this requires these edits — **Mastermind must draft them**:

- **Part 1** — replace the "until consolidation happens / `<repo>/docs/` may remain" block: the
  only doc in a code repo is its `README.md`. Add the dual-audience doc rules from §5 (purpose
  line, status marker, one-canonical-per-concept, keep indexes live).
- **Part 2** — add `oglasino-image-worker`, `oglasino-landing`, `oglasino-maintenance` to the
  repo-layout table; add the `index.md`, `architecture/`, `guides/`, `reference/` directories.
- **`decisions.md`** — a new entry recording the centralization + IA redesign and the new spaces.
- **`README.md` + `confluence-migration.md`** — add the new top-level directories.

None applied yet. Listed so the Mastermind draft is complete.

## 9. Suggested execution order

1. **Agree the IA** (§5) and the gap list (§6) with Igor + Mastermind.
2. **Conventions + decisions draft** (Mastermind) → Docs/QA applies → Igor commits.
3. **Stand up the skeleton** — `index.md`, `architecture/`, `guides/`, `reference/` with index READMEs.
4. **Author the orientation + structural docs** (§6), sourcing technical facts from engineer
   audits / Mastermind — never invented.
5. **Re-home existing content** repo by repo (router → web → backend) per §7, kebab-casing in.
6. **Thin the code-repo READMEs and delete `<repo>/docs/`, `jobs/`, scratch `.txt`** — owned by
   each **engineer agent** under its own brief (Docs/QA cannot write code repos beyond `.agent/`).
7. **Revalidate** — fix the Next.js 15/16 drift, dead links, cross-links now pointing here.

## 10. Open questions for Igor

1. **Top-level names** — `architecture/`, `guides/`, `reference/`, `index.md`: good, or different?
2. **Authoring depth** — how far do we go on the new structural docs (§6) now vs. stub-and-fill
   as engineer audits land? Some need facts only the engineer agents/Mastermind have.
3. **Code-repo cleanup ownership** — confirm deleting `<repo>/docs/`, `jobs/`, scratch `.txt`,
   and thinning READMEs is a per-repo engineer brief (outside Docs/QA's write access).
4. **`jobs/**` triage bar** — keep an artifact only if it records a decision/contract not yet in
   a spec or `decisions.md`; drop the rest. Confirm.
5. **`oglasino-private`** — confirm fully out of scope (secrets stay; only dead v4 blueprint touched).

## 11. What this plan does not do

- It does not move, merge, delete, or author any file.
- It does not edit any of the four config files.
- It does not reach into code repos; code-repo cleanup is a separate per-repo engineer brief.
- The conventions/decisions edits in §7 are **flagged for Mastermind**, not applied. Technical
  content for the new structural docs (§6) must come from engineer audits — never invented here.
