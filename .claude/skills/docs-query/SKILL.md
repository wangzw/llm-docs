---
name: docs-query
description: Query the LLM-Docs documentation system. Two modes — developer mode (default) for natural language Q&A with source citations, and agent mode (--context) for injecting relevant design context into AI coding workflows. Use when you need to understand design decisions, architecture, or constraints before modifying code.
allowed-tools: Read Glob Grep WebFetch
user-invocable: true
argument-hint: <"question" | --context file/dir>
---

# docs-query: Query Documentation System

Query the LLM-Docs documentation system. Supports two modes based on the argument:

- **Developer mode** (default): Natural language Q&A with source citations
- **Agent mode**: `--context <file/dir>` for structured context injection into AI coding workflows

## Pre-flight Check

Read `docs/schema.md`. If it does not exist, tell the user to run `/docs-init` first and stop.

## Mode Detection

Parse `$ARGUMENTS`:

- If it starts with `--context`, enter **Agent mode**. The remaining argument is the file or directory path.
- Otherwise, treat the entire argument as a natural language question and enter **Developer mode**.
- If no argument is provided, ask the user what they want to know.

## Developer Mode

### Step 1: Locate Relevant Pages

1. Read `docs/README.md` to get the wiki index
2. Based on the question's keywords and semantics, identify which wiki pages are likely relevant
3. Read those wiki pages

### Step 2: Deep Dive if Needed

If the wiki pages don't fully answer the question:

1. Check the `sources` in relevant wiki pages
2. Read the referenced raw documents for more detail
3. If the question is about historical decisions, prioritize raw/adr/ and raw/architecture/ documents

### Step 3: Synthesize Answer

Compose an answer that:

1. Directly addresses the question
2. Cites sources using `[[wiki-links]]` and `[[raw/...]]` references
3. Distinguishes between "current state" (from wiki) and "historical context" (from raw)
4. If the documentation doesn't cover the question, say so explicitly rather than guessing

### Step 4: Offer Write-back

After presenting the answer, ask:

> "这个回答是否值得写入文档？"

If yes:
1. Structure the answer as a proper document
2. Classify and write to `docs/raw/` with frontmatter (source_type: text)
3. Update the relevant wiki page(s) to incorporate the new insight
4. Update `docs/README.md` if new wiki pages were created
5. Append to `docs/log.md`:

```markdown
## [YYYY-MM-DD] query | <topic title>
- question: <original question>
- raw: raw/<category>/<filename>.md
- wiki updated: wiki/<page>.md (<created | updated>)
```

6. Commit:

```
docs: ingest query insight — <topic>
```

If no, end the interaction. Do NOT write to log.md.

## Agent Mode

### Step 1: Analyze Work Context

Read the file(s) or directory specified by `--context`:

1. Understand what the code does — its module, its responsibilities, its interfaces
2. Identify the key concepts, systems, and components involved
3. Note any patterns, conventions, or dependencies

### Step 2: Find Relevant Documentation

1. Read `docs/README.md` to get the wiki index
2. Based on the code context, identify relevant wiki pages by matching:
   - Module/component names
   - Tags overlap
   - Concept references
3. Read the matched wiki pages
4. If needed, read referenced raw documents for deeper context (especially ADRs and architecture docs)

### Step 3: Output Structured Context

Present the context as a structured summary that an AI coding agent can directly consume:

```markdown
## Design Context for <file/module>

### Design Intent
<Why this code exists and what problem it solves>

### Architecture
<Where this fits in the overall system, key dependencies>

### Constraints
<Non-obvious rules, invariants, or limitations that must be respected>

### Related Decisions
<Relevant ADRs or design decisions with [[links]]>

### Cautions
<Things to watch out for when modifying this code>
```

Do NOT write to log.md (agent mode is a read-only operation with no side effects).
