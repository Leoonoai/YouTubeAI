---
id: decision-2026-05-knowledge-lifecycle-4-fields
title: knowledge lifecycle 4 字段（maturity/importance/reviewAfter/lastValidatedAt）
status: active
confidence: 85
tags:
  - knowledge
  - lifecycle
  - maturity
  - importance
  - reviewAfter
source: task-267
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

knowledge frontmatter 加 4 个 lifecycle 字段（都可选）：

- **`maturity`**: `draft | reviewed | mature | obsolete` — 内容成熟度
- **`importance`**: `low | medium | high` — 重要性
- **`reviewAfter`**: `YYYY-MM-DD` — 何时该重审（过期标志）
- **`lastValidatedAt`**: `YYYY-MM-DD` — 最近一次确认仍有效

缺省 = null（**无 derived default** — 用户主观评级）。

**263 lint rule 6 lifecycle**（仅扫 personal）：
- 非法 enum/date warn
- maturity 缺失 + createdAt 超 90d warn
- reviewAfter 已过期 warn
- lastValidatedAt 超 180d warn
- reviewed | mature 缺 lastValidatedAt warn

## 理由

iter4 from "ByteRover Context Tree" 启发：长期个人知识需要时效 / 重要性元数据，
否则随时间 rot。

**关键决策**：lifecycle **不入 MCP search hard filter**：
- 是浏览 / lint 维度，不是检索过滤维度
- 让 270 abstain（reviewAfter < today → ai_augmented hit injected:false）通过 hit 级
  marking 消费这些字段
- Dashboard `/knowledge` 加 maturity + importance dropdown（含 unrated）；
  row badge 显示 4 字段

## 操作规范

- 写新 lesson/decision 时按需手动标（用户主观评级）
- 旧文件不强制回填（lint warn 不阻断）
- 重要 decision 标 `maturity: reviewed importance: high reviewAfter: <半年后>`
  确保到期重审

## 参考

- spec: `.imoss/rules/core/knowledge-lifecycle.md`
- 实现: `optional/mcp/memory-corpus.mjs:resolveMaturity / resolveImportance / resolveReviewAfter / resolveLastValidatedAt`
- 270 hit 级消费: [[2026-05-08-decision-mcp-search-profile-abstain-task270]]

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-08-decision-life-decision-trace-task268|imoss_memory_search 在 profile=life_decision 时附 decision trace]]
- [[2026-05-08-lesson-lifecycle-not-search-filter-task267|lifecycle 字段是浏览 / lint 维度，不要把它们做成 MCP search hard filter]]
