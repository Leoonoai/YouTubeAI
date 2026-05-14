---
title: Hooks 必须注册到用户级 settings.json（绝对路径）
type: lesson
status: active
confidence: 95
tags:
  - hooks
  - settings
  - configuration
  - multi-window
source: task-077-hooks-debug
createdAt: 2026-04-07T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# Hooks 必须注册到用户级 settings.json

## 教训
IMOSS 的 hooks 必须注册到 `~/.claude/settings.json`（用户级），不能只放 `<project>/.claude/settings.json`（项目级）。路径必须是绝对路径。

## 原因
1. 项目级 `.claude/settings.json` 不一定被所有 Claude Code 窗口加载（取决于 Claude Code 如何识别"项目根"）
2. 进入子项目后 cwd 变化，相对路径的 hook 脚本会 MODULE_NOT_FOUND
3. 多个 tmux 窗口共用同一台机器，用户级 settings 是唯一可靠的加载点

## 后果（如果忽略）
- 所有 hooks 静默失败：context-guard 不警告、pre-tool-use 不拦截、TASK_CONSOLIDATE 不提醒
- Agent 在子项目里"裸奔"，上下文管理、安全防护、知识沉淀全部失效
- 难以排查（Claude Code 不会报 hook 加载失败）

## 正确做法
1. hooks 命令用绝对路径：`node /home/devuser/dev/imoss/optional/hooks/claude/xxx.cjs`
2. 同时写入 `~/.claude/settings.json` 和 `<imoss>/.claude/settings.json`
3. 新增 hook 时两处都要更新
4. 新建项目不需要复制 hooks（用户级已全局覆盖）
