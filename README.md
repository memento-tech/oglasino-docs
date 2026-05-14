# Oglasino Docs

Single source of truth for all Oglasino documentation — infrastructure,
feature specs, design/QA topics, legal drafts, and session archives.
Eventually migrates to Confluence (see [`meta/confluence-migration.md`](meta/confluence-migration.md)).

## Top-level structure

| Directory | What lives here |
|---|---|
| [`infra/`](infra/) | Infrastructure — DigitalOcean, Cloudflare, Firebase, Vercel, Apple, Google Play, Expo, GitHub, Namecheap, runbooks. Start at [`infra/master-plan.md`](infra/master-plan.md). |
| [`features/`](features/) | Feature specs (one `.md` per feature). |
| [`sessions/`](sessions/) | Archived engineer session summaries. |
| [`future/`](future/) | Deferred / parked work, kept for reference. |
| [`design/`](design/) | QA topic pages (one `.md` per topic). |
| [`legal/`](legal/) | Pre-lawyer drafts of Privacy Policy, Terms, etc. |
| [`meta/`](meta/) | Documentation about the documentation — conventions, migration plan. |

## Root-level files

| File | What it is |
|---|---|
| [`state.md`](state.md) | Single source of truth for where the project is right now. |
| [`decisions.md`](decisions.md) | Append-only log of decisions. |
| [`issues.md`](issues.md) | Append-only log of out-of-scope findings flagged by engineer sessions. |

## Editing conventions

See [`meta/conventions.md`](meta/conventions.md) for markdown style,
file naming, status indicators, and rules around sensitive data.

## Migration intent

This repo is the staging ground while content is small and the team is
one person. Once non-technical contributors need to write docs, or the
company adopts Confluence for other reasons, the directory tree maps
cleanly onto Confluence spaces. See
[`meta/confluence-migration.md`](meta/confluence-migration.md) for the
trigger conditions and migration plan.
