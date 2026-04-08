---
name: docs-ingest
description: Add a new document to the LLM-Docs system. Supports file paths, free text, and URLs. Classifies the document, archives it to docs/raw/, updates the wiki and index. Use when recording design decisions, importing existing docs, or archiving meeting notes.
allowed-tools: Read Write Edit Bash Glob Grep WebFetch
user-invocable: true
argument-hint: <file-path | "free text" | https://url>
---

# docs-ingest: Add Document to Documentation System

Add a new document to the LLM-Docs documentation system. The document is classified, archived as an immutable raw record, and the wiki is updated to reflect the new knowledge.

## Input Parsing

The argument `$ARGUMENTS` determines the input type:

1. **File path** — argument is a path to an existing file (check with Glob/Read)
   - Read the file content
   - `source_type: file`
2. **URL** — argument starts with `http://` or `https://`
   - Fetch content via WebFetch
   - `source_type: url`
   - Record the URL in `source_url` frontmatter field
3. **Free text** — argument is quoted text or anything that is not a file path or URL
   - Use the text directly as content
   - `source_type: text`
4. **No argument** — ask the user what they want to ingest

## Workflow

### Step 1: Load Schema

Read `docs/schema.md` to understand the current directory structure, classification table, and format conventions.

If `docs/schema.md` does not exist, tell the user to run `/docs-init` first and stop.

### Step 2: Analyze Content

Analyze the input content:

1. **Classify**: Determine which `docs/raw/` subdirectory it belongs to based on the classification table in schema.md. If no existing category fits, propose a new category to the user and await confirmation. If confirmed, create the subdirectory and update schema.md.
2. **Title**: Extract or generate a concise title
3. **Tags**: Generate relevant tags based on content semantics
4. **Key entities**: Identify important concepts, decisions, components, or systems mentioned

### Step 3: Archive to Raw

Write the content to `docs/raw/<category>/YYYY-MM-DD-<topic>.md` where:
- `YYYY-MM-DD` is today's date
- `<topic>` is a kebab-case slug derived from the title
- If a file with the same name already exists, append a numeric suffix (e.g., `-2`)

Frontmatter:

```yaml
---
title: <extracted title>
source_type: file | text | url | skill
ingested_at: YYYY-MM-DD
source_url: <if URL input>
skill_name: <if skill output>
tags: [tag1, tag2]
immutable: true
---
```

The body is the original content, unmodified.

### Step 4: Check for Unprocessed Raw Files

Scan `docs/raw/` for files that exist but have no corresponding entry in `docs/log.md`. These are files written directly by other skills (e.g., brainstorming specs, implementation plans).

If found, ask the user:

> "发现 N 个未处理的原始文档：
> - docs/raw/specs/2026-04-08-example.md
> - ...
> 是否一并更新 wiki？"

If yes, include them in the wiki update step below.

### Step 5: Update Wiki

Read `docs/README.md` to understand the current wiki structure. Then decide:

1. **Create new wiki page**: If the ingested content introduces a new topic not covered by any existing wiki page
2. **Update existing wiki page**: If the content updates, extends, or supersedes information in an existing wiki page
3. **Both**: If it's a new aspect of an existing topic

For each wiki page created or updated:
- Set `sources` in frontmatter to include all raw files that contributed
- Set `last_updated` to today's date
- Set `tags` based on the content's topics and the tags from contributing raw files
- Use `[[wiki-links]]` for cross-references to other wiki pages
- Use `[[raw/category/filename]]` for source references
- Synthesize a coherent current-state view — don't just append. The wiki page should read as a standalone document reflecting the latest understanding.

### Step 6: Update README.md

Read the current `docs/README.md`. Add or update entries for any new or changed wiki pages. Each entry is one line:

```markdown
- [[wiki/page-name]] — One-line description of what this page covers
```

Group entries by topic. If a new topic group is needed, add a new `## Section` header.

### Step 7: Append to Log

Append an entry to `docs/log.md`:

```markdown
## [YYYY-MM-DD] ingest | <document title>
- source: <source_type> (<source description>)
- raw: raw/<category>/<filename>.md
- wiki updated: wiki/<page>.md (<created | updated>)
- index updated: <description of README.md changes>
```

### Step 8: Present Summary

Show the user a concise summary:

> **Ingest 完成**
> - 原始文档：`docs/raw/<category>/<filename>.md`
> - 分类：<category>
> - Wiki 更新：`docs/wiki/<page>.md` (created/updated)
> - 索引更新：已添加/更新 README.md 条目

### Step 9: Commit

Stage all changed files under `docs/`. Create a git commit:

```
docs: ingest <document title>
```
