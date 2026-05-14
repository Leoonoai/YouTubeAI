---
title: 脱敏分发副本不能只信常规 leak-check 白名单
type: lesson
status: active
confidence: 86
tags:
  - oss
  - security
  - leak-check
  - onboarding
source:
  taskId: '219'
  taskDir: .imoss/tasks/done/219-脱敏拷贝-imoss-到-mntedev2imoss-并写入
sourceTasks:
  - '219'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 教训

给朋友或外部仓库制作 IMOSS 脱敏副本时，不能只依赖常规 `leak-check`。如果 `oss-manifest.yml` 把 `.imoss/tasks/**` 或 `.imoss/knowledge/references/**` 放进 allowPathPatterns，常规扫描会直接跳过这些目录，历史任务里的旧密码、路径、业务内容可能漏出。

## 正确做法

- 常规 leak-check 之外，再跑一份严格 manifest，临时移除高风险 allowPathPatterns。
- 对 `.imoss/tasks/**`、`.imoss/knowledge/**`、`.imoss/rules/**` 做项目名和敏感词复查。
- 测试 fixture 误报要加精确白名单，不要用大目录白名单掩盖。
- 分发说明书应写在副本自己的 living task 中，避免散落到对话或临时 README。

## 额外判断

面向 Windows 用户分发 IMOSS 时，当前仍应走 Windows + WSL2 路线。tmux、多窗口、Codex/Telegram 窗口管理依赖 Linux runtime，强改为纯 Windows 会破坏核心能力。

## 证据

任务 219 的执行中，常规 leak-check 曾因 allowPathPatterns 跳过历史任务；严格扫描发现旧密码并完成替换，最终常规与严格扫描均通过。
