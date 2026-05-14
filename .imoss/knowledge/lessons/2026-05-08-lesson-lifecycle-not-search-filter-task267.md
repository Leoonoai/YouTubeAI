---
title: lifecycle 字段是浏览 / lint 维度，不要把它们做成 MCP search hard filter
type: lesson
status: active
confidence: 85
tags:
  - lifecycle
  - search
  - filter
  - separation-of-concerns
source: task-267
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# lifecycle 字段是浏览 / lint 维度，不要把它们做成 MCP search hard filter

## 教训

新加 frontmatter 字段（如 267 maturity / importance / reviewAfter）时，要分清楚
该字段属于 **read filter 维度** 还是 **浏览 / 提醒维度**：

- read filter 维度（profile / class / sourceType）→ MCP search 直接过滤掉不
  匹配的 hit
- 浏览 / lint / 时效维度（lifecycle 4 字段 / domain）→ **不**做 search hard filter，
  通过 dashboard post-filter + lint warn + hit 级标记消费

## 原因

最初设计 267 lifecycle 时考虑过"reviewAfter 过期的 hit 直接从 search 排除"。
review 阶段拍住：
- 用户搜"过去做过类似事"时，reviewAfter 已过期的 lesson 仍是有用的历史信号，
  不该完全过滤掉
- ai_augmented profile 写 system prompt 时**才**关心时效（用 270 hit 级 injected
  标记，让 AI 看到但不注入）
- dev profile 永远希望宽召回

把 lifecycle 做成 hard filter = 一刀切，破坏召回。

## 后果（如果忽略）

- search 看不到过期 lesson → 用户以为"系统忘了" / 重复学习
- 用 270 abstain 信号的精细控制无处发力（hit 已经被 server 端过滤掉）
- 想消费 lifecycle 的工具（dashboard / lint）无 hit 可以读 metadata

## 正确做法

```js
// optional/mcp/memory-corpus.mjs:applyCorpusFilters
// lifecycle 字段不在这里 filter — search 永远返回所有命中
// 让调用方 / hit 级 markInjected (270) / dashboard post-filter 消费

// dashboard/server/services/knowledge-fs.ts
function applyLifecycleFilter(items, { matchMaturity, matchImportance }) {
  return items.filter(item => /* post-filter */);
}
```

通用规则：
- 想加新元数据时先问"这是 read 控制还是浏览/提醒"
- read 控制：进 server filter + profile 集成（少数）
- 浏览/提醒：仅 dashboard + lint + hit 级 marking（多数，这是 default）

## 参考

- 实现: `dashboard/server/services/knowledge-fs.ts` (lifecycle post-filter)
- 270 hit 级 marking: `optional/mcp/search-coverage.mjs:markInjectedForAiAugmented`
- 相关：[[2026-05-08-decision-knowledge-lifecycle-4-fields-task267]]
