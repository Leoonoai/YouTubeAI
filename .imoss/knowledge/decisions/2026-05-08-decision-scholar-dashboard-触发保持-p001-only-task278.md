---
id: decision-2026-05-scholar-dashboard-trigger-p001-only
title: dashboard 项目级触发 scholar 仍走 P001 task，用 frontmatter 记 targetProject
status: active
confidence: 85
tags:
  - scholar-research
  - dashboard
  - p001-only
  - task-metadata
  - constraint-preservation
source: task-278
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

dashboard `/research` 页让用户选择目标项目（IMOSS / block / baima 等）+ topic 触发
scholar 研究时，**实际仍创建 P001 research task**，不在子项目下建 task。目标项目用
P001 task plan.md frontmatter 字段标记：

```yaml
---
taskType: research
researchMode: oneshot
topic: "..."
scholarTargetProjectId: "P010"
scholarTargetProjectName: "block"
scholarTriggerSource: "dashboard"
runIds: ["..."]
---
```

memo 列表从 P001 task frontmatter 反查目标项目展示。

## 理由

**M3 既定 P001-only 不能破**：

- `scholar-research --source codex` 必须传 P001 task ID（dispatcher 按 taskId 写锁）
- `codex-dispatch --mode scholar` 在 validation 阶段限定 IMOSS / P001（research-codex pool 限制）
- 直接传子项目 task → dispatch 错位 + runId 反写到错的 plan.md

不为 dashboard 触发场景修改 M3 P001-only 约束（M3 已稳定 + 复杂度边界清晰），改用
P001 task + targetProject metadata 适配前端"按项目选"的展示需求。

## 操作规范

**detached spawn 立即返回的 API 不能承诺同步 runId**：runId 在 `scholar-research.mjs` 子进程
内部生成；dashboard `triggerScholarRun` 只能返回 `{ taskId, status: 'started' }`，前端通过
轮询 `listScholarMemos` 等 `sourceTaskId` 命中刚建的 P001 taskId 来识别新 memo。

**spawn 模式选择**：scholar 触发是 5-15min 长任务，用 `spawn(..., { detached: true, stdio: 'ignore' }).unref()`
比 tmux send-keys 更直接（不依赖 imoss claude 当前在线）。短任务（task-create 派发）
可用 spawnSync。

**Service 函数全部 DI**：`triggerScholarRun({ projectId, topic }, { spawnSyncFn, spawnFn, now, imossRoot })`
让单测不碰真实 task-create / tmux / research-codex pool。

## 参考

- 任务 278 / dashboard/server/services/scholar-memos-fs.ts /
  dashboard/server/routers/search.ts (scholarMemoList/scholarMemoDetail/triggerScholarRun) /
  dashboard/client/src/pages/Research.tsx
- 上游决策：[[2026-05-08-decision-scholar-research-codex-worker-代替-llm-sdk-task277]]
- spec：`.imoss/rules/core/scholar-research.md`

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-08-decision-scholar-research-codex-worker-代替-llm-sdk-task277|scholar-research M3 用 Codex worker 跑 6 LLM 步骤，不引入 LLM SDK]]
