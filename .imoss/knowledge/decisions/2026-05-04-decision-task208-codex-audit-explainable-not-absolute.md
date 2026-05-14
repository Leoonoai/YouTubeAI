---
title: Codex guardrail 硬化先做可解释审计，不承诺绝对拦截
type: decision
status: active
confidence: 86
tags:
  - codex
  - audit
  - guardrails
  - git-hooks
source:
  taskId: '208'
  taskDir: .imoss/tasks/done/208-codex-guardrail-与审计硬化
sourceTasks:
  - '208'
createdAt: 2026-05-04T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

Codex 路径的 guardrail 硬化优先提升低摩擦 wrapper 入口和可解释审计：新增 `codex-wrap append`、audit explain-file、pre-commit Codex audit opt-in/strict、`imoss_precheck commit-ready` audit 信号。不宣称能从仓库内强制拦截 Codex 原生编辑工具或 `apply_patch`。

## 理由

pure Codex 没有 Claude Code 的 PreToolUse hook，IMOSS 无法在仓库代码里拦截所有原生工具。更可落地的做法是让合规路径更顺手，并在收尾/提交时用机器可读解释区分 `apply_patch`、wrapper exec 写入、脚本生成物和真实未解释缺口。

## 规则

- `missing` 的语义是“未被 Write/Edit audit 覆盖”，不是自动等同违规。
- explain-file 只能解释实际 missing 路径，避免随意掩盖。
- strict 阻断只在显式开启时生效，避免破坏既有 Claude/人工流程。
- 高频日志追加应优先用 `codex-wrap append`。

## 证据

任务 208 的测试覆盖 append、audit explain-file、pre-commit strict 入口和 MCP smoke。
