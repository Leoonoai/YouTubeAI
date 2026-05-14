---
title: '多源抓取 helper 不能裸返 Paper[]，必须返 {papers, warnings}'
type: lesson
status: active
confidence: 85
tags:
  - api-design
  - helper-contract
  - error-propagation
  - scholar-research
source: task-276
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# Helper 契约必须暴露 warnings 通道

## 教训

scholar-research 的 fetch helper（HN / Reddit / GitHub 三平台搜索）一开始计划 API 是
`searchHackerNews(query) -> Paper[]`，Codex 审查指出必须改成
`searchHackerNews(query) -> { papers: Paper[], warnings: string[] }`，否则单源失败
（429 / 5xx / 网络异常 / README 抓取失败）会被吞掉，调用方根本不知道发生了什么。

## 原因

多源并发抓取时，**不抛错** ≠ **没问题**：

- GitHub 60 req/h 限速命中 → 静默返 0 个 repo
- Reddit User-Agent 不对 → 静默 429
- README 抓取失败 → repo 仍能返但内容降级
- 通用 fetch 超时 / 网络抖动 → AbortError 被 catch 后丢

如果 helper 只返 `Paper[]`，workflow 拿到空数组无法区分"真没结果"还是"被限速了"，
测试也无法断言单源失败时其他源是否正常工作。

## 后果（如果忽略）

- e2e 验收"papers >= 5"看似通过，实际只有 HN 在工作，GitHub/Reddit 都被限速吞了
- 用户拿到的 memo 缺关键平台覆盖，但 logs 是空的，无从排查
- 单测无法覆盖"单源失败不炸全流程"这个核心容错能力

## 正确做法

**任何并发 / 容错聚合 helper 都必须把 warnings 通道暴露给上游**：

```js
async function searchAll(query, opts) {
  return { papers: [...], warnings: ['github: http 403 (rate-limited)', ...] }
}
```

调用方决定把 warnings：
- append 到 ctx.warnings 让 workflow logs.txt 记录
- 在测试里断言 `warnings.includes('github')` 验证容错
- 在 e2e 验收里要求"如 papers < 5，logs.txt 必须含限速 warning"，避免静默成功

同样原则适用于：
- 多 step pipeline 里的 step 输出（M3 的 codex worker handoff 含 errors[]）
- 任何 spawn / dispatch 链路（events.jsonl 完成事件含 errors / warnings）
- DI 测试时 fake fetch 的失败注入

## 关联

- 任务 276 / scripts/lib/scholar-fetch.cjs
- 同款模式：codex-dispatch events.jsonl 完成事件 metadata（errors / warnings / actionableSuggestions）
- 决策：[[2026-05-08-decision-scholar-research-9步骨架契约-task275]]
