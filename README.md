# Oglasino Docs

Single source of truth for all Oglasino documentation — infrastructure
today, product/design/business material as it accumulates. Eventually
migrates to Confluence (see [`meta/confluence-migration.md`](meta/confluence-migration.md)).

## Top-level structure

| Directory | What lives here |
|---|---|
| [`infra/`](infra/) | Infrastructure — DigitalOcean, Cloudflare, Firebase, Vercel, Apple, Google Play, Expo, GitHub, Namecheap, runbooks. Start at [`infra/master-plan.md`](infra/master-plan.md). |
| [`product/`](product/) | Product specs, feature briefs, roadmap notes. |
| [`design/`](design/) | Design decisions, UX writing, visual conventions. |
| [`business/`](business/) | Business documents — entity formation, contracts, finance. |
| [`meta/`](meta/) | Documentation about the documentation — conventions, migration plan. |

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
