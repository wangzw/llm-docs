# LLM-Docs

基于 LLM 的软件工程文档管理系统。

## Documentation

本项目使用 LLM 驱动的文档管理系统。

- 文档入口：docs/README.md
- 文档约定：docs/schema.md
- 在修改代码前，建议先通过 `/docs-query --context <file>` 了解相关设计背景
- 在完成重要设计决策后，通过 `/docs-ingest` 将决策归档
- Spec 文件输出到：docs/raw/specs/
- Plan 文件输出到：docs/raw/plans/
