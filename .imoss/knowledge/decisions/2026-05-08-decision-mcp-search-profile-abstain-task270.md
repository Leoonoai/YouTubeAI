---
id: decision-2026-05-mcp-search-profile-abstain
title: imoss_memory_search 按 profile 分支返 abstain 信号 / hit 级 injected 标记
status: active
confidence: 85
tags:
  - mcp
  - retrieval
  - profile
  - abstain
  - lifecycle
source: task-270
createdAt: 2026-05-08T00:00:00.000Z
relatedDecisions:
  - 2026-05-07-decision-personal-knowledge-layer-must-be-curated
relatedLessons:
  - 2026-05-08-lesson-memory-retrieval-needs-low-confidence-rejection
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

`imoss_memory_search` 按 `profile` 分支返决策辅助信号，调用方按 profile 决定行为：

| profile | metaBlock.abstain | hit.injected |
|---|---|---|
| `life_decision` | `coverage.sufficient=false` 时 `shouldAbstain=true reason='evidence-insufficient' triggers=coverage.reasons` | 不加 |
| `ai_augmented` | 永 `false` | 每个 hit 加 `injected: bool`；`reviewAfter < today` → `injected:false reason:'stale-validation'` |
| `dev`（默认）| 永 `false`（保留宽召回） | 不加 |

## 背景

MemX 启发：低置信拒答比"找到最像的就用"更安全，尤其生活决策场景。
当前 IMOSS 已有 228 coverage 启发式（hits<3 / top_score<0.5 → sufficient=false），
但 search 仍返 hits 不发拒答信号 — 调用方拿不到"该信任 vs 不该"的判断锚点。

## 理由

1. **决策权按 profile 分**：life_decision 用户视角（要拒答），ai_augmented AI 视角
   （要标 stale 但不拒），dev 默认开发者自己判（不阻断）
2. **复用既有 lifecycle 字段**：267 的 `reviewAfter` 直接用作 ai_augmented 的 stale
   判定，不引入新元数据
3. **不阻断 hits**：v1 sufficient=false 仍返 hits，让调用方决定下一步（abstain
   信号给 agent / skill 触发外部抓取）

## 操作规范

调用方消费（[[profile-abstain]] spec）：
- life_decision profile → `if (meta.abstain.shouldAbstain) { triggerAugment() } else { useHits() }`
- ai_augmented profile → `hits.filter(h => h.injected !== false)` 才注入 system prompt
- dev profile → 自己看 `coverage.sufficient` + `score`

## 参考

- spec: `.imoss/rules/core/profile-abstain.md`
- 实现: `optional/mcp/search-coverage.mjs:computeAbstain` + `markInjectedForAiAugmented`
- 入口: `optional/mcp/server.mjs:imoss_memory_search`

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-07-decision-retrieval-profiles-and-sensitivity-task258|knowledge retrieval 分 5 个 profile + sensitivity 字段控读]]
- [[2026-05-08-decision-code-graph-realtime-not-sqlite-task272|code graph 检索面用实时扫描，不入 230 SQLite 派生表]]
- [[2026-05-08-decision-knowledge-lifecycle-4-fields-task267|knowledge lifecycle 4 字段（maturity/importance/reviewAfter/lastValidatedAt）]]
- [[2026-05-08-decision-life-decision-trace-task268|imoss_memory_search 在 profile=life_decision 时附 decision trace]]
- [[2026-05-08-decision-memory-graph-relations-timeline-task269|imoss_memory_search 在 profile=ai_augmented 时附 relations + timeline]]
- [[2026-05-08-decision-memory-write-boundary-validation-task271|memory capture 的分类 / 注入扫描 / 时态校验放在写入边界，不放搜索时]]
