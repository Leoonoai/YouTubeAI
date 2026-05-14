---
id: lesson-2026-04-24-codex-project-windows-start-at-imoss-root
title: 项目 Codex 窗口应从 IMOSS root 启动
type: lesson
status: active
confidence: 90
tags:
  - codex
  - tmux
  - project-window
  - cwd
  - bootstrap
createdAt: 2026-04-24T00:00:00.000Z
sourceTasks:
  - '215'
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 背景
给 xuexi 增加 Codex 窗口时，误把 `xuexi-codex` 配成了 `cwd: projects/xuexi`。用户纠正：项目 Codex 窗口应该和 Claude Code 一样，都在 IMOSS 目录下启动。

## 教训
普通项目的 Codex agent 窗口不要设置 `cwd: projects/<name>`，应由 `restart-project-window.mjs` 默认从 IMOSS root 启动。

## 原因
- IMOSS root 才能稳定加载基础 AGENTS.md、窗口 bootstrap、项目进入协议和跨项目管理脚本。
- 在子项目目录直接启动会触发 Codex 的目录 trust prompt，bootstrap 自动输入可能撞到确认流程，导致 Codex 退出到 shell。
- `entry.project` 是窗口所属项目提示，不等于启动 cwd。项目上下文应通过 IMOSS 的 session/bootstrap/enter-project 流程恢复，而不是靠把 cwd 改到子项目。

## 正确做法
`.imoss/tmux-sessions.json` 中普通项目 Codex 窗口使用：

```json
{
  "name": "<project>-codex",
  "project": "P00X",
  "role": "agent",
  "executor": "codex",
  "runtime": "wsl",
  "channel": null,
  "env": {},
  "command": "codex --dangerously-bypass-approvals-and-sandbox"
}
```

只有明确的 creative/code 子目录窗口才允许配置 `cwd`，且需要在任务计划中说明原因。
