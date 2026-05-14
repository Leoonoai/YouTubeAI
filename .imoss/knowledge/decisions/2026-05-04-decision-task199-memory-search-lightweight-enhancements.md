---
title: MCP memory_search 增强保持轻量，LLM 改写外置
type: decision
status: active
confidence: 86
tags:
  - mcp
  - memory-search
  - retrieval
  - knowledge
source:
  taskId: '199'
  taskDir: .imoss/tasks/done/199-mcp-memory-search-叠加-multi-que
sourceTasks:
  - '199'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

`imoss_memory_search` 只负责轻量检索增强：支持调用方传入 `queries[]` 后用 RRF 合并；用规则推断 `types`；对 knowledge 结果按 incoming backlink 数量加权。MCP server 不引入 LLM SDK 来做 query rewrite。

## 理由

memory corpus 规模很小，纯 Node 文件扫描和规则增强足够快。把 query rewrite 交给调用方可以避免 MCP server 依赖 `@anthropic-ai/sdk`，也避免把模型选择、费用和失败模式塞进本地 server。

## 实施要点

- 用户显式传 `types` 时不被 intent classifier 覆盖。
- backlink boost 只作用于 knowledge，不作用于 executor memory。
- `_meta` 要暴露 `mode`、`expansionQueries`、`inferredTypes`、`backlinkBoosted`，方便调用方解释结果。
- 老的单 query 调用路径必须保持兼容。

## 证据

任务 199 新增 15 个检索增强单测并保持 `server.mjs` import 正常。

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-06-decision-task225-external-knowledge-augmentation|查询时按需联网补全外部资料的三层 skill 架构（task 225）]]
- [[2026-05-06-lesson-knowledge-system-rebuild-225-230|知识库系统重建（task 225-230）跨任务设计教训]]
- [[2026-05-06-lesson-knowledge-子目录必须扁平化|knowledge 子目录必须扁平化（不能嵌套 type 子目录）]]
