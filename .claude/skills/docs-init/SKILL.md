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
