---
id: decision-2026-05-knowledge-refactor-cli-3-subcommands
title: knowledge-refactor CLI — find-refs / rename / merge 3 subcommand 安全更新引用
status: active
confidence: 85
tags:
  - knowledge
  - refactor
  - cli
  - wikilink
  - dry-run
source: task-265
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
## 决策

新增 `scripts/knowledge-refactor.mjs` 主动重构 CLI，3 subcommand：

1. **`find-refs <slug>`** — 入链 + 出链 + frontmatter relation 引用（只读）
2. **`rename <old> <new>`** — 改文件名 + 更新所有 6 类引用形态（body wikilink + 4
   relation 字段 + supersedes/supersededBy + source.externalRef + distilledTo 路径型）
3. **`merge <from> <to>`** — body 合并 + 标 superseded + **入链重定向**（`[[from]]` → `[[to]]`）

默认 **dry-run** + `--apply` 显式确认；apply 后跑 fix-backlinks + --check + lint
--rule supersedes 三步验证。

## 理由

iter3 lesson "personal kb 必须支持 editor-grade refactoring"：
- 知识库长出来后，rename / merge / dead-link 修复是日常需求
- 手工改 6 类引用太容易漏（特别是 frontmatter relation 字段 + path 引用）
- 让 CLI 处理引用一致性，用户只决定语义

dry-run + apply 两步是因为重构是大改动，不可逆，必须给用户预览机会。

## 操作规范

- 改文件名先跑 `find-refs <slug>` 看影响范围
- `rename --apply` 后立即 grep 检查："还有没有漏改的 wikilink"（即使工具说 OK）
- merge 时优先把信息密度高的一方当 `<to>`，避免反向（避免重要 detail 丢失）
- 安全：slug 多处命中 fail-fast / new slug 不存在校验 / slug 型 vs 路径型引用区分

## 参考

- spec: `.imoss/rules/core/knowledge-refactoring.md`
- 实现: `scripts/knowledge-refactor.mjs`
