---
title: MCP memory 使用 IMOSS 与 Codex 一等路径，保留 legacy Claude 兼容
type: decision
status: active
confidence: 88
tags:
  - mcp
  - memory
  - codex
  - claude
source:
  taskId: '209'
  taskDir: .imoss/tasks/done/209-mcp-与-memory-路径去-claude-化
sourceTasks:
  - '209'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

MCP memory corpus 不再只扫描 `~/.claude/projects/*/memory`。新顺序支持每个注册项目的 `.imoss/memory`、`CODEX_HOME || ~/.codex` 下的 Codex memory、legacy Claude memory，并按 `.imoss/memory > Codex > legacy Claude` 确定性去重。access log 默认写入 `IMOSS_ROOT/.imoss/memory/_access.json`，允许 env override。

## 理由

旧路径同时绑定 Claude executor 和旧机器 slug，Codex 环境下会漏扫或写错 access log。新增 IMOSS 自有路径让 memory 成为项目资产；保留 Claude legacy 扫描避免历史经验失效。

## 实施要点

- 保持 `memory/<projectKey>/<basename>` id 兼容。
- 同名条目按来源优先级去重，避免 `imoss_memory_get` map 覆盖不稳定。
- smoke test 不能依赖本机历史 `~/.claude` 目录数量，应使用 fixture 或隔离 HOME/CODEX_HOME。
- doctor 对 Codex MCP 可见性只做 warn，不作为硬失败。

## 证据

任务 209 验证了 memory salience 测试、隔离 HOME/CODEX_HOME smoke、runtime doctor 和 diff check。
