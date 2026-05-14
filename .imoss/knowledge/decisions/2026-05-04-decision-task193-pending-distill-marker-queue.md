---
title: 归档任务蒸馏采用 pending marker 队列
type: decision
status: active
confidence: 88
tags:
  - knowledge
  - distillation
  - task-archive
  - hooks
source:
  taskId: '193'
  taskDir: .imoss/tasks/done/193-任务归档时自动蒸馏会话到-knowledge-库
sourceTasks:
  - '193'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

任务归档后不直接阻塞式生成 knowledge，而是在 `task-archive.mjs` 完成 `active -> done` 后写入 `.imoss/knowledge/pending-distill/<projectId>-<taskId>.json` marker。下次 agent 对话由 `user-prompt-submit` 提示优先运行 `distill-knowledge`，再读取归档任务材料生成 decision/lesson 并删除 marker。

## 理由

归档动作必须可靠完成，knowledge 蒸馏属于异步整理工作。marker 队列让归档本身不被 LLM 调用、文件缺失或低置信度判断拖住，同时保证后续会话能看到明确待处理清单。

## 操作规范

- marker 只保存任务 id、项目 id、任务目录、标题、创建时间和 pending 状态。
- `distill-knowledge` 必读 `plan.md`，`progress.md`、`findings.md`、`review-log.md` 缺失时按空处理。
- confidence >= 80 写 active；50-79 写 draft；<50 不写 knowledge 但仍清 marker。
- 自动蒸馏只产 decision/lesson，不产 solution。

## 证据

任务 193 的端到端验证覆盖了 marker 生成、hook 提示、低置信度跳过和高置信度产出分支。

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-06-decision-task225-external-knowledge-augmentation|查询时按需联网补全外部资料的三层 skill 架构（task 225）]]
