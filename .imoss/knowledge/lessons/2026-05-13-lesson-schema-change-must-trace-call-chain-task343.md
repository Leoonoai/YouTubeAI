---
title: 改 marker / 数据 schema 时必须遍历整条调用链，单点改 schema 会留隐性假设地雷
type: lesson
status: active
confidence: 85
tags:
  - schema
  - architecture
  - refactor
  - plan-review
  - codex-review
source: task-343
createdAt: 2026-05-13T00:00:00.000Z
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# 改 schema 时必须遍历整条调用链，单点改 schema 会留隐性假设地雷

## 教训

把字段从 "required" 改为 "optional" 时（如 343 把 marker.taskId 从必填改可空），不能只动 schema 定义本身。**整条数据流的每个消费点都隐性假设了原约束**，必须挨个验证：

- queue.cjs `enqueue` 校验 taskId
- enqueue CLI 必填 `--task`
- runner.ts 用 `taskId || entryId` fallback
- worker.mjs `--task required`
- worker.mjs 把 taskId 用于 git path / Telegram / dispatch
- enqueue 的 git commit message
- tRPC poolStatus 返字段
- UI 显示逻辑

任一漏改都会让 taskless marker 在到达 Mac 后失败（runner 把 entryId 当 taskId 传给 worker；worker 在 dispatch 时 codex-dispatch 用 RP-NN 作 plan.md 路径 → 404）。

## 原因

数据 schema 的"必填→可选"看上去是单点改动，实际隐含**全链路 invariant 放宽**。所有消费点都从这个 invariant 推导出"我拿到 taskId 一定非空"的假设——assumption tree 散布在 N 个文件，IDE 类型系统也未必能扫到（因为 null union 是 TS 层加的，运行时 JS 不强校验）。

## 后果（如果忽略）

343 plan v1 我只列了 "queue.cjs + enqueue.mjs + worker.mjs" 三个改动点，**漏了** runner.ts 的 `candidate.taskId || entryId` fallback + enqueue commit message + task-create CLI 不接 `--direction` 的事实。如果没有 Codex Round 1 review 点出这些，taskless marker push 到 Mac 后会：

- runner 把 RP-NN 当 taskId 传 `--task RP-NN`
- worker.mjs 拿 `--task RP-NN` 后续 dispatch `codex-dispatch --task RP-NN`
- dispatcher 找不到 `.imoss/tasks/active/RP-NN-*/plan.md` 失败
- marker 卡死，调试时一脸懵

更糟糕的是 commit message `chore(297): enqueue research null` 会出现，文本污染 git log。

## 正确做法

**Schema 改动 plan 必须**：

1. **跨文件 grep 字段名**：`grep -rn 'taskId' scripts/ dashboard/server/ --include='*.mjs' --include='*.ts' --include='*.cjs'`
2. **分类调用点**：write site / read site / pass-through site
3. **逐个验证假设**：每个调用点回答"if this field is null, what happens?"
4. **plan 列出全部调用点**：每个都要在 plan 步骤里有改动 entry
5. **Codex review 必跑**：自己再细心也会漏，让 Codex 当独立眼睛过一遍

把这条作为"schema 改动任务"的常规步骤，不要当成"这次特殊小心" — 系统化的事一旦依赖人脑细心就会反复失败。

## 适用范围

任何涉及：
- 必填 → 可选 / 可选 → 必填 字段变更
- 字段语义改变（e.g. 数值范围扩大、类型从 string 变 union）
- 跨进程/跨机数据流（git push/pull 同步的 marker）
- 多个 caller 共享 schema 定义的场景

## 参考

- task 343 Codex Round 1 review：4 项修订都是 plan 漏列的调用点（runner / worker / task-create / commit msg）
- 反例：task 342 把 marker 文件名从 taskId 改 seqId 时也经历类似修订（rev 1 漏了 backward compat list 调用点）→ schema 改动连续两次踩同款坑，已具有规律性
- 建议把"跨文件 grep + 调用点分类"加入 plan template（research / schema 类任务 plan checkbox）
