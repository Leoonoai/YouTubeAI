---
title: 任意文件上传必须把文件名和大小校验放在后端权威边界
type: lesson
status: active
confidence: 87
tags:
  - dashboard
  - upload
  - security
  - trpc
source:
  taskId: '214'
  taskDir: .imoss/tasks/done/214-dashboard-号按钮支持任意文件上传
sourceTasks:
  - '214'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 教训

把 Dashboard `+` 号从图片上传扩展到任意文件时，前端预检只能改善体验，安全边界必须在后端：后端不信任 `filename`，要做路径级 sanitize、大小限制、base64 round-trip 校验，并保留 `flag: 'wx'` 与 `mode: 0o600`。

## 实施要点

- Ctrl+V 粘贴图片没有可信原文件名，应保持旧命名格式。
- `+` 号选择文件可传原名，但后端只取最后一段并清理路径分隔符、控制字符、NUL、`..`、开头点号和危险字符。
- 任意文件可能产生空 MIME data URL，应归一化为 `application/octet-stream`。
- 50MiB 文件经 base64 后约 66.7MiB，tRPC body 与 data URL 上限要按编码后大小设置。
- 保存逻辑应抽成 helper，方便覆盖 path traversal、空 MIME、大小边界和权限测试。

## 证据

任务 214 新增 25 个上传 helper/Vitest 用例，全套 Dashboard 测试通过；build 成功，typecheck 仅保留既有错误基线。
