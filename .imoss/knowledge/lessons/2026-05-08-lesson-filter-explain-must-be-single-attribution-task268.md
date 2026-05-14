---
title: filter 排除计数必须按"单归因"，每条 item 命中第一个 reason 后只计入该 reason
type: lesson
status: active
confidence: 85
tags:
  - filter
  - explain
  - attribution
  - audit
source: task-268
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# filter 排除计数必须按"单归因"，每条 item 命中第一个 reason 后只计入该 reason

## 教训

写"为 search filter 输出可解释计数"helper 时，如果一条 item 同时违反多个排除规则
（例如 `profile` 不命中 + `sensitivity=high`），**只计入第一个命中的 reason**，不要
重复计入所有 reason。否则解释计数对不上 totalScanned - totalAfterFilter。

## 原因

268 explainCorpusFilters 第一版按"独立计数每个 reason"实现：每个 reason 独立检查全
corpus，命中即 +1。Codex R1 直接拍：
- 一条 item 同时是 `profile=life_decision`-only + `sensitivity=high` → 既计入
  `profile` 又计入 `sensitivityHigh`
- `sum(excluded values) = totalScanned - totalAfterFilter` 不再成立
- 给用户的"audit trace"反而误导（看起来排除了 200 条，实际只有 100 条 item 被排除）

## 后果（如果忽略）

- audit trace 计数对不上总数 → 用户 / agent 不信任 trace
- 解释为什么"过滤掉这么多"时把同一条算两遍 → 推断 root cause 错误
- 测试 `excluded.profile + excluded.sensitivityHigh = N` 这种不变量永远 false

## 正确做法

```js
// optional/mcp/memory-corpus.mjs
// 把 filter 检查抽成 ordered helper，返回第一个命中的 reason 或 null
function classifyFilterExclusion(item, opts) {
  if (!activeSources.includes(item.source)) return 'source';
  if (types && !types.includes(item.type)) return 'type';
  // ...按 applyCorpusFilters 现有顺序
  if (sens === 'high' && profile !== 'ai_augmented') return 'sensitivityHigh';
  // ...
  return null;
}

export function explainCorpusFilters(items, options) {
  const excluded = { source: 0, type: 0, ... };
  const filtered = [];
  for (const item of items) {
    const reason = classifyFilterExclusion(item, opts);
    if (reason === null) filtered.push(item);
    else excluded[reason] += 1;  // 单归因，命中即停
  }
  return { filtered, filters: { totalScanned: items.length, totalAfterFilter: filtered.length, excluded } };
}
```

通用规则：
- **任何"分类计数"**都要单归因（一条数据只被分到一个桶）
- 不变量：`sum(buckets) + included = total`
- applyCorpusFilters 和 explainCorpusFilters 共享同一 `classifyFilterExclusion`
  helper，避免语义漂移
- 测试覆盖"同时违反多 reason" 的 item，断言只计入第一个

## 参考

- 实现: `optional/mcp/memory-corpus.mjs:classifyFilterExclusion + explainCorpusFilters`
- 测试: `optional/mcp/memory-corpus.test.mjs` "single attribution" describe 块
- review-log Round 1: `.imoss/tasks/done/268-mcp-imoss-memory-search-在-prof/review-log.md`
- 相关决策: [[2026-05-08-decision-life-decision-trace-task268]]
