---
id: decision-2026-05-life-decision-trace
title: imoss_memory_search 在 profile=life_decision 时附 decision trace
status: active
confidence: 85
tags:
  - mcp
  - retrieval
  - profile
  - audit
  - life_decision
  - trace
source: task-268
createdAt: 2026-05-08T00:00:00.000Z
relatedDecisions:
  - 2026-05-07-decision-retrieval-profiles-and-sensitivity-task258
  - 2026-05-08-decision-mcp-search-profile-abstain-task270
  - 2026-05-08-decision-knowledge-lifecycle-4-fields-task267
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

`imoss_memory_search` 当 `profile === 'life_decision'` 时 metaBlock 额外附 `trace`：

```js
{
  trace: {
    hits: [{ id, sourcePaths: [...], matchedScore }],
    filters: {
      scoped: { profile, domain, knowledgeClass, sourceType, minQuality },
      totalScanned, totalAfterFilter,
      excluded: { source, type, projectId, profile, sensitivityHigh, domain, knowledgeClass, sourceType, minQuality },
    },
    warnings: [
      { type: 'lifecycle.reviewAfterExpired' | 'lifecycle.lastValidatedStale' | 'lifecycle.obsolete' | 'conflict.supersedes', ... },
    ],
  }
}
```

仅 `life_decision` profile 启用（其他 profile 不返），`sourcePaths` 必为 repo-relative
（不返本机绝对路径）；filter 计数为**单归因**（避免重复计入）；supersedes 冲突仅
检测 hit 集合内的直接关系。

## 理由

iter4 lesson "life-decision retrieval needs audit records"：
- dev profile 调上下文 = "找到就用"
- **life_decision profile** 调"我应该不应该 X / Y 跟 Z 冲突吗" 必须知道：
  memory 来自哪里 / 哪些被过滤 / 哪些已过期 / 哪些跟其他 memory 冲突

否则 AI 给生活建议时**没有透明度**，用户无法判断决策依据是否还成立。
deterministic AI audit 启发：把审计能力放在元数据层，不阻断结果。

## 操作规范

调用方拿到 trace 后可：
- 给用户展示"这次决策依据来自哪些 memory"（hits.sourcePaths）
- 提示"被排除了多少 high-sensitivity / 已过期"（filters.excluded）
- 警告"以下证据已过期未复查"（warnings.lifecycle.*）让用户主动续期
- 警告"hits 内有 supersedes 冲突"（warnings.conflict.supersedes）让用户重审

**约束**：v1 不实施 dashboard 可视化（留 268-followup）；不在 ai_augmented 启用
trace（用 270 hit 级 injected 标记代替）。

## 参考

- spec: `.imoss/rules/core/retrieval-profiles.md` 268 段
- 实现: `optional/mcp/decision-trace.mjs` + `memory-corpus.mjs:explainCorpusFilters`

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-08-decision-memory-graph-relations-timeline-task269|imoss_memory_search 在 profile=ai_augmented 时附 relations + timeline]]
- [[2026-05-08-lesson-filter-explain-must-be-single-attribution-task268|filter 排除计数必须按"单归因"，每条 item 命中第一个 reason 后只计入该 reason]]
