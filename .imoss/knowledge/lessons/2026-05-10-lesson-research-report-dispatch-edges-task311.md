---
title: research report dispatch 需要显式处理完成信号和 schema 边界
type: lesson
status: active
confidence: 85
tags:
  - dashboard
  - research
  - zod
  - dispatch
  - e2e
source: task-311
audience: work
createdAt: 2026-05-10T00:00:00.000Z
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# research report dispatch 需要显式处理完成信号和 schema 边界

## 教训

把研究报告生成接入 dispatcher 时，不能只写“调用 skill 生成报告”。必须同时定义等待完成信号、空产出跳过行为、URL-safe reportId、schema 递归写法，以及没有 runId 时的手动报告模式。

task 311 的 progress 里连续暴露了几个执行期边界：zod discriminated union 与 `z.lazy` 的组合会让 discriminator 读取失败；frontmatter 里的 `date` 可能是 Date 对象而不是 string；没有传 runId 时，report worker 会因为找不到匹配 iteration event 而按空产出跳过。

## 原因

plan 审查阶段已经指出 `--wait` 不能没有完成信号，否则 dispatcher watcher 会悬挂。因此最终设计要求 `research-report` mode 写 `research_report_done` event，并在空报告时也写 `status: "skipped"`。

实现阶段又证明 schema 和事件边界必须按真实 runtime 调整。progress 记录首次 pm2 restart 后 list/read 都报 zod `Cannot read properties of undefined (reading 'type')`，原因是 `details/aside` 使用 `z.lazy` 包装后，`discriminatedUnion` 拿不到 discriminator。修复方式是让 `detailsSection/asideSection` 保持直接 ZodObject，只在 `sections` 字段内部用 `z.lazy(() => sectionSchema)` 递归。

真实 e2e 还发现 prompt builder 不能假设 frontmatter date 一定是 string。task 311 的修复是加 `toStr()` 兜底。

## 后果

如果这些边界没提前定义，报告链路会出现三类问题：

1. dashboard list/read 端点在 runtime 才因 schema 递归写法崩溃。
2. dispatcher 等待不到事件，导致 worker watcher 卡住。
3. 手动为已有研究 task 生成阶段性报告时，因为没有 runId 匹配的 iteration event 而跳过产出。

这些问题都不是 lint 能提前发现的，必须通过端点 sanity check、pm2 restart 后的真实服务验证和浏览器 e2e 暴露。

## 正确做法

新增 dispatcher mode 时，plan 里必须写清：

1. 完成事件类型和匹配字段。
2. 空产出是否也写完成事件。
3. 手动触发和自动触发是否共享同一 prompt 分支。
4. 产物 ID 是否可直接放进 URL。
5. schema 递归对象是否能被运行时 validator 正确识别。

task 311 的最终做法可复用：`research-report` mode 用 `research_report_done + taskId + runId` 作为完成信号；没有 runId 的手动报告进入 stage report mode；reportId 使用 `2026-05-10T16-38-35Z` 这类 URL-safe 格式；递归 section 只在字段层使用 `z.lazy`。

## 关联

- 来源任务：task 311
- 关键证据：`progress.md` 中 2026-05-10T16:13Z 至 16:40Z 的 zod、prompt builder、stage report mode、真实 task 252 e2e 记录
- 相关产物：`.imoss/reports/research/252/2026-05-10T16-38-35Z.json`
