---
title: gray-matter 把 YAML 无引号 ISO 日期解析成 JS Date 实例 — date helper 必须支持
type: lesson
status: active
confidence: 90
tags:
  - yaml
  - gray-matter
  - date
  - normalization
source: task-269
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# gray-matter 把 YAML 无引号 ISO 日期解析成 JS Date 实例 — date helper 必须支持

## 教训

写 frontmatter 字段 normalize helper（如 269 timeline 的 `happenedAt` / `createdAt`）
时，**必须同时接受字符串 + JS Date 实例**。gray-matter 默认用 js-yaml 解析，
无引号的 `createdAt: 2026-04-10` 直接解析成 `Date` 对象，**不是字符串**。

## 原因

269 buildTimeline 第一版只接受字符串：
```js
function normalizeWhen(raw) {
  if (typeof raw !== 'string') return null;  // ← 这里把 Date 排除掉了
  ...
}
```

smoke-test 跑：
```
✓ 269.A timeline A happenedAt 2026-03-15   (用 quoted string，pass)
✗ 269.A timeline B createdAt 2026-04-10    (无引号，gray-matter 解析成 Date，被拒)
```

YAML 1.1 spec 把无引号的 ISO date 自动转 Date；带引号 `'2026-04-10'` 才是字符串。
现存 corpus frontmatter 大量混用两种写法，helper 必须兼容。

## 后果（如果忽略）

- timeline / lifecycle / 任何 date 字段 helper 默默忽略一半文件（无引号写法那部分）
- 单测只覆盖字符串 case 时全 pass，集成测试遇到真实 corpus 才暴露
- 用户写 frontmatter 时被迫加引号「为了让工具识别」，违反"让 YAML 自然写"原则

## 正确做法

```js
function normalizeWhen(raw) {
  if (raw == null) return null;
  // Date 实例（gray-matter YAML 无引号日期）
  if (raw instanceof Date) {
    if (Number.isNaN(raw.getTime())) return null;
    return raw.toISOString().slice(0, 10);
  }
  // 字符串（YAML 引号日期 + 显式字符串）
  if (typeof raw === 'string') {
    if (!ISO_DATE_RE.test(raw)) return null;
    return raw.slice(0, 10);
  }
  return null;
}
```

通用规则：
- 任何读 frontmatter date 的 helper 都要接受 `string | Date`
- 测试 fixture 用无引号 + 引号两种写法都覆盖
- 267 normalizeIsoDate 同理（已支持）

可选的 anti-drift：sync-rebuild / lint 阶段把所有 Date 实例字符串化成 ISO 字符串
落盘，但 v1 不强制（YAML 自然性优先）。

## 参考

- 实现: `optional/mcp/memory-graph.mjs:normalizeWhen`
- 类似机制: `optional/mcp/memory-corpus.mjs:normalizeIsoDate` (267)
- gray-matter YAML 解析: js-yaml SAFE_SCHEMA
- 相关决策: [[2026-05-08-decision-memory-graph-relations-timeline-task269]]
