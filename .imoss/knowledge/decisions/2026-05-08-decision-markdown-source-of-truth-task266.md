---
id: decision-2026-05-markdown-source-of-truth
title: knowledge 改动只能改 markdown — SQLite/FTS/索引/cache 都是派生层
status: active
confidence: 90
tags:
  - architecture
  - invariant
  - markdown
  - derived-cache
source: task-266
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

写规则文档 `.imoss/rules/core/markdown-source-of-truth.md` 明确架构 invariant：

- **事实源**（user/agent 唯一可改）：
  - `.imoss/knowledge/<subdir>/*.md`
  - `.imoss/references/external/*.md`
  - `.imoss/inbox/*.md`
  - `.imoss/tasks/<id>/{plan,progress,findings,review-log}.md`
  - `.imoss/rules/**/*.md`
- **派生层**（可删可重建）：
  - SQLite 表（dashboard）
  - FTS 索引
  - MCP corpus cache
  - dashboard cache / vector embeddings

**任何对 knowledge 的改动**必须改 markdown，**禁止**直接改 SQLite / cache 后期望
markdown 自动跟进。

`sync-rebuild.mjs` 必须 idempotent（跑两次 knowledge 表 count + sample 一致）。
266 已实测验证（count=174 一致）。

## 理由

iter3 supersedes iter4 的 lesson 升级：从"index 必须可重建缓存"到"agent memory
should be curated by task agent"——前者已经是核心；后者是更深层结构（task
agent 主动 curate）。

派生层和事实源混用是技术债灾区：
- 用户改 SQLite 直接 → 下次 sync-rebuild 覆盖
- agent 改 cache 直接 → 下次重启全丢
- 调试时不知道"真相在哪个表里"

把这条变成显式 invariant + idempotent 验证 + CI gate（如果 sync-rebuild 不
idempotent 就 break build），保证未来不退化。

## 操作规范

- 工具如 sync-rebuild / lint / refactor 都直接读 markdown，写时也只写 markdown
- 派生表完全靠 sync-rebuild 重建（每次跑结果相同）
- 如果未来某个 schema 派生不出来 → 在 markdown 里加显式字段（不是在派生表里加）
- 显式化 199 + 230 + memory-corpus cache 已有方向，保证未来不退化

## 参考

- spec: `.imoss/rules/core/markdown-source-of-truth.md`
- 验证: `node scripts/sync-rebuild.mjs` 跑两次 → count + sample 一致
- 相关：[[2026-05-08-decision-code-graph-realtime-not-sqlite-task272]]（code graph 也不入 SQLite）

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-08-lesson-sync-rebuild-must-be-idempotent-task266|sync-rebuild 必须 idempotent — 跑 N 次结果与跑 1 次相同]]
