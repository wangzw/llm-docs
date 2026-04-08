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

## Step 1: Create Directory Structure

Run:

```bash
mkdir -p docs/raw/{specs,plans,architecture,adr,api,guides,prd,meeting} docs/wiki
```

## Step 2: Generate docs/schema.md

Write the following content to `docs/schema.md` exactly:

````markdown
# Documentation Schema

本文件定义 LLM-Docs 文档管理系统的架构约定，是所有 skill 和 AI coding agent 的共享基础。

## 目录结构

```
docs/
├── README.md          # Wiki 导航索引（LLM 维护）
├── schema.md          # 本文件：架构约定
├── log.md             # 操作审计日志（append-only）
├── raw/               # 第一层：不可变原始文档
│   ├── specs/         # 设计规格
│   ├── plans/         # 实现计划
│   ├── architecture/  # 架构文档
│   ├── adr/           # 架构决策记录
│   ├── api/           # API 设计文档
│   ├── guides/        # 操作指南、Runbook、部署文档
│   ├── prd/           # 产品需求文档
│   ├── meeting/       # 会议纪要、技术决策记录
│   └── ...            # 可按需扩展
└── wiki/              # 第二层：LLM 维护的当前知识库
    └── ...            # LLM 自主组织
```

## 文档格式规范

### Raw 文档 Frontmatter

```yaml
---
title: 文档标题
source_type: file | text | url | skill
ingested_at: YYYY-MM-DD
source_url:              # URL 输入时记录
skill_name:              # skill 产出时记录
tags: [tag1, tag2]
immutable: true
---
```

### Wiki 页面 Frontmatter

```yaml
---
title: 页面标题
sources:
  - "[[raw/category/filename]]"
last_updated: YYYY-MM-DD
tags: [tag1, tag2]
---
```

### 交叉引用

统一使用 Obsidian 兼容的 `[[wiki-links]]` 语法：

- Wiki 页面之间：`[[wiki/page-name]]`
- Wiki 引用 Raw：`[[raw/category/filename]]`
- README.md 中的索引条目：`[[wiki/page-name]]`

### 文件命名

- Raw 文档：`YYYY-MM-DD-<topic>.md`（日期前缀）
- Wiki 页面：`<topic>.md`（无日期，因为持续更新）

## 分类体系

Raw 文档按以下初始分类归入子目录，LLM 可按需提议新分类（需经用户确认）：

| 分类 | 目录 | 内容 |
|------|------|------|
| 设计规格 | `specs/` | 功能设计、brainstorming 产出 |
| 实现计划 | `plans/` | 实现步骤、writing-plans 产出 |
| 架构 | `architecture/` | 系统架构、技术选型 |
| 架构决策 | `adr/` | 单个技术决策的背景、选项、结论 |
| API | `api/` | 接口设计、协议定义 |
| 操作指南 | `guides/` | 部署、运维、Runbook |
| 产品需求 | `prd/` | PRD、BRD、用户故事 |
| 会议纪要 | `meeting/` | 会议结论、技术讨论记录 |

分类规则：ingest 时由 LLM 根据内容语义自动决定。当内容不属于任何现有分类时，LLM 向用户提议创建新分类目录。

## 操作契约

### ingest

- **输入**：无参数（自动从 `git diff main...HEAD` 发现待归档文件）/ 文件路径 / 自由文本 / URL
- **输出**：raw/ 中的新文件 + wiki/ 的更新 + README.md 更新 + log.md 记录
- **不变量**：raw/ 中的文件一旦写入不可修改（update 操作除外）

### update

- **输入**：无参数（自动从 git log 匹配需更新的 raw 文档）/ `--from-commits [range]` / raw/ 文件路径 + 变更原因
- **输出**：raw/ 文件原地更新 + wiki/ 同步 + log.md 记录
- **约束**：仅通过 `/docs-update` skill 修改 raw/ 文件，变更通过 git history 追踪

### lint

- **输入**：无参数（自动检测范围）/ `--full`（强制全量）
- **范围检测**：分支模式（`git diff main...HEAD`）/ 增量模式（main 上从上次 lint commit 起）/ 全量模式
- **输出**：lint 报告（错误/警告/通过） + 可选自动修复
- **范围**：文档间一致性 + 文档与代码的深度对照

### query

- **开发者模式输入**：自然语言问题
- **开发者模式输出**：回答 + 引用来源 + 可选回写
- **Agent 模式输入**：`--context <file/dir>`
- **Agent 模式输出**：结构化上下文摘要（设计意图、约束、决策、注意事项）

## 日志格式

`log.md` 仅记录产生文件变更的操作：

```markdown
## [YYYY-MM-DD] ingest | 文档标题
- source: file | text | url | skill (来源描述)
- raw: raw/category/filename.md
- wiki updated: wiki/page.md (created | updated)
- index updated: 变更描述

## [YYYY-MM-DD] lint
- scope: branch (branch-name vs main) | incremental (last..current) | full
- commit: <short hash>
- checked: N wiki pages, M source files
- issues: N (问题摘要)
- fixed: 修复描述

## [YYYY-MM-DD] query | 回写标题
- question: 原始问题
- raw: raw/category/filename.md
- wiki updated: wiki/page.md (created | updated)

## [YYYY-MM-DD] update | 文档标题
- raw: raw/category/filename.md
- source: manual | commits (range)
- reason: 变更原因
- changes: 变更摘要
- wiki updated: wiki/page.md (updated)
```

## 演进规则

本文件（schema.md）可以被 LLM 演进。允许的变更：

- 新增分类目录（需用户确认）
- 调整 wiki 组织建议
- 补充格式规范

不允许的变更：

- 删除已有分类目录
- 修改 raw/ 的不可变性约定
- 修改 log.md 的 append-only 约定
````

## Step 3: Generate docs/README.md

Write the following content to `docs/README.md` exactly:

```markdown
# Documentation Index

本项目使用 LLM 驱动的文档管理系统。详见 [[schema]] 了解系统约定。
```

## Step 4: Generate docs/log.md

Write the following content to `docs/log.md` exactly:

```markdown
# Documentation Log
```

## Step 5: Configure CLAUDE.md

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

If `CLAUDE.md` does not exist, create it with a project title header (derived from the directory name or git remote) followed by the section above.

## Step 6: Obsidian Support (Optional)

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

## Step 7: Interactive Import (Optional)

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

## Step 8: Commit

Stage all new files under `docs/` and changes to `CLAUDE.md`. Create a git commit:

```
docs: initialize LLM-Docs documentation system
```

## Completion Message

Report to the user:

> "文档系统初始化完成。
> - 目录结构：docs/raw/ (原始文档) + docs/wiki/ (当前知识库)
> - 系统约定：docs/schema.md
> - 导航索引：docs/README.md
>
> 使用 `/docs-ingest` 添加文档，`/docs-update` 更新文档，`/docs-lint` 检查一致性，`/docs-query` 查询文档。"
