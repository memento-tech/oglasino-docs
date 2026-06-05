# Oglasino Docs

The single source of truth for Oglasino documentation and the shared brain for the project's
multi-agent workflow — infrastructure, feature specs, design/QA topics, legal drafts, decision
log, and engineer session archives. Markdown only, no code.

Part of the **Oglasino** platform. The sibling code repos read this repo for specs and
conventions; only the Docs/QA agent writes here.

---

## The four config files

These root-level files are load-bearing — every agent reads them at startup, and drift between
them and the code produces real bugs. The Docs/QA agent is their **sole writer**.

| File | What it is |
|---|---|
| [`meta/conventions.md`](meta/conventions.md) | The project rulebook — doc style, agent rules, error contract, translation namespaces, trust boundaries, schema patterns. |
| [`state.md`](state.md) | Where the project is right now — feature pipeline and statuses. |
| [`decisions.md`](decisions.md) | Append-only log of decisions (conventions/process are governed here). |
| [`issues.md`](issues.md) | Append-only backlog of out-of-scope findings flagged by engineer sessions. |

[`expo-status.md`](expo-status.md) is a living dashboard for the mobile (Expo) adoption effort.

## Top-level structure

| Directory | What lives here |
|---|---|
| [`infra/`](infra/) | Infrastructure — DigitalOcean, Cloudflare, Firebase, Vercel, Apple, Google Play, Expo, GitHub, Namecheap, runbooks. Start at [`infra/master-plan.md`](infra/master-plan.md). |
| [`features/`](features/) | Feature specs (one `.md` per feature). |
| [`sessions/`](sessions/) | Archived engineer session summaries (`yyyy-mm-dd-<repo>-<slug>-<n>.md`). |
| [`future/`](future/) | Deferred / parked work, kept for reference. |
| [`design/`](design/) | QA topic pages (one `.md` per topic). |
| [`legal/`](legal/) | Pre-lawyer drafts of Privacy Policy, Terms, etc. |
| [`meta/`](meta/) | Documentation about the documentation — conventions, Confluence migration plan, [doc centralization plan](meta/doc-centralization-plan.md). |

## How the workflow uses this repo

Oglasino is built by a set of single-repo agents — Backend, Web, Mobile, Router, Firestore
Rules, plus Docs/QA — coordinated through Mastermind (planning) with Igor as the message bus.
Engineer agents have **read** access to this repo for specs and conventions; they never write
here. Docs/QA maintains the four config files, archives session summaries into `sessions/`,
updates feature specs, and writes QA and legal drafts. The full lifecycle and agent rules are
in [`meta/conventions.md`](meta/conventions.md).

## Editing conventions

See [`meta/conventions.md`](meta/conventions.md) for markdown style, file naming, status
indicators (`[ ]` `[~]` `[x]` `[!]`), and the rule against committing sensitive data.

## Migration intent

This repo is the staging ground while content is small and the team is one person. When
non-technical contributors need to write docs — or the company adopts Confluence — the
directory tree maps cleanly onto Confluence spaces. See
[`meta/confluence-migration.md`](meta/confluence-migration.md) for trigger conditions and the
migration plan.

## Related repos

| Repo | Role |
|---|---|
| [`oglasino-backend`](../oglasino-backend) | Spring Boot API |
| [`oglasino-web`](../oglasino-web) | Next.js web client |
| [`oglasino-expo`](../oglasino-expo) | Expo / React Native mobile client |
| [`oglasino-router`](../oglasino-router) | Cloudflare Worker — edge routing & maintenance |
| [`oglasino-image-worker`](../oglasino-image-worker) | Cloudflare Worker — image PUT/GET against R2 |
| [`oglasino-firestore-rules`](../oglasino-firestore-rules) | Firestore security rules |
| [`oglasino-landing`](../oglasino-landing) · [`oglasino-maintenance`](../oglasino-maintenance) | Static "coming soon" / maintenance pages |

---

> `CLAUDE.md` governs the Docs/QA agent that works in this repo. Igor commits.
</content>
