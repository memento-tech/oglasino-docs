# Confluence Migration Plan

## Trigger

Migrate when **either** condition fires:

- Documentation grows to a point where non-technical team members need
  to contribute (Confluence has a friendlier WYSIWYG editor than git +
  markdown), OR
- The company adopts Confluence for other reasons (e.g., sales,
  support, HR are already there).

Until a trigger fires, this repo is the system of record.

## Mapping

Each top-level directory becomes a Confluence **space**:

| Directory | Confluence space |
|---|---|
| `infra/` | Infrastructure |
| `product/` | Product |
| `design/` | Design |
| `business/` | Business |
| `meta/` | (folded into a "Documentation" space or merged into Infrastructure) |

Subdirectories become parent pages with child pages underneath.
Example: `infra/cloudflare/workers.md` → page **Workers** under parent
**Cloudflare** in the **Infrastructure** space.

## Tools

- **`pandoc`** — markdown → Confluence storage format
  (`pandoc -f markdown -t html` works for most pages; for full Confluence
  XHTML use `pandoc -f markdown -t docx` then upload, or one of the
  community markdown→storage-format converters).
- **Confluence's bulk import** for the migration.

## Pre-migration checklist

- [ ] All internal links use relative paths (verify with grep —
      reject any `https://github.com/...` link to this repo)
- [ ] All images live in the same directory as the markdown that
      references them (or under a sibling `assets/`)
- [ ] No GitHub-only markdown features used (e.g., GitHub alerts
      `> [!NOTE]`, task lists rendered as live checkboxes —
      checkboxes still render in Confluence but the live-toggle UX
      differs)
- [ ] Mermaid blocks: confirm Confluence has the Mermaid macro
      installed, or pre-render to PNG before migration
