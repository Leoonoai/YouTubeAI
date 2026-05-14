---
id: decision-2026-05-personal-knowledge-lint-5-rules
title: personal-knowledge-lint 5 类规则 + 仅扫 personal/external 范围 + pre-commit 非阻断
status: active
confidence: 85
tags:
  - lint
  - schema
  - knowledge
  - anti-drift
source: task-263
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

新增 `scripts/personal-knowledge-lint.mjs` 检查 5 类问题：

1. **必需字段**（warn）— title / type / status / confidence / createdAt / source / sourceTasks / iteration
2. **supersedes 冲突**（warn）— A→B 但 B 没标 superseded:true
3. **sensitivity 泄漏**（**error**）— sensitivity=high + 显式非 ai_augmented profile
4. **stale reference**（warn）— fetchedAt 超 90 天 + distilled !== true + 无 distilledTo
5. **coverage gaps**（info）— sourceTasks × iteration 聚合数 < 阈值

**范围限定**：仅扫 P001 `personal/*` + `references/external/*`，**不扫**
`lessons/decisions/solutions/inbox/tasks`（避免一次性收紧全 corpus）。

**集成**：pre-commit hook **非阻断**（仅 stderr warn）；CI 用 `--strict` 任一 error → exit 1。

## 理由

iter2：知识库长出来后，schema drift / supersedes 链断 / 隐私字段误标等问题
不能靠 reviewer 肉眼盯。lint 是**主动发现**机制。

非阻断 + 范围限定的理由：
- 强制阻断会把 lint 变成开发摩擦，用户回避加新维度（167 lesson）
- 全 corpus 一次性收紧会有 200+ legacy warning，淹没真问题
- 收紧路径：先在 personal/external 推 schema → 再扩到 lessons/decisions（如果有信号）

## 操作规范

- 写新 personal lesson/decision 必须含必需字段（CI 会 warn）
- supersedes 冲突看到立即 fix（A→B 时 B 加 superseded:true）
- sensitivity=high 必须配 retrievalProfiles=[ai_augmented]（lint error 阻 commit）
- references/external fetchedAt 超 90d → 写 distilled:true + distilledTo:<slug>
  或者重新拉新版本

## 参考

- spec: `.imoss/rules/core/personal-knowledge-lint.md`
- 实现: `scripts/personal-knowledge-lint.mjs`
- 相关：[[2026-05-08-decision-personal-knowledge-lint-scope-must-be-explicit-task263]]

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-08-lesson-lint-scope-explicit-not-corpus-wide-task263|lint 范围必须显式 scope，不要一次性收紧全 corpus]]
