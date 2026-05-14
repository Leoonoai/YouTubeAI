---
title: rename CLI 必须实际改文件名，不能只改引用
type: lesson
status: active
confidence: 85
tags:
  - knowledge
  - refactor
  - rename
  - file-system
source: task-265
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# rename CLI 必须实际改文件名，不能只改引用

## 教训

写"rename slug"工具时，必须**同时**做两件事：
1. 实际 `fs.renameSync(oldPath, newPath)` 改文件名
2. 更新所有引用（body wikilink + frontmatter relation + path 型）

最初实现差点漏第 1 步（"只更新引用，让用户自己改文件"）。Codex R1 拍出。

## 原因

设计 rename 工具时，第一直觉是"工具只管引用一致性，文件操作交给用户"。但这
违背工具的核心价值：用户**就是想避免手工改引用**才用工具。

如果只改引用不改文件：
- 改完所有 `[[old]]` → `[[new]]`，但 `new.md` 不存在 → 全部变 dead link
- 用户必须额外手工 mv 文件，工具反而让事情更复杂

## 后果（如果忽略）

- rename 完成后所有引用都断了（指向不存在的 new.md）
- 用户不信任工具，回到手工改引用模式
- 后续 fix-backlinks 跑出来一堆 dead-link warning

## 正确做法

```js
// scripts/knowledge-refactor.mjs:renameSlug
async function renameSlug(oldSlug, newSlug, opts) {
  const oldPath = findFileBySlug(oldSlug);
  const newPath = oldPath.replace(`/${oldSlug}.md`, `/${newSlug}.md`);

  if (!opts.apply) {
    console.log(`[dry-run] would rename ${oldPath} → ${newPath}`);
    console.log(`[dry-run] would update N references`);
    return;
  }
  // 实际改名 + 更新引用
  fs.renameSync(oldPath, newPath);
  await updateAllReferences(oldSlug, newSlug);
  console.log(`renamed ${oldPath} → ${newPath}`);
  console.log(`updated N references`);
}
```

通用规则：
- 工具的 atomic operation 应该完整覆盖用户意图，不让用户做"工具没做的剩余步骤"
- dry-run 模式必须列出**所有**会发生的副作用（rename + reference updates），
  不能只列其中一种

## 参考

- 实现: `scripts/knowledge-refactor.mjs:renameSlug`
- review-log Round 1: `.imoss/tasks/done/265-knowledge-renamemergefind-refe/review-log.md`
