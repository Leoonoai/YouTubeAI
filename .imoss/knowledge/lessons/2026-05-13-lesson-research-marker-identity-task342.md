---
title: 队列 marker 身份不能复用业务聚合 ID
type: lesson
status: active
confidence: 90
tags:
  - research
  - queue
  - design-review
  - imoss
source: task-342
audience: work
createdAt: 2026-05-13T00:00:00.000Z
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 教训

队列中的 marker identity 应表示“一次请求”，不能复用 `taskId` 这类业务聚合 ID。任务 342 的 review 发现，继续用 `<taskId>.json` 作为研究池 marker 文件名，会破坏“同一课题可以多次入池”的需求。

## 原因

旧研究池状态机把 marker 文件名设为 `<taskId>.json`，并基于同一个 task 做 pending / claimed 去重。这在“每个课题同一时间最多一个请求”的模型下可用，但任务 342 的目标改成了“每条 enqueue 独立”，并要求长期保存信息池历史。

review-log Round 1 明确指出：如果 pending、processing、failed、processed 仍使用 taskId 文件名，同一课题的第二条 pending 或 failed 请求会被去重或阻塞。

## 后果

如果不拆分 request identity 和 task identity，用户在对话中无法稳定引用某一次研究请求，dashboard 也无法展示同一课题的多次入池历史。

更严重的是，处理完成后如果继续删除 marker，信息池 tab 只能看到当前状态，无法回溯研究方向、请求来源和历史失败原因。

## 正确做法

为每次 enqueue 分配独立 `seqId`，用 `RP-NNN` 作为用户可见编号和全生命周期文件名。payload 内继续保留 `taskId`，只用于关联课题。

旧数据用 legacy fallback 读取，不批量迁移。人工 CLI 如果按 taskId 命中多条 entry，应拒绝误删并列出候选 RP-NN。

这类设计审查要优先确认“文件名代表的实体”是否和新需求一致：一次请求、一个课题、一次处理尝试、还是一条历史记录。实体边界错了，后续 UI 和 worker 逻辑都会被迫补丁化。

## 关联

- `.imoss/tasks/done/342-research-加可视化-ui信息池-课题双-tabrp-/plan.md`
- `.imoss/tasks/done/342-research-加可视化-ui信息池-课题双-tabrp-/review-log.md`
- `2026-05-13-decision-research-rpseq-processed-direction-task342`
