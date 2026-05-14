---
title: Dashboard prompt-submit shim 只应注入 Codex 提交型输入
type: lesson
status: active
confidence: 86
tags:
  - dashboard
  - codex
  - prompt-submit
  - tmux
source:
  taskId: '206'
  taskDir: .imoss/tasks/done/206-dashboard-和-telegram-输入前置-prom
sourceTasks:
  - '206'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 教训

Dashboard 向 Claude Code 窗口发送文本时，Claude 自己的 `UserPromptSubmit` hook 仍会触发；如果 Dashboard 也注入一次，会重复提示。prompt-submit shim 只应对 Codex executor 且 `pressEnter=true` 的提交型输入启用。

## 实施要点

- `pressEnter=false` 只是把文字放进输入框，不视为一次 prompt submit。
- shim 应在写 tmux buffer 前运行，并对注入后的最终文本再做字节上限校验。
- runtime 错误应 fail-open，不阻断用户发送，但返回 metadata 便于诊断。
- Dashboard server 调用 CommonJS runtime helper 时使用运行时根路径加载，避免 esbuild 打包后相对路径失效。

## 证据

任务 206 的测试覆盖 Codex 注入、Claude 不重复注入、非提交不注入、runtime 失败 fail-open，以及注入后超限报错。
