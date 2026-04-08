# LLM-Docs

LLM-driven documentation management system for software engineering projects, delivered as [Claude Code](https://docs.anthropic.com/en/docs/claude-code) Skills.

Inspired by [Karpathy's LLM Wiki pattern](https://x.com/karpathy/status/1937538842549903593): humans write source documents and make decisions, the LLM synthesizes, updates, and checks consistency. Documentation becomes a living knowledge base rather than a static artifact.

## Three-Layer Architecture

```
docs/
├── raw/        # Layer 1: Immutable source documents (specs, plans, ADRs, etc.)
├── wiki/       # Layer 2: LLM-maintained current knowledge base
├── schema.md   # Layer 3: System conventions (shared by all skills)
├── README.md   # Layer 3: Wiki navigation index
└── log.md      # Append-only audit log
```

- **`raw/`** — Archived source documents, organized by type. Generally stable after creation; changes go through `/docs-update` with full audit trail.
- **`wiki/`** — LLM synthesizes raw documents into a coherent, up-to-date knowledge base. The single source of truth for current project state.
- **`schema.md` + `README.md`** — Entry point for AI coding agents and human developers.

## Skills

| Skill | What it does |
|-------|-------------|
| `/docs-init` | Initialize the documentation system in any project |
| `/docs-ingest` | Add documents (file, text, URL) and update the wiki |
| `/docs-update` | Update raw documents in-place with audit logging |
| `/docs-lint` | Check doc consistency and audit docs against code |
| `/docs-query` | Q&A with citations, or inject design context for AI agents |

All three mutation skills (`ingest`, `update`, `lint`) support **smart defaults** — run them with no arguments and they auto-discover what needs attention using `git diff` and `git log`.

## Quick Start

```
# In any project with Claude Code
/docs-init

# Add a design document
/docs-ingest path/to/spec.md

# After implementation work, sync docs with code changes
/docs-update

# Check everything is consistent
/docs-lint

# Ask questions about the project
/docs-query "What is the retry policy for the payment API?"

# Get design context before modifying code
/docs-query --context src/auth/handler.go
```

## Installation

Copy the `.claude/skills/` directory into your project:

```bash
cp -r .claude/skills/docs-* /path/to/your-project/.claude/skills/
```

Then run `/docs-init` in the target project to set up the `docs/` directory structure.

## Design Principles

1. **Controlled Mutation** — raw/ files are stable; changes are tracked via git history and log.md
2. **Single Source of Truth** — wiki/ is the sole authority for current state
3. **Minimal Footprint** — `docs/` is the only directory added to your project
4. **Progressive** — Works from cold start, enriches over time
5. **Auditable** — log.md records every operation with side effects
6. **Obsidian Compatible** — `[[wiki-links]]`, YAML frontmatter, works as an Obsidian vault
