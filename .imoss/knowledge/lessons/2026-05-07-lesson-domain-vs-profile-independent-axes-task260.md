---
title: domain（浏览）和 profile（读控制）是独立维度，不要让 domain 影响检索
type: lesson
status: active
confidence: 85
tags:
  - taxonomy
  - domain
  - profile
  - separation-of-concerns
source: task-260
createdAt: 2026-05-07T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# domain（浏览）和 profile（读控制）是独立维度，不要让 domain 影响检索

## 教训

knowledge 元数据有多维（profile / domain / class / sourceType / lifecycle），每一维
**职责必须明确**：

- **profile (258)** = read 控制（who can see what）
- **domain (260)** = 人类浏览锚点（where to put things）
- **knowledgeClass (261)** = 检索面区分（memory vs wiki retrieval surface）
- **sourceType (262)** = 内容质量元数据
- **lifecycle (267)** = 时效 / 重要性元数据

把 domain 当 profile 用（"30-life domain 不让 dev profile 看见"）会**双重过滤**，
混淆 read 控制和组织结构两个职责。

## 原因

设计 dashboard `/knowledge` 时一度想"30-life domain 自动 = life_decision profile"，
减少用户标注负担。但这等价：
- 用户写一条 30-life 笔记 = 不能在 dev 上下文看见
- 但有些 30-life 是开发相关（"在家办公 desk setup" / "ergonomics"）

把两个轴 collapse 反而让用户更难分类。

## 后果（如果忽略）

- 新维度加进来都想"derive 自其他维度"，每加一维耦合度爆炸
- 用户写 frontmatter 时困惑（"我标 domain 还是 profile？"）
- 工具行为难以预测（filter 是 domain 还是 profile 在生效？）

## 正确做法

5 个维度都**显式独立**：
- profile / class / lifecycle 由 search filter 用
- domain / sourceType 由 Dashboard 浏览 + Lint 用
- 派生默认值各自管自己（例 personal+lesson → class=memory；reference → profile.research）

如果某条 frontmatter 没显式标某维 → derived default 兜底；显式标了 → 直接用。
**不要**让一维的值反过来 override 另一维。

## 参考

- 多维度元数据架构: 258 / 260 / 261 / 262 / 267
- 实现: `optional/mcp/memory-corpus.mjs:applyCorpusFilters`
