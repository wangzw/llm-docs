---
title: LLM-Docs 系统概览
sources:
  - "[[raw/specs/2026-04-08-llm-docs-design-spec]]"
last_updated: 2026-04-08
tags: [architecture, documentation]
---

# LLM-Docs 系统概览

LLM-Docs 是一套基于 LLM 驱动的软件工程文档管理系统，以 Claude Code Skills 形式交付。

## 核心理念

借鉴 Karpathy 的 LLM Wiki 模式：人负责写原始文档和做决策，LLM 负责综合、更新、一致性检查。文档不再是写完就腐烂的静态产物，而是 LLM 持续维护的活的知识库。

## 三层架构

- **第一层 `raw/`**：不可变的原始文档归档。按软件工程文档类型分子目录（specs、plans、architecture、adr、api、guides、prd、meeting 等）。写入后永不修改。
- **第二层 `wiki/`**：LLM 自主维护的当前知识库。综合提炼 raw/ 中的历史文档，反映项目的当前状态。LLM 完全控制此目录的组织结构和交叉引用。
- **第三层 `schema.md` + `README.md`**：AI coding agent 的入口。schema.md 定义系统约定，README.md 提供 wiki 导航索引。

## 四个操作 Skill

| Skill | 功能 | 调用方式 |
|-------|------|----------|
| `/docs-init` | 初始化文档系统 | `/docs-init` |
| `/docs-ingest` | 添加文档 → 更新 wiki | `/docs-ingest <file\|text\|url>` |
| `/docs-lint` | 文档一致性 + 代码审计 | `/docs-lint` |
| `/docs-query` | 开发者问答 / agent 上下文注入 | `/docs-query <question\|--context file>` |

## 设计原则

1. **不可变性** — raw/ 写入后不修改
2. **单一信息源** — wiki/ 是"当前状态"的唯一权威
3. **最小侵入** — docs/ 是唯一新增目录
4. **渐进式** — 冷启动即可用，随使用逐步丰富
5. **可审计** — log.md 记录所有有副作用的操作
6. **Obsidian 兼容** — `[[wiki-links]]`、YAML frontmatter

## 与其他 Skill 的集成

brainstorming 的 spec 输出到 `docs/raw/specs/`，writing-plans 的 plan 输出到 `docs/raw/plans/`。通过在 CLAUDE.md 中配置路径覆盖实现。当这些文件被直接写入 raw/ 但 wiki 尚未同步时，`/docs-ingest` 会检测并提示用户。

## 相关文档

- 完整设计规格：[[raw/specs/2026-04-08-llm-docs-design-spec]]
- 系统约定：[[schema]]
