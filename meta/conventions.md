# Documentation Conventions

## Markdown style

GitHub-flavored markdown. Use ATX headings (`#`, `##`, `###`) — never
underlined headings. For URLs that repeat throughout a page, use
reference-style links.

## File naming

- kebab-case
- lowercase
- `.md` extension

## Page titles

Every doc starts with `# <Title Case Title>` matching the file
purpose.

## Diagrams

Mermaid in fenced ` ```mermaid ` blocks. Renders natively on GitHub.

## Sensitive data

**Never commit credentials, API keys, or secret values to this repo.**
Document secret **names** in
[`../infra/overview/secret-inventory.md`](../infra/overview/secret-inventory.md);
values live in GH Secrets / Vercel env / Firebase / wherever the
secret is consumed.

## Status indicators

Use the same syntax as
[`../infra/master-plan.md`](../infra/master-plan.md) for any
checklists added later:

- `[ ]` = not started
- `[~]` = in progress
- `[x]` = complete
- `[!]` = blocked (add a note explaining what's blocking)

## Cross-references

Use relative links between files (e.g.,
`[Firebase projects](../infra/firebase/projects.md)`). Never link by
absolute URL to this repo's GitHub view — relative paths survive
forks, renames, and the eventual Confluence migration.
