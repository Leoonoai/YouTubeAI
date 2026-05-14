---
title: 队列 marker 不应把课题身份当作请求身份
type: lesson
status: active
confidence: 90
tags:
  - research
  - queue
  - data-modeling
  - imoss
source: task-342
audience: work
createdAt: 2026-05-13T00:00:00.000Z
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 教训

当一个课题允许多次入池、并且每次入池都需要独立历史时，不能继续用 `taskId` 作为队列 marker 文件名或去重键。

任务 342 的旧设计若保留 `.imoss/research-pool/<taskId>.json`，会让同一课题的第二条请求被 pending/failed 去重挡住，无法满足“每条 enqueue 独立”和“同课题多次入池历史”。

## 原因

`taskId` 表示研究课题，`RP-NN` 表示一次信息池请求。两者是不同层级的身份：

- 一个 `taskId` 可以关联多条 `RP-NN` entry。
- 每条 `RP-NN` entry 有自己的 direction、状态、时间、处理结果和归档历史。

把两种身份混用，会导致数据模型只能表达“一课题一请求”，而真实需求是“一课题多请求”。

## 后果

如果继续使用 `taskId` 文件名：

- 同一课题无法连续入池多条不同研究方向。
- failed 或 pending marker 会阻塞后续请求。
- processed 历史无法按单条请求永久追踪。
- 用户在聊天中无法精确引用某一条信息池记录。

## 正确做法

先区分业务实体身份和队列请求身份。课题相关字段保留 `taskId`，队列 entry 单独分配 `seqId`，并用 `seqId` 贯穿 pending、processing、failed、processed 全生命周期。

旧数据不要强行批量迁移。列表、查找、CLI、UI 应兼容 legacy marker，并在界面上显示 legacy fallback。

## 关联

- task 342 plan.md：完成标准 “同一 task 可多条入池”“旧 marker 兼容”
- task 342 review-log.md：Round 1 指出 taskId 文件名会阻止同一课题多条入池，Round 2 接受该修正
- task 342 progress.md：Phase 1 记录 marker 文件名 v2 用 RP-NNN，legacy v1 marker 兼容读
