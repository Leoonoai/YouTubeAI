---
title: '公开 MCP content[0] 必须在 profile 分支完成后再 JSON.stringify'
type: lesson
status: active
confidence: 85
tags:
  - mcp
  - server
  - payload
  - profile
  - ordering
source: task-270
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# 公开 MCP content[0] 必须在 profile 分支完成后再 JSON.stringify

## 教训

MCP server handler 里如果按 `profile` 给 hits 加新字段（如 270 的 `injected` /
`injectedReason`），**必须**在 profile 分支跑完后再生成 `payloadText`，否则新字段只
活在 server 内存里，永远不会出现在 `content[0].text` 公开响应里。

## 原因

最初 patch 顺序是：
```js
const top = baseTop;
const payloadText = JSON.stringify(top);   // ← 在这里
// ... 后面才跑 markInjectedForAiAugmented(top, ...)
```
`top` 是 const 引用 baseTop，profile 分支之后即使重新赋值新数组也不会回写
payloadText。Codex R1 review 直接拍出来这个顺序 bug，要求改成"profile 分支 → 重赋
top → 再 stringify"。

## 后果（如果忽略）

- ai_augmented profile 调用方收到 hits **看不到 injected 字段** — 表面看正常
  但实际功能完全没生效
- 单测可能覆盖 helper（`markInjectedForAiAugmented` 行为对），但没覆盖 server
  集成，问题溜过 helper 单测进生产
- 由用户自己跑 MCP 调用才发现差异，调试要追到 server.mjs payloadText 生成顺序

## 正确做法

```js
// 1. baseTop 含全部静态字段（含 reviewAfter）
const baseTop = merged.slice(0, cappedLimit).map(...);
// 2. profile 分支决定最终 top
const top = profile === 'ai_augmented' ? markInjectedForAiAugmented(baseTop, today) : baseTop;
// 3. abstain / coverage 等 metadata 单独算
const abstain = computeAbstain({ coverage, profile });
// 4. **最后** stringify final top
const payloadText = JSON.stringify(top, null, 2);
```

防御措施：MCP smoke-test 必须用 fixture 覆盖**真实 stdio 响应** shape，不能只测
helper（详见 task 270 smoke-test 的 270.A 测试）。

## 参考

- 实现: `optional/mcp/server.mjs:1160-1220`
- review-log Round 1 修订摘要: `.imoss/tasks/done/270-imoss-memory-search-加-profile-/review-log.md`
