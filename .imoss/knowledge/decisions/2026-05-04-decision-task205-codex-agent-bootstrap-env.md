---
title: Codex agent 窗口需要显式 executor env 与 bootstrap
type: decision
status: active
confidence: 85
tags:
  - codex
  - tmux
  - bootstrap
  - windows
source:
  taskId: '205'
  taskDir: .imoss/tasks/done/205-codex-agent-window-一等化支持
sourceTasks:
  - '205'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

普通 Codex agent 窗口启动时要明确导出 `IMOSS_EXECUTOR=codex`，并默认生成 Codex bootstrap；但 reviewer 池窗口不因 command 含 `codex` 就自动收到普通 agent bootstrap。

## 理由

Codex 没有 Claude hook 层，必须靠启动环境和 bootstrap 文件恢复项目、任务、规则上下文。旧手工 Codex entry 如果没有 `codexBootstrap` 或 executor env，容易退回 unknown executor，导致 guardrail 和提示行为不稳定。

## 适用范围

- `window-register --executor codex` 注册普通 agent 时写入 `executor=codex`、`env.IMOSS_EXECUTOR=codex`、`codexBootstrap=true`。
- `restart-project-window` 对旧 Codex agent entry 推断并补 effective env/bootstrap。
- `codex-agent-start` 临时任务窗口也导出 `IMOSS_EXECUTOR=codex`。
- reviewer 池保持独立调度语义，不自动套普通 agent bootstrap。

## 证据

任务 205 的 dry-run 验证显示 effective executor、bootstrap 和启动命令均包含 Codex 语义。
