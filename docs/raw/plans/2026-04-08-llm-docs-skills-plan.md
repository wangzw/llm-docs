# LLM-Docs Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the four Claude Code Skills (docs-init, docs-ingest, docs-lint, docs-query) that form the LLM-Docs documentation management system.

**Architecture:** Each skill is a directory under `.claude/skills/` with a `SKILL.md` entry point. Skills are prompt-based — they instruct the LLM how to perform each operation. All skills read `docs/schema.md` as their shared configuration. Since skills are markdown prompt files (not executable code), testing means invoking each skill and verifying it behaves correctly.

**Tech Stack:** Claude Code Skills (SKILL.md), Markdown, YAML frontmatter

---

## File Structure

```
.claude/skills/
├── docs-init/
│   └── SKILL.md          # Initialize documentation system
├── docs-ingest/
│   └── SKILL.md          # Add documents and update wiki
├── docs-lint/
│   └── SKILL.md          # Check doc consistency + code audit
└── docs-query/
    └── SKILL.md          # Developer Q&A + agent context injection
```

All four files are new. No existing files are modified in this plan (CLAUDE.md and docs/ structure are already in place from bootstrapping).

---

### Task 1: Create docs-init Skill

**Files:**
- Create: `.claude/skills/docs-init/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p .claude/skills/docs-init
```

- [ ] **Step 2: Write SKILL.md**

Write the following content to `.claude/skills/docs-init/SKILL.md`:

```markdown
---
name: docs-init
description: Initialize the LLM-Docs documentation management system in the current project. Creates docs/ directory structure, schema.md, README.md, log.md, and configures CLAUDE.md. Use when setting up documentation for a new project.
allowed-tools: Read Write Bash Glob Grep
user-invocable: true
argument-hint: ""
---

# docs-init: Initialize Documentation System

Initialize the LLM-Docs documentation management system in the current project.

## Pre-flight Check

1. Read `docs/schema.md` — if it exists, this project already has LLM-Docs. Warn the user and ask whether to re-initialize (destructive) or abort.
2. If `docs/` directory exists but has no `schema.md`, note existing contents for the interactive import step.

## Initialization Steps

### Step 1: Create Directory Structure

Create the following directories and files:

```
docs/
├── README.md
├── schema.md
├── log.md
├── raw/
│   ├── specs/
│   ├── plans/
│   ├── architecture/
│   ├── adr/
│   ├── api/
│   ├── guides/
│   ├── prd/
│   └── meeting/
└── wiki/
```

### Step 2: Generate schema.md

Write `docs/schema.md` with the standard LLM-Docs schema. Use the following template as the base content:

- Directory structure section documenting raw/ subdirectories and wiki/
- Document format section with raw frontmatter (title, source_type, ingested_at, source_url, skill_name, tags, immutable) and wiki frontmatter (title, sources, last_updated, tags)
- Cross-reference convention: `[[wiki-links]]` syntax (Obsidian compatible)
- File naming: raw docs use `YYYY-MM-DD-<topic>.md`, wiki pages use `<topic>.md`
- Classification table mapping document types to raw/ subdirectories
- Operation contracts for ingest, lint, query
- Log format specification
- Evolution rules (what can/cannot change in schema.md)

### Step 3: Generate README.md

Write `docs/README.md` as an empty index:

```markdown
# Documentation Index

本项目使用 LLM 驱动的文档管理系统。详见 [[schema]] 了解系统约定。
```

### Step 4: Generate log.md

Write `docs/log.md`:

```markdown
# Documentation Log
```

### Step 5: Configure CLAUDE.md

Read the existing `CLAUDE.md` (if any). Append the following section if not already present:

```markdown
## Documentation

本项目使用 LLM 驱动的文档管理系统。

- 文档入口：docs/README.md
- 文档约定：docs/schema.md
- 在修改代码前，建议先通过 `/docs-query --context <file>` 了解相关设计背景
- 在完成重要设计决策后，通过 `/docs-ingest` 将决策归档
- Spec 文件输出到：docs/raw/specs/
- Plan 文件输出到：docs/raw/plans/
```

If `CLAUDE.md` does not exist, create it with a project title header followed by the section above.

### Step 6: Obsidian Support (Optional)

Ask the user:

> "是否需要 Obsidian 支持？如果是，我会生成 `docs/.obsidian/` 基础配置，使 docs/ 目录可直接作为 Obsidian vault 打开。"

If yes, create `docs/.obsidian/app.json` with:

```json
{
  "useMarkdownLinks": false,
  "newLinkFormat": "relative",
  "attachmentFolderPath": "docs/raw"
}
```

If no, skip.

### Step 7: Interactive Import (Optional)

Ask the user:

> "是否扫描项目中已有的文档进行交互式导入？"

If yes:
1. Use Glob to find all `*.md` files in the project (excluding `docs/`, `node_modules/`, `.git/`, `.claude/`)
2. List found files with one-line content summaries
3. Let the user select which files to import
4. For each selected file, perform the ingest workflow:
   a. Read the file content
   b. Classify it into the appropriate raw/ subdirectory
   c. Copy it to `docs/raw/<category>/YYYY-MM-DD-<topic>.md` with frontmatter added
   d. Create or update relevant wiki pages
   e. Update README.md index
   f. Append to log.md
5. Present a summary of all imported documents

### Step 8: Commit

Stage all new files under `docs/` and changes to `CLAUDE.md`. Create a git commit:

```
docs: initialize LLM-Docs documentation system
```

### Completion Message

Report to the user:

> "文档系统初始化完成。
> - 目录结构：docs/raw/ (原始文档) + docs/wiki/ (当前知识库)
> - 系统约定：docs/schema.md
> - 导航索引：docs/README.md
>
> 使用 `/docs-ingest` 添加文档，`/docs-lint` 检查一致性，`/docs-query` 查询文档。"
```

- [ ] **Step 3: Verify the skill file**

Run: `cat .claude/skills/docs-init/SKILL.md | head -5`
Expected: frontmatter with `name: docs-init`

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/docs-init/SKILL.md
git commit -m "feat: add docs-init skill for documentation system initialization"
```

---

### Task 2: Create docs-ingest Skill

**Files:**
- Create: `.claude/skills/docs-ingest/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p .claude/skills/docs-ingest
```

- [ ] **Step 2: Write SKILL.md**

Write the following content to `.claude/skills/docs-ingest/SKILL.md`:

```markdown
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
```

- [ ] **Step 3: Verify the skill file**

Run: `cat .claude/skills/docs-ingest/SKILL.md | head -5`
Expected: frontmatter with `name: docs-ingest`

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/docs-ingest/SKILL.md
git commit -m "feat: add docs-ingest skill for document ingestion and wiki updates"
```

---

### Task 3: Create docs-lint Skill

**Files:**
- Create: `.claude/skills/docs-lint/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p .claude/skills/docs-lint
```

- [ ] **Step 2: Write SKILL.md**

Write the following content to `.claude/skills/docs-lint/SKILL.md`:

```markdown
---
name: docs-lint
description: Check documentation consistency and audit docs against code. Validates wiki links, source references, detects contradictions between wiki pages, and performs deep comparison of documentation against actual code (API endpoints, data models, configs, function signatures). Use when you suspect docs are out of date or before major releases.
allowed-tools: Read Write Edit Bash Glob Grep
user-invocable: true
argument-hint: ""
---

# docs-lint: Documentation Consistency Check & Code Audit

Check the documentation system for internal consistency and audit documentation against the actual codebase.

## Pre-flight Check

Read `docs/schema.md`. If it does not exist, tell the user to run `/docs-init` first and stop.

## Lint Workflow

### Phase 1: Document Internal Consistency

#### 1a. Link Validation

1. Read `docs/README.md`. Extract all `[[wiki-links]]`.
2. For each link, verify the target file exists using Glob.
3. Read all wiki pages in `docs/wiki/`. Extract all `[[wiki-links]]` and `sources` references.
4. For each link, verify the target exists.
5. Report broken links as 🔴 errors.

#### 1b. Orphan Detection

1. Collect all wiki pages from `docs/wiki/`.
2. Collect all pages referenced from `docs/README.md`.
3. Pages that exist in wiki/ but are not referenced from README.md are orphans.
4. Report orphans as 🟡 warnings.

#### 1c. Unprocessed Raw Files

1. Collect all files in `docs/raw/` (recursively).
2. Read `docs/log.md` and extract all raw file paths mentioned in ingest entries.
3. Files in raw/ with no corresponding log entry are unprocessed.
4. Report unprocessed files as 🟡 warnings.

#### 1d. Wiki Contradiction Detection

1. Read all wiki pages.
2. Identify pages that cover overlapping topics (based on tags and content).
3. For overlapping pages, check whether they make contradictory claims about the same concept.
4. Report contradictions as 🔴 errors with specific quotes from both pages.

### Phase 2: Documentation-Code Deep Audit

This phase reads the actual codebase and compares it against documentation claims.

#### 2a. Discover Code Structure

Use Glob and Grep to identify:

1. **API endpoints**: Search for route definitions, handler registrations, controller annotations
2. **Data models**: Search for schema definitions, ORM models, struct/class definitions, migration files
3. **Configuration**: Search for config files, environment variable references, feature flags
4. **Key functions/interfaces**: Search for exported functions, public interfaces, service boundaries

Build a mental model of what the code actually does.

#### 2b. Cross-reference with Wiki

For each wiki page:

1. Read the page content
2. Extract claims about code behavior (e.g., "the auth service uses JWT tokens", "the payment API has 3 endpoints", "retry policy is exponential backoff with max 5 retries")
3. Verify each claim against the code:
   - **Documented but not in code**: The doc describes something that doesn't exist in the codebase → 🔴 error
   - **In code but not documented**: A significant code structure (API endpoint, data model, service) has no documentation → 🟡 warning
   - **Doc contradicts code**: The doc describes behavior differently from what the code implements → 🔴 error
   - **Consistent**: The doc accurately reflects the code → 🟢 pass

### Phase 3: Report

Present a structured lint report:

```
# Documentation Lint Report

## Summary
- 🔴 Errors: N
- 🟡 Warnings: N
- 🟢 Passed: N checks

## 🔴 Errors

### [E001] Broken link in README.md
- Location: docs/README.md line 15
- Link: [[wiki/nonexistent-page]]
- Action: Remove link or create the wiki page

### [E002] Wiki contradicts code
- Wiki: docs/wiki/auth.md claims "uses session-based auth"
- Code: src/auth/handler.go implements JWT-based auth
- Action: Update wiki page to reflect JWT implementation

## 🟡 Warnings

### [W001] Orphan wiki page
- Page: docs/wiki/old-design.md
- Not referenced from README.md
- Action: Add to README.md or archive if obsolete

### [W002] Undocumented API endpoint
- Endpoint: POST /api/v2/webhooks (src/api/webhooks.go:45)
- No wiki page covers this endpoint
- Action: Document in relevant wiki page

## 🟢 Passed Checks
- All source references in wiki pages point to existing raw files
- Data model documentation matches current schema
- ...
```

### Phase 4: Auto-fix (Optional)

After presenting the report, ask the user:

> "是否自动修复？我可以：
> - 更新 wiki 页面以匹配代码实际行为
> - 移除断裂的链接
> - 将孤立页面添加到 README.md
>
> 选择：全部修复 / 逐项确认 / 跳过"

If the user chooses to fix:
1. Apply fixes to wiki pages (update content to match code, fix broken links, add orphans to README.md)
2. Do NOT modify raw/ files (they are immutable)
3. Update `last_updated` in modified wiki pages
4. Append to `docs/log.md`:

```markdown
## [YYYY-MM-DD] lint
- checked: N wiki pages, M source files
- issues: N (summary)
- fixed: description of fixes applied
```

5. Commit:

```
docs: lint fix — update wiki to match codebase
```

If the user chooses to skip, no files are modified and no log entry is written.
```

- [ ] **Step 3: Verify the skill file**

Run: `cat .claude/skills/docs-lint/SKILL.md | head -5`
Expected: frontmatter with `name: docs-lint`

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/docs-lint/SKILL.md
git commit -m "feat: add docs-lint skill for documentation consistency checking and code audit"
```

---

### Task 4: Create docs-query Skill

**Files:**
- Create: `.claude/skills/docs-query/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p .claude/skills/docs-query
```

- [ ] **Step 2: Write SKILL.md**

Write the following content to `.claude/skills/docs-query/SKILL.md`:

```markdown
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
```

- [ ] **Step 3: Verify the skill file**

Run: `cat .claude/skills/docs-query/SKILL.md | head -5`
Expected: frontmatter with `name: docs-query`

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/docs-query/SKILL.md
git commit -m "feat: add docs-query skill for documentation Q&A and agent context injection"
```

---

### Task 5: Smoke Test All Skills

**Files:**
- No new files. Verify existing skills are discoverable.

- [ ] **Step 1: Verify all skill directories exist**

```bash
ls -la .claude/skills/*/SKILL.md
```

Expected output should list 4 files:
```
.claude/skills/docs-init/SKILL.md
.claude/skills/docs-ingest/SKILL.md
.claude/skills/docs-lint/SKILL.md
.claude/skills/docs-query/SKILL.md
```

- [ ] **Step 2: Verify frontmatter of each skill**

```bash
for f in .claude/skills/*/SKILL.md; do echo "=== $f ==="; head -4 "$f"; echo; done
```

Expected: each file starts with `---` followed by `name: docs-<type>`.

- [ ] **Step 3: Invoke /docs-query as smoke test**

Run `/docs-query "LLM-Docs 系统的三层架构是什么？"`

Expected: The skill loads, reads docs/README.md and docs/wiki/llm-docs-overview.md, and returns an answer citing the three-layer architecture with `[[wiki-links]]` references.

- [ ] **Step 4: Invoke /docs-lint as smoke test**

Run `/docs-lint`

Expected: The skill loads, reads schema.md, scans docs/ and the codebase, and produces a lint report. Since this is the LLM-Docs project itself (mostly docs, minimal code), expect mostly 🟢 passes with possible 🟡 warnings about limited code coverage.

- [ ] **Step 5: Invoke /docs-ingest as smoke test**

Run `/docs-ingest "本项目的实现计划已完成，四个 skill 均已创建并通过冒烟测试。"`

Expected: The skill archives the text to `docs/raw/meeting/YYYY-MM-DD-implementation-complete.md`, updates wiki, updates README.md, appends to log.md, and commits.
