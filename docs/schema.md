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

- **输入**：文件路径 / 自由文本 / URL
- **输出**：raw/ 中的新文件 + wiki/ 的更新 + README.md 更新 + log.md 记录
- **不变量**：raw/ 中的文件一旦写入不可修改

### lint

- **输入**：无参数，扫描整个 docs/ 和项目代码
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
- checked: N wiki pages, M source files
- issues: N (问题摘要)
- fixed: 修复描述

## [YYYY-MM-DD] query | 回写标题
- question: 原始问题
- raw: raw/category/filename.md
- wiki updated: wiki/page.md (created | updated)
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
