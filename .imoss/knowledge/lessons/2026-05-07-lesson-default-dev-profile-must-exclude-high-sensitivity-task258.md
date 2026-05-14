---
title: 默认 dev profile 必须主动排除 sensitivity=high，不能依赖调用方筛
type: lesson
status: active
confidence: 85
tags:
  - retrieval
  - sensitivity
  - profile
  - default
source: task-258
createdAt: 2026-05-07T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# 默认 dev profile 必须主动排除 sensitivity=high，不能依赖调用方筛

## 教训

profile filter 默认值 `dev` 必须在 server 侧**主动排除** `sensitivity=high` 和
`life_decision`-only 内容，**不能**返全量让调用方自己 filter。

## 原因

最初设计"profile 只决定排序权重，不做 hard filter，让调用方自己决定"。
review 阶段直接拍：开发 agent / 工具调 search 时大概率默认 profile，
如果 high sensitivity 个人记录混进 dev 调用结果 → 进 system prompt → 进 commit
log → 进对外 API 响应。一次失误可以从 server 这层保护住，**不要**依赖每个调用方
都"记得 filter"。

## 后果（如果忽略）

- 个人决策 / 隐私 lesson 在不该出现的场景泄漏
- 用户写 lesson 时担心 leakage，回避把真心话写进 IMOSS
- 调用方代码必须重复 sensitivity check 逻辑，分散且容易漏

## 正确做法

```js
// optional/mcp/memory-corpus.mjs:applyCorpusFilters
if (profileGiven) {
  const profiles = item.retrievalProfiles || resolveRetrievalProfiles(item);
  if (!profiles.includes(profile)) return false;          // intersection 过滤
  const sens = item.sensitivity || normalizeSensitivity(item.frontmatter?.sensitivity);
  if (sens === 'high' && profile !== 'ai_augmented') return false;  // hard exclude
}
```

通用规则：**任何"按场景分流"的 read filter，默认值必须保守（拒绝优先）**。
Opt-in 暴露敏感数据（显式选 ai_augmented profile）总比 opt-out 安全。

## 参考

- 实现: `optional/mcp/memory-corpus.mjs:applyCorpusFilters`
- spec: `.imoss/rules/core/retrieval-profiles.md`
