---
id: decision-2026-05-scholar-research-codex-worker
title: scholar-research M3 用 Codex worker 跑 6 LLM 步骤，不引入 LLM SDK
status: active
confidence: 85
tags:
  - scholar-research
  - codex-worker
  - dispatch
  - no-sdk
  - architecture
source: task-277
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

scholar-research 的 6 个 LLM 步骤（step 1 课题解构 / step 4 提取 claim+method+evidence /
step 5 争议 / step 6 空白 / step 7 假设卡 / step 8 红队）**不引入 LLM SDK**，复用
254/256 的 research-codex worker pool 基础设施：

- CLI 加 `--source codex` 模式：Node 端跑 M2 真实 fetch + verifyCitations，写 papers.json /
  citations.json / worker_input.json 到 runDir
- 调 `codex-dispatch.mjs --mode scholar --task <P001> --run-id <id> --run-dir <runDir> --wait`
- Codex worker 一次性跑 6 LLM 步骤，写 7 个 worker handoff JSON（含 matrix_items.json 中转文件）
- 完成信号：events.jsonl `type=scholar_research_done` + taskId/runId 匹配 + finishedAt > triggeredAt
- CLI 收到完成后用 schema factories 校验 7 handoff，跑确定性 step 9 memo writer 拼最终 12 文件

## 理由

1. **用户原始要求"不接 LLM SDK / 不要 API key"**：避开 OpenAI/Anthropic/DeepSeek 直连
2. **research-codex pool 已就绪**（256）：windowKind=research 的 codex 窗口可被 dispatcher 选中
3. **events.jsonl 完成事件机制成熟**（254）：reverse-scan + runId 匹配 + finishedAt 防 stale 复用
4. **prompt 长度 8-10KB 必须文件化**：直接 paste 到 tmux 触发 capture-pane viewport 截断 + submit-verify
   false negative。254 已踩过坑。**任何长 prompt 都先写到 `.imoss/tmp/codex-<mode>-<runId>.md`，
   短指令让 Codex 读文件**，不要直接 paste

## 操作规范

**Worker 输出职责严格收敛**：Codex 只产 6 LLM step 输出 + matrix_items.json handoff，**不**写 CSV、
**不**复制 Paper 原始字段、**不**生成最终 12 文件。Node 端用 `scholar-schemas.cjs` factories 校验
后用 `writeOutputs` 统一拼最终输出。原因：CSV/JSON 缩进风格 + paper_id 引用一致性必须由 Node 端
单点控制，worker 复制原始字段会引入篡改风险。

**`--source codex` 必须显式传 `--task`**：不隐式建任务、不猜测。dispatcher 按 task 目录定位 plan，
用 taskId 写锁与完成信号。

**失败一次就降级 mock，不重试**：M3 范围不做 retry，retry 留 M4+。Codex 写 JSON parse 错 / 缺字段 /
超时 / 缺 research-codex 窗口 → CLI 退回 mock-data + warning + 显式提示用户。

**测试用依赖注入，不 monkey-patch import 的 spawnSync**：把 dispatch 调用抽成参数
（`runCodexWorkflow(task, ctx, { dispatch = spawnSync })`），单测注入 fake dispatcher 模拟
成功/失败路径。

**`ctx.source` 控制数据源不等于关 `ctx.mock`**：M2 阶段 live 模式仍要 `ctx.mock=true` 让 step 1/5-9
走占位实现，否则会先撞 `NotImplementedError`。M3 阶段 codex 模式才真正解锁 6 LLM 步骤。

## 参考

- 任务 277 / scripts/scholar-research.mjs / scripts/codex-dispatch.mjs (mode=scholar) /
  scripts/lib/scholar-prompt.cjs
- 复用：254 research-iterate worker dispatch / 256 research-codex pool / scholar-fetch (M2)
- 上游骨架决策：[[2026-05-08-decision-scholar-research-9步骨架契约-task275]]
- 后续 milestone：M4 [[2026-05-08-decision-scholar-dashboard-触发保持-p001-only-task278]]
- spec：`.imoss/rules/core/scholar-research.md`

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-08-decision-scholar-dashboard-触发保持-p001-only-task278|dashboard 项目级触发 scholar 仍走 P001 task，用 frontmatter 记 targetProject]]
- [[2026-05-08-decision-scholar-research-9步骨架契约-task275|scholar-research M1 9 步骨架不引入 SDK/zod/LangGraph，用 plain JS + 函数表注入点]]
