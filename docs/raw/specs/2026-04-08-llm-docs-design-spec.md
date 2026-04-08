---
title: LLM-Docs 设计规格
source_type: skill
skill_name: superpowers:brainstorming
ingested_at: 2026-04-08
tags: [architecture, design, documentation]
immutable: true
---

# LLM-Docs：基于 LLM 的软件工程文档管理系统

## 概述

LLM-Docs 是一套基于 LLM 驱动的软件工程文档管理系统，以 Claude Code Skills 形式交付。它借鉴 Karpathy 的 LLM Wiki 模式，将软件文档组织为三层架构，通过四个核心操作（ingest、update、lint、query）实现文档的持续维护，服务于 AI coding agent 和人类开发者。

### 设计目标

1. **文档即代码** — 所有文档在 Git 仓库中版本控制，与代码同生命周期
2. **LLM 驱动维护** — 人写原始文档，LLM 负责综合、更新、一致性检查
3. **AI coding 友好** — 为 AI coding agent 提供结构化的项目知识入口
4. **工程师友好** — 目录结构符合软件工程惯例，支持 Obsidian 浏览编辑
5. **零外部依赖** — 纯 Claude Code Skills 实现，无需数据库或服务器

## 三层架构

### 第一层：原始文档（`docs/raw/`）

原始文档归档。每次 ingest 的原始内容存放于此。文件通常保持稳定，需要变更时通过 `/docs-update` 操作，变更经 git history 和 log.md 双重追踪。

内部按软件工程文档分类组织子目录：

```
docs/raw/
├── specs/           # 设计规格（brainstorming 产出）
├── plans/           # 实现计划（writing-plans 产出）
├── architecture/    # 架构文档
├── adr/             # 架构决策记录
├── api/             # API 设计文档
├── guides/          # 操作指南、Runbook、部署文档
├── prd/             # 产品需求文档
├── meeting/         # 会议纪要、技术决策记录
└── ...              # LLM 可按需创建新分类
```

每个文件带 frontmatter 元数据：

```markdown
---
title: 用户认证系统设计
source_type: file          # file | text | url | skill
ingested_at: 2026-04-08
source_url:                # URL 输入时记录原始地址
skill_name:                # skill 产出时记录来源 skill
tags: [architecture, auth]
immutable: true
---
（原始内容）
```

**分类规则**：ingest 时由 LLM 根据内容语义自动决定归入哪个子目录。`docs/schema.md` 中定义初始分类体系，LLM 可按需提议新增分类（需经用户确认）。

### 第二层：当前 Wiki（`docs/wiki/`）

LLM 自主维护的当前知识库。LLM 完全控制此目录的结构 — 自行决定创建哪些页面、如何组织、如何交叉引用。

每个 wiki 页面带 frontmatter 标记来源追溯：

```markdown
---
title: 认证系统架构
sources:
  - "[[raw/specs/2026-04-08-auth-design]]"
  - "[[raw/specs/2026-04-10-auth-update]]"
last_updated: 2026-04-10
tags: [architecture, auth]
---
（LLM 综合生成的当前设计文档，使用 [[wiki-links]] 交叉引用）
```

**第一层 vs 第二层的区别**：

| | 第一层（docs/raw/） | 第二层（docs/wiki/） |
|---|---|---|
| 谁写 | 人 / skill / ingest | LLM 维护 |
| 可变性 | 通常稳定，变更需经 `/docs-update` | 持续更新 |
| 内容性质 | 某个时间点的原始记录（可演进） | 综合提炼的当前状态 |
| frontmatter | 带 `source_type`、`ingested_at` | 带 `sources: [...]` |

### 第三层：AI 入口（`docs/schema.md` + `docs/README.md`）

**`docs/schema.md`** — 系统的灵魂，共享架构定义：

- 目录结构约定
- 文档格式规范（frontmatter 字段、交叉引用语法）
- 分类体系（初始分类 + LLM 自适应扩展规则）
- 操作契约（ingest/lint/query 的输入输出约定）

`schema.md` 本身也可以被 LLM 演进 — 随着项目文档增多，LLM 可提议调整分类体系或组织方式，但需经用户确认。

**`docs/README.md`** — Wiki 导航索引：

- 每次 ingest 后由 LLM 更新
- 按主题分类，每个条目带一行摘要和 `[[wiki-link]]`
- AI coding agent 通过读取此文件来决定需要深入阅读哪些 wiki 页面
- 在 GitHub 上打开 `docs/` 目录时直接渲染为导航页

**`docs/log.md`** — 操作审计日志（append-only）：

- 仅记录产生文件变更的操作（ingest、update、lint 修复、query 回写）
- 纯查询操作不记录
- 格式：`## [YYYY-MM-DD] operation | subject`

## 完整目录结构

```
project-root/
├── CLAUDE.md                              # 引用 docs/schema.md
├── .claude/
│   └── skills/
│       ├── docs-init/
│       │   └── SKILL.md
│       ├── docs-ingest/
│       │   └── SKILL.md
│       ├── docs-update/
│       │   └── SKILL.md
│       ├── docs-lint/
│       │   └── SKILL.md
│       └── docs-query/
│           └── SKILL.md
├── docs/
│   ├── README.md                          # 第三层：wiki 导航索引
│   ├── schema.md                          # 第三层：架构约定 + AI 入口
│   ├── log.md                             # 操作审计日志
│   ├── raw/                               # 第一层：不可变原始文档
│   │   ├── specs/
│   │   ├── plans/
│   │   ├── architecture/
│   │   ├── adr/
│   │   ├── api/
│   │   ├── guides/
│   │   ├── prd/
│   │   ├── meeting/
│   │   └── ...
│   └── wiki/                              # 第二层：LLM 维护的当前知识库
│       └── ...
```

## Obsidian 兼容

`docs/` 目录设计为可直接用 Obsidian 打开的 vault：

1. **交叉引用统一使用 `[[wiki-links]]`** — wiki 页面之间、wiki 对 raw 的 sources 引用均使用此语法
2. **frontmatter 兼容 Obsidian** — `tags` 使用 YAML 列表格式
3. **`docs-init` 时可选生成 `.obsidian/` 配置** — 询问用户是否需要，如需要则生成基础配置

## 五个 Skills

### Skill 1: `docs-init`

初始化文档管理系统。

**调用方式**：`/docs-init`

**流程**：

1. 创建 `docs/` 目录结构（`raw/` 各子目录、`wiki/`）
2. 生成初始 `docs/schema.md`（默认分类体系 + 约定）
3. 生成空的 `docs/README.md` 和 `docs/log.md`
4. 将 `docs/schema.md` 的引用写入 `CLAUDE.md`
5. 询问用户是否需要 Obsidian 支持
   - 是 → 生成 `docs/.obsidian/` 基础配置
   - 否 → 跳过
6. 询问用户："是否扫描项目中已有的文档进行交互式导入？"
   - 是 → 扫描项目（README.md、docs/ 目录、*.md 文件等），列出发现的文档，用户选择后批量 ingest
   - 否 → 完成
7. git commit

**CLAUDE.md 写入内容**：

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

### Skill 2: `docs-ingest`

添加新文档到系统中，并更新 wiki。

**调用方式**：

```
/docs-ingest                               # 默认：从当前分支 vs main 自动发现待归档文件
/docs-ingest path/to/file.md              # 文件输入
/docs-ingest "我们决定放弃 Redis..."        # 自由文本输入
/docs-ingest https://...                   # URL 输入
```

**自动发现模式**（无参数）：通过 `git diff --name-only main...HEAD` 找出当前分支相对于 main 新增或修改的文件，筛选出文档相关文件（`.md` 文件、匹配 spec/design/adr/rfc/plan 等模式的文件），排除已在 log.md 中记录过的文件，列出候选文件供用户选择后批量 ingest。

**流程**：

1. 读取 `docs/schema.md` 获取当前约定
2. 获取输入内容
   - 无参数 → 自动发现模式，从分支 diff 中筛选候选文件
   - 文件 → 读取文件内容
   - 自由文本 → 直接使用
   - URL → 通过 WebFetch 抓取
3. LLM 分析内容
   a. 自动分类，决定归入 `raw/` 的哪个子目录
   b. 生成 frontmatter（tags、title 等）
   c. 提取关键实体、概念、决策
4. 写入 `docs/raw/<category>/`（带 frontmatter，日期前缀命名）
5. 检测 `docs/raw/` 中是否有其他未处理的文件（有 raw 文件但 log.md 中无对应 ingest 记录），如有则提示用户是否一并处理
6. 更新 `docs/wiki/`
   a. 读取 `docs/README.md` 了解现有 wiki 结构
   b. 决定：创建新页面 / 更新现有页面 / 两者都做
   c. 维护页面间的 `[[wiki-links]]` 交叉引用
7. 更新 `docs/README.md`（添加/调整条目）
8. 追加 `docs/log.md`
9. 向用户展示本次变更摘要
10. git commit

**未处理文件检测**：当其他 skill（如 brainstorming、writing-plans）直接向 `docs/raw/` 输出文件时，这些文件的 raw 归档已完成，但 wiki 尚未更新。ingest 在执行时检测到这种情况后提示用户，确保 wiki 保持同步。

### Skill 3: `docs-update`

原地更新 raw/ 文档，并同步 wiki。

**调用方式**：

```
/docs-update                                                                   # 默认：从提交记录自动发现需更新的文档
/docs-update --from-commits HEAD~10..HEAD                                      # 指定提交范围
/docs-update docs/raw/specs/2026-04-08-design.md --from-commits               # 指定文件 + 从提交记录
/docs-update docs/raw/specs/2026-04-08-design.md "add error handling section"  # 指定文件 + 手动原因
```

**流程**：

1. 读取 `docs/schema.md` 获取当前约定
2. **自动发现模式**（默认，无参数或仅 `--from-commits`）：
   a. 确定提交范围：使用用户指定的范围，或自动从 log.md 最近一次 ingest/update 记录日期起算
   b. 读取范围内的 commit log，过滤掉仅修改 `docs/` 的提交
   c. 读取所有 raw 文件，构建主题映射（文件 → 标题、标签、关键概念）
   d. 将每个 commit 与相关的 raw 文档匹配（通过文件路径、commit message 关键词、模块名称）
   e. 列出需要更新的文档，用户选择（全部/逐项确认/跳过）
3. **手动模式**（指定文件 + 原因）：用户直接提供变更内容
4. 对每个确认的文档：原地修改 raw 文件，保留 `ingested_at`、`source_type` 等原始 frontmatter 不变
5. 找到所有 `sources` 中引用了该 raw 文件的 wiki 页面，重新综合生成
6. 如需要，更新 `docs/README.md`
7. 追加 `docs/log.md`（记录变更原因、摘要、来源为 manual 或 commits (range)）
8. 向用户展示变更摘要
9. git commit

**设计动机**：虽然 raw/ 文件设计为稳定记录，但实际工作中 spec 和 plan 在实施过程中会演进。与其让文档与现实脱节，不如提供有审计追踪的正式变更通道。git history 提供完整 diff，log.md 记录变更语义（为什么改、改了什么）。默认的自动发现模式让 `/docs-update` 成为一个"一键同步"操作——只需在实现阶段结束后运行，skill 自动分析提交记录并找出需要更新的文档。

### Skill 4: `docs-lint`

检查文档一致性，审计文档与代码的匹配度。

**调用方式**：`/docs-lint`

**流程**：

1. 读取 `docs/schema.md` 获取当前约定
2. **文档内部一致性检查**
   a. `docs/README.md` 中的 `[[wiki-links]]` 是否都指向存在的页面
   b. wiki 页面的 `sources` 引用是否指向存在的 raw 文件
   c. 是否有孤立的 wiki 页面（README.md 中无入链）
   d. wiki 页面之间是否存在矛盾描述
3. **文档-代码深度审计**
   a. 读取代码中的关键结构（API 端点、数据模型、配置项、目录结构、函数签名）
   b. 与 wiki 中的描述逐项对照
   c. 识别差异：
      - 文档描述了代码中不存在的东西
      - 代码中存在但文档未覆盖的东西
      - 文档描述与代码行为不一致
4. **输出 lint 报告**
   - 🔴 错误：文档与代码矛盾
   - 🟡 警告：文档可能过时、覆盖缺失
   - 🟢 通过：一致性确认
5. 询问用户是否自动修复（更新 wiki 页面）
6. 如有修复 → 追加 `docs/log.md` + git commit

### Skill 5: `docs-query`

查询文档系统，支持开发者问答和 agent 上下文注入两种模式。

**调用方式**：

```
/docs-query "支付回调的重试策略是什么？"           # 开发者模式（默认）
/docs-query --context auth-service/handler.go     # Agent 模式
```

#### 开发者模式

1. 读取 `docs/README.md` 定位相关页面
2. 读取相关 wiki 页面，必要时回溯 raw 原始文档
3. 综合回答，附引用来源（`[[wiki-link]]` 格式）
4. 询问："这个回答是否值得写入文档？"
   - 是 → 执行 mini-ingest：将回答结构化后写入 raw/，更新 wiki，追加 log.md，git commit
   - 否 → 结束（不写 log.md）

#### Agent 模式

1. 分析 `--context` 指定的文件/目录，理解 agent 当前工作上下文
2. 读取 `docs/README.md`，筛选与上下文相关的 wiki 页面
3. 读取相关页面，提取与当前工作直接相关的信息
4. 输出结构化的上下文摘要，供 agent 直接消费：
   - 设计意图
   - 约束条件
   - 相关历史决策
   - 注意事项
5. 不写 log.md（无副作用查询）

## 与其他 Skills 的集成

### 产出文件路径配置

在项目 `CLAUDE.md` 中声明，利用 skill 的"用户偏好覆盖默认路径"机制：

- brainstorming spec → `docs/raw/specs/`
- writing-plans plan → `docs/raw/plans/`

### 未处理文件检测

当 skill 直接向 `docs/raw/` 输出文件后，文件已归档但 wiki 未同步。`docs-ingest` 执行时自动检测这种情况并提示用户。检测逻辑：对比 `docs/raw/` 中的文件与 `docs/log.md` 中的 ingest 记录，找出差集。

## Skill 技术规范

所有 skill 遵循 Claude Code Skills 规范：

- 每个 skill 一个目录，入口文件为 `SKILL.md`
- frontmatter 中声明 `name`、`description`、`allowed-tools`、`user-invocable` 等
- `$ARGUMENTS` 接收用户参数
- 执行时先读取 `docs/schema.md` 获取当前项目约定

### 权限声明

```yaml
# docs-init
allowed-tools: Read Write Bash Glob Grep

# docs-ingest
allowed-tools: Read Write Edit Bash Glob Grep WebFetch

# docs-update
allowed-tools: Read Write Edit Bash Glob Grep

# docs-lint
allowed-tools: Read Write Edit Bash Glob Grep

# docs-query
allowed-tools: Read Glob Grep WebFetch
```

## 设计原则

1. **可控变更** — raw/ 中的文件通常保持稳定，需要修改时通过 `/docs-update` 进行，变更经 git history 和 log.md 双重追踪
2. **单一信息源** — wiki/ 是"当前状态"的唯一权威来源
3. **最小侵入** — 不改变项目现有结构，docs/ 是唯一新增目录
4. **渐进式** — 冷启动即可用，随使用逐步丰富
5. **可审计** — log.md 记录所有有副作用的操作
6. **Obsidian 兼容** — `[[wiki-links]]`、YAML frontmatter、可选 vault 配置
