# Documentation Schema

This file defines the conventions for the LLM-Docs documentation management system. It serves as the shared foundation for all skills and AI coding agents.

## Directory Structure

```
docs/
├── README.md          # Wiki navigation index (LLM-maintained)
├── schema.md          # This file: conventions and contracts
├── log.md             # Operation audit log (append-only)
├── raw/               # Layer 1: immutable source documents
│   ├── specs/         # Design specifications
│   ├── plans/         # Implementation plans
│   ├── architecture/  # Architecture documents
│   ├── adr/           # Architecture decision records
│   ├── api/           # API design documents
│   ├── guides/        # Runbooks, deployment guides
│   ├── prd/           # Product requirement documents
│   ├── meeting/       # Meeting notes, technical discussions
│   └── ...            # Extensible as needed
└── wiki/              # Layer 2: LLM-maintained knowledge base
    └── ...            # LLM-organized
```

## Document Format

### Raw Document Frontmatter

```yaml
---
title: Document title
source_type: file | text | url | skill
ingested_at: YYYY-MM-DD
source_url:              # Recorded for URL inputs
skill_name:              # Recorded for skill outputs
tags: [tag1, tag2]
immutable: true
---
```

### Wiki Page Frontmatter

```yaml
---
title: Page title
sources:
  - "[[raw/category/filename]]"
last_updated: YYYY-MM-DD
tags: [tag1, tag2]
---
```

### Cross-references

Use Obsidian-compatible `[[wiki-links]]` syntax throughout:

- Between wiki pages: `[[wiki/page-name]]`
- Wiki referencing raw: `[[raw/category/filename]]`
- README.md index entries: `[[wiki/page-name]]`

### File Naming

- Raw documents: `YYYY-MM-DD-<topic>.md` (date-prefixed)
- Wiki pages: `<topic>.md` (no date, continuously updated)

## Classification

Raw documents are filed into subdirectories by type. The LLM may propose new categories (with user confirmation):

| Category | Directory | Content |
|----------|-----------|---------|
| Design Specs | `specs/` | Feature designs, brainstorming outputs |
| Implementation Plans | `plans/` | Implementation steps, writing-plans outputs |
| Architecture | `architecture/` | System architecture, technology choices |
| Architecture Decisions | `adr/` | Background, options, and conclusions for individual decisions |
| API | `api/` | Interface design, protocol definitions |
| Guides | `guides/` | Deployment, operations, runbooks |
| Product Requirements | `prd/` | PRDs, BRDs, user stories |
| Meeting Notes | `meeting/` | Meeting conclusions, technical discussion records |

Classification rule: the LLM automatically classifies during ingest based on content semantics. When content does not fit any existing category, the LLM proposes a new subdirectory to the user.

## Operation Contracts

### ingest

- **Input**: No argument (auto-discover via `git diff main...HEAD`) / file path / free text / URL
- **Output**: New file in raw/ + wiki update + README.md update + log.md entry
- **Invariant**: Files in raw/ are not modified after creation (except via the update operation)

### update

- **Input**: No argument (auto-detect from git log) / `--from-commits [range]` / raw file path + reason
- **Output**: In-place raw file update + wiki sync + log.md entry
- **Constraint**: Raw files may only be modified via `/docs-update`; changes are tracked via git history

### lint

- **Input**: No argument (auto-detect scope) / `--full` (force full lint)
- **Scope detection**: Branch mode (`git diff main...HEAD`) / incremental mode (main, since last lint commit) / full mode
- **Output**: Lint report (errors/warnings/passes) + optional auto-fix
- **Scope**: Internal doc consistency + documentation-code deep audit

### query

- **Developer mode input**: Natural language question
- **Developer mode output**: Answer + source citations + optional write-back
- **Agent mode input**: `--context <file/dir>`
- **Agent mode output**: Structured context summary (design intent, constraints, decisions, cautions)

## Log Format

`log.md` only records operations that produce file changes:

```markdown
## [YYYY-MM-DD] ingest | <document title>
- source: file | text | url | skill (<source description>)
- raw: raw/<category>/<filename>.md
- wiki updated: wiki/<page>.md (created | updated)
- index updated: <description of changes>

## [YYYY-MM-DD] lint
- scope: branch (<branch-name> vs main) | incremental (<last>..<current>) | full
- commit: <short hash>
- checked: N wiki pages, M source files
- issues: N (<summary>)
- fixed: <description of fixes>

## [YYYY-MM-DD] query | <topic title>
- question: <original question>
- raw: raw/<category>/<filename>.md
- wiki updated: wiki/<page>.md (created | updated)

## [YYYY-MM-DD] update | <document title>
- raw: raw/<category>/<filename>.md
- source: manual | commits (<range>)
- reason: <why the update was needed>
- changes: <brief summary>
- wiki updated: wiki/<page>.md (updated)
```

## Evolution Rules

This file (schema.md) may be evolved by the LLM. Allowed changes:

- Add new classification directories (with user confirmation)
- Adjust wiki organization suggestions
- Add format conventions

Disallowed changes:

- Remove existing classification directories
- Modify the immutability convention for raw/
- Modify the append-only convention for log.md
