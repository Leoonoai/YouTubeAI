---
id: decision-2026-05-inbox-capture-first-triage-later
title: inbox 层用 capture-first / triage-later，4 分支结构化处置
status: active
confidence: 85
tags:
  - inbox
  - capture
  - triage
  - lifecycle
source: task-259
createdAt: 2026-05-07T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

新增 `.imoss/inbox/` 临时捕获层 + 4 分支 triage CLI，遵循 capture-first /
triage-later 模式：

- **capture**：Telegram 截图 / 浏览器收藏 / 语音笔记 / 手贴随手记，落盘 ≤2s 不
  过审、不分类
- **triage**：用户在合适时机批量决策，4 分支：
  1. 删除（无价值）
  2. 转 reference（外部原文存档）
  3. 蒸馏成 lesson（结构化收藏到 personal/）
  4. 绑定到 task（变成正在做任务的输入）

inbox 默认 git ignore，triage 后才进 git。Dashboard `/inbox` 页 + CLI
`scripts/inbox-{add,list,triage}.mjs`。

## 理由

iter2 supersedes iter1 的"inbox-reference-permanent 4 层生命周期" — capture-first
比 4 层提前分类更接近真实使用模式：用户截屏时不知道 final 分类，强制立刻分类只
会让人懒得 capture。

## 操作规范

- 不强制 triage 时间窗（inbox 可堆任意久）
- 4 分支以外的"半结构化保留" 不开（避免成第二个 personal/）
- 264 capture protocol 提供 stateless wrapper 让外部入口直接写 inbox

## 参考

- spec: `.imoss/rules/core/inbox-capture-protocol.md`
- 实现: `scripts/inbox-{add,list,triage}.mjs` + `dashboard/client/src/pages/Inbox.tsx`
- 相关：[[2026-05-08-decision-memory-write-boundary-validation-task271]]

## 反向引用

<!-- generated-by: knowledge-link-check -->

> 此区由 `scripts/knowledge-link-check.mjs --fix-backlinks` 自动维护，手工修改会被覆盖。

- [[2026-05-07-lesson-triage-4-branches-not-more-task259|inbox triage 4 分支够用，不要为"半结构化保留"开第 5 分支]]
- [[2026-05-08-decision-inbox-capture-stateless-wrapper-task264|inbox capture 入口用 stateless CLI wrapper，不开 daemon / 长期 worker]]
