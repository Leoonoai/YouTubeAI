---
title: 升级 frontmatter enum 必须保留 legacy alias，不要破坏旧文件
type: lesson
status: active
confidence: 85
tags:
  - frontmatter
  - enum
  - migration
  - backward-compat
source: task-262
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# 升级 frontmatter enum 必须保留 legacy alias，不要破坏旧文件

## 教训

新引入 frontmatter enum 字段（或重命名旧 enum 值）时，必须**显式列出 legacy alias**，
让旧 frontmatter 自动映射到新 enum，不需要批量改文件。

## 原因

262 sourceType enum 设计时发现现有 corpus 已有几种相近字段值：
- `user-paste` / `user-paste-promotional` / `manual-note`

新 enum 只想保留 `manual`（更通用）。如果直接换 → 所有旧 frontmatter 突然
变成"非法 enum 值"，lint 会全部报错，search filter 会过滤掉旧文件。

## 后果（如果忽略）

- lint 大批量误报 invalid enum
- MCP search 用 sourceType=manual filter 漏掉所有旧 user-paste 文件
- 用户必须批量改 200+ frontmatter 才能用新功能

## 正确做法

```js
// optional/mcp/memory-corpus.mjs:normalizeSourceType
const LEGACY_ALIASES = {
  'user-paste': 'manual',
  'user-paste-promotional': 'manual',
  'manual-note': 'manual',
};
function normalizeSourceType(raw) {
  if (LEGACY_ALIASES[raw]) return LEGACY_ALIASES[raw];
  if (VALID_SOURCE_TYPES.includes(raw)) return raw;
  return null;
}
```

通用规则：
1. 新 enum 加进来前 grep 现有 corpus 已用值
2. 显式映射所有 legacy 值（即使是 deprecated）
3. derived default + alias 都在同一 normalize helper 里
4. lint 默认放行 legacy alias（可显式 strict 模式拒绝）

不强制大批量迁移，让 corpus 有机生长。

## 参考

- 实现: `optional/mcp/memory-corpus.mjs:normalizeSourceType`
- 相关：[[2026-05-08-decision-source-metadata-multimodal-quality-task262]]
