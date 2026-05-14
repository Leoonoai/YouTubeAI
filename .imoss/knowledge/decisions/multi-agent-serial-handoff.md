---
id: decision-2026-04-multi-agent-handoff
title: 多代理协作默认用串行 handoff，不引入并发机制
status: active
confidence: 90
tags:
  - multi-agent
  - handoff
  - workflow
  - architecture
createdAt: 2026-04-05T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策
**IMOSS 的多代理协作默认只支持串行 handoff**：一个 agent 做完或做不下去后，通过 `last-handoff.json` 和 `assignedAgent` 字段把任务交给另一个 agent。不引入 lease / worktree / inbox / MCP 总线等并发机制。

## 背景
2026-04-05 用户尝试让 Claude Code 做任务 006 (Dashboard 重构)，经过 v1/v2/v3 三轮仍未达预期视觉标准，决定交给 Codex 继续。这是 IMOSS 第一次发生跨 agent 接力。

用户随后问"两个 agent 如何协同处理一个任务"，任务 008 为此做了设计调研，列出 5 套方案（文件约定/lease/worktree/inbox/MCP 总线）。但进一步对话澄清：用户没有真正的并发需求，只是串行接力场景。

## 理由
1. **现有机制已覆盖串行 handoff**：
   - plan.md 的 `assignedAgent` 字段 + `previousAgent` 链
   - `.imoss/last-handoff.json` 指向当前任务和 agent
   - 四文件任务模式的 context.json 保留交接上下文
   - git 作为原子真源，天然支持换人继续
2. **并发机制都是过度工程**（对当前场景）：
   - lease 需要管理器 + TTL + 冲突解决
   - worktree 需要协调者 + 定期 merge + 合并冲突处理
   - inbox/outbox 需要消息协议设计
   - MCP 总线需要常驻服务，与"文件是 SoT"原则冲突
3. **YAGNI 原则**：没有真实场景的机制设计会固化假设，等真需求出现时往往还要重写。

## 如何做串行 handoff（操作规范）
1. 前 agent 完成能做的部分，commit
2. 更新 `plan.md`：改 `assignedAgent` 为下一个 agent，加 `previousAgent` 字段
3. 更新 `context.json`：`agent` 改为下一个，`workingState.nextSteps` 列清楚下一步要做什么
4. `plan.md` 的核心**必须按新四段式** (rule-core-003) 包含：
   - `# 前置调研` 节列出已读文件/已跑命令（让接手者不用重做）
   - 按迭代 v1/v2/... 明确已尝试过什么
   - "接手指南"节给出快速上手命令和未完成清单
5. 更新 `.imoss/last-handoff.json` 的 `agent` 和 `resumeHint`
6. commit

## 何时需要重新考虑并发机制（触发条件）
- 单个任务大到**一个 agent 的上下文装不下**，必须并行拆分
- 出现"A 写实现 + B 做 review"的真流水线需求（不是接力）
- 两个 agent 需要**同时**改不同的文件而且等不起
- 跨项目任务涉及多个 agent

出现上述情况时，重启任务 008（状态 deferred），从 5 个方案里挑合适的。

## 参考
- 任务 006 的 plan.md 是首个串行 handoff 的合规样本
- 任务 008 `.imoss/tasks/active/008-multi-agent-coordination/` 含 5 套方案矩阵
- 规则 `rule-core-003` (`.imoss/rules/core/plan-workflow.md`) 是 handoff 的前置要求
