---
title: Design Philosophy — From Karpathy's LLM Wiki to Software Documentation
sources:
  - "[[raw/guides/2026-04-09-llm-docs-blog]]"
  - "[[raw/specs/2026-04-08-llm-docs-design-spec]]"
last_updated: 2026-04-09
tags: [philosophy, karpathy, architecture, documentation]
---

# Design Philosophy

## The Problem: Documentation Decay

Software documentation decays not because engineers don't want to write docs, but because the marginal cost of maintenance is always borne by humans — and human attention is scarce. Writing a design doc takes hours; keeping it synchronized through every code change requires tedious bookkeeping (cross-references, term consistency, boundary conditions) that is eventually abandoned.

## Karpathy's LLM Wiki Pattern

Andrej Karpathy proposed a fundamental shift: instead of forcing the LLM to rediscover patterns from raw documents on every query (RAG), build and maintain a structured wiki where knowledge compounds over time.

The core insight: **synthesis is a first-class artifact**. The wiki is a persistent, compounding artifact — cross-references already established, contradictions already flagged, relationships already mapped.

### Three-Layer Architecture (Original)

- **Raw Sources** — Immutable, human-curated documents. The LLM never touches this layer.
- **Wiki** — LLM-generated and maintained markdown pages. The LLM fully owns structure and content.
- **Schema** — Configuration defining wiki conventions, making the LLM a disciplined maintainer.

### Division of Labor

> The human's job is to curate sources, direct analysis, ask good questions. The LLM's job is everything else.

"Everything else" — updating summaries, maintaining cross-references, checking consistency — is precisely what kills human wikis. These tasks are tedious for humans but reliable and tireless for LLMs.

## Domain Adaptation for Software Engineering

LLM-Docs adapts the pattern to software engineering with several key extensions:

### Why Software Docs Fit This Pattern

1. **Verifiable correspondence** — API docs must match code endpoints; data model docs must match schemas. This correspondence is machine-checkable, enabling deep documentation-code audit.
2. **Predictable lifecycle** — Specs evolve during implementation then stabilize; ADRs are immutable once decided. Software documents have regular change patterns.
3. **Git as natural infrastructure** — `git diff` and `git log` serve as change discovery signals, enabling "one-button sync" operations.
4. **AI coding agents as consumers** — Structured knowledge bases provide design context before code modification.

### Controlled Evolution vs. Absolute Immutability

Karpathy's raw sources (papers, articles) are truly immutable — published papers don't change. But software specs and plans evolve during implementation. Rather than letting documents diverge from reality, LLM-Docs provides a formal change channel with dual audit trails (git history + log.md), preserving the full evolution narrative.

### Git as Signal Source

No independent change-tracking system. Git itself is the signal source — `git diff` reveals branch changes, `git log` reveals implementation progress. Documentation maintenance becomes a "one-button sync" operation.

### Progressive Enrichment

The system works from cold start. Each ingest adds knowledge; each lint fixes inconsistencies. Over time, the documentation system grows from blank slate to rich project knowledge base — embodying Karpathy's vision of compounding knowledge.

## Five Operations (Extended from Three)

Karpathy defined three operations (Ingest, Query, Lint). LLM-Docs extends to five for software engineering workflows:

| Operation | Karpathy's Original | Software Engineering Extension |
|-----------|---------------------|-------------------------------|
| **Init** | — (new) | Cold-start initialization for any project |
| **Ingest** | Ingest new sources → update wiki | + Auto-discovery via `git diff` |
| **Update** | — (new) | Controlled in-place raw file evolution with audit trail |
| **Lint** | Health checks for contradictions | + Deep documentation-code audit |
| **Query** | Search + synthesize + write-back | + Agent mode for AI coding context injection |

## Related

- [[wiki/llm-docs-overview]] — System architecture and skill reference
- [Karpathy's LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
