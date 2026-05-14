---
title: 研究信息池用 RP-NN 作为条目身份
type: decision
status: active
confidence: 90
tags:
  - research
  - dashboard
  - queue
  - imoss
source: task-342
audience: work
createdAt: 2026-05-13T00:00:00.000Z
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

`/research` 信息池的每次入池请求使用 `RP-NN` 顺序号作为用户可见编号和 marker 文件身份，`requestId` 保留为 request uuid，`taskId` 只保留为课题关联字段。

新的 marker 生命周期统一使用 `RP-NN.json`：

- pending: `.imoss/research-pool/RP-NN.json`
- claimed: `.imoss/research-pool/processing/<host>/RP-NN.json`
- failed: `.imoss/research-pool/failed/RP-NN.json`
- processed: `.imoss/research-pool/processed/RP-NN.json`

旧 `<taskId>.json` marker 不批量迁移，读取、列表、CLI 和 UI 保持兼容，显示为 legacy。

## 背景

任务 342 的目标是把 `/research` 改成两 tab 可视化 UI，并让信息池每条 enqueue 分配独立顺序号。用户需要在对话里用类似 `RP-007` 的编号精确引用信息池条目，同时同一课题可以连续或同时存在多条研究请求历史。

调研发现旧队列使用 `<taskId>.json` 作为 marker 文件名，并对同一 task 去重。这会让同一课题的第二条 pending 或 failed 请求被阻塞，和“每条 enqueue 独立”“同课题多次入池历史”冲突。

## 理由

`taskId` 是课题身份，不是请求身份。一个课题可以有多次研究方向、不同时间的入池请求和不同处理结果；继续用 `taskId` 做 marker identity 会把这些请求压成一条。

`RP-NN` 顺序号解决两个问题：

- 对用户：聊天和 UI 都能稳定引用单条信息池记录。
- 对系统：pending、processing、failed、processed 全生命周期都有同一个 entry id，处理完后还能归档到 processed 历史。

同机并发分配通过 `.imoss/research-pool/sequence.lock` 保护 `sequence.txt` 的 read → +1 → atomic write。跨 WSL/Mac 的极低概率冲突不引入分布式锁，按既有 git 冲突处理。

## 操作规范

新增 enqueue 必须写入 `seqId`，并使用 `RP-NN.json` 作为 marker 文件名。payload 内继续保留 `taskId`、`requestId`、`topic`、`direction` 等业务字段。

处理完成后不删除 marker，而是移动到 `.imoss/research-pool/processed/RP-NN.json`，保留永久历史。

涉及 destructive 操作时，如果用户用 `taskId` 查询且匹配多条 RP entry，CLI 必须拒绝并列出候选 `RP-NN`，避免误删或误重试。

## 参考

- task 342 plan.md：前置调研 “RP-NN 顺序号方案”“marker 文件名必须从 taskId 迁到 seqId”
- task 342 review-log.md：Round 1 修正 marker identity，Round 2 接受该修正
- task 342 progress.md：Phase 1 完成 queue 状态机、worker、runner、CLI、router 同步改造
