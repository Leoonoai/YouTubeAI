---
id: decision-2026-05-retrieval-profiles-and-sensitivity
title: knowledge retrieval 分 5 个 profile + sensitivity 字段控读
status: active
confidence: 85
tags:
  - retrieval
  - profile
  - sensitivity
  - mcp
source: task-258
createdAt: 2026-05-07T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

引入 `retrievalProfiles` + `sensitivity` 双字段做读控制：

- **profile** 5 类：`dev | life_decision | ai_augmented | research | inbox_triage`
  - `dev`（默认）：排除 `life_decision`-only 内容 + `sensitivity=high` 内容
  - `life_decision`：用户自决场景；可见 life_decision-only 个人记录
  - `ai_augmented`：高敏感唯一暴露面（含 sensitivity=high）
  - `research` / `inbox_triage`：垂直场景
- **sensitivity** 3 类：`low | medium | high`，缺省视同 medium

MCP `imoss_memory_search/list` 接受 `profile` 参数；老 200+ knowledge
没字段也工作（`deriveDefaultRetrievalProfiles` 兜底）。

## 理由

- 257 iter1 核心发现："统一存储 ≠ 全场景全量可读"。把所有个人知识进 IMOSS，
  必须有按场景分流的 read 控制
- 写新 lesson/decision 默认不写 profile/sensitivity（auto-derive）；只对个人决策 /
  隐私显式标
- profile 维度独立于 260 domain（浏览锚点）/ 261 knowledgeClass（memory vs wiki 检索面）

## 操作规范

- `dev` 调用方（开发任务、代码 review）：默认 profile 即可
- `life_decision` 调用方（人生决策推理）：显式传 profile，配合 270 abstain
- `ai_augmented` 调用方（AI 注入 system prompt）：显式传 profile，配合 270 hit
  级 injected 标记

## 参考

- spec: `.imoss/rules/core/retrieval-profiles.md`
- 相关：[[2026-05-08-decision-mcp-search-profile-abstain-task270]]

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-07-decision-knowledge-domains-johnny-decimal-task260|knowledge domain 用 Johnny.Decimal 5 类，与 profile 维度独立]]
- [[2026-05-08-decision-life-decision-trace-task268|imoss_memory_search 在 profile=life_decision 时附 decision trace]]
- [[2026-05-08-decision-memory-graph-relations-timeline-task269|imoss_memory_search 在 profile=ai_augmented 时附 relations + timeline]]
