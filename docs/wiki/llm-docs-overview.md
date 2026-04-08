---
title: LLM-Docs System Overview
sources:
  - "[[raw/specs/2026-04-08-llm-docs-design-spec]]"
  - "[[raw/plans/2026-04-08-llm-docs-skills-plan]]"
last_updated: 2026-04-08
tags: [architecture, documentation]
---

# LLM-Docs System Overview

LLM-Docs is an LLM-driven software engineering documentation management system, delivered as Claude Code Skills.

## Core Philosophy

Inspired by Karpathy's LLM Wiki pattern: humans write source documents and make decisions, the LLM synthesizes, updates, and checks consistency. Documentation is no longer a static artifact that rots after writing — it becomes a living knowledge base continuously maintained by the LLM.

## Three-Layer Architecture

- **Layer 1 `raw/`**: Source document archive. Organized by software engineering document type (specs, plans, architecture, adr, api, guides, prd, meeting, etc.). Files are generally stable after creation; changes go through `/docs-update` with an audit trail.
- **Layer 2 `wiki/`**: LLM-maintained knowledge base. Synthesizes and distills historical documents from raw/, reflecting the current state of the project. The LLM fully controls the organization and cross-references in this directory.
- **Layer 3 `schema.md` + `README.md`**: Entry point for AI coding agents. schema.md defines system conventions, README.md provides wiki navigation index.

## Five Operation Skills

| Skill | Function | Invocation |
|-------|----------|------------|
| `/docs-init` | Initialize documentation system | `/docs-init` |
| `/docs-ingest` | Add documents and update wiki | `/docs-ingest [file\|text\|url]` (auto-discovers from branch diff when no argument) |
| `/docs-update` | Update raw documents in-place with audit | `/docs-update [raw-file] [reason\|--from-commits]` (auto-detects from commit history when no argument) |
| `/docs-lint` | Documentation consistency + code audit | `/docs-lint [--full]` (auto-detects branch/incremental scope) |
| `/docs-query` | Developer Q&A / agent context injection | `/docs-query <question\|--context file>` |

### Smart Defaults

Three core skills support auto-discovery when invoked without arguments:

- **`/docs-ingest`** — `git diff main...HEAD` finds new/modified documentation files on the current branch
- **`/docs-update`** — Analyzes git log to find implementation commits and matches them to raw documents needing updates
- **`/docs-lint`** — On a branch, lints only branch changes; on main, incrementally lints since last lint commit

## Design Principles

1. **Controlled Mutation** — raw/ files are generally stable; changes go through `/docs-update`, tracked via git history and log.md
2. **Single Source of Truth** — wiki/ is the sole authority for "current state"
3. **Minimal Footprint** — docs/ is the only new directory added to the project
4. **Progressive** — Works from cold start, enriches over time with use
5. **Auditable** — log.md records all operations that produce side effects
6. **Obsidian Compatible** — `[[wiki-links]]`, YAML frontmatter

## Integration with Other Skills

Brainstorming spec output goes to `docs/raw/specs/`, writing-plans output goes to `docs/raw/plans/`. Configured via path overrides in CLAUDE.md. When files are written directly to raw/ but wiki is not yet synced, `/docs-ingest` detects and prompts the user.

## Related Documents

- Full design specification: [[raw/specs/2026-04-08-llm-docs-design-spec]]
- Implementation plan: [[raw/plans/2026-04-08-llm-docs-skills-plan]]
- System conventions: [[schema]]
