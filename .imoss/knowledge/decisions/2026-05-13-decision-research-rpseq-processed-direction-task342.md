---
title: /research 信息池采用 RP-NN 作为请求身份并保留 processed 历史
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

/research 的信息池请求以 `seqId` 作为用户可见身份和 marker 文件名，格式为 `RP-NNN`。原有 `requestId` 保留为 request uuid，`taskId` 只作为课题关联字段。

marker 生命周期统一使用 `RP-NNN.json`：pending、processing、failed、processed 四档都以同一个 entry identity 移动。处理完成后不再删除 marker，而是归档到 `.imoss/research-pool/processed/RP-NN.json`，用于永久保留研究请求历史。

研究方向不在 UI 中写入。用户通过对话和 imoss-cc 确定方向后，写到课题 `plan.md` frontmatter 的 `direction:` 字段，并在 enqueue 时同步写入 marker 的 `direction` 字段。worker prompt 优先使用该 direction 作为本轮研究约束。

## 背景

任务目标要求 `/research` 增加两个 tab：信息池和课题。信息池需要展示 `RP-NN / taskId / topic / direction / status / requestedBy / enqueuedAt`，并能回溯 pending、processing、processed、failed 全生命周期的研究决策史。

原状态机以 `<taskId>.json` 作为 marker 文件名，并对同一 task 去重。这会阻止同一课题连续或同时存在多条研究请求，和“每条 enqueue 独立”冲突。

## 理由

`taskId` 表示课题，不表示一次研究请求。继续把 `taskId` 用作文件名会让同一课题的新请求被去重或阻塞，也会让对话中“调 RP-007 方向”这种引用没有稳定对象。

`RP-NN` 是顺序、短、可读的用户界面编号，适合在聊天和 dashboard 中引用。`requestId` 仍保留随机 uuid 语义，避免把用户可见编号和底层请求追踪混为一体。

processed 归档让信息池不只是临时队列，而是研究请求历史。旧 marker 不批量迁移，读取层以 legacy fallback 兼容，避免破坏现有数据。

## 操作规范

新 enqueue 必须分配 `seqId`，并以 `.imoss/research-pool/RP-NNN.json` 写入 pending。`sequence.txt` 维护当前最大编号，同机分配用 `sequence.lock` 包住 read、increment、atomic write。

queue 操作应以 `entryId` 为主：claim、release、markFailed、markProcessed 都移动 entry 文件；payload 内的 `taskId` 仅用于关联课题。按 taskId 做人工查询时，如果命中多条新 entry，应拒绝 destructive 操作并列出候选 RP-NN。

/research UI 保持 readonly。direction 的修改入口是对话和任务 plan，不是 dashboard input。

## 参考

- `.imoss/tasks/done/342-research-加可视化-ui信息池-课题双-tabrp-/plan.md`
- `.imoss/tasks/done/342-research-加可视化-ui信息池-课题双-tabrp-/progress.md`
- `.imoss/tasks/done/342-research-加可视化-ui信息池-课题双-tabrp-/review-log.md`
