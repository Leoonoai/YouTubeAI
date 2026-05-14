---
id: decision-2026-05-memory-graph-relations-timeline
title: imoss_memory_search 在 profile=ai_augmented 时附 relations + timeline
status: active
confidence: 85
tags:
  - mcp
  - retrieval
  - profile
  - graph
  - timeline
  - ai_augmented
source: task-269
createdAt: 2026-05-08T00:00:00.000Z
relatedDecisions:
  - 2026-05-07-decision-retrieval-profiles-and-sensitivity-task258
  - 2026-05-08-decision-mcp-search-profile-abstain-task270
  - 2026-05-08-decision-life-decision-trace-task268
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

`imoss_memory_search` 当 `profile === 'ai_augmented'` 时 metaBlock 额外附：

```js
{
  relations: [
    { from: "memory/P001/...", to: "knowledge/P001/...", type: "lesson" },
    ...
  ],
  timeline: [
    { id, when: "2026-01-15", source: "happenedAt" | "createdAt" | "unknown" },
    ...
  ]
}
```

**约束**：
- 仅 `ai_augmented` profile 启用（其他 profile 不返；life_decision 已有 268 trace）
- 基于最终 capped hit 集合（不扫全 corpus）
- relations 复用 198 现有 frontmatter `relatedKnowledge / relatedLessons /
  relatedDecisions / relatedSolutions` 字段，**不**新增 `relations:` schema
- target 必在 hit 集合内；同 slug 多处命中跳过 ambiguous edge（防跨项目同名连线）
- timeline `when = happenedAt > createdAt fallback`；接受 string 和 JS Date 实例
  （gray-matter YAML 解析）；null 排最后

## 理由

iter4 from "life-memory retrieval needs relationship graph" lesson + TOBUGraph 启发：
- 个人记忆有时序 + 关系特征（A 因为 B 而发生 / X 影响了 Y）
- 纯独立 hit list 丢失关系结构，AI 综合推理时无法回放事件链

v1 决策的"轻"取舍：
- **不引入 graph DB**：实时 in-memory 计算 + capped hits 集合（≤ limit）量级小
- **不新增 frontmatter schema**：复用 198 已有 relation 字段（兼容 4 种格式：
  string slug / `{slug}` / `{type, slug}` / array）
- **不强制 enum**：edge `type` 来自字段名（lesson/decision/...）+ 可选用户自定义

## 操作规范

写新 lesson/decision 时：
- 标 relations：用 `relatedKnowledge: [{type: 'caused-by', slug: 'lesson-x'}]`（v1
  type 是 free-form）
- 标时序（可选）：frontmatter 加 `happenedAt: 2026-01-15`（事件实际发生时间，
  可能早于 createdAt 当前记录时间）
- 旧 lesson 不强制回填，timeline 自动 fallback createdAt

调用方消费：
- AI 综合推理时拿 relations + timeline 重建事件链 + 因果推断
- dashboard 可视化（v2 follow-up）

## 参考

- spec: `.imoss/rules/core/retrieval-profiles.md` 269 段
- 实现: `optional/mcp/memory-graph.mjs:extractRelations + buildTimeline`
- 198 现有 frontmatter 关系字段规范见 knowledge-link-check.mjs

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-08-lesson-graymatter-yaml-parses-dates-as-date-objects-task269|gray-matter 把 YAML 无引号 ISO 日期解析成 JS Date 实例 — date helper 必须支持]]
