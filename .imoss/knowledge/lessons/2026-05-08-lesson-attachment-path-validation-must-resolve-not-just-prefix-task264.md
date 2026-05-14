---
title: 路径穿越校验必须 resolve 后比对前缀，不能只看字符串前缀
type: lesson
status: active
confidence: 85
tags:
  - security
  - path-traversal
  - validation
source: task-264
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# 路径穿越校验必须 resolve 后比对前缀，不能只看字符串前缀

## 教训

校验一个用户传入的相对路径是否在允许目录内时，必须：
1. **拒绝 absolute** + `..` + 反斜杠（基础卫生）
2. **resolve 后**比对解析后的绝对路径前缀
3. **不能**只看字符串前缀（"以 attachments/ 开头"）— 因为 resolve 之前用户可
   以构造 `attachments/../../etc/passwd` 看似合法但解析后跳出

## 原因

264 originalAttachment 校验最初版本：
```js
if (!path.startsWith('attachments/')) reject
```
被 review 拍：用户传 `attachments/../../etc/passwd` 字符串前缀符合，resolve 后
跳出 `.imoss/inbox/attachments/`。

## 后果（如果忽略）

- 路径穿越漏洞，攻击者从 capture 入口可以引用任意系统文件
- 即使有 inbox 目录隔离，也可能通过 attachment 路径让 inbox 文件指向系统秘密

## 正确做法

```js
// scripts/inbox-add.mjs:validateAttachmentPath
function validateAttachmentPath(p, imossRoot) {
  if (typeof p !== 'string' || p.length === 0) throw new Error('必须非空字符串');
  if (p.startsWith('/') || p.match(/^[A-Za-z]:[\\/]/)) throw new Error('不接受绝对路径');
  if (p.includes('..') || p.includes('\\')) throw new Error('不接受 .. 或反斜杠');

  const attachmentsDir = getInboxAttachmentsDir(imossRoot);
  let relPath = p.startsWith('attachments/') ? p : `attachments/${p}`;
  const resolved = resolve(getInboxDir(imossRoot), relPath);
  const expectedPrefix = resolve(attachmentsDir);
  // resolve 后再比对，不仅看字符串前缀
  if (!resolved.startsWith(expectedPrefix + '/') && resolved !== expectedPrefix) {
    throw new Error(`路径必须 resolve 到 ${expectedPrefix}/`);
  }
  return relPath;
}
```

通用规则：
- 路径校验三步：syntax 检查 → resolve → resolved-prefix 比对
- 比对必须用 `resolve(absolute base) + sep` 前缀（避免 prefix 误匹配，例
  `/etc/passwd` 匹配 `/etc/passwd2`）
- 任何"用户传入的相对路径"都要走这个 helper，不要每个调用方各写一遍

## 参考

- 实现: `scripts/inbox-add.mjs:validateAttachmentPath`
- 类似机制: `scripts/lib/code-graph.cjs:validateRelativePath` (272 复用)
