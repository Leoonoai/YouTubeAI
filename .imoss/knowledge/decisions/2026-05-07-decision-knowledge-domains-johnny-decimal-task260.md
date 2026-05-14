---
id: decision-2026-05-knowledge-domains-johnny-decimal
title: knowledge domain 用 Johnny.Decimal 5 类，与 profile 维度独立
status: active
confidence: 85
tags:
  - knowledge
  - domain
  - taxonomy
  - johnny-decimal
source: task-260
createdAt: 2026-05-07T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

knowledge 加 `domain` 字段，固定 5 类 Johnny.Decimal 风格：

- `10-work` — 工作 / 项目相关（最常见 default）
- `20-learning` — 学习 / 调研
- `30-life` — 个人生活决策
- `40-inbox` — inbox 中的临时捕获
- `90-archive` — 已归档冷数据

MCP `imoss_memory_search/list` 接受 `domain` 参数；Dashboard `/knowledge` 加 domain
dropdown。

## 理由

iter1 from "Personal-KB 必须有稳定 human-friendly taxonomy" lesson：
- **人脑层面**需要"那条 X 在哪个箱子"的浏览锚点
- 但**不应该**让分类轴影响检索结果（已经有 258 profile + 261 class 做读控制）

Johnny.Decimal 比自定义分类好：
- 5 类够用（不超过短期记忆）
- 命名前缀（10/20/30/40/90）让目录排序天然分组
- 已被广泛验证，不必自己设计

## 操作规范

- domain 是**人类浏览锚点**，不是 read 控制（与 258 profile 独立）
- derived default 弱：reference→`20-learning` / personal+lessons+decisions+solutions+project-map→`10-work` / 其他→null
- 写新 30-life lesson **必须显式**标 `domain: 30-life`（否则默认 work）
- inbox-add CLI 新 capture 自动写 `domain: 40-inbox`（旧文件不回填）

## 参考

- spec: `.imoss/rules/core/knowledge-domains.md`
- 相关：[[2026-05-07-decision-retrieval-profiles-and-sensitivity-task258]]
