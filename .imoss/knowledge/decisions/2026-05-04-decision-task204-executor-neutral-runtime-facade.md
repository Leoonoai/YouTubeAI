---
title: IMOSS runtime 先提供 executor-neutral facade，不一次性重写 Claude hooks
type: decision
status: active
confidence: 87
tags:
  - runtime
  - hooks
  - codex
  - claude
source:
  taskId: '204'
  taskDir: .imoss/tasks/done/204-抽取-executor-neutral-imoss-runt
sourceTasks:
  - '204'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

IMOSS 提供 `scripts/imoss-runtime.mjs --action ...` 与 `optional/hooks/lib/runtime-facade.cjs` 作为 executor-neutral facade，统一暴露 `session-start`、`prompt-submit`、`pre-action`、`session-stop`、`pre-compact`。Claude hook adapter 继续保留，facade 在 action 内 lazy require 现有 runner。

## 理由

`pre-tool-use` 与 `stop` 已经 lib 化，适合直接复用；`session-start` 与 `user-prompt-submit` 仍含大量业务逻辑，一次性重写风险高。facade 先提供稳定入口，让 Codex、Dashboard、Telegram bridge 后续能调用同一套能力。

## 关键约束

- 调用 hook runner 前临时设置 `IMOSS_EXECUTOR` 和 `IMOSS_SESSION`，调用后恢复 env。
- `pre-action` 返回 `{allowed, reason?, warnings?}`，不使用 Claude 专属 `{blocked}` 契约。
- hook runner 必须 lazy require，避免模块加载时缓存错误 executor/session。

## 证据

任务 204 的单测覆盖 runtime facade 与 handoff/precheck/wrap 相关路径，CLI smoke 验证了 `pre-action` 不返回 Claude 专属 `blocked` 字段。
