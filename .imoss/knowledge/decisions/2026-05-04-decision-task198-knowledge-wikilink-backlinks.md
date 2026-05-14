---
title: Knowledge graph 采用正文 wikilink 加 frontmatter 关系双层模型
type: decision
status: active
confidence: 90
tags:
  - knowledge
  - wikilink
  - backlinks
  - graph
source:
  taskId: '198'
  taskDir: .imoss/tasks/done/198-给-knowledge-引入-wikilink-蒸馏时自动补
sourceTasks:
  - '198'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

`.imoss/knowledge/**` 建立双层 graph：正文引用其他 knowledge 条目时使用 `[[slug]]` 或 `[[slug|显示文本]]`；frontmatter 保留结构化关系字段，并优先使用 `relatedKnowledge: [{type, slug}]`。反向引用由脚本自动生成，不手工维护。

## 理由

调研发现 knowledge 不是完全扁平：frontmatter 已有多种关系字段，但正文没有 wikilink，字段命名也漂移。双层模型能兼容历史数据，又给未来检索和浏览提供明确图关系。

## 规则

- slug 是文件名去扩展名。
- `[[...]]` 只用于 knowledge 到 knowledge 的引用，不用于任务、commit 或外部 URL。
- 自动生成的 `

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-06-decision-task225-external-knowledge-augmentation|查询时按需联网补全外部资料的三层 skill 架构（task 225）]]
- [[2026-05-06-lesson-knowledge-system-rebuild-225-230|知识库系统重建（task 225-230）跨任务设计教训]]
- [[2026-05-06-lesson-knowledge-子目录必须扁平化|knowledge 子目录必须扁平化（不能嵌套 type 子目录）]]
