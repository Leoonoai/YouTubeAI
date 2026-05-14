---
id: decision-2026-05-inbox-capture-stateless-wrapper
title: inbox capture 入口用 stateless CLI wrapper，不开 daemon / 长期 worker
status: active
confidence: 85
tags:
  - inbox
  - capture
  - cli
  - stateless
  - protocol
source: task-264
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

外部 capture 入口（Telegram bot / 浏览器扩展 / 语音）的统一接口契约：

- **唯一入口**：`node scripts/inbox-add.mjs --json-stdin`
- **stateless wrapper**：每次调用独立 spawn，无长期 process
- **不经过主控 Claude**：外部入口不需要 IMOSS Claude / Codex 窗口运行
- **stdin JSON**：调用方 echo `{...}` 进 stdin，stdout 输出 `{ok, filePath, slug}`
- **旧 CLI args 模式向后兼容**：`--source X --text "..."` 输出裸路径

## 理由

iter3 from "专职 capture agent beats general chat" lesson：
- 不需要长期窗口 / pool（capture 是低频事件）
- 不需要 LLM（写 metadata + body 到 inbox 不需要推理）
- stateless 避免状态管理 / lifecycle 复杂性

主控 Claude 只在 triage 时读结果，不参与 capture 写入。

## 操作规范

- Telegram bot 直接 spawn `node scripts/inbox-add.mjs --json-stdin`，不调 IMOSS MCP
- 浏览器扩展走同一 wrapper（未来 259-followup-2 加 HTTP/tRPC 适配层）
- 语音转写走同一 wrapper
- `originalAttachment` 路径校验：仅相对 `attachments/` 下，拒绝 absolute / `..` / 反斜杠
- frontmatter YAML-safe 序列化：URL / 含冒号文本自动 JSON.stringify

## 参考

- spec: `.imoss/rules/core/inbox-capture-protocol.md`
- 实现: `scripts/inbox-add.mjs:buildCapture`
- 相关：[[2026-05-07-decision-inbox-capture-first-triage-later-task259]]
- 后续：[[2026-05-08-decision-memory-write-boundary-validation-task271]] 在此基础上加 4 维度校验
