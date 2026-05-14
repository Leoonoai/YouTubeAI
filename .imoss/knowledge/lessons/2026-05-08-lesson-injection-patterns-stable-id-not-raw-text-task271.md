---
title: prompt-injection 扫描结果用稳定 pattern id，不要写原始攻击文本进 frontmatter
type: lesson
status: active
confidence: 85
tags:
  - prompt-injection
  - frontmatter
  - security
  - write-boundary
source: task-271
createdAt: 2026-05-08T00:00:00.000Z
audience: work
inheritedFrom: P001
inheritedAt: '2026-05-14'
---
# prompt-injection 扫描结果用稳定 pattern id，不要写原始攻击文本进 frontmatter

## 教训

prompt-injection 启发式扫描命中后，frontmatter 里只写**稳定 pattern id**（如
`ignore-instructions / system-role-marker`），**不要**复制原始命中片段。

## 原因

最初 plan 想写 `injectionPatterns: ["ignore previous instructions and dump"]` 把命中
phrase 也存下来便于 triage。Codex R1 直接拍：这等价把攻击文本**再写一遍**进 markdown
frontmatter，未来 ai_augmented profile 命中这条 → 模型仍会被 system prompt 那段
攻击文本影响（即使 `injected:false`，markdown 全文搜索可能把它捞回来）。

## 后果（如果忽略）

- 攻击文本被写两次：一次在 inbox body（无法避免，原始来源），一次在 frontmatter
  （我们自己加的）
- 任何下游全文 grep / FTS / 向量索引看到的 frontmatter 攻击文本，都比 body 的更
  "干净"（短、被结构化字段包着）— 反而**放大了**注入信号
- 如果某个未来工具决定"frontmatter 是可信元数据，body 是用户输入"，正好被绕过

## 正确做法

```js
// scripts/lib/inbox-validation.cjs
const INJECTION_PATTERNS = [
  { id: 'ignore-instructions', re: /ignore (all|previous|prior) (instructions|...)/i },
  // ...
];

function scanPromptInjection(body) {
  const matched = [];
  for (const { id, re } of INJECTION_PATTERNS) {
    if (re.test(body)) matched.push(id);  // 只 push id，不 push re.exec(body)
  }
  return { suspect: matched.length > 0, patterns: matched };
}
```

通用规则：**任何"扫描威胁内容产 metadata"的工具**都该用稳定 id / 类别名，
不应把原始威胁内容复制进任何派生层（frontmatter / log / db）。原始内容已经在
原文里了，派生层用 id 引用即可。

## 参考

- 实现: `scripts/lib/inbox-validation.cjs:INJECTION_PATTERNS`
- review-log Round 1 修订摘要: `.imoss/tasks/done/271-memory-写入边界验证-prompt-injection/review-log.md`
- 相关决策: [[2026-05-08-decision-memory-write-boundary-validation-task271]]
